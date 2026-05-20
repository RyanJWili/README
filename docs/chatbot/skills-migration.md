# Chatbot — skills migration (TDD summary)

**Initiative:** ENG-1518 — move from monolithic LangGraph to **Vercel AI SDK skills**.

## Goals

- Faster turns (no per-message graph compile)  
- Clearer ownership of prompts per skill  
- Easier A/B via prompt-manager versions  
- Better alignment with OpenTelemetry / Vercel AI Gateway  

## Target components

| Piece | Responsibility |
|-------|----------------|
| Router | Intent from state + last messages |
| Skill modules | Prompt + tool allowlist + stop conditions |
| Shared context | Profile, school, pool, match snapshot |
| Tool executor | Reuse `llm-tools` with skill-scoped guards |

## Non-goals (for first cut)

- Rewriting imsg ingress  
- Changing Mongo message schema  
- Removing human handover path  

## Migration strategy

1. Parity tests: golden conversations for onboarding + general  
2. Shadow traffic or internal dogfood number  
3. Cut over pool-by-pool (`wednesday` last)  
4. Remove LangGraph compile path when error rates stable  

## Observability

Plan for Vercel OTEL + existing PostHog events; LangSmith may remain during transition.

See [../../architecture/chatbot-evolution.md](../../architecture/chatbot-evolution.md).

*Synthesized from TDD Chatbot Architecture Pivot — Skills + Vercel AI SDK source.*
