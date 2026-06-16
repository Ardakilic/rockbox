## ADDED Requirements

<!--
  NOTE TO IMPLEMENTORS — KEY ARCHITECTURAL DECISIONS FOR LINUX ARM64:

  1. WHY ubuntu-24.04-arm (NOT CROSS-COMPILE):
     ubuntu-24.04-arm is a native ARM64 runner. Qt6 packages exist in the
     standard Ubuntu 24.04 apt repositories for ARM64 — no PPA or third-party
     repo needed. Native compilation is faster and more reliable than
     cross-compiling from x64 with qemu-user-static or a sysroot.

  2. WHY APT (NOT AQTINSTALL):
     Ubuntu 24.04 provides complete Qt6 development packages for ARM64.
     The packages are maintained by Canonical and receive security updates.
     No need to download gigabytes from Qt CDN when apt already has them.

  3. WHY NO CMAKE_TOOLCHAIN_FILE:
     The runner IS arm64. CMake auto-detects the host architecture and
     compiler. No cross-compilation toolchain file is needed.

  4. ARCHITECTURE PARITY WITH X64:
     Both ubuntu-24.04 (x64) and ubuntu-24.04-arm (arm64) use the same
     Ubuntu version and apt repositories. The Qt version, system libraries,
     and build tools are identical — only the CPU architecture differs.
     This maximizes consistency between the two Linux builds.
-->

### Requirement: Native ARM64 Linux build
The Linux ARM64 build SHALL run on an `ubuntu-24.04-arm` runner and produce a native arm64 (aarch64) ELF binary packaged as an AppImage.

#### Scenario: Architecture verification
- **WHEN** the build completes on the `ubuntu-24.04-arm` runner
- **THEN** the resulting `RockboxUtility` executable is an aarch64 ELF binary (verifiable with `file`)

#### Scenario: AppImage output
- **WHEN** `cmake --build build --target deploy` completes
- **THEN** a `RockboxUtility.AppImage` file is produced using `linuxdeploy` with `linuxdeploy-plugin-qt`

### Requirement: Qt6 installed via APT
The Linux ARM64 build SHALL install Qt6 using `apt install` from the Ubuntu 24.04 ARM64 repositories.

#### Scenario: APT Qt6 installation
- **WHEN** the Linux ARM64 job starts
- **THEN** `sudo apt install qt6-base-dev qt6-tools-dev qt6-svg-dev qt6-multimedia-dev qt6-5compat-dev libqt6svgwidgets6 libgl1-mesa-dev libusb-1.0-0-dev ninja-build` completes successfully

#### Scenario: ARM64 Qt packages available
- **WHEN** APT resolves Qt6 package dependencies on `ubuntu-24.04-arm`
- **THEN** all Qt6 packages are found and installed from the standard Ubuntu repositories without additional PPAs

### Requirement: Linux system library linking
The Linux ARM64 build SHALL link against `libusb-1.0` (via pkg-config) and system OpenGL libraries.

#### Scenario: libusb found by pkg-config
- **WHEN** CMake configure runs on the ARM64 runner
- **THEN** `pkg_check_modules(libusb QUIET REQUIRED IMPORTED_TARGET libusb-1.0)` succeeds

#### Scenario: OpenGL for Qt Svg
- **WHEN** CMake configure runs
- **THEN** `libgl1-mesa-dev` provides the necessary OpenGL headers for Qt SVG widget rendering

### Requirement: linuxdeploy downloads during CMake configure
The Linux build SHALL download `linuxdeploy` and `linuxdeploy-plugin-qt` AppImages during CMake configure as defined by the existing `deploy.cmake`.

<!--
  LINUXDEPLOY NOTE: The deploy.cmake script downloads linuxdeploy from
  GitHub releases during CMake configure (not at build time). If the
  download fails (network issue), CMake configure fails. This is a
  transient error — retrying the job typically resolves it. The
  linuxdeploy binaries are x86_64 AppImages but they can package
  ARM64 binaries because they only copy files and set up the AppDir
  structure — they don't execute the target binary.
-->

#### Scenario: linuxdeploy downloaded
- **WHEN** CMake configure runs
- **THEN** `linuxdeploy-x86_64.AppImage` is downloaded from GitHub releases to the build directory

#### Scenario: AppImage packaging succeeds
- **WHEN** the deploy target runs
- **THEN** `linuxdeploy` bundles the Qt application and its dependencies into a self-contained AppImage

### Requirement: Build parity with Linux x64
The Linux ARM64 build SHALL produce an AppImage that is functionally identical to the Linux x64 build but compiled for the aarch64 architecture.

#### Scenario: Same source, different architecture
- **WHEN** both Linux x64 and Linux ARM64 builds complete from the same commit
- **THEN** both AppImages contain the same version of Rockbox Utility, differing only in target architecture
