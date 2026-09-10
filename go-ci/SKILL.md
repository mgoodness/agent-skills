---
name: go-ci
description: >
  Expert skill for wiring GitHub Actions CI/CD around a Go application: a GoReleaser release
  workflow, release-please for Conventional-Commits versioning and tagging, Dependabot with
  auto-merge, the CI workflow that gates them, and the security hardening (zizmor, action
  pinning, least-privilege GitHub App tokens) and branch-protection wiring that makes the
  release-tag handoff and auto-merge actually fire instead of silently no-oping. Use whenever
  setting up or reviewing a GitHub Actions workflow, `.goreleaser.yaml`, release-please config,
  `dependabot.yml`, or branch-protection rules for a Go repo. Defers Go semver/API-compatibility
  rules to the `go-release` skill; this skill owns the Actions/YAML wiring around it.
---

# Go CI: Wiring GitHub Actions Around a Go Application

This pipeline is a dependency graph, not a checklist: each piece below produces something the next one needs, and skipping the order reproduces exactly the failures this skill exists to prevent — a tag that never triggers a build, an App token that can't push, an auto-merge that fires with nothing to wait for.

## When to Activate

- Bootstrapping GoReleaser, release-please, or Dependabot for a Go repo
- Writing or reviewing a `ci.yml` (or equivalent required-check workflow)
- A GitHub Actions workflow needs to push a tag, cut a release, or auto-merge a PR
- `zizmor`, `actionlint`, or a similar Actions linter flags a workflow you wrote
- Deciding whether a repo needs a toolchain manager (`mise`, `asdf`) alongside `go.mod`

## The pipeline, in dependency order

### 1. GoReleaser config

`go-release`'s Version Injection / Distribution Checklist sections own the build recipe (ldflags, `CGO_ENABLED=0`, `-trimpath`, checksums) — read those first. The only release-please-specific trim on top:

```yaml
changelog:
  disable: true # release-please already writes the changelog
release:
  mode: keep-existing # don't clobber the release body release-please wrote
```

Verify with `goreleaser check`, then a local `goreleaser release --snapshot --clean`, before wiring any workflow around it.

### 2. release-please config — the component-tag trap

`release-please-config.json` (`release-type: "go"`) and `.release-please-manifest.json` (`{".": "0.0.0"}`) are the whole config for a single-package repo. Leave `package-name` **unset** in the package entry:

```json
"packages": { ".": {} }
```

Setting `package-name` gives release-please a _component_, and `includeComponentInTag` defaults to `true` — the tag becomes `<package-name>-v1.0.0`, not `v1.0.0`. A GoReleaser workflow triggering on `tags: ["v*"]` then never fires against that push, silently: no error, no failed run, just nothing happening. If you want a component name for another reason, pair it with `"include-component-in-tag": false`.

### 3. release-please workflow — loop prevention

GitHub's **loop prevention**: a push or PR authored by the default `GITHUB_TOKEN` cannot trigger further workflow runs. release-please pushes the release tag from step 2; if that push is `GITHUB_TOKEN`-authored, the GoReleaser workflow watching for `v*` never fires — again silently.

Fix: mint a short-lived token from a GitHub App installation via `actions/create-github-app-token`, scoped down to exactly what release-please needs rather than the App's full installation grant:

```yaml
- uses: actions/create-github-app-token@<sha> # vX.Y.Z
  id: app-token
  with:
    client-id: ${{ vars.RELEASE_PLEASE_APP_CLIENT_ID }}
    private-key: ${{ secrets.RELEASE_PLEASE_APP_PRIVATE_KEY }}
    permission-contents: write
    permission-pull-requests: write
- uses: googleapis/release-please-action@<sha> # vX.Y.Z
  with:
    token: ${{ steps.app-token.outputs.token }}
```

The downstream GoReleaser workflow does **not** need this token — it only uploads binaries to a release the tag-authoring workflow already created, so the default `GITHUB_TOKEN` is enough there.

Creating the App, and pulling its Client ID and private key into the repo's variable/secret, is one-time work only a human can click through — script it with the `wizard` skill rather than writing it as prose steps to follow by hand.

### 4. CI workflow — the required check

This job is what branch protection and auto-merge both key off in step 6, so name it deliberately: branch protection's required-check `context` matches the job's `name:` field, not its `jobs.<id>` key.

Run `go-release`'s release checklist (`go mod tidy` drift check, build, vet, `test -race`, `govulncheck`, `gorelease`) plus `gofmt` — the one step that checklist omits:

- `gorelease` needs a **pristine checkout** — it refuses to run against any uncommitted change, which only bites a local dry run, never CI's own fresh checkout.
- It resolves the base version against the **module proxy over the network**, not local git tags, so a shallow `fetch-depth` checkout is fine.

Trigger this workflow on plain `pull_request` (the default read-only token) rather than `pull_request_target` — this is the workflow that actually checks out and runs PR-authored code, including forks', so it's the one place the safer, read-only event matters.

`goreleaser.yml` re-runs this same checklist _minus_ `gorelease` at tag time, as a second, independent gate: by then release-please's own merged PR has already passed it once, and `gorelease`'s pre-v1 breaking-change findings aren't something that should block a release.

### 5. Dependabot config

`gomod` + `github-actions` ecosystems, weekly schedule, `cooldown.default-days >= 7` — anything shorter trips zizmor's `dependabot-cooldown` audit.

### 6. Dependabot auto-merge

Gate on `dependabot/fetch-metadata`'s `update-type` output: auto-merge `semver-minor`/`semver-patch`, leave `semver-major` for manual review — every major bump is a real compatibility question, not busywork.

Two settings this depends on, invisible in any file in the repo:

- "Allow auto-merge" enabled on the repo itself.
- Branch protection on the default branch requiring the CI job's check (step 4). Without it, "enable auto-merge" just merges on PR open with zero verification — the required check is load-bearing, not incidental.

This workflow runs on `pull_request_target`, for the write-scoped token Dependabot's own `pull_request` event never gets (regardless of the workflow's `permissions:` block). That's the one place `pull_request_target` is safe to use here, and only because the workflow never checks out PR code and only ever calls `gh pr merge` against the PR's URL — see Security hardening below for how to justify it to a linter.

Don't generalize this into a blanket "auto-merge for anyone with write access" workflow by default. Auto-merging mechanical, high-volume dependency bumps is worth automating; auto-merging human-authored PRs is a per-repo call each maintainer should make explicitly (e.g. enabling per-PR by hand), not something to generate speculatively alongside the Dependabot piece.

## Security hardening (what zizmor will flag)

- **Pin every third-party action to a commit SHA with a trailing `# vX.Y.Z` comment**, never a bare tag. The version comment is what lets Dependabot's `github-actions` ecosystem keep bumping it — a bare SHA with no comment is a pin nothing can track.
- **`persist-credentials: false`** on any `actions/checkout` step that doesn't need the default token to push afterward, and always on the step that checks out untrusted PR code.
- **Bot-identity checks**: use `github.event.pull_request.user.login`, never `github.actor` — `actor` is spoofable (zizmor's `bot-conditions` audit).
- **`pull_request_target` is flagged on principle** (`dangerous-triggers`): the usual failure mode is checking out and running the PR's own code under a write-scoped token. Suppress only when you can name, in the same comment, why _this_ workflow is immune — no checkout of PR code, and no PR-derived value interpolated directly into a `run:` script (only ever passed through `env:`):
  ```yaml
  on: pull_request_target # zizmor: ignore[dangerous-triggers] no checkout of PR code; PR-derived values only flow through env:, never interpolated into run: scripts
  ```
- **Scope minted App tokens down** with `permission-*` inputs (step 3); never let a job's token inherit an App's full installation grant when it only needs to push a tag and open a PR.

## `go.mod`'s `go` directive: apps aren't libraries

`go-release`'s "set the `go` directive to the oldest version you need" is a _library_ rule — it's the floor every dependent inherits. An application binary has no dependents to floor, so there's no compatibility reason to hold it below your actual toolchain. Concretely: `go mod init`'s default is often stale by the time CI actually gets wired up — an unreviewed leftover, not a decision — so bump it to match the toolchain you build with rather than leaving it low "to be safe."

## Skip a toolchain manager until it earns its keep

`go.mod`'s `go` directive plus Go's own `GOTOOLCHAIN=auto` (default since 1.21) already pins the compiler version and auto-fetches it when the local `go` is older; `actions/setup-go`'s `go-version-file: go.mod` makes CI read that same number. A `mise.toml` (or `asdf`) pinning the same version a second time is a second source of truth for a fact `go.mod` already owns — skip it for a single-language repo. It earns its place once there's a _second_ tool with no native version pin anywhere (a locally-installed `goreleaser` while CI runs `version: latest`, say) or the repo goes polyglot.

## Common Mistakes

- **Setting `package-name` in `release-please-config.json`** for a single-package repo — silently mismatches the tag GoReleaser is watching for.
- **Trusting `github.actor` for a bot-identity `if:` check** — spoofable; use the event payload's `user.login` instead.
- **Using `pull_request_target` on the workflow that also checks out PR code** — that's the exact combination the trigger is dangerous for.
- **Minting an App token with no `permission-*` inputs** — it inherits the App's entire installation grant instead of the one job's actual needs.
- **Expecting `gorelease` to read local git tags** — it needs network access to the module proxy, and refuses to run against an uncommitted change.
- **Naming the required-check branch-protection rule after a job's id instead of its `name:`** — GitHub matches on `name:`.
- **Adding a `mise.toml` that just re-pins what `go.mod` already pins** — redundant source of truth for a single-language repo.
- **Building a general-purpose "auto-merge for write access" workflow alongside the Dependabot one** — auto-merging human PRs is a per-repo, per-PR decision, not a default to ship.
- **Not rebasing a Dependabot PR stuck on stale CI** — merging a fix to the base branch doesn't retroactively re-run an already-open PR's checks; comment `@dependabot rebase` to make it pick up the new base and re-run.
