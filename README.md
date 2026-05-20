# Ditto documentation hub

A single index for Ditto **architecture**, **diagrams**, **strategy docs**, **matchmaking reference**, and **per-service READMEs** from the production codebase.

Content is aggregated from the workspace `docs/` tree, `ditto-infra`, and `projects/*`. It is documentation only—not a runnable monorepo. For behavior and APIs, use the live service repositories.

**Not included:** `ditto-platform` (Nx monorepo) — not published in this bundle yet.

---

## What is Ditto?

Ditto is an AI-powered campus matchmaking product. Users interact primarily via **SMS/iMessage**; an AI chatbot handles onboarding, profile help, and match updates. Operations teams use an **internal dashboard** for matchmaking, messaging, and prompts. Infrastructure runs on **GCP (GKE)** with **MongoDB**, **Redis**, **MeiliSearch**, and **RabbitMQ**.

---

## Documentation map

```
README.md                    ← you are here
├── architecture/            Platform & service architecture, runbooks
├── diagrams/                Mermaid ERDs and schema graphs
├── docs/                    Product strategy, roadmaps, chatbot specs
├── infra/                   Terraform / GKE / CI (ditto-infra)
├── reference/               Matchmaking 3.x design & external refs
└── repos/                   README copy per service repo
```

---

## 1. Architecture

**Entry point:** [architecture/system-architecture-overview.md](architecture/system-architecture-overview.md)

| Link | Description |
|------|-------------|
| [architecture/README.md](architecture/README.md) | Full index |
| [architecture/ditto-internal-frontend.md](architecture/ditto-internal-frontend.md) | Admin UI structure |
| [architecture/profile-analysis-service.md](architecture/profile-analysis-service.md) | Profile analysis service |
| [architecture/proj-coach-langgraph-migration.md](architecture/proj-coach-langgraph-migration.md) | Chatbot migration (LangGraph → skills) |
| Runbooks under `architecture/` | Rollouts, Linq guard, chatbot state, ENG notes |

**ASCII overview (high level):**

```
Users (SMS/iMessage) → imsg-service → proj-coach-backend ←→ MongoDB / Redis / MeiliSearch
                              ↑              ↓
                    ditto-internal-frontend   RabbitMQ → profile-analysis, delayed-task, prompt-manager
Admin (browser) ───────────────────────────── ditto-internal-ws
```

---

## 2. Diagrams

**Index:** [diagrams/README.md](diagrams/README.md)

| Link | Description |
|------|-------------|
| [diagrams/proj-coach-schemas-architecture.md](diagrams/proj-coach-schemas-architecture.md) | Service graph + MongoDB collections (Mermaid) |
| [diagrams/ufl-erd.md](diagrams/ufl-erd.md) | UFL / `sms_chat_segments` ERD (Mermaid) |

These diagrams match the current schema packages in `proj-coach-schemas` and the UFL tool. Open in GitHub preview or VS Code Mermaid support.

---

## 3. Strategy & product docs

**Index:** [docs/README.md](docs/README.md)

| Priority | Document |
|----------|----------|
| Roadmap | [docs/q2-roadmap-and-initiatives.md](docs/q2-roadmap-and-initiatives.md) |
| Monorepo RFC | [docs/rfc-monorepo-driven-development.md](docs/rfc-monorepo-driven-development.md) |
| Chatbot audit | [docs/chatbot-pipeline-production-issues.md](docs/chatbot-pipeline-production-issues.md) |
| Onboarding | [docs/conversational-onboarding-agent.md](docs/conversational-onboarding-agent.md) |
| Latency RCA | [docs/outbound-message-latency-rca.md](docs/outbound-message-latency-rca.md) |

---

## 4. Infrastructure

**Index:** [infra/README.md](infra/README.md)

| Link | Description |
|------|-------------|
| [infra/gcp-architecture.md](infra/gcp-architecture.md) | GCP topology, GKE, peering, Infisical |
| [infra/ditto-infra-README.md](infra/ditto-infra-README.md) | Repo setup and Cloud Build triggers |

---

## 5. Matchmaking reference

**Index:** [reference/README.md](reference/README.md)

| Link | Description |
|------|-------------|
| [reference/ditto-matchmaking/matchmaking-engine-3x.md](reference/ditto-matchmaking/matchmaking-engine-3x.md) | Engine 3.x |
| [reference/ditto-matchmaking/proposed-strategy-implementation-plan.md](reference/ditto-matchmaking/proposed-strategy-implementation-plan.md) | Strategy + implementation plan |

---

## 6. Service repositories

**Index:** [repos/INDEX.md](repos/INDEX.md)

Each service has a copied README under `repos/<name>-README.md` (e.g. [proj-coach-backend](repos/proj-coach-backend-README.md)).

---

## Recommended reading order

1. [architecture/system-architecture-overview.md](architecture/system-architecture-overview.md) — how the system fits together  
2. [diagrams/proj-coach-schemas-architecture.md](diagrams/proj-coach-schemas-architecture.md) — data model and service edges  
3. [docs/q2-roadmap-and-initiatives.md](docs/q2-roadmap-and-initiatives.md) — current priorities  
4. [reference/ditto-matchmaking/proposed-strategy-implementation-plan.md](reference/ditto-matchmaking/proposed-strategy-implementation-plan.md) — matchmaking direction  
5. [infra/gcp-architecture.md](infra/gcp-architecture.md) — where it runs  

---

## Maintenance

When source docs change in the workspace, re-copy into this repo and keep filenames stable (see section indexes). Do not commit API keys or `.tfvars`; example config in service READMEs should use placeholders only.

```bash
cp ../docs/*.md docs/   # then re-apply renames if needed
git add -A && git commit -m "docs: sync from workspace"
git push origin main
```

---

## Excluded

- Full `docs/REFER/ohl-platform/` clone (thousands of third-party files) — one overview only under `reference/`  
- `ditto-platform` monorepo until submitted  
- Secrets, `.env`, Terraform state, `node_modules`
