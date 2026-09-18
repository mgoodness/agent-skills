---
name: cleanup-branch
description: Clean up a branch and its associated worktree, remote ref, and local tracking branch. Use when the user wants to clean up after a PR is merged, or mentions "delete worktree" or "delete branch".
---

# Cleanup Branch

Remove a branch and its worktree once its PR is merged: the local branch, the remote ref, and the herdr workspace it ran in. See the `herdr` skill for CLI mechanics and JSON shapes.

## Workflow

### 1. Find the branch and its PR

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

### 7. Tear down the worktree's herdr workspace before removing it

Merge status only tells you the git state is clean — not that an agent has stopped touching the directory. If a worktree exists, resolve it to its herdr workspace:

```sh
herdr worktree list | jq -r --arg p "<path>" '.result.worktrees[] | select(.path==$p) | .open_workspace_id'
```

A null or empty result means no workspace is open there — skip to step 8.

Otherwise inspect the workspace's panes:

```sh
herdr pane list --workspace <workspace-id>
```

Any pane carrying an `agent` field is a live agent — any value counts (`claude`, `pi`, `codex`, …), regardless of `agent_status`. A pane without the field is a bare shell. Closing the workspace stops whatever runs in it, so this tears down every agent kind with no per-agent exit command:

```sh
herdr workspace close <workspace-id>
```

Confirm it no longer resolves (`herdr workspace get <workspace-id>`) before continuing, so the agent is stopped before the directory goes away. If herdr refuses the close, surface that to the user rather than forcing it; the `herdr` skill covers close semantics.

### 8. Remove the worktree and/or branch

**If a worktree exists** — use worktrunk, which handles removal, metadata pruning, and branch deletion in one step:

```sh
wt remove <branch>
```

See the `worktrunk` skill for what `wt remove` actually does (hook timing, background trash-and-prune, merge detection) rather than re-deriving it here.

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

### 10. Report what was done

Note the live agent stopped (if one was found), branch deleted, and worktree removed (if applicable).

## Completion criterion

The branch is gone locally and from `origin`, its worktree no longer appears in `git worktree list`, and no herdr workspace remains for it.
