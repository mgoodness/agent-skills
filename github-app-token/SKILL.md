---
name: github-app-token
description: >
  Expert skill for minting and using a GitHub App installation token in place of the default
  `GITHUB_TOKEN`, via `actions/create-github-app-token`. Covers the two silent handicaps that
  token carries — loop prevention (a `GITHUB_TOKEN`-authored push or PR never triggers a
  downstream workflow) and the workflow-run approval gate (a bot-authored PR's checks can sit in
  `action_required` waiting on a manual click even when it isn't a fork PR) — plus the mechanics
  of minting the token, scoping it, and re-establishing a bot identity for steps that still key
  off `github.actor`. Use whenever a workflow's push, tag, PR, comment, or merge needs to act as a
  first-party identity instead of `github-actions[bot]`; whenever a tag-triggered or PR-triggered
  downstream workflow silently never fires; whenever a bot-authored PR's check run is stuck
  needing manual approval; or whenever standing up any new automation that needs to push, tag, or
  open PRs with elevated trust. One App, reused across every workflow in a repo that needs this —
  don't mint a new one per use case. Defers the repo-admin ruleset/Dependabot/Actions-hardening
  wiring this token still depends on to `repo-hardening`, and release-please's own config/draft-mode
  traps to the `release-please` skill.
---

# GitHub App Tokens: Escaping the Default GITHUB_TOKEN's Silent Limits

The default `GITHUB_TOKEN` carries two handicaps that don't error when they bite — they just quietly fail to do the thing you expected. Both trace back to the same cause: GitHub treats the `github-actions[bot]` identity behind that token as second-class, deliberately, as an anti-abuse measure. The fix for both is the same one-line swap: mint a token from a GitHub App installation instead, via `actions/create-github-app-token`. This skill is the pattern once, so it stops getting rediscovered piecemeal inside whichever workflow hits it next — release-please's tag push was one site; an autobump job's pull-request run was another.

## When to Activate

- A push or tag authored by `GITHUB_TOKEN` should trigger a downstream workflow, but that workflow never runs — no error, just silence.
- A bot- or automation-authored PR's checks sit in `action_required`, waiting on a manual "Approve and run" click, even though it isn't a fork PR.
- A workflow needs to push, tag, comment, or open/merge a PR as a real, trusted identity instead of `github-actions[bot]`.
- Standing up new automation (a bump job, a release job, anything that opens PRs on a schedule) that will need any of the above.

## 1. Loop prevention

A push or PR authored by the default `GITHUB_TOKEN` cannot trigger further workflow runs. If a workflow pushes a tag, or opens a PR, expecting some other `on: push`/`on: pull_request` workflow to pick it up, that second workflow simply never fires when the first push was `GITHUB_TOKEN`-authored — nothing fails, nothing logs, the run just doesn't exist.

## 2. The workflow-run approval gate

Separately, a `pull_request`-triggered run can land in `action_required` instead of running — GitHub's way of asking a human to click "Approve and run" before untrusted code executes. This is usually explained as a fork-PR protection, but it isn't only that: it also catches same-repo PRs opened by an identity GitHub doesn't consider a collaborator, which includes `github-actions[bot]` itself. An autobump-style workflow that pushes a branch into the same repo (no fork involved) and opens a PR under the default token can still trip this gate on its own PR.

**The PR's `author_association` field is not a reliable signal here, and this is the trap:** both `github-actions[bot]` and an installed GitHub App's own bot identity report `author_association: NONE` on a PR they open — displayed identically — yet only the former gets gated in practice. Confirmed by direct test: pushing and opening a PR with a live App installation token produced a `pull_request` run that went straight to `in_progress` and completed on its own, while the equivalent PR opened via `secrets.GITHUB_TOKEN` sat in `action_required`. Whatever GitHub actually keys the gate on, it isn't the association field a PR's metadata shows you — treat that field as uninformative for this question, and verify empirically against the target repo rather than reasoning from it. This was observed on a personal (non-org) public repo with no branch protection; org-owned repos expose an explicit toggle for this (`Settings → Actions → General → Fork pull request workflows`) that may interact differently — confirm behavior on the repo in question before relying on it.

## 3. Minting the token

```yaml
- name: Mint App token
  id: app-token
  uses: actions/create-github-app-token@<sha> # vX.Y.Z
  with:
    client-id: ${{ vars.MY_APP_CLIENT_ID }}
    private-key: ${{ secrets.MY_APP_PRIVATE_KEY }}
    permission-contents: write
    permission-pull-requests: write
```

- Omit `owner`/`repositories` to default to the current repository; the action logs this choice explicitly (`Inputs 'owner' and 'repositories' are not set. Creating token for this repository.`) so it's easy to confirm you got the scope you expected.
- Scope every mint down with `permission-*` inputs to exactly what that job needs — never let a job's token inherit the App's full installation grant. See `repo-hardening`'s Security hardening section for this discipline, and for the commit-SHA pinning (`# vX.Y.Z` comment, dereferenced to the tag's actual commit) that applies to this action like any other.
- Feed the minted token wherever the workflow previously passed `secrets.GITHUB_TOKEN` — as a step's `with: token:`, or as an env var a CLI (`gh`, `brew`, etc.) reads.

## 4. Re-establishing identity downstream

Swapping the credential doesn't change what `github.actor` resolves to in the rest of the job — that context value still reflects whatever triggered the run, not the App. Any downstream step that defaults its notion of "who is committing this" to `github.actor` (a git-identity-lookup action, a changelog generator, anything that looks up a user by login) will silently commit or attribute as the wrong identity unless told otherwise. Pass the App's own login explicitly using the token-minting step's `app-slug` output:

```yaml
- name: Set up git identity
  uses: some/git-user-config-action@<sha> # vX.Y.Z
  with:
    token: ${{ steps.app-token.outputs.token }}
    username: ${{ steps.app-token.outputs.app-slug }}[bot]
```

Generalize this to any step in the job that resolves an identity from context rather than from the token actually being used.

## 5. One App, many workflows

An App installation's permissions apply repo-wide (or across every repo it's installed on, per its `repository_selection`), so a single App backs every workflow in a repo that needs this escape hatch — a release job, an autobump job, a Dependabot follow-up — rather than minting a new App per use case. Reuse the same Client ID/private key pair across every `create-github-app-token` step that needs it, scoping each individual mint's `permission-*` inputs down to only what that particular job does.

## Setup (one-time, human-only)

Creating the App and installing it on the target repo(s) is manual dashboard work — script it with the `wizard` skill rather than writing it as prose steps to follow by hand. Store its Client ID as a repo or org **variable** and its private key as a repo or org **secret**; name the pair after the App, not after any one workflow that consumes it, since step 5 expects reuse.

## Common Mistakes

- **Reading `author_association` (or any other PR metadata) to predict whether a run needs manual approval** — it doesn't reliably distinguish a gated identity from an ungated one; see step 2.
- **Minting a new GitHub App per workflow** instead of reusing one installation across a repo's automation — see step 5.
- **Swapping the token but not the identity** — a downstream step still resolving `github.actor` will commit or attribute as the wrong bot; see step 4.
- **Minting an App token with no `permission-*` inputs** — see `repo-hardening`'s Security hardening section.
- **Assuming this fixes every "workflow didn't run" case** — confirm which of loop prevention (step 1) or the approval gate (step 2) is actually in play before reaching for this; they're distinct mechanisms with the same fix, not one mechanism.
