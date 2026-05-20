# OpenClaw — matchmaking integration

Technical guide for integrating external agent frameworks (OpenClaw) with Ditto matchmaking concepts.

## Purpose

Explores how external **agentic** workflows could read Ditto user state, propose pairs, or simulate outcomes without replacing core `proj-coach-backend` matching authority in production.

## Key integration points (conceptual)

| Ditto surface | Use |
|---------------|-----|
| User profile + User Memory (future) | Features for agent reasoning |
| Match scores / AI match endpoints | Candidate retrieval |
| Exclusion rules | Pair constraints |
| Human approval | Internal dashboard still gates production matches |

## Alignment with Matchmaking 3.x

OpenClaw experiments should respect multi-head semantics (likelihood, intensity, chemistry, readiness)—see [../../reference/matchmaking/engine-3x-summary.md](../../reference/matchmaking/engine-3x-summary.md).

## Production boundary

Agents may **recommend**; production matching still flows through audited pipelines and ops tools unless explicitly approved for automation.

*Synthesized from OpenClaw Deep Technical Guide source; details trimmed for hub navigation—refer to engineering source for full API examples.*
