# CICD_GITLAB_COMPONENTS

Reusable GitLab CI/CD components for building, deploying, and scanning projects.

## Components

| Component | Description |
|---|---|
| [build](build/README.md) | Build and publish Docker images |
| [deploy](deploy/README.md) | Deploy via Ansible, Helm, or Terraform |
| [devsec](devsec/README.md) | Security scanning (Checkov, KICS, Hadolint, Trivy, Dockle) |

## Usage

Include a component in your `.gitlab-ci.yml`:

```yaml
include:
  - component: $CI_SERVER_FQDN/components/build/build@1
    inputs:
      image_name: group/project/app
```

See each component's README for available templates, inputs, and jobs.

## License

[MIT](LICENSE)
