# proj-coach-backend

**Runtime:** NestJS on Bun  
**Role:** System of record orchestrator for users, matching, SMS chat, chatbot, internal admin APIs, billing, and scheduling.

## Boundaries

| In scope | Out of scope |
|----------|----------------|
| HTTP/REST and internal routes | Raw iMessage provider I/O (`imsg-service`) |
| LangGraph chatbot (production) | Prompt file CMS (`prompt-manager-backend`) |
| Match lifecycle and coach workflows | Heavy scoring batches (`profile-analysis-service`) |
| MongoDB reads/writes for core entities | gRPC auth implementation (`proj_coach_auth`) |

## Major modules (conceptual)

- **Users & profiles** — signup, pools, schools, profile images (GCS)  
- **SMS chat** — conversation state, quiet checks, outbound via RabbitMQ → imsg  
- **Chatbot** — LangGraph graph, tools (`llm-tools`), handover to human coaches  
- **Matching** — proposals, scheduling, integration with analysis service / Restate  
- **Internal** — endpoints consumed by `ditto-internal-frontend`  

## Data dependencies

MongoDB (primary), Redis (locks/cache), RabbitMQ (RPC/events), MeiliSearch (search), optional GCS for media.

## Observability

LLM tracing via LangSmith in production config—**API keys belong in Infisical**, not in committed `config.toml` examples.

## Related

- [../chatbot.md](../chatbot.md)  
- [../../diagrams/sms-pipeline.md](../../diagrams/sms-pipeline.md)  
- [../../diagrams/match-state-machine.md](../../diagrams/match-state-machine.md)  
