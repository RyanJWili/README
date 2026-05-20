# Chatbot agent routing

One-shot routing at the start of a LangGraph turn (production). Order matches `routeToAgent` in `proj-coach-backend`—**onboarding is checked before event pools**.

```mermaid
flowchart TD
  START([Incoming message]) --> LOAD[Load profile + messages]
  LOAD --> ONB{onboarding complete\nand status not N/A?}
  ONB -->|no| ONBOARD[Onboarding agent]
  ONB -->|yes| POOL{activePool yik-yak?}
  POOL -->|yes| YAK[Yak agent]
  POOL -->|no| MATCHED{status Matched\nor matchId set?}
  MATCHED -->|yes| MATCH[Match agent via loadMatchContext]
  MATCHED -->|no| NMI{NeedMoreInfo\nor InReview?}
  NMI -->|yes| PROF[Profile improvement agent]
  NMI -->|no| GEN[General agent]
  YAK --> TOOLS[Tool loop]
  ONBOARD --> TOOLS
  MATCH --> TOOLS
  PROF --> TOOLS
  GEN --> TOOLS
  TOOLS --> SEND[sendResponse]
```

## Notes

- **InReview** users go to profile improvement (same node as `NeedMoreInfo`) until analysis promotes them to `Waiting`.  
- Event pools (`nyc-gala`, `la-love-yacht`, etc.) use the **general** agent after onboarding unless `activePool === yik-yak`.  
- Routing does **not** re-run after tool calls in the same turn.  

Target **skills-based** routing: [../docs/chatbot/skills-migration.md](../docs/chatbot/skills-migration.md).  
User vs match status: [match-state-machine.md](match-state-machine.md).
