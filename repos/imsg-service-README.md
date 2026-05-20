# iMessage Service

A RabbitMQ-based microservice for handling iMessage communication through various providers using a plugin-based architecture with dependency injection.

## Quick Start

To install dependencies:

```bash
bun install
```

To run development server:

```bash
bun dev
```

To lint code:

```bash
bun run lint
```

To check TypeScript:

```bash
tsc
```

## Architecture Overview

This service follows a plugin-based architecture with dependency injection using `tsyringe` for handling iMessage communication.

### Core Components

- **Dependency Injection System**: Custom DI container using `tsyringe` with `@autoInjectable`
- **RPC Framework**: Base classes and validation decorators for RabbitMQ RPC communication
- **Provider System**: Pluggable message service implementations via `IMessageProvider` interface
- **Configuration**: Environment-based config with TOML provider settings

### Message Flow

1. RabbitMQ messages arrive at `AppService.handleMessage()`
2. Payload validated for required `method` and `params` fields
3. RPC controller method validation via `@ValidationSchema` decorator
4. Method execution through `RpcController` → `RpcService`
5. Response sent back via RabbitMQ reply queue (if `replyTo` present)

## Available RPC Methods

The service provides the following RPC methods:

- **`sendMessage`**: Send SMS/iMessage with message content and recipient
- **`lookupNumber`**: Check if a phone number supports iMessage capability
- **`sendTypingIndicator`**: Send typing notification between numbers
- **`addContact`**: Add a new contact to the system
- **`updateContactName`**: Update an existing contact's name
- **`updateStatus`**: Update message delivery status

Each method uses corresponding DTOs for validation (e.g., `SendMessageDto`, `LookupCapabilityDto`).

## Configuration

Required environment variables:
- `posthogKey`: PostHog analytics key for tracking and debugging
- `amqpUrl`: RabbitMQ connection URL for message queuing
- `baseUrl`: Service base URL for webhook callbacks
- `redis`: Redis connection settings for caching
- `providers`: Provider configuration file path (TOML format)

Provider-specific settings are configured in TOML format under `[providers.providername]`.

### Example TOML Configuration

```toml
[providers.sendblue]
api_key = "your_sendblue_api_key"
api_secret = "your_sendblue_api_secret"
```

## Project Structure

```
src/
├── common/
│   ├── config/          # Configuration management
│   └── injector/        # Dependency injection system
├── exceptions/          # Custom exception classes
├── providers/           # Infrastructure providers (RabbitMQ, Redis, PostHog)
├── services/
│   ├── dto/            # Data Transfer Objects for validation
│   ├── providers/      # Message service providers
│   ├── app.service.ts  # Main application service
│   ├── rmq.service.ts  # RabbitMQ service wrapper
│   ├── rpc.*.ts        # RPC framework components
│   └── rpc.controller.ts # Main RPC controller
└── utils/              # Utility functions
```

## Provider Development

To add a new message provider:

1. **Implement Interface**: Create a class implementing `IMessageProvider` interface
2. **Register Provider**: Use `registerIMessageProvider(YourProvider)` in `src/index.ts`
3. **Add Webhooks**: Optionally expose webhook routes via the `routes` property
4. **Configure Settings**: Add provider config in TOML under `[providers.yourprovider]`

### Required Provider Methods

- `sendMessage(data: SendMessageDto): Promise<void>`
- `sendTypingIndicator(number: string): Promise<void>`
- `lookup(number: string): Promise<boolean>`
- `updateStatus(messageId: string, provider: string): Promise<void>`

### Optional Provider Methods

- `init?(): Promise<boolean>` - Initialize provider
- `addContact?(data: AddContactDto): Promise<void>`
- `updateContactName?(number: string, name: string, userId: string): Promise<void>`

## Data Models

The service uses MongoDB with schemas from `@dodo-world/proj-coach-schemas`:

- **`DittoNumberSchema`**: Phone number management
- **`SmsChatSchema`**: Chat conversation data
- **`SmsChatMessageSchema`**: Individual message records

## Dependencies

### Runtime & Framework
- **Bun**: JavaScript runtime with ES modules and decorators
- **tsyringe**: Dependency injection container
- **class-validator/class-transformer**: Request validation and transformation

### External Services
- **RabbitMQ** (amqplib): Message queuing and RPC communication
- **MongoDB** (mongoose): Document database for data persistence
- **Redis** (ioredis): Caching and session management
- **PostHog**: Analytics and error tracking

### Message Providers
- **SendBlue SDK**: SMS/iMessage provider implementation

## Development Notes



- All RPC methods must use `@ValidationSchema(DTO)` decorator
- Services are auto-registered using various `register*()` functions in `src/index.ts`
- The service supports graceful shutdown with proper task cleanup
- Use PostHog for monitoring and debugging in production
