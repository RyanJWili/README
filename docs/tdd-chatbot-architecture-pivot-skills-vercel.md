# TDD: Chatbot Architecture Pivot — Skills + Vercel AI SDK

# 1. Executive Summary

This document describes the architectural pivot of the Ditto chatbot from a LangGraph-based multi-agent system to a skills-based single agent powered by the Vercel AI SDK. The refactor eliminates the LangChain/LangGraph dependency stack, replaces five hardcoded agents with one unified agent that activates domain-specific skills on demand, and migrates observability to OpenTelemetry with multiple backends under evaluation (LangSmith, Raindrop, Braintrust, Confident AI).

| Field | Value |
| --- | --- |
| **Author** | Muskan / Ditto Engineering |
| **Date** | March 2026 |
| **Status** | Draft |
| **Linear Epic** | ENG-1518 |
| **Team** | Eng |

## 1.1 Goals

- Replace 5 status-routed agents with 1 unified agent using skill-based architecture
- Remove LangGraph and all `@langchain/*` dependencies. Since the pipeline is linear, no graph framework needed
- Adopt Vercel AI SDK (`ai` package) for model-agnostic tool execution with `streamText` + `maxSteps`
- Migrate observability to OpenTelemetry — trial multiple backends (LangSmith, Raindrop, Braintrust, Confident AI) simultaneously

- Implement self-learning Handbook (MongoDB) to reduce `handOverToTeam` escalations over time

## 1.2 Non-Goals

- Changing content moderation (Qwen3Guard + regex guardrails stay untouched)
- Migrating away from MongoDB for conversation/user state
- Changing the LLM provider — Vercel AI SDK is model-agnostic, provider is a config choice
- Rewriting `sendResponse` (chunk splitting, quiet-check, pending review flow stays as-is)

---

# 2. Current Architecture

## 2.1 System Overview (dev branch)

The production chatbot on the `dev` branch runs on LangGraph with a `StateGraph` that routes users to one of five specialized agents based on user status. Each agent is a separate file with its own system prompt (loaded via PromptService), its own tool subset, and a dedicated state reminder builder. The LLM is invoked via LangChain's `ChatOpenAI` wrapper (through Vercel AI Gateway). Observability is handled by LangSmith (direct integration, not OpenTelemetry).

**Key architectural characteristics:**

- **5 agents**: generalAgent, matchAgent, onboardingAgent, profileImprovementAgent, yakAgent
- `routeToAgent()` in `chatbot-graph.ts` selects the agent based on `activePool`, `completedOnboarding`, `currentStatus`, and `matchId`
- Each agent uses `buildAgentNodes()` to create a `callModel` + `tools` node pair with explicit tool-loop iteration tracking (max 10)
- Tools defined with LangChain `tool()` and return `Command` objects with `shouldRespond` flags + `ToolMessage`
- **4 separate state reminder builders** (one per agent type) that mix facts with behavioral instructions
- Per-agent prompt templates managed by PromptService with school-specific Mustache rendering
- LangSmith tracing via direct `@langchain` integration (not OpenTelemetry)

**Graph execution flow:**

```tsx
START → maliciousGuard → emojiGuard → loadProfile → loadMessages
  → routeToAgent(state)
    if activePool === 'yik-yak'              → yakAgent
    if !completedOnboarding                   → onboardingAgent
    if currentStatus === 'Matched' || matchId → matchAgent
    if currentStatus === 'NeedMoreInfo'       → profileImprovementAgent
    else                                      → generalAgent
  → each agent has _callModel ↔ _tools loop
  → sendResponse → END
```

## 2.2 Problems

### 2.2.1 Excessive escalations

When an agent encounters a question outside its prompt-embedded knowledge, it calls `handOverToTeam` because it has no way to look up the answer. The agent often hits this path simply because it lacks knowledge, not because the question is genuinely complex.

### 2.2.2 High cost of adding features

Adding a new event pool (e.g. NYC Gala) requires modifying `routeToAgent()`, creating/updating agent files, writing new prompts, updating state reminders, and configuring tool subsets across 4-6 files.

### 2.2.3 Knowledge drift

Each of the 5 agents has its own prompt with overlapping personality rules, safety instructions, and behavioral guidelines. These were ~70% rules, ~20% examples — the inverse of what works best.

### 2.2.4 State reminders mix facts with instructions

Current state reminders contain both factual state AND behavioral directives. Instructions pretending to be state should live in knowledge documents.

### 2.2.5 Unnecessary framework complexity

LangGraph's `StateGraph`, `Annotation.Root`, checkpointing, and channel reducers are overhead for what is now a linear pipeline.

## 2.3 Current File Inventory

| File | Purpose | Fate |
| --- | --- | --- |
| `chatbot-graph.ts` | StateGraph definition, edges, routing | Delete → `chatbot-pipeline.ts` |
| `chatbot-graph-standalone.ts` | Standalone graph variant | Delete |
| `chatbot-state.ts` | `Annotation.Root()` state schema | Simplify → plain TS interface |
| `agent-node.ts` | `buildAgentNodes()` generic builder | Delete (Vercel AI SDK handles) |
| `chatbot-tools.ts` | 30+ tool definitions, LangChain `tool()` | Rewrite → Vercel AI SDK `tool()` |
| `state-reminder.ts` | 4 separate state reminder builders | Unify → single facts-only builder |
| `agents/general-agent.ts` | General agent prompt + tools | Delete → `general.skill.ts` |
| `agents/match-agent.ts` | Match agent prompt + tools | Delete → `matchmaking.skill.ts` |
| `agents/onboarding-agent.ts` | Onboarding agent prompt + tools | Delete → `onboarding.skill.ts` |
| `agents/profile-improvement-agent.ts` | Profile agent prompt + tools | Delete → `profile.skill.ts` |
| `agents/yak-agent.ts` | Yak agent prompt + tools | Delete → `yak.skill.ts` |
| `agents/data-loader.ts` | loadProfileNode, loadMessagesNode | Simplify (no LangGraph wrappers) |
| `nodes/guard-node.ts` | maliciousGuard, emojiGuard | Simplify (plain functions) |
| `model/graph-dependencies.ts` | Service bundle (GraphDependencies interface) | Simplify → Dependencies interface |
| `chatbot.middleware.ts` | Entry point, calls `graph.invoke()` | Update → call `pipeline()` |

---

# 3. Proposed Architecture

## 3.1 Architecture Overview

The new architecture has four layers:

| Layer | What | How |
| --- | --- | --- |
| **Static Prompt** | Personality, product overview, skill metadata, special instructions | 4 markdown files assembled by `buildUnifiedPrompt`, prefix-cached |
| **Skills** | Domain knowledge + action tools, progressively disclosed via `loadSkill`  | Agent loads skills on demand; system gates action tools by state |
| **State Reminder** | Facts-only snapshot of user state, rebuilt every iteration | `buildUnifiedStateReminder` — no instructions, only facts |
| **Pipeline** | Linear execution: guard → load → streamText → send | Vercel AI SDK `streamText` with `maxSteps: 10` |

**Target pipeline:**

```tsx
async function runChatbotPipeline(input, deps) {
  await maliciousGuard(input);
  await emojiGuard(input);
  const profile = await loadProfile(input, deps);
  const messages = await loadMessages(input, deps);
  const skillMetadata = getSkillMetadata(profile.state, deps);

  const result = await streamText({
    model: provider('model-id'),    // swappable
    system: buildUnifiedPrompt(state, skillMetadata, deps),
    messages,
    tools: resolveTools(state, deps),  // core + loaded skill tools
    maxSteps: 10,
  });

  await sendResponse(result, deps);
}
```

## 3.2 Skills Architecture (Progressive Disclosure)

The skills system follows the **Agent Skills** open standard (Anthropic, 2025; adopted by OpenAI, LangChain, Vercel, Microsoft) — a progressive disclosure architecture where the agent decides which knowledge to load, while the system enforces safety constraints on action tools.

### 3.2.1 Design Principles

Two separate concerns require two separate mechanisms:

| Concern | Who Decides | Mechanism |
| --- | --- | --- |
| **Knowledge retrieval** | Agent (on demand) | `loadSkill(name)` tool — agent calls when it needs domain knowledge |
| **Action tool safety** | System (state-gated) | Skill registry filters tools based on user state before exposing them |

This separates knowledge discovery (agent judgment) from tool safety (system constraint). The agent sees skill metadata and decides what is relevant; the system ensures only safe tools are available.

### 3.2.2 Progressive Disclosure (3 Tiers)

**Tier 1 — Metadata (always in system prompt, ~100 tokens total)**

Skill name + one-line description for each available skill. Tells the agent what skills exist and when to load them. Cheap enough to include for all state-eligible skills every turn.

**Tier 2 — Knowledge (loaded on demand via `loadSkill`)**

Full markdown knowledge document. Only loaded when the agent determines it is relevant to the user's question. Saves tokens on turns where the user asks a simple question or just wants to chat.

**Tier 3 — Action tools (available after skill loaded, state-gated)**

Domain-specific action tools become available after the agent loads the skill. Still filtered by user state for safety (e.g. no `triggerContactExchange` pre-reveal, no `pauseUserAccount` if already paused).

### 3.2.3 Skill Interface

```tsx
interface Skill {
  name: string;                   // identifier for logging/tracing
  description: string;            // when to load (~1 line, shown in system prompt)
  knowledge: string;              // markdown doc (returned by loadSkill on demand)
  tools: Record<string, Tool>;    // action tools (state-gated)
}
```

### 3.2.4 How It Works

1. **System prompt** includes skill metadata for all available skills (state-filtered):

```
Available Skills (call loadSkill to activate):
- matchmaking: Match lifecycle, scheduling, reveals, contact exchange
- profile: Profile editing, photos, hobbies, improvement guidance
- account-mgmt: Pause, resume, deactivation flows
```

1. **Agent calls `loadSkill("matchmaking")`** when the user asks about their match — returns knowledge markdown, records in `loadedSkills` state, action tools become available on next iteration
2. **Agent uses action tools** (e.g. `getMatchInfo`) — informed by the loaded knowledge
3. **Core tools** always available without loading any skill:
    - `loadSkill(name)` — the progressive disclosure tool itself
    - `handOverToTeam` — escalation to human team
    - `noReply` — suppress response
    - `getLoginLink` — user's personal page link
    - `getDittoSocialMediaHandles` — Ditto social accounts
    - `searchHandbook(query)` — handbook search (always available as fallback)

### 3.2.5 Example: Matchmaking Skill

```tsx
// skills/matchmaking.skill.ts
export function getMatchmakingSkill(state, deps): Skill {
  const knowledge = readFileSync('knowledge-tools/matchmaking.md', 'utf-8');

  const tools = {
    getMatchInfo: tool({ ... }),
    getMatchPosterLink: tool({ ... }),
    getMatchScheduleLink: tool({ ... }),
    getMatchingHistory: tool({ ... }),
  };

  // Sub-gating: post-reveal tools only after reveal
  const preReveal = ['Matched', 'Making Poster', 'Poster Done'];
  if (!preReveal.includes(state.matchStatus)) {
    tools.markNotGoing = tool({ ... });
    tools.triggerContactExchange = tool({ ... });
  }

  tools.pauseMatchedUserAccount = tool({ ... });
  tools.userAccountDeactivation = tool({ ... });

  return {
    name: 'matchmaking',
    description: 'Match lifecycle, scheduling, reveals, contact exchange',
    knowledge,
    tools,
  };
}
```

### 3.2.6 Skill Registry

The registry exposes three functions:

```tsx
// skills/skill-registry.ts

/** Returns metadata for system prompt (state-filtered) */
function getSkillMetadata(state, deps): SkillMetadata[] { ... }

/** Returns core tools + action tools from loaded skills (state-gated) */
function getActiveTools(state, loadedSkills, deps): Tool[] { ... }

/** Loads a skill by name: returns knowledge, marks as loaded */
function loadSkill(name, state, deps): { knowledge: string } { ... }
```

Activation rules (which skills appear in metadata):

```tsx
function getSkillMetadata(state, deps) {
  const skills = ['general', 'account-mgmt', 'handbook'];  // always

  if (!state.completedOnboarding) {
    skills.push('onboarding');
    return skills;  // minimal skills for unonboarded users
  }

  skills.push('profile');
  if (state.currentStatus === 'Waiting') skills.push('waiting');
  if (state.matchId) skills.push('matchmaking');
  if (state.activePool === 'yik-yak') skills.push('yak');
  if (state.activePool !== 'yik-yak' && state.yakMatchId) skills.push('hybrid-yak');
  if (eventPools.includes(state.activePool)) skills.push('events');

  return skills;
}
```

### 3.2.7 Skills Inventory

| Skill | Knowledge Doc | Key Tools | Activation |
| --- | --- | --- | --- |
| **general** | [general-faq.md](http://general-faq.md) | handOverToTeam, noReply, getLoginLink | Always |
| **account-mgmt** | [account-management.md](http://account-management.md) | pause*, resume, deactivate | Always |
| **handbook** | (MongoDB search) | searchHandbook(query) | Always |
| **onboarding** | [onboarding.md](http://onboarding.md) | switchPool | !completedOnboarding |
| **profile** | [profile-improvement.md](http://profile-improvement.md) | updateProfile, updateImages | Onboarded |
| **matchmaking** | [matchmaking.md](http://matchmaking.md) | getMatchInfo, markNotGoing* | matchId exists |
| **yak** | [yak-event.md +](http://yak-event.md) [interactions.md](http://interactions.md) | revealMatch*, relayToMatch | activePool = yik-yak |
| **events** | Event-specific docs | Event-specific tools | Event pool active |
|  |  |  |  |

* = sub-gated within the skill based on additional state conditions*

## 3.3 Vercel AI SDK Integration

### 3.3.1 Why Vercel AI SDK

- **Model-agnostic**: provider is a one-line swap (anthropic, openai, google, etc.)
- **Built-in tool execution loop** via `streamText` + `maxSteps` — no manual callModel/tools loop
- **First-class NestJS support** via `pipeDataStreamToResponse`
- **OpenTelemetry-native** telemetry for multi-backend observability
- **Active ecosystem**: AI SDK 6 unifies `generateObject` and `generateText`

### 3.3.2 Tool Definition Migration

**Before (LangChain):**

```tsx
const getMatchInfo = tool(async (input) => {
  const data = await llmToolsService.getMatchInfo(state);
  return new Command({
    update: { shouldRespond: true },
    resume: new ToolMessage({ content: JSON.stringify(data), tool_call_id })
  });
}, { name: 'getMatchInfo', schema: z.object({}) });
```

**After (Vercel AI SDK):**

```tsx
import { tool } from 'ai';
const getMatchInfo = tool({
  description: 'Get match details for the current user',
  parameters: z.object({}),
  execute: async () => {
    return await llmToolsService.getMatchInfo(state);
  },
});
```

### 3.3.3 Response Handling

Vercel AI SDK's `streamText` returns a result object. How we handle this depends on the chatbot's response pattern — whether we stream to the client, buffer the full response, or send via a messaging channel (e.g. webhook-based). The `sendResponse` function will consume the result and handle chunk splitting, quiet-check, and pending review flow as it does today.

### 3.3.4 Migration Notes

- `Command` objects with `shouldRespond` + `ToolMessage` → reimplemented as post-processing after the tool loop completes
- `requireToolCallId()` — Vercel AI SDK manages tool call IDs internally; verify no conflicts
- Prompt caching — Vercel AI SDK supports provider-level prompt caching; verify cache hit rate post-migration
- Model provider is a one-line config change — zero pipeline code changes required to swap providers

## 3.4 Observability: Multi-Backend via OpenTelemetry

Vercel AI SDK exposes OpenTelemetry-compatible telemetry. We instrument once and export traces to multiple backends simultaneously for evaluation during a trial period.

### 3.4.1 Instrumentation

```tsx
// instrumentation.ts
import { registerOTel } from '@vercel/otel';

registerOTel({
  serviceName: 'ditto-chatbot',
  traceExporter: exporter, // LangSmith, Raindrop, Braintrust, etc.
});
```

### 3.4.2 Backends Under Evaluation

| Tool | Strength | Integration |
| --- | --- | --- |
| **LangSmith** | Trace visualization, prompt playground, cost tracking | `langsmith/vercel` OTel exporter (native) |
| **Raindrop** | Unified observability platform, team adoption target | OTel collector / custom exporter (TBD) |
| **Braintrust** | Evals, scoring, dataset management, A/B testing | OTel-compatible (verify ingestion) |
| **Confident AI** | Hallucination detection, eval frameworks | API-based logging |

### 3.4.3 Key Metrics

- Token usage per conversation (input / output / cached)
- Cost per conversation (derived from token counts + model pricing)
- Tool/skill call frequency — which skills are activated most
- Step count per invocation (tool loops before final response)
- Latency — time to first token, total response time
- Escalation rate — `handOverToTeam` calls as % of total conversations
- Prefix cache hit rate — verify static prompt caching is working

## 3.5 Self-Learning Handbook

A MongoDB collection that grows over time, reducing `handOverToTeam` escalations.

### 3.5.1 Schema

```tsx
interface HandbookEntry {
  question: string;
  answer: string;
  tags: string[];
  category: 'matchmaking' | 'account' | 'billing' | ...;
  source: 'escalation' | 'manual';
  approvedBy?: string;
  createdAt: Date;
}
```

### 3.5.2 Flow

1. User asks a question the agent can't answer from knowledge tools
2. Agent calls `searchHandbook(query)` — finds nothing
3. Agent escalates via `handOverToTeam` with `suggestedHandbookEntry` in payload
4. Team reviews suggestion, edits if needed, approves
5. Next user with same question → `searchHandbook` returns answer → no escalation

*The escalation rate decreases automatically as the handbook grows.*

---

# 4. What Doesn't Change

| Component | Details |
| --- | --- |
| **Content moderation** | Qwen3Guard + regex guardrails, sensitive category handling. Runs before the agent, completely untouched. |
| **MongoDB state** | All user, match, profile, and conversation state stays in MongoDB. No migration. |
| **Prompt caching** | Static system prompt structure enables provider-level prompt caching. The new unified prompt preserves this pattern. |
| **sendResponse** | Quiet-check, chunk splitting for long messages, pending review flow. Unchanged. |
| **Max 10 tool iterations** | Same limit, now enforced by Vercel AI SDK `maxSteps: 10`. |
| **Feature flag rollout** | PostHog feature flag for gradual rollout: 5% → 25% → 50% → 100%. |

---

# 5. Implementation Plan

| # | Issue | Scope | Linear |
| --- | --- | --- | --- |
| 1 | Agent Skills refactor | Merge knowledge tools + action tools into self-contained skills with per-skill sub-gating | ENG-1515 |
| 2 | Remove LangGraph | Delete StateGraph, Annotation.Root, checkpointing. Replace with `chatbot-pipeline.ts` | ENG-1516 |
| 3 | Vercel AI SDK integration | Install `ai` package, rewrite tools to `tool()` format, wire `streamText`  • `maxSteps` | ENG-1513 |
| 4 | Multi-backend observability | Migrate to OpenTelemetry. Trial LangSmith, Raindrop, Braintrust, Confident AI simultaneously | ENG-1520 |
| 5 | Handbook backend | MongoDB collection, HandbookService, searchHandbook, enhanced handOverToTeam | ENG-1517 |

## 5.1 Dependencies

- Steps 1-3 are tightly coupled — skills, LangGraph removal, and Vercel AI SDK should be done together or in rapid sequence
- Step 4 (observability) can run in parallel with 1-3
- Step 5 (Handbook) is independent and can be done at any point

---

# 6. Current Progress

Before the pivot to the skills-based architecture described in this document, an intermediate refactor ("Knowledge-as-Tools") was partially implemented on feature branches. Several of these artifacts are reusable and will accelerate the current effort.

## 6.1 Completed Intermediate Work

| Artifact | Description | Reuse in New Plan |
| --- | --- | --- |
| `unified-agent.ts` | Single agent replacing 5 separate agents | Core concept reused; reimplemented on Vercel AI SDK |
| `build-unified-prompt.ts` | Assembles 4 static .md files into one system prompt with prefix caching | Directly reusable — prompt assembly logic stays |
| `unified-state-reminder.ts` | Single facts-only state reminder replacing 4 builders | Directly reusable — facts-only pattern preserved |
| `knowledge-tools/*.md` | 9 markdown knowledge documents (matchmaking, onboarding, profile, etc.) | Directly reusable — become skill knowledge docs |
| `knowledge-tools.ts` | Tool definitions that return knowledge markdown on demand | Replaced by `loadSkill` tool within skill-registry; knowledge docs stay as `.md` files read on demand |
| `tool-registry.ts` | Centralized tool gating by user state (136 lines) | Replaced by `skill-registry.ts` with per-skill activation |
| Simplified chatbot graph | Graph reduced from 20+ nodes to 7 linear nodes | Confirms linear pipeline is sufficient; graph deleted entirely |

## 6.2 What Changed

The original Knowledge-as-Tools plan kept LangGraph as the runtime and retained the LangChain tool/message patterns. Based on team feedback, three decisions changed the approach:

- **Knowledge tools alone are insufficient** — bundling knowledge + action tools into skills gives the agent a more coherent mental model per domain
- **LangGraph is unnecessary** — with conditional routing (`routeToAgent`) eliminated, the graph is linear. A plain async function replaces `StateGraph`
- **Vercel AI SDK replaces all `@langchain/*` dependencies** — model-agnostic providers, native tool execution loop, OpenTelemetry-native telemetry

*The intermediate work de-risks the pivot: the unified prompt, knowledge documents, and facts-only state reminder are directly reusable. The primary new work is the skills layer, Vercel AI SDK integration, and observability migration.*

---

## 6.3 Skills Refactor — Completed (2026-03-31)

Branch: `refactor/chatbot-unify-agents` | Commit: `6dbf9766` | Linear: ENG-1515

### What was built

- 8 skill files in `src/chatbot/skills/` (general, account-mgmt, onboarding, profile, matchmaking, yak, hybrid-yak, events)
- `skill-registry.ts` with `getSkillMetadata()`, `getActiveTools()`, and `loadSkill` tool
- `loadSkill(name)` tool for progressive knowledge disclosure (returns .md content on demand)
- Dynamic tool resolution in `agent-node.ts` — `getTools(state)` replaces static `tools` array
- Skill metadata injected into system prompt via Mustache in `build-unified-prompt.ts`
- Parity test: 15 test cases verifying tool selection matches old `tool-registry.ts` for all user states
- Deleted: `tool-registry.ts`, `knowledge-tools.ts` (9 knowledge tools → 1 `loadSkill`)

### Key decision: action tools always available, knowledge gated

We initially tried full progressive disclosure (agent must call `loadSkill` to unlock both knowledge AND action tools). During testing, the agent

saw "account-mgmt: pause, resume, deactivation" in the metadata description and responded conversationally saying it paused the account — but never
called `loadSkill` and never called `pauseUserAccount`. It faked the action.

**Root cause:** The SOTA Agent Skills pattern (Anthropic, 2025) was designed for coding agents loading instructions/knowledge. For our chatbot,

action tools that MUST be executed can't be gated behind an extra step the agent might skip.

**Final design:**

- **Action tools**: always available, state-gated by user status (same reliability as old system)
- **Knowledge**: agent calls `loadSkill(name)` when it needs operational details (progressive disclosure)
- **Metadata**: skill names + descriptions in system prompt (~100 tokens)

### Parity fixes found during testing

- **Bug fix**: Unonboarded users in event pools (e.g. NYC Gala) now get event tools — was missing in `tool-registry.ts` but present in the old
5-agent onboarding agent
- **Gating fix**: `account-mgmt` tools restricted to onboarded users only (unonboarded users shouldn't pause/deactivate)

### Token cost analysis

- Production (dev, per-agent prompts): ~6.3K input tokens
- Refactor (unified prompt): ~7.8K input tokens
- The ~1.5K increase is from the unified agent approach itself (one prompt covering all domains), not from the skills refactor
- `product-overview.md` (502 words) is the biggest contributor — didn't exist in per-agent prompts
- Skills refactor reduced tool count: 25 tools (9 knowledge + 16 action) → 17 tools (1 loadSkill + 16 action)

### Prompt behavior fix

- Agent kept repeating "matching isn't live at your school" every turn

# 7. Risks and Mitigations

| Risk | Severity | Mitigation |
| --- | --- | --- |
| Tool behavior regression | **High** | Comprehensive test suite for all 30+ tools before/after migration. Feature flag for gradual rollout. |
| Prompt caching breaks | Medium | Verify cache hit rate via provider dashboard immediately post-deploy. Static prompt structure preserved. |
| Vercel AI SDK limitations | Medium | SDK is actively maintained (v6). Fallback: use provider SDK directly with custom tool loop. |
| Observability gap during migration | Low | Keep LangSmith running during transition. OTel supports multiple exporters — all backends receive traces from day one. |
| Handbook cold start | Low | Seed initial entries from existing FAQ + common escalation patterns. |

---

# 8. Acceptance Criteria

- [ ]  Zero `@langchain` or `langsmith` imports in `src/chatbot/`
- [ ]  Agent runs via Vercel AI SDK `streamText` with `maxSteps: 10`
- [ ]  Skills bundle knowledge + action tools; agent activates by context
- [ ]  All 30+ existing tool functions preserved with no regression
- [ ]  Content moderation pipeline untouched
- [ ]  Observability backends receiving full traces via OpenTelemetry
- [ ]  Prefix cache hit rate maintained or improved
- [ ]  Feature-flagged rollout via PostHog (5% → 25% → 50% → 100%)
- [ ]  `searchHandbook` returns real results from MongoDB
- [ ]  Adding a new skill requires only one new file + one line in registry

---

# 9. References

## Internal

(These are my local files)

- `REFACTOR-EXECUTION-SUMMARY.md` — Full technical specification from Knowledge-as-Tools phase
- `LINEAR-ISSUES.md` — Issue descriptions for all sub-issues
- `Knowledge-as-Tools-Explainer.md` —  Explainer with SOTA references
- ENG-1518 (Linear) — Parent epic with all sub-issues

## External

- [Vercel AI SDK docs](https://sdk.vercel.ai/docs)
- [Vercel AI SDK NestJS example](https://sdk.vercel.ai/examples/api-servers/nest)
- [Vercel AI SDK Agents](https://sdk.vercel.ai/docs/ai-sdk-core/agents)
- [Vercel AI SDK providers](https://sdk.vercel.ai/docs/ai-sdk-core/providers-and-models)
- [OpenTelemetry + Vercel AI SDK](https://vercel.com/docs/observability/otel-overview)
- [LangSmith Vercel exporter](https://docs.smith.langchain.com/integrations/vercel)

## Agent Skills (SOTA)

- [Anthropic Agent Skills Overview](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/overview)
- [Agent Skills Specification](https://agentskills.io/specification)
- [Equipping Agents for the Real World (Anthropic Engineering)](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Vercel Agent Skills FAQ](https://vercel.com/blog/agent-skills-explained-an-faq)
- [Microsoft Agent Framework Skills](https://learn.microsoft.com/en-us/agent-framework/agents/skills)

Notes: 
- Make the skills metadata more rich. mention tools that are relevant to the skills.