# Templates for Using the Rust Release Workflows

This file provides copy-paste templates for common Rust project layouts.

> **Prerequisites for every template**
>
> - Cargo-based project (`Cargo.toml` at the configured `cargo_toml_path`)
> - `CHANGELOG.md` with an unreleased placeholder line of the form `## [X.Y.Z-SNAPSHOT]` (copy [`CHANGELOG.template.md`](CHANGELOG.template.md) as a starting point)
> - Default branch is `main` (or pass `target_branch:` to `finalize-release.yml` and `create-release.yml`)
> - `Cargo.lock` defaults to the repo root. Override with `cargo_lock_path` for non-standard layouts, or pass `cargo_lock_path: ''` for libraries that don't commit a lockfile.
> - Examples below pin to `@v1.0.0`; bump the tag (e.g. `@v1.1.0`) to adopt newer workflow versions
> - Templates 1-3 use the **granular** workflows so you can add custom build steps. Templates 4-5 use the **orchestrators** (`release.yml`, `release-docker.yml`) when the standard pipeline is enough.

---

## Template 1: Single-Crate Rust Project

```yaml
name: Release

on:
  push:
    branches: [ main ]

jobs:
  determine_type:
    runs-on: ubuntu-latest
    if: |
      contains(github.event.head_commit.message, '[release]') ||
      contains(github.event.head_commit.message, '[release:patch]') ||
      contains(github.event.head_commit.message, '[release:minor]') ||
      contains(github.event.head_commit.message, '[release:major]')
    outputs:
      release_type: ${{ steps.type.outputs.value }}
    steps:
      - name: Determine release type
        id: type
        run: |
          MSG="${{ github.event.head_commit.message }}"
          [[ "$MSG" == *"[release:major]"* ]] && echo "value=major" >> "$GITHUB_OUTPUT" || \
          [[ "$MSG" == *"[release:minor]"* ]] && echo "value=minor" >> "$GITHUB_OUTPUT" || \
          echo "value=patch" >> "$GITHUB_OUTPUT"

  bump_version:
    needs: determine_type
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/bump-version.yml@v1.0.0
    with:
      release_type: ${{ needs.determine_type.outputs.release_type }}
    permissions:
      contents: write

  finalize:
    needs: bump_version
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/finalize-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_version.outputs.new_version }}
    permissions:
      contents: write

  create_release:
    needs: [finalize, bump_version]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/create-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_version.outputs.new_version }}
      changelog_section_heading: ${{ needs.bump_version.outputs.changelog_section_heading }}
    permissions:
      contents: write

  open_next:
    needs: [finalize, create_release]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/open-next-snapshot.yml@v1.0.0
    permissions:
      contents: write
```

---

## Template 2: Cargo Workspace / Non-Root Crate

For a Rust workspace where the released crate lives in a subdirectory. The lockfile stays at the workspace root, so `cargo_lock_path` keeps its default `Cargo.lock`:

```yaml
name: Release

on:
  push:
    branches: [ main ]

env:
  MAIN_CRATE_DIR: crates/my-crate

jobs:
  determine_type:
    runs-on: ubuntu-latest
    if: contains(github.event.head_commit.message, '[release')
    outputs:
      release_type: ${{ steps.type.outputs.value }}
    steps:
      - name: Determine release type
        id: type
        run: |
          MSG="${{ github.event.head_commit.message }}"
          [[ "$MSG" == *"[release:major]"* ]] && echo "value=major" >> "$GITHUB_OUTPUT" || \
          [[ "$MSG" == *"[release:minor]"* ]] && echo "value=minor" >> "$GITHUB_OUTPUT" || \
          echo "value=patch" >> "$GITHUB_OUTPUT"

  bump_version:
    needs: determine_type
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/bump-version.yml@v1.0.0
    with:
      cargo_toml_path: ${{ env.MAIN_CRATE_DIR }}/Cargo.toml
      changelog_path: ${{ env.MAIN_CRATE_DIR }}/CHANGELOG.md
      release_type: ${{ needs.determine_type.outputs.release_type }}
    permissions:
      contents: write

  finalize:
    needs: bump_version
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/finalize-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_version.outputs.new_version }}
      cargo_toml_path: ${{ env.MAIN_CRATE_DIR }}/Cargo.toml
      changelog_path: ${{ env.MAIN_CRATE_DIR }}/CHANGELOG.md
    permissions:
      contents: write

  create_release:
    needs: [finalize, bump_version]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/create-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_version.outputs.new_version }}
      changelog_section_heading: ${{ needs.bump_version.outputs.changelog_section_heading }}
      changelog_path: ${{ env.MAIN_CRATE_DIR }}/CHANGELOG.md
    permissions:
      contents: write

  open_next:
    needs: [finalize, create_release]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/open-next-snapshot.yml@v1.0.0
    with:
      cargo_toml_path: ${{ env.MAIN_CRATE_DIR }}/Cargo.toml
      changelog_path: ${{ env.MAIN_CRATE_DIR }}/CHANGELOG.md
    permissions:
      contents: write
```

---

## Template 3: Rust Project with Build Artifacts

For a Rust project that ships compiled binaries (or other built assets) attached to the GitHub release:

```yaml
name: Release

on:
  push:
    branches: [ main ]

jobs:
  determine_type:
    runs-on: ubuntu-latest
    if: contains(github.event.head_commit.message, '[release')
    outputs:
      release_type: ${{ steps.type.outputs.value }}
    steps:
      - name: Determine release type
        id: type
        run: |
          MSG="${{ github.event.head_commit.message }}"
          [[ "$MSG" == *"[release:major]"* ]] && echo "value=major" >> "$GITHUB_OUTPUT" || \
          [[ "$MSG" == *"[release:minor]"* ]] && echo "value=minor" >> "$GITHUB_OUTPUT" || \
          echo "value=patch" >> "$GITHUB_OUTPUT"

  bump_version:
    needs: determine_type
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/bump-version.yml@v1.0.0
    with:
      release_type: ${{ needs.determine_type.outputs.release_type }}
    permissions:
      contents: write

  build:
    needs: bump_version
    runs-on: ubuntu-latest
    outputs:
      artifact_path: ${{ steps.build.outputs.artifact }}
    steps:
      - uses: actions/checkout@v6
      - uses: dtolnay/rust-toolchain@stable
      - name: Build release binary
        id: build
        run: |
          cargo build --release
          mkdir -p dist
          cp target/release/my-app dist/my-app-${{ needs.bump_version.outputs.new_version }}
          echo "artifact=dist/my-app-${{ needs.bump_version.outputs.new_version }}" >> "$GITHUB_OUTPUT"
      - name: Upload build artifact
        uses: actions/upload-artifact@v7
        with:
          name: app-build
          path: dist/
          retention-days: 1

  finalize:
    needs: [bump_version, build]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/finalize-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_version.outputs.new_version }}
    permissions:
      contents: write

  create_release:
    needs: [finalize, bump_version, build]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/create-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_version.outputs.new_version }}
      changelog_section_heading: ${{ needs.bump_version.outputs.changelog_section_heading }}
      release_assets: ${{ format('["{0}"]', needs.build.outputs.artifact_path) }}
    permissions:
      contents: write

  open_next:
    needs: [finalize, create_release]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/open-next-snapshot.yml@v1.0.0
    permissions:
      contents: write
```

---

## Template 4: Plain Release via Orchestrator (`release.yml`)

Minimal `bump → finalize → create → open-next-snapshot` pipeline for projects that don't ship extra build artifacts. The release type is inferred from the head commit message:

```yaml
name: Release

on:
  push:
    branches: [ main ]

jobs:
  release:
    if: |
      contains(github.event.head_commit.message, '[release]') ||
      contains(github.event.head_commit.message, '[release:patch]') ||
      contains(github.event.head_commit.message, '[release:minor]') ||
      contains(github.event.head_commit.message, '[release:major]')
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/release.yml@v1.0.0
    permissions:
      contents: write
```

For a workspace, pass the per-crate paths through:

```yaml
  release:
    # ...
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/release.yml@v1.0.0
    with:
      cargo_toml_path: crates/my-crate/Cargo.toml
      changelog_path: crates/my-crate/CHANGELOG.md
    permissions:
      contents: write
```

---

## Template 5: Release with Docker Image via Orchestrator (`release-docker.yml`)

For a Rust service that publishes a Docker image to GHCR alongside the GitHub release. The orchestrator runs `bump → docker-build-push → finalize → create → open-next-snapshot`, so a failed image push aborts the release before any tag is pushed.

```yaml
name: Release

on:
  push:
    branches: [ main ]

jobs:
  release:
    if: |
      contains(github.event.head_commit.message, '[release]') ||
      contains(github.event.head_commit.message, '[release:patch]') ||
      contains(github.event.head_commit.message, '[release:minor]') ||
      contains(github.event.head_commit.message, '[release:major]')
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/release-docker.yml@v1.0.0
    with:
      platforms: linux/amd64,linux/arm64
    permissions:
      contents: write
      packages: write
```

If your `Dockerfile` consumes pre-built binaries built with `cargo-zigbuild` (the `travel-rs-bot` pattern), enable the cross-compile mode:

```yaml
  release:
    # ...
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/release-docker.yml@v1.0.0
    with:
      platforms: linux/amd64,linux/arm64,linux/arm/v7
      cargo_zigbuild: 'true'
      rust_targets: |
        x86_64-unknown-linux-musl
        aarch64-unknown-linux-musl
        armv7-unknown-linux-musleabihf
      build_args: |
        BIN_PATH_AMD64=target/x86_64-unknown-linux-musl/release/my-app
        BIN_PATH_ARM64=target/aarch64-unknown-linux-musl/release/my-app
        BIN_PATH_ARMV7=target/armv7-unknown-linux-musleabihf/release/my-app
    permissions:
      contents: write
      packages: write
```

Pushing to a non-GHCR registry? Pass `registry`, `registry_username`, and the `registry_password` secret:

```yaml
  release:
    # ...
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/release-docker.yml@v1.0.0
    with:
      registry: docker.io
      image_name: myorg/my-app
      registry_username: ${{ vars.DOCKER_HUB_USER }}
    secrets:
      registry_password: ${{ secrets.DOCKER_HUB_TOKEN }}
    permissions:
      contents: write
```

---

## Customization Points

### Modify Commit Detection
Change this condition in any template to match your git commit convention:
```yaml
if: |
  contains(github.event.head_commit.message, '[release]') ||
  contains(github.event.head_commit.message, '[release:patch]')
```

### Change Target Branch
If your default branch is not `main`, pass `target_branch:` to both `finalize-release.yml` and `create-release.yml`:
```yaml
finalize:
  uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/finalize-release.yml@v1.0.0
  with:
    version: ${{ needs.bump_version.outputs.new_version }}
    target_branch: develop

create_release:
  uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/create-release.yml@v1.0.0
  with:
    version: ${{ needs.bump_version.outputs.new_version }}
    changelog_section_heading: ${{ needs.bump_version.outputs.changelog_section_heading }}
    target_branch: develop
```

### Add Release Validation
Insert a Cargo validation job before `finalize`:
```yaml
validate:
  needs: bump_version
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v6
    - uses: dtolnay/rust-toolchain@stable
    - run: cargo test --all-features
    - run: cargo clippy --all-targets -- -D warnings
```
Then add `validate` to the `needs:` list of the `finalize` job.

### Customize Auto-Snapshot Opening (or Disable It)

All templates include an optional `open_next` job that automatically bumps the version to the next snapshot
and prepends a new changelog section. This prepares the repository for the next development cycle immediately
after release.

**To disable automatic snapshot opening:**
Remove the `open_next` job from the template. You'll then manually add a fresh `## [X.Y.Z-SNAPSHOT]`
line to `CHANGELOG.md` before the next release can be prepared.

**To customize the paths for workspaces:**
Update the `open_next` job's `with:` section (similar to how other jobs are configured):
```yaml
open_next:
  needs: [finalize, create_release]
  uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/open-next-snapshot.yml@v1.0.0
  with:
    cargo_toml_path: crates/my-crate/Cargo.toml
    changelog_path: crates/my-crate/CHANGELOG.md
  permissions:
    contents: write
```

### Add Rollback on Failure (Granular Templates)

The orchestrator workflows (Templates 4 and 5) already wire a rollback job with `if: failure()`. If you're composing the **granular** workflows yourself (Templates 1-3), you can opt into the same behavior by appending a rollback job that depends on every other release job:

```yaml
  rollback:
    if: failure() && needs.bump_version.outputs.new_version != ''
    needs: [bump_version, finalize, create_release]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/rollback-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_version.outputs.new_version }}
      delete_github_release: 'true'
      delete_tag: 'true'
    permissions:
      contents: write
      packages: write
```

For a Docker-publishing pipeline, also pass `delete_container_image: 'true'` (and `container_package_name` when your image name doesn't match the repository name):

```yaml
  rollback:
    if: failure() && needs.bump_version.outputs.new_version != ''
    needs: [bump_version, build_and_push_docker, finalize, create_release]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/rollback-release.yml@v1.0.0
    with:
      version: ${{ needs.bump_version.outputs.new_version }}
      delete_github_release: 'true'
      delete_container_image: 'true'
      delete_tag: 'true'
    permissions:
      contents: write
      packages: write
```

### Manual Rollback via `workflow_dispatch`

Add a stand-alone workflow that lets you trigger `rollback-release.yml` from the **Actions** tab to clean up after a partial release that wasn't auto-rolled-back:

```yaml
name: Rollback Release

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to roll back (without v prefix)'
        required: true
        type: string
      delete_container_image:
        description: 'Also delete the matching container image version'
        required: false
        default: 'false'
        type: choice
        options: ['true', 'false']

jobs:
  rollback:
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/rollback-release.yml@v1.0.0
    with:
      version: ${{ inputs.version }}
      delete_container_image: ${{ inputs.delete_container_image }}
    permissions:
      contents: write
      packages: write
```

---

## Getting Started

1. **Choose your template**:
   - **Templates 1-3** (granular workflows) when your release pipeline needs custom steps between bump and finalize.
   - **Templates 4-5** (orchestrators) when the standard pipeline (with optional Docker push) is enough.
2. **Copy it to `.github/workflows/release.yml`** in your repo
3. **Ensure `CHANGELOG.md` has a `## [X.Y.Z-SNAPSHOT]` placeholder** that the bump workflow can rewrite (copy [`CHANGELOG.template.md`](CHANGELOG.template.md) as `CHANGELOG.md` to get a ready-made scaffold)
4. **Adjust `cargo_toml_path` / `changelog_path`** if your crate is not at the repo root
5. **Test with a `[release:patch]` commit to `main`**
6. **Stay on a pinned tag** (e.g. `@v1.0.0`) and bump it deliberately when newer workflow versions are released
