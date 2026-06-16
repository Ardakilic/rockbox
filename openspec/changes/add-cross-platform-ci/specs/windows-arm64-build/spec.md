## ADDED Requirements

<!--
  NOTE TO IMPLEMENTORS — KEY ARCHITECTURAL DECISIONS FOR WINDOWS ARM64:

  1. RUNNER STATUS (June 2026):
     GitHub completed the transition of ARM64 runner images from Arm Limited, LLC
     to GitHub-managed pipelines (ref: actions/runner-images#14100). The
     windows-11-arm image exists with VS 2022 Enterprise, CMake 4.3.3, full
     MSVC ARM64 toolchain. However, the label 'windows-11-arm' is NOT in the
     standard-runner table in the runner-images README. It MAY be a "larger
     runner" (paid tier). continue-on-error: true is set as a safety net.

  2. WHY QT 6.8.0 (NOT 6.10):
     The win64_msvc2022_arm64 target exists in Qt 6.8.x but was DROPPED
     from Qt 6.10. ARM64 Windows prebuilt binaries are not yet provided
     for Qt 6.10. If a future Qt version restores ARM64 Windows support,
     the version pin can be bumped.

  3. WHY AQTINSTALL (NOT CHOCOLATEY):
     Chocolatey Qt packages trail official releases by months and don't
     offer architecture selection. aqtinstall pulls official binaries
     from Qt CDN at pinned versions with full architecture targeting.

  4. WHY NINJA (NOT MSBUILD):
     Ninja works identically across all platforms. MSBuild would require
     different CMake generator flags and platform-specific command syntax.
     Ninja comes preinstalled with Visual Studio on all Windows runners.
-->

### Requirement: Native Windows ARM64 build
The Windows ARM64 build SHALL run on a `windows-11-arm` runner (when available) and produce a native ARM64 PE binary packaged as a ZIP archive.

#### Scenario: Architecture verification
- **WHEN** the build completes on the `windows-11-arm` runner
- **THEN** the resulting `RockboxUtility.exe` is an ARM64 PE binary (verifiable with `dumpbin /HEADERS`)

#### Scenario: ZIP output
- **WHEN** `cmake --build build --target deploy` completes
- **THEN** a `RockboxUtility.zip` file is produced containing the executable and all required Qt DLLs

### Requirement: Qt6 installed via aqtinstall for Windows ARM64
The Windows ARM64 build SHALL install Qt6 using `aqtinstall` with the `win64_msvc2022_arm64` target at version 6.8.0.

<!--
  QT VERSION PIN: 6.8.0 is the LAST Qt release with win64_msvc2022_arm64
  prebuilt binaries. Qt 6.10 dropped this target. If Qt restores ARM64
  Windows binaries in a future release, bump the version. The Qt version
  is a BUILD-TIME dependency — it does not affect runtime compatibility
  of the produced binary with different Qt DLL versions.
-->

#### Scenario: aqtinstall downloads Qt for ARM64
- **WHEN** the Windows ARM64 job runs `aqt install-qt windows desktop 6.8.0 win64_msvc2022_arm64`
- **THEN** Qt6 ARM64 binaries are downloaded to `$env:RUNNER_TEMP\Qt\6.8.0\msvc2022_arm64`

#### Scenario: CMake discovers Qt6 ARM64
- **WHEN** CMake configure runs with `CMAKE_PREFIX_PATH` pointing to the aqtinstall output
- **THEN** `find_package(QT NAMES Qt6 Qt5 REQUIRED)` discovers the ARM64 Qt6 installation

### Requirement: MSVC ARM64 toolchain
The Windows ARM64 build SHALL use the Visual Studio 2022 ARM64 native tools (preinstalled on the `windows-11-arm` runner) to compile the application.

#### Scenario: MSVC ARM64 compiler
- **WHEN** the build runs
- **THEN** the Visual Studio 2022 ARM64 compiler (`cl.exe` targeting ARM64) is used

#### Scenario: Ninja generator with MSVC
- **WHEN** CMake configures with `-G Ninja`
- **THEN** the Ninja generator detects the MSVC ARM64 toolchain from the Visual Studio developer environment

### Requirement: Windows resource files
The Windows ARM64 build SHALL include platform-specific resource files: `rbutilqt-win.qrc` and `rbutilqt.rc`.

#### Scenario: Windows QRC included
- **WHEN** CMake configure runs with `WIN32` defined
- **THEN** `rbutilqt-win.qrc` and `rbutilqt.rc` are added as sources to the `RockboxUtility` target

#### Scenario: Windows system library linking
- **WHEN** the build links `rbbase`
- **THEN** the following Win32 libraries are linked: `setupapi`, `ws2_32`, `netapi32`, `crypt32`, `iphlpapi`

### Requirement: windeployqt bundles Qt runtime
The Windows ARM64 build SHALL use `windeployqt` (from the aqtinstall-installed Qt) to bundle all required Qt DLLs into the deploy directory.

#### Scenario: windeployqt execution
- **WHEN** the deploy target runs
- **THEN** `windeployqt` copies Qt6 Core, Widgets, Svg, Network, and platform plugin DLLs into the deploy folder

#### Scenario: Self-contained ZIP
- **WHEN** the ZIP is created from the deploy folder
- **THEN** the archive contains `RockboxUtility.exe` and all DLLs required to run on a Windows ARM64 device without a separate Qt installation

### Requirement: Graceful fallback when runner unavailable
The Windows ARM64 build SHALL use `continue-on-error: true` in the workflow matrix so that unavailability of the `windows-11-arm` runner does not fail the entire CI run.

<!--
  WHY continue-on-error: true:
  This is a SAFETY NET, not a permanent design choice. The first CI run
  will reveal whether windows-11-arm provisions on the free tier:
  - If YES (job succeeds): Remove continue-on-error from the matrix entry.
    Windows ARM64 becomes a first-class citizen in the matrix.
  - If NO (job skipped/failed): Keep continue-on-error. Windows ARM64
    remains a best-effort bonus platform. Users get the other 4 binaries.
  
  The release job (ci-release-upsert) only requires that non-continue-on-error
  builds succeed. If Windows ARM64 is skipped, the release still publishes
  with 4 platforms — Windows ARM64 is simply absent from the release assets.
-->

#### Scenario: Runner not provisioned
- **WHEN** GitHub cannot provision a `windows-11-arm` runner (e.g., free tier not available)
- **THEN** the Windows ARM64 job is skipped or reports a warning, and the workflow succeeds with the remaining four platforms' artifacts

#### Scenario: Runner provisions successfully
- **WHEN** GitHub provisions a `windows-11-arm` runner on the free tier
- **THEN** the Windows ARM64 ZIP is built, uploaded as a workflow artifact, and included in the release assets
