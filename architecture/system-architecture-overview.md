# Ditto — System Architecture Overview

This document describes the overall architecture of the Ditto platform, all services and tools, how they relate to each other, and how to run the system locally.

---

## What Is Ditto?

Ditto is an AI-powered dating/matching application. Users communicate primarily via **SMS/iMessage** (not a traditional chat UI). An AI chatbot handles all user interactions — onboarding, profile help, match updates, and general conversation — using a LangGraph state machine backed by LLMs routed through a Vercel AI Gateway. The platform includes sophisticated matchmaking algorithms, multi-channel communications, voice calling, and a comprehensive admin operations dashboard.

---

## Repository Map

All repositories live under the `dodo-world` GitHub org. Local clones are in `/home/ryan/ditto/projects/`.

### Core Services

| Repo | Local Path | Stack | Role |
|------|-----------|-------|------|
| `proj-coach-backend` | `projects/proj-coach-backend` | TypeScript / NestJS / Bun | **Main backend** — 48 modules, REST API, chatbot, matchmaking, all business logic |
| `proj-coach-schemas` | `projects/proj-coach-schemas` | TypeScript / Mongoose | Shared MongoDB schemas/types as npm package (`@dodo-world/proj-coach-schemas`, 80+ schemas) |
| `proj-coach-ws` | `projects/proj-coach-ws` | TypeScript / Socket.IO / Bun | WebSocket service for real-time events to end users (port 3000) |
| `proj-coach-voip` | `projects/proj-coach-voip` | TypeScript / LiveKit / Bun | Voice agent using OpenAI Realtime API for AI coaching calls |
| `proj-coach-protos` | `projects/proj-coach-protos` | Protobuf v3 | Shared `.proto` definitions (auth gRPC: ValidateSession, CreateSession, DeleteSession) |
| `imsg-service` | `projects/imsg-service` | TypeScript / tsyringe / Bun | iMessage/SMS delivery via pluggable providers (SendBlue, Linq, PhotonHQ, Virtual) |
| `prompt-manager-backend` | `projects/prompt-manager-backend` | TypeScript / NestJS / Bun | Prompt CMS — immutable versioning, per-prompt publish workflow, multi-LLM support |
| `profile-analysis-service` | `projects/profile-analysis-service` | TypeScript / tsyringe / Bun | AI profile analysis, CLIP embeddings, hobby/vibe/attractiveness matching, Restate workflows |
| `delayed-task-service` | `projects/delayed-task-service` | TypeScript / tsyringe / Bun | Scheduled task execution (emails, SMS, queue messages) with approval workflows |
| `ditto-internal-ws` | `projects/ditto-internal-ws` | TypeScript / Socket.IO / Bun | WebSocket for internal admin frontend with presence tracking (port 3002) |

### Frontends & Dashboards

| Repo | Local Path | Stack | Role |
|------|-----------|-------|------|
| `ditto-internal-frontend` | `projects/ditto-internal-frontend` | React 18 / Vite / Tailwind | **Central admin dashboard** — 25+ pages for matchmaking ops, chat monitoring, prompt editing, analytics |
| `marketing-internal-tool` | `projects/marketing-internal-tool` | Next.js 16 / React 19 / Supabase | Marketing poster generator with QR codes, school batch operations, analytics |
| `proj-coach-app` | *(no access)* | Unknown | **Main user-facing frontend** (`app.ditt.ai`) |
| `proj-coach-frontpage` | *(no access)* | Unknown | Marketing home page (`ditt.ai`) |
| `ditto-user-dashboard` | *(no access)* | Unknown | User dashboard |

### Infrastructure & Tooling

| Repo | Local Path | Stack | Role |
|------|-----------|-------|------|
| `otel` | `projects/otel` | Go / OCB | Custom OpenTelemetry Collector for Cloud Run sidecars and k3s clusters |
| `claude-marketplace` | `projects/claude-marketplace` | Node.js / Claude plugins | Shared Claude Code skills (PR comments, gitmoji commits, code search) |
| `ufl` | `projects/ufl` | Python 3.12 / MongoDB / Ollama | Synthetic data generation: user profiles, matching records, chat analysis pipelines |
| `matchmake_experimentation` | `projects/matchmake_experimentation` | TypeScript / Bun / OpenRouter | R&D: LLM match prediction, 3-agent photo analysis (attractiveness, vibe, lifestyle) |
| `event-202603-yik-yak` | `projects/event-202603-yik-yak` | Python / Pandas / NumPy | One-off matchmaking engine for ~68k Yik Yak event users |

### Not Cloned (No Access)

| Repo | Stack | Role |
|------|-------|------|
| `proj_coach_auth` | Go / gRPC | Auth service — session create/validate/delete on port 50051 |
| `ditto-internal-frontend` (separate from local) | React | Internal admin frontend (separate deployment) |

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         User Channels                                │
│                                                                      │
│  SMS/iMessage ──► imsg-service ──► proj-coach-backend               │
│  Web App (app.ditt.ai) ──────────► proj-coach-backend (HTTP/REST)   │
│  Voice ──► proj-coach-voip (LiveKit + OpenAI Realtime)              │
│  Admin (internal.ditt.ai) ──────► ditto-internal-frontend           │
└────────────────────────┬──────────────────────────────┬──────────────┘
                         │                              │
            RabbitMQ / HTTP / gRPC              Socket.IO WebSocket
                         │                              │
┌────────────────────────▼──────────────────────────────▼──────────────┐
│                    proj-coach-backend (NestJS)                        │
│                         48 Modules                                    │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │  User/Auth   │  │   Matching   │  │  Chatbot (LangGraph)       │ │
│  │  Google OAuth │  │  Stable Match│  │  5 Agents + 40+ Tools      │ │
│  │  gRPC Auth   │  │  AI Match    │  │  Qwen3Guard Moderation     │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘ │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │  SMS/iMsg    │  │  Email       │  │  Internal Admin (30+ APIs) │ │
│  │  Twilio      │  │  SendGrid   │  │  Match approval, search    │ │
│  │  Telegram    │  │  Postmark   │  │  AI matches, autofilter    │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘ │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │  Scheduler   │  │  Payments   │  │  Events / Waitlist          │ │
│  │  Availability│  │  Stripe     │  │  Tickets, Leaderboard       │ │
│  │  Scheduling  │  │  Webhooks   │  │  Priority Users             │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘ │
└──────┬──────────┬──────────┬──────────┬──────────┬──────────────────┘
       │          │          │          │          │
       ▼          ▼          ▼          ▼          ▼
   MongoDB      Redis    RabbitMQ   ClickHouse  GCS/S3
   (rs0)       (cache)   (queues)  (analytics) (files)
```

---

## Detailed Service Descriptions

### 1. proj-coach-backend (Main Backend)

The heart of the platform. NestJS monolith with 48 modules.

**Key Modules:**
- **User** — Registration, login (email/SMS OTP), Google OAuth, profile CRUD, photo upload (Sharp image processing)
- **Match** — Matching algorithm, accept/reject, feedback, Redlock distributed locking
- **Match Status** — State machine: Active → Paused → Banned → Deactivated → Matched
- **SMS Chat** — LangGraph chatbot integration, daily message limits (100/user), Redis processing locks
- **Chatbot** — 5 LangGraph agents (Onboarding, General, Match, Profile Improvement, Yak), 40+ tools, MongoDB checkpointer
- **Internal** — 30+ admin sub-services: user management, match approval, AI matches, search (MeiliSearch), autofilter, stable matching, case library, poster generation
- **Email** — Template system with approval workflow, SendGrid + Postmark dual provider
- **Telegram** — Bot integration for admin notifications
- **Scheduler** — Appointment/availability management with email/Telegram notifications
- **Stripe** — Payment processing with webhook handling
- **Cloud Storage** — GCS (primary) + S3 backend, multi-format image support (JPEG, PNG, HEIC, AVIF)
- **LiveKit** — Video/audio session token generation
- **Profile Review** — AI-powered profile moderation via profile-analysis-service
- **System Chatbot** — Event-driven automated messages (match notifications, reminders)
- **Delayed Task** — RabbitMQ-based job scheduling
- **WebSocket** — Real-time events via RabbitMQ fanout exchange

**Auth Mechanisms:** gRPC auth service (primary), Google OAuth, Turnstile CAPTCHA, Alternate tokens (service-to-service), Internal API secret

**API Surface:** 40+ controllers with routes under `/user`, `/match`, `/chat`, `/sms-chat`, `/scheduler`, `/newsletter`, `/mailing-list`, `/waitlist`, `/stripe`, `/livekit`, `/auth`, `/internal/*`

---

### 2. imsg-service (iMessage/SMS Delivery)

RabbitMQ consumer microservice with pluggable message providers.

**RPC Methods (via RabbitMQ):** `sendMessage`, `initiateChat`, `lookupNumber`, `sendTypingIndicator`, `addContact`, `updateContactName`, `updateStatus`, `sendHealthCheckAck`

**Providers:** SendBlue, Linq, PhotonHQ, Virtual — each manages its own webhooks

**Dependencies:** MongoDB (message logs, chat mappings), Redis (rate limiting), RabbitMQ, MeiliSearch (chat indexing), PostHog

---

### 3. prompt-manager-backend (Prompt CMS)

NestJS backend for AI prompt management with immutable versioning.

**Key APIs:**
- `GET/POST /prompts` — CRUD for prompt metadata
- `GET/POST /prompts/:id/versions` — Immutable version management
- `POST /prompts/:id/versions/:id/publish` — Publish version to production
- RPC controllers for Chat, Profile, Match, Prompt operations

**LLM Support:** OpenAI, Anthropic Claude, Google Vertex AI

**Dependencies:** MongoDB, Redis, RabbitMQ, JWT auth

---

### 4. profile-analysis-service (AI Profile Analysis)

Runs AI-powered profile evaluation, matching, and embedding generation. Supports dual deployment: RabbitMQ and/or Restate durable workflows.

**RPC Methods:** `runAnalysis`, `generateImageEmbeddings`, `runAgent`, `runFeedbackAnalysis`, `runMatchMaker`, `runMatchingAgent`, `similarityMatch`

**Restate Handlers (10+):** userAnalysis, attractivenessMatching, hobbyMatching, heightMatching, vibeMatching, pimMatching, matchMaker, etc.

**Dependencies:** MongoDB, Redis, RabbitMQ, Replicate (CLIP embeddings), MeiliSearch, Restate, Vercel AI SDK, PostHog

---

### 5. delayed-task-service (Background Jobs)

Executes scheduled tasks (emails, SMS, queue messages) with optional admin approval.

**RPC Methods:** `getTasks`, `getAllTasks`, `approveTask`, `denyTask`, `deleteAutomationTasks`

**Task Types:** Email (SendGrid/Postmark), SMS (Twilio/SendBlue), Queue Messages (arbitrary RabbitMQ publishing)

**Dependencies:** MongoDB, Redis (Redlock), RabbitMQ, SendGrid, Postmark, Twilio

---

### 6. proj-coach-ws (User WebSocket)

Real-time Socket.IO server for end-user notifications.

**Flow:** RabbitMQ fanout exchange → `proj-coach-ws` → Socket.IO → User browser

**Events:** `sendMessage` (route to user), `chat:message:new` (new message notification)

**Auth:** gRPC `ValidateSession` against auth service

**Port:** 3000

---

### 7. proj-coach-voip (Voice Agent)

LiveKit agent for AI voice coaching using OpenAI Realtime API.

**Flow:** User joins LiveKit room → Agent connects → OpenAI Realtime (voice+text) → Conversation logged via RabbitMQ

**Dependencies:** LiveKit, OpenAI Realtime, RabbitMQ, prompt-manager-backend (for prompts)

---

### 8. ditto-internal-ws (Admin WebSocket)

Socket.IO server for the internal admin dashboard with presence tracking.

**Events:** `admin:chat:viewing`, `admin:chat:leave`, `admin:chat:typing`, `sendMessage`, `team:broadcast`

**Auth:** JWT validation via internal auth service public key

**Port:** 3002

---

### 9. ditto-internal-frontend (Admin Dashboard)

React SPA — the central command center for all Ditto operations.

**Tech:** React 18, Vite, Tailwind, Radix UI/shadcn, Jotai + Zustand + SWR state management

**Pages (25+):**
- **Matchmaking:** Matchmaker V2, AI Matchmaker, User Labeler, Matching Status, Ethnicity Analysis
- **Communications:** Chat Dashboard (SMS monitoring), Message Sender, Template Editor, Email History/Approvals
- **Analytics:** Analytics Dashboard, Sigma analytics, Schedules, Dates, Feedbacks
- **Developer:** PromptEditor (immutable versions, Monaco editor, multi-LLM), Asset Editor, AI Chat with MCP
- **Admin:** Access Management, Ditto Number Config, PosterMaker

**Performance:** IndexedDB caching for 3000+ profiles, incremental sync, route-based code splitting

**Dependencies:** Supabase (auth), MeiliSearch (user search), Socket.IO (real-time), PostHog (analytics), Vercel AI SDK

---

### 10. marketing-internal-tool (Poster Generator)

Next.js app for batch-generating marketing posters with QR codes.

**Features:** Template editor (text/image layers + QR slots), school-batch generation, PV/UV analytics via short links

**Dependencies:** Supabase (PostgreSQL), jspdf, jszip, Recharts

---

### 11. otel (OpenTelemetry Collector)

Custom OTel Collector distribution built with Go.

**Deployment:** Cloud Run sidecars + Kubernetes k3s

**Receivers:** OTLP (gRPC/HTTP), MongoDB, GCP Pub/Sub, Prometheus, K8s Cluster/Logs

**Exporters:** OTLP gRPC + HTTP to Grafana Tempo/Better Stack

---

### 12. matchmake_experimentation (R&D)

LLM-based match prediction validation using multi-agent photo analysis.

**3-Agent Pipeline:** Attractiveness Agent (50% weight) → Vibe & Aesthetic Agent → Lifestyle Agent

**Scoring:** Dual-track (text-only + vision via Gemini), weighted average

**MCP Server:** `analyze_profile`, `run_batch_analysis`, `get_profile_info`, `list_matches`

---

### 13. ufl (Synthetic Data Generation)

Python pipelines for generating test data.

**Pipelines:** Profile generation (deterministic) → LLM enrichment (Ollama/OpenAI) → Matching generation → Chat analysis (TODO)

**Dependencies:** MongoDB (Motor async), Ollama (local LLM), Sentence Transformers, HDBSCAN

---

### 14. event-202603-yik-yak (Event Matchmaking)

One-off Python engine for ~68k Yik Yak event users. Deterministic matching for romantic, friend, and group matches with 99.8% coverage.

---

### 15. claude-marketplace (Developer Tools)

Claude Code plugins: `/pr-comments` (address PR reviews), `/commit` (gitmoji commits), `/commit-push-pr` (full workflow), code context search via Exa.

---

## Service Communication Map

```
                    ┌───────────────────────┐
                    │   proj_coach_auth     │
                    │   (Go, gRPC :50051)   │
                    └──────────┬────────────┘
                         gRPC  │  ValidateSession
                               │  CreateSession
                               │  DeleteSession
┌──────────────┐               │
│ proj-coach-  │◄──────────────┘
│   backend    │
│  (NestJS)    │
│              │──── HTTP ────► prompt-manager-backend (:3002)
│              │                  (fetch prompts, model config)
│              │
│              │──── RabbitMQ ──► imsg-service
│              │    queue: "be"    (send SMS/iMessage)
│              │
│              │──── RabbitMQ ──► delayed-task-service
│              │  queue: "delayed-tasks" (scheduled emails, SMS)
│              │
│              │──── RabbitMQ ──► proj-coach-ws (:3000)
│              │  exchange: "sockets" (user WebSocket events)
│              │
│              │──── RabbitMQ ──► ditto-internal-ws (:3002)
│              │  exchange: "socket-internal" (admin WebSocket)
│              │
│              │◄─── RabbitMQ ─── profile-analysis-service
│              │                    (analysis results, embeddings)
│              │
│              │──── RabbitMQ ──► proj-coach-voip
│              │  exchange: "sockets" (voice chat messages)
└──────┬───────┘
       │
       │ Shared MongoDB (rs0), Redis, RabbitMQ
       │
┌──────▼───────────────────────────────────────────────────────────┐
│                    Shared Infrastructure                          │
│                                                                  │
│  MongoDB (:27017)   Redis (:6379)   RabbitMQ (:5672/:15672)    │
│  replica set rs0    sessions/cache  queues + fanout exchanges   │
│  db: ditto          Redlock         ClickHouse (:8123/:9000)    │
│  db: langgraph      rate limiting   analytics/time-series       │
└──────────────────────────────────────────────────────────────────┘
```

### RabbitMQ Queue/Exchange Map

| Queue/Exchange | Type | Producer | Consumer | Purpose |
|---------------|------|----------|----------|---------|
| `be` | Queue | proj-coach-backend | imsg-service | SMS/iMessage delivery |
| `delayed-tasks` | Queue | proj-coach-backend | delayed-task-service | Scheduled task execution |
| `sockets` | Fanout Exchange | proj-coach-backend, proj-coach-voip | proj-coach-ws | User WebSocket events |
| `socket-internal` | Fanout Exchange | proj-coach-backend | ditto-internal-ws | Admin WebSocket events |
| (RPC queues) | Queue | proj-coach-backend | profile-analysis-service | Profile analysis requests |
| (RPC queues) | Queue | profile-analysis-service | proj-coach-backend | Analysis results |

---

## MongoDB Database Layout

### `ditto` database (98 collections, ~1.3 GB on dev)

**Users & Profiles (11):** users, user_profiles, user_profile_histories, user_images, ideal_date_images, user_attractiveness, user_scores, user_score_history, user_vibes, height_preferences, user_ideal_personas

**Matching (12):** matchings, matching_histories, matching_statuses, match_approvals, ai_matchs, stats_match, profile_image_matches, profile_hobby_matches, match_maker_tasks, match_maker_steps, match_maker_results, matchmaker_weights

**Communication (12):** sms_chats, sms_chat_messages, chat_messages, chats, email_logs, email_groups, email_streams, email_template_config, pending_email_approvals, pending_chat_responses, imsg_logs, imsg_chat_id_mappings

**Content & Config (8):** assets, message_templates, msg_case_library, prompts, prompt_versions, automation_config, schools, pools

**Events & Waitlist (7):** event_tickets, event_applications, event_matchmakers, waitlist, waitlist_leaderboard, pre_signups, priority_users

**Utilities (6):** shortened_urls, shortened_url_logs, ditto_numbers, alternate_tokens, tg_notify_chats, health_check_pings

### `langgraph` database
- `checkpoints` — LangGraph state checkpoints
- `checkpoint_writes` — Pending state writes

---

## Chatbot System

The chatbot is the core AI product. Full details in [Chatbot - Conversation AI System Overview](./Chatbot%20-%20Conversation%20AI%20System%20Overview%20319d2ecb07cc800b8af0fd3b86bf6947.md).

### Agent Routing (priority cascade)

```
User Message
    │
    ├─ activePool === "yik-yak"? ──► Yak Agent (28 tools)
    │
    ├─ !completedOnboarding? ──────► Onboarding Agent (4 tools)
    │
    ├─ status === "Matched"? ──────► Match Agent (14 tools, loads match context first)
    │
    ├─ status === "NeedMoreInfo"
    │  or "InReview"? ─────────────► Profile Improvement Agent (12 tools)
    │
    └─ default ────────────────────► General Agent (13 tools)
```

### LangGraph Flow

```
START → loadProfileNode → loadMessagesNode → routeToAgent
    → {agent}_callModel ↔ {agent}_tools (loop, max 10 iterations)
    → sendResponse → END
```

### Guardrails
- **Qwen3Guard** — AI content moderation via Replicate (with regex fallback)
- **Emoji reaction check** — Skip responses to emoji-only messages
- **Quiet check** — Suppress stale responses if new user messages arrived during processing
- **Long response routing** — Messages > 3000 chars forced to pending approval queue

---

## External Service Integrations

| Service | Purpose | Used By |
|---------|---------|---------|
| **Vercel AI Gateway** | Unified LLM endpoint (Anthropic, OpenAI, etc.) | backend, profile-analysis |
| **OpenAI** | GPT models, Realtime API (voice) | backend, voip, prompt-manager |
| **Anthropic Claude** | Claude models | prompt-manager, matchmake_experimentation |
| **Google Vertex AI** | Gemini models | prompt-manager |
| **Replicate** | Qwen3Guard (moderation), CLIP (embeddings) | backend, profile-analysis |
| **LangSmith** | LangGraph tracing and observability | backend |
| **Restate** | Durable workflows for analysis pipelines | profile-analysis |
| **SendBlue / Linq / PhotonHQ** | iMessage providers | imsg-service |
| **Twilio** | SMS delivery | backend, delayed-task |
| **SendGrid** | Transactional email (primary) | backend, delayed-task |
| **Postmark** | Transactional email (secondary) | backend, delayed-task |
| **LiveKit** | VoIP / real-time audio rooms | backend, voip |
| **Stripe** | Payment processing | backend |
| **Google Cloud Storage** | File/image storage | backend |
| **MeiliSearch** | Full-text user/chat search | backend, imsg-service, internal-frontend |
| **Supabase** | Auth + PostgreSQL for internal tools | internal-frontend, marketing-tool |
| **Telegram Bot API** | Admin notifications | backend |
| **PostHog** | Product analytics | backend, imsg-service, profile-analysis, internal-frontend |
| **Grafana Loki** | Centralized log aggregation | backend (via Winston) |
| **Grafana Tempo** | Distributed tracing backend | otel collector |
| **Better Stack** | Monitoring | otel collector |
| **Ollama** | Local LLM for dev/testing | ufl |

---

## Infrastructure & Deployment

### Tailscale Network

| Host | IP | Role |
|------|----|------|
| desktop-quumnkk | 100.68.134.102 | Developer machine |
| proj-coach-dev | 100.99.190.45 | Dev environment |
| proj-coach-prod | 100.97.255.11 | Production |
| proj-coach-prod-infra | 100.86.210.116 | Prod infrastructure |
| proj-coach-prod-infra-2 | 100.80.187.40 | Prod infrastructure 2 |
| proj-coach-control-plane | 100.68.182.115 | K8s control plane |
| proj-coach-vcdb | 100.91.149.114 | Vector/analytics DB |

### Local Development Stack

| Component | Source | Port |
|-----------|--------|------|
| MongoDB | Installed locally (v8.0.20, rs0) | 27017 |
| Redis | Docker (docker-compose.dev.yml) | 6379 |
| RabbitMQ | Docker (docker-compose.dev.yml) | 5672 / 15672 |
| ClickHouse | Docker (docker-compose.dev.yml) | 8123 / 9000 |
| proj-coach-backend | `bun run local` | 8002 |
| prompt-manager-backend | `bun run start:dev` | 3002 |
| proj-coach-ws | `bun run src/index.ts` | 3000 |
| ditto-internal-ws | `bun run src/index.ts` | 3002 |
| ditto-internal-frontend | `bun run dev` | 5173 |

### Running Locally

```bash
# 1. Start infrastructure (Redis, RabbitMQ, ClickHouse — MongoDB is native)
cd projects/proj-coach-backend
docker compose -f docker-compose.dev.yml up -d

# 2. Create local config
cp config/config-template.toml config/config.local.toml
# Edit: set mongo uri to localhost, redis to localhost, etc.

# 3. Install and run backend
bun install
bun run local

# 4. (Optional) Start prompt manager
cd ../prompt-manager-backend
bun install && bun run start:dev

# 5. (Optional) Start internal frontend
cd ../ditto-internal-frontend
bun install && bun run dev
```

---

## Shared Package: @dodo-world/proj-coach-schemas

80+ Mongoose schemas published to GitHub Packages. Used by all TypeScript services.

**Install requires:** `NPM_TOKEN` with GitHub Packages read access

**Release:** `git tag v0.X.X && git push origin v0.X.X` → GitHub Actions publish

**Key schema domains:** Users (11), Matching (12), Communication (12), Content (8), Events (7), Utilities (6)

---

## Configuration Reference

All backend config uses TOML files in `config/`:

| Section | Key Settings |
|---------|-------------|
| `[mongo]` | `uri`, `db` |
| `[redis]` | `host`, `port`, `db` |
| `[rmq]` | `url`, `queue`, `socketsExchange`, `internalExchange`, `delayedTaskQueue` |
| `[auth]` | `grpcAddr`, `authRedirectUrl`, `[google]`, `[turnstile]` |
| `[prompt]` | `url`, `secret` |
| `[internal]` | `apiSecret`, `internalAuth` |
| `[sendgrid]` | `apiKey`, `verificationTemplate` |
| `[twilio]` | `accountSid`, `authToken`, `fromNumber` |
| `[telegram]` | `token`, `domain`, `secret` |
| `[cloudStorage]` | `bucket`, `backend`, `[s3]` |
| `[livekit]` | `apiKey`, `apiSecret` |
| `[stripe]` | `secretKey`, `webhookSecret` |
| `[meili]` | `host`, `apiKey` |
| `[sendblue]` | `keyId`, `secretKey`, `defaultNumber`, `webhookSecret`, `numberContactCardMapping` |
| `[posthog]` | `apiKey`, `host`, `secureApiKey` |
| `[qwen3guard]` | `apiToken`, `timeoutMs`, `enabled` |
| `[langsmith]` | `tracing`, `endpoint`, `apiKey`, `project` |
| `[tracing]` | `enabled`, `serviceName`, `exporter`, `endpoint`, `basicAuth` |
| `[loki]` | `host`, `basicAuth` |
| Top-level | `aiGatewayKey`, `domain`, `linkShortenerUrl`, `postmarkToken` |
