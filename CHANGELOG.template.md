# Changelog

All notable changes to this project will be documented in this file.

The format is loosely based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and tailored to the `rust-github-workflows` release flow:

- The `bump-version` workflow looks for the `## [X.Y.Z-SNAPSHOT]` placeholder and rewrites it
  into a versioned section of the form `## [X.Y.Z] - YYYY-MM-DD` followed by a release-type
  sub-heading (one of `### 💥 Major Release`, `### ✨ Minor Release`, `### 🔧 Patch Release`).
- The `open-next-snapshot` workflow automatically bumps to the next snapshot version and
  prepends a new snapshot section so development can immediately resume. If you use
  `open-next-snapshot`, you don't need to manually add the next placeholder. Otherwise,
  manually add a fresh `## [X.Y.Z-SNAPSHOT]` line at the top after each release.

## [X.Y.Z-SNAPSHOT] - Unreleased

### Added

- _Describe new user-facing features here._

### Changed

- _Describe changes to existing behavior here._

### Fixed

- _List bug fixes._
