# Bootstrap: GCP Infrastructure Manager Setup

## Context

The `ditto-infra` repo (`dodo-world/ditto-infra`, private) is an empty skeleton intended to hold Terraform configurations managed by GCP Infrastructure Manager. The GCP project `petkeley` already has all required APIs enabled and the Cloud Build GitHub App installed. This bootstrap fills the remaining gaps with a one-time bash script.

## What Already Exists (No Action Needed)

- GCP project `petkeley` (project number `111182377487`), account `nathan@ditto.ai`
- APIs enabled: `config.googleapis.com`, `cloudbuild.googleapis.com`, `secretmanager.googleapis.com`, `iam.googleapis.com`, `storage.googleapis.com`, and ~85 others
- Cloud Build GitHub App installed on `dodo-world` org (installation ID: `38145556`)
- Service account `proj-coach-infra@petkeley.iam.gserviceaccount.com` (existing, not used by this bootstrap)
- `.gitignore` configured for Terraform

## What Bootstrap Creates

A bash script (`bootstrap/setup.sh`) that:

1. **Cloud Build v2 GitHub connection** via OAuth flow (opens browser, no PAT/token expiry)
2. **Cloud Build v2 repository link** for `dodo-world/ditto-infra`
3. **Infra Manager service account** (`infra-manager@petkeley.iam.gserviceaccount.com`) for executing Terraform
4. **Cloud Build service account** (`cb-infra-manager@petkeley.iam.gserviceaccount.com`) for running triggers
5. **IAM bindings**:
   - `infra-manager` SA: `roles/config.agent`, `roles/editor`
   - `cb-infra-manager` SA: `roles/config.admin`, `roles/iam.serviceAccountUser`, `roles/logging.logWriter`
5. **Cloud Build triggers with inline config** (x2):
   - PR trigger: runs `gcloud infra-manager previews create` on pull requests to `main`
   - Apply trigger: runs `gcloud infra-manager deployments apply` on merge to `main`

## File Structure

```
bootstrap/
  setup.sh              # One-time bootstrap script (gcloud CLI)
  preview.cloudbuild.yaml   # Inline config for PR preview trigger
  apply.cloudbuild.yaml     # Inline config for merge apply trigger
```

The `*.cloudbuild.yaml` files are consumed by `--inline-config` during trigger creation. They are NOT committed to the repo root or referenced by triggers at runtime.

## Key Configuration

| Setting | Value |
|---------|-------|
| Project | `petkeley` |
| Region | `us-central1` |
| Connection name | `dodo-world-github` |
| Repository name | `ditto-infra` |
| Infra Manager SA | `infra-manager@petkeley.iam.gserviceaccount.com` (new) |
| Cloud Build SA | `cb-infra-manager@petkeley.iam.gserviceaccount.com` (new) |
| Terraform version (IM) | 1.5.7 (latest supported by Infrastructure Manager) |
| Deployment ID | `ditto-infra` |
| Deployment ref | `main` |
| TF root directory | Repo root |

## How to Run

```bash
cd bootstrap
chmod +x setup.sh
./setup.sh
# Browser opens for GitHub OAuth — authorize Cloud Build
```

## Post-Bootstrap Workflow

1. Write Terraform `.tf` files in repo root
2. Open PR → CB trigger fires → Infra Manager preview (`terraform plan`)
3. Merge to `main` → CB trigger fires → Infra Manager apply (`terraform apply`)

## Approach

Bash script with gcloud CLI instead of Terraform module. Chosen because:
- One-time setup doesn't benefit from state management
- OAuth connection avoids PAT expiry issues
- Simpler, more transparent, fewer dependencies
- Fresh `infra-manager` SA with clear purpose
