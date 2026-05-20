# Architecture

## Dependency Injection Pattern

The application uses a custom tsyringe-based DI system (`src/common/container/`):

- Async factory providers for complex initialization
- Schema registration for MongoDB models via `injectModel(Schema)` decorator
- Dependency tree resolution for startup ordering in `injector.ts`
- Graceful shutdown handling with 120s timeout

## Core Services

**AppService** (`src/app.service.ts`) - Entry point

- RMQ consumer listening for messages
- RPC request validation and routing
- Task tracking for graceful shutdown

**RpcService** (`src/services/rpc.service.ts`) - Request handler

- RPC method routing with `@ValidationSchema` decorators
- Redis-based debouncing (30s for analysis, 60s for matching)
- MongoDB model operations for profiles, matching status, approvals
- Delayed task creation via RMQ

**AgentService** (`src/services/agent.service.ts`) - AI orchestration

- MCP server connection via StreamableHTTPClientTransport
- AI Gateway LLM integration with tool calling
- Async generator pattern for multi-step agent runs
- Zod schemas for response validation

**GenService** (`src/services/gen.service.ts`) - Content generation

- Profile analysis and bio assessment
- Word count and content quality validation

**CLIPService** (`src/services/clip.service.ts`) - Image/text embeddings

- Replicate API integration for CLIP embeddings
- Used for preference image matching

**FeedbackAnalysisService** (`src/services/feedback-analysis.service.ts`) - Match feedback scoring

- Orchestrates: load schedule + match, scheduler analysis (UserActivityService), chat/summary (MatchSummaryService), score combination, Feedback persistence
- Does not catch errors; caller (RpcService) handles at top level for logging and RabbitMQ retries

**UserChatService** (`src/services/user-chat.service.ts`) - User SMS chat messages

- `getMessages(userId, options?: { interval?: Interval })` – fetches user messages, optionally in a Luxon `Interval` (queried as [start, end) end exclusive). Decoupled from match/feedback; caller decides the time window.

**MatchSummaryService** (`src/services/match-summary.service.ts`) - LLM summary/flags from chat

- Run prompts for chat intent flags and match summary text (PromptService, RunPromptService)
- `getChatFeedbackFlags(userId, chatMessages)` and `generateMatchSummary(userId, chatMessages)` – caller passes messages (e.g. from UserChatService); does not catch errors (errors bubble).

**UserActivityService** (`src/services/user-activity.service.ts`) - Scheduler activity only

- `analyzeUserActivity(schedule, userId)` – visit/pick patterns from schedule updateLog
- `calculateSchedulerScore(activity)`, `calculateChatScore(flags)` – pure scoring from analysis/flags
- No match/chat models or LLM; chat flags come from MatchSummaryService

## Providers

Located in `src/providers/`:

- **RmqProvider**: RabbitMQ connection, queue consumption, message parsing
- **RedisProvider**: Connection via Bun RedisClient
- **PostHogProvider**: Analytics and error tracking
- **McpTokenProvider**: Auth tokens for MCP server
- **ReplicateProvider**: Image embedding API calls
- **MeilisearchProvider**: Meilisearch profile index for search queries

## Provider Pattern (AsyncFactoryProvider)

Providers handle async initialization of external services. They follow a consistent pattern:

### Structure

1. **Provider file** (`src/providers/[name].provider.ts`) - Creates the raw client
2. **Service file** (`src/services/[name].service.ts`) - Wraps client with app-specific methods

### Creating a New Provider

**Step 1: Create the Provider**

```typescript
// src/providers/example.provider.ts
import { ExampleClient } from "example-sdk";
import type { AsyncFactoryProvider } from "../common/injector/async-factory-provider";
import { Config } from "../common/config/config";

export const ExampleProvider: AsyncFactoryProvider = {
  provide: ExampleClient, // Use SDK class as token, or Symbol for interfaces
  inject: [Config], // Dependencies to inject
  useFactory: async (conf: Config) => {
    const client = new ExampleClient({
      host: conf.exampleHost,
      apiKey: conf.exampleApiKey,
    });
    await client.connect(); // Optional async initialization
    return client;
  },
};
```

**Step 2: Register the Provider**

```typescript
// src/index.ts
import { registerAsyncProvider } from "./common/injector";
import { ExampleProvider } from "./providers/example.provider";

registerAsyncProvider(ExampleProvider);
```

**Step 3: Create a Service (optional, for app-specific methods)**

```typescript
// src/services/example.service.ts
import { autoInjectable } from "tsyringe";
import { ExampleClient } from "example-sdk";

@autoInjectable()
export class ExampleService {
  constructor(private readonly client?: ExampleClient) {}

  async doSomething(): Promise<Result> {
    return this.client!.operation();
  }
}
```

### When to Use Symbols vs Classes

- **Use SDK class** (`provide: ExampleClient`) when the SDK exports a class
- **Use Symbol** (`provide: Symbol("ExampleIndex")`) when providing an interface or internal type

### Disposal Pattern

For clients that need cleanup on shutdown:

```typescript
class DisposableClient extends BaseClient implements Disposable {
  async dispose(): Promise<void> {
    await this.shutdown();
  }
}
```

## Message Flow

1. Messages consumed from RMQ queue
2. JSON parsed and RPC method validated against DTO
3. Method executed; ACK sent immediately
4. Errors captured to PostHog
5. Delayed tasks sent to `delayed-tasks` queue

## Deployment Modes

The `MODE` environment variable controls which components are loaded at startup:

| Mode | RMQ Consumer | Restate Server | Use Case |
|------|-------------|----------------|----------|
| `all` | Yes | Yes | Local development, integration tests |
| `rmq` | Yes | No | Legacy RabbitMQ-only deployment |
| `restate` | No | Yes | Cloud Run deployment |

- **AppService** (RMQ consumer) is only active when MODE is `rmq` or `all`
- Default is `all` when MODE is not set
- Cloud Run deploys with `MODE=restate` (no RMQ dependency needed)

## Infrastructure

- **Skaffold** (`skaffold.yaml`) -- Builds the Docker image and deploys to Cloud Run
- **Cloud Run** -- Production runtime on GCP (project `petkeley`, region `us-central1`)
- **Artifact Registry** -- Docker images stored in `proj-coach-registry` repo
- **Dual deployment** -- During migration, images are pushed to both Docker Hub (existing CI) and Artifact Registry (Cloud Run). This is temporary until Cloud Run is fully validated.

See [Cloud Run Setup](CLOUD_RUN_SETUP.md) for GCP infrastructure configuration.

## External Dependencies

- **Schemas**: `@dodo-world/proj-coach-schemas` provides MongoDB schemas (UserProfile, UserImage, MatchingStatus, etc.)
- **Prompt Service**: External HTTP service for template rendering (configured via PROMPT_SVC)
- **MCP Server**: Model Context Protocol server for agent tools
