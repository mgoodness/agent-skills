---
name: repo-hardening
description: >
  Harden a GitHub repo so required checks and automated merges are actually safe: rulesets,
  merge settings, Dependabot with auto-merge, immutable releases, and Actions security practices
  (SHA pinning, `pull_request_target` hygiene, token scoping). Use when configuring or auditing
  a ruleset, merge settings, Dependabot, auto-merge, immutable releases, or when zizmor/actionlint
  flags a workflow. Defers CI content to a language skill (e.g. `go-ci`), release-please to
  `release-please`, and App-token mechanics to `github-app-token`.
---

# Repo Hardening: Making Required Checks and Auto-Merge Actually Bite

A required-check status and an "auto-merge" button are both easy to turn on and easy to leave toothless. This skill is the dependency chain that makes them real: name the check correctly, gate on it with a ruleset, back that with repo-wide merge settings that agree with the ruleset, then layer Dependabot and its auto-merge on top — each step here produces something the next one needs, so skipping the order reproduces exactly the failure this skill exists to prevent: an auto-merge that fires with nothing actually gating it.

Nothing here is language-specific. What the required CI job itself builds, tests, and lints is owned by a language skill (`go-ci` for Go), which points back here for the ruleset, Dependabot, and hardening. Cutting a versioned release from that same repo's commit history is a separate concern, owned by the `release-please` skill.

## 1. Repository ruleset — turning the required check on

Whatever job gates merges (a language skill's CI workflow, e.g. `go-ci`) is what the ruleset and auto-merge (step 4) both key off. Name it deliberately: the ruleset's `required_status_checks` rule matches the job on its `name:` field, not its `jobs.<id>` key.

GitHub's **repository rulesets** are the current mechanism for this — classic branch protection (the single `PUT .../branches/{branch}/protection` call) is the predecessor API, still supported but no longer where new capability lands: rulesets let more than one ruleset apply to a branch at once (most-restrictive rule wins), target several branches with one glob pattern, and target tags/pushes too. Start new repos here; an existing repo with classic protection should migrate rather than run both indefinitely (see Common Mistakes).

This is the step that makes the required check and auto-merge (step 4) load-bearing rather than decorative: skip it and the CI job still runs and reports a status, it just blocks nothing. Not a CI step — creating a ruleset needs repo-admin permissions the default `GITHUB_TOKEN` doesn't carry — so this is a one-time, admin-run `gh api` call. The payload is in [`RULESET-PAYLOAD.json`](RULESET-PAYLOAD.json); adapt the `context` value to the CI job's actual `name:` field.

- `conditions.ref_name.include` is a glob pattern (`refs/heads/main`, or `refs/heads/release/*` to cover several branches with one ruleset) — the ruleset's equivalent of classic protection being scoped to a single named branch.
- `deletion`, `non_fast_forward`, and `required_linear_history` are separate rule *types*, each opted into by including it, rather than boolean fields on one object — the inverse polarity from classic protection's `allow_deletions`/`allow_force_pushes: false`. Omitting a rule type means that constraint isn't enforced at all, the same as leaving the classic boolean at its permissive default.
- The `pull_request` rule replaces `required_pull_request_reviews`; `required_review_thread_resolution` is its `required_conversation_resolution` equivalent. `required_approving_review_count: 0` is still valid and common for a solo-maintainer repo: a PR and its checks are still required, just not a second human reviewer.
- The `required_status_checks` rule's own `required_status_checks` array (yes, the rule type and its parameter share a name) takes `context` values matching the CI job's `name:` field — the same mismatch step 1 warns about, made concrete here. `strict_required_status_checks_policy: true` is `strict` renamed: it requires the PR branch be up to date with the base before merging.
- **`bypass_actors: []` is not optional to reason about — it's the load-bearing line.** Rulesets have no implicit admin exemption: unlike classic protection's `enforce_admins`, which defaults to *not* enforcing against admins until you flip it on, an empty (or omitted) `bypass_actors` list means the ruleset binds literally everyone, repo and org admins included. Classic protection's `enforce_admins: true` closed an escape hatch that existed by default; a ruleset with no bypass actors has no escape hatch to close in the first place. Add an entry only for a bypass you actually want (e.g. `{"actor_type": "OrganizationAdmin", "bypass_mode": "always"}`), and confirm the exact `actor_type`/`bypass_mode` shape against current docs before scripting it — this is the newest, still-evolving part of the payload.
- `required_linear_history` forces every merge through squash or rebase, not a merge commit — pair it with step 2's repo-level merge setting or the two disagree about what history should look like. The `pull_request` rule also supports an `allowed_merge_methods` parameter that enforces the same restriction at ruleset scope rather than repo-wide; useful when different branches need different merge-method rules, but verify its exact shape against current docs before relying on it, same caveat as `bypass_actors`.

## 2. Repo merge settings — squash-only, auto-delete branches

Two repo-level settings (`Settings → General → Pull Requests`, or `gh api -X PATCH repos/<owner>/<repo>`), not part of the ruleset at all, but load-bearing alongside it:

```sh
gh api -X PATCH repos/<owner>/<repo>` \
  -F allow_merge_commit=false \
  -F allow_squash_merge=true \
  -F allow_rebase_merge=true \
  -F delete_branch_on_merge=true \
  -F allow_auto_merge=true
```

- `allow_merge_commit=false` disables merge commits repo-wide, so squash or rebase are the only options left — this is what actually enforces step 1's `required_linear_history`; setting one without the other leaves them disagreeing about what history should look like.
- `delete_branch_on_merge=true` deletes a PR's head branch automatically once merged, so a repo doesn't accumulate a long tail of stale merged branches that every `git branch -a` and every branch-selection prompt has to wade through.
- `allow_auto_merge=true` is the other prerequisite step 4 needs — without it, `gh pr merge --auto` (or "enable auto-merge" in the UI) simply isn't available to request.

## 3. Dependabot config

`gomod`/`npm`/`pip`/etc. plus `github-actions` ecosystems, weekly schedule, `cooldown.default-days >= 7` — anything shorter trips zizmor's `dependabot-cooldown` audit.

## 4. Dependabot auto-merge

Gate on `dependabot/fetch-metadata`'s `update-type` output: auto-merge `semver-minor`/`semver-patch`, leave `semver-major` for manual review — every major bump is a real compatibility question, not busywork.

This only fires safely because of the prerequisites steps 1 and 2 cover (the ruleset requiring the CI job's check, and the repo's own "Allow auto-merge") — configure those there, not here; skip either and "enable auto-merge" merges on PR open with zero verification.

This workflow runs on `pull_request_target`, for the write-scoped token Dependabot's own `pull_request` event never gets (regardless of the workflow's `permissions:` block). That's the one place `pull_request_target` is safe to use here, and only because the workflow never checks out PR code and only ever calls `gh pr merge` against the PR's URL — see Security hardening below for how to justify it to a linter.

Keep auto-merge scoped to Dependabot only by default. Auto-merging mechanical, high-volume dependency bumps is worth automating; auto-merging human-authored PRs is a per-repo call each maintainer should make explicitly (e.g. enabling per-PR by hand), not something to generate speculatively alongside the Dependabot piece.

## Immutable releases — locking published tags and assets

A repo-admin, one-time `gh api` call like step 1's ruleset, but independent of the required-check/auto-merge chain above: it hardens the repo's *release* supply chain instead, and neither depends on nor is depended on by steps 1–4.

```sh
gh api -X PUT repos/<owner>/<repo>/immutable-releases
```

- Once a release is published under this setting, its Git tag is locked to the commit it was created at — can't be moved, retargeted, or deleted while the release exists, and the tag name can't be reused even if the repo itself is deleted and recreated (protection against repository-resurrection attacks). Every asset already attached is protected from modification or deletion. Title, release notes, and the prerelease/latest flags stay editable. Publishing also mints a release attestation (a cryptographically verifiable record binding the tag, commit SHA, and asset list) as a byproduct — nothing extra to configure for it.
- **Publish as late as possible: draft, attach every asset, then publish.** GitHub's docs are explicit about the previous bullet's asset lock, not merely recommending this as best practice: it's a hard blocker for the default release-please + GoReleaser pipeline (pipeline specifics are `release-please`/`go-ci` territory, not this skill's): release-please's action publishes the GitHub Release immediately when it creates the tag, with only changelog notes attached; GoReleaser then attaches build artifacts to that *already-published* release on the later tag-push trigger — exactly the second pass GitHub's guidance forbids. The fix is holding the release as a draft until every artifact is attached, then publishing it once: release-please's own `draft`/`force-tag-creation` config (see `release-please`'s draft-mode trap) plus the downstream tool finding and publishing that same draft rather than creating a separate release (GoReleaser's `use_existing_draft` — see `go-ci`'s GoReleaser recipe). Enabling immutable releases without confirming that reorder is in place breaks the very next tag push.
- An org admin can also mandate this repo-wide (`PUT /orgs/{org}/settings/immutable-releases` with `enforced_repositories: all`/`none`/`selected`) — if the org already enforces `all`, the per-repo call above just confirms existing state rather than creating new state.

## Security hardening (what zizmor will flag)

- **Pin every third-party action to a commit SHA with a trailing `# vX.Y.Z` comment**, never a bare tag. The version comment is what lets Dependabot's `github-actions` ecosystem keep bumping it — a bare SHA with no comment is a pin nothing can track. Resolve the SHA from the tag's **commit**, not an annotated tag's own object: the GitHub API's `git/refs/tags/<tag>` returns an `object.sha` that, for an annotated tag, names a tag object (`type: "tag"`), one level short of the commit — dereference it (`git/tags/<that-sha>`) or use `git rev-parse <tag>` with the `^{commit}` suffix locally. Pinning the tag object's SHA instead still resolves (Actions accepts it), so nothing fails until zizmor's `ref-version-mismatch` audit flags the mismatch between the comment's version and what the hash actually points to.
- **`persist-credentials: false`** on any `actions/checkout` step that doesn't need the default token to push afterward, and always on the step that checks out untrusted PR code.
- **Bot-identity checks**: use `github.event.pull_request.user.login`, never `github.actor` — `actor` is spoofable (zizmor's `bot-conditions` audit).
- **`pull_request_target` is flagged on principle** (`dangerous-triggers`): the usual failure mode is checking out and running the PR's own code under a write-scoped token. Suppress only when you can name, in the same comment, why _this_ workflow is immune — no checkout of PR code, and no PR-derived value interpolated directly into a `run:` script (only ever passed through `env:`):
  ```yaml
  on: pull_request_target # zizmor: ignore[dangerous-triggers] no checkout of PR code; PR-derived values only flow through env:, never interpolated into run: scripts
  ```
- **Scope minted App tokens down** with `permission-*` inputs; never let a job's token inherit an App's full installation grant when it only needs to push a tag and open a PR. See the `github-app-token` skill for minting mechanics, reusing one App across a repo's automation, and a workflow-run approval gate that same pattern also happens to clear.

## Common Mistakes

- **Key bot-identity checks on `github.event.pull_request.user.login`** — `github.actor` is spoofable; see Security hardening's bot-identity bullet.
- **Use `pull_request_target` only on workflows that never check out PR code** — see Security hardening's `pull_request_target` bullet.
- **Scope every minted App token down with `permission-*` inputs** — see Security hardening's token-scoping bullet.
- **Name the required-check ruleset rule after the job's `name:` field, not its `id`** — see step 1.
- **Wire the ruleset requiring the CI check before Dependabot auto-merge** — see step 4's prerequisites.
- **Set `strict_required_status_checks_policy: true`** — leaving it unset or `false` lets a PR merge on a check result from a now-stale base branch.
- **Disable `allow_merge_commit` whenever `required_linear_history` is enabled** — see step 2.
- **Confirm `bypass_actors` is empty (or explicitly set) rather than assuming admin exemption** — classic protection's `enforce_admins` defaulted to exempting admins; rulesets do not. See step 1's `bypass_actors` bullet.
- **Migrate fully from classic branch protection to rulesets** — running both simultaneously produces the most-restrictive combination, which is confusing to reason about and audit. Remove the classic rule once the ruleset covers the same ground.
- **Pin every action to the commit SHA the tag points to, not the annotated tag's own object SHA** — see Security hardening's SHA-pinning bullet.
- **Keep auto-merge scoped to Dependabot only by default** — avoid general-purpose "auto-merge for write access" workflows; see step 4.
- **Rebase stale Dependabot PRs with `@dependabot rebase`** — merging a fix to the base branch doesn't retroactively re-run an already-open PR's checks.
- **Switch a release-please + GoReleaser pipeline to draft-first before enabling immutable releases** — see the Immutable releases section above.

## Completion criterion

Ruleset is active on the target branch; required check `context` matches the CI job's `name:` field; merge settings disable merge commits and enable auto-merge; Dependabot config and auto-merge workflow are present; every third-party action in every workflow is pinned to a commit SHA with a `# vX.Y.Z` comment.
