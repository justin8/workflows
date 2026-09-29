# Shared Workflows and Actions

Central repository for shared GitHub Actions workflows and composite actions used across Python projects.

## Components

### 1. `python-build-test.yml` (Reusable Workflow)

Location: `.github/workflows/python-build-test.yml`

A reusable workflow (`workflow_call`) that builds and tests Python packages using [`uv`](https://github.com/astral-sh/uv).

**Features:**
- Installs `uv` with CI caching enabled.
- Reads Python version from `.python-version`.
- Syncs dependencies with `uv sync --locked --all-extras --dev` and builds artifacts with `uv build`.
- Lints code using `uv run ruff check .`.
- Runs tests with `pytest` and generates coverage reports (`dist`, `coverage.xml`, `htmlcov`).
- Uploads the `dist` artifact for downstream publishing jobs.

**Usage:**

Create `.github/workflows/build-test.yml` in the caller repository:

```yaml
name: Build and Test

on: [push, pull_request, workflow_call]

jobs:
  build-and-test:
    uses: justin8/workflows/.github/workflows/python-build-test.yml@main
```

---

### 2. `python-tag-and-publish` (Composite Action)

Location: `python-tag-and-publish/action.yml`

A composite action (`uses: justin8/workflows/python-tag-and-publish@main`) that packages, uploads coverage, publishes to PyPI, and creates a GitHub release.

**Features:**
- Downloads built artifacts from the `dist` artifact.
- Uploads coverage reports to Codecov using tokenless OIDC (`use_oidc: true`).
- Publishes packages to PyPI with `uv publish` using PyPI Trusted Publishing (OIDC).
- Creates a GitHub Release for the pushed tag with auto-generated release notes and attached distribution assets.

#### Why a Composite Action instead of a Reusable Workflow?

PyPI Trusted Publishing requires that the OIDC token minted by GitHub Actions matches the configured repository in PyPI's trusted publisher settings. When using reusable workflows (`workflow_call`), GitHub sets the `job_workflow_ref` claim to the workflow path in `justin8/workflows` rather than the caller's repository. PyPI rejects such tokens with `422 Unprocessable Entity: invalid-publisher` ([warehouse#11096](https://github.com/pypi/warehouse/issues/11096)).

Because a composite action runs in the context of the caller's job, `job_workflow_ref` correctly reflects the target repository, allowing tokenless Trusted Publishing to succeed.

**Usage:**

Create `.github/workflows/publish.yml` in the caller repository:

```yaml
name: Publish

on:
  push:
    tags:
      - "*.*.*"

# Using shared composite action because PyPI Trusted Publishing requires OIDC claims to match this repository (warehouse#11096)
permissions:
  contents: write
  id-token: write

jobs:
  build-and-test:
    uses: justin8/workflows/.github/workflows/python-build-test.yml@main

  publish:
    needs: [build-and-test]
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: justin8/workflows/python-tag-and-publish@main
```
