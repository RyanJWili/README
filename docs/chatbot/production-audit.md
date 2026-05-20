# Chatbot — production audit

Summary of the full pipeline review. Use this as a remediation backlog, not as runtime documentation.

## Critical issues

| ID | Issue | Impact |
|----|--------|--------|
| C1 | RabbitMQ ACK before JSON parse | Message loss on malformed payloads |
| C2 | Malicious guard fails open on timeout | Unsafe content may proceed |
| C3 | `buildAgentGraph()` + compile every message | Latency and CPU waste |
| C4 | School timezone fetched every message, uncached | Extra RPC/load |
| C5 | Agent route fixed for entire tool loop | Wrong agent if state changes mid-turn |
| C6 | iMessage send RPC: 30s timeout, no retry | Dropped replies |

## Design debt

- Denormalized chat fields updated non-atomically with messages  
- Duplicate malicious checks (ingress + agent iteration 0)  
- Quiet-check suppression can hide stale replies—by design but confusing in incidents  
- System-triggered invocations skip user “hiccup” recovery path  

## Refactor themes (aligned with Q2)

1. **Ingress hardening** — parse-then-ack, structured dead-letter  
2. **Graph lifecycle** — compile once / pool compiled graphs per agent set  
3. **Skills migration** — shrink graph, explicit routers ([skills-migration.md](skills-migration.md))  
4. **Caching** — timezone and school metadata  
5. **Outbound reliability** — retry policy for imsg RPC  

## Related diagrams

- [../../diagrams/sms-pipeline.md](../../diagrams/sms-pipeline.md) — ingress path where C1–C6 occur  
- [../../diagrams/chatbot-routing.md](../../diagrams/chatbot-routing.md) — agent selection (C5)  

## Testing recommendations

- Integration tests for multi-burst SMS ordering (drain loop)  
- Failure injection on guard timeout and imsg RPC  
- Regression suite before disabling LangGraph paths  

*Synthesized from “Chatbot Pipeline — Production Issues & Refactoring Plan.”*
