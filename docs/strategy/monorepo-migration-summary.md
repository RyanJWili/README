# Monorepo migration — summary

Phased move from multi-repo `projects/*` to `ditto-platform` Nx workspace.

## Phases (high level)

| Phase | Scope |
|-------|--------|
| 1 | Import `proj-coach-schemas`, `proj-coach-protos` into `libs/` |
| 2 | Migrate backends and workers with unchanged deploy paths initially |
| 3 | Frontends + `otel-collector` |
| 4 | Python tools (`ufl`) and release cadence exceptions |

## Deployment note

Most services today deploy via internal deploy URLs / Skaffold; otel uses **tag-based** releases—RFC keeps separate workflow for that cadence.

## Dependency rule

Apps depend on libs; libs must not depend on apps. Schemas/protos publish internally via workspace references, not repeated npm tag dance.

## Current documentation gap

Until `ditto-platform` is added to this hub, treat **per-repo READMEs** under [../../repos/README.md](../../repos/README.md) and [../../architecture/platform-overview.md](../../architecture/platform-overview.md) as source of truth for production layout.

*Source: internal Monorepo Migration Plan.*
