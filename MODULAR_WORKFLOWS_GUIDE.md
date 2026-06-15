# Modular Rust Release Workflow Guide

This guide explains how to use the reusable GitHub Actions workflows for managing releases of **Rust projects**. The workflows are Cargo-based: they read and update `Cargo.toml` (and `Cargo.lock` when present) and use [`cargo set-version`](https://github.com/killercup/cargo-edit) to bump the crate version.

## Prerequisites

- The consumer repository is a Rust/Cargo project (single crate or workspace).
- A `Cargo.toml` exists at the path passed via `cargo_toml_path` (default `Cargo.toml`).
- A `CHANGELOG.md` exists with an unreleased placeholder line such as `## [X.Y.Z-SNAPSHOT]` that the bump workflow rewrites into the new version section. See [`CHANGELOG.template.md`](CHANGELOG.template.md) for a ready-to-copy scaffold.
- The repository default branch is `main`, or `target_branch` is set to the branch the release should be pushed to.

## Architecture Overview

The repository ships two layers of reusable workflows:

- **Granular workflows** (one job each) — building blocks you compose yourself when the consumer pipeline needs custom steps:
    - [`bump-version.yml`](#1-bump-versionyml)
    - [`docker-build-push.yml`](#5-docker-build-pushyml)
    - [`finalize-release.yml`](#2-finalize-releaseyml)
    - [`create-release.yml`](#3-create-releaseyml)
    - [`open-next-snapshot.yml`](#4-open-next-snapshotyml)
    - [`rollback-release.yml`](#6-rollback-releaseyml)
- **Orchestrators** — chain the granular ones for the most common pipelines (with automatic rollback on failure):
    - [`release.yml`](#7-releaseyml) — `bump → finalize → create → open-next-snapshot → rollback-release (on failure)`
    - [`release-docker.yml`](#8-release-dockeryml) — same as above but with a Docker build/push between bump and finalize

Use an orchestrator when you don't need extra build steps; otherwise compose the granular workflows yourself.

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

### 2. **docker-build-push.yml**
Builds a (multi-arch) Docker image and pushes it to a container registry. Two modes:

- **Plain Docker build** (default) — just runs `docker/setup-buildx-action`, `docker/login-action`, and `docker/build-push-action` against the configured `dockerfile`/`context`/`platforms`.
- **Rust cross-compile** (`cargo_zigbuild: 'true'`) — also installs the Rust toolchain plus zig and `cargo-zigbuild` and builds every entry of `rust_targets` before the Docker build, so a `Dockerfile` that copies pre-built binaries via build-args (e.g. `BIN_PATH_AMD64=target/x86_64-unknown-linux-musl/release/<bin>`) works out of the box.

**Inputs (most common):**
- `version` (optional, default: empty) - When set, pushes `<image>:v<version>`
- `image_name` (optional, default: lowercased `github.repository`) - Image name without registry, e.g. `org/app`
- `registry` (optional, default: `ghcr.io`)
- `registry_username` (optional, default: `github.actor`)
- `dockerfile` (optional, default: `Dockerfile`)
- `context` (optional, default: `.`)
- `platforms` (optional, default: `linux/amd64`) - Comma-separated, e.g. `linux/amd64,linux/arm64,linux/arm/v7`
- `push_latest` (optional, default: `'true'`) - Also tag as `:latest`
- `extra_tags` (optional) - Newline-separated additional fully-qualified tags
- `build_args` (optional) - Newline-separated `KEY=value` build args
- `cargo_zigbuild` (optional, default: `'false'`) - Enable Rust cross-compile mode
- `rust_targets` (optional, default: `x86_64-unknown-linux-musl`) - Targets to build when `cargo_zigbuild=true`
- `cargo_build_args` (optional) - Extra args appended to each `cargo zigbuild --release --target <t>`
- `zig_version` (optional, default: `0.13.0`)

**Secrets:**
- `registry_password` (optional) - Falls back to `secrets.GITHUB_TOKEN` when omitted.

**Outputs:**
- `image` - Fully-qualified `registry/name` that was pushed
- `tags` - Newline-separated list of pushed tags

**Usage:**
```yaml
uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/docker-build-push.yml@v1.0.0
with:
  version: ${{ needs.bump_version.outputs.new_version }}
  platforms: linux/amd64,linux/arm64
permissions:
  contents: read
  packages: write
```

---

### 3. **finalize-release.yml**
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

### 4. **create-release.yml**
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

### 5. **open-next-snapshot.yml**
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

### 6. **rollback-release.yml**
Best-effort cleanup of a release that failed mid-pipeline. Tries to:

1. Delete the `vX.Y.Z` GitHub release (if it was created).
2. Delete the matching container image version from the registry's package (if it was pushed). Currently supports `ghcr.io` only; other registries are skipped.
3. Delete the `vX.Y.Z` git tag locally and on the remote (if finalize already pushed it).

Each step is guarded so a missing/already-cleaned artifact does not fail the rollback. Designed to be called as `if: failure()` from an orchestrator (the bundled `release.yml` and `release-docker.yml` already do this), but also safe to invoke manually via `workflow_dispatch` to clean up after a partial release.

**Inputs:**
- `version` (required) - The version that was being released (without `v` prefix).
- `delete_github_release` (optional, default: `'true'`)
- `delete_container_image` (optional, default: `'false'`) - Set to `'true'` for Docker-publishing pipelines.
- `container_package_name` (optional, default: `github.event.repository.name`) - Override when the GHCR package name doesn't match the repository name.
- `registry` (optional, default: `ghcr.io`) - Container deletion is skipped for any other registry.
- `delete_tag` (optional, default: `'true'`)
- `target_branch` (optional, default: `main`) - Used as the checkout ref for the tag deletion step.

**Usage (manual cleanup, e.g. via `workflow_dispatch`):**
```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to roll back (without v prefix)'
        required: true
        type: string

jobs:
  rollback:
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/rollback-release.yml@v1.0.0
    with:
      version: ${{ inputs.version }}
      delete_container_image: 'true'
    permissions:
      contents: write
      packages: write
```

---

### 7. **release.yml**
Orchestrator that chains `bump-version → finalize-release → create-release → open-next-snapshot`. The release type (`major`/`minor`/`patch`) is inferred from the head commit message tag (`[release:major]`, `[release:minor]`, `[release:patch]`, or `[release]` → patch).

Use this when the only artifacts attached to the release are the auto-generated notes and the CHANGELOG section. If you need to build extra artifacts (binaries, deploy bundles, etc.), compose the granular workflows directly instead.

**Inputs:**
- `cargo_toml_path` (optional, default: `Cargo.toml`)
- `cargo_lock_path` (optional, default: `Cargo.lock`) - Set to `''` to skip the lockfile.
- `changelog_path` (optional, default: `CHANGELOG.md`)
- `target_branch` (optional, default: `main`)
- `release_assets` (optional, default: `[]`) - JSON array of asset paths

**Outputs:**
- `new_version`
- `changelog_section_heading`
- `next_version`

**Usage:**
```yaml
jobs:
  release:
    if: contains(github.event.head_commit.message, '[release')
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/release.yml@v1.0.0
    permissions:
      contents: write
```

If any job in the chain fails, an `if: failure()` rollback job runs `rollback-release.yml` to delete the partially-created GitHub release and `vX.Y.Z` tag.

---

### 8. **release-docker.yml**
Orchestrator that chains `bump-version → docker-build-push → finalize-release → create-release → open-next-snapshot`. The Docker image is tagged with `:v<new_version>` (and `:latest` when `push_latest=true`) and pushed to the configured registry **before** finalize, so a failed image push aborts the release before any tag/commit is pushed.

**Inputs:** all of `release.yml`'s inputs plus all of `docker-build-push.yml`'s inputs (see those sections for details).

**Secrets:**
- `registry_password` (optional) - Forwarded to `docker-build-push.yml`. Falls back to `secrets.GITHUB_TOKEN` when omitted.

**Outputs:**
- `new_version`
- `changelog_section_heading`
- `next_version`
- `image`
- `tags`

**Usage:**
```yaml
jobs:
  release:
    if: contains(github.event.head_commit.message, '[release')
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/release-docker.yml@v1.0.0
    with:
      platforms: linux/amd64,linux/arm64
    permissions:
      contents: write
      packages: write
```

If any job in the chain fails, an `if: failure()` rollback job runs `rollback-release.yml` with `delete_container_image: 'true'` to delete the partially-created GitHub release, the pushed container image version, and the `vX.Y.Z` tag.

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
      - uses: actions/checkout@v6
      - name: Build your artifacts
        run: # your build commands
      - uses: actions/upload-artifact@v7
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
