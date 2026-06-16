## 1. Scaffold

- [x] 1.1 Create `.github/workflows/` directory if it does not exist
- [x] 1.2 Create `.github/workflows/build-rbutil.yml` with:
  - Workflow `name: Build Rockbox Utility`
  - `permissions.contents: write` (required for release creation/deletion)
  - Header comment block explaining all architectural decisions (copy from design.md code-comment blocks)

## 2. Workflow Triggers and Concurrency

- [x] 2.1 Configure `on.push` with `branches: [main, master]` and `paths` filter:
  - `utils/**`, `tools/**`, `lib/**`, `.github/workflows/build-rbutil.yml`
- [x] 2.2 Configure `on.pull_request` with same `paths` filter
- [x] 2.3 Configure `on.workflow_dispatch` for manual triggering
- [x] 2.4 Set `concurrency` group to `${{ github.workflow }}-${{ github.ref }}` with `cancel-in-progress: true`

## 3. Build Matrix

- [x] 3.1 Define `jobs.build` with `strategy.fail-fast: false` and `matrix.include`:
  ```yaml
  - { os: ubuntu-24.04,      arch: x64,   ext: AppImage, label: linux-x64 }
  - { os: ubuntu-24.04-arm,  arch: arm64, ext: AppImage, label: linux-arm64 }
  - { os: macos-14,          arch: arm64, ext: dmg,      label: macos-arm64 }
  - { os: windows-2025,      arch: x64,   ext: zip,      label: windows-x64 }
  - { os: windows-11-arm,    arch: arm64, ext: zip,      label: windows-arm64,
      continue-on-error: true }
  ```
- [x] 3.2 Set `runs-on: ${{ matrix.os }}`
- [x] 3.3 Set `continue-on-error: ${{ matrix.continue-on-error || false }}`

## 4. Checkout Step

- [x] 4.1 Add `actions/checkout@v4` with `fetch-depth: 0` (required for `gitversion.cmake`)

## 5. Linux Build Steps (x64 and arm64)

- [x] 5.1 Guard all Linux steps with `if: runner.os == 'Linux'`
- [x] 5.2 Install Qt6 and deps:
  ```bash
  sudo apt update
  sudo apt install -y qt6-base-dev qt6-tools-dev qt6-svg-dev \
    qt6-multimedia-dev qt6-5compat-dev libqt6svgwidgets6 \
    libgl1-mesa-dev libusb-1.0-0-dev ninja-build
  ```
- [x] 5.3 CMake configure (no `CMAKE_PREFIX_PATH` needed — apt installs to system path):
  ```bash
  cmake -S utils -B build -DCMAKE_BUILD_TYPE=Release -G Ninja
  ```
- [x] 5.4 CMake build: `cmake --build build --parallel`
- [x] 5.5 CMake deploy: `cmake --build build --target deploy`
- [x] 5.6 Upload workflow artifact with unique name:
  ```yaml
  - uses: actions/upload-artifact@v4
    with:
      name: RockboxUtility-${{ matrix.label }}-${{ github.sha }}
      path: build/RockboxUtility.AppImage
      if-no-files-found: error
  ```

## 6. macOS Build Steps (arm64)

- [x] 6.1 Guard all macOS steps with `if: runner.os == 'macOS'`
- [x] 6.2 Install Qt6 and Ninja: `brew install qt@6 ninja`
- [x] 6.3 Set `QT_PATH` env var: `echo "QT_PATH=/opt/homebrew/opt/qt@6" >> $GITHUB_ENV`
- [x] 6.4 CMake configure:
  ```bash
  cmake -S utils -B build -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_PREFIX_PATH="${{ env.QT_PATH }}" -G Ninja
  ```
- [x] 6.5 CMake build: `cmake --build build --parallel`
- [x] 6.6 CMake deploy: `cmake --build build --target deploy`
- [x] 6.7 Verify the DMG was produced: `ls -la build/RockboxUtility.dmg`
- [x] 6.8 Upload workflow artifact with unique name and `if-no-files-found: error`

## 7. Windows Build Steps (x64 and arm64)

- [x] 7.1 Guard all Windows steps with `if: runner.os == 'Windows'`
- [x] 7.2 Install aqtinstall: `pip install aqtinstall`
- [x] 7.3 Install Qt6 via aqt with architecture-aware command:
  ```powershell
  if ("${{ matrix.arch }}" -eq "arm64") {
    aqt install-qt windows desktop 6.8.0 win64_msvc2022_arm64 -O $env:RUNNER_TEMP\Qt
    echo "QT_PATH=$env:RUNNER_TEMP\Qt\6.8.0\msvc2022_arm64" >> $env:GITHUB_ENV
  } else {
    aqt install-qt windows desktop 6.8.0 win64_msvc2022_64 -O $env:RUNNER_TEMP\Qt
    echo "QT_PATH=$env:RUNNER_TEMP\Qt\6.8.0\msvc2022_64" >> $env:GITHUB_ENV
  }
  ```
- [x] 7.4 CMake configure:
  ```bash
  cmake -S utils -B build -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_PREFIX_PATH="${{ env.QT_PATH }}" -G Ninja
  ```
- [x] 7.5 CMake build: `cmake --build build --parallel`
- [x] 7.6 CMake deploy: `cmake --build build --target deploy`
- [x] 7.7 Upload workflow artifact with unique name and `if-no-files-found: error`
  - Path should use glob: `build/RockboxUtility*.zip`

## 8. Release Upsert Job

- [x] 8.1 Define `jobs.release` with:
  - `runs-on: ubuntu-latest`
  - `needs: build` (waits for ALL build matrix jobs)
  - `if: success()` (only runs if all non-continue-on-error builds passed)
  - `permissions.contents: write`
- [x] 8.2 Download all build artifacts using `actions/download-artifact@v4`:
  ```yaml
  - uses: actions/download-artifact@v4
    with:
      pattern: RockboxUtility-*
      path: release-artifacts
      merge-multiple: true
  ```
- [x] 8.3 Rename downloaded files to stable names (strip commit SHA):
  ```bash
  # Example: RockboxUtility-linux-x64-abc1234.AppImage → RockboxUtility-linux-x64.AppImage
  for f in release-artifacts/*; do
    newname=$(echo "$f" | sed -E 's/-[0-9a-f]{7,40}//')
    mv "$f" "$newname"
  done
  ```
- [x] 8.4 Delete existing release (if any):
  ```bash
  gh release delete latest-builds --yes --cleanup-tag || true
  ```
  - `--cleanup-tag` removes the tag so it can be reused
  - `|| true` prevents failure on first run when no release exists
- [x] 8.5 Create new release:
  ```bash
  gh release create latest-builds \
    --title "Latest Rockbox Utility Builds" \
    --notes "$(cat <<EOF
  Automated CI build from commit [${{ github.sha }}](https://github.com/${{ github.repository }}/commit/${{ github.sha }})
  
  **Build Date:** $(date -u +"%Y-%m-%d %H:%M UTC")
  **Branch:** ${{ github.ref_name }}
  
  | Platform | Architecture | Download |
  |----------|-------------|----------|
  | Linux | x64 | RockboxUtility-linux-x64.AppImage |
  | Linux | arm64 | RockboxUtility-linux-arm64.AppImage |
  | macOS | arm64 | RockboxUtility-macos-arm64.dmg |
  | Windows | x64 | RockboxUtility-windows-x64.zip |
  | Windows | arm64 | RockboxUtility-windows-arm64.zip |
  
  > These are automated CI builds. They are not signed or notarized.
  > For official releases, see the [Releases page](https://github.com/${{ github.repository }}/releases).
  EOF
  )" \
    --prerelease \
    release-artifacts/*
  ```
  - `--prerelease` marks as pre-release (CI builds, not official)
  - `release-artifacts/*` attaches all renamed binaries
- [x] 8.6 (Optional) Add a summary comment on the commit/PR with download links using `actions/github-script@v7`

## 9. .gitignore Update

- [x] 9.1 Add `build/` and `build-*/` to `.gitignore` if not already present

## 10. First-Run Validation

- [ ] 10.1 Push the workflow to `main` (or a test branch) and verify all jobs trigger
- [ ] 10.2 Check Windows ARM64 job — does `windows-11-arm` provision on free tier?
  - If YES: Remove `continue-on-error: true` from the matrix entry
  - If NO: The job fails gracefully, 4-platform build continues working
- [ ] 10.3 Verify the "Latest Rockbox Utility Builds" release is created and has correct assets
- [ ] 10.4 Push a second commit and verify the release is updated (old release deleted, new one created)
- [ ] 10.5 Verify `workflow_dispatch` manual trigger works from the Actions tab
- [ ] 10.6 Verify path filtering: push a change only to `firmware/` — workflow does NOT trigger
- [ ] 10.7 Download each binary from the release and smoke-test:
  - macOS DMG: Mounts and app launches on Apple Silicon Mac
  - Linux x64 AppImage: Executes on x64 Linux
  - Linux arm64 AppImage: Executes on ARM64 Linux (e.g., Raspberry Pi 5)
  - Windows x64 ZIP: Extracts and exe runs on x64 Windows
  - Windows arm64 ZIP: Extracts and exe runs on ARM64 Windows (e.g., Surface Pro 11) — if available
- [ ] 10.8 Verify git version string embedded in each binary (strings binary | grep -E '[0-9a-f]{7,40}')

## 11. Documentation

- [x] 11.1 ~~Add a note in `utils/rbutilqt/INSTALL` referencing the CI-built binaries and the "Latest Rockbox Utility Builds" release URL~~ (DECISION: skipped — preserve zero source changes for clean rebase)
