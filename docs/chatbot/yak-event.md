# Chatbot — Yik Yak event

**Status:** Event closed (Mar 2026). Romantic match delivered Mar 9; registration closed Mar 8. Users convert to **Wednesday** pool for ongoing matching.

## What it was

Partnership with Yik Yak: users text a referral code (`yak match! Code:…`). Backend fetches profile from Yik Yak API, maps into Ditto profile, assigns **`yik-yak` pool** only.

## Chatbot behavior (historical)

| Phase | Behavior |
|-------|----------|
| Referral ingest | `_handleYikYakReferral`, pool = yik-yak |
| Match reveal | `revealMatch` — no extra user text before/after |
| No match | Personalized roast-style message + pitch Wednesday |
| After close | Redirect to Ditto form / `wednesday` keyword |

## Same-school rule

Yak romantic matches were **same-school only** (posters showed first name + images).

## Engineering artifacts

- `event-202603-yik-yak` Python matching engine (~68k profiles)  
- Internal **Yak Matches** admin UI  
- Knowledge doc `yak-event.md` in backend repo for agent tools  

## Active pools note

`yik-yak`, `la-love-yacht`, `nyc-gala` are **inactive** for new signups—use `switchPool` to `wednesday`, not legacy resume helpers.

*Synthesized from yak-agent and yak-event knowledge sources.*
