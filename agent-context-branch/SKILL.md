---
name: agent-context-branch
description: Preserve repo-wide agent context docs (AGENTS.md, CLAUDE.md, CONTEXT.md, docs/adr, or similar design/domain docs) on a dedicated branch off the default branch, separate from feature-branch code, so they can be pulled into other worktrees. Use when the user wants to preserve or save context docs across worktrees, keep design docs out of feature-branch or default-branch commits, or sync a doc correction back onto that branch.
---

# Agent Context Branch

A context branch holds a repo's agent-facing docs off the default branch, in one of two patterns:

- **Context-only docs** — `CONTEXT.md`, `docs/adr/`, or similar — tracked nowhere else. On every other branch they stay untracked in the working tree: present for reference, absent from that branch's commits.
- **Overlay files** — `CLAUDE.md`, `AGENTS.md` — tracked normally on the default branch too. The context branch's copy is an **overlay**: the same shared baseline as the default branch, plus branch-only sections layered on top (personal-machine overrides, local-only conventions, anything not safe or not ready to upstream). Every edit to an overlay file — even one that looks general and unrelated to the branch-only sections — is made on the context branch, never on the default or a feature branch. Promoting shared content into the default branch is a separate, deliberate action (a normal PR) outside this workflow, not a side effect of any step here.

The context branch is the single source of truth for both patterns; every worktree pulls from it.

## Workflow

### 1. Identify the docs and the branch name

Confirm which files are context docs, and which pattern each follows: diff the file between the context branch and `origin/<default-branch>`. Missing from the default branch entirely -> context-only. Present on both -> overlay, and the diff you see is the overlay content. Commonly `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md`, `docs/`, but defer to whatever the repo actually has.

Check whether a context branch already exists before creating a second one:

```sh
git branch -a | grep -i context
```

Default name: `agent-context`.

### 2. Branch off the default branch, not the current branch

Local `main` can be stale, so fetch and compare before branching:

```sh
git fetch origin <default-branch> --quiet
git rev-parse HEAD <default-branch> origin/<default-branch>
```

Create the context branch from `origin/<default-branch>` (or check out the existing one) — never from the current feature branch, or its code commits ride along:

```sh
git checkout -b agent-context origin/<default-branch>
```

Context-only docs carry over automatically as untracked (`??`); overlay files carry over as the tracked default-branch baseline, ready for the overlay sections to be added on top.

### 3. Commit and push

```sh
git add AGENTS.md CLAUDE.md CONTEXT.md docs/
git commit -m "..."
```

**Push with an explicit refspec — never a bare `git push -u origin <branch>` here.** `checkout -b <branch> origin/<default-branch>` sets the new branch's upstream to the default branch, not to itself. A bare push then targets that upstream, which can land the docs commit on `main`. Branch protection may reject it, but don't rely on that:

```sh
git push origin agent-context:agent-context
```

### 4. Return to the feature branch

```sh
git checkout <feature-branch>
```

Context-only docs disappear from the working tree here — expected, not an error: they're tracked-only on the context branch now, and the feature branch never tracked them. Overlay files revert to the feature branch's own tracked baseline, losing any overlay sections — also expected.

### 5. Restore the docs for local reference

Context-only docs come back untracked, with nothing staged:

```sh
git checkout agent-context -- CONTEXT.md docs/
git restore --staged CONTEXT.md docs/
```

Overlay files come back as a tracked-but-uncommitted modification — there's no staged state to undo, since the file was already tracked at the feature branch's baseline:

```sh
git checkout agent-context -- CLAUDE.md AGENTS.md
```

Leave this diff uncommitted indefinitely. It is the local view of the overlay, not a change queued for the feature branch — `git status` should keep showing it as `M`, never staged, never committed here.

### 6. Propagate a correction back

A context doc sometimes needs fixing mid-feature-work (a stale claim caught while implementing, a general note worth adding to an overlay file). The context branch must get the same fix or it silently goes stale — and the fix belongs there even when it would look like a harmless, general edit if committed straight to the feature branch. Stash before switching, since the context-only docs are untracked and the overlay files are tracked-modified, both of which would conflict with the context branch's own copies at the same paths:

```sh
git stash push -u -m "context docs" AGENTS.md CLAUDE.md CONTEXT.md docs/
git checkout agent-context
# reapply the same edit here — for an overlay file, edit only the intended
# hunk; don't let a whole-file copy from the feature branch's stash
# clobber the branch-only overlay sections that aren't in that stash
git add <changed-file>
git commit -m "..."
git push origin agent-context:agent-context
git checkout <feature-branch>
git stash pop
```

## Completion criterion

`origin/agent-context` carries the commit. On the feature branch: context-only docs are restored-but-untracked, overlay files are restored-but-uncommitted (`M`), and `git status` is otherwise clean against the feature branch's own commits.
