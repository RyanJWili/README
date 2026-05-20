# profile-analysis-service

**Runtime:** Bun with Restate (Knative on GKE)  
**Role:** LLM-assisted profile analysis, embeddings, scoring rounds, and batch matchmaking helpers.

## Why it exists

Keeps CPU- and LLM-heavy work off the main API process. Handlers are durable workflows: retries, parallelism, and long-running scoring without blocking HTTP requests in `proj-coach-backend`.

## Typical flows

1. **Autofilter / retrieval** — narrow candidate sets (often with MeiliSearch)  
2. **Scoring handlers** — feature-based and model-based scores per pair  
3. **Stable matching rounds** — scheduled jobs coordinating weekly pools  
4. **Embeddings** — profile text → vector features for search and models  

## Deployment

Runs on GKE with Infisical-injected secrets. Identity must match Terraform `infisical_workload_service_accounts`—see [../../infra/specs/infisical-injector-gaps.md](../../infra/specs/infisical-injector-gaps.md).

## Data

Reads/writes MongoDB match and profile collections defined in `proj-coach-schemas`. Publishes scores consumed by backend matching orchestration.

## Related

- [../../reference/matchmaking/engine-3x-summary.md](../../reference/matchmaking/engine-3x-summary.md)  
- [../../reference/matchmaking/implementation-plan-summary.md](../../reference/matchmaking/implementation-plan-summary.md)  
