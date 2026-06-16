## Context

Rockbox Utility (`utils/rbutilqt/`) is a Qt-based C++ desktop application built with CMake 3.12+. The build system already supports macOS (`.dmg`), Linux (`.AppImage`), and Windows (`.zip`) packaging via the custom `deploy.cmake` module. All extraneous libraries (quazip, mspack, bz2, ucl, rbspeex, tomcrypt) are bundled. External dependencies are limited to Qt6, platform system libraries, zlib, and bzip2.

There are zero existing CI/CD workflows. The codebase is a fork — minimal diff is critical to enable clean rebase from upstream.

The application version is `1.5.2`. `Info.plist`, `dmgbuild.cfg`, and `RockboxUtility.desktop` already contain correct metadata.

## Goals / Non-Goals

**Goals:**
- Produce downloadable build artifacts for all five target platforms on every push/PR
- Publish a stable GitHub Release ("Latest Rockbox Utility Builds") that always contains the freshest binaries
- Run all builds in parallel; release upsert runs after all builds complete
- Use GitHub Actions free tier (public repos: unlimited)
- Add zero modifications to existing source code or build system files
- Single workflow file for maintainability

**Non-Goals:**
- Code signing, notarization, or certificate management (future spec)
- Docker-based local builds (future spec)
- Unit test execution in CI (future spec)
- Fixing the Carbon dependency for macOS 15+ SDK (future spec)
- Cross-compilation (all builds are native on their respective runner OS)
- Uploading to package registries (brew, chocolatey, apt)

## Decisions

<!--
  NOTE TO IMPLEMENTORS: Each decision below is annotated with the rationale
  that should be preserved as YAML comments in the workflow file. When writing
  build-rbutil.yml, include a comment block at the top explaining the key
  architectural choices so future maintainers understand WHY, not just WHAT.
-->

### Decision 1: Native builds on each platform vs Docker cross-compilation

**Chosen: Native builds on each platform via GitHub-hosted runners.**

```yaml
# DESIGN DECISION: Native builds, not Docker cross-compilation
# ------------------------------------------------------------
# Docker images exist (stateoftheartio/qt6:6.8-macos-aqt for osxcross,
# stateoftheartio/qt6:6.8-mingw-aqt for Windows) but cross-compiled
# macOS binaries have known limitations:
#   - macdeployqt is crippled (no hdiutil, no proper DMG)
#   - codesign is unavailable
#   - IOKit USB device access may fail at runtime
#   - Carbon framework headers may be missing from osxcross SDK
# Native builds on GitHub's macOS/Windows/Linux runners produce correct,
# fully-functional binaries with proper .app bundles, DMGs, and ZIPs.
# Trade-off: 5 separate runner types instead of 1 Docker host.
```

### Decision 2: macOS runner — macos-14 (not macos-15)

**Chosen: `macos-14` (Apple Silicon arm64, macOS Sonoma).**

```yaml
# DESIGN DECISION: macos-14, NOT macos-15
# -----------------------------------------
# The Carbon framework was DEPRECATED in macOS 10.8 (2012) and REMOVED
# from the macOS 15 SDK. Two source files require Carbon:
#   utils/rbutilqt/base/ttscarbon.cpp:27  → #include <Carbon/Carbon.h>
#   utils/rbutilqt/base/utils.cpp:58      → #include <Carbon/Carbon.h>
# The CMakeLists.txt links:  ${FRAMEWORK_CARBON}
#
# macos-13 = Intel x64, has Carbon → NO (user needs arm64)
# macos-14 = Apple Silicon arm64, has Carbon → YES
# macos-15 = Apple Silicon arm64, Carbon REMOVED → build fails
#
# Binaries compiled against macOS 14 SDK run on macOS 15+ (backward compat).
# Carbon.dylib is still present at RUNTIME on macOS 15, just not in the SDK.
# When Apple eventually removes Carbon.dylib at runtime, we'll need a
# follow-up spec to replace TTS Carbon with AVSpeechSynthesizer.
```

### Decision 3: Qt installation method per platform

**Chosen: apt on Linux, brew on macOS, aqtinstall on Windows.**

```yaml
# DESIGN DECISION: Platform-native Qt installation
# --------------------------------------------------
# Linux:   apt install qt6-*     → Ubuntu 24.04 repos have full Qt6 for
#                                  both x64 and arm64. No PPA needed.
# macOS:   brew install qt@6     → Homebrew provides pre-built Qt6 for
#                                  Apple Silicon. Keg-only install at
#                                  /opt/homebrew/opt/qt@6 — must set
#                                  CMAKE_PREFIX_PATH explicitly.
# Windows: aqtinstall (pip)      → Chocolatey Qt packages are outdated.
#                                  aqtinstall pulls official Qt binaries
#                                  directly from Qt CDN at pinned versions.
#                                  Qt 6.8.0 chosen because it's the last
#                                  version with win64_msvc2022_arm64 target.
#                                  6.10 dropped ARM64 Windows Qt binaries.
#
# REJECTED: jurplel/install-qt-action
# Adds external action dependency with opaque version mapping and caching.
# Direct commands are simpler, faster, and fully transparent.
```

### Decision 4: Qt version pinning

```yaml
# DESIGN DECISION: Qt version strategy
# -------------------------------------
# Linux:   Whatever Ubuntu 24.04 ships (~Qt 6.4 to 6.7 depending on apt)
#          Both x64 and arm64 repos have matching versions.
# macOS:   Whatever Homebrew ships (~Qt 6.8, updated frequently)
# Windows: PINNED to Qt 6.8.0 via aqtinstall
#          win64_msvc2022_arm64 target ONLY exists in Qt 6.8.x.
#          Qt 6.10 dropped ARM64 Windows prebuilt binaries.
#          x64 uses win64_msvc2022_64 at the same 6.8.0 version.
#
# TRADE-OFF: Different Qt versions across platforms. Acceptable because
# the CMakeLists.txt supports both Qt5 and Qt6, and Qt6 minor versions
# are API-compatible. If a version-specific bug appears, Linux/macOS
# can be pinned to aqtinstall too.
```

### Decision 5: Windows ARM64 runner — status and fallback

```yaml
# DESIGN DECISION: Windows ARM64 runner (windows-11-arm)
# ---------------------------------------------------------
# As of June 2026, GitHub completed the transition of ARM64 runner images
# from Arm Limited, LLC to GitHub-managed pipelines (ref: issue #14100).
# The image exists at actions/runner-images/images/windows/Windows11-Arm64
# and includes VS 2022 Enterprise, CMake 4.3.3, and full MSVC ARM64 tools.
#
# UNCERTAINTY: The label 'windows-11-arm' is NOT listed in the standard
# runner table in the README (only windows-2025, windows-2022 are listed).
# It may be available only as a "larger runner" (paid tier).
#
# MITIGATION: continue-on-error: true on this matrix entry.
# If the runner is free:   5-platform build, Windows ARM64 zip produced.
# If the runner is paid:   Job fails silently, 4-platform build succeeds,
#                          workflow passes. No user impact.
#
# TO VERIFY: Push the workflow and check if windows-11-arm provisions
# on a free-tier public repo. If it does, remove continue-on-error.
```

### Decision 6: Release upsert — "Latest Rockbox Utility Builds"

**Chosen: Delete-and-recreate using `gh release` CLI executed on `ubuntu-latest`.**

```yaml
# DESIGN DECISION: Release upsert strategy
# -----------------------------------------
# GOAL: Always-visible, single-URL, always-latest downloadable binaries.
#
# APPROACH: Delete old release + create new release (not update-in-place)
#   WHY DELETE+RECREATE: GitHub Releases API doesn't support "upsert".
#   You can update a release's metadata but NOT replace its assets
#   atomically. Deleting and recreating ensures exactly one release
#   with exactly the current set of binaries.
#
# ALTERNATIVES CONSIDERED:
#   1. softprops/action-gh-release@v2: Can create/update release but
#      can't atomically replace ALL assets. Old assets accumulate.
#      Risk of users downloading stale binaries.
#   2. gh release upload --clobber: Uploads new assets but requires
#      deleting old ones individually via API. More API calls, more
#      failure points.
#   3. Nightly.link: External service. Adds dependency. Not needed.
#
# CHOSEN: gh release delete latest-builds --yes --cleanup-tag
#         gh release create latest-builds --title "Latest ..." --prerelease
#         gh release upload latest-builds *.zip *.dmg *.AppImage
#
# WHY --prerelease: Marks the release as "pre-release" so users know
# these are CI builds, not official versioned releases. The tag
# 'latest-builds' is stable — never changes — so the URL is permanent.
#
# WHY ubuntu-latest (not the build runners): Lightweight shell job.
# gh CLI is pre-installed on all runners. No compilation needed.
#
# PERMISSION: Requires 'contents: write' at workflow level for
# release creation/deletion.
#
# CONCURRENCY: The release job is NOT in the build matrix — it runs
# once, after all builds complete (needs: build). This avoids race
# conditions from parallel release mutations.
```

### Decision 7: Build system generator

**Chosen: Ninja.**

```yaml
# DESIGN DECISION: Ninja generator (not Unix Makefiles, not MSBuild)
# -------------------------------------------------------------------
# Ninja is consistently available across all runners:
#   Linux:   apt install ninja-build
#   macOS:   brew install ninja (or preinstalled)
#   Windows: Comes with Visual Studio (preinstalled on all Windows runners)
# Ninja provides faster parallel builds than Make and simpler cross-platform
# command syntax than MSBuild.
```

### Decision 8: Artifact naming and release

```yaml
# DESIGN DECISION: Artifact naming convention
# --------------------------------------------
# Workflow artifacts:  RockboxUtility-{label}-{sha}.{ext}
#   → Per-commit uniqueness. 90-day retention on Actions tab.
#
# Release assets:       RockboxUtility-{label}.{ext}
#   → Clean, stable names on the "Latest Builds" release page.
#   → No SHA in release asset name (users download from one URL).
#   → SHA is embedded IN the binary via gitversion.cmake.
#
# Release name:         "Latest Rockbox Utility Builds"
# Release tag:          latest-builds
# Release body:         Auto-generated with commit SHA, build date,
#                       and platform list.
```

### Decision 9: Trigger paths filtering

**Chosen: Include `utils/**`, `tools/**`, `lib/**`, `.github/workflows/build-rbutil.yml`.**

```yaml
# DESIGN DECISION: Path-based trigger filtering
# ----------------------------------------------
# The Rockbox repository contains firmware, bootloader, manual, docs,
# and other directories unrelated to Rockbox Utility. Path filtering
# prevents CI from triggering on firmware-only changes, conserving
# CI minutes and avoiding false "build failure" notifications.
#
# Included paths:
#   utils/**    → rbutilqt source, all static libs, CMake files, deploy scripts
#   tools/**    → Shared tools (voicefont, telechips, iriver, mkboot, etc.)
#   lib/**      → libspeex codec, skin_parser library
#   .github/... → The workflow file itself (so CI changes are tested)
#
# NOT included (no CI trigger):
#   firmware/, bootloader/, apps/, docs/, manual/, fonts/, icons/, wps/
```

### Decision 10: Full git clone required

```yaml
# DESIGN DECISION: fetch-depth: 0 (full clone)
# --------------------------------------------
# utils/cmake/gitversion.cmake uses:
#   git rev-parse --verify --short=10 HEAD
#   git diff --quiet --exit-code
# These commands require full git history. A shallow clone (default
# fetch-depth: 1) would produce "N/A" as the version string.
# Full clone adds ~10-15 seconds to checkout time but embeds the real
# commit hash in the binary — critical for traceability.
```

## Risks / Trade-offs

<!--
  NOTE TO IMPLEMENTORS: When you encounter issues during implementation,
  check this table first. Each risk has a pre-planned mitigation.
-->

| Risk | Severity | Mitigation |
|------|----------|------------|
| `windows-11-arm` runner unavailable on free tier | Medium | `continue-on-error: true` — workflow succeeds with 4 platforms. Remove the flag if runner provisions on free tier. |
| Carbon.dylib removed at runtime from macOS 16+ | Medium | Binary compiled against macOS 14 SDK links Carbon dynamically. If Apple removes the dylib, app crashes on launch. Future spec: replace TTS Carbon with AVSpeechSynthesizer. |
| `macdeployqt` fails because brew Qt is keg-only | Medium | Explicit `CMAKE_PREFIX_PATH=/opt/homebrew/opt/qt@6` ensures CMake and macdeployqt are found. Verify in CI. |
| `linuxdeploy` download fails during CMake configure | Low | The `download.cmake` script handles failures gracefully. Network issues are transient; retry the job. |
| Race condition on release upsert (two pushes in rapid succession) | Low | `concurrency` group by `github.ref` ensures only one workflow runs per branch at a time. The release job `needs: build` — all builds complete before release mutation. |
| Qt 6.8 on Windows goes EOL / security patches needed | Low | Qt is a build-time dependency, not runtime. Version can be bumped in the workflow YAML. If ARM64 Windows Qt binaries are dropped entirely, the matrix entry can be removed or switched to x64 emulation. |
| Linux apt Qt6 version mismatch between x64 and arm64 | Low | Both architectures pull from the same Ubuntu 24.04 repository. CMake's `find_package` is flexible; minor version differences are handled by Qt's backward compatibility. |
| Release deletion fails because `latest-builds` release doesn't exist (first run) | Low | `gh release delete` with `|| true` fallback. The `create` step always runs. |
| Ninja duplicate rule error (`dmgbuild.stamp`) from two `deploy_qt()` calls on macOS | High | **Workaround:** Bypass cmake deploy entirely on macOS. Build only `--target RockboxUtility`, then manually run `macdeployqt` + `dmgbuild`. This avoids the ninja conflict in the generated build file. See workflow comments for details. |
| `linuxdeploy-x86_64.AppImage` exec format error on ARM64 | High | **Workaround:** Pre-download the aarch64 linuxdeploy binaries before cmake configure, placed under the x86_64 filename. The `download.cmake` script checks `if(EXISTS)` before downloading — pre-placed files are accepted. |
| `Qt6Core5Compat` not found in aqt base install | Medium | **Fix:** Add `-m qt5compat -m qtsvg -m qttools` to aqt install-qt command. Core5Compat, SvgWidgets, and LinguistTools are add-on modules not in the default package. |
| `qt_base` not found for `win64_msvc2022_arm64` architecture | Medium | **Workaround:** Install x64 Qt first (`win64_msvc2022_64` with all modules), then arm64 on top (`win64_msvc2022_arm64` without modules). ARM64 Qt for Windows requires the x64 base install for host tools (moc, rcc). If this still fails, the `continue-on-error` flag on the ARM64 matrix entry handles it gracefully. |

## Open Questions

- **Is `windows-11-arm` free for public repos?** GitHub issue #14100 confirms the image is now GitHub-managed (not partner-maintained), suggesting standard-runner status. But it's absent from the standard-runner table in the README. **Verify on first push** — if it provisions, remove `continue-on-error`.
- **Should we add a `release` event trigger for versioned releases?** A future spec could add `on.release.types: [published]` that creates a proper versioned release (e.g., `v1.5.2`) in addition to the rolling `latest-builds`.
- **Should unit tests run in CI?** The five test targets require Qt Test runtime. A future spec could add a test job after build.
