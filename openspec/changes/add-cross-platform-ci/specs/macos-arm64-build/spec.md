## ADDED Requirements

<!--
  NOTE TO IMPLEMENTORS — KEY ARCHITECTURAL DECISIONS FOR MACOS:

  1. WHY macos-14 NOT macos-15:
     Carbon framework was REMOVED from the macOS 15 SDK. Two source files
     (#include <Carbon/Carbon.h>) and the CMakeLists.txt (${FRAMEWORK_CARBON})
     depend on it. Using macos-14 (Sonoma, Apple Silicon arm64) preserves
     Carbon headers without modifying any code. The compiled binary runs on
     macOS 15+ because Carbon.dylib is still present at RUNTIME.

  2. WHY BREW NOT AQTINSTALL:
     Homebrew on macOS runners provides pre-built Qt6 for Apple Silicon.
     /opt/homebrew/opt/qt@6 is a keg-only install — must set
     CMAKE_PREFIX_PATH explicitly. No need to download gigabytes of Qt
     binaries from CDN when brew already has them cached on the runner.

  3. WHY NO CMAKE_OSX_ARCHITECTURES FLAG:
     The runner IS arm64. CMake auto-detects the host architecture.
     Setting CMAKE_OSX_ARCHITECTURES="arm64" explicitly would be
     redundant and could conflict with CMake's auto-detection.
-->

### Requirement: Native Apple Silicon build
The macOS build SHALL run on a `macos-14` runner and produce a native arm64 (Apple Silicon) Mach-O binary in a `.app` bundle packaged as a DMG.

#### Scenario: Architecture verification
- **WHEN** the build completes on the `macos-14` runner
- **THEN** the resulting `RockboxUtility` executable is an arm64 Mach-O binary (verifiable with `file` or `lipo -info`)

#### Scenario: App bundle structure
- **WHEN** `cmake --build build --target deploy` completes
- **THEN** a valid `.app` bundle is produced with `Contents/MacOS/RockboxUtility`, `Contents/Info.plist`, and `Contents/Resources/`

#### Scenario: DMG output
- **WHEN** the deploy target completes
- **THEN** a `RockboxUtility.dmg` file is produced using `dmgbuild` with the configuration from `utils/rbutilqt/dmgbuild.cfg`

### Requirement: Carbon framework compatibility via macos-14
The macOS build SHALL run on `macos-14` (Sonoma) to ensure the Carbon framework headers are available during compilation, since Carbon was removed from the macOS 15 SDK.

<!--
  CARBON DEPENDENCY DETAIL:
  - utils/rbutilqt/base/ttscarbon.cpp → Carbon Speech Synthesis API
  - utils/rbutilqt/base/utils.cpp      → Carbon utility functions
  - utils/CMakeLists.txt:274           → ${FRAMEWORK_CARBON} linked to rbbase
  - If Apple removes Carbon.dylib at RUNTIME in macOS 16+, app crashes.
    Mitigation: future spec to replace TTS Carbon with AVSpeechSynthesizer.
-->

#### Scenario: Carbon framework found by CMake
- **WHEN** CMake configure runs on `macos-14`
- **THEN** `find_library(FRAMEWORK_CARBON Carbon)` succeeds and the `rbbase` library links against Carbon

#### Scenario: TTS Carbon source compiles
- **WHEN** the build runs on `macos-14`
- **THEN** `utils/rbutilqt/base/ttscarbon.cpp` compiles successfully with `#include <Carbon/Carbon.h>`

#### Scenario: Backward compatibility with macOS 15+
- **WHEN** a user downloads and runs the DMG on macOS 15 Sequoia
- **THEN** the application launches and the Carbon-based TTS functionality is available at runtime (Carbon.dylib is present on macOS 15 at runtime despite being removed from the SDK)

### Requirement: Apple framework linking
The macOS build SHALL link against all required Apple frameworks: IOKit, CoreFoundation, Carbon, SystemConfiguration, and CoreServices.

#### Scenario: IOKit framework for USB device access
- **WHEN** the build completes
- **THEN** the `rbbase` library is linked against `IOKit.framework` for iPod USB communication

#### Scenario: CoreFoundation framework
- **WHEN** the build completes
- **THEN** the `rbbase` library is linked against `CoreFoundation.framework`

### Requirement: Qt6 installed via Homebrew
The macOS build SHALL install Qt6 using `brew install qt@6` and set `CMAKE_PREFIX_PATH=/opt/homebrew/opt/qt@6`.

<!--
  BREW KEG-ONLY NOTE: Homebrew installs qt@6 as "keg-only" — it's not
  symlinked into /usr/local or /opt/homebrew/bin. CMake's find_package
  will NOT find it unless CMAKE_PREFIX_PATH is explicitly set to
  /opt/homebrew/opt/qt@6. The workflow must set this in $GITHUB_ENV
  before the CMake configure step.
-->

#### Scenario: Homebrew Qt6 installation
- **WHEN** the macOS job starts
- **THEN** `brew install qt@6` completes successfully and Qt6 binaries are available under `/opt/homebrew/opt/qt@6/bin/`

#### Scenario: Qt6 takes precedence over Qt5
- **WHEN** CMake configure runs with `CMAKE_PREFIX_PATH=/opt/homebrew/opt/qt@6`
- **THEN** `find_package(QT NAMES Qt6 Qt5 REQUIRED)` discovers Qt6 first and sets `QT_VERSION_MAJOR=6`

### Requirement: macOS packaging tools
The macOS build SHALL use `macdeployqt` (from the Qt6 installation) and `dmgbuild` (Python, installed in a virtualenv by `deploy.cmake`) to produce the final DMG.

#### Scenario: macdeployqt bundles Qt frameworks
- **WHEN** the deploy target runs `macdeployqt RockboxUtility.app`
- **THEN** the `.app` bundle contains all required Qt frameworks inside its `Contents/Frameworks/` directory

#### Scenario: dmgbuild creates DMG
- **WHEN** the deploy target runs `dmgbuild -s dmgbuild.cfg`
- **THEN** a valid DMG file is created that can be opened on any Apple Silicon Mac
