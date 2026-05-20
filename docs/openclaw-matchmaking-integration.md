# OpenClaw Deep Technical Guide — Ditto Matchmaking Integration

**Status:** Internal Reference
**Prepared for:** 45-minute technical session
**Date:** 2026-03-24
**Context:** Evaluating OpenClaw as the Agent OS layer for Ditto Matchmaking Engine 3.x

---

## Table of Contents

1. [How the Kernel Works Under the Hood](#1-how-the-kernel-works-under-the-hood)
2. [How the Pluggable Context Engine Works](#2-how-the-pluggable-context-engine-works)
3. [Incorporating Domain Knowledge and SOPs](#3-incorporating-domain-knowledge-and-sops-into-the-agent-os)
4. [Enforcing Procedures with Plugins](#4-enforcing-procedures-with-plugins)
5. [OpenClaw Security Best Practices](#5-openclaw-security-best-practices)
6. [Q&A — Ditto-Specific](#6-qa--ditto-specific)
7. [Appendix: OpenClaw × Matchmaking 3.x Mapping](#appendix-openclaw--matchmaking-3x-mapping)

---

## 1. How the Kernel Works Under the Hood

### Overview

OpenClaw's kernel is a **gateway-centric, event-driven agent runtime**. The gateway is a single always-on Node.js process that multiplexes three concerns on one port (default 18789):

- **WebSocket control plane** — agent RPC, session management, presence, events
- **HTTP APIs** — OpenAI-compatible endpoints, tool invocation, responses API
- **Control UI** — web dashboard and hook endpoints

### Agent Turn Lifecycle (The Core Loop)

When a message arrives, the kernel executes this pipeline:

```
1. ENTRY & VALIDATION
   └─ `agent` RPC validates params, resolves session (via sessionKey/sessionId)
   └─ Persists metadata, returns { runId, acceptedAt } immediately

2. COMMAND ORCHESTRATION (`agentCommand`)
   └─ Resolves model defaults + auth profile
   └─ Loads skills snapshot (cached per session start)
   └─ Invokes `runEmbeddedPiAgent` (the pi-agent-core runtime)

3. EMBEDDED RUNTIME (`runEmbeddedPiAgent`)
   └─ Serializes runs through per-session lane + optional global lane
   └─ Resolves model/auth, builds pi session
   └─ Subscribes to pi events, enforces timeout with abort
   └─ Returns payload with usage metadata

4. CONTEXT ASSEMBLY (before model call)
   └─ Workspace resolves (sandbox redirects if applicable)
   └─ Skills load from snapshot → inject into environment + prompt
   └─ System prompt built: base → skills → bootstrap files → per-run overrides
   └─ Token budget enforced, compaction reserve calculated

5. MODEL INVOCATION
   └─ LLM called with assembled context + tool definitions
   └─ Streaming: deltas arrive as `assistant` events
   └─ If model returns tool_calls → execute tools → loop back to model
   └─ Loop protection: configurable max iterations

6. EVENT BRIDGING (`subscribeEmbeddedPiSession`)
   └─ Pi-agent-core events → OpenClaw streams:
       tool events    → stream: "tool" (start/update/end)
       assistant deltas → stream: "assistant"
       lifecycle       → stream: "lifecycle" (start/end/error)

7. RESPONSE ASSEMBLY
   └─ Combine: assistant text + optional reasoning + tool summaries
   └─ NO_REPLY token filters silent responses
   └─ Duplicate message suppression for messaging channels
   └─ Auto-compaction may trigger retry (buffers reset)
```

### Concurrency Model

Runs are **serialized per session key** (session lane) with an optional global lane. This prevents tool/session races:

- Each session gets exclusive sequential execution
- Messaging channels use queue modes (collect/steer/followup) feeding into the lane
- If the main queue is busy (e.g., during heartbeat), new work waits

### System Prompt Construction

The system prompt is rebuilt each run and contains these fixed sections:

| Section | Content |
|---------|---------|
| **Tooling** | Current tool inventory with descriptions |
| **Safety** | Guardrail reminders against oversight circumvention |
| **Skills** | Compact XML list of eligible skills (name, description, location) |
| **Workspace** | Working directory reference |
| **Workspace Files** | Injected bootstrap files (AGENTS.md, SOUL.md, USER.md, etc.) |
| **Sandbox** | Runtime isolation details (if applicable) |
| **Current Date & Time** | User timezone, format preferences |
| **Runtime** | Host, OS, Node version, model, thinking level |

Three prompt modes exist:

- **`full`** (default): All sections for primary agents
- **`minimal`**: Sub-agent variant omitting Skills, Memory, Self-Update, User Identity, Heartbeats
- **`none`**: Base identity line only

### Hook Interception Points

The kernel has two hook layers that intercept execution at strategic points:

**Internal (Gateway) hooks:**

| Event | When It Fires |
|-------|---------------|
| `agent:bootstrap` | Before system prompt finalization — can mutate bootstrap files |
| `command:new/reset/stop` | On slash commands |
| `message:received` | Inbound message from any channel (before media processing) |
| `message:transcribed` | After audio transcription and link understanding |
| `message:preprocessed` | After all media/link understanding, before agent processes |
| `message:sent` | Outbound message successfully sent |
| `session:compact:before/after` | Around compaction events |
| `gateway:startup` | After channels load and hooks register |

**Plugin hooks:**

| Hook | Purpose |
|------|---------|
| `before_model_resolve` | Override model/provider per-session |
| `before_prompt_build` | Inject additional context post-session |
| `before_tool_call` / `after_tool_call` | Intercept tool params/results |
| `tool_result_persist` | Synchronously transform results before storage |
| `before_compaction` / `after_compaction` | Observe compaction |
| `message_received` / `message_sending` / `message_sent` | Message lifecycle |
| `session_start` / `session_end` | Session boundary events |

### Key Kernel Properties

- **Fail-fast**: Clients fail immediately when gateway unavailable (no fallback)
- **No event replay**: On sequence gaps, clients must refresh state via `health` query
- **Graceful shutdown**: Emits `shutdown` event before socket closure
- **Hot-reload**: Config changes apply via `hybrid` mode without restart (except server bind settings)
- **Default timeouts**: `agent.wait` = 30s; agent runtime = 600s (configurable)

### Heartbeat — Autonomous Background Monitoring

The heartbeat system runs **periodic agent turns** in the main session without user-initiated requests:

- Default interval: 30 minutes
- Default prompt: "Read HEARTBEAT.md if it exists. Follow it strictly. If nothing needs attention, reply HEARTBEAT_OK."
- Configurable per-agent: interval, model override, target channel, active hours
- `isolatedSession: true` runs each heartbeat in a fresh session (reduces ~100K tokens to ~2-5K per run)
- Empty HEARTBEAT.md → heartbeat run is skipped entirely (saves API cost)

---

## 2. How the Pluggable Context Engine Works

### Overview

The context engine is a **modular slot** that controls how model context is assembled, compacted, and managed. OpenClaw ships with a built-in `legacy` engine and allows plugins to register alternatives.

### Lifecycle — 4 Critical Points

```
1. INGEST — when new messages arrive
   └─ Engine can store/index messages in custom data stores
   └─ Legacy engine: no-op (session manager handles persistence)

2. ASSEMBLE — before each model invocation (most important)
   └─ Returns ordered message set fitting the token budget
   └─ Can inject dynamic systemPromptAddition
   └─ Legacy engine: pass-through via sanitize → validate → limit pipeline

3. COMPACT — when context fills or user runs /compact
   └─ Summarize older history to free space
   └─ Legacy engine: single summary, preserve recent messages

4. AFTER TURN — post-execution
   └─ State persistence, background compaction triggers, index updates
   └─ Optional: onSubagentEnded for child session cleanup
```

### The ContextEngine Interface

```typescript
interface ContextEngine {
  // Required
  info: { id: string; name: string; ownsCompaction: boolean };
  ingest(params: { sessionId, message, isHeartbeat }): Promise<{ ingested: boolean }>;
  assemble(params: { sessionId, messages, tokenBudget }): Promise<AssembleResult>;
  compact(params: { sessionId, force }): Promise<{ ok: boolean; compacted: boolean }>;

  // Optional
  bootstrap?(params): Promise<void>;              // Initialize session state
  ingestBatch?(params): Promise<void>;             // Ingest completed turn as batch
  afterTurn?(params): Promise<void>;               // Post-run work
  prepareSubagentSpawn?(params): Promise<void>;    // Setup child session shared state
  onSubagentEnded?(params): Promise<void>;         // Child session cleanup
  dispose?(): Promise<void>;                       // Release resources on shutdown
}

interface AssembleResult {
  messages: Message[];            // Ordered messages for the model
  estimatedTokens: number;        // Engine's token count (required for compaction)
  systemPromptAddition?: string;  // Prepended to system prompt (dynamic injection)
}
```

### The `ownsCompaction` Flag

| Value | Behavior |
|-------|----------|
| `true` | Engine owns compaction entirely. OpenClaw disables Pi's built-in auto-compaction. The engine handles `/compact`, overflow recovery, and proactive compaction via `afterTurn()`. |
| `false` (default) | Pi's auto-compaction may still run. Engine's `compact()` handles `/compact` and overflow recovery as supplement. |

**Warning**: A no-op `compact()` with `ownsCompaction: false` is **unsafe** — it disables normal compaction paths without providing a replacement.

### Configuration

```json5
{
  plugins: {
    slots: {
      contextEngine: "my-custom-engine"  // Exclusive slot — one engine at a time
    },
    entries: {
      "my-custom-engine": { enabled: true }
    }
  }
}
```

### Plugin Registration Example

```typescript
export default function register(api) {
  api.registerContextEngine("ditto-matchmaking", () => ({
    info: {
      id: "ditto-matchmaking",
      name: "Ditto Matchmaking Context Engine",
      ownsCompaction: true,
    },
    async ingest({ sessionId, message, isHeartbeat }) {
      // Store match decisions in structured trace store
      return { ingested: true };
    },
    async assemble({ sessionId, messages, tokenBudget }) {
      // RAG over match history — inject only relevant context
      const relevantContext = await retrieveMatchContext(sessionId, tokenBudget);
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: `Current cycle status: ${relevantContext.summary}`,
      };
    },
    async compact({ sessionId, force }) {
      // Extract structured decision traces before discarding history
      await extractAndPersistDecisionTraces(sessionId);
      return { ok: true, compacted: true };
    },
  }));
}
```

### Relationship to Other Systems

- **Memory plugins** (`plugins.slots.memory`) provide search/retrieval; context engines control what the model sees. They can collaborate — engines may query memory data during assembly.
- **Session pruning** (trimming old tool results) runs regardless of active engine.
- Engine errors **do not fall back** to legacy — runs fail until the engine is fixed.
- Existing sessions maintain current history when switching engines; the new engine takes over for future runs.

### Ditto-Specific Opportunities

A custom context engine for matchmaking could:

- **RAG over match history**: `assemble()` does vector retrieval over past matches, user profiles, coach notes — injecting only relevant context per decision
- **Dynamic state injection**: `systemPromptAddition` can inject real-time matchmaking state (current cycle scores, user engagement metrics) without static workspace files
- **Intelligent compaction**: Instead of generic summarization, compact matchmaking sessions by extracting and persisting structured decision traces
- **Cross-agent context**: `prepareSubagentSpawn` / `onSubagentEnded` can share matchmaking context between the match reviewer and coach assistant agents

---

## 3. Incorporating Domain Knowledge and SOPs into the Agent OS

OpenClaw has a **layered knowledge injection system** with increasing specificity and enforcement strength.

### Layer 1: Workspace Bootstrap Files (Always-Loaded Domain Knowledge)

Seven files are injected into the system prompt on **every turn**:

| File | Purpose | Matchmaking Use |
|------|---------|-----------------|
| **AGENTS.md** | Operating instructions, rules, behavioral priorities | Matchmaking procedures, escalation rules, quality standards, standing orders |
| **SOUL.md** | Persona, tone, boundaries | "You are Ditto's matchmaking intelligence. Precise, empathetic, data-driven." |
| **USER.md** | User context, communication preferences | Coach team context, school-specific knowledge |
| **IDENTITY.md** | Agent name, personality, emoji | "Match Reviewer Agent" identity |
| **TOOLS.md** | Notes about local tools and conventions | How to use `get_candidate_scores`, `approve_match`, etc. |
| **HEARTBEAT.md** | Lightweight checklist for autonomous monitoring | "Check for stale matches, calibration drift, coach queue depth" |
| **MEMORY.md** | Curated long-term memory | Accumulated matchmaking patterns, past cycle learnings |

**Size constraints**: 20,000 chars per file, 150,000 chars total. Files are truncated with markers when exceeded. Keep these concise — they consume tokens every turn.

**Missing files** trigger marker injection without halting execution. `openclaw setup` recovers missing defaults without overwriting existing content.

### Layer 2: Standing Orders (SOPs as Autonomous Programs)

Standing orders in AGENTS.md grant **permanent operating authority** within defined boundaries. Each standing order program must specify:

1. **Scope** — authorized actions and boundaries
2. **Triggers** — execution timing (schedule, event, condition)
3. **Approval gates** — human sign-off requirements
4. **Escalation rules** — conditions requiring human intervention

#### Example: Weekly Match Cycle Standing Order

```markdown
## Standing Order: Weekly Match Cycle

### Scope
You are authorized to:
- Run the scoring pipeline for all active schools
- Review all proposed matches using the feasibility-gate and diversity-injector skills
- Auto-approve matches with confidence > 0.8
- Escalate matches with confidence < 0.5 to coach channel

### Triggers
- Every Monday and Thursday at 9:00 AM ET via cron

### Approval Gates
- First 30 days: all matches require coach confirmation
- After 30 days: only low-confidence matches require confirmation

### Escalation Rules
- If match-to-conversation rate drops below 40%: pause automation, alert #matchmaking-ops
- If any safety flag is raised: halt cycle, alert immediately
- If scoring pipeline fails: retry once, then alert engineering

### What You Must NOT Do
- Override trust/safety filters under any circumstances
- Match users who are in "Paused" or "Banned" status
- Send matches to users without active profiles
```

#### Execution Discipline: Execute-Verify-Report

All tasks must follow this pattern:

1. **Execute** — Complete actual work, not just acknowledge
2. **Verify** — Confirm correctness (matches logged, scores checked)
3. **Report** — Document what was done and what verification confirmed
4. If failure: retry once with adjusted approach. If still fails: report with diagnosis. **Never silently fail.**

### Layer 3: Skills (Composable Domain Procedures)

Each `SKILL.md` encodes a specific matchmaking procedure:

```markdown
---
name: feasibility-gate
description: Apply mutual feasibility threshold filtering to match candidates
---

## Procedure
1. Retrieve calibrated Likelihood scores for all candidates via get_candidate_scores
2. Determine threshold based on user state:
   - Cold-start (< 10 interactions): threshold = 0.45
   - Convergent (strong recent outcomes): threshold = 0.70
   - Default: threshold = 0.60
3. For each candidate below threshold: reject with reason logged
4. NEVER drop threshold below 0.30 (hard safety floor)
5. Return filtered candidate list with decision trace
```

Skills are **loaded on-demand** — only file paths appear in the system prompt; the agent `read`s SKILL.md when needed. This keeps base context small.

**Skill types and precedence** (highest wins):

1. `<workspace>/skills/` — workspace-specific (per agent)
2. `~/.openclaw/skills/` — managed/local (shared across workspaces)
3. Bundled skills — shipped with installation

**Skill gating**: Skills can specify requirements (bins, env vars, config, OS) via `metadata.openclaw.requires`. Ineligible skills are excluded entirely.

### Layer 4: Memory (Persistent Cross-Session Knowledge)

- **Daily logs**: `memory/YYYY-MM-DD.md` files with accumulated learnings
- **MEMORY.md**: Curated long-term memory loaded in primary sessions
- Memory persists on disk; context exists only within the current model window
- Automatic memory flushing supplements manual curation

### Layer 5: Hooks (Programmatic Enforcement)

Hooks enforce procedures at the **code level** — not just LLM instructions:

```typescript
// hooks/match-safety-gate/handler.ts
export default async function(event) {
  if (event.type === 'tool' && event.action === 'approve_match') {
    const { pairId } = event.context.args;
    const safetyCheck = await checkTrustFlags(pairId);
    if (!safetyCheck.passed) {
      event.context.blocked = true;
      event.messages.push(`Match ${pairId} blocked: ${safetyCheck.reason}`);
    }
  }
}
```

### Enforcement Strength Comparison

| Layer | What It Enforces | Can LLM Bypass? | Best For |
|-------|-----------------|-----------------|----------|
| **Tool policy** (`tools.deny`) | Block entire tool categories | No — gateway enforces | Hard tool restrictions |
| **Plugin hooks** (`before_tool_call`) | Validate/block specific invocations | No — code runs before tool | Safety constraints, data validation |
| **Tool handler validation** | Parameter constraints | No — code rejects invalid inputs | Input quality enforcement |
| **Sandbox** | Filesystem/network isolation | No — OS-level | Untrusted content handling |
| **Standing orders** (AGENTS.md) | Behavioral procedures | Soft — LLM may deviate | Operational SOPs |
| **Skills** (SKILL.md) | Step-by-step procedures | Soft — LLM may interpret differently | Domain procedures |
| **SOUL.md persona** | Tone, boundaries | Soft — advisory only | Personality, brand voice |

**Key principle**: *"Safety guardrails in the system prompt are advisory. They guide model behavior but do not enforce policy. Hard enforcement relies on tool policy, exec approvals, sandboxing, and channel allowlists."*

---

## 4. Enforcing Procedures with Plugins

Plugins are the **strongest enforcement mechanism** because they operate at the runtime level, not the prompt level.

### Plugin Architecture

Plugins extend OpenClaw across multiple dimensions:

```typescript
export default definePluginEntry({
  id: "@ditto/matchmaking",
  register(api) {
    api.registerProvider({...});        // Model providers
    api.registerChannel({...});         // Chat channels
    api.registerTool({...});            // Agent tools
    api.registerContextEngine({...});   // Context assembly
    api.registerHook({...});            // Lifecycle hooks
    // + speech, image gen, web search, HTTP endpoints, CLI commands,
    //   context engines, background services
  }
});
```

### Enforcement Points for Matchmaking

#### 1. Tool Interception — Enforce Before/After Every Tool Call

```typescript
api.registerHook("before_tool_call", async ({ toolName, args, session }) => {
  // Block approve_match if safety flags exist
  if (toolName === "approve_match") {
    const safety = await checkSafety(args.pairId);
    if (!safety.ok) return { blocked: true, reason: safety.reason };
  }
  // Enforce logging — every tool call must produce a decision trace
  await logDecisionTrace(toolName, args, session.id);
});

api.registerHook("after_tool_call", async ({ toolName, result, session }) => {
  // Validate outputs — reject malformed match decisions
  if (toolName === "approve_match" && !result.reasoning) {
    return { error: "Match approval requires reasoning field" };
  }
});
```

#### 2. Tool Result Persistence — Transform Before Storage

```typescript
api.registerHook("tool_result_persist", ({ toolName, result }) => {
  // Sanitize PII from decision traces before persisting to session transcript
  if (toolName.startsWith("get_user_")) {
    return sanitizeForPersistence(result);
  }
});
```

#### 3. Context Injection — Enforce Context at Prompt Level

```typescript
api.registerHook("before_prompt_build", async ({ session }) => {
  const cycleStatus = await getCurrentCycleStatus();
  return {
    systemPromptAddition: `Current cycle: ${cycleStatus.summary}`
  };
});
```

#### 4. Message Lifecycle — Enforce Output Quality

```typescript
api.registerHook("message_sending", async ({ message, channel }) => {
  // Validate coach-facing messages meet quality bar
  if (channel === "slack" && message.includes("approve")) {
    const quality = await validateMatchExplanation(message);
    if (!quality.ok) return { blocked: true, reason: "Insufficient explanation" };
  }
});
```

#### 5. Custom Tools — Register Matchmaking Tools with Built-In Validation

```typescript
api.registerTool({
  name: "approve_match",
  description: "Approve a proposed match with structured reasoning",
  parameters: {
    pairId: { type: "string", required: true },
    reasoning: { type: "string", required: true },
    confidence: { type: "number", required: true }
  },
  handler: async ({ pairId, reasoning, confidence }) => {
    // Hard enforcement: reasoning must be substantive
    if (reasoning.length < 50) throw new Error("Reasoning too brief");
    // Hard enforcement: confidence must be above threshold
    if (confidence < 0.5) throw new Error("Cannot auto-approve below 0.5");
    // Hard enforcement: safety check
    const safety = await checkTrustFlags(pairId);
    if (!safety.ok) throw new Error(`Safety block: ${safety.reason}`);

    return await commitMatch(pairId, reasoning, confidence);
  }
});
```

### Plugin Configuration and Management

```json5
{
  plugins: {
    entries: {
      "@ditto/matchmaking": {
        enabled: true,
        env: { MONGODB_URI: "...", RESTATE_ENDPOINT: "..." },
        config: { safetyThreshold: 0.3, maxAutoApproveConfidence: 0.8 }
      }
    }
  }
}
```

### Plugin Safety Constraints

- **Kill-switch**: Any plugin can be disabled instantly via config or CLI (`openclaw plugins disable @ditto/matchmaking`)
- **Per-plugin KPIs**: Track measurable outcomes per plugin
- **Audit trail**: Every plugin hook invocation is traceable via gateway logs
- **Version pinning**: Pin plugin versions to prevent unexpected behavior changes
- **Deny by default**: New plugins are disabled until explicitly enabled

### What Plugins Should NOT Do

From the Matchmaking 3.x strategy:

> "Plugins should shape policy around calibrated outputs; they should not replace the underlying representation, state, or prediction layers."

Specifically avoid:
- Using plugins to compensate for broken model predictions
- Building hidden ranking systems inside plugin hooks
- Unbounded heuristic overrides on top of model outputs
- Accumulating more than 5 active policy plugins without quarterly audit

---

## 5. OpenClaw Security Best Practices

### Foundational Principle

> *"OpenClaw is **not** a hostile multi-tenant security boundary for multiple adversarial users. One user/trust boundary per gateway."*

### Trust Boundaries (5 Nested Layers)

```
Layer 1: Channel Access
   └─ Device pairing (30s grace), AllowFrom/AllowList, token/password/Tailscale auth

Layer 2: Session Isolation
   └─ Session keys: agent:channel:peer, per-agent tool policies, transcript logging

Layer 3: Tool Execution
   └─ Docker sandbox or host exec via approvals, SSRF protection (DNS pinning, IP blocking)

Layer 4: External Content
   └─ Fetched URLs/emails wrapped in XML tags with security notice injection

Layer 5: Supply Chain
   └─ ClawHub skill publishing requirements, pattern-based moderation, (planned) VirusTotal scan
```

### The 10 Security Practices

#### 1. Start Restrictive, Loosen Deliberately

```json5
{
  tools: {
    profile: "messaging",  // minimal tool set
    deny: ["group:automation", "group:runtime", "group:fs",
           "sessions_spawn", "sessions_send"],
    exec: { security: "deny" },
    elevated: { enabled: false }
  }
}
```

#### 2. Isolate Sessions Per User

```json5
{ session: { dmScope: "per-channel-peer" } }
```

Default `main` scope leaks context between users. Always use per-peer isolation for multi-user deployments (e.g., multiple coaches messaging the agent).

#### 3. Use Pairing Mode for DM Access

This is the default — don't change it. Unknown senders receive expiring pairing codes. Codes expire after 1 hour. Max 3 pending per channel. No message processing until approved via `openclaw pairing approve`.

#### 4. Bind to Loopback, Authenticate Everything

```json5
{
  gateway: {
    bind: "loopback",
    auth: { mode: "token", token: "<long-random-token>" }
  }
}
```

**Never** expose the gateway unauthenticated on `0.0.0.0`.

#### 5. Use Strong Models for Tool-Enabled Agents

From the threat model: *"Prompt-injection risk with older/smaller models is often too high. Do not run those workloads on weak model tiers."* Use Claude Sonnet 4/Opus 4 or equivalent for agents with tool access.

#### 6. Sandbox by Default for Untrusted Content

```json5
{
  agents: { defaults: { sandbox: {
    mode: "all",
    workspaceAccess: "ro"
  }}}
}
```

#### 7. Treat External Content as Hostile

The kernel wraps fetched URLs/emails in XML tags with security notices, but LLMs may ignore wrappers. Defense: restrict `web_fetch`, `browser` tools. Use tool allowlists for specific domains.

#### 8. Plugin/Skill Supply Chain Hygiene

- Plugins run **in-process** with the gateway — treat as trusted code
- Review unpacked code before enabling
- Pin versions, use explicit `plugins.allow` allowlists
- ClawHub moderation uses regex patterns — don't rely on it alone

#### 9. Run `openclaw security audit` Regularly

Scans for: DM/group exposure, tool blast radius, exec approval drift, network exposure, browser/node control, disk permissions, plugin allowlists. Use `--deep` for live gateway probe.

#### 10. Lock Down Filesystem Permissions

```
~/.openclaw/              → 700
~/.openclaw/openclaw.json → 600
credentials/              → 600
```

`openclaw security audit --fix` auto-corrects permissions.

### Hardened Baseline Configuration

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: { mode: "token", token: "replace-with-long-random-token" },
  },
  session: { dmScope: "per-channel-peer" },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs",
           "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } }
    },
  },
}
```

### Ditto-Specific Security Concerns

| Risk | Mitigation |
|------|-----------|
| Agent approves unsafe match | Plugin hook `before_tool_call` on `approve_match` checks trust/safety flags — **code enforcement, not prompt** |
| Coach data leaks between sessions | `dmScope: per-channel-peer` isolates each coach |
| Prompt injection via user chat data | Never pass raw user messages to matchmaking agent tools. Sanitize through data layer. |
| Agent modifies match scores in DB | Tool handler validates: agent can only approve/reject, not modify scores directly. Write tools restricted to specific collections. |
| Credential exposure | Use `SecretRef` for MongoDB URI, API keys. Never in AGENTS.md or workspace files. |

### Known Threat Model Gaps (from OpenClaw's own assessment)

| Area | Risk Level | Current Mitigation | Gap |
|------|-----------|-------------------|-----|
| Prompt injection via channel messages | Critical | Pattern detection (no blocking) | LLM-dependent; no hard prevention |
| Malicious skill publication to ClawHub | Critical | Regex moderation, GitHub account age | No sandboxing, limited review |
| Token theft from plaintext config | High | File permissions only | No encryption at rest |
| Tool argument manipulation via injection | High | Exec approvals | Relies on user judgment |
| Data exfiltration via web_fetch | High | SSRF blocks internal networks only | External URLs unrestricted |

### Incident Response

1. **Contain**: Stop process, set `gateway.bind: "loopback"`, disable Tailscale, switch channels to `dmPolicy: "disabled"`
2. **Rotate**: Gateway auth token, all provider credentials (WhatsApp, Slack, Discord tokens, API keys)
3. **Audit**: Check `/tmp/openclaw/openclaw-YYYY-MM-DD.log`, review session transcripts, run `security audit --deep`

---

## 6. Q&A — Ditto-Specific

### "Can OpenClaw replace our Restate pipeline?"

**No.** OpenClaw is an LLM orchestration platform, not a compute framework. Use OpenClaw for judgment, policy, and communication. Keep Restate for parallel scoring, batch computation, and durable execution of numerical workloads. They are complementary:

- **OpenClaw** = brain (LLM reasoning, policy decisions, communication)
- **Restate** = muscle (parallel computation, durable execution)
- **MongoDB** = memory (data storage)
- **UFL** = senses (signal extraction from conversations)

### "How does OpenClaw scale with user volume?"

OpenClaw scales by adding agents (multi-agent routing) and gateway instances (multiple gateways). Each agent is isolated. But LLM inference is the bottleneck — every agent turn is an API call. For batch operations (scoring 500 pairs), use Restate. For decision operations (reviewing 20 proposed matches), use OpenClaw.

### "What happens if the gateway goes down during a match cycle?"

The cron system has built-in retry: transient errors retry up to 3x with exponential backoff (30s → 1m → 5m). Persistent errors disable the job and alert. The Restate pipeline (scoring) has its own durability guarantees independent of OpenClaw.

### "Can we run OpenClaw in production without Docker?"

Yes. The gateway is a plain Node.js process. Docker is optional for sandboxing. For Ditto's internal use case (trusted agents, controlled tools), you can run bare metal. Use Docker sandbox for any agent that processes untrusted user content.

### "How do we version-control matchmaking skills?"

Skills are SKILL.md files in the workspace. Put them in git. Workspace skills take precedence over managed/bundled skills. Deploy by updating the workspace directory. ClawHub can distribute skills across environments.

### "What's the token cost per matchmaking decision?"

| Component | Token Estimate |
|-----------|---------------|
| System prompt (tools + skills list + bootstrap files) | 2K–5K tokens |
| Per-match context (scores, profiles, history) | 1K–3K tokens |
| LLM response | 500–2K tokens |
| **Total per match review** | **4K–10K tokens** |

For 50 matches per cycle = ~250K–500K tokens. With `isolatedSession: true` on cron, you avoid conversation history accumulation.

### "How do we integrate OpenClaw with the existing LangGraph chatbot?"

Two integration paths:

1. **Coexistence**: OpenClaw handles matchmaking agents (match reviewer, coach assistant, monitoring). LangGraph handles user-facing chatbot (onboarding, general, match, profile improvement, yak agents). They share MongoDB but run independently.

2. **OpenClaw as orchestrator**: OpenClaw agents use tools to trigger LangGraph chatbot flows for specific user interactions (e.g., post-match check-in). The `sessions_send` tool or webhooks can bridge the two systems.

Recommendation: Start with coexistence (path 1). Migrate to path 2 only if maintaining two agent runtimes becomes burdensome.

---

## Appendix: OpenClaw × Matchmaking 3.x Mapping

### Where OpenClaw Fits in the 3.x Architecture

| 3.x Layer | What It Does | OpenClaw Fit |
|-----------|-------------|-------------|
| 1. Hard Constraints | Deterministic filtering | **No** — database queries, not LLM |
| 2. Representation & Retrieval | Embedding backbone, ANN search | **No** — ML infrastructure |
| 3. Temporal User State | Evolving taste, receptivity | **Partial** — sessions track history; state computation is ML |
| 4. Four Decision Heads | Likelihood, Intensity, Chemistry, Readiness | **No for inference**; **Yes for interpretation** of head outputs |
| 5. Calibration + Agent Skills/Plugins | Modular policy operators | **Core fit — this is OpenClaw's sweet spot** |
| 6. Learning Layer | RL training, outcome attribution | **No** — ML training infrastructure |

### Proposed Multi-Agent Architecture

```
OpenClaw Gateway
├── Agent: "match-reviewer"
│   ├── Skills: feasibility-gate, diversity-injector, coach-escalation
│   ├── Tools: get_candidate_scores, approve_match, reject_match, escalate_to_coach
│   ├── Cron: "Match review cycle Mon/Thu 9am"
│   └── Channels: WebChat (dashboard), Slack (notifications)
│
├── Agent: "coach-assistant"
│   ├── Skills: match-explanation, user-insight, outcome-prediction
│   ├── Tools: get_match_explanation, get_similar_past_matches, submit_coach_decision
│   ├── Channels: WhatsApp (coaches), Slack
│   └── Session: per-coach isolation (dmScope: per-channel-peer)
│
├── Agent: "user-engagement"
│   ├── Skills: post-match-checkin, post-date-feedback, preference-discovery
│   ├── Tools: update_user_state, record_feedback, update_preferences
│   ├── Cron: "Check in 48h after match" / "Post-date feedback request"
│   └── Channels: WhatsApp, SMS
│
└── Agent: "matchmaking-ops"
    ├── Skills: calibration-monitor, fairness-check, exploration-diagnostics
    ├── Tools: get_funnel_metrics, get_calibration_report, get_cohort_fairness
    ├── Cron: "Weekly system health report Mon 8am"
    └── Channels: Slack (#matchmaking-ops)
```

### Time Savings Estimate

| Component | Without OpenClaw | With OpenClaw | Savings |
|-----------|-----------------|---------------|---------|
| Agent skills/plugins framework | Design + build from scratch | Use skills system + plugin architecture | 4–6 weeks |
| Coach communication | Custom notification system | Native WhatsApp/Telegram/Slack channels | 2–3 weeks |
| Scheduled match cycles | Custom scheduler + retry + error handling | Cron with delivery modes, retry, session isolation | 1–2 weeks |
| Match review workflow | Custom agent framework + review routing | Multi-agent routing + tool-based review | 3–4 weeks |
| User feedback collection | Custom outreach system | User engagement agent via existing channels | 2–3 weeks |
| System monitoring | Custom dashboards + alerting | Monitoring agent on cron, Slack delivery | 1–2 weeks |
| **Total** | | | **13–20 weeks** |

### References

- OpenClaw Docs: https://docs.openclaw.ai
- OpenClaw GitHub: https://github.com/openclaw/openclaw
- ClawHub Skills Marketplace: https://clawhub.ai
- Ditto Matchmaking Engine 3.x Strategy: `docs/Ditto Matchmaking Engine 3 x 326d2ecb07cc804a90e1da42480d0aad.md`
