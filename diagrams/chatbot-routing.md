# Chatbot agent routing

One-shot routing at the start of a LangGraph turn (production). Does not re-route after tool calls within the same turn.

```mermaid
flowchart TD
  START([Incoming message]) --> LOAD[Load profile + messages]
  LOAD --> POOL{active pool yik-yak?}
  POOL -->|yes| YAK[Yak agent]
  POOL -->|no| ONB{onboarding complete?}
  ONB -->|no| ONBOARD[Onboarding agent]
  ONB -->|yes| MATCHED{status Matched?}
  MATCHED -->|yes| MATCH[Match agent]
  MATCHED -->|no| NMI{NeedMoreInfo?}
  NMI -->|yes| PROF[Profile improvement agent]
  NMI -->|no| GEN[General agent]
  YAK --> TOOLS[Tool loop]
  ONBOARD --> TOOLS
  MATCH --> TOOLS
  PROF --> TOOLS
  GEN --> TOOLS
  TOOLS --> SEND[sendResponse]
```

Target **skills-based** routing is described in [../docs/chatbot/skills-migration.md](../docs/chatbot/skills-migration.md).
