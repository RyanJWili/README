# Matchmaking — phased implementation plan (summary)

**Status:** Proposed technical strategy (Mar 2026).  
**Contrast:** Sequences investment so each phase produces data for the next—unlike building all 3.x layers at once.

## Phase map

| Phase | Layer | Delivers |
|-------|--------|----------|
| **0** | Instrumentation | Decision traces, event timelines, outcome linker |
| **1** | Automation | OpenClaw match reviewer, confidence scoring, coach review workflow |
| **2** | Feature optimization | Outcome-weighted features, directional A→B / B→A, coach-pattern signals |
| **3** | Temporal state | Engagement vector, receptivity (GBT), anomaly gating |
| **4–5** | ML heads | Likelihood (XGBoost → two-tower), Intensity (LambdaMART → neural ranker) |
| **6** | Policy | OpenClaw skills, plugin enforcement, gated rerank pipeline |
| **7** | Learning loop | Monthly retrain, calibration monitoring, counterfactual logs |
| **8** | Conditional heads | Chemistry (residual uplift), Readiness (temporal)—only if data validates |

## Integration surfaces (conceptual)

```
OpenClaw agents  →  trigger / review
Restate handlers →  autofilter, scoring rounds, stable matching
MongoDB          →  profiles, matches, outcomes, traces
MeiliSearch      →  candidate retrieval
MLflow           →  model registry → inference servers
UFL pipeline     →  chat topic / synthetic signals into training
```

## Guiding principles

- **Evidence before complexity** — log decisions before adding heads.  
- **Human-in-the-loop** — coach review remains until automation confidence proven.  
- **Directional scoring** — two-sided mutuality explicit in features and models.  
- **Plugins over monolith** — policy as composable skills on calibrated outputs.  

## Related

- [engine-3x-summary.md](engine-3x-summary.md) — strategic vision  
- [../../docs/integrations/openclaw.md](../../docs/integrations/openclaw.md) — agent integration guide  
- [../../diagrams/match-state-machine.md](../../diagrams/match-state-machine.md)  

*Synthesized from Ditto Proposed Matchmaking Strategy — Technical Implementation Plan (~1.1k lines in source).*
