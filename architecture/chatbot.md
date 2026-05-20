# Chatbot system

The Ditto chatbot is the primary AI surface for users on SMS/iMessage. It onboard profiles, answers questions, and supports match-related flows. Implementation lives in `proj-coach-backend` (`chatbot`, `sms-chat`, `llm-tools` modules).

## Inbound path

1. Provider webhook → RabbitMQ → `sms-chat` consumer  
2. Optional **Qwen3Guard** malicious-content check (fails open on timeout—documented risk)  
3. Message persisted; Redis lock acquired; **drain loop** processes queued user text (handles rapid multi-text)  
4. Routing by chat state: login, Yik Yak referral, or onboarded AI chat  

## Agent routing (LangGraph, production)

After profile and messages load, **one-shot** routing selects an agent for the turn:

| Condition | Agent |
|-----------|--------|
| Pool includes `yik-yak` | Yak agent |
| Onboarding incomplete | Onboarding agent |
| User status `Matched` / active match | Match agent (+ match context) |
| Status `NeedMoreInfo` | Profile improvement agent |
| Default | General agent |

Each agent loop: model call → tools (up to 10 iterations) → `sendResponse`. Outbound text is split and sent via `imsg-service`; responses suppressed if the user sent newer messages during the run.

## Tools and side effects

40+ tools cover profile updates, email verification, match actions, pool switches, handoff to human team, Yik Yak conversion, etc. Tool implementations are in `llm-tools.service.ts`; prompts are loaded from **prompt-manager-backend**.

## Limits and safety

- **Daily message limit** per user (timezone from school), typically 100/day  
- `pauseAIReply` / account status gates  
- Malicious check duplicated at ingress and iteration 0 (known redundancy)  
- On hard failure: user-facing “hiccup” SMS + internal handover  

## Observability

LangSmith traces graph runs (configure with env vars—never commit API keys). PostHog for product analytics.

## Known production weaknesses

Summarized from the pipeline audit—detail in [../docs/chatbot/production-audit.md](../docs/chatbot/production-audit.md):

- RabbitMQ ACK before parse (loss risk on bad payloads)  
- `buildAgentGraph()` + compile on **every** message (latency cost)  
- Timezone lookup per message without cache  
- Agent route not re-evaluated mid-tool-loop  
- iMessage RPC 30s timeout without retry  

## Evolution

See [chatbot-evolution.md](chatbot-evolution.md) for the planned move to **skills + Vercel AI SDK** and deprecation of the monolithic LangGraph graph.
