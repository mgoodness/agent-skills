---
name: implement-in-worktrees
description: Open one git worktree per ticket and start an independent Claude Code session in each via muxy, so several frontier tickets get implemented in parallel. Use when the user wants to work multiple tickets/issues at once, mentions "open worktrees for these", "start sessions in each", "work these in parallel", or wants to fan /implement out across the frontier.
---

# Implement in Worktrees

Fan `/implement` out across several tickets at once: one worktree, one Claude session, per ticket. The current session stays put and never enters any of them. Each new session starts with `/implement` staged, not submitted — the user decides when it actually begins.

Requires `wt` (see the `worktrunk` skill) and a running Muxy (see `muxy-cli`) — read those for the mechanics this skill assumes rather than re-deriving them here.

## Workflow

### 1. Name a branch per ticket

Use `issue-<number>-<short-slug>` (e.g. `issue-2-tool-package`), so the worktree path stays traceable back to the tracker.

### 2. Create each worktree

```sh
wt switch --create <branch> --no-cd
```

`--no-cd` keeps _this_ session's working directory untouched — you're spawning siblings, not moving in yourself.

If this repo's `.config/wt.toml` hooks aren't yet approved, `wt` refuses to run non-interactively. That approval is the user's call, not yours — follow the `worktrunk` skill's "Hook Approvals in Non-Interactive Sessions" guidance rather than reaching for `--yes` on your own judgment.

Once the user has given a standing answer for _this repo_ (e.g. "just use `--yes`"), apply it on subsequent `wt switch --create` calls in the same repo without re-asking — the trust decision was already made, re-litigating it every ticket is noise. Ask again only for a repo they haven't decided on yet, or if `.config/wt.toml`'s hooks have changed since they decided.

### 3. Claim the hook's pane — don't spawn a new one

If this repo's `post-start` hook wires muxy in (the `worktrunk` skill's post-start pattern), it already opens and focuses a pane at the new worktree's path as a side effect of step 2. That pane is your target.

**Do not call `muxy new-tab --worktree <branch>`.** By the time you'd call it, the hook has usually already made that worktree the active one, so the new-tab targeting rule ("stays backgrounded unless the target is already active") no longer holds — the tab lands in the _current_, visible tab strip instead. That's a stray tab in the wrong place, not a background session.

Find the pane the hook already made instead:

```sh
muxy list-panes | awk -F'\t' -v p="<worktree-path>" '$3==p{print $1}'
```

Confirm it's fresh — empty prompt, no prior output — with `muxy read-screen --pane <id>` before using it.

### 4. Name the pane, launch Claude, then stage `/implement` — don't submit it

```sh
muxy rename-pane --pane <id> "Issue #<n>: <short title>"
muxy send --pane <id> "claude"
muxy send-keys --pane <id> Enter
```

Confirm Claude's own prompt has replaced the shell — poll `muxy read-screen --pane <id>` until it shows Claude's UI, not a bare shell prompt. Staging into a shell that hasn't handed off yet lands the text in the wrong place.

Once it's ready, stage the command — send the text, and stop there:

```sh
muxy send --pane <id> "/implement <ticket URL>"
```

**Do not follow this with `muxy send-keys --pane <id> Enter`.** Submitting it defeats the point: the user, not you, decides when implementation actually starts. The command sits typed and waiting for them to press Enter.

### 5. Repeat per ticket, then verify no stray panes

After spawning every worktree, diff `muxy list-panes` against the pre-run baseline. Every new pane should map 1:1 to a worktree path you targeted. A pane with a generic title at a path you didn't specifically create is a stray from step 3 done wrong — after confirming via `read-screen` that it's empty and not one of the sessions you just started, close it with `muxy close-pane --pane <id>`.

Tell the user which panes are staged and waiting — they're the ones who press Enter.

## Completion criterion

One pane per ticket, each named for its issue, each running Claude Code with `/implement <ticket URL>` staged in its input line but **not submitted** — and the pane/tab count matches exactly: no extras, nothing missing.
