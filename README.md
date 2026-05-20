# Ditto documentation hub

Curated architecture, diagrams, strategy, and reference material for the Ditto campus matchmaking platform. Content here is **written for navigation**—clean filenames, synthesized prose, and cross-links—not a dump of Notion export names or full repo README copies.

**Mirrors:** [github.com/RyanJWili/README](https://github.com/RyanJWili/README)  
**Upstream code org:** [dodo-world](https://github.com/dodo-world)

---

## Table of contents

### [Architecture](architecture/README.md)

| | |
|---|---|
| [Platform overview](architecture/platform-overview.md) | Product channels, logical stack, repository map |
| [Service catalog](architecture/service-catalog.md) | Per-service ownership |
| [Chatbot](architecture/chatbot.md) | SMS agent in `proj-coach-backend` |
| [Chatbot evolution](architecture/chatbot-evolution.md) | LangGraph → skills migration |
| [Service deep dives](architecture/services/README.md) | Backend, imsg, analysis, prompts, admin UI |
| [Matching priority](architecture/operations/matching-priority.md) | ENG-1030 tiers |
| [Linq spam guard](architecture/operations/linq-spam-guard.md) | Line health runbook |

### [Diagrams](diagrams/README.md)

| | |
|---|---|
| [Platform topology](diagrams/platform-topology.md) | Services and data flows |
| [Data model](diagrams/data-model.md) | Mongo entities (conceptual) |
| [SMS pipeline](diagrams/sms-pipeline.md) | Inbound/outbound messaging |
| [Match state machine](diagrams/match-state-machine.md) | Match lifecycle |
| [Chatbot routing](diagrams/chatbot-routing.md) | Intent → skill/tool path |

### [Docs](docs/README.md)

**Strategy**

- [Q2 roadmap](docs/strategy/q2-roadmap.md)
- [Monorepo RFC summary](docs/strategy/monorepo-rfc-summary.md)
- [Monorepo migration summary](docs/strategy/monorepo-migration-summary.md)
- [Claude governance summary](docs/strategy/claude-governance-summary.md)

**Chatbot**

- [System overview](docs/chatbot/system-overview.md)
- [Production audit](docs/chatbot/production-audit.md)
- [Onboarding](docs/chatbot/onboarding.md)
- [Skills migration](docs/chatbot/skills-migration.md)
- [Yik Yak event](docs/chatbot/yak-event.md)
- [Outbound latency](docs/chatbot/outbound-latency.md)

**Integrations**

- [OpenClaw](docs/integrations/openclaw.md)

### [Infrastructure](infra/README.md)

- [Overview](infra/overview.md) — GKE, Terraform, CI/CD
- [Infisical on GKE](infra/infisical-gke.md) — runtime secrets
- [Bootstrap spec](infra/specs/bootstrap-infra-manager.md) — Infrastructure Manager setup
- [Injector gaps spec](infra/specs/infisical-injector-gaps.md) — workload secret auth fixes

### [Reference](reference/README.md)

- [Matchmaking 3.x summary](reference/matchmaking/engine-3x-summary.md)
- [Implementation plan summary](reference/matchmaking/implementation-plan-summary.md)
- [Technical review notes](reference/matchmaking/technical-review-notes.md)

### [Repositories](repos/README.md)

Service catalog with GitHub links—install and run instructions stay in each repo’s own README.

---

## What is intentionally excluded

| Excluded | Reason |
|----------|--------|
| `platform/` (ditto-platform) | Monorepo not submitted to this bundle |
| `findings/` (Mongo analytics) | Operational research, not product architecture |
| Raw Notion filenames | Replaced by the paths above |
| Full README copies | Avoid drift and secret leakage from dev examples |

---

## How to extend this hub

1. Add a short synthesized doc under the right folder (`architecture/`, `docs/`, `reference/`, etc.).  
2. Link it from the section `README.md` and from this file.  
3. Prefer diagrams in `diagrams/` when a picture helps more than prose.  
4. Never commit API keys—reference Infisical paths in infra docs only.
