## ADDED Requirements

### Requirement: Release job triggers after all builds complete
A separate release job SHALL run after all platform build jobs succeed, using `needs: build` to ensure builds complete first.

#### Scenario: All five builds succeed
- **WHEN** all five platform build jobs complete with status `success`
- **THEN** the release job starts and upserts the "Latest Rockbox Utility Builds" release

#### Scenario: Any build fails
- **WHEN** at least one build job fails
- **THEN** the release job is skipped and no release is created (stale release remains, preserving last-known-good binaries)

#### Scenario: Windows ARM64 build skipped (runner unavailable)
- **WHEN** the Windows ARM64 build is skipped due to `continue-on-error: true` and runner unavailability
- **THEN** the release job still runs (the other 4 platforms built successfully)

### Requirement: Release upsert deletes previous release
The release job SHALL delete any existing release tagged `latest-builds` before creating a new one, ensuring the release page always shows exactly one set of current binaries.

#### Scenario: First workflow run (no existing release)
- **WHEN** no release with tag `latest-builds` exists
- **THEN** the delete command is a no-op (returns success) and the create command succeeds

#### Scenario: Subsequent workflow run (existing release)
- **WHEN** a release with tag `latest-builds` already exists
- **THEN** the existing release and all its assets are deleted before the new release is created

#### Scenario: Release tag is immutable
- **WHEN** the release is deleted and recreated with the same tag `latest-builds`
- **THEN** the tag always points to the latest commit's build artifacts

### Requirement: Release is created with fixed metadata
The new release SHALL be created with:
- Tag: `latest-builds`
- Title: `Latest Rockbox Utility Builds`
- Prerelease flag: `true`
- Body: Auto-generated with commit SHA, build timestamp, and list of included platforms

#### Scenario: Release body contains relevant info
- **WHEN** the release is created
- **THEN** the body includes the full commit SHA, short SHA, build date, and a table of attached binaries with their target platforms

#### Scenario: Release is marked as prerelease
- **WHEN** the release is created
- **THEN** it is marked as a prerelease so users understand these are CI builds, not official versioned releases

### Requirement: All platform binaries are attached to the release
The release job SHALL download all build artifacts from the build jobs and attach them to the release with clean, stable filenames.

#### Scenario: Five binaries attached
- **WHEN** all five platforms built successfully
- **THEN** the release page shows five downloadable assets: `RockboxUtility-linux-x64.AppImage`, `RockboxUtility-linux-arm64.AppImage`, `RockboxUtility-macos-arm64.dmg`, `RockboxUtility-windows-x64.zip`, `RockboxUtility-windows-arm64.zip`

#### Scenario: Release asset filenames are stable
- **WHEN** the release is created
- **THEN** asset filenames do NOT include the commit SHA (stable names for bookmarking)

### Requirement: Release job uses minimal runner
The release job SHALL run on `ubuntu-latest` since it only performs `gh` CLI operations and file manipulation — no compilation.

#### Scenario: Release job uses ubuntu-latest
- **WHEN** the release job runs
- **THEN** it provisions an `ubuntu-latest` runner (x64, free tier)

### Requirement: Workflow has release creation permissions
The workflow SHALL declare `permissions.contents: write` at the job or workflow level to allow release creation and deletion.

#### Scenario: Permission check
- **WHEN** the workflow attempts to delete or create a release
- **THEN** the `GITHUB_TOKEN` has sufficient permissions to perform the operation
