# Matching priority (ENG-1030)

Automation should not match users purely FIFO. Priority tiers focus ops and algorithms on users who need a match most.

## Tiers

| Tier | Label | Who qualifies |
|------|-------|----------------|
| 1 | **Ultra fast** | Asked for update via chatbot; reactivated users |
| 2 | **Fast** | Recently rejected; waiting >1 week with ≥1 historical match; date cancelled after scheduling; match expired (`PickTimeFailed` / `Failed - Expired`) |
| 3 | **Default** | Everyone else |

## Intent

Reduce manual workload and improve experience for users stuck waiting, without starving brand-new users entirely—tier 3 still receives matches on normal cadence.

## Implementation note

Exact job names and cron hooks live in `proj-coach-backend` internal matching modules; verify code before changing thresholds.

*Source: ENG-1030 matching priority system doc.*
