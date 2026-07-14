---
name: cleanup-branch
description: Clean up a branch and its associated worktree, remote ref, and local tracking branch, with user confirmation. Use when the user wants to clean up after a PR is merged, or mentions "delete worktree" or "delete branch".
---

# Cleanup Branch

## Workflow

### 1. Detect context

Run `git rev-parse --show-toplevel` to get the current worktree path.
Run `git worktree list` to find all worktrees.

Find the default branch:

```sh
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
```

Identify the **main worktree** as the one whose branch matches the default branch.

If the current worktree IS the main worktree, enumerate all non-main worktrees and look up their associated PRs. For each non-main worktree, prefer running from within that worktree's directory when possible:

```sh
gh pr view --json number,state,mergedAt,title,headRefName
```

If the worktree path is outside the project root (not usable as a `cd` target), fall back to querying by branch name from the main worktree — include `--state all` to catch merged PRs:

```sh
gh pr list --head <branch> --state all --json number,state,mergedAt,title,headRefName
```

Present a table of results to the user:

| #   | PR  | Title         | State  | Worktree path     |
| --- | --- | ------------- | ------ | ----------------- |
| 1   | #42 | Fix login bug | MERGED | /path/to/worktree |
| 2   | #39 | Old branch    | MERGED | —                 |

Include all PRs found — merged, open, and those with no associated PR (show "—" for PR and title). Show "—" for worktree path when no worktree exists for that branch. Then ask: "Which branch should be cleaned up? (enter row number or PR number)" Resolve the selection to a `headRefName`, worktree path (or none), and continue from step 2.

### 2. Check for uncommitted changes

Run `git status --porcelain`. If output is non-empty, warn:

> "Uncommitted changes detected in this worktree. Stash or commit them, or use `wt remove --force` to discard."

Abort unless the user explicitly chooses `--force`.

### 3. Verify the merge

Run:

```sh
gh pr view --json number,state,mergedAt,title,headRefName
```

- If `state` is not `"MERGED"`, abort: "PR #N is not merged (state: STATE). Aborting."
- Record `headRefName` (branch) and `number` for later steps.

### 4. Switch to the main worktree

All remaining steps must run from the main worktree. Use the main worktree path as the `cd` parameter for every subsequent terminal call.

Verify you are in the right place:

```sh
git rev-parse --abbrev-ref HEAD
```

This should print the default branch name (e.g. `main`).

### 5. Pull the main worktree

From the main worktree path, check whether an `upstream` remote exists:

```sh
git remote | grep -q upstream && echo yes || echo no
```

If `upstream` exists, pull from it explicitly:

```sh
git pull upstream <default-branch>
```

Otherwise:

```sh
git pull
```

### 6. Confirm before deleting

Show the user exactly what will be removed:

- Branch: `<headRefName>`
- Worktree path: `<path>` (or "none")

Ask: "Delete branch `<branch>`[and remove worktree at `<path>`]? (y/N)"

Abort if the user declines.

### 7. Stop any live session in the worktree first

Merge status only tells you the git state is clean — not that a process has stopped touching the directory. If a worktree exists, check whether Claude is still running there before removing anything. Resolve the worktree path to a cmux workspace, then to that workspace's terminal surface:

```sh
cmux workspace list --json | jq -r --arg p "<path>" '.workspaces[] | select(.current_directory==$p) | .ref'
cmux list-pane-surfaces --workspace <workspace-ref>
```

If a workspace matches, read its surface with `cmux read-screen --surface <surface-id>` and confirm what's actually running. A bare shell prompt means nothing's live. Claude's own UI — status bar, a pending approval prompt, a busy spinner — means it is, even if it looks idle.

If it's live, ask it to exit rather than pulling the directory out from under it:

```sh
cmux send-key --surface <surface-id> Escape   # dismiss any pending approval prompt
cmux send-key --surface <surface-id> Ctrl+C   # interrupt anything in flight
cmux send --surface <surface-id> "/exit"
cmux send-key --surface <surface-id> Enter
```

Confirm the exit actually happened — `read-screen` should now show a bare shell prompt, not Claude's UI — before continuing to the next step. If it's still showing Claude's UI after a moment, don't force it closed: tell the user a live session is blocking cleanup and let them close it.

If nothing matches (no workspace, or it's already a bare shell), skip straight to the next step.

### 8. Remove the worktree and/or branch

**If a worktree exists** — use worktrunk, which handles removal, metadata pruning, and branch deletion in one step:

```sh
wt remove <branch>
```

Worktrunk runs `pre-remove` hooks, moves the worktree to `.git/wt/trash/` (background), prunes git metadata, and deletes the branch if merged.

Fallbacks:

- **Dirty worktree:** `wt remove --force <branch>`
- **Squash-merged / unmerged branch:** `wt remove -D <branch>` (offer this if `wt remove` declines to delete the branch)

**If no worktree exists** — just delete the local branch:

```sh
git branch -d <branch>
```

If `-d` fails (squash-merge), offer: `git branch -D <branch>`.

Alternatively, for bulk cleanup of all merged branches at once:

```sh
gh poi --state merged --dry-run   # preview
gh poi --state merged             # delete
```

Use `gh poi lock <branch>` to protect any branch that should be kept.

### 9. Delete remote branch

Runs for both paths — no-op if GitHub already deleted it:

```sh
git ls-remote --heads origin <branch> | grep -q . && git push origin --delete <branch> || true
```

### 10. Close the stale cmux workspace (worktree path only)

`wt remove` already deleted the directory, so the workspace resolved in step 7 (if any) now points at a path that no longer exists. cmux has no "refresh" to reconcile this — close it directly:

```sh
cmux close-workspace --workspace <workspace-ref>
```

Skip if no worktree existed, or step 7 found no matching workspace.

### 11. Confirm completion

Report what was done: live session stopped (if one was found), branch deleted, worktree removed (if applicable), stale cmux workspace closed (if applicable).
