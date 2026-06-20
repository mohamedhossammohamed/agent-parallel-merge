# AGENTS.md — Concurrency & Deployment Rules

First run: read SOUL.md — this is who you are.
This file governs git, CI, and merges. Operational only.

## Roles

Every agent is a **Worker** unless explicitly invoked as the **Merge Agent**
for this session. Only one Merge Agent runs at a time per repo.

- **Worker**: produces a change, opens a PR, stops. Never merges, never
  touches `main`, never holds the lock.
- **Merge Agent**: holds the deploy lock, processes the PR queue one at a
  time, merges, verifies, releases the lock. Does not write feature code.

If you were not explicitly told you are the Merge Agent, you are a Worker.
The Merge Agent is invoked explicitly — e.g. "Run as Merge Agent on this
repo" — or by a dedicated loop/skill. Workers never self-promote into the
Merge Agent role, regardless of queue length or wait time.

## Worker Rules

1. Never `git checkout -b` inside the primary project directory.
2. Always start a task with a fresh worktree off `origin/main`:
   ```
   git fetch --all --prune
   AGENT_DIR="../wt-$(date +%s)-<slug>"
   git worktree add -b "agent/<slug>" "$AGENT_DIR" origin/main
   cd "$AGENT_DIR"
   ```
3. Never push directly to `main`.
4. Never run `git merge` or `git rebase` against `main` and push the result yourself.
5. On task completion, push the branch and open a PR. Do not merge it.
   ```
   git push -u origin "agent/<slug>"
   gh pr create --base main --label agent-merge-queue \
     --title "<imperative summary>" \
     --body "Task: <one line>. Worktree: $AGENT_DIR. Verified: <how>."
   ```
6. The `agent-merge-queue` label is required on every PR. This is the only
   handoff to the Merge Agent — do not message it directly, do not retry
   the merge yourself if it's slow.
7. If `main` has advanced more than one merge ahead of your branch: abort.
   Do not resolve the conflict. Delete the worktree, re-fetch `origin/main`,
   restart the task in a new worktree.
8. If your PR is ejected from the queue: treat as the same abort signal as
   rule 7. Restart, do not patch.
9. Never poll CI or the GitHub API in a tight loop. Use:
   ```
   gh pr checks "agent/<slug>" --watch --interval 30
   ```
10. When waiting on another agent's task, poll its PR per rule 9. Never
    assume completion.
11. On task completion, clean up the worktree immediately:
    ```
    cd ..
    git worktree remove "$AGENT_DIR"
    git branch -D "agent/<slug>"
    ```
12. If `git worktree add` fails (path or branch exists): re-derive the slug.
    Do not force-overwrite.

## Merge Agent Rules

13. Before processing any PR, verify `.github/workflows/ci.yml` contains:
    ```yaml
    concurrency:
      group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
      cancel-in-progress: ${{ github.event_name == 'pull_request' }}
    ```
    If missing, add it first, in its own PR, before processing anything else.
14. Acquire the deploy lock before touching any queued PR. Use a file-based
    lock at the repo root, independent of whether GitHub's merge queue
    feature is enabled — do not assume it is. If running this protocol
    across multiple repos, the lock is per-repo — never share one lockfile
    across repos:
    ```bash
    LOCKFILE=".deploy.lock"
    WAITED=0
    MAX_WAIT=900   # 15 minutes
    while [ -f "$LOCKFILE" ]; do
      sleep 15
      WAITED=$((WAITED + 15))
      if [ "$WAITED" -ge "$MAX_WAIT" ]; then
        echo "Lock held >15min — stale lock suspected. Stop. Do not force-remove. Flag for human review."
        exit 1
      fi
    done
    touch "$LOCKFILE"
    ```
15. Process exactly one PR labeled `agent-merge-queue` at a time, oldest first:
    - Fetch latest `main`.
    - Skip (do not close) any PR more than one `main` commit behind or with
      unresolved review comments — leave it for the next pass.
    - Merge with:
      ```bash
      gh pr merge <number> --squash --delete-branch
      ```
    - Run `gh pr checks --watch --interval 30` against the resulting `main`
      run before moving to the next PR.
    - Verify from a clean checkout, not from the PR's worktree:
      ```bash
      git fetch origin
      git checkout main
      git pull
      # run the repo's smoke test / health check here
      ```
    - If CI fails on `main` after merge, or the clean-checkout verification
      fails: stop processing the queue, leave the lock held, flag the PR,
      do not auto-revert.
16. Release the lock after each PR is fully verified, not after the whole
    queue — this lets a fresh Merge Agent invocation resume mid-queue if
    interrupted:
    ```bash
    rm "$LOCKFILE"
    ```
17. Never merge two PRs in the same pass without the CI-verify step between
    them, even if both look unrelated.

## Forbidden

- `git push --force` / `git push -f` to any shared branch
- Direct commits to `main`
- `git clean -fdx` or recursive destructive cleanup outside your own worktree
- Manual resolution of a merge conflict by rewriting both sides of a diff
- Tight-loop polling without `--interval`/backoff
- A Worker acquiring the deploy lock or merging its own PR
- More than one Merge Agent holding the lock at once
- Force-removing a lock file after a timeout instead of flagging for review
