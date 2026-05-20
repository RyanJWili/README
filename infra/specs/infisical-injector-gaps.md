# Infisical injector — gap fixes (design)

Design notes for making the GKE Infisical agent injector authenticate workload pods in the `ditto` namespace.

## Problem

Machine identity `gke-infisical-injector` was missing pieces required for:

- Reading secrets from the Ditto Infisical project  
- Authenticating pods in `ditto` (not only `infisical-system`)  
- TokenReview without granting every workload SA `system:auth-delegator`  

## Required changes (checklist)

| # | Change | Status pattern |
|---|--------|----------------|
| 1 | `infisical_project_membership` for injector identity | Required for secret read |
| 2 | Add `ditto` namespace to `allowed_namespaces` | Workload pods eligible |
| 3 | `var.infisical_project_id` instead of hardcoded workspace ID | Config hygiene |
| 4 | Include `default` SA in `allowed_service_account_names` | Pods using default SA |
| 5 | Dedicated `kubernetes.io/service-account-token` secret as `token_reviewer_jwt` | Injector TokenReview |
| 6 | GitHub provider + `INFISICAL_IDENTITY_ID` variable on workload repo | Deploy pipeline substitution |

## Out of scope (separate PRs)

- Agent ConfigMap content in application repos  
- Fixing hardcoded org-vs-identity ID in workload ConfigMap templates  

## Verification

1. `terraform validate`  
2. `terraform plan` — no unexpected destroys  
3. `tflint`  
4. Deploy test pod in `ditto` with injection annotations → secret mounts present  

## Related

- [../infisical-gke.md](../infisical-gke.md)  
- [../../architecture/services/profile-analysis-service.md](../../architecture/services/profile-analysis-service.md)  

*Synthesized from `ditto-infra/docs/superpowers/specs/2026-04-13-infisical-injector-gaps-design.md`.*
