# Matchmaking 3.x — technical review notes

Condensed feedback from engineering review of the 3.x strategy draft. Use when aligning roadmap with buildable increments.

## Themes raised in review

| Theme | Summary |
|-------|---------|
| **Sequencing** | Prefer phased delivery (see [implementation-plan-summary.md](implementation-plan-summary.md)) over simultaneous multi-head + learning stack |
| **Label quality** | Downstream outcomes are sparse; define proxy hierarchy before training heads |
| **Operational load** | Coach review and priority tiers must scale with automation—link to ENG-1030 |
| **Retrieval vs decide** | Keep Meili/embedding retrieval separate from head calibration |
| **Governance** | Agent skills need versioned rollout and rollback like prompt-manager |
| **Metrics** | Per-head calibration dashboards before policy plugins go fully automatic |

## Open questions (typical)

- Minimum sample size per school/pool before head-specific models activate  
- How chemistry head avoids rewarding novelty without safety constraints  
- Cold-start users: readiness head vs hard pool rules  

## Action items pattern

1. Ship instrumentation (phase 0) in production matching path.  
2. Document decision trace schema in `proj-coach-schemas` / backend.  
3. Run shadow scoring for new heads before changing live ranking.  

## Related

- [engine-3x-summary.md](engine-3x-summary.md)  
- [../../docs/strategy/q2-roadmap.md](../../docs/strategy/q2-roadmap.md)  

*Synthesized from Matchmaking 3.x — Technical Review Comments source; not a verbatim comment thread.*
