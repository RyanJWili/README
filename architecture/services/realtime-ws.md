# Realtime WebSocket services

Two Socket.IO services fan out events from RabbitMQ to browsers—one for end users, one for internal admins.

## proj-coach-ws

| | |
|---|---|
| **Audience** | User-facing web (`proj-coach-app`) |
| **Ingress** | RabbitMQ fanout from backend |
| **Typical events** | Match updates, notifications tied to user session |

## ditto-internal-ws

| | |
|---|---|
| **Audience** | `ditto-internal-frontend` |
| **Ingress** | RabbitMQ fanout for admin channels |
| **Typical events** | Presence, chat monitor refreshes, internal tooling |

## Shared pattern

Neither service is the source of truth—MongoDB and backend mutations are. WS only reflects committed state for low-latency UI refresh.

## Auth

Both call **proj_coach_auth** (gRPC) for token validation patterns aligned with their client apps.

## Related

- [ditto-internal-frontend.md](ditto-internal-frontend.md)  
- [proj-coach-backend.md](proj-coach-backend.md)  
