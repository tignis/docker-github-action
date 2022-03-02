# docker-github-action
github action wrapping a typical docker build for a tignis app. This workflow assumes that you will
tag images based on the git commit hash for pull requests and pushes to branches, and for releases
it will use the semver tag of the release.

Below is an example of how to use this action.
```
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
        uses: actions/checkout@v2
      - name: Build and Push docker image
        uses: tignis/docker-github-action@v1.1.0
        with:
          images: |
            tignis.azurecr.io/tignis/docker_github_action
          acr-username: ${{ secrets.AZURE_APP_ID_ACR }}
          acr-password: ${{ secrets.AZURE_PASSWORD_ACR }}
          pip-extra-index-url: ${{secrets.PIP_EXTRA_INDEX_URL}}"

```

## Input options

`images`: A list of names to use to tag your image with. Should be a multiline string with each line containing a single name.

`push`: Boolean to determine if the image should be pushed to the remote repoistory. Defaults to `true`.

`acr-username`: The username to use to login to acr. Fetch this value from a github secret.

`acr-password`: The password to use to login to acr. Fetch this value from a github secret.

`acr-registry-url`: The url of which repository to use in ACR. Default to `tignis.azurecr.io`.

`pip-extra-index-url`: The extra index url for pip to fetch packages from our jfrog repository. Fetch this value from a secret.

`docker-build-context`: What directory to use as the build context for docker. Defaults to the current directory.
