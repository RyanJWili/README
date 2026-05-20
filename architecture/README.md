# Architecture

How Ditto is structured as a product and as a set of services.

## Documents

| Document | Description |
|----------|-------------|
| [platform-overview.md](platform-overview.md) | End-to-end platform: channels, backend, data stores |
| [service-catalog.md](service-catalog.md) | One-line ownership per repository |
| [chatbot.md](chatbot.md) | Chatbot placement in the backend |
| [chatbot-evolution.md](chatbot-evolution.md) | LangGraph → skills / Vercel AI SDK direction |

## Services

| Document | Description |
|----------|-------------|
| [services/README.md](services/README.md) | Index of service deep dives |
| [services/proj-coach-backend.md](services/proj-coach-backend.md) | Core API and chatbot |
| [services/imsg-service.md](services/imsg-service.md) | SMS/iMessage providers |
| [services/profile-analysis-service.md](services/profile-analysis-service.md) | Scoring and Restate workflows |
| [services/prompt-manager-backend.md](services/prompt-manager-backend.md) | Prompt CMS |
| [services/ditto-internal-frontend.md](services/ditto-internal-frontend.md) | Admin dashboard |
| [services/proj-coach-schemas.md](services/proj-coach-schemas.md) | Shared MongoDB schemas |
| [services/delayed-task-service.md](services/delayed-task-service.md) | Scheduled / approval-queue jobs |
| [services/realtime-ws.md](services/realtime-ws.md) | User and admin Socket.IO |

## Operations

| Document | Description |
|----------|-------------|
| [operations/matching-priority.md](operations/matching-priority.md) | ENG-1030 tiered matching priority |
| [operations/linq-spam-guard.md](operations/linq-spam-guard.md) | iMessage line health runbook |

## Diagrams

Visual flows live under [../diagrams/](../diagrams/).
