# docker-github-action

GitHub action wrapping a typical Docker build for a Tignis app. This workflow assumes that you will tag images based on the git commit hash for pull requests and pushes to branches, and for releases it will use the semver tag of the release.

## Usage Options

### Option 1: Single Action (Simple)

Use the action directly for single-platform builds or when you don't need separate architecture builds:

```yaml
name: ci

on:
  push:
    branches:
      - 'main'
  pull_request:
    branches:
      - 'main'
  release:
    types: [created]

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Build and Push docker image
        uses: tignis/docker-github-action@v2.3.1
        with:
          images: |
            tignis.azurecr.io/tignis/my_app
          acr-username: ${{ secrets.AZURE_APP_ID_ACR }}
          acr-password: ${{ secrets.AZURE_PASSWORD_ACR }}
          pip-extra-index-url: ${{ secrets.PIP_EXTRA_INDEX_URL }}
```

### Option 2: Multi-Architecture Workflow (Recommended)

Use the reusable workflow for multi-architecture builds with dedicated runners for each architecture:

```yaml
name: ci

permissions:
  contents: read
  issues: write
  pull-requests: write

on:
  push:
    branches:
      - 'main'
  pull_request:
    branches:
      - 'main'
      - 'dev'
  release:
    types: [created]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Install package
        env:
          PIP_EXTRA_INDEX_URL: ${{ secrets.PIP_EXTRA_INDEX_URL }}
        run: |
          python -m pip install --upgrade pip setuptools
          pip install '.[testing]'
      - name: Test with pytest
        run: pytest

  docker:
    needs:
      - build  # Optional: depend on tests passing first
    uses: tignis/docker-github-action/.github/workflows/workflows.yaml@v2.3.1
    with:
      images: tignis.azurecr.io/tignis/my_app
    secrets:
      acr-username: ${{ secrets.AZURE_APP_ID_ACR }}
      acr-password: ${{ secrets.AZURE_PASSWORD_ACR }}
      pip-extra-index-url: ${{ secrets.PIP_EXTRA_INDEX_URL }}
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Multi-Architecture Workflow Features

The reusable workflow (`workflows.yaml`) provides:

- **Separate Architecture Builds**: AMD64 builds on GitHub-hosted runners, ARM64 builds on self-hosted runners
- **Multi-Architecture Manifest**: Automatically creates a manifest list that supports both architectures
- **Status Reporting**: Provides a unified status check for branch protection rules
- **PR Comments**: Automatically comments on pull requests with the built image tag
- **Build Summary**: Creates detailed build summaries in GitHub Actions

### Workflow Jobs

The multi-architecture workflow consists of:

1. **docker-amd64**: Builds AMD64 image on `ubuntu-latest`
2. **docker-arm64**: Builds ARM64 image on `[self-hosted, linux, ARM64]`
3. **docker-manifest**: Creates multi-architecture manifest from individual builds
4. **docker**: Summary job that reports overall build status

### Required Secrets

For the multi-architecture workflow, you need these repository secrets:

- `AZURE_APP_ID_ACR`: Azure Container Registry username
- `AZURE_PASSWORD_ACR`: Azure Container Registry password  
- `PIP_EXTRA_INDEX_URL`: Private pip package index URL
- `GITHUB_TOKEN`: Automatically provided by GitHub (no setup needed)

## Input Options

### Action Inputs (Option 1)

`images`: A list of names to use to tag your image with. Should be a multiline string with each line containing a single name.

`push`: Boolean to determine if the image should be pushed to the remote repository. Defaults to `true`.

`acr-username`: The username to use to login to ACR. Fetch this value from a GitHub secret.

`acr-password`: The password to use to login to ACR. Fetch this value from a GitHub secret.

`acr-registry-url`: The URL of which repository to use in ACR. Defaults to `tignis.azurecr.io`.

`pip-extra-index-url`: The extra index URL for pip to fetch packages from our JFrog repository. Fetch this value from a secret.

`docker-build-context`: What directory to use as the build context for Docker. Defaults to the current directory.
**Note:** If changing the build context, ensure that the `dockerfile` parameter described below is also adjusted to be prefixed with the build context. For example if you have `docker-build-context: ./tignis/app`, then you'll also likely set `dockerfile: ./tignis/app/Dockerfile` too.

`dockerfile`: The name of the Dockerfile to use. Defaults to `Dockerfile`.
**Note:** This path is always from the root of the repository, not from the root of the build-context.

`platforms`: A comma separated list of platforms to build images for. Defaults to `linux/amd64,linux/arm64`.

`tag-prefix`: A prefix to add to the generated image tag. Defaults to an empty string.

### Workflow Inputs (Option 2)

The reusable workflow accepts similar inputs but through the `with:` section:

- `images`: Container image name (required)
- `acr-registry-url`: Registry URL (optional, defaults to `tignis.azurecr.io`)
- `push`: Whether to push images (optional, defaults to `true`)
- `docker-build-context`: Build context directory (optional, defaults to `.`)
- `dockerfile`: Dockerfile name (optional, defaults to `Dockerfile`)

## Outputs

### Action Outputs (Option 1)

`tag`: The image tag that was generated.

### Workflow Outputs (Option 2)

`tag`: The final multi-architecture manifest tag that was created.

## Self-Hosted Runner Requirements

For the multi-architecture workflow to work properly, you need:

- Self-hosted runner with `[self-hosted, linux, ARM64]` labels
- Docker installed and configured on the ARM64 runner
- Access to your container registry from the self-hosted runner

## Migration Guide

To migrate from the single action to the multi-architecture workflow:

1. Replace the `steps:` section with a `uses:` reference to the workflow
2. Move your parameters from `with:` (action) to `with:` (workflow) 
3. Move secrets from `with:` to the `secrets:` section
4. Add `GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}` to secrets
5. Add required permissions to your workflow file
6. Ensure you have ARM64 self-hosted runners available

The multi-architecture workflow is recommended for production applications that need to support both AMD64 and ARM64 platforms.
