---
name: go-ci
description: >
  The Go-specific band of a GitHub Actions release pipeline: GoReleaser build recipe, Go checks
  for a required CI job, golangci-lint scoping, and go.mod/toolchain judgment calls. Use when
  writing or reviewing a Go CI workflow, `.goreleaser.yaml`, or golangci-lint config. Defers
  semver rules to `go-release`, release cutting to `release-please`, and repo hardening to
  `repo-hardening`.
---

# Go CI: The Go-Specific Band of a Release Pipeline

This skill owns exactly what's Go-specific in a GitHub Actions release pipeline: the GoReleaser build recipe, the Go checks a required CI job runs, golangci-lint's scope, and `go.mod`/toolchain judgment calls. Everything else is language-agnostic and lives elsewhere: `release-please` owns cutting the release itself (config, loop prevention); `repo-hardening` owns the ruleset, Dependabot, auto-merge, and Actions security hardening around the required check this skill describes.

## GoReleaser config

`go-release`'s Version Injection / Distribution Checklist sections own the build recipe (ldflags, `CGO_ENABLED=0`, `-trimpath`, checksums) — read those first. The only release-please-specific trim on top:

```yaml
changelog:
  disable: true # release-please already writes the changelog
release:
  mode: keep-existing # don't clobber the release body release-please wrote
```

If the repo also has `repo-hardening`'s immutable releases enabled, release-please is holding its release as a draft specifically so artifacts can still be attached (see `release-please`'s draft-mode trap for its half of this pairing) — add the setting that finds and finishes that same draft instead of creating a second release for the same tag:

```yaml
release:
  use_existing_draft: true # attach to release-please's draft release, then GoReleaser publishes it
```

Verify with `goreleaser check`, then a local `goreleaser release --snapshot --clean`, before wiring any workflow around it.

## CI workflow — the Go-specific checks

This is the job `repo-hardening` calls "the required check": its own steps 1–2 cover naming it for the ruleset and wiring auto-merge around it. What belongs in the job itself, Go-side:

Run `go-release`'s release checklist (`go mod tidy` drift check, build, vet, `test -race`, `govulncheck`, `gorelease`) plus `gofmt` — the one step that checklist omits:

- `gorelease` needs a **pristine checkout** — it refuses to run against any uncommitted change, which only bites a local dry run, never CI's own fresh checkout.
- It resolves the base version against the **module proxy over the network**, not local git tags, so a shallow `fetch-depth` checkout is fine.

Trigger this workflow on plain `pull_request` (the default read-only token) rather than `pull_request_target` — this is the workflow that actually checks out and runs PR-authored code, including forks', so it's the one place the safer, read-only event matters.

`goreleaser.yml` re-runs this same checklist _minus_ `gorelease` at tag time, as a second, independent gate: by then release-please's own merged PR has already passed it once, and `gorelease`'s pre-v1 breaking-change findings aren't something that should block a release.

**golangci-lint** is worth adding to this job once `go vet` isn't catching enough, but scope it narrow: `linters.default: none` plus an explicit `enable:` list — typically just `errcheck` (unchecked error returns) and `staticcheck` (a strict superset of `go vet`) — rather than accepting golangci-lint's own default linter set, which is stylistic opinion (revive-style rules, import-order nags, `wsl`'s whitespace demands) that churns with every golangci-lint minor bump, not correctness. Pin `golangci-lint-action` itself to a SHA like every other action (see `repo-hardening`'s Security hardening section for the pinning discipline), but leave its `version:` input at `latest` rather than hand-pinning the golangci-lint binary too — one moving part to track instead of two, the same shape as `govulncheck`'s `@latest` and `goreleaser-action`'s `version: latest` elsewhere in this same pipeline. This also sidesteps a real gotcha: a `go.mod` `go` directive ahead of what an old pinned golangci-lint release supports fails opaquely (`export data version N is greater than maximum supported version M`), not with a clear version-mismatch error.

## `go.mod`'s `go` directive: apps aren't libraries

`go-release`'s "set the `go` directive to the oldest version you need" is a _library_ rule — it's the floor every dependent inherits. An application binary has no dependents to floor, so there's no compatibility reason to hold it below your actual toolchain. Concretely: `go mod init`'s default is often stale by the time CI actually gets wired up — an unreviewed leftover, not a decision — so bump it to match the toolchain you build with rather than leaving it low "to be safe."

## Skip a toolchain manager until it earns its keep

`go.mod`'s `go` directive plus Go's own `GOTOOLCHAIN=auto` (default since 1.21) already pins the compiler version and auto-fetches it when the local `go` is older; `actions/setup-go`'s `go-version-file: go.mod` makes CI read that same number. A `mise.toml` (or `asdf`) pinning the same version a second time is a second source of truth for a fact `go.mod` already owns — skip it for a single-language repo. It earns its place once there's a _second_ tool with no native version pin anywhere (a locally-installed `goreleaser` while CI runs `version: latest`, say) or the repo goes polyglot.

## Common Mistakes

- **Expecting `gorelease` to read local git tags** — it needs network access to the module proxy, and refuses to run against an uncommitted change.
- **Adding a `mise.toml` that just re-pins what `go.mod` already pins** — redundant source of truth for a single-language repo.
- **Setting `use_existing_draft` without release-please's paired `draft`/`force-tag-creation` config, or vice versa** — the two are one recipe; half of it alone either leaves release-please's release stuck as a permanent draft, or gives GoReleaser no draft to find, so it creates a second release for the same tag instead.
- release-please and repo-hardening mistakes live in those skills' own Common Mistakes — check both too when reviewing a full pipeline.

## Completion criterion

`.goreleaser.yaml` passes `goreleaser check` and includes `changelog.disable: true` and `release.mode: keep-existing` (plus `use_existing_draft: true` if immutable releases are enabled); CI workflow runs `gofmt`, `go vet`, `go test -race`, `go mod tidy` drift check, `govulncheck`, and `gorelease`; golangci-lint config uses `linters.default: none` with an explicit enable list.
