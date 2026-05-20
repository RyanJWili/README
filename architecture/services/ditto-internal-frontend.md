# ditto-internal-frontend

**Runtime:** React + Vite  
**Role:** Internal admin dashboard for matchmaking, chat monitoring, prompts, and analytics.

## Primary surfaces

| Area | Purpose |
|------|---------|
| Matchmaker | Review proposals, approve/reject pairs, priority queues |
| iMessages monitor | Live SMS threads, coach takeover |
| AI chat tools | Test prompts, inspect bot behavior |
| Prompts | UI over `prompt-manager-backend` |
| Analytics | Funnels, school/pool views (where enabled) |

## Realtime

Subscribes via `ditto-internal-ws` (Socket.IO) for presence and live events; REST to `proj-coach-backend` for mutations.

## Auth

Uses same identity stack as other internal tools (org SSO / token flow as configured in deployment—not documented here to avoid env-specific drift).

## Related

- [../platform-overview.md](../platform-overview.md)  
- [../operations/matching-priority.md](../operations/matching-priority.md)  
