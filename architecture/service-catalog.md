# Service catalog

One-line ownership map. Deep dives: [services/README.md](services/README.md). Overview: [platform-overview.md](platform-overview.md). GitHub links: [../repos/README.md](../repos/README.md).

## Core

| Service | Runtime | Owns |
|---------|---------|------|
| **proj-coach-backend** | NestJS / Bun | Users, auth integration, matching API, SMS chat, LangGraph chatbot, internal admin APIs, Stripe, scheduling |
| **proj-coach-schemas** | TypeScript lib | MongoDB schema definitions shared via npm |
| **proj-coach-protos** | Protobuf | Auth gRPC service contracts |

## Messaging & realtime

| Service | Runtime | Owns |
|---------|---------|------|
| **imsg-service** | Bun | Provider plugins (SendBlue, Linq, PhotonHQ), send/receive RPC |
| **proj-coach-ws** | Bun / Socket.IO | User-facing realtime fanout from RabbitMQ |
| **ditto-internal-ws** | Bun / Socket.IO | Admin presence and internal events |

## AI & background work

| Service | Runtime | Owns |
|---------|---------|------|
| **prompt-manager-backend** | NestJS / Bun | Prompt versions, publish workflow, RPC to consumers |
| **profile-analysis-service** | Bun / Restate | Profile scoring, embeddings, batch matchmaking helpers |
| **delayed-task-service** | Bun | Scheduled email/SMS, approval queue |
| **proj-coach-voip** | Bun / LiveKit | Voice sessions |

## Internal & growth

| Service | Runtime | Owns |
|---------|---------|------|
| **ditto-internal-frontend** | React / Vite | Matchmaker UI, chat monitor, prompts, analytics pages |
| **marketing-internal-tool** | Next.js | Posters, QR, school campaign data |

## Infra & tooling

| Service | Runtime | Owns |
|---------|---------|------|
| **ditto-infra** | Terraform | GKE, networking, Infisical injector, Cloud Build |
| **otel** | Go / OCB | Telemetry collector builds |
| **ufl** | Python | Segment/taxonomy analysis on SMS chats |
| **event-202603-yik-yak** | Python | Batch matching for Yik Yak event |
| **matchmake_experimentation** | Bun | Offline LLM match experiments |

## External / not in bundle

| Service | Notes |
|---------|--------|
| **proj_coach_auth** | Go gRPC auth (referenced by backend/ws) |
| **proj-coach-app**, **frontpage** | User-facing web (separate repos) |
