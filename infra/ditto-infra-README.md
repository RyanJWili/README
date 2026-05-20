# ditto-infra
Ditto Terraform for Infrastructure Manager on GCP.

## Setup

Install [mise](https://mise.jdx.dev/) then run:

```bash
mise install
```

This installs pinned versions of Terraform, tflint, and lefthook, and sets up pre-commit hooks automatically.

## Deploying

Terraform is applied via [Infrastructure Manager](https://cloud.google.com/infrastructure-manager/docs), a managed Terraform service on GCP. You don't run `terraform apply` locally.

1. Open a PR to `main` — Cloud Build runs `terraform plan` via Infrastructure Manager
2. Merge to `main` — Cloud Build runs `terraform apply` (requires manual approval in the GCP console)

All `.tf` files go in the repo root. Infrastructure Manager reads from `.` as its entrypoint.

## Bootstrap

First-time setup for service accounts, IAM, Cloud Build triggers, and the GitHub connection:

```bash
bash bootstrap/setup.sh
```

The script is idempotent. It also prints manual steps for setting up the Infisical machine identity.
