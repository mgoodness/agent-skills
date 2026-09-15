---
name: release-please
description: >
  Expert skill for wiring release-please, the GitHub Action that cuts versioned releases
  automatically from Conventional Commits: the component-tag trap that silently mismatches the
  tag a downstream release workflow watches for, GitHub's loop prevention that can silently
  swallow release-please's own tag push before that workflow ever fires, and the
  draft/force-tag-creation pairing needed when a downstream tool must finish attaching release
  assets before the release can publish (e.g. a repo with immutable releases enabled). Use
  whenever setting up or reviewing `release-please-config.json`,
  `.release-please-manifest.json`, or the GitHub Actions workflow that runs release-please
  itself, for any repo regardless of language. Defers the App-token mechanics that fix the
  loop-prevention trap to the `github-app-token` skill, the ruleset/Dependabot/Actions-hardening
  wiring the release still depends on to the `repo-hardening` skill, and the resulting
  build/tag/publish recipe to a language skill (e.g. `go-ci`).
---

# release-please: Cutting Releases from Conventional Commits

release-please reads Conventional Commits history and opens (then, on merge, tags) a release PR. Three traps stop that tag from ever doing anything, or from ever finishing: a config mismatch that changes what the tag looks like, GitHub's own loop prevention that can silently swallow the push that creates it, and (when the release must stay a draft) GitHub deferring the tag itself until publish.

## When to Activate

- A tag-triggered release workflow (GoReleaser or otherwise) never seems to fire after release-please merges a release PR

## 1. Config — the component-tag trap

`release-please-config.json` and `.release-please-manifest.json` (`{".": "0.0.0"}`) are the whole config for a single-package repo. Leave `package-name` **unset** in the package entry:

```json
"packages": { ".": {} }
```

Setting `package-name` gives release-please a _component_, and `includeComponentInTag` defaults to `true` — the tag becomes `<package-name>-v1.0.0`, not `v1.0.0`. A release workflow triggering on `tags: ["v*"]` then never fires against that push, silently: no error, no failed run, just nothing happening. If you want a component name for another reason, pair it with `"include-component-in-tag": false`.

## 2. Workflow — loop prevention

GitHub's **loop prevention**: a push or PR authored by the default `GITHUB_TOKEN` cannot trigger further workflow runs. release-please pushes the release tag from step 1; if that push is `GITHUB_TOKEN`-authored, the downstream release workflow watching for `v*` never fires — again silently.

Fix: mint a short-lived token from a GitHub App installation via `actions/create-github-app-token`, scoped down to exactly what release-please needs. See the `github-app-token` skill for the mechanics — minting, scoping, reusing one App across a repo's automation — and for a second, distinct GitHub restriction (the workflow-run approval gate) that pattern also happens to clear:

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

The downstream release workflow does **not** need this token — it only uploads artifacts to a release the tag-authoring workflow already created, so the default `GITHUB_TOKEN` is enough there.

Creating the App, and pulling its Client ID and private key into the repo's variable/secret, is one-time work only a human can click through — script it with the `wizard` skill rather than writing it as prose steps to follow by hand.

The release-please PR itself merges through the same ruleset/required-check gate as any other PR (`repo-hardening`'s steps 1–2) — nothing special to configure here beyond making sure that gate exists.

## 3. Draft mode — the immutable-releases pairing, and its own tag trap

A repo with `repo-hardening`'s immutable releases enabled can't let release-please publish the GitHub Release the moment it creates the tag: GitHub blocks adding assets to an already-published immutable release, and by default that's exactly when release-please publishes — before a downstream tool (GoReleaser or otherwise) has attached a single build artifact. Hold it as a draft instead:

```json
{
  "draft": true,
  "force-tag-creation": true
}
```

`draft: true` alone reproduces the same failure shape as the component-tag trap in step 1: GitHub defers *tag creation itself* for a draft release until it's published ("lazy tag creation"), so the tag-triggered downstream workflow — the same one step 2's loop prevention exists to protect — never fires, again silently. `force-tag-creation: true` forces the tag into existence immediately despite the release staying unpublished, which is what lets that workflow still trigger.

Finding that same draft release by tag, attaching every artifact, and publishing it is the downstream tool's responsibility, not release-please's — see `go-ci`'s GoReleaser recipe for the GoReleaser side of this pairing.

## Common Mistakes

- **Setting `package-name` in `release-please-config.json`** for a single-package repo — silently mismatches the tag the release workflow is watching for.
- **Minting an App token with no `permission-*` inputs** — see `repo-hardening`'s Security hardening section.
- **Setting `draft: true` without `force-tag-creation: true`** — GitHub defers tag creation on a draft release until it's published, so a tag-triggered downstream workflow never fires; same silent-failure shape as the component-tag trap.
