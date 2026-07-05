# build

GitLab CI/CD components for building and publishing Docker images.

[[_TOC_]]

## Create release

1. Add an entry to `CHANGELOG.md` before merging to `main`:

    ```md
    ## 1.0.0 (2026-03-23)
    ### Added
    - Added `build` component
    ```

2. Merge to `main`.
3. Create a [Tag](../../-/tags) from `main` using semantic versioning (`X.Y.Z`).

    Tag examples:
    - `1.0.0` — release
    - `1.0.0-rc` — release candidate

    > RC versions are **not** fetched when using partial version selectors like `@1`.
    > To use a pre-release, specify the full version: `1.0.0-rc`.

## Templates

| Template | Description |
|---|---|
| [build](/templates/build.yml) | Build and push a Docker image with BuildKit |
| [buildchain](/templates/buildchain.yml) | Hadolint scan followed by `build` |

## [build](/templates/build.yml)

Builds a Docker image with rootless BuildKit (`buildctl-daemonless.sh`) and pushes it to a registry. Docker Hub pulls are mirrored through `dockerhub.timeweb.cloud`, `cr.yandex/mirror`, and `mirror.gcr.io`.

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/build/build@1
    inputs:
      image_name: group/project/app
      dockerfile_dir: docker
      push_to_latest: true
```

### Inputs

| Input | Description | Type | Default | Required |
|---|---|---|---|---|
| `image_name` | Full image name including registry path (e.g. `registry.example.com/group/project/app`) | string | — | yes |
| `tag` | Image tag | string | `${CI_COMMIT_REF_SLUG}-${CI_PIPELINE_ID}` | no |
| `latest_image_tag` | Image tag with `latest` suffix. Pushed if `push_to_latest` is `true` | string | `${CI_COMMIT_REF_SLUG}-latest` | no |
| `push_to_latest` | If true also push image to `latest_image_tag` | boolean | `false` | no |
| `context` | Docker build context path | string | `.` | no |
| `dockerfile_dir` | Directory containing the Dockerfile (passed as `--local dockerfile=<dir>`) | string | `.` | no |
| `dockerfile_name` | Dockerfile filename inside `dockerfile_dir` | string | `Dockerfile` | no |
| `registry` | Container registry URL to authenticate against | string | `${CI_REGISTRY_IMAGE}` | no |
| `registry_user` | Registry username | string | `$CI_REGISTRY_USER` | no |
| `registry_password` | Registry password or token | string | `$CI_REGISTRY_PASSWORD` | no |
| `buildctl_extra_args` | Extra flags appended to `buildctl build` (e.g. `--opt build-arg.VERSION=1.0`) | string | `""` | no |
| `job_name` | CI job name | string | `build` | no |
| `runner_tags` | GitLab runner tags | array | `[yandex, docker]` | no |
| `stage` | CI stage for the build job | string | `build` | no |
| `rules_config` | GitLab CI rules applied to the build job | array | MR manual + `dockerfile_dir`/`dockerfile_name` changes | no |
| `needs` | Job needs | array | `[]` | no |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `build-<job_name>` | `stage` | MR/push with changes in `<dockerfile_dir>/<dockerfile_name>` (auto); MR (manual); web (manual) |

## [buildchain](/templates/buildchain.yml)

Wraps [hadolint_scan](../devsec#hadolint_scan) and [build](#build) so a Dockerfile is linted before the image is built. The build job `needs` the hadolint job.

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/build/buildchain@1
    inputs:
      image_name: app
      dockerfile_dir: ./docker
```

### Inputs

| Input | Description | Default | Required |
|---|---|---|---|
| `stage` | Job stage | `build` | no |
| `context` | Build context | `.` | no |
| `dockerfile_dir` | Path to directory with Dockerfile | `./docker` | no |
| `dockerfile_name` | Dockerfile name inside `dockerfile_dir` | `Dockerfile` | no |
| `image_name` | Image name, e.g. `nginx` or `app` | — | yes |
