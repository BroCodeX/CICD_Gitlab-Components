# devsec

GitLab CI/CD components for security scanning pipelines.

[[_TOC_]]

## Create release

1. Add an entry to `CHANGELOG.md` before merging to `main`:

    ```md
    ## 1.0.0 (2026-03-23)
    ### Added
    - Added `code_scan` component
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
| [checkov](/templates/checkov.yml) | Static code analysis for Ansible/Terraform (Checkov) |
| [kics](/templates/kics.yml) | Static code analysis for Ansible/Terraform (KICS) |
| [semgrep](/templates/semgrep.yml) | Static application security testing / SAST (Semgrep) |
| [hadolint_scan](/templates/hadolint_scan.yml) | Dockerfile linting (Hadolint) |
| [trivy_scan](/templates/trivy_scan.yml) | Container image vulnerability scanning (Trivy) |
| [dockle_scan](/templates/dockle_scan.yml) | Docker image best-practice and CIS benchmark scanning (Dockle) |

## [checkov](/templates/checkov.yml)

Scans IaC code (e.g. Ansible or Terraform) for security issues using [Checkov](https://www.checkov.io/).

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/devsec/checkov@1
    inputs:
      target: terraform
      severity: HIGH,CRITICAL
```

### Inputs

| Input | Description | Default |
|---|---|---|
| `stage` | Job stage | `scan` |
| `runner_tags` | Runner tags | `[yandex, docker]` |
| `allow_failure` | Allow job to fail | `true` |
| `needs` | Job needs | `[]` |
| `target` | What to scan (`ansible`, `terraform`, etc) | `terraform` |
| `checkov_image` | Image for the Checkov scanner | `bridgecrew/checkov:latest` |
| `framework` | Framework to scan | `all` |
| `output_format` | Output format(s), comma-separated (`cli`, `json`, `junitxml`, `github_failed_only`, `sarif`, `csv`, `spdx`, `cyclonedx`) | `json` |
| `check` | Checks to run, comma-separated check/incident IDs (e.g. `CKV_AWS_1,CKV_AWS_2`). Leave empty to run all | `""` |
| `skip_check` | Checks to skip, comma-separated check/incident IDs | `""` |
| `soft_fail` | Run scan without failing the job regardless of findings (`--soft-fail`) | `false` |
| `soft_fail_on` | Checks/severities that soft-fail even when `severity` would otherwise fail the job (`--soft-fail-on`) | `""` |
| `compact` | Do not display code blocks in CLI output (`--compact`) | `false` |
| `quiet` | Display only failed checks in CLI output (`--quiet`) | `false` |
| `download_external_modules` | Download external terraform modules before scanning (`--download-external-modules`) | `false` |
| `config_file` | Path to a Checkov config file (`--config-file`) | `""` |
| `var_file` | Path to a terraform tfvars file (`--var-file`) | `""` |
| `baseline` | Path to a Checkov baseline file to suppress known findings (`--baseline`) | `""` |
| `extra_args` | Extra raw CLI args, appended as-is to the `checkov` command | `""` |
| `severity` | Severity levels to fail on (`HIGH`, `CRITICAL`, `MEDIUM`, `LOW`, `UNKNOWN`), comma-separated. Leave empty to disable | `HIGH,CRITICAL` |
| `rules_checkov_config` | GitLab CI rules | MR manual + branch |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `checkov-<target>` | `stage` | `rules_checkov_config` |

## [kics](/templates/kics.yml)

Scans IaC code (e.g. Ansible or Terraform) for security issues using [KICS](https://kics.io/).

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/devsec/kics@1
    inputs:
      target: ansible
      severity: HIGH,CRITICAL
```

### Inputs

| Input | Description | Default |
|---|---|---|
| `stage` | Job stage | `scan` |
| `runner_tags` | Runner tags | `[yandex, docker]` |
| `allow_failure` | Allow job to fail | `true` |
| `needs` | Job needs | `[]` |
| `target` | What to scan (`ansible`, `terraform`, etc) | `ansible` |
| `kics_image` | Image for the KICS scanner | `checkmarx/kics:alpine` |
| `report_format` | Report format | `json` |
| `severity` | Severity levels to fail on (`HIGH`, `CRITICAL`, `MEDIUM`, `LOW`, `UNKNOWN`), comma-separated | `HIGH,CRITICAL` |
| `rules_kics_config` | GitLab CI rules | MR manual + branch |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `kics-<target>` | `stage` | `rules_kics_config` |

## [semgrep](/templates/semgrep.yml)

Runs [Semgrep](https://semgrep.dev/) SAST scans. Supports Semgrep registry rulesets (`auto`, `p/secrets`, `p/ci`, ...) alongside custom rule directories/files, combined freely.

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/devsec/semgrep@1
    inputs:
      target: src
      configs: "auto,p/secrets"
      custom_rules_dir: "sast-rules"
      severity: ERROR,WARNING
```

### Inputs

| Input | Description | Default |
|---|---|---|
| `job_name` | Job name suffix and report file naming | `code` |
| `stage` | Job stage | `scan` |
| `runner_tags` | Runner tags | `[yandex, docker]` |
| `allow_failure` | Allow job to fail | `true` |
| `needs` | Job needs | `[]` |
| `semgrep_image` | Image for the Semgrep scanner | `semgrep/semgrep:latest` |
| `target` | Path (relative to repo root) to scan | `.` |
| `configs` | Rulesets to run, comma-separated (registry configs like `auto`, `p/secrets`, `p/ci`, or local paths) | `auto` |
| `custom_rules_dir` | Path to a directory of custom Semgrep rule YAML files; every rule file in it is loaded. Leave empty to skip | `""` |
| `custom_rules_files` | Specific custom rule files to load, comma-separated | `""` |
| `exclude` | Glob patterns to exclude, comma-separated (`--exclude`) | `""` |
| `include` | Glob patterns to restrict the scan to, comma-separated (`--include`) | `""` |
| `severity` | Severities to report, comma-separated (`ERROR`, `WARNING`, `INFO`). Leave empty for all | `""` |
| `fail_on_findings` | Fail the job when matching findings are found (`--error`) | `true` |
| `output_format` | Report format saved as an artifact (`json` \| `sarif` \| `gitlab-sast` \| `junit-xml`) | `json` |
| `extra_args` | Extra raw CLI args, appended as-is to the `semgrep` command | `""` |
| `rules_config` | GitLab CI rules | MR manual + branch |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `semgrep-<job_name>` | `stage` | `rules_config` |

### Notes

- `configs`, `custom_rules_dir`, and `custom_rules_files` are all additive — each becomes its own `--config` flag, so registry rulesets and custom rules run together in one scan.
- A report is always saved to `security-reports/semgrep-<job_name>.<ext>` regardless of exit status; the job only fails afterwards if `fail_on_findings` is `true`.

## [hadolint_scan](/templates/hadolint_scan.yml)

Lints a Dockerfile for best-practice violations using [Hadolint](https://github.com/hadolint/hadolint).

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/devsec/hadolint_scan@1
    inputs:
      job_name: app
      dockerfile_dir: docker
      severity: error
```

### Inputs

| Input | Description | Default |
|---|---|---|
| `job_name` | Label used in job and report naming | `lint-dockerfile` |
| `stage` | Job stage | `scan` |
| `runner_tags` | Runner tags | `[yandex, docker]` |
| `allow_failure` | Allow job to fail | `true` |
| `needs` | Job needs | `[]` |
| `hadolint_image` | Image for the Hadolint scan | `hadolint/hadolint:latest-alpine` |
| `dockerfile_dir` | Path to the Dockerfile from the repo root | `./docker` |
| `dockerfile_name` | Dockerfile name inside `dockerfile_dir` | `Dockerfile` |
| `severity` | Minimum severity to fail on (`error` \| `warning` \| `info` \| `style` \| `ignore` \| `none`) | `error` |
| `output_format` | CI log output format (`tty` \| `json` \| `checkstyle` \| `codeclimate` \| `gitlab_codeclimate` \| `gnu` \| `codacy` \| `sonarqube` \| `sarif` \| `junit`) | `tty` |
| `rules_config` | GitLab CI rules | MR manual + branch |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `hadolint-<job_name>` | `stage` | `rules_config` |

## [trivy_scan](/templates/trivy_scan.yml)

Scans container images for vulnerabilities using Trivy. Produces CycloneDX SBOM and JSON reports.

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/devsec/trivy_scan@1
    inputs:
      image_to_scan: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
      scanners: vuln,secret
      severity: HIGH,CRITICAL
```

### Inputs

| Input | Description | Default |
|---|---|---|
| `stage` | Job stage | `scan` |
| `runner_tags` | Runner tags | `[yandex, docker]` |
| `rules_config` | GitLab CI rules | MR manual + branch |
| `allow_failure` | Allow job to fail | `true` |
| `needs` | Job needs | `[]` |
| `image_to_scan` | Image reference to scan | `""` |
| `scanners` | What to scan (`vuln`, `misconfig`, `secret`, `license`), comma-separated | `vuln` |
| `severity` | Severity levels to display (`UNKNOWN`, `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`), comma-separated | `HIGH,CRITICAL` |
| `ignore_unfixed` | Skip vulnerabilities with no fix | `true` |
| `trivy_image` | Trivy Docker image | `aquasec/trivy:latest` |
| `job_name` | Job name suffix | `images` |
| `registry` | Container registry URL to authenticate, e.g. `${CI_REGISTRY_IMAGE}` | `""` |
| `registry_user` | Container registry username, e.g. `$CI_REGISTRY_USER` | `""` |
| `registry_password` | Container registry password, e.g. `$CI_REGISTRY_PASSWORD` | `""` |
| `sbom_format` | Output format for the SBOM report (`cyclonedx` \| `spdx` \| `spdx-json` \| `github`) | `cyclonedx` |
| `report_format` | Output format for the vulnerability report (`json` \| `table` \| `sarif` \| `template` \| `cosign-vuln`) | `json` |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `trivy-<job_name>` | `stage` | `rules_config` |

## [dockle_scan](/templates/dockle_scan.yml)

Scans a built Docker image for CIS benchmark violations and best-practice issues using [Dockle](https://github.com/goodwithtech/dockle). Downloads the Dockle binary at runtime — no Docker socket or privileged mode required.

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/devsec/dockle_scan@1
    inputs:
      job_name: app
      image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
      exit_level: fatal
      registry: $CI_REGISTRY_IMAGE
      registry_user: $CI_REGISTRY_USER
      registry_password: $CI_REGISTRY_PASSWORD
```

### Inputs

| Input | Description | Default |
|---|---|---|
| `stage` | Job stage | `scan` |
| `runner_tags` | Runner tags | `[yandex, docker]` |
| `allow_failure` | Allow job to fail | `true` |
| `needs` | Job needs | `[]` |
| `job_name` | Label used in job and report naming | `image` |
| `image` | Docker image reference to scan | `$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA` |
| `exit_level` | Minimum level to fail the job (`fatal` \| `warn` \| `info`) | `fatal` |
| `output_format` | CI log output format (`list` \| `json` \| `sarif`) | `list` |
| `dockle_version` | Dockle binary version to download | `0.4.14` |
| `registry` | Container registry URL to authenticate, e.g. `${CI_REGISTRY_IMAGE}` | `""` |
| `registry_user` | Container registry username, e.g. `$CI_REGISTRY_USER` | `""` |
| `registry_password` | Container registry password, e.g. `$CI_REGISTRY_PASSWORD` | `""` |
| `rules_config` | GitLab CI rules | MR manual + branch |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `dockle-<job_name>` | `stage` | `rules_config` |

### Notes

- Registry auth via `DOCKLE_USERNAME` / `DOCKLE_PASSWORD` is only active when `registry`, `registry_user`, and `registry_password` are explicitly set — they default to empty strings.
- A JSON report is always saved to `security-reports/dockle-<job_name>.json` regardless of exit status.
- To ignore specific checkpoints, set `DOCKLE_IGNORES=CIS-DI-0001,DKL-DI-0006` as a CI variable or add a `.dockleignore` file to the repo root.
