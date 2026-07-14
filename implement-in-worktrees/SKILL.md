---
name: implement-in-worktrees
description: Open one git worktree per ticket and start an independent Claude Code session in each via cmux, so several frontier tickets get implemented in parallel. Use when the user wants to work multiple tickets/issues at once, mentions "open worktrees for these", "start sessions in each", "work these in parallel", or wants to fan /implement out across the frontier.
---

# Implement in Worktrees

Fan `/implement` out across several tickets at once: one worktree, one Claude session, per ticket. The current session stays put and never enters any of them. Each new session starts with `/implement` staged, not submitted — the user decides when it actually begins.

Requires `wt` (see the `worktrunk` skill) and a running cmux (see the `cmux` and `cmux-workspace` skills) — read those for the mechanics this skill assumes rather than re-deriving them here.

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

If this repo's `post-start` hook still wires in a different multiplexer (e.g. a leftover muxy integration from before this skill's cmux port), ignore whatever pane or tab it opens — this skill doesn't use it and ends up spawning its own workspace in the next step regardless. That hook is a separate cleanup, not something to fix here.

### 3. Spawn a backgrounded workspace directly — don't wait on a hook

Unlike a hook-populated pane, `cmux new-workspace` takes an explicit `--focus` flag, so you can create the worktree's workspace yourself and know it lands backgrounded — no racing a hook, no "claim the existing pane" indirection:

```sh
cmux new-workspace --cwd <worktree-path> --name "Issue #<n>: <short title>" --focus false
```

This also names the workspace up front, so there's no separate rename step later. The workspace comes with one initial terminal surface already open — resolve its ref before continuing:

```sh
cmux list-pane-surfaces --workspace <workspace-id>
```

**Do not create the surface as `--type agent-session --provider claude`.** It looks like the native way to launch Claude, but agent-session surfaces aren't plain terminals: `send`, `send-key`, and `read-screen` all fail against them with `Surface is not a terminal`, and the only prompt-delivery path found (`cmux rpc workspace.prompt_submit`) submits immediately with no stage-only mode. That breaks the whole point of this skill — staging without submitting. Launch Claude in the plain terminal surface instead, the same way the next step does.

### 4. Launch Claude, then stage `/implement` — don't submit it

```sh
cmux send --surface <surface-id> "claude"
cmux send-key --surface <surface-id> Enter
```

Confirm Claude's own prompt has replaced the shell — poll `cmux read-screen --surface <surface-id>` until it shows Claude's UI, not a bare shell prompt. Staging into a shell that hasn't handed off yet lands the text in the wrong place.

Once it's ready, stage the command — send the text, and stop there:

```sh
cmux send --surface <surface-id> "/implement <ticket URL>"
```

**Do not follow this with `cmux send-key --surface <surface-id> Enter`.** Submitting it defeats the point: the user, not you, decides when implementation actually starts. The command sits typed and waiting for them to press Enter.

### 5. Repeat per ticket, then verify no stray workspaces

After spawning every worktree, diff `cmux workspace list --json` against the pre-run baseline. Every new workspace should map 1:1 to a worktree path you targeted, via its `current_directory` field. A workspace at a path you didn't specifically create is a stray — after confirming via `read-screen` that it's empty and not one of the sessions you just started, close it with `cmux close-workspace --workspace <id>`.

Tell the user which workspaces are staged and waiting — they're the ones who press Enter.

## Completion criterion

One workspace per ticket, each named for its issue, each running Claude Code with `/implement <ticket URL>` staged in its input line but **not submitted** — and the workspace count matches exactly: no extras, nothing missing.
