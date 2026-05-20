# OHL Platform — Comprehensive Technical Document

**OneHealthLink (OHL) · Medication Adherence SaaS Platform**
*Monorepo: `ohl-platform` · Last Updated: March 2026*

---

## Table of Contents

1. [Platform Purpose & Vision](#1-platform-purpose--vision)
2. [Repository Architecture](#2-repository-architecture)
3. [Application Deep-Dives](#3-application-deep-dives)
   - [3.1 Orchestration Engine (`orch-eng`)](#31-orchestration-engine----appsorch-eng)
   - [3.2 Chatbot API (`chatbot-api`)](#32-chatbot-api----appschatbot-api)
   - [3.3 OHL Chatbot API (`ohl-chatbot-api`)](#33-ohl-chatbot-api----appsohl-chatbot-api)
   - [3.4 Chatbot Widget (`chatbot-widget` / `ohl-chatbot-widget`)](#34-chatbot-widget----appschatbot-widget--appsohl-chatbot-widget)
   - [3.5 Admin Chatbot Backend (`admin-chatbot`)](#35-admin-chatbot-backend----appsadmin-chatbot)
   - [3.6 Admin Chatbot Frontend (`admin-chatbot-frontend`)](#36-admin-chatbot-frontend----appsadmin-chatbot-frontend)
   - [3.7 Live Agent Portal Backend (`lap-backend-app`)](#37-live-agent-portal-backend----appslap-backend-app)
   - [3.8 Rails Chat Backend (`rails`)](#38-rails-chat-backend----appsrails)
   - [3.9 Messaging API (`messaging-api`)](#39-messaging-api----appsmessaging-api)
   - [3.10 Messaging Worker (`messaging-worker`)](#310-messaging-worker----appsmessaging-worker)
   - [3.11 Airflow ETL (`airflow`)](#311-airflow-etl----appsairflow)
   - [3.12 OHL Monitoring Backend (`ohl-backend`)](#312-ohl-monitoring-backend----appsohl-backend)
   - [3.13 OHL Platform Frontend (`ohl-frontend`)](#313-ohl-platform-frontend----appsohl-frontend)
4. [Shared Libraries](#4-shared-libraries)
5. [System Architecture & App Relationships](#5-system-architecture--app-relationships)
6. [Data Flows End-to-End](#6-data-flows-end-to-end)
7. [Infrastructure & DevOps](#7-infrastructure--devops)
8. [What Makes This Platform Unique & Cutting-Edge](#8-what-makes-this-platform-unique--cutting-edge)

---

## 1. Platform Purpose & Vision

**OneHealthLink (OHL)** is a healthcare SaaS platform built to solve one of the most persistent and costly problems in managed care: **medication non-adherence**. Approximately 50% of patients with chronic conditions do not take their medications as prescribed, costing the US healthcare system over $300 billion annually in avoidable hospitalizations, emergency visits, and disease progression.

OHL targets **CMS Star Ratings measures** — the Centers for Medicare & Medicaid Services scoring system that directly determines health plan reimbursements. The three therapeutic categories tracked by CMS are:

| CMS Measure | Drug Class | Target Conditions |
|---|---|---|
| **D08** | Diabetes Medications (DME) | Type 2 Diabetes |
| **D09** | RASA/ARB/ACEI agents | Hypertension, Heart Failure |
| **D10** | Statins | High Cholesterol, CVD Risk |

The platform operates a **fully automated, AI-driven, regulatory-compliant member outreach lifecycle** — from ingesting pharmacy claims through SFTP, triggering personalized conversational outreach via SMS/voice, running intelligent chatbot sessions that understand medication barriers, escalating to pharmacists or live human agents when needed, and generating the monitoring data health plans need to track their Star Rating performance.

The entire system is built with **HIPAA compliance**, **TCPA (Telephone Consumer Protection Act) compliance**, and **AES-256 encryption of all PHI** as first-class architectural requirements.

---

## 2. Repository Architecture

The repository is an **Nx v22 monorepo** using a multi-language polyglot structure — an unusual and powerful combination of Python, TypeScript, and Ruby all managed within a single workspace.

```
ohl-platform/
├── apps/
│   ├── orch-eng/              # Python · FastAPI + Temporal (core brain)
│   ├── chatbot-api/           # Python · FastAPI + LangGraph (multi-client chatbot)
│   ├── ohl-chatbot-api/       # Python · FastAPI + LangGraph (OHL-enhanced chatbot)
│   ├── chatbot-widget/        # TypeScript · React 19 + Vite (chat UI)
│   ├── ohl-chatbot-widget/    # TypeScript · React 19 + Vite (OHL-branded chat UI)
│   ├── admin-chatbot/         # Python · FastAPI (script management API)
│   ├── admin-chatbot-frontend/# TypeScript · React 18 + Vite (flow editor UI)
│   ├── lap-backend-app/       # TypeScript · NestJS + Socket.IO (live agent portal)
│   ├── rails/                 # Ruby 3 · Rails 7 (chat message persistence)
│   ├── messaging-api/         # Python · FastAPI (unified SMS/Email/Voice API)
│   ├── messaging-worker/      # Python · Temporal worker (async message sending)
│   ├── airflow/               # Python · Apache Airflow 2.x (ETL pipelines)
│   ├── ohl-backend/           # Python · FastAPI (monitoring/dashboard API)
│   └── ohl-frontend/          # TypeScript · React 19 + Vite (main platform portal)
│
├── libs/
│   ├── messaging-core/        # Python · Domain entities, ports (zero deps)
│   ├── messaging-db/          # Python · SQLAlchemy + Alembic repositories
│   ├── messaging-providers/   # Python · Twilio + Mailgun adapters
│   ├── messaging-events/      # Python · Redis pub/sub event bus
│   ├── shared/                # Python · AES-256, PII masking, validation
│   ├── nsr-service/           # Python · LangGraph NSR AI agent
│   ├── telemetry/             # Python+Ruby+TS · OpenTelemetry wrappers
│   └── otel-azure/            # Python · Azure Application Insights integration
│
├── packages/
│   └── ai_governance/         # Python · AI prompt auditing, tamper detection, governance
│
├── docs/                      # Architecture documentation
├── scripts/                   # Start/stop/setup shell scripts
├── nx.json                    # Nx workspace config
└── package.json               # Root workspace + all Nx task scripts
```

**Monorepo toolchain:**
- **Nx v22** — task orchestration, project graph, caching, affected-build detection
- **`@nxlv/python`** — Nx plugin enabling Python (Poetry) projects in the workspace
- **`@nx/js`** — Nx plugin for TypeScript/Node projects
- **Poetry** — dependency management for all Python apps and libs
- **npm** — root-level package manager for TypeScript apps

---

## 3. Application Deep-Dives

---

### 3.1 Orchestration Engine — `apps/orch-eng`

**Runtime:** Python 3.12 · FastAPI · Temporal · SQLAlchemy · Alembic · PostgreSQL  
**Port:** 8002  
**Role:** The central nervous system of the entire platform.

#### Purpose

`orch-eng` is the durable, stateful **medication adherence workflow engine**. It manages every enrolled member's entire outreach lifecycle using Temporal's workflow orchestration, ensuring that no interaction is ever lost even through server crashes, network failures, or long multi-week gaps between interventions.

#### Use Cases (6 Intervention Types)

| UC | Name | Trigger | Goal |
|---|---|---|---|
| **UC1** | Medication Onboarding | New prescription filled | Welcome + adherence education |
| **UC2** | Refill Reminder | ~7 days before refill due | Proactive refill prompt |
| **UC3** | Missed Refill Recovery | Refill overdue | Re-engagement outreach |
| **UC4** | 30→90 Day Conversion | On 30-day supply | Motivate switch to 90-day/mail-order |
| **UC5** | Prescription Renewal | Rx expiry approaching + no refills left | Facilitate provider contact via SureScripts |
| **UC6** | Monthly Check-In | Ongoing adherence monitoring | Barrier identification, ongoing support |

#### Temporal Workflow Hierarchy

```
MemberWorkflow (global, one per member, runs indefinitely)
└── CategoryWorkflow × 3 (one per therapeutic class: DME / RASA / Statin)
    └── UseCaseWorkflow (active intervention for that category)
        └── Activities: rules evaluation → message send → callback wait → escalation
```

This three-level hierarchy means a single member can have up to 3 active category workflows concurrently (one per drug class), each running a different intervention use case, all coordinated by their parent `MemberWorkflow`.

#### Rules Engine

`rules/engine.py` evaluates all 6 use cases on every eligibility check and returns a prioritized `EligibilityResult`. The engine considers:
- Current claim data (fill date, refills remaining, days supply, expiry date)
- Existing active workflows (prevents duplicate outreach)
- Consent and enrollment status
- Time-based eligibility windows (e.g., UC2 activates 7 days before refill due)

#### TCPA Compliance Engine

Hard-coded compliance guardrails in every messaging activity:
- Maximum **1 message per day** per member
- Maximum **3 messages per week** per member
- Outreach windows restricted to **8:00 AM – 6:00 PM local member time**
- Opt-out/DNC list enforcement before every send

#### Escalation Integrations

- **Google Sheets:** Creates tasks for pharmacist review when automated flows fail
- **Zoom Contact Center:** Triggers supervised outbound calls
- **SureScripts:** Sends electronic prescription renewal requests to prescribers (UC5)

#### API Surface (consumed by Airflow)

```
POST /api/v1/members/onboard             # Initial member enrollment
POST /api/v1/ingest/claims               # NCPDP claim signal → Temporal signal
POST /api/v1/members/{id}/eligibility-update  # Enrollment status changes
POST /api/v1/callbacks/surescripts       # Provider response callbacks
POST /api/v1/callbacks/escalation        # Pause/resume category workflows
```

---

### 3.2 Chatbot API — `apps/chatbot-api`

**Runtime:** Python 3.12 · FastAPI · LangGraph · LangChain · OpenAI · PostgreSQL · Redis · WebSocket  
**Port:** 8000  
**Role:** The multi-client conversational AI engine — the production chatbot backend serving multiple health plan clients.

#### Purpose

`chatbot-api` powers real-time, multi-turn, intelligent conversations with health plan members. It is built around a **database-driven conversation graph** — the entire flow logic (nodes, transitions, conditions) lives in PostgreSQL, not code, enabling non-engineers to create and modify conversation flows through the admin panel without any deployments.

#### Conversation Architecture

The engine uses **LangGraph** as the underlying state machine runtime, but dynamically loads the graph structure from the database at runtime:

```
PostgreSQL (nodes table)
    ↓ load at session start
LangGraph State Machine
    ↓ process each user message
Node Evaluation → Condition Matching → State Transition
    ↓ AI-assisted for non-standard replies
Response Generation → WebSocket stream to member
```

#### Dual Module Architecture

| Module | Type | Behavior |
|---|---|---|
| `scripted_conversation` | Rule-based | Strict DB-driven flow, scripted responses, deterministic |
| `assessment` | AI-powered | LLM-augmented flows, open-ended responses, RAG-supported |

#### Assessment Flow Coverage

The `assessment` module handles 10 clinical/operational programs:
- Medication Onboarding, Refill Reminder, Missed Refill Recovery, Prescription Renewal, Monthly Check-In
- CSNP (Chronic Special Needs Plan) Conversion
- Cancer Screening, Wheelchair assessment, Actemra prescribing support

#### AI Features

- **NSR (Non-Standard Reply) Handling:** When a member's message doesn't match expected inputs, an LLM agent (via `libs/nsr-service`) interprets intent and routes appropriately
- **RAG Knowledge Base:** Multi-vector-store support (Pinecone, FAISS, ChromaDB) for medication education queries
- **Multilingual Support:** English and Spanish
- **Archetype/Persona Logic:** Adapts communication style to member profiles

#### Session & Real-Time Features

- **WebSocket endpoints:** `/chat` (new session), `/resume/{member_id}` (resume interrupted session)
- **Redis caching:** Session state, message break timing, invalidation listeners
- **Typing indicators:** Simulated human-like response timing
- **LAP Integration:** WebSocket bridge to `lap-backend-app` when human handoff is needed

#### Condition Types (routing logic)

`auto`, `input_in`, `sql`, `daytime`, `variable_in`, `session_metadata_in`, `http_response_in`, `regex_match`, `intent_match`

---

### 3.3 OHL Chatbot API — `apps/ohl-chatbot-api`

**Runtime:** Python 3.12 · FastAPI · LangGraph · spaCy · Pinecone · Azure Service Bus · WebSocket  
**Port:** 8000 (alternative deployment)  
**Role:** OHL's enhanced, production-hardened chatbot API with advanced orchestration capabilities added in early 2026.

#### What Differentiates It From `chatbot-api`

`ohl-chatbot-api` is the strategic evolution of the chatbot platform with OHL-specific hardening:

**1. The Orchestrator Pattern (February 2026)**

A sophisticated multi-step conversation management system:
- **Script-as-checklist:** Tracks which questions in a script have been answered, not just current node position
- **Interruption Recovery:** Can seamlessly return to interrupted scripts after handling off-topic messages
- **Immediate Script Return:** Priority mechanism to always return to primary script after diversions
- **Multi-question awareness:** Can detect when a member's single message answers multiple script questions simultaneously

**2. Three-Tier Medical Escalation**

Tiered emergency handling:
```
Symptom detected → Pharmacist consultation
    ↓ (if critical)
    → Doctor referral
        ↓ (if life-threatening)
        → 911 emergency escalation
```

**3. Barrier Education Integration**

- Automatic detection of medication adherence barriers (cost, side effects, forgetting, beliefs)
- Triggers education flows specific to each identified barrier type
- Tracks barrier resolution across sessions

**4. Intent Analysis Engine**

Single LLM call analyzes all user intents simultaneously:
```
Priority order: emergency → urgency → empathy → script questions
```

**5. Azure Service Bus Integration**

Asynchronous message queuing to `orch-eng` for claim updates and workflow signals, replacing synchronous HTTP calls for improved reliability.

**6. LLM Observability**

Arize Phoenix tracing integration — full visibility into every LLM call, token usage, latency, and decision paths.

#### Production Flow Coverage

`refill_reminder`, `medication_onboarding`, `missed_refill`, `prescription_renewal`, `monthly_checkin`, `mail_order_conversion`

#### Database Schema

22 Alembic migrations (`001`–`022`) including dedicated tables for: workflows, executions, barriers, consent records, escalations, monitoring events, orchestration state, script checklists, silent disease education.

---

### 3.4 Chatbot Widget — `apps/chatbot-widget` / `apps/ohl-chatbot-widget`

**Runtime:** TypeScript · React 19 · Vite 6 · TailwindCSS 4 · Jest  
**Port:** 5173  
**Role:** The embeddable member-facing chat interface.

#### Purpose

A lightweight, embeddable React widget that renders the conversation UI in the member's browser. The widget connects to either `chatbot-api` or `ohl-chatbot-api` via WebSocket and handles real-time streaming rendering.

#### Key Features

**Adaptive Input System**

The widget renders different input components based on the `expected_input_type` field returned by the API:
- Text / number / money+currency / date picker
- Multiple choice buttons (single or multi-select)
- Inline action links

**Real-Time Streaming**
- Character-by-character typing effect for AI responses
- Typing indicators during server processing
- Sound notifications for new messages
- Optimistic UI updates

**Session Management**
- Automatic session token handling
- `?standalone=true` mode for full-page deployment
- `MainPage` embedded mode for iframe/widget deployment
- Session metadata passed on connect (member ID, plan, language)

**Theming System**
- `ThemeProvider` context for brand-level CSS variables
- Domain-specific design configuration loaded from API response (`design_config` object)
- Per-client color schemes, fonts, logos — all server-driven

`ohl-chatbot-widget` is the OHL-branded variant, kept in parallel for independent deployment configuration while maintaining functional parity.

---

### 3.5 Admin Chatbot Backend — `apps/admin-chatbot`

**Runtime:** Python 3.12 · FastAPI · SQLAlchemy · PostgreSQL  
**Port:** 8001  
**Role:** The configuration API for managing chatbot conversation scripts and flows.

#### Purpose

Provides the REST API layer for the admin panel. All chatbot conversation logic — nodes, edges, conditions, variables, templates — is stored in PostgreSQL and managed through this API. This is what enables zero-deployment flow changes.

#### Managed Entities

- **Scripts:** Top-level conversation programs (versioned, publishable)
- **Flows:** Node/edge graph definitions within a script
- **Nodes:** Individual conversation steps (text, input, condition, action)
- **Transitions/Edges:** Directed connections between nodes with conditions
- **Variables:** Session-scoped and global variables
- **Templates:** Reusable message content with interpolation
- **Conditions:** Logical rules for branching (`input_in`, `sql`, `daytime`, etc.)
- **Clients / Campaigns:** Multi-tenant organization
- **HTTP Configs:** External API integrations triggered from flows
- **Node Groups:** Reusable sub-flow components
- **Quiet Periods:** TCPA-compliant blackout windows

#### Version Control

Scripts support a full lifecycle: `draft → published → deprecated`. New versions can be created from any published script, enabling safe iteration without disrupting active sessions.

---

### 3.6 Admin Chatbot Frontend — `apps/admin-chatbot-frontend`

**Runtime:** TypeScript · React 18 · Vite · React Router v6 · TanStack Query · React Hook Form · Yup · TailwindCSS · Axios  
**Port:** 3000  
**Role:** The single-page admin web application for chatbot configuration.

#### Purpose

A rich, visual SPA that non-technical clinical program managers use to design and manage conversation flows without writing code.

#### Key UI Features

- **Visual Flow Editor:** Canvas-based node/edge graph editor (`FlowDetailV2Page`) for building conversation flows visually
- **Script Management:** Create, publish, version, and deprecate conversation scripts
- **Entity Management:** Full CRUD for all admin entities (clients, campaigns, variables, templates, actions, node groups, HTTP configs, quiet periods)
- **Action Builder:** Configure automated actions within flows — Email sends, SMS sends, SQL queries with parameterized inputs
- **Real-time filtering and pagination** across all list views
- **JWT Authentication:** Login flow with token stored in localStorage, auto-refresh

**Tech Stack Choices**
- TanStack Query for all server state — automatic background refetch, cache invalidation
- React Hook Form + Yup for all form validation — schema-driven, typed
- Axios with interceptors for unified error handling and auth headers

---

### 3.7 Live Agent Portal Backend — `apps/lap-backend-app`

**Runtime:** TypeScript · Node.js 20 · NestJS 10 · TypeORM · PostgreSQL · Redis · BullMQ · Socket.IO · Auth0 · Firebase  
**Port:** 3000 (main API)  
**Role:** The human agent portal — enabling trained health coaches and pharmacists to handle member conversations that the AI cannot fully resolve.

#### Purpose

When the chatbot AI reaches its limits (complex medication questions, emotional distress, SureScripts follow-up), it seamlessly hands off to a live human agent through LAP. LAP is a full multi-tenant contact center platform built on NestJS.

#### Architecture: Three Entry Points

```
apps/lap-backend-app/
├── src/api/           # Main public API (NestJS app, port 3000)
├── src/internalApi/   # Private subnet API (trusted internal services)
└── src/worker/        # BullMQ background job processor
```

#### Core Capabilities

**Real-Time Chat Infrastructure**
- Socket.IO with Redis adapter for horizontal scaling
- Agent presence management (online/away/offline)
- Conversation routing rules (round-robin, skill-based, priority)
- Real-time typing indicators and read receipts

**Multi-Tenant Organization Management**
- Organizations with configurable RBAC (granular roles and permissions)
- Agent performance dashboards
- Routing rule configuration per organization

**AI Copilot for Agents**
- In-conversation AI assistance — suggests responses, surfaces relevant KB articles
- Reduces average handle time and improves response quality

**Integration Surface**
- Auth0 JWKS JWT verification for authentication
- Firebase Cloud Messaging for agent mobile push notifications
- Azure Blob Storage for file attachments in conversations
- BullMQ + Redis for background job processing (notifications, analytics events)
- ROR Chats App webhook (`/api/v1/ror-chats-app/webhook`) for external chatbot V1 integration

**CQRS Pattern**

Uses `@nestjs/cqrs` throughout — Command/Query separation ensures clean boundaries between write operations (commands) and read operations (queries), making the codebase testable and scalable.

**Observability**
- Sentry error tracking with request context
- Azure Application Insights telemetry (via `libs/telemetry`)

---

### 3.8 Rails Chat Backend — `apps/rails`

**Runtime:** Ruby 3.0 · Rails 7 · PostgreSQL · Doorkeeper (OAuth2) · Sidekiq · Rswag  
**Role:** The foundational chat message and user persistence layer — the original infrastructure that predates NestJS.

#### Purpose

Rails serves as the persistent store for the underlying chat infrastructure: users, groups, conversations, messages, and push notification device registrations. It is the "source of truth" for raw chat data.

#### Key Models

- **User:** Polymorphic roles (`bot`, `admin`, `user`, `business`, `support`, `application_web_admin`)
- **Conversation:** Thread container with group support
- **Message:** Individual messages with metadata
- **Device:** Push notification tokens (iOS/Android) for member mobile apps
- **Group:** Multi-party conversation groups

#### Integration Role

- Exposes a `/lap/webhook` endpoint that `lap-backend-app` calls for chat lifecycle events
- Doorkeeper OAuth2 provides token-based auth for API access
- Sidekiq processes background jobs including `not_responding_message_job.rb` — triggers automated follow-up messages when conversations go unanswered

**Architectural Context**

Rails is a more mature, legacy component being progressively wrapped by the NestJS `lap-backend-app`. NestJS handles modern agent portal features while Rails handles the raw message persistence layer — a strangler fig pattern migration.

---

### 3.9 Messaging API — `apps/messaging-api`

**Runtime:** Python 3.12 · FastAPI · PostgreSQL · Alembic  
**Port:** 8002  
**Role:** The unified external-facing communication gateway — every SMS, email, and voice call in the platform flows through here.

#### Purpose

`messaging-api` is a single, auditable, encrypted, rate-limited communication hub. No application in the platform sends messages directly to Twilio or Mailgun — they all route through this API, ensuring consistent compliance, logging, and audit trails.

#### Hexagonal Architecture

```
HTTP Router (FastAPI)
    ↓
Application Service Layer (business logic, validation, TCPA check)
    ↓
Ports (interfaces)
    ├── MessageRepository Port → libs/messaging-db (PostgreSQL)
    ├── SMSProvider Port       → libs/messaging-providers/twilio
    ├── EmailProvider Port     → libs/messaging-providers/mailgun
    └── EventPublisher Port    → libs/messaging-events (Redis)
```

#### Features

**Dual Invocation Modes**
1. **HTTP REST mode** — for external or cross-service calls with `X-API-Key` auth header and rate limiting (60 req/min default)
2. **Direct service call mode** — for trusted internal callers (e.g., `messaging-worker`) that import the service layer directly, skipping HTTP overhead and auth

**Security**
- AES-256-CBC encryption of all PII before database persistence (phone numbers, email addresses, message content)
- HIPAA-compliant logging — PII masked in all log output via `libs/shared`
- API key authentication with per-key rate limiting

**Capability Matrix**

| Channel | Provider | Features |
|---|---|---|
| SMS | Twilio | Text + MMS (media_urls), delivery callbacks |
| Email | Mailgun | HTML body or template-based, event webhooks |
| Voice | Twilio | Outbound call initiation |

**Webhook Endpoints**
- `POST /webhooks/twilio/status` — Delivery status callbacks (delivered, failed, undelivered)
- `POST /webhooks/twilio/inbound` — Inbound SMS from members (opt-outs, responses)

---

### 3.10 Messaging Worker — `apps/messaging-worker`

**Runtime:** Python 3.12 · Temporal  
**Role:** Asynchronous Temporal worker for processing message sending activities.

#### Purpose

`messaging-worker` acts as the Temporal activity executor for all asynchronous messaging operations. When `orch-eng` workflows need to send a message, they schedule a Temporal activity that this worker picks up and executes — providing durable, retry-safe delivery with Temporal's guarantees.

#### How It Works

```
orch-eng Temporal Workflow
    ↓ schedule activity: send_message
Temporal Server (activity queue)
    ↓ dispatch to registered worker
messaging-worker (Temporal Worker)
    ↓ calls messaging-api service layer directly (no HTTP)
libs/messaging-providers (Twilio/Mailgun)
    ↓ delivery status
libs/messaging-events (Redis event bus)
    ↓ publish: MessageSent / MessageFailed event
```

This design gives `orch-eng` workflows durable message delivery — if the worker crashes mid-send, Temporal automatically retries the activity on the next available worker instance.

---

### 3.11 Airflow ETL — `apps/airflow`

**Runtime:** Python 3.11 · Apache Airflow 2.x · SFTP · PostgreSQL  
**Port:** 8080 (Airflow Web UI)  
**Role:** ETL orchestration — the data ingestion pipeline that feeds member and claims data into the platform.

#### Purpose

`airflow` is the data pipeline that connects external pharmacy data sources (CVS, Clover Health) to the OHL platform. Without Airflow, `orch-eng` has no data to act on — Airflow is what triggers the entire medication adherence lifecycle.

#### DAG Catalog

**`sftp_file_processing` (every 6 hours)**
```
SFTP server (CVS pharmacy claims CSV)
    ↓ extract: pull and decrypt files
    ↓ validate: schema validation, duplicate detection
    ↓ transform: NCPDP format normalization
    ↓ load: write to PostgreSQL patient_medications table
    ↓ trigger: POST /api/v1/ingest/claims on orch-eng
```

**`clover_member_etl_dag`**
```
Clover Health member feed (enrollment/disenrollment)
    ↓ extract + transform
    ↓ POST /api/v1/members/onboard on orch-eng (new enrollments)
    ↓ POST /api/v1/members/{id}/eligibility-update (status changes)
```

**`cvs_claims_etl_dag`**
```
CVS pharmacy claims data
    ↓ extract + transform (NCPDP normalization)
    ↓ POST /api/v1/ingest/claims on orch-eng
```

#### Design

DAGs are modular — each step (extract, validate, transform, load, trigger) is a separate Airflow task, enabling fine-grained retry control and partial re-runs when a single step fails.

---

### 3.12 OHL Monitoring Backend — `apps/ohl-backend`

**Runtime:** Python 3.12 · FastAPI · PostgreSQL  
**Port:** 8001  
**Role:** Monitoring and observability API for the `ohl-frontend` dashboard.

#### Purpose

`ohl-backend` aggregates data from across the platform — PostgreSQL member data, Temporal workflow histories, and Airflow DAG run logs — into a unified REST API that the `ohl-frontend` dashboard consumes for operational visibility.

#### API Endpoints

```
GET /api/v1/monitoring/flows                           # End-to-end member outreach flows
GET /api/v1/monitoring/members/{id}/claims             # Claims history for a member
GET /api/v1/monitoring/orchestration/decisions         # Orch-eng rules engine decisions log
GET /api/v1/monitoring/temporal/workflows/{id}/history # Temporal workflow event history
```

#### Integration Sources

| Data Source | Purpose |
|---|---|
| PostgreSQL (`orch-eng` schema) | Member data, claims, therapeutic categories, decisions |
| Temporal HTTP API | Workflow execution history and status |
| Airflow DB | DAG run status, task-level success/failure |

---

### 3.13 OHL Platform Frontend — `apps/ohl-frontend`

**Runtime:** TypeScript · React 19 · Vite 6 · React Router v6 · Zustand · TanStack Query · TailwindCSS 4 · Axios · Lucide React  
**Port:** 3001  
**Role:** The main operational portal for health plan clients and OHL staff.

#### Purpose

The primary web interface for health plan clients to monitor their medication adherence programs, manage client configurations, and view platform performance. It provides program managers with visibility into the entire member journey.

#### Pages & Features

- **Dashboard:** Program-level KPIs (adherence rates, outreach counts, escalation rates)
- **Programs Management:** View and configure medication adherence programs
- **Client Management:** Multi-client administration
- **System Configuration:** Platform-level settings

**State Management Stack**
- Zustand for client-side application state (selected client, filters, UI state)
- TanStack Query for all server state (fetching, caching, background refresh)

---

## 4. Shared Libraries

The `libs/` directory contains 8 shared Python/TypeScript libraries forming the platform's foundational layer.

---

### `libs/messaging-core`

**Language:** Python · Zero external dependencies  
**Pattern:** Hexagonal Architecture — Domain + Ports (no infrastructure)

The **pure domain layer** for all messaging operations. Defines:
- Domain entities: `Message`, `Conversation`, `ConsentRecord`
- Enums: `Channel` (SMS/Email/Voice), `MessageType`, `ConsentStatus`, `OptOutReason`
- `ConsentService`: Enforces TCPA and CAN-SPAM opt-in/opt-out rules
- Port interfaces (abstract base classes) that `messaging-db` and `messaging-providers` implement

No database, no HTTP, no external dependencies — pure Python domain logic. This enables the entire messaging domain to be tested without any infrastructure.

---

### `libs/messaging-db`

**Language:** Python · SQLAlchemy (async) · Alembic · PostgreSQL

Infrastructure implementation of the `messaging-core` repository ports:
- `MessageRepository` — async CRUD for message records
- `ConsentRepository` — consent record management, opt-in/opt-out history
- `ConversationRepository` — conversation threading
- `MailgunEventRepository` — email event tracking (open, click, delivery)
- Full Alembic migration history for `messaging` and `shared` PostgreSQL schemas

---

### `libs/messaging-providers`

**Language:** Python · httpx · Twilio SDK · Mailgun REST

Infrastructure implementation of the `messaging-core` provider ports:
- `TwilioAdapter`: SMS, MMS, Voice — implements `SMSProvider` and `VoiceProvider` ports
- `MailgunAdapter`: Transactional email — implements `EmailProvider` port
- **Factory pattern** for runtime adapter selection based on configuration

---

### `libs/messaging-events`

**Language:** Python · Redis · Pydantic

Async event bus for messaging lifecycle events:
- Events: `MessageSent`, `MessageReceived`, `MessageDelivered`, `MessageFailed`, `ConsentChanged`, `ConversationStarted`
- Redis pub/sub with Pydantic models for type-safe event payloads
- Both publisher and subscriber implementations with async/await

---

### `libs/shared`

**Language:** Python

Platform-wide security and utility library:
- **AES-256-CBC encryption** with PKCS7 padding for PHI/PII fields
- **E.164 phone number validation** and normalization
- **Email validation** with domain verification
- **PII masking** for HIPAA-compliant logging (masks phone numbers, emails, names in log output)
- Optional **OpenTelemetry** integration hooks

---

### `libs/nsr-service`

**Language:** Python · LangGraph · Azure OpenAI

The **Non-Standard Reply (NSR) AI agent** — a 5-step LangGraph pipeline that handles member messages that don't match expected conversation inputs:

```
Step 1: Context Enrichment     — pull member history, current script context
Step 2: Compliance Validation  — check HIPAA-safe response boundaries
Step 3: Response Generation    — Azure OpenAI LLM call with context
Step 4: Tone Adjustment        — match member communication style
Step 5: Final Validation       — safety + compliance check before sending
```

Loaded with client-specific configuration from PostgreSQL — each health plan client can have different response boundaries, tone guidelines, and escalation thresholds.

---

### `libs/telemetry`

**Language:** Python + Ruby + TypeScript (multi-language)

OpenTelemetry wrappers for all three platform languages targeting Azure Container Apps:
- **Python:** FastAPI auto-instrumentation, ASGI middleware, trace context propagation
- **Ruby:** Rails and Sidekiq instrumentation
- **TypeScript:** React frontend trace injection, Web Vitals integration

Auto-detects Azure Container Apps managed OpenTelemetry endpoint and configures exporters accordingly. Enables distributed tracing across the entire polyglot platform.

---

### `libs/otel-azure`

**Language:** Python

Standalone Azure Application Insights integration:
- `init_telemetry()` — bootstraps the OpenTelemetry SDK with Azure exporter
- `instrument_fastapi(app)` — adds FastAPI middleware with automatic span creation
- Connection string-based Azure Application Insights configuration

---

### 4.1 Packages

The `packages/` directory contains standalone, reusable Python packages that are installed into apps via Poetry path dependencies.

---

### `packages/ai_governance`

**Language:** Python · Pydantic · structlog · OpenTelemetry · Azure Monitor · Redis
**Version:** 0.1.0
**Role:** Centralized AI prompt auditing, telemetry, and compliance governance for all LLM-powered services.

#### Purpose

In a HIPAA-regulated healthcare platform where LLMs generate clinical content for patients, every AI interaction must be auditable, tamper-proof, and observable. `ai_governance` is the shared package that wraps every prompt/response exchange across the platform with structured audit logging, tamper detection, and durable delivery guarantees.

Any service making LLM calls (`chatbot-api`, `ohl-chatbot-api`, `nsr-service`) installs this package and instruments its AI interactions with a single function call.

#### Architecture

```
ai_governance/
├── core/
│   ├── config.py        # Pydantic Settings config (AI_GOV_* env vars)
│   ├── telemetry.py     # OpenTelemetry + structlog initialization
│   └── audit.py         # emit_prompt_response() — the primary API
├── queue/
│   ├── base.py          # Abstract queue interface
│   ├── file_queue.py    # File-based durable queue (dev/fallback)
│   ├── redis_queue.py   # Redis-based durable queue (production)
│   └── factory.py       # Queue backend factory + enqueue_fallback()
├── adapters/
│   ├── base.py          # EvaluationAdapter abstract interface
│   └── registry.py      # Pluggable adapter registry
├── security/
│   └── tamper.py        # HMAC-SHA256 signing & verification
└── workers/
    └── verification.py  # Background tamper detection worker
```

#### Core Capabilities

**1. Prompt/Response Audit Logging**

The primary API — `emit_prompt_response()` — captures every LLM interaction with rich metadata:

```python
from ai_governance import initialize_telemetry, emit_prompt_response, PromptMetadata

# At app startup (once)
initialize_telemetry(app)

# On every LLM call
result = await emit_prompt_response(
    prompt=user_message,
    response=llm_response,
    prompt_metadata=PromptMetadata(
        user_id="member-123",
        session_id="session-456",
        model="gpt-4",
        temperature=0.7,
        tags={"flow": "refill_reminder", "use_case": "UC2"},
    ),
)
```

Each event captures: prompt text, response text, prompt/response lengths, user_id, session_id, model name, temperature, token count, latency_ms, finish reason, and arbitrary tags. Events are emitted to both **structlog** (structured JSON for application logs) and **OpenTelemetry** (for Azure Application Insights export).

**2. Content Redaction (HIPAA Compliance)**

For environments where prompt/response content may contain PHI:

```python
# Global redaction via config
TelemetryConfig(redact_prompts=True, redact_responses=True)

# Per-call redaction
await emit_prompt_response(prompt, response, redact_content=True)
# Logs "[REDACTED]" instead of actual text, but still captures metadata
```

**3. Tamper-Proof Audit Trail (HMAC-SHA256)**

Every audit event can be cryptographically signed to detect post-hoc tampering — a regulatory requirement for healthcare audit logs:

```python
from ai_governance import sign_event, verify_event

signed = sign_event(audit_event)
# Adds: _signature (HMAC-SHA256), _signed_at (timestamp), _signature_version

is_authentic = verify_event(signed)
# Uses constant-time comparison to prevent timing attacks
```

The signing secret is loaded from Azure Key Vault via environment variable (`AI_GOV_SIGNING_SECRET`). Canonical JSON serialization (sorted keys, no whitespace) ensures deterministic signature computation.

**4. Durable Fallback Queue**

If Azure Application Insights is unreachable (network issues, service outages), audit events are never lost:

```python
from ai_governance import enqueue_fallback, get_fallback_queue

# Automatic fallback on export failure
try:
    export_to_app_insights(event)
except Exception as e:
    await enqueue_fallback(event, metadata={"error": str(e)})

# Two backends:
# - File queue (default): writes JSON to ./fallback_queue/ directory
# - Redis queue (production): RPUSH to configurable Redis URL
```

A background worker replays queued events when the exporter recovers.

**5. Pluggable Evaluation Adapters**

Extensible adapter system for content safety evaluation — toxicity scoring, PII detection, clinical accuracy checks:

```python
from ai_governance import EvaluationAdapter, register_evaluation_adapter

class ToxicityAdapter(EvaluationAdapter):
    async def evaluate(self, prompt, response, prompt_id, response_id, context=None):
        score = await toxicity_model.score(response)
        return EvaluationMetadata(
            prompt_id=prompt_id,
            response_id=response_id,
            adapter_name="toxicity",
            scores={"toxicity": score},
            flags=["unsafe"] if score > 0.7 else ["safe"],
        )

register_evaluation_adapter(ToxicityAdapter(name="toxicity"))
```

Adapters are invoked on every `emit_prompt_response()` call and their results are included in the audit event.

**6. Background Verification Worker**

A long-running async worker that periodically scans recent audit events and verifies their HMAC signatures:

```python
from ai_governance import start_verification_worker

worker = await start_verification_worker(
    alert_callback=lambda alert: send_to_pagerduty(alert),
)
# Runs every 300 seconds (configurable), scans last 24 hours
# Fires alert_callback if any signatures fail verification
```

#### Configuration

All settings via `AI_GOV_*` environment variables (Pydantic Settings):

| Variable | Default | Description |
|---|---|---|
| `AI_GOV_SERVICE_NAME` | `ai-service` | Service identifier in telemetry |
| `AI_GOV_ENVIRONMENT` | `development` | Deployment environment tag |
| `AI_GOV_ENABLED` | `true` | Master telemetry toggle |
| `AI_GOV_REDACT_PROMPTS` | `false` | Strip prompt text from logs |
| `AI_GOV_REDACT_RESPONSES` | `false` | Strip response text from logs |
| `AI_GOV_FALLBACK_QUEUE_BACKEND` | `file` | `file` or `redis` |
| `AI_GOV_FALLBACK_REDIS_URL` | — | Redis connection for production queue |
| `AI_GOV_SIGNING_SECRET` | — | HMAC key (from Key Vault in prod) |
| `AI_GOV_VERIFICATION_INTERVAL_SECONDS` | `300` | Tamper check frequency |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | — | Azure App Insights export target |

#### Telemetry Pipeline

```
LLM Call in chatbot-api / ohl-chatbot-api
    ↓
emit_prompt_response()
    ├── structlog → JSON stdout → Azure Container Apps log stream
    └── OpenTelemetry LoggingHandler → BatchLogRecordProcessor
            ↓
        AzureMonitorLogExporter → Application Insights
            ↓ (if export fails)
        enqueue_fallback() → Redis / file queue → retry on recovery
```

In Azure Application Insights, events are queryable via Kusto:

```kusto
traces
| where message == "prompt_response_audit"
| extend user = tostring(customDimensions.user_id),
         model = tostring(customDimensions.model),
         latency = todouble(customDimensions.latency_ms)
| summarize avg_latency = avg(latency), count() by model, bin(timestamp, 1h)
```

#### Why This Package Exists

| Concern | How `ai_governance` Addresses It |
|---|---|
| **HIPAA audit trail** | Every LLM interaction logged with user, session, timestamp, model |
| **Tamper evidence** | HMAC-SHA256 signing + background verification worker |
| **PHI in prompts** | Configurable content redaction (`redact_prompts`, `redact_responses`) |
| **Content safety** | Pluggable adapters for toxicity, PII, clinical accuracy |
| **Observability** | Structured logs + Azure Application Insights + OpenTelemetry tracing |
| **Reliability** | Durable fallback queue ensures zero event loss |
| **Consistency** | Single package enforces same audit format across all AI services |

---

## 5. System Architecture & App Relationships

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                             EXTERNAL WORLD                                    │
│  SFTP (CVS Claims)    Health Plan Members        Admin Users      Agents      │
└──────────┬────────────────────┬──────────────────────┬───────────────┬───────┘
           │                    │                      │               │
           ▼                    ▼                      ▼               ▼
    ┌─────────────┐   ┌──────────────────┐  ┌──────────────────┐ ┌──────────┐
    │   airflow   │   │ chatbot-widget   │  │ admin-chatbot-   │ │   lap-   │
    │  (ETL DAGs) │   │ (React 19)       │  │ frontend         │ │ backend  │
    └──────┬──────┘   └────────┬─────────┘  │ (React 18)       │ │ (NestJS) │
           │                   │            └────────┬─────────┘ └─────┬────┘
           │                   │ WebSocket           │                  │ Socket.IO
           │                   ▼                     ▼                  │
           │         ┌──────────────────┐  ┌─────────────────┐         │
           │         │  chatbot-api or  │  │  admin-chatbot  │         │
           │         │ ohl-chatbot-api  │  │  (FastAPI)      │         │
           │         │ (FastAPI +       │◄─┤                 │         │
           │         │  LangGraph)      │  └─────────────────┘         │
           │         └────────┬─────────┘                              │
           │                  │ ← human handoff                         │
           │                  └───────────────────────────────────────► │
           │                                                    ┌───────┴──────┐
           │ POST /onboard                                      │    rails     │
           │ POST /ingest/claims                                │ (Ruby/Rails) │
           ▼                                                    └──────────────┘
    ┌─────────────────┐    Temporal Activities
    │   orch-eng      │─────────────────────────────►
    │  (FastAPI +     │                           ┌───────────────────────┐
    │   Temporal)     │                           │   messaging-worker    │
    └─────────────────┘                           │   (Temporal Worker)   │
                                                  └───────────┬───────────┘
                                                              │ direct service call
                                                              ▼
                                                  ┌───────────────────────┐
                                                  │    messaging-api      │
                                                  │    (FastAPI)          │
                                                  └───────────┬───────────┘
                                                              │
                                               ┌─────────────┴──────────────┐
                                               ▼                            ▼
                                           Twilio                       Mailgun
                                      (SMS / Voice)                    (Email)

    ┌─────────────┐
    │ ohl-frontend│◄──── ohl-backend ◄──── PostgreSQL + Temporal API + Airflow DB
    │ (React 19)  │      (FastAPI)
    └─────────────┘

    PostgreSQL (shared instance, schema-isolated)
    ┌──────────┬────────────┬────────────┬────────────┐
    │ public   │ messaging  │ admin_panel│   shared   │
    │(orch-eng)│ (msg API)  │  (admin)   │ (util)     │
    └──────────┴────────────┴────────────┴────────────┘
```

---

### Dependency Matrix

| Consumer → | `orch-eng` | `chatbot-api` | `ohl-chatbot-api` | `messaging-api` | `lap-backend-app` | `rails` | `messaging-*` libs |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `airflow` | **calls** | | | | | | |
| `orch-eng` | | | | **calls** | | | |
| `messaging-worker` | | | | **calls directly** | | | **uses** |
| `chatbot-api` | | | | **calls** | **WebSocket** | | |
| `ohl-chatbot-api` | **Service Bus** | | | **calls** | | | |
| `chatbot-widget` | | **WebSocket** | **WebSocket** | | | | |
| `admin-chatbot-frontend` | | | | | | | |
| `admin-chatbot` | | **configures** | **configures** | | | | |
| `lap-backend-app` | | **WebSocket** | | | | **webhook** | |
| `ohl-backend` | **queries DB** | | | | | | |
| `ohl-frontend` | | | | | | | |

---

## 6. Data Flows End-to-End

### Flow A: Member Medication Adherence Journey (Automated)

```
Day 0:   CVS pharmacy fills prescription → NCPDP claim record created

Day 1:   Airflow sftp_file_processing DAG runs (every 6h)
             → pulls CSV from CVS SFTP
             → transforms to internal schema
             → POST /api/v1/ingest/claims → orch-eng
             → Temporal: signals CategoryWorkflow with new claim

Day 1:   orch-eng rules engine evaluates: UC1 (Onboarding) eligible
             → UseCaseWorkflow[UC1] started
             → TCPA check: within 8am-6pm local time? ✓
             → schedule Temporal activity: send_message

Day 1:   messaging-worker picks up activity
             → direct call to messaging-api service layer
             → messaging-api encrypts PII, logs to DB, calls Twilio
             → member receives SMS: "Welcome to your medication program..."

Day 1:   Member replies → Twilio webhook → messaging-api inbound
             → published to Redis event bus
             → orch-eng CategoryWorkflow receives signal

Day 2:   Member opens chatbot link in SMS
             → chatbot-widget loads, WebSocket /chat connects
             → ohl-chatbot-api loads script from DB (medication_onboarding)
             → LangGraph state machine begins conversation
             → member completes onboarding assessment
             → barrier identified (cost concerns)
             → barrier education flow triggered
             → session ends, orch-eng UC1 workflow marked complete

Day 23:  Airflow ingests refill-due claim
             → orch-eng evaluates: UC2 (Refill Reminder) eligible
             → (cycle repeats with new use case)
```

### Flow B: Human Escalation

```
Member mid-conversation signals complexity beyond AI capability
    ↓ chatbot-api LAP node triggers
    ↓ WebSocket connection established to lap-backend-app
    ↓ lap-backend-app routes to available human agent
    ↓ agent sees full conversation context in LAP UI
    ↓ agent handles conversation with AI Copilot assistance
    ↓ rails persists all messages to conversation/message tables
    ↓ conversation concluded, transcript returned to chatbot flow
```

### Flow C: Prescription Renewal (UC5)

```
orch-eng detects: Rx expiry within 14 days, 0 refills remaining
    ↓ UC5 UseCaseWorkflow starts
    ↓ send SMS to member: "Your Rx is expiring..."
    ↓ member opens chatbot
    ↓ ohl-chatbot-api: prescription_renewal flow
    ↓ member confirms they want renewal request
    ↓ orch-eng calls SureScripts API: electronic renewal request to prescriber
    ↓ POST /api/v1/callbacks/surescripts callback when provider responds
    ↓ if approved: member notified, workflow complete
    ↓ if denied: Google Sheets task created for pharmacist follow-up
```

---

## 7. Infrastructure & DevOps

### Target Platform: Microsoft Azure

| Azure Service | Used By | Purpose |
|---|---|---|
| **Azure Container Apps** | All services | Serverless container hosting with built-in scaling |
| **Azure PostgreSQL Flexible Server** | All Python apps | Primary relational database |
| **Azure Container Registry** | All apps | Docker image storage |
| **Azure Key Vault** | All apps | Secrets management (API keys, connection strings) |
| **Azure Application Insights** | All apps | APM, distributed tracing, logs |
| **Azure Service Bus** | `ohl-chatbot-api` | Async message queuing to `orch-eng` |
| **Azure Blob Storage** | `lap-backend-app` | Chat file attachments |
| **Azure AD / Auth0** | `lap-backend-app` | Agent authentication (JWKS) |

### External Services

| Service | Used By | Purpose |
|---|---|---|
| **Twilio** | `messaging-api` | SMS, MMS, Voice calls |
| **Mailgun** | `messaging-api` | Transactional email |
| **Pinecone** | `chatbot-api`, `ohl-chatbot-api` | Vector embeddings for RAG |
| **OpenAI / Azure OpenAI** | `chatbot-api`, `nsr-service` | LLM inference |
| **Temporal Cloud** | `orch-eng`, `messaging-worker` | Durable workflow orchestration |
| **Redis** | `chatbot-api`, `lap-backend-app`, `messaging-events` | Caching, pub/sub, job queues |
| **SureScripts** | `orch-eng` | Electronic prescription renewal |
| **Arize Phoenix** | `ohl-chatbot-api` | LLM observability and tracing |
| **Sentry** | `lap-backend-app` | Error tracking |
| **Firebase** | `lap-backend-app` | Mobile push notifications |

### Developer Experience

- All apps have Nx targets: `serve`, `build`, `test`, `lint`
- `scripts/` directory contains shell scripts for starting/stopping the full platform locally
- Poetry lock files for reproducible Python environments
- Alembic migrations for every Python service with a database

---

## 8. What Makes This Platform Unique & Cutting-Edge

### 1. Agentic AI Meets Durable Workflow Orchestration

OHL is one of the few healthcare platforms that combines **LangGraph agentic AI** (for intelligent conversations) with **Temporal durable workflows** (for reliable long-running outreach journeys) in a production system. Most platforms choose one or the other. OHL uses LangGraph where intelligence and flexibility are needed (conversations), and Temporal where durability and reliability are paramount (weeks-long member journeys), with clean handoffs between the two paradigms.

### 2. Database-Driven Conversation Flows — Zero Deployment Changes

The entire chatbot logic — every question, every branch, every condition — lives in PostgreSQL, not code. Clinical program managers can redesign entire conversation flows through the admin panel and publish them instantly with zero deployments. This is an architectural decision that fundamentally changes the velocity of clinical program iteration.

### 3. Orchestrator Pattern for Interrupted Conversations

The `ohl-chatbot-api` Orchestrator Pattern (Feb 2026) solves a uniquely hard problem in healthcare conversations: members frequently go off-script (reporting symptoms, asking questions, expressing concerns). The Orchestrator tracks the script as a **checklist** rather than a linear state machine, enabling the AI to handle any interruption and seamlessly return to incomplete script items — something that traditional chatbot frameworks cannot do.

### 4. Compliance as Infrastructure, Not Afterthought

HIPAA and TCPA compliance are embedded at every layer:
- **AES-256-CBC encryption** of all PHI before database writes — the database never stores plaintext PII
- **PII masking** in all log output across the entire platform — developers never see real member data in logs
- **TCPA frequency caps** enforced at the workflow level — impossible to accidentally over-contact members
- **Consent tracking** with full audit history — opt-in/opt-out events immutably logged

### 5. True Polyglot Monorepo with Unified Tooling

Managing Python, TypeScript, and Ruby in a single Nx monorepo with shared libraries, unified build commands, and consistent CI/CD is a significant engineering achievement. The `@nxlv/python` plugin bridges the Nx ecosystem to Poetry-managed Python projects, enabling features like affected-build detection and task graph computation to work across all three languages simultaneously.

### 6. Hexagonal Architecture for the Messaging Domain

`libs/messaging-core` (zero-dependency pure domain) + `libs/messaging-db` (persistence port) + `libs/messaging-providers` (channel adapters) is a textbook hexagonal architecture implementation. This means:
- The entire messaging domain can be unit-tested without any network or database
- Swapping from Twilio to a different SMS provider requires changing only one adapter class
- The business rules (consent, frequency caps, channel selection) are completely isolated from infrastructure

### 7. Multi-Tier AI Escalation for Patient Safety

The three-tier medical escalation system (pharmacist → doctor → 911) in `ohl-chatbot-api` is a clinically significant safety feature. The AI continuously monitors conversation content for symptom keywords and severity indicators, automatically escalating to the appropriate level of human medical support. This is not a simple keyword match — it uses LLM-based intent classification to distinguish routine medication questions from genuine medical emergencies.

### 8. Full Observability Stack Across Languages

The `libs/telemetry` library provides **distributed tracing** across Python (FastAPI), Ruby (Rails/Sidekiq), and TypeScript (React/NestJS) — all unified into Azure Application Insights with proper trace context propagation. Combined with Arize Phoenix for LLM call tracing, the platform has end-to-end observability from the member's browser click through every LLM inference and database query, all the way to the Twilio delivery webhook.

### 9. Dual-Mode Messaging API

The `messaging-api` pattern of supporting both HTTP (with auth + rate limiting) and direct Python service calls (bypassing HTTP for internal workers) is an elegant optimization. Internal callers get zero-overhead, typed, in-process function calls to the same business logic that external callers access via REST. This avoids the classic microservices problem of internal HTTP overhead for trusted callers while maintaining a clean external API boundary.

### 10. CMS Star Ratings as a Product Core

Unlike generic "patient engagement" platforms, OHL is architected specifically around **CMS Star Ratings measures** (D08, D09, D10) — the actual financial mechanism through which health plans are reimbursed. Every workflow, every intervention timing, every escalation threshold maps to real CMS measurement criteria. This domain specificity means the platform doesn't just improve adherence metrics abstractly — it moves the specific needles that determine health plan revenue.

---

*This document reflects the state of the `ohl-platform` monorepo as of March 2026. The platform is under active development, with `ohl-chatbot-api`, `ohl-frontend`, and `ohl-backend` being the most rapidly evolving components.*
