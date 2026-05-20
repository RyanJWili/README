# Chatbot evolution: LangGraph → skills

**Status:** Migration in progress (ENG-1518 direction). Production still runs LangGraph; new docs describe the target architecture.

## Why change

| LangGraph (today) | Skills (target) |
|-------------------|-----------------|
| Single compiled graph per message | Modular skills loaded by intent |
| Heavy compile/runtime overhead | Lighter Vercel AI SDK paths |
| Tool routing embedded in graph nodes | Explicit skill registry + handoffs |
| Harder to A/B prompt slices | Aligns with prompt-manager versioning |

## Target shape

- **Router** classifies intent (onboarding, match, general, yak, etc.)  
- **Skills** bundle prompts, allowed tools, and completion rules  
- **Vercel AI Gateway** for model routing; OpenTelemetry via Vercel OTEL patterns  
- State reminders and profile context injected per skill, not one global graph  

## What stays the same

- SMS/iMessage ingress via `sms-chat`  
- MongoDB for messages and chat state  
- Prompt content owned by `prompt-manager-backend`  
- Human handover and daily limits  

## Reading order

1. [chatbot.md](chatbot.md) — production behavior  
2. [../docs/chatbot/skills-migration.md](../docs/chatbot/skills-migration.md) — TDD / design notes  
3. [../docs/chatbot/production-audit.md](../docs/chatbot/production-audit.md) — issues to fix during migration  
