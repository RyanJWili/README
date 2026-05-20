# Infrastructure overview

Ditto production infrastructure runs on **Google Cloud** in project `petkeley` (`us-central1`), managed as code in the **`ditto-infra`** repository via [Infrastructure Manager](https://cloud.google.com/infrastructure-manager/docs) (managed Terraform).

## What this repo provisions

| Area | Summary |
|------|---------|
| **GKE Autopilot** | Regional cluster for workloads that exceed Cloud Run limits |
| **Networking** | Dedicated `gke` VPC peered to `default` (legacy k3s / DB connectivity) |
| **Service mesh** | Cloud Service Mesh (Istio) for ingress and cluster-local routing |
| **Knative Serving** | Scale-to-zero for Restate handlers (`RestateDeployment` + `deploymentMode: knative`) |
| **Secrets** | Infisical agent injector + Kubernetes auth for workload pods |
| **CI/CD** | Cloud Build previews on PR; apply on merge (approval gate) |

## CI/CD flow

```
PR touching *.tf / *.tfvars
  → Cloud Build preview (terraform plan via Infrastructure Manager)

Merge to main
  → Cloud Build apply (terraform apply, requires approval)

GitHub Actions (parallel)
  → tflint on Terraform changes
```

## Service accounts

- **`infra-manager@petkeley`** — runs Terraform apply/plan (`config.agent`, `editor`)
- **`cb-infra-manager@petkeley`** — Cloud Build triggers (`config.admin`, logging)

## Relationship to application repos

Application services build container images in their own repos; `ditto-infra` defines cluster capacity, namespaces, injectors, and shared platform primitives. Runtime config and secrets are pulled at pod start via Infisical annotations—not committed to git.

## Deeper reading

- [infisical-gke.md](infisical-gke.md) — secret injection pattern on GKE  
- [specs/bootstrap-infra-manager.md](specs/bootstrap-infra-manager.md) — one-time GCP bootstrap  
- [specs/infisical-injector-gaps.md](specs/infisical-injector-gaps.md) — injector auth fixes  
- [../architecture/platform-overview.md](../architecture/platform-overview.md) — application topology  
- Upstream source: `ditto-infra/docs/architecture.md` in the infra repository  
