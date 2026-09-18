Status: shadow-only (not live) — first day of real data collected 18 Sep 2026

# Ribbon switch-shadow monitoring

## What it is

Built 18 Sep 2026 alongside the MA-ribbon-expansion ranking feature
(`ribbon_score.py`, `reversal_filters.rank_by_ribbon_expansion`/
`rank_by_ribbon_breakdown`). Ranking-only (picking the best CE/PE
candidate among a multi-stock alert) went live behind
`RIBBON_RANKING_ENABLED`/`RIBBON_RANKING_PE_ENABLED`. The other half of
what was backtested — actively **switching**, i.e. exiting an already-
held real position for a better-scoring new candidate — was deliberately
kept shadow-only: `reversal_filters.log_ribbon_switch_shadow_for_alert`
runs unconditionally on every webhook alert (Options/Luxury/Futures,
CE and PE), scores the strategy's real currently-open positions'
*current* trend health, asks `ribbon_score.decide_switch()` what it
would recommend against the new alert's top-ranked candidate, and just
logs the answer to `history/<date>_ribbon_switch_shadow.log`. It never
touches a real position and never affects the real webhook response —
see the traderBoy repo's `reversal_filters.py` (search
`RIBBON_SWITCH_SHADOW_LOG_NAME`) for the exact implementation.

## Known bug on day 1 (fixed same day)

`_log_switch_shadow_sync` builds `open_positions` from
`position_store.snapshot()`. Called in-process (not via the HTTP
endpoint), `snapshot()` returns the `Position` dataclass's raw
`__dict__` — `opened_at` is a real `datetime` object there, not the ISO
string it only becomes after FastAPI's own JSON serialization. This
raised `TypeError: fromisoformat: argument must be str` on **every**
call from this morning's restart until fixed (commit `61f430c`,
~09:14 IST) — caught and logged with zero effect on real trading, but
it meant switch-shadow recorded nothing for the first ~2.5 hours of the
day. Worth checking for this same pattern anywhere else `position_store.
snapshot()` is consumed in-process rather than over HTTP.

## First day's real results (18 Sep 2026, post-fix)

78 evaluations logged (all CE; Futures 45, Luxury 33 — Options logged
zero, because the function only appends a record when the strategy has
at least one real open position of that option_type at alert time, and
Options apparently never had a qualifying open CE position at the same
moment as a qualifying alert today):

| Outcome | Count |
|---|---|
| `no_open_position_beaten_by_the_required_margin` | 52 |
| `capacity_available_no_switch_needed` (room to add without exiting anyone) | 20 |
| **Would actually switch** | **6** |

The 15-point margin (`decide_switch`'s default) is a real, working
brake — it fired a switch recommendation on only 6/78 (7.7%) of
evaluations, not on every alert where the candidate merely scored higher.

### The 6 switch recommendations, checked against what actually happened

Cross-referenced each against `history/2026-09-18_real_trades.log` (same
strategy, same symbol) where a comparable real trade existed:

1. **13:21 IST, Futures**: would exit SIEMENS (held 378 min, health 62)
   for PHOENIXLTD (score 79.77). Real SIEMENS leg held to 13:35 lost
   ₹1,338.75 (`LIQUIDITY_GUARD_ZERO_VOLUME`). Real PHOENIXLTD — actually
   entered independently by Futures at 13:25 — lost only ₹385
   (also `LIQUIDITY_GUARD_ZERO_VOLUME`). **Switching would have saved
   ~₹954** on this leg (clean same-strategy comparison, both real fills).
2. **14:24 IST, Luxury**: would exit ZYDUSLIFE (health 0) for GVT&D
   (score 53.33). GVT&D was never traded for real by Luxury, so no
   direct comparable — the shadow evaluator's independent CE-ATM sim for
   GVT&D that day lost ₹2,093.75 (`STOP_LOSS_HIT`), worse than
   ZYDUSLIFE's actual ₹1,440 EOD loss. **Directionally suggests
   switching would have been worse here**, though not a same-strategy
   real fill.
3. **15:01/15:02/15:04 IST, Futures** (same recommendation repeated
   across 3 evaluation cycles): would exit HINDPETRO (health 52) for ABB
   (score 71.52). ABB had already been tried twice for real by Futures
   earlier that day (12:33 and 13:04) and **lost both times**
   (₹2,737.50 `MAX_LOSS_HIT`, then ₹1,831.25 `SUPERTREND_EXIT`) — no
   real ABB re-entry exists after 13:30 to check against this later
   recommendation, so this one is **unverifiable from today's data**,
   but ABB's same-day track record is not encouraging.
4. **15:12 IST, Luxury**: would exit ZYDUSLIFE (health 52) for HINDPETRO
   (score 67.31). HINDPETRO was actually entered by Luxury for real at
   15:12 (essentially the same moment) and lost only ₹202.50
   (`SUPERTREND_EXIT`), vs. ZYDUSLIFE's actual ₹1,440 EOD loss.
   **Switching would have saved ~₹1,238** on this leg — the cleanest
   comparison of the day (both real fills, same strategy, same minute).

### Bottom line for day 1

Net measurable effect if switching had been live today: roughly
**+₹2,192** better across the 2 cleanly-verifiable legs (SIEMENS→
PHOENIXLTD, ZYDUSLIFE→HINDPETRO), 1 leg that looks like it would have
been worse (GVT&D), and 3 unverifiable (ABB, no fresh real fill to
compare against). This is one day, 6 signals, only 2-3 with a solid
counterfactual — **nowhere near enough to justify flipping switching
live**. Keep accumulating; a "go live" decision should wait for
multiple days of `would_switch` events with clean same-strategy real
comparables, the same bar `RIBBON_RANKING_ENABLED` itself was held to
before its own backtest-then-flip (see traderBoy commits `92b576c` →
`1022b55`).

## What real live wiring would still need (per `ribbon_score.decide_switch`'s own docstring)

- A new `exit_reason` for a switch-driven exit.
- Real-time re-scoring of open positions inside the monitor loop (today
  it's only computed reactively, at the moment a *new* alert arrives).
- The same explicit user sign-off every other live-trading-behavior
  change in this codebase has gone through.
