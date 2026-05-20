# Chatbot Pipeline — Production Issues & Refactoring Plan

Full audit of the Ditto conversational AI system covering: pipeline architecture, silent failure paths, agent routing gaps, infrastructure issues, and refactoring plan.

---

## Pipeline Architecture

### End-to-End Flow

```
INBOUND
──────────────────────────────────────────────────────────────────
User SMS/iMessage
  → Provider (Photon HQ / SendBlue / Linq)
  → RabbitMQ
  → rmq.service.ts consume()
      ⚠ ACKs message BEFORE parsing (message loss on parse error)
  → @OnEvent("queue:sms-message")
  → sms-chat.service.ts handleWebhook()
      → Health check? → respond & exit
      → Qwen3Guard malicious check  ⚠ fails-open on timeout
      → Save message to MongoDB
      → Update chat denormalized fields  ⚠ not atomic
      → Fire-and-forget: acquire Redis lock (120s TTL)
           return 200 immediately
      ┌─── DRAIN LOOP (max 5 retries) ───────────────────────────┐
      │  _handleMessage()                                         │
      │    → Route by SmsChatState:                               │
      │        PendingSmsNumber / PendingSwitchToSms → verify     │
      │        IMessageNumber / SmsVerified → onboarded           │
      │    → _handleOnboardedStates()                             │
      │        → Route by nextIntent:                             │
      │            Login → _handleLoginIntent                     │
      │            YikYakReferral → _handleYikYakReferral         │
      │            Default → _handleAIChatIntent                  │
      │    → _handleAIChatIntent()                                │
      │        → Daily limit check (100/day, tz-aware)            │
      │          ⚠ timezone RPC call on EVERY message, no cache   │
      │        → pauseAIReply check                               │
      │        → chatbotService.handleAIChat()                    │
      │  Check for newer messages → loop or exit                  │
      └───────────────────────────────────────────────────────────┘

LANGGRAPH EXECUTION
──────────────────────────────────────────────────────────────────
chatbot.service.ts handleAIChat()
  ⚠ buildAgentGraph() + graph.compile() run on EVERY message

  ┌─── LANGGRAPH STATE GRAPH ────────────────────────────────────┐
  │  START                                                        │
  │    → loadProfileNode                                          │
  │        Load profile, status, school, timezone, pool info      │
  │    → loadMessagesNode                                         │
  │        Classify media URLs, load checkpoint, inject gap       │
  │        messages (team/automated), append incoming message     │
  │    → routeToAgent  (ONE-SHOT — not re-evaluated mid-loop)     │
  │        activePool == "yik-yak"       → yakAgent               │
  │        !completedOnboarding          → onboardingAgent        │
  │        status == "Matched"/matchId   → loadMatchContext       │
  │        status == "NeedMoreInfo"      → profileImprovementAgent│
  │        default                       → generalAgent           │
  │                                                               │
  │  {agent}_callModel                                            │
  │    → Malicious check (iteration 0 only) ⚠ duplicated call    │
  │    → Resolve dangling tool calls from checkpoint              │
  │    → Inject state reminder (SystemMessage, stripped after)    │
  │    → Invoke LLM with retry (429/5xx only, NOT timeouts)       │
  │    → shouldContinue?                                          │
  │        tool calls → {agent}_tools                             │
  │          → Execute via ToolNode (recreated per invocation)    │
  │          → Tools return Command{update, goto}                 │
  │          → afterTools → loop back (max 10 iterations)         │
  │        no tool calls → sendResponse                           │
  │                                                               │
  │  sendResponse                                                 │
  │    → Quiet-check: new user messages since invocation start?   │
  │        yes → suppress (sendSuppressed=true)                   │
  │        no  → extract text, split, send via smsService         │
  │              >3000 chars → pending response queue             │
  └───────────────────────────────────────────────────────────────┘

  Post-graph:
    → Set needsResponse=false
    → Stamp messages with LangSmith runId
    → PostHog capture for malicious blocks
    → Error recovery: SMS "hiccup" + handOverToTeam
      ⚠ Skipped entirely for system-triggered invocations

OUTBOUND
──────────────────────────────────────────────────────────────────
smsService.smsToChat()
  → Deactivation check
  → Route: phone or iMessage email
  → imessageService.sendMessage()
  → rmqService.sendToIMessage()  (RPC, 30s timeout, no retry)
  → Provider delivers to user
```

### Agent Roster

| Agent | Trigger condition | Tools available |
|-------|-------------------|-----------------|
| Onboarding | `!completedOnboarding` | getLoginLink, handOverToTeam, noReply, switchPool, updateUserProfile (conversational variant), sendEmailVerificationCode, verifyEmailCode |
| General | default | Full account, profile, pool tools + transferToYakAgent |
| Match | `status=Matched` or `matchId` set | Match-specific tools: getMatchInfo, markNotGoing, triggerContactExchange, getMatchScheduleLink |
| Profile Improvement | `status=NeedMoreInfo` or `InReview` | Profile update tools, pauseUserAccount |
| Yak | `activePool="yik-yak"` | Yak-specific: revealMatch, relayToMatch, getYakMatchInfo, convertYikYakToWednesday + transferToGeneralAgent |

### Key Design Strengths

- **Checkpoint integrity** — `resolveDanglingToolCalls()` and `requireToolCallId()` prevent checkpoint corruption from partial tool loops
- **Gap message injection** — watermark system ensures team/automated messages sent outside the graph are injected into the next invocation
- **Quiet-check + drain loop** — double-text prevention with drain loop retry
- **Layered guardrailing** — structural check → Qwen3Guard semantic check → regex fallback
- **State reminders** — per-invocation SystemMessage strips before checkpoint save to avoid token bloat

---

## Part 1: Silent AI Failure Paths

### Issue 1 (P0): Quiet-Check + Drain Loop Race Condition

**Symptom:** User sends 2 messages in quick succession. AI generates a response to message 1 but discards it. Message 2 may or may not get processed.

**Root cause:** The quiet-check in `sendResponse` (`chatbot-graph.ts:62`) and the drain loop in `sms-chat.service.ts:726` can desynchronize:

1. Message 1 arrives → webhook fires → acquires Redis lock → starts graph invocation
2. Message 2 arrives → webhook fires → Redis lock held → returns early (never enters drain loop)
3. Graph finishes processing message 1 → `sendResponse` finds message 2 → suppresses (`sendSuppressed: true`)
4. Drain loop detects message 2 → loops back and re-processes
5. If drain loop hits `MAX_DRAIN_RETRIES` (5), throws, or process crashes → message 2 is permanently lost

**Location:**
- `src/chatbot/chatbot-graph.ts:62-79`
- `src/sms-chat/sms-chat.service.ts:726-763`

**Proposed fix:**
- When `sendSuppressed` is true, persist `needsResponse: true` on the chat document
- Add a startup sweep (`onModuleInit`) and periodic cron that finds chats with `needsResponse: true` and no held Redis lock, then re-triggers the graph
- Shared mechanism with Issue 2 below

---

### Issue 2 (P0): Orphaned Messages on Process Crash

**Symptom:** Backend crash during graph execution → message is saved to DB but no response is ever sent.

**Root cause:** The entire AI pipeline runs as a fire-and-forget `Promise`. No persistent job queue, no startup recovery, no dead-letter mechanism. Redis lock TTL expires, unblocking future messages, but the orphaned message is never reprocessed.

**Location:** `src/sms-chat/sms-chat.service.ts:693-772` — no `onModuleInit` recovery exists anywhere

**Proposed fix:**
- Set `needsResponse: true` on the chat document BEFORE starting AI processing
- Clear it AFTER `sendResponse` succeeds or error recovery completes
- On `onModuleInit`, sweep for `needsResponse: true` + `pauseAIReply: false` chats and re-trigger — same mechanism as Issue 1

---

### Issue 3 (P1): System-Triggered Failures Leave State Dirty

**Symptom:** Graph triggered by a system event (profile review, match notification) fails — state flags are never cleaned up, future messages may behave unexpectedly.

**Root cause:** `chatbot.service.ts:380-405` skips all recovery for system-triggered runs:

```typescript
if (!systemContext) {
  // recovery SMS + handOverToTeam
}
// nothing runs for system-triggered failures
```

Correct to skip the user-facing SMS, but any dirty state (checkpoint, `needsResponse`) is left behind.

**Location:** `src/chatbot/chatbot.service.ts:380-405`

**Proposed fix:**
- Add a `finally` block that cleans up state flags regardless of trigger type
- For system-triggered failures: log a structured alert, clean up state, do not send recovery SMS

---

### Issue 4 (P1): TimeoutError Mid-Tool-Loop Leaves Checkpoint Corrupted

**Symptom:** User receives a "hiccup" error message after the agent already executed tools — confusing partial state.

**Root cause:** `llm-retry.ts` intentionally excludes `TimeoutError` from retries (risk of duplicate side-effects). When a timeout occurs mid-tool-loop (`toolCallIterations > 0`):

1. Tools already executed and checkpoint saved partial state
2. LLM call times out → propagates to error recovery
3. Recovery sends "hiccup" SMS and calls `handOverToTeam`
4. Checkpoint now has tool calls without responses
5. Next invocation patches it via `resolveDanglingToolCalls` — but user already got the error message

**Location:**
- `src/chatbot/llm-retry.ts:77-78`
- `src/chatbot/chatbot.service.ts:380-405`
- `src/chatbot/agent-node.ts:33-87`

**Proposed fix:**
- Differentiate recovery by `toolCallIterations`:
  - `== 0`: simple timeout, "hiccup" is appropriate
  - `> 0`: partial execution — suppress recovery SMS, let `resolveDanglingToolCalls` handle it on next user message

---

### Issue 5 (P2): Graph Rebuilt Per Message

**Symptom:** Unnecessary latency on every message — full graph construction and compilation happens even though the graph structure never changes.

**Root cause:** `chatbot.service.ts:244-267` calls `buildAgentGraph()` + `graph.compile()` on every invocation. Graph topology is static; only state changes.

**Location:** `src/chatbot/chatbot.service.ts:244-267`

**Proposed fix:**
- Build and compile the graph once in `onModuleInit`, reuse the compiled instance
- Per-invocation dependencies (e.g., `chat` document) passed via state or LangGraph config rather than closure

---

### Issue 6 (P2): Duplicated Malicious Content Check

**Symptom:** Qwen3Guard API called twice per message — once before DB save, once inside the graph.

**Root cause:** Two independent calls serve overlapping purposes:
- `sms-chat.service.ts:621-631` — sets `isMalicious=true` on DB document, triggers `handOverToTeam` for sensitive categories
- `agent-node.ts:167` — masks message content, blocks AI response

**Location:**
- `src/sms-chat/sms-chat.service.ts:621-631`
- `src/chatbot/agent-node.ts:166-196`

**Proposed fix:**
- Run the check once in `sms-chat.service.ts`, pass result into graph state (`isMalicious`, `maliciousCategories`)
- Agent-node reads `state.isMalicious` instead of calling the API again
- Saves one Replicate API call per message and eliminates divergence risk

---

### Issue 7 (P2): RabbitMQ ACKs Before Parsing

**Symptom:** A malformed RabbitMQ payload (e.g., invalid JSON) causes the message to be silently dropped — permanently lost with no retry.

**Root cause:** `rmq.service.ts:117` ACKs the message before attempting to parse it. The existing TODO comment in the file acknowledges this:

```typescript
// TODO: do 3 together to rely on queue for reliability
// 1. queue handler  2. do not process through event.  3. ack at the end
this.chan.ack(msg);  // ← ACK happens HERE
try {
  const payload = JSON.parse(msg.content.toString());
  // ...
} catch {
  this.logger.warn("Failed to deserialize queue payload");
  // message already acked — gone forever
}
```

**Location:** `src/rmq/rmq.service.ts:113-132`

**Proposed fix:**
- Move `chan.ack(msg)` to after successful processing
- Use `chan.nack(msg, false, true)` on parse error to requeue

---

### Priority Summary

| # | Issue | Priority | Impact | Effort |
|---|-------|----------|--------|--------|
| 1 | Quiet-check + drain loop race condition | **P0** | Messages silently lost | Medium |
| 2 | Orphaned messages on process crash | **P0** | Messages permanently lost | Medium |
| 3 | System-triggered failures dirty state | **P1** | State stuck, future messages affected | Low |
| 4 | TimeoutError mid-tool-loop confusion | **P1** | Confusing partial responses | Medium |
| 5 | Graph rebuilt per message | **P2** | Unnecessary latency | Low |
| 6 | Duplicated malicious check | **P2** | Wasted API call, divergence risk | Low |
| 7 | RMQ ACK before parse | **P0** | Silent message loss on bad payload | Low |

---

## Part 2: Agent Routing & Pool Autonomy Gaps

The root problem across all gaps in this section: **routing is a one-shot decision at graph start, but tool executions can mutate state mid-invocation in ways that should change the agent**. There is no mechanism to re-route after a state change within the same graph run.

### Gap 1 (HIGH): `switchPool` Does Not Update `activePool` in Graph State

When a user switches pools via the `switchPool` tool, the database is updated but `state.activePool` — which drives `routeToAgent()` — remains stale for the rest of the invocation.

**Location:** `src/chatbot/chatbot-tools.ts:959-966`

```typescript
return new Command({
  update: {
    ...(result.userStatus ? { currentStatus: result.userStatus } : {}),
    messages: [...],
    // ❌ Missing: activePool: targetPool
  },
});
```

**Impact:** Agent continues operating with the old pool context. The pool-specific prompt, state reminder, and tool set are all wrong for the remainder of the conversation turn. Corrects itself on the next message when `loadProfileNode` re-fetches from DB.

**Proposed fix:** Add `activePool: targetPool` to the Command update. If routing should change immediately (e.g., switching into yik-yak), also add `goto: "yakAgent_callModel"`.

---

### Gap 2 (HIGH): `convertYikYakToWednesday` Does Not Update `activePool`

Same pattern as Gap 1. After a yak user successfully converts to Wednesday, `state.activePool` still reads `"yik-yak"`. The yak agent continues handling the rest of the conversation with yak-specific tools and context.

**Location:** `src/chatbot/chatbot-tools.ts:1204-1209`

**Proposed fix:** Add `activePool: undefined` (or `"wednesday"`) to the Command update, and `goto: "generalAgent_callModel"` to immediately hand off to the correct agent.

---

### Gap 3 (HIGH): No Mid-Invocation Agent Re-Routing

`routeToAgent()` runs exactly once per graph invocation, immediately after data loading. Any tool that changes the routing-relevant state (`activePool`, `currentStatus`, `completedOnboarding`) does not trigger re-routing within the same invocation.

**Location:** `src/chatbot/chatbot-graph.ts:185-206`

Affected scenarios:
- Pool switch → agent stays in old pool's agent
- Status change (Waiting → Matched) → agent stays in general agent, not match agent
- Profile improvement resolved → agent stays in profile improvement agent

**Transfer tools only partially mitigate this:**
- `transferToYakAgent` and `transferToGeneralAgent` exist and work — but only cover the general↔yak pair
- No transfer tools exist for onboarding, match, or profile improvement agents
- The LLM must decide to call them — the system does not enforce re-routing after state changes

**Two approaches to fix:**

*Option A — Explicit per-tool goto:* When a state-changing tool succeeds, include `goto: "<correct_agent>_callModel"` in the Command. Requires every tool author to handle this correctly.

*Option B — `reRouteAgent` checkpoint node:* Insert a node between `afterTools` and `callModel` that re-runs `routeToAgent()` against the current (post-tool) state. If the result differs from the current agent, jump to the new agent's `callModel`. This is structural and catches all state changes automatically.

Option B is more robust and future-proof.

---

### Gap 4 (MEDIUM): Event Pools Have No Dedicated Agents

`routeToAgent()` only has a dedicated branch for `"yik-yak"`. All other event pools route to the general agent:

```typescript
if (state.activePool === "yik-yak") return "yakAgent_callModel";
// nyc-gala → generalAgent
// la-love-yacht → generalAgent
```

The general agent receives a conditionally-added `getNycGalaFAQ` tool when `activePool === "nyc-gala"`, but the agent prompt is still the general Wednesday prompt. There is no structural guarantee the agent will prioritize event-specific behavior. NYC Gala and La Love Yacht users get generic Wednesday matching context.

**Location:** `src/chatbot/chatbot-graph.ts:193-195`, `src/chatbot/chatbot-tools.ts:1328`

**Proposed fix:** Either add dedicated agent setups for active event pools (similar to yak agent), or enrich the general agent prompt with pool-specific instructions loaded from the prompt service.

---

### Gap 5 (MEDIUM): Onboarding Variant Locked at Agent Setup

The onboarding variant (conversational vs. form) is determined once when the agent node is set up:

```typescript
// src/chatbot/agents/onboarding-agent.ts:17-19
const isConversational =
  deps.onboardingVariant === "conversational" &&
  (!state.activePool || state.activePool === "wednesday");
```

If an onboarding user switches pools mid-conversation (e.g., wednesday → nyc-gala):
1. `activePool` is not updated in state (Gap 1)
2. Even if it were, the variant was already chosen — the agent setup does not re-evaluate
3. User stays in conversational onboarding with wednesday-pool tools while now in an event pool

**Location:** `src/chatbot/agents/onboarding-agent.ts:17-19`

---

### Gap 6 (MEDIUM): Transfer Tools Are Asymmetric — Only Cover General ↔ Yak

| Agent | Can transfer to |
|-------|----------------|
| General | Yak (`transferToYakAgent`) |
| Yak | General (`transferToGeneralAgent`) |
| Onboarding | Nobody |
| Match | Nobody |
| Profile Improvement | Nobody |

If a matched user's match is cancelled during conversation (status reverts to Waiting), the match agent continues handling the conversation with match-specific tools until the next message. No way to self-correct within the same turn.

**Location:** `src/chatbot/chatbot-tools.ts:1220-1266`

---

### Gap 7 (LOW): Inactive Pools Referenced Inconsistently

`switchPool` blocks switching to `"la-love-yacht"` (listed as inactive):

```typescript
// src/chatbot/chatbot-tools.ts:949
const INACTIVE_POOLS = ["yik-yak", "la-love-yacht"];
```

But `data-loader.ts` still loads `isInLaLoveYachtPool` and `loveYachtPlacement` for existing users in that pool. Users already in the pool are handled correctly, but the messaging ("pool is closed") conflicts with the system still recognizing their pool membership for tool availability.

**Location:** `src/chatbot/chatbot-tools.ts:948-949`, `src/chatbot/agents/data-loader.ts`

**Note:** Also hardcodes inactive pool names — the existing TODO suggests fetching from DB instead.

---

## Part 3: Code Quality & Infrastructure Issues

### Critical

**RMQ ACK before parse** — covered in Part 1, Issue 7.

**Guardrail fails-open silently**

If Qwen3Guard times out or throws, the check silently treats the message as safe:

```typescript
// src/sms-chat/sms-chat.service.ts:629-631
} catch (err) {
  this.logger.warn("runMaliciousContentCheck failed unexpectedly, treating as safe", err);
  // malicious content may pass through
}
```

A safety check failure should at minimum be a structured error-level log with context (userId, chatId). Consider failing-closed for sensitive use cases.

**Hardcoded Yik Yak API token**

```typescript
// src/sms-chat/sms-chat.service.ts:133
authorization: "Bearer yy_live_zCLDQlaGcG8nrbQ7sdlTR4tOGQyymDHQ"  // FIXME: hardcoded
```

Live production credential committed in source. Move to `ConfigService` immediately.

**No MongoDB transactions**

Multiple writes that should be atomic are split across separate operations:

```typescript
// Message saved (sms-chat.service.ts:635-642)
const msg = await this.smsMessageModel.create({...});

// Chat denormalized fields updated separately (sms-chat.service.ts:671-680)
await this.smsChatModel.updateOne({...});
```

If the second write fails, the message exists but the chat's last-message state is stale. No `session()` or `withTransaction()` usage anywhere in the pipeline.

---

### High

**Timezone lookup on every AI message**

`getSchoolBasicInfo()` RPC call fires for every message to get the school timezone for daily-limit calculation. For a user at 100 messages/day this is 100 redundant lookups.

**Location:** `src/sms-chat/sms-chat.service.ts:1206-1233`

**Fix:** Cache timezone per userId in Redis with a 24-hour TTL.

**Duplicate Qwen3Guard client instances**

Both `chatbot.service.ts` and `sms-chat.service.ts` initialize their own `Qwen3GuardClient` from the same config. The client is stateless. Extract to a shared NestJS provider.

**Location:** `src/chatbot/chatbot.service.ts:113`, `src/sms-chat/sms-chat.service.ts:127`

**No graph execution timeout**

`handleAIChat()` awaits `graph.invoke()` with no explicit timeout. If the graph hangs, the only recovery is the 120s Redis lock TTL expiring. A hung graph holds the lock for the full TTL, blocking all subsequent messages from that user.

**Fix:** Wrap `graph.invoke()` in a `Promise.race` with a 60-second timeout that triggers graceful recovery.

---

### Medium

**Duplicate `_toStringSafe()` utility**

Identical function exists in two files with no shared import:

- `src/llm-tools/llm-tools.service.ts:1986`
- `src/chatbot/agents/data-loader.ts:242`

Extract to `src/chatbot/utils/to-string-safe.ts`.

**Inconsistent tool result shapes**

`LlmToolsService` methods return different object shapes with no shared interface:

```typescript
pauseUserAccount()    → { status, message, userStatus? }
updateUserProfile()   → { status, message }
getMatchInfo()        → { status, message, data? }
markNotGoing()        → { status, message, userStatus, matchStatus }
```

Tool implementations must defensively check which fields are present. Define a shared `ToolResult<T>` type.

**Profile summary is stringly-typed**

`profileSummary` is stored in state as `"User Profile Summary: <JSON string>"`. Three separate parse-patch-stringify operations exist in the codebase (`patchProfileSummary`, `unsetFromProfileSummary`, `buildConversationalOnboardingReminderContent`). A format change breaks all three silently.

**Location:** `src/chatbot/chatbot-tools.ts:82-121`, `src/chatbot/state-reminder.ts`

**Fix:** Store `profileData` as a typed object in state alongside the serialized `profileSummary` string. Patch the object; serialize only for LLM injection.

**`console.log` in production RMQ service**

```typescript
// src/rmq/rmq.service.ts:122-127
console.log(`[RMQ Service] Received event ${payload.event}...`);
```

Critical queue events bypass the Winston/Loki structured logger. Replace with `this.logger`.

**Onboarding variant hardcoded**

```typescript
// src/chatbot/chatbot.service.ts:245
const onboardingVariant = "conversational" as const;  // TODO: Replace with PostHog A/B test
```

The "control" (form-based) variant code path exists but is unreachable. Either implement the A/B test or remove the dead path.

---

### Low

**`llm-tools.service.ts` has 19 injected dependencies**

The service handles account management, profile data, image uploads, match operations, and scheduling — all in one class. Any change ripples through a large constructor. Break into focused sub-services: `AccountManagementService`, `ProfileDataService`, `ImageManagementService`, `MatchDataService`.

**Deprecated profile APIs still in use**

`profile-client.service.ts` has two `@deprecated` methods (`updateSelfProfile`, `updateExpectedProfile`) that are still called from `llm-tools.service.ts`. Migrate to `updateBasicInfo` / `updateDeepInfo`.

**Dead graph edges**

The graph defines conditional edges from `generalAgent_tools` → `yakAgent_callModel` and from `yakAgent_tools` → `generalAgent_callModel`, but `afterTools` never returns the cross-agent node name — it always returns the same agent. These edges are unreachable. Agent transfers happen via `Command{ goto }` in tools, not via `afterTools` routing. Remove the dead edges or document the intended use.

**Location:** `src/chatbot/chatbot-graph.ts:330-334`, `src/chatbot/chatbot-graph.ts:375-379`

---

## Part 4: God Class Refactoring Plan

### `sms-chat.service.ts` (~1400 lines)

Handles: webhook processing, drain loop, onboarding state machine, AI delegation, rate limiting, automated messaging, health checks, admin CRUD.

**Proposed split:**

#### 1. `sms-chat-orchestrator.service.ts` (~300 lines)
Webhook entry, drain loop, Redis lock, message routing.
- `handleWebhook()`, drain loop, `_handleMessage()`
- Malicious content check (pre-save), health check intercept
- Depends on: SmsChatQueryService, SmsChatOnboardService, SmsChatAIService

#### 2. `sms-chat-query.service.ts` (~250 lines)
All read operations and message CRUD.
- `listPaginated()`, `getLastMessage()`, message save/delete/edit
- Search integration (user, content, phone modes)
- Extract `buildBaseFilter()` helper — the 3 search branches duplicate filter assembly

#### 3. `sms-chat-onboard.service.ts` (~400 lines)
Full onboarding state machine.
- New chat creation, signup code matching, SMS number verification
- Welcome/migration messages, contact card sending, `_sendOnboardMessages()`

#### 4. `sms-chat-ai.service.ts` (~200 lines)
AI-specific delegation.
- `_handleAIChatIntent()` — pause check, daily limit
- `_handleAIChat()` — user validation, chatbot invocation
- `_handleLoginIntent()`, rate limiting logic

**Migration order:**
1. Query service first — most self-contained, no state mutations
2. AI service next — small surface, enables P0 fixes in isolation
3. Onboard service — largest chunk, requires state machine testing
4. Orchestrator slims to routing and coordination

Each extraction = separate PR with before/after integration tests.

---

### `chatbot-tools.ts` (~1405 lines)

Single `createTools()` function defines 70+ tools for all agents. Adding a new agent requires modifying 4+ tool arrays. `userId` extraction pattern is repeated 8+ times.

**Proposed split by domain:**

| File | Contents |
|------|----------|
| `tools/account-tools.ts` | pauseUserAccount, resumeUserAccount, requestAccountDeactivation, confirmAccountDeactivation |
| `tools/profile-tools.ts` | updateUserProfile, updateUserProfileImage, getProfileUpdateInfo, unsetUserProfileFields |
| `tools/match-tools.ts` | getMatchInfo, getMatchPosterLink, markNotGoing, triggerContactExchange, getMatchScheduleLink |
| `tools/yak-tools.ts` | revealMatch, relayToMatch, getYakMatchInfo, convertYikYakToWednesday |
| `tools/utility-tools.ts` | handOverToTeam, noReply, getLoginLink, switchPool, getDittoSocialMediaHandles |
| `tools/transfer-tools.ts` | transferToYakAgent, transferToGeneralAgent |

Extract `resolveUserId(state, chat)` helper to eliminate the repeated pattern.

---

## Appendix: File Reference

| File | Lines | Role |
|------|-------|------|
| `src/sms-chat/sms-chat.service.ts` | ~1400 | SMS webhook, drain loop, onboarding state machine, AI delegation, CRUD |
| `src/chatbot/chatbot.service.ts` | ~525 | Graph build + invoke, error recovery, post-graph cleanup |
| `src/chatbot/chatbot-graph.ts` | ~386 | StateGraph definition, sendResponse quiet-check, routeToAgent |
| `src/chatbot/agent-node.ts` | ~295 | Generic agent builder: callModel, shouldContinue, afterTools, malicious check |
| `src/chatbot/chatbot-tools.ts` | ~1405 | All tool definitions (70+) and per-agent tool sets |
| `src/chatbot/chatbot-tools-schema.ts` | ~220 | Zod schemas for tool parameters |
| `src/chatbot/chatbot-state.ts` | ~248 | 40+ LangGraph state fields via Annotation API |
| `src/chatbot/state-reminder.ts` | ~220 | State-of-truth SystemMessage builder per agent |
| `src/chatbot/llm-retry.ts` | ~116 | LLM retry: exponential backoff, TimeoutError excluded |
| `src/chatbot/chatbot.middleware.ts` | ~300 | Malicious content check: structural + Qwen3Guard + regex |
| `src/chatbot/agents/data-loader.ts` | ~579 | Profile + message loading, gap injection, media URL validation |
| `src/chatbot/agents/general-agent.ts` | ~86 | General agent setup (all 5 agents follow same pattern) |
| `src/chatbot/agents/onboarding-agent.ts` | ~100 | Onboarding agent: conversational vs form variant |
| `src/chatbot/agents/match-agent.ts` | ~200 | Match agent + loadMatchContext node |
| `src/chatbot/agents/yak-agent.ts` | ~96 | Yak agent with yak-specific tools |
| `src/chatbot/agents/profile-improvement-agent.ts` | ~110 | Profile improvement agent for NeedMoreInfo/InReview status |
| `src/llm-tools/llm-tools.service.ts` | ~2000 | All tool implementations (19 deps, god service) |
| `src/sms/sms.service.ts` | ~140 | SMS send routing: phone vs iMessage email |
| `src/imessage/imessage.service.ts` | ~120 | RabbitMQ RPC to iMessage provider |
| `src/rmq/rmq.service.ts` | ~430 | RabbitMQ consumer, producer, RPC client |
| `src/profile-client/profile-client.service.ts` | ~250 | Profile read/write abstraction, underage status check |
| `src/utils/dayjs-init.util.ts` | ~13 | dayjs plugin registration (utc, tz, isLeapYear, customParseFormat) |

---

## Production Issues Log (2026-04-09 — 2026-04-10)

Traced from LangSmith production traces. Each entry includes the root cause, impact, and resolution status.

### ISSUE-001: Roast generator fails with "URL sources are not supported" (Anthropic)

- **Trace:** `368a94d8-aeb8-48d7-9a3d-2f00bb53ab6f` (roast_generator tool)
- **Date:** 2026-04-09
- **Error:** `400 messages.0.content.9.image.source.base64.data: URL sources are not supported`
- **Root cause:** `generateWelcomeMessage` in `profile-review.service.ts` passed raw GCS image URLs as `image_url` blocks. The Vercel AI Gateway converted these to Anthropic's format, placing the URL string into the `base64.data` field instead of using the `url` source type. Anthropic's API rejected it.
- **Impact:** Users never received their onboarding welcome/roast message.
- **Fix:** ENG-1559 / PR #1090 — Convert image URLs to inline base64 data URLs before sending. Added Content-Type sanitization, 10s fetch timeout, 10MB size cap, and post-download size check.
- **Status:** Fixed

### ISSUE-002: LangGraph hangs 30+ minutes when AI Gateway stalls

- **Trace:** `368a94d8-aeb8-48d7-9a3d-2f00bb53ab6f` (onboardingAgent_callModel pending)
- **Date:** 2026-04-09
- **Error:** No error — graph hung indefinitely with `status: pending`
- **Root cause:** The Vercel AI Gateway accepted the TCP connection but never returned a response body. `ChatOpenAI` timeout (60s) only covers connection timeout, not stalled responses. `invokeLlmWithRetry` only retries on thrown errors. `compiledGraph.invoke()` had no overall deadline.
- **Impact:** User never received a response. Graph invocation blocked indefinitely, consuming server resources.
- **Fix:** ENG-1559 / PR #1090 — Added 5-minute graph-level `AbortController` (signal passed to `compiledGraph.invoke`), 90s per-LLM-call `AbortController` in `agent-node.ts`, and `classifyGraphError` patterns for abort errors.
- **Status:** Fixed. Confirmed working in trace `49c842f5` — graph correctly aborted at exactly 5 minutes.

### ISSUE-003: Image MIME type mismatch (WebP declared, JPEG actual)

- **Trace:** `595a5a99-8e5d-45ee-ab9e-4fdc9609c823`
- **Date:** 2026-04-09
- **Error:** `400 messages.43.content.1.image.source.base64: The image was specified using the image/webp media type, but the image appears to be a image/jpeg image`
- **Root cause:** User's image URL had a `.webp` extension or content-type header, but the actual image bytes were JPEG. Anthropic validates MIME type matches actual content. The `IMAGE_ERROR_PATTERNS` in `error-classifier.ts` didn't match this error, so the image-fallback retry (`stripImageParts`) never fired. Error was classified as "blocking" instead of "retryable".
- **Impact:** Immediate team handover for a self-healing error.
- **Fix:** ENG-1559 / PR #1090 — Broadened `IMAGE_ERROR_PATTERNS` to `/image/i` (safe because `isImageRelated400` gates on `status === 400`). Created separate `IMAGE_GRAPH_ERROR_PATTERNS` with specific patterns for `classifyGraphError` (no status gate).
- **Status:** Fixed

### ISSUE-004: MongoDB connection lost during graph execution

- **Trace:** `481c0997-d7aa-48f3-a36c-7e1101d47c54`
- **Date:** 2026-04-10
- **Error:** `Client must be connected before running operations`
- **Failed at:** `loadProfileNode`
- **Root cause:** Transient MongoDB connection loss. The MongoDB client was disconnected when the Mongoose query ran in `loadProfileNode`.
- **Impact:** User gets "try again" SMS. Next message succeeds.
- **Fix:** Already classified as "retryable" in `error-classifier.ts`. No code change needed — infrastructure-level transient error.
- **Status:** Known/accepted (transient)

### ISSUE-005: LLM returns tool_calls but graph routes to sendResponse

- **Trace:** `32fb9bf1-db34-4609-82a8-437e7d1d97f8`
- **Date:** 2026-04-10
- **Error:** `[sendResponse] Received invalid response content from LLM (contentType=string)`
- **Failed at:** `sendResponse` — last message was an AIMessage with `tool_calls`, not text
- **Root cause:** `onboardingAgent_callModel` returned an AIMessage with `tool_calls` (wanting to call `updateUserProfile`), but `toolsCondition(state)` did not detect the tool_calls on the last message in state. Suspected `MessagesAnnotation` reducer edge case where message ordering or ID merging didn't place the AI message at the end.
- **Impact:** User gets "try again" SMS. Profile update lost for that turn.
- **Fix:** Pre-existing intermittent issue. Already classified as "retryable". Needs deeper investigation into `MessagesAnnotation` reducer behavior.
- **Status:** Open — needs investigation

### ISSUE-006: Image classification fails — Gemini can't fetch GCS URL

- **Trace:** `5b03559d-9aa1-49f8-b33b-e0a86ae244ee`
- **Date:** 2026-04-09
- **Error:** `400 Cannot fetch content from the provided URL.`
- **Failed at:** `ChatOpenAI` inside `generateImageDescriptions` (image-classification.service.ts)
- **Root cause:** The image classification service passes GCS image URLs as `image_url` blocks to Gemini Flash Lite. Gemini tried to fetch the URL but couldn't — likely a private/signed GCS URL that Gemini's servers can't access, or the URL expired. Same class of issue as ISSUE-001 but in a different service.
- **Impact:** Image descriptions/tags not generated for that user's photos. Silent failure — service catches error and returns.
- **Fix:** Not covered by PR #1090 (only covers roast generator and chatbot graph). The image classification service needs the same base64 conversion or accessible URLs.
- **Status:** Open

### ISSUE-007: Match agent loops getMatchInfo 9 times — status/data mismatch

- **Trace:** `979cdede-6a92-4fa3-b4a5-13a995a219d6`
- **Date:** 2026-04-10
- **Error:** `[sendResponse] Received invalid response content from LLM (contentType=string)` (after 9 tool-call iterations)
- **Root cause:** User's `matchingStatus` says "Matched" (so `routeToAgent` sends them to `matchAgent`), but the actual match document no longer exists in the database. `getMatchInfo` returns "Error: No active match found" on every call. The LLM has no way to handle this contradiction — it's told the user is matched and has a `getMatchInfo` tool, so it keeps calling it. After `MAX_TOOL_ITERATIONS` (10), `onMaxIterations` fires (handover to team), but the graph still tries `sendResponse` with the last AIMessage which contains tool_calls, not text — causing the crash.
- **Impact:** User gets "team will help" SMS after ~30s of wasted LLM calls. 9 unnecessary API calls.
- **Fix needed:** (1) `getMatchInfo` or the match agent prompt should handle the "no active match" case gracefully — e.g. after 1-2 failed attempts, the LLM should tell the user their match status is being updated. (2) `loadMatchContext` should detect the mismatch and update the user's status or reroute to a different agent. (3) `sendResponse` should handle the edge case where the last message has tool_calls (after max iterations) — either extract partial text or send a fallback message.
- **Status:** Open

### ISSUE-008: Graph-level abort confirmed working (5-minute timeout)

- **Trace:** `49c842f5-e293-48e5-b9f1-6d89ead65c2b`
- **Date:** 2026-04-10
- **Error:** `Abort` (from LangGraph Pregel runner)
- **Root cause:** AI Gateway stalled. The 5-minute `GRAPH_TIMEOUT_MS` AbortController fired correctly. LangGraph threw `Error("Abort")`. Duration: exactly 5 minutes (`05:22:48` → `05:27:48`).
- **Impact:** User gets recovery SMS after 5 minutes instead of hanging for 30+ minutes.
- **Fix:** ENG-1559 / PR #1090 — this trace confirms the fix is working as designed.
- **Status:** Mitigated (timeout working; underlying gateway stall is external)

### ISSUE-009: Per-call 90s abort fires on stalled LLM call (onboarding)

- **Trace:** `bc36b488-4bf0-47fb-946d-fd9940ad283c`
- **Date:** 2026-04-10
- **Error:** `AbortError: Request was aborted.`
- **Failed at:** `onboardingAgent_callModel` — ran ~99 seconds before abort fired
- **Root cause:** AI Gateway stalled on the LLM call. The 90s per-call `AbortController` (`LLM_CALL_TIMEOUT_MS`) fired and aborted the request. This is the per-call timeout (added in PR #1090) working as designed.
- **Impact:** User gets recovery SMS after ~100s instead of hanging indefinitely.
- **Status:** Mitigated (same class as ISSUE-008, at LLM-call level)

### ISSUE-010: Match agent getMatchInfo loop (repeat occurrence)

- **Trace:** `003f6fb5-edf7-45b2-88c9-86ed945f8d48`
- **Date:** 2026-04-10
- **Chat:** `69bd6bb2e5f789eda03e88f5` (same user as ISSUE-007)
- **Error:** `[sendResponse] Received invalid response content from LLM (contentType=string)`
- **Root cause:** Identical to ISSUE-007. Same user, same bug — `matchAgent` loops `getMatchInfo` 9 times, each returning "Error: No active match found". User's status still says "Matched" but no match document exists. This user is stuck in a broken state and will hit this every time they message.
- **Impact:** Repeated handover to team. User cannot use the chatbot until the status/data mismatch is resolved manually.
- **Status:** Open (ISSUE-007) — needs fix in `loadMatchContext` or `routeToAgent` to detect and handle the mismatch

### ISSUE-011: Per-call 90s abort fires on standalone ChatOpenAI call

- **Trace:** `f45f74e9-6fc9-46ee-883f-8440a20df5ad`
- **Date:** 2026-04-10
- **Error:** `AbortError: Request was aborted.`
- **Failed at:** `ChatOpenAI` — standalone LLM call (not inside LangGraph node), ran ~104 seconds
- **Root cause:** AI Gateway stalled. Per-call abort fired. Same class as ISSUE-009. The root trace is a `ChatOpenAI` run (not `LangGraph`), suggesting this is from a tool or sub-invocation within the graph that also has the abort signal propagated.
- **Impact:** Error propagates up to graph; user gets recovery SMS.
- **Status:** Mitigated (our fix working)
