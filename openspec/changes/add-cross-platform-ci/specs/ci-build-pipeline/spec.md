## ADDED Requirements

<!--
  NOTE TO IMPLEMENTORS: This spec covers the overall workflow structure.
  Platform-specific build requirements are in their respective specs:
    macos-arm64-build/spec.md
    linux-arm64-build/spec.md
    windows-arm64-build/spec.md
  Release management is in: ci-release-upsert/spec.md

  KEY NUANCE — The workflow file should include a header comment block
  explaining every architectural decision. See design.md "Decisions"
  section for the exact YAML comments to embed.
-->

### Requirement: Workflow triggers on code changes
The CI pipeline SHALL automatically trigger on every push and pull request that modifies files under `utils/**`, `tools/**`, `lib/**`, or the workflow file itself.

<!--
  PATH FILTERING RATIONALE: The Rockbox repo contains firmware/, bootloader/,
  docs/, manual/ and other directories unrelated to Rockbox Utility. Without
  path filtering, EVERY commit would trigger 5 parallel builds — wasting CI
  minutes and burning through free-tier limits on private repos.
-->

#### Scenario: Push to a branch with utility source changes
- **WHEN** a commit is pushed that modifies any file under `utils/rbutilqt/`
- **THEN** the CI workflow is queued and all platform builds are initiated

#### Scenario: Pull request with only firmware changes
- **WHEN** a pull request is opened that only modifies files under `firmware/` or `bootloader/`
- **THEN** the CI workflow is NOT triggered

#### Scenario: Workflow file modification
- **WHEN** a commit modifies `.github/workflows/build-rbutil.yml`
- **THEN** the CI workflow is triggered to test the workflow changes

### Requirement: Manual workflow dispatch
The CI pipeline SHALL support manual triggering via the `workflow_dispatch` event for ad-hoc rebuilds without code changes.

#### Scenario: Manual trigger from GitHub UI
- **WHEN** a user clicks "Run workflow" on the Actions tab with the `build-rbutil.yml` workflow selected
- **THEN** the workflow starts with all five platform builds

### Requirement: All platform builds run in parallel
The CI pipeline SHALL execute all platform builds concurrently using a GitHub Actions matrix strategy with `fail-fast: false`.

<!--
  FAIL-FAST: FALSE — If one platform fails (e.g., Windows ARM64 runner
  unavailable), the other builds continue and produce artifacts. Users
  still get 4 out of 5 binaries. The release job (ci-release-upsert)
  only runs if ALL non-continue-on-error builds succeed.
-->

#### Scenario: One platform build fails
- **WHEN** the Windows ARM64 build fails due to runner unavailability
- **THEN** the Linux x64, Linux ARM64, macOS ARM64, and Windows x64 builds continue and complete independently

#### Scenario: All platforms succeed
- **WHEN** all five platform builds complete successfully
- **THEN** five separate artifact archives are uploaded AND the release upsert job begins

### Requirement: Build artifacts are uploaded to workflow
The CI pipeline SHALL upload each platform's build artifact using `actions/upload-artifact@v4` with a unique name including platform label and commit SHA.

<!--
  WORKFLOW ARTIFACTS (vs Release Assets):
  - Workflow artifacts: Per-commit, 90-day retention, SHA in filename
  - Release assets:       Stable names on "Latest Builds" release, no SHA
  Both serve different purposes: workflow artifacts for CI debugging,
  release assets for end-user downloads.
-->

#### Scenario: Artifact name uniqueness
- **WHEN** two commits trigger the workflow
- **THEN** artifacts from each run are stored separately and do not overwrite each other

### Requirement: Platform-specific Qt6 installation
The CI pipeline SHALL install Qt6 using the appropriate package manager for each platform: `apt` on Linux, `brew` on macOS, and `aqtinstall` on Windows.

<!--
  WHY NOT install-qt-action? Direct commands are simpler, faster, and
  transparent. The action adds version-mapping indirection and caching
  semantics that obscure what's happening. The three installation methods
  (apt, brew, aqt) are each 1-3 lines of shell — no action needed.
-->

#### Scenario: Linux runner Qt installation
- **WHEN** the workflow runs on an `ubuntu-*` runner
- **THEN** Qt6 development packages are installed via `sudo apt install` and CMake discovers them automatically (no CMAKE_PREFIX_PATH needed)

#### Scenario: macOS runner Qt installation
- **WHEN** the workflow runs on a `macos-*` runner
- **THEN** Qt6 is installed via `brew install qt@6` and `CMAKE_PREFIX_PATH` is set to `/opt/homebrew/opt/qt@6`

#### Scenario: Windows runner Qt installation
- **WHEN** the workflow runs on a `windows-*` runner
- **THEN** Qt6 is installed via `aqt install-qt` to a temporary directory and `CMAKE_PREFIX_PATH` is set accordingly

### Requirement: Build system uses existing CMake files without modification
The CI pipeline SHALL invoke the existing build system exactly as documented: `cmake -S utils -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=<qt-path> -G Ninja` followed by `cmake --build build --parallel` and `cmake --build build --target deploy`.

<!--
  ZERO CODE CHANGES: This is the core constraint. The fork must remain
  cleanly rebaseable against upstream. All CI logic lives in the
  workflow YAML file. No CMake, no source, no deploy script changes.
  The Carbon framework blocker is solved by runner choice (macos-14),
  not by code modification.
-->

#### Scenario: Deploy target produces platform-specific package
- **WHEN** `cmake --build build --target deploy` completes on macOS
- **THEN** the existing `deploy.cmake` macOS branch runs `macdeployqt` and `dmgbuild` to produce a DMG

### Requirement: Full git history is available for version embedding
The CI pipeline SHALL perform a full git clone (`fetch-depth: 0`) so that `gitversion.cmake` can embed the commit hash into the binary.

<!--
  FULL CLONE RATIONALE: gitversion.cmake runs:
    git rev-parse --verify --short=10 HEAD
    git diff --quiet --exit-code
  A shallow clone (fetch-depth: 1) produces "N/A" as version.
  Full clone adds ~15s to checkout but embeds real SHA — critical
  for knowing exactly which commit produced a binary.
-->

#### Scenario: Version string in built binary
- **WHEN** the build completes
- **THEN** the binary contains a version string derived from the current git commit hash

### Requirement: Concurrency control prevents duplicate runs
The CI pipeline SHALL use a `concurrency` group keyed on `github.ref` to cancel in-progress workflow runs when a new push arrives on the same branch.

<!--
  CONCURRENCY RATIONALE: Without this, rapid pushes create multiple
  overlapping workflows, each with 5 builds + 1 release job = 30 parallel
  VMs. With concurrency, the previous run is cancelled and the new one
  starts — saving resources and preventing release race conditions.
-->

#### Scenario: Rapid successive pushes
- **WHEN** two commits are pushed to the same branch within 30 seconds
- **THEN** the workflow for the first commit is cancelled and only the second workflow runs to completion

### Requirement: Windows ARM64 build is optional
The CI pipeline SHALL include Windows ARM64 in the build matrix with `continue-on-error: true` so that unavailability of the ARM64 Windows runner does not fail the entire workflow.

<!--
  WINDOWS ARM64 STATUS (June 2026):
  - GitHub issue #14100: ARM64 images now fully maintained by GitHub
    (transitioned from Arm Limited, LLC to GitHub's internal pipelines)
  - The windows-11-arm image EXISTS in actions/runner-images repo
    with VS 2022 Enterprise, CMake 4.3.3, MSVC ARM64 tools
  - BUT: The label 'windows-11-arm' is ABSENT from the standard
    runner table in the README (only windows-2025, windows-2022 are listed)
  - It MAY be available only as a "larger runner" (paid tier)
  - Using continue-on-error: true as safety net
  - FIRST-RUN VALIDATION: Check the workflow run logs to see if the
    windows-11-arm job provisions. If it succeeds on a free-tier public
    repo, remove continue-on-error: true from the matrix entry.
-->

#### Scenario: Windows ARM64 runner unavailable
- **WHEN** GitHub cannot provision a `windows-11-arm` runner on the free tier
- **THEN** the workflow reports a warning for that job but the overall workflow succeeds with the other four artifacts

#### Scenario: Windows ARM64 runner available
- **WHEN** GitHub provisions a `windows-11-arm` runner
- **THEN** the Windows ARM64 ZIP artifact is built and uploaded alongside the other four
