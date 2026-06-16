## Why

Rockbox Utility (rbutilqt) is the official graphical installer and housekeeping tool for Rockbox firmware. It has a mature CMake-based build system but zero CI/CD automation. There are no GitHub Actions workflows, no automated builds, and no published artifacts. macOS arm64 (Apple Silicon), which now dominates the Mac ecosystem, has never had a build produced. Users must manually compile from source. This change establishes a fully automated, multi-platform CI pipeline that builds Rockbox Utility for five target platforms on every commit, publishes them to a pinned GitHub Release titled "Latest Rockbox Utility Builds" so users always have a single, stable download URL for the freshest binaries.

## What Changes

- Add a single GitHub Actions workflow (`.github/workflows/build-rbutil.yml`) that triggers on every push and pull request touching relevant source paths (`utils/**`, `tools/**`, `lib/**`)
- Build Rockbox Utility for **five platforms in parallel** using GitHub-hosted runners:
  - Linux x64 (`ubuntu-24.04`) — AppImage artifact
  - Linux arm64 (`ubuntu-24.04-arm`) — AppImage artifact
  - macOS arm64 (`macos-14`, Apple Silicon) — DMG artifact
  - Windows x64 (`windows-2025`) — ZIP artifact
  - Windows arm64 (`windows-11-arm`) — ZIP artifact
- After all builds complete, a **release job upserts** a GitHub Release named "Latest Rockbox Utility Builds" — deleting the previous release and its assets, then creating a fresh one with all five platform binaries attached
- The release tag is fixed (`latest-builds`) so the download URL is stable and can be bookmarked or linked from documentation
- Upload all build artifacts as workflow artifacts too (90-day retention on Actions tab, for per-commit access)
- Support manual triggering via `workflow_dispatch`
- No modifications to existing build system, CMake files, or source code — the Carbon framework dependency is resolved by using `macos-14` (Sonoma) which still ships Carbon in its SDK

## Capabilities

### New Capabilities

- `ci-build-pipeline`: Automated multi-platform CI/CD workflow that builds Rockbox Utility on every commit/push/PR across Linux (x64, arm64), macOS (arm64), and Windows (x64, arm64), producing downloadable workflow artifacts AND upserting them into a pinned GitHub Release
- `ci-release-upsert`: After all platform builds succeed, a dependent job deletes any existing "Latest Rockbox Utility Builds" release and creates a fresh one with the new binaries attached, providing a stable download URL
- `macos-arm64-build`: Native Apple Silicon (arm64) build of Rockbox Utility as a DMG, compiled on macOS 14 runners with the Apple Clang toolchain and Carbon framework compatibility
- `linux-arm64-build`: Native arm64 Linux build of Rockbox Utility as an AppImage, compiled on Ubuntu 24.04 ARM runners
- `windows-arm64-build`: Native arm64 Windows build of Rockbox Utility as a ZIP, compiled on Windows 11 ARM runners with MSVC

### Modified Capabilities

<!-- No existing capabilities modified — this is a greenfield addition -->

## Impact

- **New files**: `.github/workflows/build-rbutil.yml` (single workflow file)
- **No existing files modified**: All CMakeLists.txt, source code, deploy scripts, and configuration files remain untouched — zero rebase conflict surface
- **Dependencies**: Qt6 (already supported by the build system), CMake 3.12+ (already required), Ninja build generator. No new code dependencies introduced
- **CI cost**: All standard runners are free for public repositories. Private repos consume ~290 minutes per full matrix run. The `windows-11-arm` runner may consume larger-runner minutes if not provisioned as a standard runner
- **Release permission**: Workflow requires `contents: write` permission to create/delete releases
