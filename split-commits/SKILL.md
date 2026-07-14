---
name: split-commits
description: Split a dirty working tree into well-scoped Conventional Commits, staged and proposed for approval before anything is committed. Use when the user wants to split or organize pending changes into commits, turn a messy working tree into clean commits, or asks for a commit plan before committing.
---

# Split Commits

## Default workflow

1. Inspect the repo state with:
   - `git --no-pager status --short`
   - `git --no-pager diff --stat`
   - `git --no-pager diff --cached --stat`
   - `git --no-pager log --oneline -10` — to learn established commit message style and scopes
   - targeted `git --no-pager diff -- <paths>` for changed files
2. Group changes by intent, not just by directory.
   - Prefer one commit per behavior change, feature, fix, cleanup, or tooling update.
   - Split mixed files by hunk when necessary.
3. Propose the commit plan and wait for approval before making any changes — see **Safety rules**.
4. Draft Conventional Commit subjects using scopes discovered from the log (step 1) and the repo structure.
   - Prefer `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `build`, or `ci`.
   - Add a scope when it improves clarity.
   - Keep subjects short, imperative, and review-friendly.
5. If the user approves execution:
   - stage files or hunks carefully (see **Execution details** for hunk-level staging)
   - commit in the agreed order
   - keep unrelated unstaged changes out of each commit
   - re-check status after each commit when needed
6. Finish by reporting per **Communication pattern** below.

## Splitting rules

### Good reasons to split

- Different user-visible behaviors changed
- Separate tools or subsystems changed independently
- One file contains unrelated hunks
- Personal editor settings are mixed with project changes
- Mechanical cleanup is mixed with functional edits

### Good reasons to combine

- Multiple files implement one coherent change
- Renames, follow-up callsite fixes, and config needed for the same feature
- Separating the changes would leave one of the commits broken or nonsensical on its own

## Conventional Commit guidance

Use the smallest accurate type.

- `feat(scope): ...` for new behavior or tooling additions
- `fix(scope): ...` for bug fixes or behavior corrections
- `chore(scope): ...` for maintenance, local settings, cleanup, or non-user-facing config
- `refactor(scope): ...` for structure changes without behavior change
- `docs(scope): ...` for documentation-only updates

Derive scopes from `git log --oneline` first, then from repo structure. Do not invent scopes that conflict with established conventions. Examples:

- `feat(zed): switch default editor to Zed Preview`
- `fix(fish): export Tide command visibility variables`
- `chore(git): remove gitconfig modelines from managed config files`

## Execution details

When the user asks you to run the split:

- Use `git add <file>` for whole-file commits.
- **Never use `git add -p` — it is interactive and will hang.** For hunk-level staging, use the patch-file approach:
  1. `git --no-pager diff -- <file>` to read the full diff.
  2. Write a `.patch` file containing only the hunks that belong to this commit.
  3. `git apply --cached <patch-file>` to stage exactly those hunks.
  4. Delete the temporary patch file after staging.
- If pre-commit hooks modify files during commit, inspect the new staged/unstaged state and fold formatter-only follow-up changes into the commit they belong to.
- Do not accidentally commit unrelated local settings or user-private files unless the user included them.
- If a file must be split across commits, explain which hunks belong to which commit before staging.

## Communication pattern

When proposing a plan, include:

- the ordered commit list
- which files or hunks belong to each commit
- a short reason for each grouping

When execution finishes, include:

- commit hashes and subjects (from `git log --oneline`)
- any hook or formatter behavior encountered during commit
- any skipped or remaining uncommitted files
- whether the working tree is clean

## Safety rules

- Never rewrite or discard user changes without asking.
- Never commit without first presenting the plan and receiving explicit approval — even if the user's request implies execution (e.g. "split and commit all but X").
- If the working tree includes ambiguous mixed hunks, say so and propose the safest split.
- If the requested number of commits would force unrelated changes together, say that clearly and suggest a better split.
