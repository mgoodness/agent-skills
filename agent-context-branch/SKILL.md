---
name: agent-context-branch
description: Preserve repo-wide agent context docs (AGENTS.md, CONTEXT.md, docs/adr, or similar design/domain docs) on a dedicated branch off the default branch, separate from feature-branch code, so they can be pulled into other worktrees. Use when the user wants to preserve or save context docs across worktrees, keep design docs out of feature-branch commits, or sync a doc correction back onto that branch.
---

# Agent Context Branch

A **context branch** holds a repo's agent-facing docs — `AGENTS.md`, `CONTEXT.md`, `docs/adr/`, or whatever similar files the repo uses — committed on their own, off the default branch, independent of any feature branch. On a feature branch these files stay untracked in the working tree: present for reference, absent from the branch's commits. The context branch is the single source of truth; every worktree pulls from it.

## Workflow

### 1. Identify the docs and the branch name

Confirm which untracked files are the context docs — `git status --porcelain` shows them as `??`. Commonly `AGENTS.md`, `CONTEXT.md`, `docs/`, but defer to whatever the repo actually has.

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

Untracked files carry over automatically; `git status` should still show them as `??`.

### 3. Commit and push

```sh
git add AGENTS.md CONTEXT.md docs/
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

The context docs disappear from the working tree here — expected, not an error. They're tracked-only on the context branch now, and the feature branch never tracked them.

### 5. Restore the docs for local reference, still untracked

```sh
git checkout agent-context -- AGENTS.md CONTEXT.md docs/
git restore --staged AGENTS.md CONTEXT.md docs/
```

This pulls the committed content back into the working tree without adding it to the feature branch's commits — `git status` should show `??` again.

### 6. Propagate a correction back

A context doc sometimes needs fixing mid-feature-work (a stale claim caught while implementing). The context branch must get the same fix or it silently goes stale. The docs are untracked on the feature branch, so switching branches directly conflicts with the context branch's tracked copies at the same paths — stash first:

```sh
git stash push -u -m "context docs" AGENTS.md CONTEXT.md docs/
git checkout agent-context
# reapply the same edit here
git add <changed-file>
git commit -m "..."
git push origin agent-context:agent-context
git checkout <feature-branch>
git stash pop
```

## Completion criterion

`origin/agent-context` carries the commit, the feature branch's working tree has the docs restored-but-untracked, and `git status` on the feature branch is clean against its own commits.
