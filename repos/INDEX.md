# Service repositories

README copies from `dodo-world/projects/*` (and related infra). Each file is named `<repo>-README.md`.

> **Note:** The Nx monorepo (`ditto-platform`) is not included in this bundle until it is submitted for publication.

## Infrastructure

| Repository | README | Role |
|------------|--------|------|
| ditto-infra | [infra/ditto-infra-README.md](../infra/ditto-infra-README.md) | GCP Terraform, GKE, Infrastructure Manager CI |

## Core backend

| Repository | README | Role |
|------------|--------|------|
| proj-coach-backend | [proj-coach-backend-README.md](proj-coach-backend-README.md) | NestJS API, SMS chat, matchmaking, chatbot |
| proj-coach-schemas | [proj-coach-schemas-README.md](proj-coach-schemas-README.md) | Shared Mongoose schemas (npm package) |
| proj-coach-ws | [proj-coach-ws-README.md](proj-coach-ws-README.md) | Public WebSocket (Socket.IO) |
| proj-coach-voip | [proj-coach-voip-README.md](proj-coach-voip-README.md) | Voice / LiveKit |
| profile-analysis-service | [profile-analysis-service-README.md](profile-analysis-service-README.md) | LLM profile analysis, Restate workflows |
| prompt-manager-backend | [prompt-manager-backend-README.md](prompt-manager-backend-README.md) | Prompt versioning and CMS |
| imsg-service | [imsg-service-README.md](imsg-service-README.md) | iMessage/SMS provider plugins |
| delayed-task-service | [delayed-task-service-README.md](delayed-task-service-README.md) | Scheduled tasks and approvals |

## Internal tools & frontends

| Repository | README | Role |
|------------|--------|------|
| ditto-internal-frontend | [ditto-internal-frontend-README.md](ditto-internal-frontend-README.md) | Admin dashboard (matchmaking, chat, prompts) |
| ditto-internal-ws | [ditto-internal-ws-README.md](ditto-internal-ws-README.md) | Internal WebSocket + presence |
| marketing-internal-tool | [marketing-internal-tool-README.md](marketing-internal-tool-README.md) | Posters, schools, marketing ops |

## Data science & events

| Repository | README | Role |
|------------|--------|------|
| ufl | [ufl-README.md](ufl-README.md) | Unified feedback loop / segment analysis |
| event-202603-yik-yak | [event-202603-yik-yak-README.md](event-202603-yik-yak-README.md) | Yik Yak event matchmaking engine |
| matchmake_experimentation | [matchmake_experimentation-README.md](matchmake_experimentation-README.md) | Matchmaking R&D |
| claude-marketplace | [claude-marketplace-README.md](claude-marketplace-README.md) | Claude Code plugins |
