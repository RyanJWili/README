# Architecture

## Overview

GCP infrastructure for the `petkeley` project, managed via [Infrastructure Manager](https://cloud.google.com/infrastructure-manager/docs) (a managed Terraform service). Terraform files live at the repo root.

## GCP Configuration

| Setting | Value |
|---------|-------|
| Project | `petkeley` |
| Region | `us-central1` |
| GitHub org | `dodo-world` |
| Deployment ID | `ditto-infra` |
| Terraform version | 1.5.7 (pinned — max supported by Infrastructure Manager) |
| Google provider | `~> 6.0` |

## Service Accounts

- **`infra-manager@petkeley.iam.gserviceaccount.com`** — executes Terraform via Infrastructure Manager (`roles/config.agent`, `roles/editor`)
- **`cb-infra-manager@petkeley.iam.gserviceaccount.com`** — runs Cloud Build triggers (`roles/config.admin`, `roles/logging.logWriter`)

## CI/CD Pipeline

Two Cloud Build triggers, both scoped to `**/*.tf` and `**/*.tfvars` changes:

1. **PR to `main`** → `bootstrap/preview.cloudbuild.yaml` → Infrastructure Manager preview (`terraform plan`)
2. **Merge to `main`** → `bootstrap/apply.cloudbuild.yaml` → Infrastructure Manager apply (`terraform apply`, requires approval)

Additionally, GitHub Actions runs **tflint** on PRs that touch `*.tf` or `.tflint.hcl`.

## GKE Autopilot

A regional GKE Autopilot cluster (`autopilot`) in `us-central1`, replacing Cloud Run for workloads that exceed its hard platform limits (32 MiB HTTP/1 request body, 60-min timeout, 8 vCPU / 32 GiB memory cap, no persistent storage).

### Networking

Dedicated custom-mode VPC (`gke`) with a single subnet, peered to the `default` VPC for connectivity to existing k3s VMs and databases.

| Resource | Name | Details |
|----------|------|---------|
| VPC | `gke` | Custom-mode, no auto subnets |
| Subnet | `gke-us-central1` | `172.16.0.0/20` (nodes) |
| Pod range | `pods` | `172.20.0.0/14` (~262K IPs) |
| Service range | `services` | `172.24.0.0/20` (4,094 IPs) |
| Control plane | — | `172.16.16.0/28` |
| Router | `gke-router` | us-central1 |
| NAT | `gke-nat` | AUTO_ONLY IPs, scoped to GKE subnet |
| Peering | `gke-to-default` / `default-to-gke` | Bidirectional, no custom route export |

### Cluster Config

- **Fleet registered** to `petkeley` project (auto-registered at creation)
- **Private nodes** with public control plane endpoint (no VPN needed for kubectl)
- **Release channel:** REGULAR
- **Workload Identity:** enabled by default (Autopilot)

### Cloud Service Mesh

Managed Cloud Service Mesh (`MANAGEMENT_AUTOMATIC`) provisioned via fleet API. Provides the networking layer (Istio ingress/cluster-local gateways) required by Knative Serving.

### Knative Serving

Open-source Knative Serving installed via the [Knative Operator](https://knative.dev/docs/install/operator/knative-with-operators/) Helm chart, using the existing Cloud Service Mesh (Istio) as the networking layer (`net-istio`). Provides scale-to-zero serverless workloads for Restate service handlers.

Deployed in two phases (the `KnativeServing` CRD requires the operator to be deployed first):
1. **Phase 1:** Knative Operator Helm release (Terraform)
2. **Phase 2:** `KnativeServing` CR with Istio ingress (Terraform, separate apply)

Individual Restate handlers are deployed as `RestateDeployment` CRDs with `deploymentMode: knative`, managed by the Restate Operator.

## Bootstrap

`bootstrap/setup.sh` is a one-time script that provisions the Cloud Build v2 GitHub connection (OAuth), service accounts, IAM bindings, and triggers. See `docs/superpowers/specs/2026-04-10-bootstrap-infra-manager-design.md` for the full design rationale.
