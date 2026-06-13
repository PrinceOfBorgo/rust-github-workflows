# Rust Release Workflows

Reusable GitHub Actions workflows for automating releases of **Rust projects** across multiple repositories. The workflows expect a Cargo-based project (`Cargo.toml`, optional `Cargo.lock`) and use [`cargo set-version`](https://github.com/killercup/cargo-edit) to bump the crate version.

## Requirements

- A Rust project with a `Cargo.toml` (workspaces and non-root paths are supported via `cargo_toml_path`).
- A `CHANGELOG.md` containing an unreleased placeholder line of the form `## [X.Y.Z-SNAPSHOT]` that the bump workflow will replace with the new version section.
- A repository default branch named `main` (overridable per call via the `target_branch` input on `finalize-release.yml` and `create-release.yml`).
- If your project commits a `Cargo.lock`, point `cargo_lock_path` at it (defaults to `Cargo.lock` at the repo root). Libraries that don't commit a lockfile should pass `cargo_lock_path: ''`.

## What This Repository Contains

- `.github/workflows/bump-version.yml`
- `.github/workflows/finalize-release.yml`
- `.github/workflows/create-release.yml`
- `MODULAR_WORKFLOWS_GUIDE.md`
- `TEMPLATES_RELEASE_WORKFLOWS.md`

Use `MODULAR_WORKFLOWS_GUIDE.md` as the primary documentation and `TEMPLATES_RELEASE_WORKFLOWS.md` for copy-paste examples.

## Quick Start

1. Add a job that determines release type from commit message.
2. Call the reusable bump workflow.
3. Run your project-specific build jobs.
4. Call finalize and create-release reusable workflows.

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
```

## Versioning Policy

- Prefer pinned tags in consumer repositories, for example `@v1.0.0`.
- Use `@main` only while testing in-flight changes against the latest workflow code.

## Documentation Structure

- `MODULAR_WORKFLOWS_GUIDE.md`: architecture, behavior, and integration flow
- `TEMPLATES_RELEASE_WORKFLOWS.md`: ready-to-copy templates for common project types
