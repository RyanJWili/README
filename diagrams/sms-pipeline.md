# SMS / iMessage pipeline

End-to-end flow for conversational SMS (production). See [../architecture/chatbot.md](../architecture/chatbot.md) for agent detail.

```mermaid
sequenceDiagram
  participant U as User
  participant P as Provider
  participant RMQ as RabbitMQ
  participant SMS as sms-chat module
  participant CB as chatbot
  participant IMSG as imsg-service

  U->>P: iMessage / SMS
  P->>RMQ: webhook payload
  RMQ->>SMS: consume
  SMS->>SMS: guard + persist + lock
  SMS->>CB: handleAIChat
  CB->>CB: LangGraph agents + tools
  CB->>SMS: reply text
  SMS->>IMSG: RPC sendMessage
  IMSG->>P: deliver
  P->>U: message
```

## Operational risks

| Stage | Risk |
|-------|------|
| Consume | Early ACK before parse → lost message on malformed payload |
| Guard | Malicious check timeout → treated as safe |
| Graph | Full compile per message → latency |
| Send | RPC timeout, no retry → silent drop |

Mitigations are tracked in [../docs/chatbot/production-audit.md](../docs/chatbot/production-audit.md).
