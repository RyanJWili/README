# Repositories

Personal mirrors: [github.com/RyanJWili](https://github.com/RyanJWili). Upstream org: **dodo-world**. Each repo’s own `README.md` remains the install/run source of truth—this page is a **catalog**, not a copy.

## Core platform

| Repository | Stack | Responsibility |
|------------|-------|----------------|
| [proj-coach-backend](https://github.com/RyanJWili/proj-coach-backend) | NestJS / Bun | Users, matching API, SMS chat, LangGraph chatbot, internal admin, Stripe, scheduling |
| [proj-coach-schemas](https://github.com/RyanJWili/proj-coach-schemas) | TypeScript | Shared MongoDB schemas (`@dodo-world/proj-coach-schemas`) |
| [proj-coach-protos](https://github.com/RyanJWili/proj-coach-protos) | Protobuf | Auth gRPC contracts |

## Messaging & realtime

| Repository | Stack | Responsibility |
|------------|-------|----------------|
| [imsg-service](https://github.com/RyanJWili/imsg-service) | Bun | SendBlue, Linq, PhotonHQ plugins; send/receive RPC |
| [proj-coach-ws](https://github.com/RyanJWili/proj-coach-ws) | Bun / Socket.IO | User realtime from RabbitMQ |
| [ditto-internal-ws](https://github.com/RyanJWili/ditto-internal-ws) | Bun / Socket.IO | Admin presence and internal events |

## AI & async work

| Repository | Stack | Responsibility |
|------------|-------|----------------|
| [prompt-manager-backend](https://github.com/RyanJWili/prompt-manager-backend) | NestJS / Bun | Prompt versions, publish workflow |
| [profile-analysis-service](https://github.com/RyanJWili/profile-analysis-service) | Bun / Restate | Scoring, embeddings, batch matchmaking |
| [delayed-task-service](https://github.com/RyanJWili/delayed-task-service) | Bun | Scheduled email/SMS, approval queue |
| [proj-coach-voip](https://github.com/RyanJWili/proj-coach-voip) | Bun / LiveKit | Voice sessions |

## Internal tools & events

| Repository | Stack | Responsibility |
|------------|-------|----------------|
| [ditto-internal-frontend](https://github.com/RyanJWili/ditto-internal-frontend) | React / Vite | Matchmaker UI, chat monitor, prompts |
| [marketing-internal-tool](https://github.com/RyanJWili/marketing-internal-tool) | Next.js | Posters, QR, school campaigns |
| [event-202603-yik-yak](https://github.com/RyanJWili/event-202603-yik-yak) | Python | Yik Yak batch matching (event closed) |
| [matchmake_experimentation](https://github.com/RyanJWili/matchmake_experimentation) | Bun | Offline LLM match experiments |

## Infra & observability

| Repository | Stack | Responsibility |
|------------|-------|----------------|
| [ditto-infra](https://github.com/RyanJWili/ditto-infra) | Terraform | GKE, Infisical, Cloud Build — see [../infra/overview.md](../infra/overview.md) |
| [otel](https://github.com/RyanJWili/otel) | Go | OpenTelemetry collector builds |

## Research & analysis

| Repository | Stack | Responsibility |
|------------|-------|----------------|
| [ufl](https://github.com/RyanJWili/ufl) | Python | SMS chat segmentation / taxonomy analysis |

## Tooling

| Repository | Stack | Responsibility |
|------------|-------|----------------|
| [claude-marketplace](https://github.com/RyanJWili/claude-marketplace) | Markdown / plugins | Shared Claude Code plugins and commit commands |

## External / not mirrored here

| Name | Notes |
|------|--------|
| `proj_coach_auth` | Go gRPC auth (consumed by backend / ws) |
| `proj-coach-app`, `frontpage` | User-facing web |
| `ditto-platform` | Nx monorepo migration in progress — excluded from this doc bundle |

## Security note

Public mirrors must not contain live API keys. If push protection blocks a mirror, use a redacted orphan snapshot (see conversation history). Never paste LangSmith/OpenAI secrets into hub docs.
