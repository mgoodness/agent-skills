---
name: start-ticket
description: Create or adopt a Jira ticket, assign it, add to active sprint, mark In Progress, and stage a worktree session via /spin-up-worktrees. Use when the user wants to start a ticket and branch, mentions "spin up a ticket", "create a ticket and branch", or asks to go from idea to ready-to-code. When work is done, follow up with /shipit.
---

# Start Ticket

End-to-end flow: ticket → In Progress → worktree → ready to code.

## Workflow

### 1. Existing ticket or new?

If the user invoked the skill with a ticket key (e.g. `/start-ticket OZZI-2225`), use it — treat it as an existing ticket.

Otherwise ask:

> Do you have an existing Jira ticket, or should I create a new one? (provide the key, or say "new")

- **Existing key provided** → step 3 adopts it (sprint + assignment still normalized).
- **"new"** → step 3 creates it.

### 2. Gather context (parallel)

Run all of these at once:

- `jira me` — current user (for assignment in step 3)
- `git remote -v` — confirm repo remote

### 3. Find the active sprint + project key

```sh
jira sprint list --board 2093 --state active --plain --columns id,name
```

Take the first (current) active sprint → note its `id`.

Get the project key from the configured default or ask the user. If unknown, run:

```sh
jira project list --plain --columns key,name
```

and pick the relevant project.

### 4. Create or adopt the ticket

**If new**, create it:

```sh
jira issue create \
  --type Task \
  --summary "<user-provided title>" \
  --no-input
```

Note the returned key (e.g. `OZZI-2225`).

**Either path** — normalize sprint membership and assignment:

```sh
jira sprint add <sprint-id> <TICKET-KEY>
jira issue assign <TICKET-KEY> me
```

These are idempotent — safe to run even if already set.

### 5. Transition to In Progress

```sh
jira issue move <TICKET-KEY> "In Progress"
```

### 6. Spin up the worktree session

Invoke `/spin-up-worktrees` with the ticket key:

```
/spin-up-worktrees <TICKET-KEY>
```

Branch naming, worktree placement, and staging `/implement` are `spin-up-worktrees`' concern — see that skill for the mechanics rather than re-deriving them here. This session stays put on the current branch; report the new workspace as staged and waiting per that skill's completion criterion. The ticket gets implemented there, followed by `/shipit` to commit and open the PR.

## CLI reference

| Action              | Command                                                    |
| ------------------- | ---------------------------------------------------------- |
| Current user        | `jira me`                                                  |
| List active sprints | `jira sprint list --board 2093 --state active`             |
| Create issue        | `jira issue create --type Task --summary "..." --no-input` |
| Add to sprint       | `jira sprint add <sprint-id> <TICKET-KEY>`                 |
| Transition issue    | `jira issue move <TICKET-KEY> "In Progress"`               |
| List projects       | `jira project list --plain --columns key,name`             |
| Spin up worktree    | `/spin-up-worktrees <TICKET-KEY>` (skill)                  |

Board ID `2093` is the configured default (Cloud Platform Sprint Board).

## Edge cases

- **User specifies a different project:** pass `--project <KEY>` to `jira issue create`.
- **`jira issue move` status name mismatch:** run `jira issue move <KEY>` without a status arg to list available transitions interactively.
