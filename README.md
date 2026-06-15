# Rust Release Workflows

Reusable GitHub Actions workflows for automating releases of **Rust projects** across multiple repositories. The workflows expect a Cargo-based project (`Cargo.toml`, optional `Cargo.lock`) and use [`cargo set-version`](https://github.com/killercup/cargo-edit) to bump the crate version.

## Requirements

- A Rust project with a `Cargo.toml` (workspaces and non-root paths are supported via `cargo_toml_path`).
- A `CHANGELOG.md` containing an unreleased placeholder line of the form `## [X.Y.Z-SNAPSHOT]` that the bump workflow will replace with the new version section. Copy [`CHANGELOG.template.md`](CHANGELOG.template.md) into your repo as `CHANGELOG.md` to start from a ready-made scaffold.
- A repository default branch named `main` (overridable per call via the `target_branch` input on `finalize-release.yml` and `create-release.yml`).
- If your project commits a `Cargo.lock`, point `cargo_lock_path` at it (defaults to `Cargo.lock` at the repo root). Libraries that don't commit a lockfile should pass `cargo_lock_path: ''`.

## What This Repository Contains

Granular reusable workflows (compose them yourself when you need extra build steps):

- `.github/workflows/bump-version.yml`
- `.github/workflows/docker-build-push.yml`
- `.github/workflows/finalize-release.yml`
- `.github/workflows/create-release.yml`
- `.github/workflows/open-next-snapshot.yml`
- `.github/workflows/rollback-release.yml`

Orchestrator workflows (call once to run a pre-wired pipeline, including automatic rollback on failure):

- `.github/workflows/release.yml` — `bump → finalize → create → open-next-snapshot → rollback-release (on failure)`
- `.github/workflows/release-docker.yml` — `bump → docker-build-push → finalize → create → open-next-snapshot → rollback-release (on failure)` (rollback will also delete the pushed Docker image)

Documentation:

- `MODULAR_WORKFLOWS_GUIDE.md`
- `TEMPLATES_RELEASE_WORKFLOWS.md`
- `CHANGELOG.template.md`

Use `MODULAR_WORKFLOWS_GUIDE.md` as the primary documentation, `TEMPLATES_RELEASE_WORKFLOWS.md` for copy-paste workflow examples, and `CHANGELOG.template.md` as the starting point for your project's `CHANGELOG.md`.

## Quick Start

You have two options depending on how much custom logic the consumer pipeline needs.

### Option A — Use an orchestrator (recommended for the common cases)

If the only thing your release does is bump version + create a GitHub release (optionally with a Docker push), call one of the orchestrators and you're done.

Plain release (no Docker):

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

Release with Docker image push:

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

Both orchestrators infer the release type (`major` / `minor` / `patch`) from the head commit message tag and run the full pipeline including the next-snapshot bump.

### Option B — Compose granular workflows

Use this when you need extra steps between stages (custom builds, deploy bundles, validation jobs, etc.).

1. Add a job that determines release type from commit message.
2. Call the reusable bump workflow.
3. Run your project-specific build jobs.
4. Call finalize and create-release reusable workflows.
5. Call open-next-snapshot to automatically prepare the next development iteration.

Example:

```yaml
jobs:
  prepare:
    runs-on: ubuntu-latest
    if: |
      contains(github.event.head_commit.message, '[release]') ||
      contains(github.event.head_commit.message, '[release:patch]') ||
      contains(github.event.head_commit.message, '[release:minor]') ||
      contains(github.event.head_commit.message, '[release:major]')
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

  prepare_release_data:
    needs: prepare
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/bump-version.yml@v1.0.0
    with:
      release_type: ${{ needs.prepare.outputs.release_type }}
    permissions:
      contents: write

  finalize_release:
    needs: prepare_release_data
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/finalize-release.yml@v1.0.0
    with:
      version: ${{ needs.prepare_release_data.outputs.new_version }}
    permissions:
      contents: write

  create_github_release:
    needs: [prepare_release_data, finalize_release]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/create-release.yml@v1.0.0
    with:
      version: ${{ needs.prepare_release_data.outputs.new_version }}
      changelog_section_heading: ${{ needs.prepare_release_data.outputs.changelog_section_heading }}
    permissions:
      contents: write

  open_next:
    needs: [finalize_release, create_github_release]
    uses: PrinceOfBorgo/rust-github-workflows/.github/workflows/open-next-snapshot.yml@v1.0.0
    permissions:
      contents: write
```

## Versioning Policy

- Prefer pinned tags in consumer repositories, for example `@v1.0.0`.
- Use `@main` only while testing in-flight changes against the latest workflow code.

## Documentation Structure

- `MODULAR_WORKFLOWS_GUIDE.md`: architecture, behavior, and integration flow
- `TEMPLATES_RELEASE_WORKFLOWS.md`: ready-to-copy templates for common project types
- `CHANGELOG.template.md`: drop-in `CHANGELOG.md` scaffold with the `## [X.Y.Z-SNAPSHOT]` placeholder the bump workflow expects
