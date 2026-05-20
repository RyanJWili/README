# Q2 roadmap (Apr–Jun)

**Theme:** Deepen quality → scale → summer launch.

## Q1 outcomes (context)

| Metric | End Q1 | Q1 goal | Result |
|--------|--------|---------|--------|
| Successful dates | 1,203 | 2,000 | Miss |
| Matches / Wednesday peak | 634 | 2,500 | Miss |
| Phone-verified users | ~119k | — | Strong growth |
| iMessage lines | 117 | — | Scale achieved; flagging risk |

**Lessons:** Matchmaker v3 not shippable as planned; too much firefighting; iMessage line health depends on engagement; Shadow Profile / UFL over-engineered → replaced by **User Memory**; Yik Yak partnership validated scale.

## Q2 milestones

| Date | Milestone |
|------|-----------|
| Apr 14 | Chatbot context refactor complete |
| Mid-May | Summer **soft launch** (nationwide matching, limited marketing) |
| Mid-Jun | Summer **full launch** + scaled marketing |
| Jun 30 | Q2 close |

## Top priorities

1. **Summer launch** — cross-school matching, city campaigns (SF, LA, NYC, Chicago, Boston), in-person events  
2. **Match quality** — User Memory, chatbot context, matchmaker v2 improvements  
3. **Velocity** — ship full-stack changes ~3× faster (monorepo RFC)  
4. **iMessage line health** — drive engagement on lines to reduce carrier flags  

## Major initiatives

### Summer launch

Remove strict school-only boundaries for summer matching; coordinate product, engineering, and marketing for soft then full launch.

### User Memory

Single canonical layer extracting preferences and signals from chat (and related inputs) for **matchmaker + chatbot**. Replaces Shadow Profile, UFL-as-profile, and fragmented “dynamic profile” experiments.

### Chatbot reliability

Context improvements, skills migration (ENG-1518), and fixes from production audit—see [../chatbot/production-audit.md](../chatbot/production-audit.md).

### iMessage engagement

Treat line flagging as a product/engagement problem, not only infra capacity.

### Matchmaker v2 → 3.x

Improve v2 in Q2; 3.x strategy in [../../reference/matchmaking/engine-3x-summary.md](../../reference/matchmaking/engine-3x-summary.md).

## Engineering enablers

- Monorepo migration ([monorepo-rfc-summary.md](monorepo-rfc-summary.md))  
- Claude Code governance for Nx ([claude-governance-summary.md](claude-governance-summary.md))  
- Infra: GKE + Infisical ([../../infra/overview.md](../../infra/overview.md))  

*Source: internal Q2 roadmap doc; metrics as of late Q1.*
