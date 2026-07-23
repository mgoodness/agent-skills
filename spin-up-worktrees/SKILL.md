---
name: spin-up-worktrees
description: Open one or more git worktrees and start an independent Claude Code session in each via herdr — a single task or several in parallel, with or without an associated ticket. Defaults to staging /implement <task> unstarted in each session; say "just open the session(s)" or "skip /implement" to leave them bare instead. Use when the user wants to work on a ticket/issue/task in its own worktree, mentions "open a worktree (or worktrees) for this", "start a session for", "work these in parallel", or wants to fan /implement out across several.
---

# Spin Up Worktrees

Spin up one or more worktrees, each with its own Claude session. The current session stays put and never enters any of them. Works the same for one worktree or several in parallel, and whether the work is a tracker ticket, a plain description, or nothing more specific than a branch name.

By default, each new session starts with `/implement <task>` staged, not submitted — the user decides when it actually begins. `<task>` is the ticket URL when one exists, otherwise a short description of the work. If the user instead asks to just open the session(s) (e.g. "skip /implement", "leave it bare", "just start Claude"), stop once Claude's prompt is ready and don't stage anything — see step 4.

Requires `wt` (see the `worktrunk` skill) and a running herdr server. Check with `herdr status` — `server: running` means it's up. If it's not, start it yourself rather than asking the user: `herdr server` runs headless (it's a documented mode — `herdr --help` lists it under "Advanced commands: Run as headless server"), and none of this skill's operations (`workspace create`, `worktree open`, `pane send-text`, etc.) need an attached TUI client, only the server. Background it — `herdr server >/dev/null 2>&1 &` or your tool's equivalent — then re-check `herdr status` before continuing.

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

If this repo's `post-start` hook opens its own pane or workspace — whether it's wired to herdr directly, a lingering cmux integration, or something older — ignore it. This skill spawns its own workspace in the next step regardless, and step 5 cleans up whatever the hook left behind.

### 3. Open the worktree as a nested herdr workspace — don't wait on a hook

herdr's worktree commands are repo-aware: opening a worktree checkout registers it against the repo it belongs to — resolved from the worktree's own git metadata, not from anything you tell it — and nests it under any other open workspace on that same repo automatically. No `--workspace`/`--cwd` parent hint needed (verified), so there's no manual grouping step and no need to look up which workspace you're calling from first.

Per worktree:

```sh
herdr worktree open --path <worktree-path> --label "<short title>" --no-focus
```

`--no-focus` is what keeps this backgrounded — without it the new workspace steals focus. herdr's CLI already returns JSON by default (no `--json` needed), and the response is the only place you need to look — no separate pane lookup like cmux required:

```json
{ "result": { "workspace": { "workspace_id": "w3" }, "root_pane": { "pane_id": "w3:p1" } } }
```

Pull `<workspace-id>` and `<pane-id>` from that response and carry them into the next step.

Use the ticket's number and title for `<short title>` if one exists (e.g. `Issue #<n>: <short title>`); otherwise a short description of the task, or the branch name if there's nothing more descriptive to give it. This also names the workspace up front, so there's no separate rename step later.

### 4. Launch Claude as a tracked agent, then stage `/implement` — unless told to leave it bare

```sh
herdr agent start <name> --kind claude --pane <pane-id>
```

`<name>` is any identifier you want to refer back to this agent by later (the branch name is a reasonable choice). This single call launches Claude itself and blocks until it's ready for input — there's no manual "type claude, press Enter, then poll the screen for the handoff" dance the way cmux needed; readiness detection is herdr's job here, not yours.

Default timeout is 30s (`--timeout` in ms to extend, max 300000). If it times out, don't assume the session is dead — check `herdr pane read <pane-id> --source visible` before concluding anything failed. A slow first launch can outrun the default timeout without anything actually being wrong.

**Bare-session mode** (user asked to skip `/implement`): stop here. The session is up and ready; move to the next worktree, if there is one.

**Default mode**: once `agent start` returns successfully, stage the command — send the text, and stop there:

```sh
herdr pane send-text <pane-id> "/implement <task>"
```

`<task>` is the ticket URL when one exists, otherwise a short description of what to do — plain text is fine, it doesn't need to resolve to anything. **Do not use `herdr pane run` or anything else that submits it.** Submitting it defeats the point: the user, not you, decides when implementation actually starts. `send-text` types without submitting — verified behavior, not an assumption — so the command sits typed and waiting for them to press Enter.

### 5. Repeat for each additional worktree, then verify no stray workspaces

If spinning up more than one, repeat steps 1–4 per worktree. Either way, check `herdr workspace list` for each worktree path you targeted: it should map to exactly one workspace, and that workspace's `workspace_id` should be the one your own `herdr worktree open` call returned in step 3 — no pre-run snapshot needed, since you already know the id you're checking for. A workspace at your target path whose id you didn't mint yourself is a stray — this repo's `post-start` hook can spawn one on its own, not just a leftover multiplexer, so treat any hook-spawned workspace at your path as disposable regardless of what it's wired to. Id identity is the signal here, not screen content — don't gate the close on `pane read`, since a freshly opened pane can be slow to report on its first read whether or not it's a stray.

**Close strays with `herdr workspace close <id>` — never `herdr worktree remove`.** `remove` deletes the underlying git worktree checkout from disk (verified); `close` only tears down the herdr-side presentation and leaves the checkout untouched. This skill only ever creates worktrees — deleting one is `cleanup-branch`'s job, not a byproduct of tidying up a duplicate pane here.

In default mode, tell the user which workspaces are staged and waiting — they're the ones who press Enter. In bare-session mode, just confirm each session is live and ready.

## Completion criterion

One workspace per worktree, each named for its task, each running Claude Code as a tracked herdr agent — with `/implement <task>` staged in its input line but **not submitted** in default mode, or simply sitting at a ready prompt in bare-session mode — and the workspace count matches exactly: no extras, nothing missing.
