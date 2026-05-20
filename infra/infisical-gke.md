# Infisical on GKE

Workloads receive secrets at runtime through the **Infisical Kubernetes Agent Injector**, not from `.env` files in images.

## Components

| Piece | Role |
|-------|------|
| Helm chart `infisical-agent-injector` | Mutating webhook; mounts agent sidecar / init semantics per chart version |
| Namespace `infisical-system` | Operator + injector deployment |
| `infisical_identity` (Terraform) | Machine identity `gke-infisical-injector` bound to project |
| Kubernetes auth | Maps allowed namespaces + service accounts to Infisical identity |
| Pod annotations | Select secret path / environment per deployment |

## Auth model

1. Terraform creates Infisical identity and project binding.  
2. Injector service account gets a bound token reviewer secret.  
3. `infisical_identity_kubernetes_auth` restricts which K8s service accounts may assume the identity (injector SA + workload SAs in `ditto` namespace).  
4. Application pods annotated for injection receive secrets without storing them in the manifest.

## Operations checklist

- Rotating secrets: update in Infisical UI/API; restart affected deployments if not hot-reloaded.  
- New service: add service account to `infisical_workload_service_accounts` in Terraform before first deploy.  
- Never commit API keys or LangSmith/OpenAI tokens—use Infisical paths referenced only in cluster config.

## Related

- [overview.md](overview.md)  
- Application repos document *which* secret keys they expect; this doc covers *how* they arrive on cluster.

*Synthesized from `ditto-infra/workloads.tf` and architecture docs—verify chart version and annotation keys in repo before changing production.*
