# Contributing to flowtoys repositories

This is the short version of the rules in [`CLAUDE.md`](CLAUDE.md), for
anyone working on a flowtoys-inc repo, with or without an AI assistant.
Each repo's own README says how to build and test it.

## Making a change

1. **Branch** off the default branch, named `feat/…`, `fix/…`, `docs/…`,
   `chore/…`, `ci/…` or `build/…`. Never push straight to the default
   branch.
2. **Open a pull request** with a [Conventional Commit](https://www.conventionalcommits.org/)
   title, for example `fix: keep brightness after a power cycle`. Allowed
   types are `feat`, `fix`, `perf`, `refactor`, `docs`, `test`, `build`,
   `ci`, `chore` and `revert`. Add `!` for a breaking change, as in
   `feat!: …`.
3. **Wait for CI to go green.** GitHub won't block a red PR on our plan,
   so check before merging.
4. **Squash-merge.** The PR title becomes the commit message, and
   release-please uses it to pick the next version and write the
   changelog.

## Before you push

- Run the repo's formatter, linter and tests. The README lists the
  commands.
- Add or update tests with every change. Fixing a bug starts with a test
  that fails.
- For firmware that touches LEDs, motion, radio, battery, flash or
  timing, test on a real device. Say in the PR what you flashed and what
  you saw.
- Never commit secrets. Every PR is scanned for them.

## CI costs real money

Private repos share 2,000 GitHub Actions minutes a month, and a macOS
minute counts as about ten. A workflow change needs an estimate of its
minutes in the PR. Ask before adding macOS or Windows jobs.

## Hardware repos

- Use the KiCad version pinned in the README.
- Tag each board revision that goes to fab.
- Large binaries (STEP, zip, PDF) go through Git LFS.
