# Build image in two steps: Caching dependencies [![Test Twostep container](https://github.com/epinowcast/actions/actions/workflows/test-twostep-container-build.yml/badge.svg)](https://github.com/epinowcast/actions/actions/workflows/test-twostep-container-build.yml)

<sup>*(This action is a fork of the [CDCgov/cfa-actions/twostep-container-build](https://github.com/CDCgov/cfa-actions/tree/main/twostep-container-build) action)*</sup>

This action will build a container image for a project in two steps and push the image to a container registry. During the first step, using the container file `container-file-1`, it will build and cache the image containing the dependencies of the main project. After the first step, a second build and push process happens based on the container file `container-file-2`. The `container-file-2` uses as base image the one created during the first step.

```mermaid
flowchart LR
  Containerfile1[container-file-1] -->|Generates|Image1
  Image1-->|Is used as a baseline for|Containerfile2
  Containerfile2-->|Generates|Image2
```

The bulk of the action is dealing with whether the first image is available or not. If available, the action will check if the cache key matches the hash of the container file. If it does, the action will skip the first step and use the cached image. If the cache key does not match, the action will build the image and push it to the registry (if `push-image-1` is set to `true`):

```mermaid
flowchart TB
  subgraph "Caching"
    start((Start)) --> tag_exists
    tag_exists{Tag exists}-->|Yes|pull_image
    pull_image[Pull image<br>& inspect<br>labels]-->check_label
    check_label{Label<br>matches<br>hash}-->|Yes|done((Done))
    check_label-->|No|build_image["Build image<br>& push (optional)"]
    tag_exists-->|No|build_image
    build_image-->done
  end
```

Caching is done by storing the cache-key as a label in the image (`TWO_STEP_BUILD_CACHE_KEY`); and the build and push process is done using the [docker/build-push-action@v6](https://github.com/docker/build-push-action/tree/v6) action. Users have to explicitly provide the cache key for the first step. For example, if you are dealing with an R package, you can cache the dependencies by passing the key `${{ hashFiles('DESCRIPTION') }}` to the `first-step-cache-key` input. That way, the first step will only be executed if the dependencies change.

## Inputs and Outputs

| Field | Description | Required | Default |
|-------|-------------|----------|---------|
| `container-file-1` | Path to the first container file | true | |
| `container-file-2` | Path to the second container file | true | |
| `first-step-cache-key` | Cache key for the first step | true | |
| `image` | Name of the image | true | |
| `registry` | Registry to push the image to | true |  |
| `username` | Username for the registry | false |  |
| `password` | Password for the registry | false |  |
| `main-branch-name` | Name of the main branch | false | `'main'` |
| `main-branch-tag` | Tag to use for the main branch | false | `'latest'` |

The following are arguments passed to the [docker/build-push-action@v6](https://github.com/docker/build-push-action/tree/v6) action.

| Field | Description | Required | Default |
|-------|-------------|----------|---------|
| `push-image-1` | Push the image created during the first step | false | `false` |
| `push-image-2` | Push the image created during the second step | false | `false` |
| `build-args-1` | Build arguments for the first step | false | |
| `build-args-2` | Build arguments for the second step | false | |
| `labels-1` | Labels for the first step | false | |
| `labels-2` | Labels for the second step | false | |

The action has the following outputs:

| Field | Description |
|-------|-------------|
| `tag` | Container tag of the built image |
| `branch` | Branch name |
| `summary` | A summary of the action: (`built`, `re-built`, or `cached`) |

## Example: Using ghcr.io

The workflow is triggered on pull requests and pushes to the main branch. The image is pushed to `ghcr.io` and the image name is `epinowcast/actions` (full name is `ghcr.io/epinowcast/actions`). A functional version of this workflow is executed [here](../.github/workflows/test-twostep-container-build.yml).

```yaml
name: Building the container and put it on ghcr.io

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    # Since we are using ghcr.io, we need to set the permissions to write
    # for the packages.
    permissions:
      contents: read
      packages: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        name: Checkout code

      - name: Two-step build
        uses: epinowcast/actions/twostep-container-build@v1.2.0
        with:
          # Login information
          registry: ghcr.io/
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

          # Paths to the container files
          container-file-1: Containerfile.dependencies
          container-file-2: Containerfile

          # We are using the dependency container for caching
          first-step-cache-key: ${{ hashFiles('Containerfile.dependencies') }}

          # The image to build includes the organization (that's how it is
          # on ghcr.io)
          image: epinowcast/actions

```

The container files (which can be found under the [examples](examples) directory) have the following structure:

[`Containerfile.dependencies`](examples/Containerfile.dependencies)

```Containerfile
FROM rocker/r-base:4.4.0

RUN install2.r epiworldR

CMD ["bash"]
```

[`Containerfile`](examples/Containerfile)

```Containerfile
# Collection of ARGs
ARG TAG=dependencies-latest
ARG IMAGE=ghcr.io/epinowcast/actions
ARG GH_SHA=default_var

FROM ${IMAGE}:${TAG}

# Re-declaring the ARGs here is necessary to use them in the LABEL
ARG GH_SHA
LABEL GH_SHA=${GH_SHA}

COPY . /app/.
CMD ["bash"]
```

Notice the `TAG` argument which is passed to the second container file. During runs of the action, `TAG` takes the value of the `dependencies-[branch name]` or `dependencies-latest` if the branch is the main branch.
