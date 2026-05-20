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

## Phased integration (aligned with implementation plan)

| Phase | OpenClaw role |
|-------|----------------|
| 0 | Read decision traces; no automation |
| 1 | Match reviewer agent + coach review workflow |
| 6+ | Policy skills on calibrated head outputs |

Detail: [../../reference/matchmaking/implementation-plan-summary.md](../../reference/matchmaking/implementation-plan-summary.md).

## Runtime touchpoints

- **Restate** — scoring rounds, autofilter (`profile-analysis-service`)  
- **MongoDB** — profiles, matches, outcomes  
- **Internal dashboard** — human approval before production pairs ship  

## Production boundary

Agents may **recommend**; production matching still flows through audited pipelines and ops tools unless explicitly approved for automation.

*Synthesized from OpenClaw Deep Technical Guide source; details trimmed for hub navigation—refer to engineering source for full API examples.*
