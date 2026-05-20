# Outbound message latency — RCA summary

Investigation into delayed or out-of-order SMS delivery to users.

## Typical path

`chatbot` / `sms-chat` → `smsService` → RabbitMQ RPC → `imsg-service` → provider → carrier → device.

## Common contributors

| Factor | Effect |
|--------|--------|
| Redis lock + drain loop | Batches rapid texts; can delay last segment |
| Graph compile + long tool loops | Seconds before reply starts |
| iMessage RPC queue depth | Provider-side backlog |
| Provider rate limits / line health | Throttling or flagging |
| Quiet-check suppression | Intentional delay when user still typing |

## Mitigations (recommended)

- Cache school timezone; reduce per-message RPC  
- Compiled graph reuse  
- Metrics on lock wait time and imsg RPC latency percentiles  
- Alert on p95 reply latency > SLA  

## What this doc is not

Does not replace provider status pages or line-rotation runbooks—pair with [../../architecture/operations/linq-spam-guard.md](../../architecture/operations/linq-spam-guard.md).

*Synthesized from Outbound Message Latency — Full Root Cause Analysis source.*
