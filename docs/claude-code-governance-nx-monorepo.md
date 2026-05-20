# Claude Code Governance in the Nx Monorepo

| Field | Value |
| --- | --- |
| **Author** | Ryan Willis / Ditto Engineering |
| **Date** | April 2026 |
| **Status** | Companion to RFC: Monorepo-Driven Development |
| **Team** | Engineering |

<aside>
📋

This is a practical guide for governing Claude Code (and Cursor) in the Ditto monorepo. It covers what to commit to the repo, what stays personal, and what's not worth overthinking. It's written for the team we have today — not a hypothetical perfectly-disciplined 50-person org.

</aside>

---

# 1. Where We Are Today

Before proposing changes, let's be honest about the current state.

**CLAUDE.md coverage**: 7 of 18 repos (39%). The files that exist are good but completely inconsistent:

| Repo | Lines | Style | Focus |
| --- | --- | --- | --- |
| proj-coach-backend | 107 | Architecture + commands | NestJS modules, config system |
| imsg-service | 77 | Architecture + conventions | Message flow, DI, providers |
| prompt-manager-backend | 241 | Architecture + testing patterns | Detailed mock patterns, test setup |
| ditto-internal-frontend | 318 | Everything | State mgmt, API layer, prompt manager internals |
| otel | 45 | Brief + gotchas | Build commands, deployment pitfalls |
| event-202603-yik-yak | 96 | Workflow protocol | STAR framework, file protocol, autonomy rules |
| ufl | 182 | Workflow protocol | Information barriers, MongoDB pipeline |

**Settings/hooks in use**: Only 3 repos have `.claude/` configs, all doing different things:
- `prompt-manager-backend` — hooks for lint-on-save and test-before-PR
- `ditto-internal-frontend` — permission allowlists (pnpm, git, MCP tools)
- `ufl` — minimal permissions (git, uv, curl)

**What's missing from 8 repos**: `delayed-task-service`, `ditto-internal-ws`, `marketing-internal-tool`, `proj-coach-protos`, `proj-coach-voip`, `proj-coach-ws`, `matchmake_experimentation`, `claude-marketplace` — no CLAUDE.md at all. AI tools working in these repos start blind.

**Memory**: Entirely per-engineer, per-repo. No sharing. When someone learns that "profile-analysis-service needs CLIP warm-up before scoring" or "never use `findOneAndUpdate` without `{ new: true }` in the match module", that knowledge dies with their local memory. Next engineer hitting the same issue re-discovers it from scratch.

---

# 2. What the Monorepo Changes

Moving to a single Nx monorepo changes three things for AI tooling:

**1. One project = one memory.** Today Claude Code maintains separate memory per repo. In the monorepo, all context — backend patterns, iMessage quirks, schema conventions, deployment gotchas — lives in a single project memory. When you work on the iMessage service, Claude still remembers what it learned about the backend yesterday. This happens automatically.

**2. CLAUDE.md hierarchy becomes real.** Claude Code walks up the directory tree loading CLAUDE.md files. In separate repos, this just loads one file. In the monorepo, working in `apps/backend/src/chatbot/` loads:
- Root `CLAUDE.md` (architecture overview, cross-cutting rules)
- `apps/backend/CLAUDE.md` (NestJS patterns, backend commands)
- Subdirectory CLAUDE.md files load lazily when Claude reads files there

**3. One `.claude/` directory for the whole team.** Settings, hooks, skills, and MCP configs committed once, used by everyone.

---

# 3. CLAUDE.md — What to Actually Write

### Don't aim for perfection

The existing CLAUDE.md files work because they're written by the people who know the code, for the AI that's working in it. They're inconsistent — and that's fine. A 45-line file that captures the otel-collector's gotchas is better than a 300-line template-following file that's generic.

### Root CLAUDE.md

This is the only new CLAUDE.md we need to write carefully. It loads for every conversation in the monorepo. Keep it **under 150 lines** — Claude Code loads the first ~200 lines of CLAUDE.md at session start, and shorter files get better adherence.

What belongs in root CLAUDE.md:

```markdown
# Ditto Monorepo

## Architecture
- Nx monorepo with `apps/`, `libs/`, `packages/`, `tools/`, `ai/`, `docs/`
- Primary backend: apps/backend (NestJS, 48 modules, Bun runtime)
- Shared schemas: libs/schemas (79 Mongoose schemas, workspace dependency)
- Runtime: Bun for TypeScript, Python 3.12 for tools/, Go for otel-collector

## Key Commands
- `nx run <app>:build` — Build a specific app
- `nx run <app>:test` — Test a specific app
- `nx affected --target=test` — Test only what changed
- `nx graph` — Visualize dependency graph
- `bun install` — Install all workspace dependencies (run from root)

## Conventions
- TypeScript strict mode, ESLint enforced
- Mongoose schemas in libs/schemas/ — never duplicate schema definitions in apps
- Config via TOML files (apps/*/config/config.{env}.toml)
- Branch naming: ENG-XXXX/description
- Commit format: conventional commits (feat:, fix:, chore:, etc.)

## Cross-Service Patterns
- Services communicate via RabbitMQ (exchanges: be, delayed-tasks, sockets)
- Auth via gRPC (protos in libs/protos/)
- Shared types go in libs/shared/, not duplicated across apps
- Schema changes in libs/schemas/ affect: backend, imsg-service,
  profile-analysis, delayed-tasks — always run `nx affected` after changes

## What NOT to Do
- Don't install dependencies in app-level package.json if they exist at root
- Don't modify libs/schemas/ without checking consumers: `nx affected --target=build`
- Don't add new Mongoose schemas outside libs/schemas/
- Don't hardcode environment-specific values — use TOML config
```

### Per-App CLAUDE.md Files

**Migrate existing files as-is.** The 7 existing CLAUDE.md files should move into the monorepo with minimal changes:
- `proj-coach-backend/CLAUDE.md` → `apps/backend/CLAUDE.md` (update paths)
- `imsg-service/CLAUDE.md` → `apps/imsg-service/CLAUDE.md`
- `prompt-manager-backend/CLAUDE.md` → `apps/prompt-manager/CLAUDE.md`
- `ditto-internal-frontend/CLAUDE.md` → `apps/internal-frontend/CLAUDE.md`
- `otel/CLAUDE.md` → `apps/otel-collector/CLAUDE.md`

**Don't write CLAUDE.md for the 8 missing repos on day 1.** It's better to let engineers write them as they work in those directories. A CLAUDE.md written by someone who just moved files is worse than none at all. The root CLAUDE.md covers the basics. App-specific files will grow organically.

### Path-Scoped Rules (`.claude/rules/*.md`)

Claude Code supports rules files with path-based frontmatter — they only load when Claude works with matching files. This is useful for a monorepo where different areas have different conventions:

```
.claude/rules/
├── schemas.md          # Rules for libs/schemas/**
├── chatbot.md          # Rules for apps/backend/src/chatbot/**
├── ai-prompts.md       # Rules for ai/prompts/**
└── testing.md          # Rules for **/*.spec.ts, **/*.test.ts
```

Example — `schemas.md`:

```markdown
---
paths:
  - "libs/schemas/**"
---

# Schema Rules

- All schemas must include `timestamps: true` option
- Schema names are PascalCase (UserProfile, not userProfile)
- Never modify existing field types — add a new field and deprecate the old one
- Always export from libs/schemas/lib/index.ts
- After any change, verify consumers: `nx affected --target=build`
```

Example — `ai-prompts.md`:

```markdown
---
paths:
  - "ai/prompts/**"
  - "ai/evals/**"
---

# AI Prompt Rules

- Prompt files are version-controlled markdown — treat changes like code changes
- Every prompt change should ideally have eval results in the PR description
- Don't modify production prompts without reviewing current eval baselines
- Playbook entries in ai/playbook/ should reference the escalation they resolved
```

**Start with 2-3 rules files**, not 10. Add more as the team discovers patterns that keep getting violated.

---

# 4. Settings — What to Commit vs. Keep Personal

### Committed: `.claude/settings.json`

This is the team-wide baseline. Keep it minimal — overloaded settings get overridden by frustrated engineers via `settings.local.json`.

```json
{
  "env": {
    "NX_DAEMON": "true",
    "NODE_ENV": "development"
  },
  "permissions": {
    "deny": [
      "Edit(.env*)",
      "Edit(.claude/settings.json)",
      "Bash(rm -rf *)",
      "Bash(git push --force*)",
      "Bash(kubectl delete namespace prod*)",
      "Bash(kubectl delete namespace staging*)"
    ]
  }
}
```

That's it. The deny list prevents genuinely dangerous operations. Everything else is allowed by default. Don't try to allowlist every command — it creates friction and engineers will bypass it.

**Why no `allow` list**: The existing repos show different allowlists (pnpm in frontend, uv in ufl, bun in backend). In the monorepo, engineers work across apps with different toolchains. A restrictive allowlist breaks someone's workflow. Deny the dangerous stuff, allow everything else.

### Personal: `.claude/settings.local.json` (gitignored)

Each engineer customizes their own setup. Some realistic examples:

**Backend engineer who wants lint-on-save** (inspired by prompt-manager-backend's current hooks):

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "cd \"$CLAUDE_PROJECT_DIR\" && bun lint --fix --quiet 2>/dev/null || true",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

**Engineer who wants test-before-commit**:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "if": "Bash(git commit*)",
        "hooks": [
          {
            "type": "command",
            "command": "cd \"$CLAUDE_PROJECT_DIR\" && nx affected --target=test --base=HEAD~1 2>&1 | tail -5",
            "timeout": 60000,
            "statusMessage": "Running affected tests..."
          }
        ]
      }
    ]
  }
}
```

**Engineer who uses MCP tools** (Linear, Playwright, etc.):

```json
{
  "permissions": {
    "allow": [
      "mcp__linear__*",
      "mcp__playwright__*",
      "mcp__exa__*"
    ]
  }
}
```

### Why hooks stay personal (for now)

The prompt-manager-backend already has team-committed hooks (lint-on-save, test-before-PR). These work for a single-service repo where everyone runs the same commands. In the monorepo, different people work in different areas:
- A frontend engineer doesn't need `bun lint` running on every edit — they need `pnpm lint`
- A Python data engineer doesn't need any JavaScript linting
- Slow hooks (test runners) on every commit will get disabled immediately

**Pragmatic approach**: Share hook recipes in a `docs/claude-code-hooks.md` file. Engineers opt-in by copying what they want into their `settings.local.json`. If a hook proves universally useful after a month, promote it to the committed `settings.json`.

---

# 5. Skills (Custom Slash Commands)

Skills live in `.claude/skills/` and are available to every engineer. These should solve real, repeated workflows — not hypothetical ones.

### Start with what we actually need

```
.claude/skills/
├── test/
│   └── SKILL.md          # /test — run affected tests
├── lint-fix/
│   └── SKILL.md          # /lint-fix — fix lint across affected packages
└── add-schema/
    └── SKILL.md          # /add-schema — scaffold a new Mongoose schema
```

**`/test`** — The most universally useful command:

```markdown
---
name: test
description: Run tests for affected packages based on recent changes
allowed-tools: Bash, Read, Grep
---

Run the affected tests using Nx. Steps:

1. Run `nx affected --target=test --base=HEAD~1` to test packages affected by recent changes
2. If tests fail, read the failing test files and the source files they test
3. Summarize: which tests passed, which failed, and for failures explain the likely cause
4. If all tests pass, just say "All affected tests pass."
```

**`/add-schema`** — Prevents the common mistake of defining schemas in the wrong place:

```markdown
---
name: add-schema
description: Scaffold a new Mongoose schema in libs/schemas following conventions
allowed-tools: Read, Write, Edit, Glob, Grep
argument-hint: "<SchemaName>"
---

Create a new Mongoose schema in the shared schemas library. The schema name is: $ARGUMENTS

Steps:
1. Read `libs/schemas/lib/index.ts` to understand the export pattern
2. Read 1-2 existing schema files in `libs/schemas/lib/` to match the conventions
3. Create the new schema file at `libs/schemas/lib/$0.schema.ts` following the same patterns:
   - Include `timestamps: true`
   - Use PascalCase for the schema name
   - Export both the schema and the TypeScript interface
4. Add the export to `libs/schemas/lib/index.ts`
5. Run `nx run schemas:build` to verify it compiles
6. Run `nx affected --target=build` to check no consumers break
```

**Don't create skills for things people do once a month.** Skills should automate the annoying-but-frequent stuff. Add more only when someone actually asks "I wish Claude could just do X."

### Skills we might add later (but not on day 1)

- `/deploy-slot` — Create a k3s testing namespace (once the slot system exists)
- `/review` — Code review with Ditto conventions (once we agree on what those are)
- `/add-skill` — Scaffold a new chatbot skill (once the skills architecture stabilizes after ENG-1518)

---

# 6. Memory — What's Real and What's Not

### How memory actually works

Claude Code's memory is **local to each engineer's machine** at `~/.claude/projects/{project-hash}/memory/`. It's not shared, not committed to the repo, not synced anywhere. When Claude learns something during a conversation, it saves a markdown file to this directory. Next conversation, it loads `MEMORY.md` (the index) and reads topic files on demand.

In the monorepo, all apps share one project hash = one memory directory. This is the key win: Claude's memory about backend patterns is available when working on the iMessage service.

### What memory is good for

Memory works best for things that are:
- **Specific to how you work** — "Ryan prefers single bundled PRs for cross-service changes"
- **Discovered through trial and error** — "The profile-analysis tests require `REPLICATE_API_TOKEN` to be set even in test mode"
- **Contextual and evolving** — "We're in the middle of migrating from LangGraph to Vercel AI SDK (ENG-1518), don't add LangGraph code"

### What memory is NOT good for

Memory is the wrong place for:
- **Stable conventions** — Put those in CLAUDE.md (they apply to everyone, not just you)
- **Architecture docs** — Put those in `docs/` (they outlive any AI session)
- **Build commands** — Put those in CLAUDE.md (everyone needs them)

### The "shared knowledge" problem

The biggest limitation: **there's no built-in way to share memory across engineers**. When Nathan discovers that "the delayed-task-service silently drops messages if the RabbitMQ exchange doesn't exist", only Nathan's Claude remembers.

**Realistic solutions (pick one, don't do all three)**:

**Option A — Let CLAUDE.md absorb it (recommended)**

When someone discovers a non-obvious pattern, add it to the relevant CLAUDE.md or rules file via PR. This is the simplest approach and it's how the best existing CLAUDE.md files were built — the otel file's "gotchas" section, the prompt-manager's testing patterns.

The process:
1. Engineer hits a non-obvious issue
2. Claude saves it to personal memory (automatic)
3. If it's useful for others, engineer adds it to the appropriate CLAUDE.md or `.claude/rules/*.md` via PR
4. Team reviews and merges
5. Now every engineer's AI knows it

**Option B — A knowledge file that's not CLAUDE.md**

Create `docs/engineering-knowledge.md` — a living document of discovered patterns, gotchas, and tribal knowledge. Reference it from root CLAUDE.md:

```markdown
## Team Knowledge
For non-obvious patterns and gotchas, see docs/engineering-knowledge.md
```

This is less structured than CLAUDE.md but easier to add to (lower bar for contribution).

**Option C — Do nothing**

Honestly? The monorepo already solves 80% of the memory fragmentation problem by unifying into one project. Personal memory in a monorepo is dramatically more useful than personal memory across 18 repos. The remaining 20% (cross-engineer sharing) can wait until the team actually feels the pain.

### What we'll do

**Start with Option A.** When the root CLAUDE.md and rules files exist, engineers will naturally promote discoveries into them. If the CLAUDE.md files get too long (>200 lines), split into more specific rules files. This doesn't require any new process — just a team norm: "if you hit a gotcha, add it to the relevant CLAUDE.md."

---

# 7. MCP Servers

MCP servers are configured in `.claude/.mcp.json` (not `settings.json`). Some are team-wide, some are personal.

### Committed: `.claude/.mcp.json`

Only include MCP servers that the entire team benefits from and that don't require personal credentials:

```json
{
  "mcpServers": {
    "linear": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-linear"],
      "env": {
        "LINEAR_API_KEY": "${LINEAR_API_KEY}"
      }
    }
  }
}
```

**Note**: The `${LINEAR_API_KEY}` expands from each engineer's environment. The config is shared, but the credential stays personal.

### Personal MCP servers

Some engineers use Playwright for testing, Exa for code search, or custom MCPs. These go in `~/.claude/.mcp.json` (user-scoped) — not committed.

### Don't overload MCP on day 1

The existing repos show that only 2 engineers actively use MCP tools (Linear, Playwright, Exa references in the frontend settings). Don't mandate MCP adoption. Let it grow organically as people find tools that help.

---

# 8. What This Looks Like in Practice

### Committed to the monorepo (shared by everyone)

```
ditto/
├── CLAUDE.md                           # ~120 lines, architecture + commands + conventions
├── .claude/
│   ├── settings.json                   # Deny list only (dangerous commands)
│   ├── .mcp.json                       # Team MCP servers (Linear)
│   ├── rules/
│   │   ├── schemas.md                  # Rules for libs/schemas/**
│   │   └── ai-prompts.md              # Rules for ai/prompts/**, ai/evals/**
│   └── skills/
│       ├── test/SKILL.md              # /test
│       ├── lint-fix/SKILL.md          # /lint-fix
│       └── add-schema/SKILL.md        # /add-schema
├── apps/
│   ├── backend/CLAUDE.md              # Migrated from proj-coach-backend
│   ├── imsg-service/CLAUDE.md         # Migrated from imsg-service
│   ├── prompt-manager/CLAUDE.md       # Migrated from prompt-manager-backend
│   ├── internal-frontend/CLAUDE.md    # Migrated from ditto-internal-frontend
│   └── otel-collector/CLAUDE.md       # Migrated from otel
└── ...
```

### Personal to each engineer (not committed)

```
ditto/
└── .claude/
    └── settings.local.json             # Personal hooks, extra permissions, MCP tools

~/.claude/
├── settings.json                       # User-wide preferences (model, theme)
├── .mcp.json                          # Personal MCP servers (Playwright, Exa)
└── projects/{hash}/memory/            # Auto-managed by Claude Code
    ├── MEMORY.md
    ├── project_chatbot_refactor.md
    ├── feedback_testing_patterns.md
    └── ...
```

---

# 9. Migration Checklist

These are ordered by impact. Do the top 3 on migration day. The rest can wait.

| Priority | Task | Effort | When |
| --- | --- | --- | --- |
| **P0** | Write root `CLAUDE.md` (~120 lines) | 1 hour | Migration day |
| **P0** | Move existing CLAUDE.md files into monorepo (update paths) | 30 min | Migration day |
| **P0** | Create `.claude/settings.json` with deny list | 10 min | Migration day |
| **P1** | Create `/test` and `/add-schema` skills | 30 min | Migration week |
| **P1** | Create `schemas.md` rules file | 15 min | Migration week |
| **P2** | Create `ai-prompts.md` rules file | 15 min | When ai/ directory is populated |
| **P2** | Add `.claude/.mcp.json` with Linear server | 10 min | When team agrees on MCP tools |
| **P3** | Write CLAUDE.md for apps that don't have one | Varies | As engineers work in those areas |
| **P3** | Document hook recipes in `docs/claude-code-hooks.md` | 1 hour | After 2 weeks of monorepo use |

---

# 10. What We're NOT Doing

Things that sound good in theory but aren't worth the effort right now:

**Formal role-based access control.** Claude Code doesn't have built-in RBAC. You can approximate it with different `settings.local.json` per role, but there's no enforcement — anyone can edit their local settings. If someone wants to do something dangerous, they'll find a way. Trust the team, deny the obviously dangerous commands, and move on.

**Mandatory hooks in committed settings.** Hooks that slow people down will get overridden via `settings.local.json` within a week. Start with zero committed hooks. Let the team discover which hooks they want and vote with their feet.

**A formal memory seed system.** The previous version of this document proposed committed "memory seeds" with PR review, lifecycle management, and audit cycles. That's overengineered. CLAUDE.md files are the shared knowledge layer. Personal memory is personal. If the team outgrows this in 6 months, revisit.

**CLAUDE.md for every directory.** 7 of 18 repos currently have CLAUDE.md. Aim for the important ones (backend, schemas, frontend, AI directory) and let the rest grow organically. Nobody reads or maintains documentation written just to fill a checkbox.

**Standardizing CLAUDE.md format.** The existing files work because they're tailored to their codebase. The 45-line otel file and the 318-line frontend file serve different needs. Don't force a template. The only convention: start with key commands, then architecture, then conventions.

---

# 11. How to Know It's Working

After 2 weeks of monorepo use, check these signals:

| Signal | Healthy | Unhealthy |
| --- | --- | --- |
| Engineers using Claude Code across services | Claude makes cross-service changes in single sessions | Claude still confused about service boundaries, can't find schemas |
| Root CLAUDE.md | Referenced and occasionally updated via PRs | Ignored or stale (last touched on migration day) |
| Skills usage | `/test` used frequently, engineers requesting new skills | Nobody uses skills, everyone types commands manually |
| Settings overrides | Engineers adding personal hooks in `settings.local.json` | Engineers disabling committed settings because they're too restrictive |
| Memory | Claude remembers cross-service patterns between sessions | Claude keeps re-asking the same questions about architecture |
| Rules files | 2-4 focused rules files, occasionally updated | 0 (never created) or 15+ (over-documented, never read) |

If things are unhealthy, the fix is almost always "simplify" — not "add more rules."
