# Matchmaking Engine 3.x — executive summary

**Status:** Internal strategy draft (Notion export).  
**Intent:** Move from single similarity score to a **stateful, multi-head, two-sided** decision system.

## Problem framing

Campus matchmaking has **sparse, delayed, noisy, partially observed** feedback. Optimizing likes alone misaligns with real outcomes (dates, chemistry, continuation). Similarity ranking collapses four distinct questions into one scalar.

## Four decision heads

| Head | Question |
|------|----------|
| **Likelihood** | Can this pair realistically work (mutual viability)? |
| **Intensity** | If it works, how strong will engagement be? |
| **Chemistry** | Is there upside beyond obvious overlap (complementarity)? |
| **Readiness** | Is now the right moment for both users? |

New heads may be added when a dimension can be labeled and evaluated independently.

## Five pillars (architecture)

1. **Hard constraints** — eligibility, safety, age/gender/distance, exclusions  
2. **Shared representation + retrieval** — tractable candidate search (e.g. Meili + embeddings)  
3. **Temporal user state** — taste, receptivity, recent outcomes change over weeks  
4. **Calibrated heads + agent skills** — modular policy plugins on top of model outputs  
5. **Learning layer** — downstream outcomes feed retraining and diagnostics  

## Product semantics preserved

The engine must distinguish:

- viable but low-energy pairs  
- strong but mistimed pairs  
- safe but flat pairs  
- feasible, timely, high-upside pairs  

## Current production bridge

Today’s **Matchmaker 3.0.1 seven-feature** scoring in `profile-analysis-service` / backend orchestration is the operational baseline; 3.x is the north-star architecture, not a single big-bang deploy.

## Related docs in this hub

- [implementation-plan-summary.md](implementation-plan-summary.md) — phased, evidence-based rollout  
- [technical-review-notes.md](technical-review-notes.md) — review comments on the 3.x draft  
- [../../architecture/operations/matching-priority.md](../../architecture/operations/matching-priority.md) — ops prioritization tiers  

*Synthesized from ~3.7k-line strategy source; diagrams and formulas live in engineering Notion / source export.*
