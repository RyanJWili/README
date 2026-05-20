# Monorepo RFC — summary

**Problem:** Shared schema or chatbot changes touch 5+ repos, separate PRs, and hours of mechanical work per small field addition.

**Proposal:** Consolidate Ditto services into an **Nx monorepo** (`ditto-platform`) with shared libs and affected-based CI.

## Expected benefits

| Area | Benefit |
|------|---------|
| `proj-coach-schemas` | Single PR updates types for all consumers |
| Chatbot + prompts + schemas | One branch for ENG-1518-style refactors |
| CI | `nx affected -t test,build,lint` on changed subgraph only |
| AI tooling | One root `CLAUDE.md` + per-app context |

## Planned layout (target)

```
apps/          # backend, frontends, workers, otel-collector
libs/          # schemas, protos, shared UI, telemetry
tools/         # ufl, experimentation
```

## Q2-critical consumers

Chatbot skills refactor, matchmaker dealbreakers, metrics/analytics packages—listed as **CRITICAL** in the RFC because they span multiple current repos.

## Status

Migration plan exists ([monorepo-migration-summary.md](monorepo-migration-summary.md)). **The monorepo is not published in this documentation repo yet**—docs here describe the direction, not the live tree.

## Risks called out in RFC

- CI time without remote cache discipline  
- Go otel-collector as special-case app  
- Team habit change (trunk-based, affected commands)  

*Full RFC: internal “Monorepo-Driven Development — Ship Q2 3x Faster” source.*
