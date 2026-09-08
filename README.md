# docker-github-action

GitHub action for building and pushing Docker images to Azure Container Registry. Uses Depot for fast multi-arch builds (ARM64 + AMD64). Falls back to AMD64-only if Depot unavailable.

## Quick Start

```yaml
name: ci

on:
  push:
    branches: ['main']
  pull_request:
    branches: ['main']

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build and Push docker image
        uses: tignis/docker-github-action@depot-integration
        with:
          images: tignis.azurecr.io/tignis/my_app
          acr-username: ${{ secrets.AZURE_APP_ID_ACR }}
          acr-password: ${{ secrets.AZURE_PASSWORD_ACR }}
          pip-extra-index-url: ${{ secrets.PIP_EXTRA_INDEX_URL }}
          platforms: linux/amd64,linux/arm64
          depot-token: ${{ secrets.DEPOT_TOKEN }}
          depot-project: ${{ secrets.DEPOT_PROJECT }}
```

## Build Modes

**Default:** Depot multi-arch builds (ARM64 + AMD64, 1-3 min). `DEPOT_TOKEN` is set at organization level.

**Automatic fallback:** AMD64-only Docker build for catastrophic scenarios (AWS/Depot outage). Enables shipping customer fixes when Depot unavailable. AMD64 sufficient for customer deployments.

To trigger: Settings → Secrets and Variables → Actions → Add `DEPOT_TOKEN=''` (empty string) at repo level.

For multi-arch fallback, see [Fallback Options](#fallback-options).

## Secrets

Configure in repository/organization settings:

**Required:**
- `AZURE_APP_ID_ACR` / `AZURE_PASSWORD_ACR` - ACR credentials

**Required for private packages:**
- `PIP_EXTRA_INDEX_URL` - When using pip
- `UV_INDEX_URL` - When using UV
- `ARTIFACTORY_TOKEN` - When using npm against private (Artifactory) scopes

**Default (set at organization level):**
- `DEPOT_TOKEN` - Enables fast multi-arch Depot builds. Falls back to AMD64-only if not set

## Package Management

### Using pip (Standard)

Set `pip-extra-index-url` secret and mount it in your Dockerfile:

```dockerfile
RUN --mount=type=secret,id=pipconf,target=/etc/pip.conf \
    --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

### Using UV (Fast)

Set `uv-index-url` secret:

```dockerfile
RUN --mount=type=secret,id=uvconfig,target=/root/.config/uv/uv.toml \
    --mount=type=cache,target=/root/.cache/uv \
    uv pip install -r requirements.txt
```

### Using npm (Artifactory)

Check a `.npmrc` into the repo root with a `${ARTIFACTORY_TOKEN}` placeholder, e.g.:

```ini
@myorg:registry=https://mycompany.jfrog.io/artifactory/api/npm/my-npm/
//mycompany.jfrog.io/artifactory/api/npm/my-npm/:_authToken=${ARTIFACTORY_TOKEN}
always-auth=true
```

Set `artifactory-token` and mount it in your Dockerfile:

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/app/.npmrc \
    npm ci
```

If no `.npmrc` exists in the repo, or `artifactory-token` isn't set, the mounted secret is just empty — harmless for a Dockerfile that doesn't reference it.

## Dockerfile Best Practices

Use multi-stage builds with proper layer caching:

```dockerfile
# ==============================================
# BUILD STAGE - Install dependencies
# ==============================================
FROM python:3.12-slim AS build

ENV PYTHONUNBUFFERED=1

WORKDIR /build

# Copy requirements first for layer caching
COPY requirements.txt .

# Install dependencies with secret and cache mounts
RUN --mount=type=secret,id=pipconf,target=/etc/pip.conf \
    --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Copy source code (separate layer)
COPY . .

# Install application
RUN --mount=type=secret,id=pipconf,target=/etc/pip.conf \
    --mount=type=cache,target=/root/.cache/pip \
    pip install .

# ==============================================
# TEST STAGE - Run tests
# ==============================================
FROM build AS test

RUN pytest ./tests/ --disable-warnings -n auto

# ==============================================
# RUNTIME STAGE - Minimal production image
# ==============================================
FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Copy only installed packages from build stage
COPY --from=build /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=build /usr/local/bin /usr/local/bin

ENTRYPOINT ["python", "-m", "your_app"]
```

**Key points:**
- `test` stage runs automatically during build (no separate CI job needed)
- Build fails if tests fail (automatic quality gate)
- Layer caching optimizes rebuild times
- Final image is minimal (no source code, no build tools)

## Examples

### Basic Multi-Platform Build

```yaml
- uses: tignis/docker-github-action@depot-integration
  with:
    images: tignis.azurecr.io/tignis/my_app
    acr-username: ${{ secrets.AZURE_APP_ID_ACR }}
    acr-password: ${{ secrets.AZURE_PASSWORD_ACR }}
    pip-extra-index-url: ${{ secrets.PIP_EXTRA_INDEX_URL }}
    platforms: linux/amd64,linux/arm64
    depot-token: ${{ secrets.DEPOT_TOKEN }}
    depot-project: ${{ secrets.DEPOT_PROJECT }}
```

### Single Platform (AMD64 only)

```yaml
- uses: tignis/docker-github-action@depot-integration
  with:
    images: tignis.azurecr.io/tignis/my_app
    acr-username: ${{ secrets.AZURE_APP_ID_ACR }}
    acr-password: ${{ secrets.AZURE_PASSWORD_ACR }}
    platforms: linux/amd64
    depot-token: ${{ secrets.DEPOT_TOKEN }}
    depot-project: ${{ secrets.DEPOT_PROJECT }}
```

### UV-Based Build

```yaml
- uses: tignis/docker-github-action@depot-integration
  with:
    images: tignis.azurecr.io/tignis/my_app
    acr-username: ${{ secrets.AZURE_APP_ID_ACR }}
    acr-password: ${{ secrets.AZURE_PASSWORD_ACR }}
    uv-index-url: ${{ secrets.UV_INDEX_URL }}
    depot-token: ${{ secrets.DEPOT_TOKEN }}
    depot-project: ${{ secrets.DEPOT_PROJECT }}
```

## Troubleshooting

### Build fails with "Configuration file could not be loaded"

**Cause:** The pip.conf secret file has invalid formatting.

**Fix:** Ensure your `pip-extra-index-url` secret is set correctly in repository settings.

### Build only produces AMD64 image (no ARM64)

**Cause:** Automatic fallback to Docker build (Depot service unavailable or token expired).

**Fix:** Verify that:
1. Depot service is operational
2. `depot-project` parameter is specified in the workflow
3. The Depot project ID is correct

### Tests fail but build continues

**Cause:** Docker test stage not properly configured.

**Fix:** Ensure your Dockerfile has a `test` stage and the action doesn't use `target` to skip it:

```dockerfile
FROM build AS test
RUN pytest ./tests/ --disable-warnings -n auto
```

## Fallback Options

### v2.3.2 with Self-Hosted Runners (Manual)

For multi-arch builds during Depot outages, temporarily switch to v2.3.2 with self-hosted ARM64 runners:

**Current v3:**
```yaml
jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: tignis/docker-github-action@v3
        with:
          images: tignis.azurecr.io/tignis/my_app
          depot-token: ${{ secrets.DEPOT_TOKEN }}
```

**Emergency switch to v2.3.2:**
```yaml
jobs:
  docker:
    uses: tignis/docker-github-action/.github/workflows/workflows.yaml@v2.3.2
    with:
      images: tignis.azurecr.io/tignis/my_app
    secrets:
      acr-username: ${{ secrets.AZURE_APP_ID_ACR }}
      acr-password: ${{ secrets.AZURE_PASSWORD_ACR }}
      pip-extra-index-url: ${{ secrets.PIP_EXTRA_INDEX_URL }}
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
