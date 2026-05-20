# Outbound Message Latency — Full Root Cause Analysis & Fix Plan

## The Problem

Brand new users experience 3-5 minute delays between sending their first iMessage and receiving any response from Ditto. The user sends a message, waits over a minute with no response, then receives generic onboarding messages — not a reply to what they said. The actual AI response arrives 4-7 minutes later.

This affects users on multiple lines (+14154898138, +14154260535, +14154349815) and is likely systemic for all new user signups under load.

### What the user experiences

Real example: user +15627510474 on line +14154349815, 2026-04-13 at 23:25 UTC.

```
23:25:50  User sends: "hey ditto, i'm ready to go on a date. can you set me up? my code is ssu_TPDCXYUJ"
23:26:07  User sends: "Help me"
23:26:09  User sends: "With homework"
23:26:24  User sends: "Not love"
23:26:27  User sends: "Need school first"
23:26:41  User sends: "Answer me or I hate you"
23:26:51  User sends: "R u with another lover"
23:26:53  User sends: "Don't cheat"

          ─── 1 min 46 sec silence ───

23:27:36  Bot [automated]: "yo it's ditto, a dating initiative built by college students who get it"
23:28:29  Bot [automated]: "no swiping, no small talk, i will plan you a fun date every wednesday 7pm"
23:29:11  Bot [automated]: "let's get you set up real quick. what's your name?"

          ─── 1 min 45 sec gap ───

23:30:56  Bot [assistant]: "whoa whoa slow down bro 😭 i can set you up on a date but i'm not doing your homework"
23:32:57  Bot [assistant]: "first things first, what's your name?"
23:34:02  Bot [assistant]: "lmaooo not you friendzoning me before we even start 😭 school first, love later..."
          ... 5 more bot messages between 23:34:13 and 23:34:39
```

The user sent 8 messages in 63 seconds with zero response. The first bot reply came 1 minute 46 seconds later. The chatbot's actual response to what they said came 5 minutes after that.

---

## How the System Works

```
User's iPhone
    │
    ▼
Linq (Apple Mac device) ──webhook──► imsg-service (Bun microservice)
                                          │
                                     RMQ "be" queue
                                          │
                                          ▼
                                     proj-coach-backend (NestJS)
                                          │
                                     LangGraph chatbot (~4s)
                                          │
                                     RMQ "imsgs" queue (fire-and-forget)
                                          │
                                          ▼
                                     imsg-service
                                          │
                                     ┌─────────────────────────────────┐
                                     │  Per-line rate limiter          │
                                     │  Bottleneck: maxConcurrent=1   │
                                     │  500ms min gap between sends   │
                                     │                                │
                                     │  INSIDE THE LOCK:              │
                                     │  1. getChatIdByPhone()         │
                                     │     └─ V2 findChat (sync)      │
                                     │     └─ checkCapability (2 APIs) │
                                     │     └─ migrateChat (sync)      │
                                     │  2. sendMessageToChat()        │
                                     │     └─ V3 async (85-924ms)     │
                                     └─────────────────────────────────┘
                                          │
                                          ▼
                                     Linq API → Mac device → Apple → User
```

The rate limiter exists for a good reason: Apple flags lines that send too many messages too fast. Without it, burst sends would get lines blocked.

The problem is **what sits inside the lock**.

---

## Investigation: Where Is the Time Going?

### Step 1: Rule out the backend

The backend publishes messages to RMQ fire-and-forget (`rmq.service.ts:291-293`, `timeoutMs: false`). Three onboarding messages and 8 chatbot responses were all published within 68 seconds of the user's first text. LangSmith traces confirm each chatbot turn takes 3-5 seconds.

**Conclusion:** The backend is fast. Not the bottleneck.

### Step 2: Measure per-send gaps in imsg-service

Queried `imsg_logs` for all sends from line +14154349815 during the incident window:

```
TIME         TO     STATUS     GAP FROM PREV
23:26:44     0474   sent       — (first)
23:27:36     0474   failed     52.8s
23:28:30     0474   failed     53.1s
23:29:13     8684   sent       43.5s
23:29:59     0474   sent       45.5s
23:31:25     8870   sent       86.1s
23:32:13     0474   sent       48.4s
23:33:49     0474   sent       52.2s
─── TRANSITION ───
23:34:05     0474   sent       15.7s
23:34:17     0474   sent       12.4s
23:34:24     0474   sent        7.1s
23:34:31     0474   sent        6.2s
23:34:37     0474   sent        6.9s
─── RECOVERED ───
23:35:17     7022   sent       32.8s (new user)
23:35:18     7022   sent        1.3s
23:35:19     7022   sent        1.0s
```

Each gap is one `provider.sendMessage()` execution inside the rate limiter. The gaps transition from ~50s → ~7s → ~1s over 9 minutes. This is a **transient degradation**, not a fixed bottleneck.

### Step 3: Isolate getChatIdByPhone vs sendMessageToChat

Pulled Logtail/BetterStack data for this line during the incident. Two queries:

- **"Message sent"** logs contain `duration` (the sendMessageToChat time) and `start`/`end` epoch timestamps
- **"Called sendMessage"** RPC logs show when imsg-service received each message from RMQ

**sendMessageToChat is fast:**

| Time | duration | Content |
| --- | --- | --- |
| 23:26:44 | 924ms | "yo it's ditto..." (onboard) |
| 23:27:36 | 98ms | "no swiping..." (onboard) |
| 23:28:30 | 85ms | "let's get you set up..." |
| 23:29:59 | 100ms | "whoa whoa slow down bro" |
| 23:32:13 | 388ms | "first things first..." |
| 23:35:18 | 272ms | (recovered, different user) |

Never exceeds 1 second. **The Linq send API is not the bottleneck.**

**getChatIdByPhone is the bottleneck** (derived from timestamp gaps):

| Time | sendMessageToChat | getChatIdByPhone (derived) |
| --- | --- | --- |
| 23:26:44 | 924ms | ~52s (first, new user) |
| 23:27:36 | 98ms | 52.2s |
| 23:28:30 | 85ms | 52.5s |
| 23:29:13 | 232ms | 42.8s |
| 23:29:59 | 100ms | 44.9s |
| 23:31:25 | 91ms | 85.4s |
| 23:33:49 | 490ms | 50.8s |
| 23:34:04 | 365ms | 14.7s (recovering) |
| 23:34:24 | 329ms | 6.1s |
| 23:35:18 | 272ms | 0.5s (recovered) |

getChatIdByPhone takes **43-85 seconds** even for cached users (entries #2-9 all have mappings in MongoDB after entry #1). A DB lookup should take 10ms.

### Step 4: Explain why cached lookups are slow

Linq's API documentation (`apidocs.linqapp.com`) confirms a critical behavioral difference:

> **V2: Synchronous processing, returns final delivery statusV3: Asynchronous — returns immediately with message_id, webhooks confirm outcome**
> 

getChatIdByPhone calls V2 synchronous endpoints that block on the Mac device:

| Call | Endpoint | Version | Behavior |
| --- | --- | --- | --- |
| `v2Client.findChat()` | `GET /v2/chats/find` | **V2** | Synchronous — blocks on Mac |
| `client.imessageCheck()` | `POST /v3/capability/...` | V3 | Undocumented — likely Mac |
| `client.rcsCheck()` | `POST /v3/capability/...` | V3 | Undocumented — likely Mac |
| `v2Client.migrateChat()` | `GET /v2/chats/.../migrate` | **V2** | Synchronous — blocks on Mac |

Meanwhile, `sendMessageToChat` is V3 async — returns in <1s.

The Mac device was overwhelmed by **concurrent typing indicators**. Typing indicators bypass the rate limiter but call the same getChatIdByPhone function. With 7+ chatbot responses each sending typing indicators for a new user (no cached mapping), this creates 30+ simultaneous V2 API calls to one Mac device. The device can't keep up, and even simple DB-only lookups for cached users get delayed by Node.js event loop contention from the concurrent slow API calls.

Supporting evidence:

- The recovery pattern (52s → 6s → 0.5s) matches typing indicators completing over ~9 minutes
- Linq's own `message.failed` webhook reported "+14154349815 unreachable" during the window
- Health check data shows elevated round-trip (26.5s) for this line

---

## End-to-End Wait Times

The backend published all 11 messages within 68 seconds. Here's how long each waited in the imsg-service pipeline:

| Message | RPC received | Delivered | **Wait** |
| --- | --- | --- | --- |
| "yo it's ditto..." (onboard 1) | 23:25:50 | 23:26:44 | **53s** |
| "no swiping..." (onboard 2) | 23:25:50 | 23:27:36 | **106s** |
| "let's get you set up..." (onboard 3) | 23:25:51 | 23:28:30 | **159s** |
| "whoa whoa slow down bro" (bot 1) | 23:26:16 | 23:29:59 | **223s** |
| "first things first..." (bot 2) | 23:26:17 | 23:32:13 | **357s** |
| "we can handle love..." (bot 8) | 23:26:59 | 23:34:37 | **458s** |

The last chatbot response waited **7.6 minutes** in the send queue. 100% of this delay is `getChatIdByPhone` inside the rate limiter.

---

## Root Cause Summary

Two compounding problems:

1. **getChatIdByPhone runs inside the per-line rate limiter lock.** It makes 3-4 synchronous V2 Linq API calls for new users. When those calls are slow (Mac device degraded), every message for that line waits 43-85s per send. Only `sendMessageToChat` (85-924ms, V3 async) needs to be serialized — the lookup does not.
2. **Typing indicators bypass the rate limiter but call the same expensive APIs.** For new users with no cached mapping, each typing indicator triggers 3-4 V2 API calls. With 7+ concurrent chatbot responses sending typing indicators, this floods the Mac device with 30+ simultaneous requests, causing the degradation that makes even cached lookups slow.

---

## Fix Plan (Multi-Phase)

### Phase 1: Stop the flooding (imsg-service) — PR #82

**Move getChatIdByPhone outside the rate limiter**

- `rpc.service.ts`: calls `provider.resolveChatId()` before `rateLimiter.schedule()`
- `linq.provider.ts`: `getChatIdByPhone` renamed to `resolveChatId` (public), `sendMessage` accepts pre-resolved chatId
- `imessage-provider.ts`: added optional `resolveChatId()` to interface
- Race condition: `chatIdMappingModel.create()` → `updateOne` with `upsert: true`

**Skip typing indicators for new users**

- `linq.provider.ts:sendTypingIndicator()`: check for cached V3 mapping before calling Linq API. No mapping = skip. Eliminates the 30+ concurrent V2 API call flood.

**Cache capability checks**

- `linq.provider.ts:checkCapability()`: in-memory `Map` with 24h TTL. Eliminates 2 Linq API calls (imessageCheck + rcsCheck) on repeat interactions.

**Add getChatIdByPhone instrumentation**

- `linq.provider.ts:resolveChatId()`: logs `duration`, `cached` flag, and `phone` to Logtail on every call. Filter: `message = "getChatIdByPhone completed"`.

### Phase 2: Stop the message flood (proj-coach-backend) — pending

**Debounce rapid-fire messages**

- `sms-chat.service.ts`: when a user sends multiple messages rapidly, wait 3-5s after the last message before invoking the chatbot. Batch into one LangGraph invocation. Cuts 7 invocations to 1, eliminates the "spam response" pattern.

**Respond first, onboard after**

- `sms-chat.service.ts` (PendingSwitchToSms handler): if the user texted with a real message + signup code, prioritize the chatbot response over the 3 automated onboarding messages. The chatbot already asks "what's your name?" — the automated messages are redundant.

### Phase 3: Eliminate V2 dependency (imsg-service) — backlog

**Replace V2 findChat with V3 chat creation**

- Use `POST /v3/chats` (async) instead of `GET /v2/chats/find` (synchronous). Removes the synchronous Mac device dependency entirely.

**Pre-warm chat mappings at signup**

- When web signup creates the chat in proj-coach-backend, proactively create the Linq chat and cache the mapping. First message from a new user hits the fast cached path.

**Cache capability checks globally**

- Already done in Phase 1 (in-memory 24h TTL).

---

## Verification After Deploy

1. **Logtail**: search `message = "getChatIdByPhone completed"`. Cached users: `duration < 100ms`. New users on healthy lines: `duration < 5s`.
2. **imsg_logs gaps**: consecutive sends on the same line should be ~1-2s, not 40-50s.
3. **Health check**: HC Received times unchanged (measures Apple delivery, not our API calls).
4. **No regressions**: typing indicators still work for existing users (cached mapping exists). New users just don't see a typing bubble on their first message.

---

## Open Questions for Linq

1. What does `message.failed` with `reason: "+14154349815 unreachable"` mean in their system?
2. `GET /v2/chats/find` appeared to take 43-85s — do they have latency metrics for V2 endpoints?
3. `/capability/check_imessage` and `/capability/check_rcs` are not in their public docs — what's the expected latency? Do these hit the Mac device?
4. Error code 1007 (HTTP 429) exists but limits aren't documented — what are the actual thresholds?
5. Can they provide Mac device health logs for +14154349815 during 2026-04-13 23:25-23:35 UTC?

---

## Summary Table

| Claim | Status | Evidence |
| --- | --- | --- |
| Backend is fast (seconds) | **Confirmed** | Fire-and-forget RMQ. LangSmith: 4.3s/turn. All 11 msgs in 68s. |
| Rate limiter serializes per line | **Confirmed** | Code: `maxConcurrent: 1`, key is `dittoNumber` |
| `sendMessageToChat` is fast (85-924ms) | **Confirmed** | Logtail `duration` field, all 29 messages in the window |
| `getChatIdByPhone` is the bottleneck (43-85s) | **Confirmed** | Derived from Logtail `start`/`end` timestamps |
| Slow even for cached users | **Confirmed** | Entries #2-9 all have cached mappings, still 43-85s |
| V2 API is synchronous | **Confirmed** | Linq API docs: V2 sync, V3 async |
| Linq reported line "unreachable" | **Confirmed** | `imsg_logs` errorMessage from `message.failed` webhook |
| Typing indicators flood the Mac device | **Hypothesis** | Code confirms they bypass rate limiter + call getChatIdByPhone |
| Mac device was overloaded | **Hypothesis** | Consistent with all evidence; need Linq device health logs to prove |