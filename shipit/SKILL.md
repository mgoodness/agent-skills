---
name: shipit
description: Commit staged changes via /split-commits and open a pull request. Use when work is done and the user wants to ship — after /start-ticket, or any time there's a branch ready to PR. Optionally accepts a Jira ticket key to include in the PR title and body.
---

# Shipit

Commit the work and open a pull request.

## Workflow

### 1. Get the ticket key

If the user passed a key (e.g. `/shipit OZZI-2225`), use it.
If the current branch name starts with a Jira key pattern (`[A-Z]+-\d+`), extract it.
Otherwise ask: "What's the Jira ticket key? (or skip)"

### 2. Commit staged changes

Invoke `/split-commits` — it handles grouping by intent, conventional commit subjects, hunk-level staging, and user approval before touching anything.

If there are no staged or modified files, skip and note it; use `git commit --allow-empty -m "chore(<scope>): init [<TICKET-KEY>]"` as a placeholder if GitHub requires a commit ahead of base.

If a ticket key is known, include it as a suffix in each commit subject: `[<TICKET-KEY>]`.

### 3. Create the pull request

After split-commits finishes, push and open the PR:

```sh
gh pr create \
  --title "[<TICKET-KEY>] <summary>" \
  --body "..." \
  --base main \
  --head <branch>
```

PR body template:

```md
## Summary

<one-liner>

Resolves [<TICKET-KEY>](https://<your-jira-site>/browse/<TICKET-KEY>)

## Changes

- `path/to/file` — what it does
```

Omit the `Resolves` line if no ticket key is available.

## Edge cases

- **No ticket key:** omit `[<TICKET-KEY>]` from commit subjects and PR title; skip `Resolves` line in body.
- **Branch already has commits:** skip `/split-commits` if the user confirms there's nothing new to stage; just push and open the PR.
- **Draft PR:** if the user says "draft", add `--draft` to `gh pr create`.
