# imsg-service

**Runtime:** Bun  
**Role:** Single ingress/egress for SMS and iMessage through pluggable providers.

## Architecture

Plugin-based design with `tsyringe` dependency injection:

```
RabbitMQ RPC  →  AppService  →  RpcController  →  provider plugin
                                              ↓
                                    SendBlue · Linq · PhotonHQ · …
```

Providers implement `IMessageProvider`. Configuration is environment + TOML per provider—no second webhook stack in backend unless provider architecture changes.

## RPC surface (representative)

| Method | Purpose |
|--------|---------|
| `sendMessage` | Outbound SMS/iMessage |
| `lookupNumber` | iMessage capability check |
| `sendTypingIndicator` | Typing bubble |
| `addContact` / `updateContactName` | Contact book |
| `updateStatus` | Delivery receipts |

Replies use RabbitMQ `replyTo` when the caller expects a synchronous RPC result.

## Callers

`proj-coach-backend` (SMS chat, chatbot replies), `delayed-task-service` (scheduled sends), internal tools for line tests.

## Operations

Line health and spam risk: [../operations/linq-spam-guard.md](../operations/linq-spam-guard.md). Latency: [../../docs/chatbot/outbound-latency.md](../../docs/chatbot/outbound-latency.md).

## Related

- [../../diagrams/sms-pipeline.md](../../diagrams/sms-pipeline.md)  
