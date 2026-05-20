# Infisical Agent Injector — Gap Fixes

## Context

The `gke-infisical-injector` machine identity in Terraform is missing several required configurations, preventing the Infisical agent injector from authenticating and injecting secrets into workloads in the `ditto` namespace. The workload repo (`profile-analysis-service`) also hardcodes the org ID where the identity ID should be — a known bug that depends on fixing the infra side first.

## Changes

### 1. Project membership (already done)

`infisical_project_membership.injector` grants the identity `member` access to the project. Without this, the identity can authenticate but can't read secrets.

### 2. Ditto namespace in allowed_namespaces (already done)

Added `kubernetes_namespace.ditto.metadata[0].name` to `allowed_namespaces` so workload pods in `ditto` can authenticate.

### 3. Variable for project ID (already done)

Replaced hardcoded workspace ID `e256ddd8-...` with `var.infisical_project_id` in both the project membership resource and the `data.infisical_secrets.restate` data source.

### 4. Add `default` SA to allowed_service_account_names

Workload pods in `ditto` use the `default` service account. The injector sidecar authenticates using the pod's own SA token, so `default` must be in `allowed_service_account_names`.

**File:** `workloads.tf` line 140

### 5. Dedicated token reviewer JWT

Create a `kubernetes.io/service-account-token` Secret for the injector SA and pass it as `token_reviewer_jwt`. This lets the injector call TokenReview on behalf of workload pods without each workload SA needing `system:auth-delegator` RBAC.

**File:** `workloads.tf` — new `kubernetes_secret` resource + `token_reviewer_jwt` field on `infisical_identity_kubernetes_auth.injector`

### 6. GitHub provider + repo variable for identity ID

Add the `github` Terraform provider (authenticated via a PAT read from Infisical) and a `github_actions_variable` resource to set `INFISICAL_IDENTITY_ID` on `dodo-world/profile-analysis-service`. This lets the workload repo's deploy pipeline substitute the correct identity ID into its agent ConfigMap.

**Files:** `versions.tf` (provider block), `workloads.tf` (data source for PAT, variable resource)

## Files Modified

| File | Changes |
|------|---------|
| `versions.tf` | Add `github` provider with token from Infisical |
| `variables.tf` | `infisical_project_id` variable (already done) |
| `workloads.tf` | Project membership (done), allowed_namespaces (done), allowed SAs, token reviewer JWT secret, github_actions_variable |
| `outputs.tf` | Add `infisical_identity_id` output |

## Out of Scope

- Agent config ConfigMap — managed in the workload repo
- Fixing the identity-id value in the workload repo's ConfigMap — separate PR, depends on this
- `infisical_project_membership` vs `infisical_project_identity` rename — verify during `terraform plan`

## Verification

1. `terraform validate` — syntax check
2. `terraform plan` — verify new resources are created correctly, no unexpected changes
3. `tflint --format compact` — lint check
