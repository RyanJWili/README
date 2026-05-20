# Platform overview

Ditto is an AI campus matchmaking product. Students sign up, build a profile through SMS/iMessage conversation, and receive curated matches—typically on a weekly cadence—with dates planned on campus. Operations run through an internal admin dashboard; infrastructure is on Google Cloud (GKE) with MongoDB as the system of record.

## Product channels

| Channel | Role |
|---------|------|
| **SMS / iMessage** | Primary user experience: onboarding, chatbot, match notifications |
| **Web app** (`app.ditt.ai`) | Profile and flows that complement messaging |
| **Internal dashboard** | Matchmaking, chat monitoring, prompts, analytics |
| **Voice** (LiveKit) | Optional AI coaching via `proj-coach-voip` |

## Logical architecture

```
                    ┌─────────────────────────────────────┐
                    │           User channels              │
                    │  SMS/iMessage · Web · Voice          │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │         imsg-service                 │
                    │  (SendBlue, Linq, PhotonHQ, …)       │
                    └──────────────┬──────────────────────┘
                                   │ RabbitMQ / HTTP
┌──────────────────────────────────▼──────────────────────────────────┐
│                    proj-coach-backend (NestJS)                         │
│  Users · Matching · SMS chat · Chatbot · Internal APIs · Scheduling   │
└─┬────────────┬─────────────┬──────────────┬──────────────┬───────────┘
  │            │             │              │              │
  ▼            ▼             ▼              ▼              ▼
MongoDB     Redis        RabbitMQ      MeiliSearch    GCS / S3
              │             │
              │    ┌────────┴────────┬─────────────────┐
              │    ▼                 ▼                 ▼
              │ profile-analysis  delayed-task    prompt-manager
              │    -service          -service         -backend
              │
              ▼
         gRPC auth · public WS · internal WS · link shortener
```

## Repository layout

Production code lives in separate repositories under the `dodo-world` GitHub organization (mirrored personally as `RyanJWili/*`). The **Nx monorepo (`ditto-platform`)** is in migration and is **not** part of this documentation bundle yet.

| Repository | Purpose |
|------------|---------|
| `proj-coach-backend` | Core API, SMS chat, chatbot, matchmaking orchestration |
| `proj-coach-schemas` | Shared Mongoose schemas (`@dodo-world/proj-coach-schemas`) |
| `proj-coach-protos` | gRPC/protobuf (auth service contracts) |
| `proj-coach-ws` / `ditto-internal-ws` | Real-time Socket.IO (users / admins) |
| `proj-coach-voip` | LiveKit voice agent |
| `imsg-service` | Outbound/inbound iMessage and SMS |
| `profile-analysis-service` | LLM analysis, embeddings, Restate workflows |
| `prompt-manager-backend` | Prompt versioning and CMS |
| `delayed-task-service` | Scheduled email/SMS/queue jobs |
| `ditto-internal-frontend` | Admin UI |
| `marketing-internal-tool` | Posters, school campaigns |
| `ditto-infra` | GCP Terraform, GKE, CI |
| `ufl` | Chat segmentation / feedback analysis (Python) |
| `event-202603-yik-yak` | One-off Yik Yak event matching |
| `matchmake_experimentation` | Matchmaking R&D |
| `otel` | OpenTelemetry collector (Go) |

## Data and messaging

- **MongoDB** — users, profiles, matches, SMS chats, schools, prompts, events
- **Redis** — locks, rate limits, caching
- **RabbitMQ** — RPC to workers (`profile-analysis`, `delayed-task`, `imsg`), fanout to WebSockets
- **MeiliSearch** — user search for internal tools

## Chatbot (current vs target)

**Today (production):** LangGraph state machine in `proj-coach-backend`, multiple agents (onboarding, general, match, profile improvement, Yik Yak), MongoDB checkpointer, prompts from `prompt-manager-backend`.

**Target (in flight):** Skills-based architecture on Vercel AI SDK (see [chatbot-evolution.md](chatbot-evolution.md) and [../docs/chatbot/skills-migration.md](../docs/chatbot/skills-migration.md)).

## Matchmaking

Weekly **Wednesday** pool is the default product loop. Internal tools run stable matching, AI-assisted scoring (`profile-analysis-service`), and human approval workflows. Matchmaking 3.x strategy is documented under [../reference/matchmaking/](../reference/matchmaking/).

## Schools and campaigns

Users map to a **school** via email domain (`schools` collection, `febLaunchStatus` for launched campuses). Event pools (e.g. Yik Yak, love-yacht, NYC gala) layer on top of the default `wednesday` pool—see [../docs/chatbot/yak-event.md](../docs/chatbot/yak-event.md).

## Related documents

- [Service catalog](service-catalog.md) — responsibilities per repo
- [Service deep dives](services/README.md) — backend, imsg, analysis, prompts, admin UI
- [Chatbot](chatbot.md) — runtime behavior
- [../diagrams/platform-topology.md](../diagrams/platform-topology.md) — Mermaid topology
- [../infra/overview.md](../infra/overview.md) — GCP deployment
