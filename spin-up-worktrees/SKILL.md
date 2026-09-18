---
name: spin-up-worktrees
description: Open one or more git worktrees, each in its own nested herdr workspace — a single task or several in parallel, with or without an associated ticket. Use when the user wants to work on a ticket/issue/task in its own worktree, mentions "open a worktree (or worktrees) for this", "spin up worktrees", or wants several worktrees ready in parallel.
---

# Spin Up Worktrees

Spin up one or more worktrees, each in its own nested herdr workspace running a plain shell. The current session stays put and never enters any of them. Works the same for one worktree or several in parallel, and whether the work is a tracker ticket, a plain description, or nothing more specific than a branch name.

Each workspace is left with a shell pane at the worktree path, ready for whatever starts next — starting a coding agent is a separate step.

Requires `wt` (see the `worktrunk` skill) and a running herdr server (see the `herdr` skill for CLI mechanics and JSON shapes). Check with `herdr status` — `server: running` means it's up. If it's not, start it yourself rather than asking the user: `herdr server` runs headless (it's a documented mode — `herdr --help` lists it under "Advanced commands: Run as headless server"), and none of this skill's operations (`worktree open`, `workspace list`, `workspace close`) need an attached TUI client, only the server. Background it — `herdr server >/dev/null 2>&1 &` or your tool's equivalent — then re-check `herdr status` before continuing.

## Workflow

### 1. Name a branch per worktree

If a tracker ticket exists, use `issue-<number>-<short-slug>` (e.g. `issue-2-tool-package`) so the worktree path stays traceable back to it. Otherwise, pick a short kebab-case branch name that describes the work (e.g. `refactor-auth-middleware`).

If the work already has a branch — an existing PR, a bot-created branch (Renovate, Dependabot), anything you were handed rather than invented — use that name verbatim and treat it as **adopted**: step 2 checks out the real branch instead of creating a fresh one.

### 2. Create the worktree(s)

Check whether the branch is adopted or new before picking the command — this decides `--create` or not:

```sh
git fetch origin <branch>
```

If that succeeds, the branch already exists on the remote — track it directly, no `--create`:

```sh
wt switch <branch> --no-cd
```

Otherwise it's new:

```sh
wt switch --create <branch> --no-cd
```

`--no-cd` keeps _this_ session's working directory untouched — you're spawning a sibling, not moving in yourself.

**`wt switch --create` on a branch that already exists on the remote doesn't fail — it silently creates a new local branch from the base instead of checking out the real one**, warning `creating new branch from base instead` on the way. If you see that warning, you used the wrong form: `wt remove --foreground <branch>` to undo it, then rerun as `wt switch <branch> --no-cd`.

If this repo's `.config/wt.toml` hooks aren't yet approved, `wt` refuses to run non-interactively. That approval is the user's call, not yours — follow the `worktrunk` skill's "Hook Approvals in Non-Interactive Sessions" guidance rather than reaching for `--yes` on your own judgment.

Once the user has given a standing answer for _this repo_ (e.g. "just use `--yes`"), apply it on subsequent `wt switch --create` calls in the same repo without re-asking — the trust decision was already made, re-litigating it every time is noise. Ask again only for a repo they haven't decided on yet, or if `.config/wt.toml`'s hooks have changed since they decided.

If this repo's `post-start` hook opens its own pane or workspace, ignore it — this skill spawns its own workspace in step 3, and step 4 cleans up whatever the hook left behind.

### 3. Open the worktree as a nested herdr workspace — don't wait on a hook

herdr's worktree commands are repo-aware: opening a checkout nests its workspace under any others on the same repo automatically, so no `--workspace` parent hint is needed. See the `herdr` skill for CLI mechanics and response shapes.

Per worktree:

```sh
herdr worktree open --path <worktree-path> --label "<short title>" --no-focus
```

`--no-focus` is what keeps this backgrounded — without it the new workspace steals focus. Pull the workspace id from the response (`.result.workspace.workspace_id`) and carry it into step 4.

Use the ticket's number and title for `<short title>` if one exists (e.g. `Issue #<n>: <short title>`); otherwise a short description of the task, or the branch name if there's nothing more descriptive to give it. This also names the workspace up front, so there's no separate rename step later.

### 4. Repeat for each additional worktree, then verify no stray workspaces

If spinning up more than one, repeat steps 1–3 per worktree. Either way, check `herdr workspace list` for each worktree path you targeted: it should map to exactly one workspace, and that workspace's `workspace_id` should be the one your own `herdr worktree open` call returned in step 3 — no pre-run snapshot needed, since you already know the id you're checking for. A workspace at your target path whose id you didn't mint yourself is a stray — this repo's `post-start` hook can spawn one on its own, not just a leftover multiplexer, so treat any hook-spawned workspace at your path as disposable regardless of what it's wired to. Id identity is the signal here, not screen content — don't gate the close on `pane read`, since a freshly opened pane can be slow to report on its first read whether or not it's a stray.

**Close strays with `herdr workspace close <id>` — never `herdr worktree remove`.** `remove` deletes the underlying git worktree checkout from disk (verified); `close` only tears down the herdr-side presentation and leaves the checkout untouched. This skill only ever creates worktrees — deleting one is `cleanup-branch`'s job, not a byproduct of tidying up a duplicate pane here.

Confirm each workspace is open at its worktree path, then stop.

## Completion criterion

One herdr workspace per worktree, each named for its task and rooted at its worktree path with a plain shell — and the workspace count matches exactly: no extras, nothing missing.
