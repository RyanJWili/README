# ditto-internal-ws

WebSocket service for internal/admin users in the Ditto internal frontend. This service handles real-time communication via Socket.IO and RabbitMQ.

## Features

- **JWT Authentication**: Validates tokens using JWT public key (for internal/admin users)
- **Socket.IO**: Real-time WebSocket connections
- **RabbitMQ Integration**: Consumes messages from RabbitMQ fanout exchange
- **User-specific messaging**: Routes messages to specific users by userId

## Setup

### Install dependencies:

```bash
bun install
```

### Environment Variables

Create a `.env` file with the following variables:

```env
INTERNAL_AUTH_URL=https://auth.internal.ditt.ai
INTERNAL_API_SECRET=your-internal-api-secret
AMQP_URL=amqp://localhost:5672
EXCHANGE=sockets
PORT=3002
```

- `INTERNAL_AUTH_URL`: Base URL for internal auth service (to fetch JWT public key)
- `INTERNAL_API_SECRET`: Secret key for authenticating with internal auth service
- `AMQP_URL`: RabbitMQ connection URL
- `EXCHANGE`: RabbitMQ fanout exchange name for socket messages
- `PORT`: Port to listen on (default: 3002)

### Run:

```bash
# Development (with watch)
bun dev

# Production
bun start
```

## How It Works

1. **Client Connection**: Clients connect via Socket.IO with JWT token in `auth.token` or `Authorization` header
2. **Authentication**: Service validates JWT token using public key from internal auth service
3. **Message Consumption**: Listens to RabbitMQ exchange for messages with events:
   - `sendMessage`: Generic socket message to specific user
   - `chat:message:new`: New chat message notification
4. **Message Delivery**: Routes messages to connected clients by userId

## Message Format

Messages from RabbitMQ should follow this format:

```json
{
  "event": "sendMessage",
  "data": {
    "toUser": "userId",
    "event": "event-name",
    "data": { /* payload */ }
  }
}
```

## Differences from proj-coach-ws

- Uses JWT validation instead of gRPC session validation
- Designed for internal/admin users
- Runs on port 3002 by default (vs 3000)

This project was created using `bun init` in bun v1.1.42. [Bun](https://bun.sh) is a fast all-in-one JavaScript runtime.
