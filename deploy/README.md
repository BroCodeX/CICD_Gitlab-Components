# deploy

GitLab CI/CD components for deployment pipelines.

[[_TOC_]]

## Create release

1. Add an entry to `CHANGELOG.md` before merging to `main`:

    ```md
    ## 1.0.0 (2026-03-23)
    ### Added
    - Added `ansible` component
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
| [ansible](/templates/ansible.yml) | Run Ansible playbooks (check + deploy) |
| [helm](/templates/helm.yml) | Deploy a Helm chart (lint + diff + apply + uninstall) |
| [terraform](/templates/terraform.yml) | Manage a Terraform module (validate + plan + apply + destroy) |

## [ansible](/templates/ansible.yml)

Runs an Ansible playbook with a check (dry-run) stage followed by a deploy stage.

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/deploy/ansible@1
    inputs:
      playbook: vault
      limit: production
      ansible_tags: vault
      ansible_image: $CI_REGISTRY_IMAGE/ansible:latest
      has_vault: true
```

### Inputs

| Input | Description | Type | Default | Required |
|---|---|---|---|---|
| `playbook` | Playbook name (`keycloak`, `vault`, `minikube`) | string | — | yes |
| `ansible_image` | Docker image with ansible | string | — | yes |
| `inventory` | Path to inventory file | string | `ansible/inventory/inventory.yml` | no |
| `limit` | Limit to host or group names (comma-separated) | string | `minikube` | no |
| `ansible_tags` | Ansible tags (comma-separated) | string | `minikube` | no |
| `extra_args` | Extra args passed to `ansible-playbook` | string | `""` | no |
| `has_vault` | Use `--vault-password-file` | boolean | `false` | no |
| `runner_tags` | GitLab runner tags | array | `[yandex, docker]` | no |
| `check_stage` | Check job stage | string | `check` | no |
| `stage` | Deploy job stage | string | `deploy` | no |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `<playbook> - <limit> - <ansible_tags> - check` | `check_stage` | MR to `master` (manual); push to `master` with changes in `ansible/` (auto); push to `master` (manual); web (manual) |
| `<playbook> - <limit> - <ansible_tags> - deploy` | `stage` | Push to `master` with changes in `ansible/` (auto); push to `master` (manual); web (manual) |

### Variables

The following CI/CD variables must be set in your project or group settings:

| Variable | Description |
|---|---|
| `ANSIBLE_SSH_PRIVATE_KEY` | Base64-encoded SSH private key |
| `SSH_KNOWN_HOSTS` | Base64-encoded `known_hosts` file |
| `ANSIBLE_VAULT_PASSWORD_FILE` | Path to vault password file (required when `has_vault: true`) |

## [helm](/templates/helm.yml)

Deploys a Helm chart through `lint` → `diff` → `apply` stages, plus a manual `uninstall` job. Uses [helm-diff](https://github.com/databus23/helm-diff) to preview changes before `helm upgrade --install`.

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/deploy/helm@1
    inputs:
      release: my-app
      chart: ./my-app
      namespace: production
      helm_image: $CI_REGISTRY_IMAGE/helm:latest
      values: values.yaml values.production.yaml
```

### Inputs

| Input | Description | Type | Default | Required |
|---|---|---|---|---|
| `release` | Helm release name | string | `""` | yes |
| `chart` | Chart to deploy (local path under `helm_dir`, or a repo/OCI reference) | string | — | yes |
| `namespace` | Kubernetes namespace | string | — | yes |
| `helm_image` | Docker image with helm and the helm-diff plugin | string | — | yes |
| `helm_dir` | Path to the Helm charts root from the repo root | string | `helm` | no |
| `chart_version` | Chart version (`--version`); leave empty for local charts | string | `""` | no |
| `values` | Values files, space separated (each passed with `-f`) | string | `""` | no |
| `create_namespace` | Create the namespace if it does not exist (`--create-namespace`) | boolean | `true` | no |
| `atomic` | Roll back on a failed apply (`--rollback-on-failure`) | boolean | `true` | no |
| `wait` | Wait for resources to become ready (`--wait`) | boolean | `true` | no |
| `timeout` | Time to wait for any individual operation (`--timeout`) | string | `5m` | no |
| `kube_context` | Kubeconfig context to use (`--kube-context`); leave empty for the current context | string | `""` | no |
| `kubeconfig` | Path to a base64-encoded kubeconfig file, as produced by a GitLab CI/CD variable of type File (`--kubeconfig`); leave empty for the default | string | `$KUBECONFIG` | no |
| `extra_args` | Extra args shared by lint, diff and apply (e.g. `--set key=val`) | string | `""` | no |
| `extra_lint_args` | Extra args for `helm lint` only | string | `""` | no |
| `extra_diff_args` | Extra args for `helm diff` only | string | `""` | no |
| `extra_apply_args` | Extra args for `helm upgrade` only | string | `""` | no |
| `extra_uninstall_args` | Extra args for `helm uninstall` only (e.g. `--keep-history`) | string | `""` | no |
| `needs` | Job needs | array | `[]` | no |
| `runner_tags` | GitLab runner tags | array | `[yandex, docker]` | no |
| `lint_stage` | Lint job stage | string | `lint` | no |
| `diff_stage` | Diff job stage | string | `diff` | no |
| `stage` | Apply job stage | string | `deploy` | no |
| `rules_config` | GitLab CI rules applied to the lint, diff and apply jobs | array | MR to `main` (manual); push to `main` with changes in `helm_dir` (auto); push to `main` (manual); web (manual) | no |
| `allow_failure` | Allow job to fail | boolean | `true` | no |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `<release> - lint` | `lint_stage` | `rules_config` |
| `<release> - diff` | `diff_stage` | `rules_config` (needs `lint`) |
| `<release> - apply` | `stage` | `rules_config` (needs `diff`) |
| `<release> - uninstall` | `stage` | manual only (needs `apply`) |

## [terraform](/templates/terraform.yml)

Manages a Terraform module through `validate` → `plan` → `apply` stages, plus a manual `destroy` job. Caches the plugin cache dir and `.terraform` keyed on the module's `.terraform.lock.hcl`.

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/components/deploy/terraform@1
    inputs:
      module: vault
      terraform_image: $CI_REGISTRY_IMAGE/terraform:latest
```

### Inputs

| Input | Description | Type | Default | Required |
|---|---|---|---|---|
| `module` | Terraform module to deploy (subdirectory name under `terraform_dir`); one of `keycloak`, `vault` | string | — | yes |
| `terraform_image` | Docker image with terraform | string | — | yes |
| `terraform_dir` | Path to terraform root from the repo root | string | `terraform` | no |
| `needs` | Job needs for the validate job | array | `[]` | no |
| `runner_tags` | GitLab runner tags | array | `[yandex, docker]` | no |
| `validate_stage` | Validate job stage | string | `validate` | no |
| `plan_stage` | Plan job stage | string | `plan` | no |
| `stage` | Apply job stage | string | `deploy` | no |
| `terraform_parallelism` | Terraform parallelism (`-parallelism`) | string | `10` | no |
| `tf_plugin_cache_dir` | Terraform plugin cache directory | string | `${CI_PROJECT_DIR}/.terraform-plugin-cache` | no |
| `plan_file` | Terraform plan output file name (relative to module directory) | string | `plan.tfplan` | no |
| `terraform_cli_config` | Path to terraform CLI config file (sets `TF_CLI_CONFIG_FILE` when set) | string | `.terraform-cli-config.tfrc` | no |
| `extra_init_args` | Extra args for `terraform init` | string | `""` | no |
| `extra_plan_args` | Extra args for `terraform plan` | string | `""` | no |
| `extra_apply_args` | Extra args for `terraform apply` | string | `""` | no |
| `extra_destroy_args` | Extra args for `terraform destroy` | string | `""` | no |
| `allow_failure` | Allow job to fail | boolean | `true` | no |
| `rules_config` | GitLab CI rules applied to validate, plan, and apply jobs | array | MR to `main` (manual); push to `main` with changes in `terraform_dir/module` (auto); push to `main` (manual); web (manual) | no |

### Jobs

| Job | Stage | Trigger |
|---|---|---|
| `<module> - validate` | `validate_stage` | `rules_config` |
| `<module> - plan` | `plan_stage` | `rules_config` (needs `validate`) |
| `<module> - apply` | `stage` | `rules_config` (needs `plan` artifacts) |
| `<module> - destroy` | `stage` | manual only (needs `apply`) |
