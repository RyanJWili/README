# Bootstrap — Infrastructure Manager

One-time setup for managing `ditto-infra` Terraform through GCP Infrastructure Manager.

## Preconditions (already in petkeley)

- APIs: `config.googleapis.com`, `cloudbuild.googleapis.com`, Secret Manager, IAM, Storage, etc.  
- Cloud Build GitHub App on `dodo-world` org  
- Terraform `.gitignore` in repo  

## What `bootstrap/setup.sh` creates

| Resource | Purpose |
|----------|---------|
| Cloud Build v2 GitHub connection | OAuth (no expiring PAT) |
| Repository link | `dodo-world/ditto-infra` |
| `infra-manager@petkeley` SA | Runs `terraform plan/apply` via IM |
| `cb-infra-manager@petkeley` SA | Executes Cloud Build triggers |
| IAM | `config.agent` + `editor` (infra SA); `config.admin` + logging (CB SA) |
| Triggers ×2 | PR → preview plan; merge → apply (approval) |

Inline build configs live in `bootstrap/preview.cloudbuild.yaml` and `bootstrap/apply.cloudbuild.yaml`—referenced at trigger creation, not as runtime repo paths.

## Key settings

| Setting | Value |
|---------|-------|
| Project | `petkeley` |
| Region | `us-central1` |
| Terraform (IM) | 1.5.7 max |
| Deployment ID | `ditto-infra` |
| TF root | Repository root |

## Post-bootstrap workflow

1. Author `.tf` in repo root  
2. Open PR → Infrastructure Manager preview  
3. Merge to `main` → apply (with approval gate)  
4. `tflint` on PR via GitHub Actions  

## Why bash, not Terraform for bootstrap

One-time setup; OAuth connection is clearer in script; avoids circular dependency on IM before IM exists.

*Synthesized from `ditto-infra/docs/superpowers/specs/2026-04-10-bootstrap-infra-manager-design.md`.*
