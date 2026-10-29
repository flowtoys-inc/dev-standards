# dev-standards

[![CI](https://github.com/flowtoys-inc/dev-standards/actions/workflows/ci.yml/badge.svg)](https://github.com/flowtoys-inc/dev-standards/actions/workflows/ci.yml)

Shared development standards for every
[flowtoys-inc](https://github.com/flowtoys-inc) repository: firmware for
flowtoys LED flow props, and the tools around it. This repo holds:

1. **The rules:** [`CLAUDE.md`](CLAUDE.md), with a section per stack:
   - Rust
   - C/C++
   - Dart/Flutter
   - Python
   - KiCad
   - docs

   People and AI agents follow these rules in every flowtoys repo. A
   repo's own `CLAUDE.md` or `AGENTS.md` adds to them and wins where they
   conflict. [`CONTRIBUTING.md`](CONTRIBUTING.md) is the short,
   human-readable version for contributors who don't use Claude.
2. **Reusable GitHub Actions workflows,** so repos call the same PR
   checks, Rust CI and release automation instead of copying them.

The org is on GitHub Free: 2,000 Actions minutes a month (macOS counts
about 10×), and no rulesets or required checks on private repos. The
workflows here are built to that budget; see "Treat Actions minutes as a
budget" in `CLAUDE.md`.

## Using the rules with Claude Code

Clone this repo next to your other checkouts and import it from your
user-level memory:

```sh
gh repo clone flowtoys-inc/dev-standards ~/src/flowtoys-dev-standards
echo '@~/src/flowtoys-dev-standards/CLAUDE.md' >> ~/.claude/CLAUDE.md
```

Update with `git -C ~/src/flowtoys-dev-standards pull`. A repo can also
point at the rules from its own `AGENTS.md`, so tools other than Claude
Code find them.

## Reusable workflows

| Workflow | Use it for | Check name in the caller |
|---|---|---|
| `pr-checks.yml` | Conventional Commit PR title + TruffleHog secret scan, in one job; the scan skips `edited` events | `<job id> / pr-checks` |
| `rust-ci.yml` | `rust-toolchain.toml`, `cargo fmt`, `cargo deny`, clippy, tests, optional `extra-checks`, in one job | `<job id> / Rust` |
| `release-please.yml` | semver release PRs, `CHANGELOG.md`, tags, GitHub Releases; outputs `release_created` and `tag_name` | (not a check) |

Every workflow runs on `ubuntu-26.04`, pins its actions to commit SHAs and
asks only for the permissions it needs. The caller must grant at least
those permissions, at the workflow or calling-job level.

### Quickstart

Callers pin a commit SHA of a release tag, with the version in a comment.
Look up the SHA with:

```sh
gh api repos/flowtoys-inc/dev-standards/commits/v0.1.0 --jq .sha
```

`.github/workflows/pr.yml`:

```yaml
name: PR checks

on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

permissions:
  contents: read
  pull-requests: read

jobs:
  pr-checks:
    uses: flowtoys-inc/dev-standards/.github/workflows/pr-checks.yml@<40-char sha> # v0.1.0
```

`.github/workflows/ci.yml`, for a Rust repo:

```yaml
name: CI

on:
  push:
    branches: [main]
    paths-ignore: ["brain/**", "**/*.md"]
  pull_request:
    paths-ignore: ["brain/**", "**/*.md"]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  rust:
    uses: flowtoys-inc/dev-standards/.github/workflows/rust-ci.yml@<40-char sha> # v0.1.0
    with:
      apt-packages: libclang-dev
```

`.github/workflows/release.yml`, plus `release-please-config.json` and
`.release-please-manifest.json` (copy this repo's and change
`release-type` and `package-name`):

```yaml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release:
    uses: flowtoys-inc/dev-standards/.github/workflows/release-please.yml@<40-char sha> # v0.1.0
```

Add `.github/dependabot.yml` with a weekly, grouped `github-actions`
entry (copy [this repo's](.github/dependabot.yml)), and it will propose
bumps to the pinned SHA as new releases of this repo ship.

### Access from other repos

This repo is public, so any repo can call its workflows, private or
public. Keep it free of secrets and of private repo or product names.
Name a stack in the rules, not the repos that use it.

## Adopting in an existing repo

1. Replace inline PR-title and secret-scan jobs with a call to
   `pr-checks.yml`. The check is renamed to `<job id> / pr-checks`.
2. Replace an inline release-please job with a call to
   `release-please.yml`. Keep any `if:` on the calling job, and grant it
   `contents: write` and `pull-requests: write`. Jobs that build release
   assets keep `needs:` on it and read `release_created` and `tag_name`.
3. Keep firmware builds that need a cross toolchain or vendor SDK in the
   repo's own CI. A reusable workflow can't run the caller's composite
   actions before its own steps.
4. Make sure `.github/dependabot.yml` bumps `github-actions` weekly, so
   it picks up new releases of this repo.
5. Link this repo's `CLAUDE.md` and `CONTRIBUTING.md` from the repo's
   README or `AGENTS.md`.

## New repositories

1. `gh repo create flowtoys-inc/<name> --private`, with `main` as the
   default branch.
2. Apply the repository settings listed in `CLAUDE.md` (squash-only,
   delete branches on merge, Actions may create PRs).
3. Add `pr.yml`, `ci.yml`, `release.yml`, the release-please config and
   `dependabot.yml` from the quickstart.

## Repository settings as code (planned)

`ayanich/github-terraform-ayanich` manages a personal account's repo
settings with OpenTofu. A `flowtoys-inc/github-terraform-flowtoys` repo
would do the same for the org: merge options, Actions permissions
(including this repo's access level), security updates and Dependabot
files. Until it exists, settings are applied by hand, as `CLAUDE.md`
describes. On GitHub Free it can't add rulesets or branch protection to
private repos.

## Developing this repo

- Branch, open a PR with a Conventional Commit title, squash-merge.
- CI runs [actionlint](https://github.com/rhysd/actionlint) when anything
  under `.github/` changes. Run it locally with `actionlint`.
- Releases are cut by release-please: merging its release PR tags
  `vX.Y.Z`. Callers move to it through their Dependabot PRs.
- Changing a reusable workflow's inputs, outputs, job names or required
  permissions breaks callers: mark it `feat!:` or `fix!:`.
- Every workflow change states its estimated Actions minutes in the PR.
