# Development standards (flowtoys-inc)

These are the general rules for every repository in the
[flowtoys-inc](https://github.com/flowtoys-inc) GitHub organization:
firmware for flowtoys LED flow props, and the tools around it.

- **A repo's own `CLAUDE.md` or `AGENTS.md` adds project-specific rules,**
  and wins where the two conflict.
- **Follow these rules without being asked.**

## Plan constraints

The org is on **GitHub Free**. Two limits shape everything below:

- **2,000 Actions minutes a month** across all private repos. A macOS
  minute counts about 10× a Linux one. When the quota runs out, every
  workflow in the org stops until the next cycle. This has already
  happened once, in a single afternoon.
- **No rulesets, branch protection or required status checks on private
  repos.** Nothing stops a red PR from merging, so humans and agents check
  CI before they merge.

## Scope

These rules cover every active repo. Each stack section below applies
to any repo that uses that stack.

- **Forks of upstream projects** (for example `flutter_file_picker`)
  follow upstream's conventions, not these.
- **Dormant repos,** with no commits since 2021, are out of scope.
  Archive them on GitHub rather than bringing them up to standard.

## Firmware in general

These rules apply to every firmware repo, whatever the language.

- **Green host tests are not working firmware.** Mocks and simulators lie
  about timing, ISR and DMA ordering, flash failures, sensor noise and RF.
  Any change that touches hardware behavior ends with an explicit
  hardware-verification hand-off: how to flash, what to do, what success
  and failure look like. Never call such a change done from CI alone.
- **Wire and flash layouts are stable.** Persisted records and radio
  packets carry a version. Change one by adding a new version and a
  migration, never by reordering or reusing fields. Lock each layout with
  a golden-byte test.
- **Cross toolchains are pinned.** CI downloads a specific
  `arm-none-eabi` release and checks its SHA-256 before using it.
- **Vendor SDKs** (for example Silicon Labs' Simplicity SDK) are pinned
  submodules at a tagged release, and CI caches them by commit.

## Rust (embedded and host)

Firmware, host test rigs and simulators.

- **Pin the toolchain in `rust-toolchain.toml`** to the latest stable,
  including targets and components. CI installs from it with
  `rustup toolchain install`. Bump the pin in its own PR.
- **Commit `Cargo.lock`,** for libraries as well as binaries.
- **Style and lints:**
  - `cargo fmt --check`
  - `cargo clippy` with `-D warnings`
  - `cargo deny check`, for licenses, advisories and duplicate crates
- **Tests:**
  - Unit tests sit next to the code in `#[cfg(test)]` modules.
  - Integration tests go in each crate's `tests/`.
  - Code reaches hardware through a trait seam that tests mock, so the
    logic runs on the host: mockall with `#[double]` swaps in the mock
    under `cfg(test)`.
- **CI:** call `rust-ci.yml`, or for firmware that needs a cross
  toolchain, keep firmware CI in the repo:
  - Host and device builds run as parallel jobs, sharing a composite
    toolchain action.
  - rust-cache is saved only from the default branch.

## C and C++ (firmware and desktop)

Desktop apps (FlowtoysUpdater, built with JUCE), legacy Make-based
firmware, and C++ libraries called from Rust through bindgen.

- **Build:** new code uses CMake ≥ 3.22. Legacy firmware keeps its
  Makefiles, and isn't migrated just for the sake of it.
- **Third-party code** comes in as shallow submodules pinned to a tag
  (`shallow = true` in `.gitmodules`), not as vendored copies.
- **Style:** clang-format with a committed `.clang-format`. Add
  clang-tidy where it's practical, starting with new code.
- **Warnings:** keep them on (`-Wall -Wextra`). Put `-Werror` behind a
  build option that CI turns on, never on by default, because a newer
  compiler would then break every user's build.
- **Tests:** ctest, run in CI.
- **Desktop matrices:** build macOS and Windows only where Linux can't
  cover it.
  - A public repo (FlowtoysUpdater) gets free minutes, so a full matrix
    is fine there.
  - Runner pins still follow the newest-GA rule.

## Dart and Flutter

Mobile and desktop apps.

- **Pin Flutter** to the latest stable in `.fvmrc` (FVM), and use that
  same version as `flutter-version` in CI.
- **Commit `pubspec.lock`.**
- **Style:** very_good_analysis, with `dart format` at the configured
  page width (80).
- **Checks:** `dart format --set-exit-if-changed`, `flutter analyze` and
  `flutter test`.
- **CI:**
  - Run analyze and tests on Linux on every PR.
  - Build Android on Linux.
  - Build iOS and macOS only on tags or `workflow_dispatch`, because
    macOS minutes cost about 10×.
- **Dependabot** covers the `pub` ecosystem.

## Python

Python is for tooling, scripts, test harnesses and firmware updaters.

- **Use the latest stable CPython.** It is currently **3.14**. Pin it in
  `.python-version`, with `requires-python` in `pyproject.toml`.
- **Manage everything with [uv](https://docs.astral.sh/uv/).**
  - `uv sync` creates `.venv`. Run everything as `uv run …`.
  - Never `pip install` into a global interpreter.
- **Dependencies:**
  - Add them with `uv add <pkg>`, or `uv add --dev <pkg>` for tools.
  - Commit `uv.lock`. CI installs with `uv sync --locked`.
- **Single-file scripts** declare their dependencies in a PEP 723 header
  and run with `uv run script.py`.
- **`setup.py` with unpinned dependencies is the pattern to migrate
  away from.** Move such projects to `pyproject.toml` and `uv.lock` the next time they're
  touched.
- **Style:** ruff for linting and formatting, configured in
  `pyproject.toml`, with the `UP` rules on.
- **Tests:**
  - Unit tests go in `tests/unit/`: no network, no sleeps.
  - Integration tests go in `tests/integration/`, marked with
    `pytestmark = pytest.mark.integration`.

## KiCad hardware

PCB projects.

- **Pin the KiCad major version** in the README, and open the project
  with that version only. Upgrading is its own PR.
- **Tag each board revision** that goes to fab (for example `v1.2`).
  Commit fab outputs (gerbers, drill files, BOM, placement) only for
  tagged releases, or attach them to the GitHub Release.
- **Store large binaries in Git LFS:** STEP models, zips, PDFs and
  rendered images.
- **CI is optional.** If a repo adds ERC/DRC (for example KiBot), run it
  on Linux only, path-filtered to the schematic and PCB files.

## Docs (AsciiDoc and Markdown)

Protocol specs, and the docs in every repo.

- If a doc builds into an artifact (for example asciidoctor to HTML or
  PDF), build it on GitHub Actions, path-filtered to the doc sources.

## Other languages

Use the ecosystem's latest stable toolchain, with a committed lockfile,
its standard formatter and linter, and the same test, CI and release
rules.

## Tests

- **Every change ships with tests.** A bug fix starts with a failing test.
- **Tests never touch production systems or real devices on their own.**
  They run against local fakes, mocks or simulators.
- **When real hardware or a real system behaves in a new way,** first
  teach the fake or simulator to reproduce it, then write the test.

## CI/CD (GitHub Actions)

- **Every repo has CI.** It is green before anything merges, and since
  private repos can't require checks, look before you merge.
- **The minimum CI depends on the repo:**

  | Repo type | Minimum CI |
  |---|---|
  | any repo | `pr-checks.yml` (PR title + secret scan) |
  | firmware, desktop, tooling | also build, lint and test on Linux |
  | Flutter app | also `flutter analyze` and `flutter test` on Linux |
  | KiCad hardware | nothing else required |
  | docs | also build the docs, path-filtered |

- **GitHub Actions is the only CI.** Some repos still carry a
  `.gitlab-ci.yml` from the GitLab migration, which never runs here.
  Replace it with workflows that call the reusable ones, or delete it.
- **Public repos** get free Actions minutes, so the minute budget matters
  less there. Every other rule still applies, and every private repo
  counts against the org's minutes.
- **Call the reusable workflows in `flowtoys-inc/dev-standards`** rather
  than copying them:

  | Workflow | What it does |
  |---|---|
  | `pr-checks.yml` | Conventional Commit PR title + TruffleHog secret scan, in one job |
  | `rust-ci.yml` | `cargo fmt`, `cargo deny`, clippy and tests, in one job |
  | `release-please.yml` | semver release PRs, tags and GitHub Releases |

  Pin each call to a commit SHA with a `# vX.Y.Z` comment.
- **Workflow policy:**
  - every third-party action pinned to a full commit SHA with a
    `# vX.Y.Z` comment
  - explicit least-privilege `permissions:`, at the workflow level and
    widened per job only where needed
  - `persist-credentials: false` on checkouts that don't push
  - no PR-head checkout under `pull_request_target`
  - no `${{ github.event.* }}` or `${{ inputs.* }}` interpolated into
    `run:`; pass it through `env:` instead
- **Pin runner images to the newest GA version explicitly:** for example
  `ubuntu-26.04` or `macos-26`, never `ubuntu-latest` or `macos-latest`.
  `-latest` lags new releases by months and then moves under you. Bump the
  pin in its own PR when GitHub ships a newer image.
- **Treat Actions minutes as a budget.** When the org's monthly quota runs
  out, every workflow in every repo stops. Design each workflow change for
  cost as well as correctness:
  - Estimate the billed minutes per push and per month, and put the
    estimate in the PR. Every job rounds up to a whole minute, and a macOS
    minute counts about 10× a Linux one.
  - Run macOS or Windows only for what Linux can't test. Path-filter those
    jobs into their own workflow, and never make a path-filtered job a
    required check, because it stays pending when it doesn't run.
  - Put small checks in one job instead of many 10-second jobs. Use
    `if: success() || failure()` so later checks still report.
  - Save caches only from the default branch and restore them on PRs.
    Save even when the job fails, for example rust-cache's
    `cache-on-failure`.
  - Skip work that changes nothing: `paths-ignore` docs-only changes, use
    `concurrency` with `cancel-in-progress`, and don't rerun scans on
    events that change no commits, such as a PR `edited` event.
  - Don't compile the same thing twice, across jobs or between CI and
    release. Pass artifacts along instead.
  - Dependabot's own update jobs are free; the CI they trigger is not.
    Group its updates.
  - After a change lands, check the real per-job timings with
    `gh api repos/<owner>/<repo>/actions/runs/<id>/jobs`.
- **Dependabot:**
  - Run weekly, with grouped updates and `open-pull-requests-limit: 10`
    or more.
  - Use no `ignore:` entries.
  - Cover every ecosystem in the repo (`cargo`, `pub`, `uv`,
    `gitsubmodule`, …).
  - Also cover `github-actions`, including `/.github/actions/*` for
    composite actions.
- **Never commit secrets.** Secret values are set out of band
  (`gh secret set`), never in code.

## Repository settings

Settings are applied by hand for now. They will move to code in a
`flowtoys-inc/github-terraform-flowtoys` repo (OpenTofu), modeled on
`ayanich/github-terraform-ayanich`; until it exists, make these settings
with the GitHub UI or `gh` and record any change in the PR that needs it:

- squash merge only, with the PR title as the commit message
- delete head branches on merge
- allow GitHub Actions to create pull requests (release-please needs it)
- Dependabot security updates on

Once that repo exists, settings change only through it.

## Git, versioning and releases

- **Never commit directly to the default branch.** New repos use `main`;
  existing repos keep theirs (some use `master`).
  - Branch as `feat/…`, `fix/…`, `chore/…`, `docs/…`, `ci/…` or `build/…`.
  - Open a PR and **squash-merge** it.
- **PR titles and commit messages follow Conventional Commits:** `feat`,
  `fix`, `perf`, `refactor`, `docs`, `test`, `build`, `ci`, `chore`,
  `revert`. Mark breaking changes with `!` or a `BREAKING CHANGE:` footer.
  The squash commit takes the PR title, and release-please reads it.
- **Use semantic versioning, with one version per repo,** managed by
  release-please.
  - Merging the release PR tags `vX.Y.Z`, updates `CHANGELOG.md` and
    publishes a GitHub Release.
  - **Never hand-edit** `CHANGELOG.md`, `.release-please-manifest.json`
    or the version files release-please owns.
  - Before 1.0: `feat` bumps the minor version, `fix` the patch, and a
    breaking change the minor.
- **Release PRs don't get CI.** release-please opens them with
  `GITHUB_TOKEN`, so no workflows run on them. They only touch version
  files and `CHANGELOG.md`.
- **Ask before outward-facing actions:** merging, pushing to the default
  branch, publishing releases, or changing repo or org settings.

## Documentation

- **Write for people, not only agents.** Outside collaborators work
  without Claude, so every repo's `README.md` alone is enough to build,
  test and contribute. Contribution rules live in
  [`CONTRIBUTING.md`](CONTRIBUTING.md) here, and repos link to it.
- **Keep these up to date in the same PR as the change:**
  - `README.md`: what the repo is, plus a quickstart
  - development workflow and commands, in `docs/` or the README
  - `CLAUDE.md` / `AGENTS.md`: project-specific rules
- **Record facts learned from real hardware and systems** (devices,
  radios, SDK quirks) in `docs/`, with where and on what each was
  observed.
