# agent-skills

Michael Goodness's [Agent Skills](https://agentskills.io) — reusable instruction sets for AI coding agents, developed and used daily in [dotfiles](https://github.com/mgoodness/dotfiles).

## Skills

| Skill | Description |
| --- | --- |
| [`agent-context-branch`](agent-context-branch/SKILL.md) | Preserve repo-wide agent context docs (AGENTS.md, CLAUDE.md, CONTEXT.md, docs/adr, or similar design/domain docs) on a dedicated branch off the default branch, separate from feature-branch code. |
| [`cleanup-branch`](cleanup-branch/SKILL.md) | Remove a merged branch's worktree, remote ref, and local tracking branch, tearing down its herdr workspace first. |
| [`github-app-token`](github-app-token/SKILL.md) | Mint a GitHub App installation token to escape the default `GITHUB_TOKEN`'s loop-trap and gate-trap, scope it, and re-establish bot identity downstream. |
| [`go-ci`](go-ci/SKILL.md) | GoReleaser build recipe, Go checks for a required CI job, golangci-lint scoping, and go.mod/toolchain judgment calls. |
| [`release-please`](release-please/SKILL.md) | Wire release-please to cut releases from Conventional Commits: the tag-trap, loop-trap, and draft-trap. |
| [`repo-hardening`](repo-hardening/SKILL.md) | Rulesets, merge settings, Dependabot with auto-merge, immutable releases, and Actions security practices. |
| [`spin-up-worktrees`](spin-up-worktrees/SKILL.md) | Open one or more git worktrees, each in its own nested herdr workspace with a plain shell. |
| [`split-commits`](split-commits/SKILL.md) | Split a dirty working tree into well-scoped Conventional Commits, staged and proposed for approval before anything is committed. |

## Install

With [`gh skill`](https://cli.github.com/) (GitHub CLI, preview):

```sh
gh skill install mgoodness/agent-skills --all --agent claude-code --scope user
```

With [`npx skills`](https://github.com/vercel-labs/skills):

```sh
npx skills add mgoodness/agent-skills --all
```

## License

[MIT](LICENSE)
