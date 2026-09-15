# agent-skills

Michael Goodness's [Agent Skills](https://agentskills.io) — reusable instruction sets for AI coding agents, developed and used daily in [dotfiles](https://github.com/mgoodness/dotfiles).

## Skills

| Skill | Description |
| --- | --- |
| [`agent-context-branch`](agent-context-branch/SKILL.md) | Preserve repo-wide agent context docs (AGENTS.md, CLAUDE.md, CONTEXT.md, docs/adr, or similar design/domain docs) on a dedicated branch off the default branch, separate from feature-branch code. |
| [`github-app-token`](github-app-token/SKILL.md) | Minting and using a GitHub App installation token in place of the default `GITHUB_TOKEN`: loop prevention, the workflow-run approval gate, scoping, and re-establishing bot identity downstream. |
| [`go-ci`](go-ci/SKILL.md) | The Go-specific band of a GitHub Actions release pipeline: the GoReleaser build recipe, the Go-specific checks a required CI job runs, scoping golangci-lint, and go.mod/toolchain-manager judgment calls. |
| [`release-please`](release-please/SKILL.md) | Wiring release-please, the GitHub Action that cuts versioned releases automatically from Conventional Commits: the component-tag trap, the loop-prevention fix, and the draft/force-tag-creation pairing. |
| [`repo-hardening`](repo-hardening/SKILL.md) | Hardening a GitHub repo so a required check and automated merges are actually safe: a repository ruleset, merge settings, Dependabot with auto-merge, immutable releases, and Actions security practices. |
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
