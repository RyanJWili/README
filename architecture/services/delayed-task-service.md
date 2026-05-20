# delayed-task-service

**Runtime:** Bun  
**Role:** Time-based and approval-gated outbound work—email, SMS, and internal queue jobs that must not block HTTP/RPC in `proj-coach-backend`.

## Responsibilities

- Schedule sends at a future time (reminders, follow-ups, campaign nudges)  
- Hold tasks in an approval queue when ops must sign off  
- Invoke `imsg-service` or email providers via RabbitMQ when fired  

## Pattern

```
proj-coach-backend  →  enqueue job (RabbitMQ)
delayed-task-service  →  wait until due  →  execute  →  imsg / email
```

## Boundaries

Does not own conversation state or chatbot logic—only executes pre-defined tasks with payloads from the enqueuer.

## Related

- [imsg-service.md](imsg-service.md)  
- [../../diagrams/platform-topology.md](../../diagrams/platform-topology.md)  
