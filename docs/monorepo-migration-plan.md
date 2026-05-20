# Monorepo Migration Plan

| Field | Value |
| --- | --- |
| **Author** | Ryan Willis / Ditto Engineering |
| **Date** | April 2026 |
| **Status** | Draft |
| **Team** | Engineering |

<aside>
📋

This plan migrates 18 repos into a single Nx monorepo. Scope is intentionally narrow: **move the code, unify CI, change nothing else.** Deployment process, branching strategy, and infrastructure remain unchanged. GKE Autopilot, per-PR environments, trunk-based branching, and AI governance are separate follow-up initiatives that the monorepo enables but does not require.

</aside>

---

# 1. What Changes and What Doesn't

| | Changes | Stays the Same |
| --- | --- | --- |
| **Code location** | 18 repos → 1 monorepo | Actual source code (no refactoring) |
| **CI pipeline** | 13 workflow dirs → shared workflows using `nx affected` | Build triggers (push to `dev` → dev, push to `main` → prod) |
| **Internal deps** | `@dodo-world/proj-coach-schemas` npm package → `workspace:*` | All external deps (versions, registries) |
| **Docker images** | Built from monorepo context | Same image names, same registries (Docker Hub + Artifact Registry) |
| **Branch strategy** | None | `dev`/`main` per the current flow |
| **Deployment** | None | Docker Hub → `deploy.internal.ditto.ai/deploy/{code}`, Skaffold for PA, tag-based for otel |
| **Infrastructure** | None | k3s + Cloud Run + Docker Compose for local |
| **Dev environment** | None | Shared dev sandbox |

---

# 2. Current State — What We're Actually Migrating

## 2.1 Services and Their Deploy Pipelines

| Service | Image Registry | Deploy Mechanism | Schemas Dep? | Dockerfile Complexity |
| --- | --- | --- | --- | --- |
| **proj-coach-backend** | Docker Hub `dodoworld/proj-coach` | curl `/deploy/be` | Yes (0.12.65) | High — 4-stage, compiles libvips + sharp from source |
| **imsg-service** | Docker Hub `dodoworld/ditto-imsg` | curl `/deploy/imsg` | No | Low — single stage Bun |
| **profile-analysis-service** | Docker Hub + GCP Artifact Registry | curl `/deploy/pa` + Skaffold → Cloud Run | Yes (0.12.47) | High — 3 Dockerfile variants (dev/prod/scratch) |
| **prompt-manager-backend** | Docker Hub `dodoworld/prompt-manager` | curl `/deploy/pm` | No | Low — single stage Bun Alpine |
| **delayed-task-service** | Docker Hub `dodoworld/delayed-tasks` | curl `/deploy/dt` | Yes (0.3.2) | Low — single stage Bun |
| **proj-coach-ws** | Docker Hub `dodoworld/proj-coach-ws` | None (manual) | No | Low — single stage Bun |
| **ditto-internal-ws** | Docker Hub `dodoworld/proj-coach-internal-ws` | None (manual) | No | Low — single stage Bun |
| **proj-coach-voip** | Docker Hub `dodoworld/proj-coach-voip` | None (manual) | No | Medium — multi-stage pnpm/Node |
| **otel** | GCP Artifact Registry | Tag-based semver | No | Medium — multi-stage Go + OCB |
| **ditto-internal-frontend** | (separate deploy) | (scoped out) | No | N/A |
| **marketing-internal-tool** | (separate deploy) | (scoped out) | No | N/A |

## 2.2 Shared Dependencies

Only **3 services** consume `@dodo-world/proj-coach-schemas`:
- `proj-coach-backend` (v0.12.65)
- `profile-analysis-service` (v0.12.47 — already 18 versions behind)
- `delayed-task-service` (v0.3.2 — significantly behind, likely pinned intentionally)

All CI workflows use `secrets.KELLY_PAT` as `NPM_TOKEN` for private package access. In the monorepo, this is no longer needed for schemas (workspace dependency), but may still be needed for `@dodo-world/config-loader`.

## 2.3 The One Complex Dockerfile

`proj-coach-backend` has a 4-stage Dockerfile that compiles **libvips from source** (libde265, x265, libaom, libhwy, libspng, cgif, libheif, libvips) and then builds **sharp's native addon**. This takes ~10 minutes to build from scratch. The Docker build context and layer caching must be preserved carefully during migration. This is the highest-risk Dockerfile to migrate.

---

# 3. Target Monorepo Structure

```
ditto/
├── nx.json
├── package.json                    # Bun workspaces root
├── bun.lock
├── tsconfig.base.json
├── .github/
│   └── workflows/
│       ├── ci-dev.yml              # Push to dev: nx affected build + push + deploy
│       ├── ci-prod.yml             # Push to main: nx affected build + push + deploy
│       └── lint.yml                # PR: nx affected lint + test
│
├── libs/
│   ├── schemas/                    # @ditto/schemas (was @dodo-world/proj-coach-schemas)
│   │   ├── lib/                    # 79 schema files
│   │   ├── package.json            # name: @ditto/schemas
│   │   └── project.json            # Nx config
│   ├── protos/                     # @ditto/protos (gRPC definitions)
│   └── shared/                     # @ditto/shared (new, if needed later)
│
├── apps/
│   ├── backend/                    # proj-coach-backend
│   │   ├── src/
│   │   ├── config/
│   │   ├── Dockerfile              # The complex libvips build stays as-is
│   │   ├── package.json            # @dodo-world/proj-coach-schemas → @ditto/schemas: workspace:*
│   │   ├── project.json
│   │   └── CLAUDE.md
│   ├── imsg-service/
│   ├── profile-analysis/
│   ├── prompt-manager/
│   ├── delayed-tasks/
│   ├── ws/
│   ├── internal-ws/
│   ├── voip/
│   ├── internal-frontend/
│   ├── marketing-tool/
│   └── otel-collector/             # Go service, Nx orchestrates build commands
│
├── tools/
│   ├── matchmake-experimentation/
│   ├── ufl/
│   ├── claude-marketplace/
│   └── event-yikyak/
│
└── docs/
```

---

# 4. Migration Phases

## Phase 0: Preparation (Day 1-2)

**Goal**: Set up the monorepo skeleton without moving any code yet.

| Task | Detail | Risk |
| --- | --- | --- |
| Create monorepo repo on GitHub | `ditto-monorepo` (or reuse existing `ditto` org repo) | None |
| Initialize Nx workspace | `nx.json`, root `package.json` with `workspaces`, `tsconfig.base.json` | None |
| Set up root configs | ESLint config, Prettier config, `.gitignore` | None |
| Create directory structure | `apps/`, `libs/`, `tools/`, `docs/` directories | None |
| Configure Bun workspaces | `package.json` workspaces field pointing to `apps/*`, `libs/*`, `tools/*` | None |
| Test empty workspace | `bun install` from root should succeed | None |

**Deliverable**: Empty monorepo that builds. No code migrated yet.

---

## Phase 1: Migrate Shared Libraries (Day 2-3)

**Goal**: Move `proj-coach-schemas` and `proj-coach-protos` into `libs/`. This is the foundation — everything else depends on schemas.

### 1a. Migrate schemas

```bash
# Option A: Copy without history (simpler, recommended)
cp -r projects/proj-coach-schemas/* monorepo/libs/schemas/

# Option B: Preserve history (complex but preserves git blame)
cd monorepo
git subtree add --prefix=libs/schemas ../projects/proj-coach-schemas main
```

Update `libs/schemas/package.json`:
```json
{
  "name": "@ditto/schemas",
  "version": "0.12.65",
  "main": "lib/index.ts"
}
```

Create `libs/schemas/project.json`:
```json
{
  "name": "schemas",
  "targets": {
    "build": { "command": "tsc --noEmit" },
    "lint": { "command": "eslint lib/" }
  }
}
```

**Verification**: `nx run schemas:build` passes.

### 1b. Migrate protos

Same process for `proj-coach-protos` → `libs/protos/`.

**Verification**: Protos compile.

### 1c. Decision: Git history

| Approach | Pros | Cons |
| --- | --- | --- |
| Copy (no history) | Simple, clean repo | Lose `git blame` for old code |
| `git subtree add` | Preserves full history | Messy merge commits, larger repo |
| `git filter-repo` | Clean history, no merge commits | Complex setup, easy to mess up |

**Recommendation**: Copy without history. Old repos stay archived on GitHub — you can always check history there. Don't let history preservation block the migration.

---

## Phase 2: Migrate Schema Consumers (Day 3-5)

**Goal**: Move the 3 services that depend on `@dodo-world/proj-coach-schemas` and switch them to `workspace:*`.

**Order**: backend → profile-analysis → delayed-tasks (most complex first)

### For each service:

1. Copy code into `apps/{name}/`

2. Update `package.json`:
   ```diff
   - "@dodo-world/proj-coach-schemas": "0.12.65"
   + "@ditto/schemas": "workspace:*"
   ```

3. Update imports (if package name changed):
   ```diff
   - import { UserSchema } from '@dodo-world/proj-coach-schemas';
   + import { UserSchema } from '@ditto/schemas';
   ```
   Or: keep the old package name in `libs/schemas/package.json` as `@dodo-world/proj-coach-schemas` to avoid import changes. This is the safer option — zero code changes in consumers.

4. Create `project.json` with build/test/lint targets

5. **Dockerfile context**: This is the critical change. Dockerfiles assumed build context was the repo root. In the monorepo, context is either:
   - **Option A**: Build from monorepo root, Dockerfile in `apps/backend/Dockerfile`
     ```dockerfile
     COPY apps/backend/package.json .
     COPY libs/schemas/ ./libs/schemas/
     ```
   - **Option B**: Build from `apps/backend/` with libs copied in CI before build

   **Recommendation**: Option A. It's how Nx Docker builds work. The CI workflow sets build context to monorepo root.

6. Verify: `nx run {app}:build` passes, `nx run {app}:test` passes, Docker build succeeds

### Backend-Specific Concerns

The backend Dockerfile compiles libvips from source (~10 min cold build). This doesn't change — the build stages are self-contained. Only the final `COPY` stage changes to reference monorepo paths.

```dockerfile
# Only this part changes:
COPY --from=install /app/node_modules ./node_modules
COPY apps/backend/src ./src
COPY apps/backend/config ./config
COPY libs/schemas ./libs/schemas
# The libvips/sharp compilation stages are unchanged
```

**Docker cache**: The expensive libvips/sharp stages will cache as before since they don't depend on source code. Only the final COPY stage invalidates on code changes.

### Profile-Analysis-Specific Concerns

Has 3 Dockerfile variants. Migrate all three:
- `Dockerfile` (dev) → `apps/profile-analysis/Dockerfile`
- `Dockerfile.prod` → `apps/profile-analysis/Dockerfile.prod`
- `Dockerfile.scratch` → `apps/profile-analysis/Dockerfile.scratch`

Also has Skaffold config. Update `skaffold.yaml` to reference new Dockerfile path:
```yaml
build:
  artifacts:
    - image: us-central1-docker.pkg.dev/petkeley/proj-coach-registry/pa-scratch
      docker:
        dockerfile: apps/profile-analysis/Dockerfile.scratch
```

### Version Alignment

`profile-analysis-service` is on schemas v0.12.47, backend is on v0.12.65. With `workspace:*`, both will use the same schemas code. **This is a risk** — profile-analysis hasn't been tested with the latest 18 versions of schema changes.

**Mitigation**: Before switching to `workspace:*`, run profile-analysis tests against current schemas head. Fix any type errors. This might add a day.

---

## Phase 3: Migrate Independent Services (Day 5-7)

**Goal**: Move services with no schemas dependency. These are lower risk.

| Service | Notes |
| --- | --- |
| imsg-service | Simple Bun Dockerfile, no schemas dep |
| prompt-manager | Simple Bun Alpine Dockerfile, has complex hooks in `.claude/settings.json` — migrate those too |
| proj-coach-ws | Simple Bun Dockerfile |
| ditto-internal-ws | Simple Bun Dockerfile |
| proj-coach-voip | Multi-stage pnpm/Node build — note: uses **pnpm**, not Bun |
| otel-collector | Go service, tag-based release workflow. Keep its release workflow separate. |

### VoIP Service Note

`proj-coach-voip` uses **pnpm**, not Bun. In the monorepo, it can either:
- Stay on pnpm with its own lockfile (Nx supports per-project package managers)
- Or migrate to Bun (minor effort but a code change — scope out for now)

### Otel Collector Note

The otel-collector has a unique release workflow (semver tag bumping). This is different from all other services. Keep its `project.json` targets pointing to its existing build commands (`go build` via OCB). The tag-based release can stay as a separate workflow or be integrated into Nx later.

---

## Phase 4: Migrate Frontends + Tooling (Day 7-9)

| App | Notes |
| --- | --- |
| ditto-internal-frontend | Uses **pnpm**, React/Vite. Deployment is scoped out — just move code. |
| marketing-internal-tool | Uses **npm**, Next.js, Supabase. Independent. Just move code. |
| matchmake-experimentation | Python. Goes in `tools/`. |
| ufl | Python. Goes in `tools/`. Has `.claude/` config — migrate it. |
| claude-marketplace | Goes in `tools/`. |
| event-yikyak | Python. Goes in `tools/`. |

### Python Services

Nx supports Python via `@nxlv/python` plugin. But for the initial migration, keep it simple:

```json
// tools/ufl/project.json
{
  "name": "ufl",
  "targets": {
    "test": { "command": "cd tools/ufl && uv run pytest" },
    "lint": { "command": "cd tools/ufl && uv run ruff check" }
  }
}
```

No need for the `@nxlv/python` plugin on day 1. Just wrap existing commands.

---

## Phase 5: Unify CI/CD (Day 9-12)

**Goal**: Replace 13 separate workflow directories with shared monorepo workflows that preserve existing deploy behavior.

### CI Workflow: `ci-dev.yml` (push to dev)

```yaml
name: CI Dev
on:
  push:
    branches: [dev]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # nx affected needs history

      - uses: oven-sh/setup-bun@v2

      - run: bun install

      - name: Determine affected apps
        id: affected
        run: |
          echo "apps=$(nx show projects --affected --type=app --json)" >> $GITHUB_OUTPUT

      - name: Build and push affected Docker images
        run: |
          nx affected --target=docker:build
          nx affected --target=docker:push
        env:
          DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
          DOCKERHUB_SECRET: ${{ secrets.DOCKERHUB_SECRET }}

      - name: Deploy affected services
        run: nx affected --target=deploy:dev
        env:
          DITTO_DEPLOY_KEY: ${{ secrets.DITTO_DEPLOY_KEY }}
```

Each app's `project.json` defines its own `docker:build`, `docker:push`, and `deploy:dev` targets that wrap the existing commands:

```json
// apps/backend/project.json
{
  "name": "backend",
  "targets": {
    "build": { "command": "tsc --noEmit" },
    "test": { "command": "bun test" },
    "lint": { "command": "eslint src/" },
    "docker:build": {
      "command": "docker build -t dodoworld/proj-coach -f apps/backend/Dockerfile ."
    },
    "docker:push": {
      "command": "docker push dodoworld/proj-coach"
    },
    "deploy:dev": {
      "command": "echo 'No auto-deploy on dev for backend'"
    },
    "deploy:prod": {
      "command": "curl -X POST https://deploy.internal.ditto.ai/deploy/be -H 'Authorization: Bearer $DITTO_DEPLOY_KEY'"
    }
  }
}
```

**Key principle**: Each app's deploy target wraps the **exact same curl command** it uses today. We're not changing how deployment works — we're just triggering it from a unified pipeline.

### Profile-Analysis Special Handling

Profile-analysis has the most complex deploy (k3s + Cloud Run via Skaffold). Its `deploy:prod` target wraps the existing Skaffold commands:

```json
{
  "deploy:prod": {
    "command": "cd apps/profile-analysis && skaffold run --default-repo=us-central1-docker.pkg.dev/petkeley/proj-coach-registry"
  }
}
```

### Otel Special Handling

Otel uses tag-based semver releases. Keep a separate `release-otel.yml` workflow that triggers on tags matching `otel-v*`. Don't force it into the `nx affected` pipeline — it's a different release cadence.

### What `nx affected` gives us

When someone pushes to `dev` and they only changed `apps/imsg-service/`, the CI pipeline:
1. Detects only imsg-service is affected
2. Builds only the imsg-service Docker image
3. Deploys only imsg-service

When someone changes `libs/schemas/`, the CI pipeline:
1. Detects backend, profile-analysis, and delayed-tasks are affected (all schema consumers)
2. Builds all 3 Docker images
3. Deploys all 3

This is the exact behavior we want — and it happens automatically from the Nx dependency graph.

---

## Phase 6: Testing and Cutover (Day 12-15)

### Pre-Cutover Testing

| Test | How | Pass Criteria |
| --- | --- | --- |
| All apps build | `nx run-many --target=build --all` | Zero errors |
| All tests pass | `nx run-many --target=test --all` | Same pass rate as individual repos |
| All Docker images build | `nx run-many --target=docker:build --all` | All 9 images build successfully |
| Schema consumer compatibility | Run profile-analysis and delayed-tasks tests against latest schemas | Zero new failures |
| Deploy pipeline (dev) | Push a test change to dev, verify affected services deploy | Images pushed, curl triggers fire |
| Deploy pipeline (prod) | Push a test change to main, verify affected services deploy | Images tagged `:prod`, deploy triggers fire |
| Backend Dockerfile | Build from monorepo root with `apps/backend/Dockerfile` | libvips compiles, sharp works, image starts |
| Profile-analysis Dockerfiles | Build all 3 variants from monorepo context | All variants work, Skaffold deploys |

### Cutover Day

1. **Freeze**: Announce to team — no new PRs to old repos
2. **Final sync**: Pull latest from all old repos, copy any last-minute changes to monorepo
3. **Verify**: Run full test suite one more time
4. **Switch**: Push monorepo to GitHub, enable branch protection on `dev` and `main`
5. **Archive**: Set all old repos to read-only on GitHub (don't delete — preserve for git blame history)
6. **Notify**: Team starts working in monorepo

### Rollback Plan

If critical issues surface after cutover:
1. Old repos are archived but not deleted — unarchive them
2. Re-enable workflows in old repos
3. Engineers switch back
4. Fix the monorepo issue, try cutover again

There's no data to lose — it's just code. The worst case is a day of confusion.

---

# 5. Timeline

| Week | Phase | What Happens | Who |
| --- | --- | --- | --- |
| **Week 1** | Phase 0-1 | Nx workspace setup, migrate schemas + protos | Ryan (1 person) |
| **Week 1-2** | Phase 2 | Migrate 3 schema consumers (backend, PA, delayed-tasks), fix PA schema version gap | Ryan |
| **Week 2** | Phase 3 | Migrate 6 independent services | Ryan + 1 engineer |
| **Week 2-3** | Phase 4 | Migrate frontends + Python tooling | Ryan + 1 engineer |
| **Week 3** | Phase 5 | Unified CI/CD workflows, test deploy pipeline | Ryan + Nathan |
| **Week 3-4** | Phase 6 | Testing, cutover, team onboarding | Whole team |

**Total: 3-4 weeks** with buffer for unexpected issues. Core code migration is ~2 weeks. CI/CD unification + testing is ~1-2 weeks.

**Team impact during migration**: Near zero. Engineers continue working in old repos until cutover day. The monorepo is built in parallel.

---

# 6. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Backend Dockerfile breaks with new build context | Medium | High — blocks backend deploys | Test Docker build early (Day 4). The libvips stages are self-contained. Only COPY paths change. |
| Profile-analysis schemas version gap (v0.12.47 → v0.12.65) | Medium | Medium — type errors in PA | Run PA tests against latest schemas before migration. Fix breaking changes. Budget 1 extra day. |
| Skaffold config breaks with monorepo paths | Low | Medium — blocks Cloud Run deploys | Test Skaffold deploy in dev before cutover. Path changes are straightforward. |
| `bun install` resolution issues with workspace deps | Low | Low — fixable | Test workspace resolution incrementally as each app is added. |
| CI secrets not available in new repo | Low | High — all deploys fail | Migrate all secrets to new repo before cutover day. Test with a dry-run deploy. |
| Engineers push to old repos after cutover | Medium | Low — confusion | Archive old repos immediately on cutover. Slack announcement + calendar invite. |
| Nx learning curve slows team | Low | Low | Day-to-day commands are simple: `nx run backend:build`, `nx affected --target=test`. Write a 1-page cheat sheet. |
| pnpm/npm services (voip, frontend, marketing) conflict with Bun workspace | Low | Medium | Keep per-app package managers. Nx supports this. Don't force Bun on everything. |

---

# 7. What This Migration Does NOT Include

These are all follow-up initiatives. They are enabled by the monorepo but not part of this migration:

| Initiative | Why Not Now | When |
| --- | --- | --- |
| Trunk-based branching | Reviewer asked to see completed migration first | After team is comfortable with monorepo (1-2 months) |
| GKE Autopilot | Needs sizing exercise, separate infrastructure initiative | Separate RFC |
| Per-PR dev environments | Much larger effort, needs its own design | Separate RFC after GKE decision |
| AI governance (`ai/` directory, prompts in git) | Scope creep, separate doc already exists | Separate initiative |
| Consolidating Docker Hub → Artifact Registry | Not blocking, can do incrementally | Phase 2 of infra consolidation |
| Nx Cloud (remote caching) | Nice-to-have, not needed for migration | After migration stabilizes |
| CLAUDE.md standardization | Migrate existing files as-is, improve later | Organic / Claude Code Governance doc |
| Frontend deployment changes | Scoped out per reviewer feedback | TBD |

---

# 8. Day-1 Monorepo Cheat Sheet (for the team)

Post this in Slack on cutover day:

```
MONOREPO QUICK START

Clone:
  git clone <monorepo-url> && cd ditto && bun install

Build one app:
  nx run backend:build

Test one app:
  nx run backend:test

Test what you changed:
  nx affected --target=test

See what's affected by your changes:
  nx affected --target=build --dry-run

Run the full project graph:
  nx graph

Find an app:
  ls apps/         # backend, imsg-service, profile-analysis, ...
  ls libs/         # schemas, protos
  ls tools/        # ufl, matchmake-experimentation, ...

Old repo → New location:
  proj-coach-backend       → apps/backend/
  proj-coach-schemas       → libs/schemas/
  imsg-service             → apps/imsg-service/
  profile-analysis-service → apps/profile-analysis/
  prompt-manager-backend   → apps/prompt-manager/
  delayed-task-service     → apps/delayed-tasks/
  ditto-internal-frontend  → apps/internal-frontend/
  proj-coach-ws            → apps/ws/
  ditto-internal-ws        → apps/internal-ws/
  proj-coach-voip          → apps/voip/
  otel                     → apps/otel-collector/
  marketing-internal-tool  → apps/marketing-tool/

Everything else works the same. Same branches (dev/main).
Same deploy process. Same Docker images. Same URLs.
```
