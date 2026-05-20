# Claude Code governance — summary

How AI assistants (Claude Code) should work across Ditto repos—and in the future Nx monorepo.

## Principles

- **Per-app `CLAUDE.md`** beats one giant template; otel’s short gotcha list is the model for small services.  
- **Root monorepo `CLAUDE.md`** (~120 lines): commands, graph of apps, cross-cutting rules only.  
- **`.cursor/rules` or `.claude/rules`** for enforceable patterns (imports, testing, no secrets).  

## Gaps today

Eight repos had **no** CLAUDE.md (e.g. `delayed-task-service`, `proj-coach-ws`, `proj-coach-voip`, `matchmake_experimentation`). Governance doc recommends minimal files: how to run, test, deploy, and 3–5 gotchas.

## Monorepo mapping (target)

| Current | Target path |
|---------|-------------|
| `proj-coach-backend/CLAUDE.md` | `apps/proj-coach-backend/CLAUDE.md` |
| `ditto-internal-frontend/CLAUDE.md` | `apps/ditto-internal-frontend/CLAUDE.md` |
| `otel/CLAUDE.md` | `apps/otel-collector/CLAUDE.md` |

## Skills to add later

- `/add-skill` scaffolding once chatbot skills architecture stabilizes  
- Shared rule: never commit Infisical tokens, LangSmith keys, or `.tfvars`  

*Source: internal Claude Code Governance in the Nx Monorepo doc.*
