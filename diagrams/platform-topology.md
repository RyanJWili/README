# Platform topology

High-level Ditto runtime topology. User-facing web apps exist but are omitted here where source repos are outside this bundle.

```mermaid
flowchart TB
  subgraph users [Users]
    SMS[SMS / iMessage]
    WEB[Web app]
  end

  subgraph edge [Messaging edge]
    IMSG[imsg-service]
  end

  subgraph core [Core]
    BE[proj-coach-backend]
  end

  subgraph workers [Async workers]
    PA[profile-analysis-service]
    DT[delayed-task-service]
    PM[prompt-manager-backend]
  end

  subgraph realtime [Realtime]
    WS[proj-coach-ws]
    IWS[ditto-internal-ws]
  end

  subgraph data [Data plane]
    MONGO[(MongoDB)]
    REDIS[(Redis)]
    RMQ{{RabbitMQ}}
    MEILI[(MeiliSearch)]
  end

  subgraph admin [Operations]
    INT[ditto-internal-frontend]
  end

  SMS --> IMSG
  IMSG --> RMQ
  RMQ --> BE
  WEB --> BE
  BE --> MONGO
  BE --> REDIS
  BE --> MEILI
  BE --> RMQ
  RMQ --> PA
  RMQ --> DT
  RMQ --> IMSG
  BE --> PM
  PA --> PM
  BE --> WS
  BE --> IWS
  INT --> BE
  INT --> PM
  INT --> IWS
```

## Communication patterns

| Pattern | Used for |
|---------|----------|
| HTTPS REST | Web clients, internal dashboard → backends |
| RabbitMQ RPC | Backend → imsg, profile-analysis, delayed-task |
| RabbitMQ fanout | Backend → WebSocket services |
| gRPC | Backend / WS → auth service |
| Restate | Durable workflows in profile-analysis-service |

See also [data-model.md](data-model.md) and [sms-pipeline.md](sms-pipeline.md).
