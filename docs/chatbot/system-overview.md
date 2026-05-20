# Chatbot — system overview

Ditto’s conversational AI runs inside **proj-coach-backend**, orchestrating SMS/iMessage turns via LangGraph (production) with prompts from **prompt-manager-backend**.

## Responsibilities

- Onboard new users (profile fields, email verification, photos)  
- Answer questions and maintain tone (Gen-Z SMS persona)  
- Support active matches (scheduling, contact exchange, FAQs)  
- Handle event pools (Yik Yak referral, closed events redirecting to Wednesday pool)  
- Escalate to human ops when tools cannot safely complete a request  

## Architecture layers

| Layer | Module | Role |
|-------|--------|------|
| Ingress | `sms-chat` | Webhooks, locks, drain loop, daily limits |
| Orchestration | `chatbot` | LangGraph graph, checkpointing |
| Tools | `llm-tools` | Side effects (DB, email, match actions) |
| Content | `prompt-manager` RPC | Versioned system prompts |
| Moderation | Qwen3Guard | Inbound / first-iteration screening |

## Agents (production)

Onboarding, General, Match, Profile improvement, Yak—routed once per turn. See [../../diagrams/chatbot-routing.md](../../diagrams/chatbot-routing.md).

## Data

- **MongoDB** — `sms_chats`, `sms_chat_messages`, user profile collections  
- **LangGraph checkpoint** — `langgraph` DB collections for thread state  
- **Redis** — per-chat processing lock  

## Observability

LangSmith for traces; PostHog for product events. Configure via environment variables only.

## Related

- [production-audit.md](production-audit.md) — risks and refactor plan  
- [skills-migration.md](skills-migration.md) — target architecture  
- [onboarding.md](onboarding.md) — conversational onboarding spec  
- [../../architecture/chatbot.md](../../architecture/chatbot.md) — ops-oriented summary  

*Synthesized from internal chatbot overview and pipeline audit sources.*
