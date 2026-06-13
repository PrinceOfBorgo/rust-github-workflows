# Modular Rust Release Workflow Guide

This guide explains how to use the reusable GitHub Actions workflows for managing releases of **Rust projects**. The workflows are Cargo-based: they read and update `Cargo.toml` (and `Cargo.lock` when present) and use [`cargo set-version`](https://github.com/killercup/cargo-edit) to bump the crate version.

## Prerequisites

- The consumer repository is a Rust/Cargo project (single crate or workspace).
- A `Cargo.toml` exists at the path passed via `cargo_toml_path` (default `Cargo.toml`).
- A `CHANGELOG.md` exists with an unreleased placeholder line such as `## [X.Y.Z-SNAPSHOT]` that the bump workflow rewrites into the new version section. See [`CHANGELOG.template.md`](CHANGELOG.template.md) for a ready-to-copy scaffold.
- The repository default branch is `main`, or `target_branch` is set to the branch the release should be pushed to.

## Architecture Overview

The release process is split into modular reusable workflows:

### 1. **bump-version.yml**
Bumps the Cargo crate version (major/minor/patch) using `cargo set-version` and rewrites the `## [X.Y.Z-SNAPSHOT]` line in `CHANGELOG.md` with the new version and date.

**Inputs:**
- `cargo_toml_path` (optional, default: `Cargo.toml`) - Path to your Cargo.toml file
- `cargo_lock_path` (optional, default: `Cargo.lock`) - Path to Cargo.lock. Set to empty string (`''`) to skip the lockfile (e.g. libraries that do not commit it). Workspaces should point this at the workspace-root lockfile.
- `changelog_path` (optional, default: `CHANGELOG.md`) - Path to your CHANGELOG file
- `release_type` (required) - One of: `major`, `minor`, `patch`

**Outputs:**
- `new_version` - The bumped version
- `changelog_section_heading` - The section heading added to CHANGELOG

**Usage:**
```yaml
uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/bump-version.yml@v1.0.0
with:
  release_type: ${{ env.RELEASE_TYPE }}
```

---

### 2. **finalize-release.yml**
Commits the updated `Cargo.toml`, `Cargo.lock` (when present), and `CHANGELOG.md`, creates a `vX.Y.Z` git tag, and pushes to the target branch.

**Inputs:**
- `version` (required) - Version to release (without `v` prefix)
- `cargo_toml_path` (optional, default: `Cargo.toml`)
- `cargo_lock_path` (optional, default: `Cargo.lock`) - Path to Cargo.lock. Set to empty string (`''`) to skip committing the lockfile.
- `changelog_path` (optional, default: `CHANGELOG.md`)
- `commit_message` (optional, default: `chore: release version`)
- `target_branch` (optional, default: `main`) - Branch to push the release commit and tags to.

**Usage:**
```yaml
- uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/finalize-release.yml@v1.0.0
  with:
    version: ${{ needs.prepare.outputs.new_version }}
```

---

### 3. **create-release.yml**
Creates a GitHub release with auto-generated notes and optional asset attachments.

**Inputs:**
- `version` (required) - Version being released (without `v` prefix)
- `changelog_section_heading` (required) - Section heading in CHANGELOG for linking
- `changelog_path` (optional, default: `CHANGELOG.md`)
- `release_assets` (optional, default: `[]`) - JSON array of asset paths
- `generate_release_notes` (optional, default: `true`) - Auto-generate from commits
- `target_branch` (optional, default: `main`) - Branch the release tag points to; also used as the base for the CHANGELOG link in the release body.

**Usage:**
```yaml
- uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/create-release.yml@v1.0.0
  with:
    version: ${{ needs.prepare.outputs.new_version }}
    changelog_section_heading: ${{ needs.prepare.outputs.changelog_section_heading }}
    release_assets: '["path/to/asset.zip"]'
```

---

### 4. **open-next-snapshot.yml**
Automatically opens the next development iteration after a release by bumping the Cargo version to `X.Y.(Z+1)-SNAPSHOT` and prepending a new snapshot section to the CHANGELOG.

**Inputs:**
- `cargo_toml_path` (optional, default: `Cargo.toml`) - Path to your Cargo.toml file
- `cargo_lock_path` (optional, default: `Cargo.lock`) - Path to Cargo.lock. Set to empty string (`''`) to skip (e.g. libraries).
- `changelog_path` (optional, default: `CHANGELOG.md`) - Path to your CHANGELOG file
- `commit_message` (optional, default: `chore: prepare for next development iteration`) - Commit message
- `target_branch` (optional, default: `main`) - Branch to push snapshot commit to

**Outputs:**
- `next_version` - The new snapshot version that was set

**Usage:**
```yaml
uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/open-next-snapshot.yml@v1.0.0
with:
  # Optional: can customize paths and branch if needed
```

---

## Using These Workflows in Other Projects

### Step 1: Reference Workflows from the Shared Repository

Call the reusable workflows directly from this repository:
```yaml
uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/bump-version.yml@v1.0.0
```

### Step 2: Create Your Project-Specific Release Workflow

Here's a minimal example for a single-crate Rust project (no extra build artifacts):

```yaml
name: Release

on:
  push:
    branches: [ main ]

jobs:
  prepare:
    runs-on: ubuntu-latest
    if: contains(github.event.head_commit.message, '[release')
    outputs:
      release_type: ${{ steps.determine_type.outputs.type }}
    steps:
      - name: Determine release type
        id: determine_type
        run: |
          COMMIT_MSG="${{ github.event.head_commit.message }}"
          if [[ "$COMMIT_MSG" == *"[release:major]"* ]]; then
            echo "type=major" >> "$GITHUB_OUTPUT"
          elif [[ "$COMMIT_MSG" == *"[release:minor]"* ]]; then
            echo "type=minor" >> "$GITHUB_OUTPUT"
          else
            echo "type=patch" >> "$GITHUB_OUTPUT"
          fi

  # Use the bump workflow from the shared repository
  bump_and_changelog:
    needs: prepare
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/bump-version.yml@v1.0.0
    with:
      release_type: ${{ needs.prepare.outputs.release_type }}
    permissions:
      contents: write

  # Use the finalize workflow from the shared repository
  finalize:
    needs: bump_and_changelog
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/finalize-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_and_changelog.outputs.new_version }}
    permissions:
      contents: write

  # Use the create-release workflow from the shared repository
  create_release:
    needs: [finalize, bump_and_changelog]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/create-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_and_changelog.outputs.new_version }}
      changelog_section_heading: ${{ needs.bump_and_changelog.outputs.changelog_section_heading }}
    permissions:
      contents: write
```

### Step 3: For Rust Projects with Build Artifacts

If you produce additional release artifacts (compiled binaries, Docker images, packaged assets, etc.), add the build job between bump and finalize:

```yaml
  build:
    needs: bump_and_changelog
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - name: Build your artifacts
        run: # your build commands
      - uses: actions/upload-artifact@v4
        with:
          name: release-artifacts
          path: dist/

  finalize:
    needs: [bump_and_changelog, build]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/finalize-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_and_changelog.outputs.new_version }}
    permissions:
      contents: write

  create_release:
    needs: [finalize, bump_and_changelog, build]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/create-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_and_changelog.outputs.new_version }}
      changelog_section_heading: ${{ needs.bump_and_changelog.outputs.changelog_section_heading }}
      release_assets: '["dist/app.zip"]'
    permissions:
      contents: write
```
