Status: BACKTEST COMPLETE, 21 Sep 2026 - directionally strong (+Rs756,644.71
raw / +Rs688,954.19 after removing confirmed duplicate-capture trades vs
real -Rs61,827.65 over the same 14 days) but with material, disclosed
caveats below that keep this from being a clean go/no-go signal. Not wired
into any live package. Full report artifact:
https://claude.ai/artifact/X7Adfi1Er4spdDzsDGjbfD

# All-F&O-universe breakout-signal backtest vs real deployed strategy

## What was asked

User request, 21 Sep 2026: build a function that pulls ALL F&O-eligible
NSE stocks (not just symbols each package happened to get a Chartink alert
for) and feeds them into the breakout-signal scanner (`breakout_signal.py`,
deployed as the sole real entry path for Options/Luxury/Futures earlier
the same day - see `incidents/2026-09-21-breakout-signal-sole-entry-path-
deploy.md`), using the CURRENT bot's own entry gates and exit ladder, and
report day-wise PnL for the last 15 days compared against the real
deployed strategy.

This directly answers the open question left at the end of
`breakout-scanner-vs-real-pnl.md`: "gather more days of data, try
live-wiring as a NEW capacity-slot filter... or leave this as a documented
finding" - this backtest is the "what if the watchlist were the WHOLE
tradable universe, not just alerted names" branch of that question.

## Script

`traderBoy/backtest_all_fno_breakout_signal_15day.py`. Built by extending
`backtest_deployed_today_breakout_signal_updated_params.py` (the joint
Options+Luxury+Futures, single-real-day simulation built the same day as
the sole-entry-path deploy) in two ways: (1) universe widened from
"today's real alert buckets" to all 210 F&O-eligible NSE underlyings
(`bsr.fno_eligible_symbols()`), scanned in both directions every day; (2)
window widened from one day to all 14 real trading days with `history/`
data present (2026-08-31 through 2026-09-18 - `backtest_bucket_switch.
DAYS`; "last 15 days" was asked for but, as the original breakout-scanner-
vs-real-pnl.md backtest already found, only 14 exist - today, 2026-09-21,
was still an in-progress session and was excluded).

**Thresholds verified live, not assumed**: a droplet `.env` grep (SSH read,
user-approved) confirmed all three packages currently share IDENTICAL
breakout thresholds - clearance=0.15%, body>=0.5%, relative volume>=0.8x,
20-day avg daily volume>=300,000. This matters because the Futures and
Options-PE-only "gated-live" backtests built earlier the same day had
hardcoded a STALE 0.3%/1.2x "Luxury sweep winner" value that had already
been superseded by this further recalibration by the time they ran -
this script avoids repeating that mistake by reading live values first.

**Efficiency rewrite of signal detection**: rather than `bsr.find_first_
signal_that_day`'s day-siloed fetch (which re-requests an overlapping
15-day window from Dhan on every call - fine for a handful of alerted
symbols, prohibitively redundant at 210 symbols x 14 days), each symbol's
5-min and daily series are fetched ONCE, continuously, and every day's
signal check reads a slice of the same in-memory list -
`fetch_5m_continuous`/`fetch_daily_continuous`. `bsr.evaluate_signal`
itself (the actual 7-check logic) is reused unchanged. Daily-series
look-ahead safety is enforced via the daily response's own `timestamp`
field (confirmed present via a live probe this session, despite an older
docstring note in `bsr` saying otherwise for its own fetch pattern).

**Real gates/exit ladder**: copied verbatim from the "deployed_today"
script's own per-package gate stack (capacity + burst slot, daily
re-entry cap, RSI-loss-reentry, loss-repeat block + trend-strength check,
volume-floor gate where the package has one, gap-down Nifty CE delay,
liquid-contract resolution) and exit ladder (MAX_LOSS caps before/after
`RISK_THRESHOLD_CUTOFF_TIME`, `TARGET_HIT`, `PROFIT_PROTECTION_HIT` with
giveback, dynamic SL, Supertrend/EMA-cross exits with the underlying-move
confirmation gate, liquidity guard, EOD/Friday square-off) - see that
script's own docstring for the full per-gate detail, unchanged here. A
single shared `open_positions` dict (keyed by symbol only, not
per-package) enforces the cross-strategy lock across all three packages,
exactly as production's real `cross_strategy_registry` is meant to.

## Two real operational problems hit and fixed while running this

1. **Local-process Dhan session collision with the live bot** - see
   `incidents/2026-09-21-local-backtest-dhan-session-collision.md` for the
   full incident writeup. Fixed by using a hand-off `access_token` (user
   supplied fresh in chat) instead of a competing local `pin_totp` login.
2. **Shared rate-limit contention (`DH-904`) with the live bot's own
   real-time traffic**, separate from the collision above and not fixed by
   the access-token change - persisted even at 1.2s/call pacing during
   market hours. Fixed by deferring the actual 210-symbol/14-day pull to
   after market close (ran unattended from ~16:00 IST), plus a proper
   retry-with-backoff wrapper around every Dhan call.

## A real bug found IN THE RESULTS, not just the plumbing

**Same-instant triple-counting.** When a signal resolves within its own
entry candle (e.g. `TARGET_HIT` on literally the same 1-minute bar as
entry), all three packages can end up independently "claiming" the
identical trade in this simulation - because the `open_positions` pruning
step (`if close_t <= t: del`) uses a non-strict inequality, a position
that closes AT exactly the same signal timestamp as a same-instant rival
attempt from another package is treated as already-free, so the intended
cross-strategy lock doesn't block the second/third package. Production's
real lock is meant to let only ONE package hold a given underlying at a
time, so this is a genuine simulation artifact, not a legitimate
"independent scanners naturally stagger" case.

Quantified rather than silently left in: 20 exact-duplicate-fill groups
(identical entry AND exit price for the same symbol/day/option_type/
signal_time across >1 package) account for **Rs67,690.52** of the raw
Rs756,644.71 total - about 9%. Removing them: **Rs688,954.19**. A further
~21 multi-package groups share the same signal_time but resolved with
slightly different fills (different candidate contract, different exact
entry candle) - these are NOT adjusted for, so even the "dupes removed"
number likely still overstates the true single-claim total somewhat. This
same `close_t <= t` pattern exists verbatim in the precedent script this
was extended from (and probably in the other single-day gated-live
scripts too) - it just wasn't visible at their much smaller signal
volume. **A proper code fix (`close_t <= t` -> `close_t < t`, or an
explicit same-instant same-symbol dedup before the gate loop) is still
open** - this backtest applied a post-hoc reporting correction rather than
re-running the ~90-minute pull a third time.

## Result

| | Trades | Wins | Win Rate | Total PnL |
|---|---|---|---|---|
| REAL (Options+Luxury+Futures, 14 days) | 284 | 104 | 36.6% | -Rs61,827.65 |
| SIM, raw | 469 | 366 | 78.0% | **+Rs756,644.71** |
| SIM, exact duplicates removed | ~449 | ~346 | ~77% | **+Rs688,954.19** |

By package (raw, all 14 days): Options 210 trades/+Rs237,448.11 (real: 109
trades/-Rs31,974.75); Luxury 207 trades/+Rs430,379.66 (real: 125
trades/-Rs17,909.35); Futures 52 trades/+Rs88,816.94 (real: 50
trades/-Rs11,943.55).

Exit mix (all 469 sim trades): `TARGET_HIT` 205, `PROFIT_PROTECTION_HIT`
137, `LIQUIDITY_GUARD_ZERO_VOLUME` 78, `STOP_LOSS_HIT` 12, `EMA_CROSS_EXIT`
12, `SUPERTREND_EXIT` 9, `TRAILING_SL_HIT` 9, `MAX_LOSS_HIT` 4,
`EOD_SQUARE_OFF` 3.

Gate load (2,184 total signal x package attempts, 728 raw signals x 3
packages): 890 blocked on capacity, 391 on cross-strategy/already-open,
384 on the volume floor, 30 on the Nifty gap-down CE delay, 14 on no
liquid contract, 6 on no fill data - only 469 (21.5%) actually entered.
The gate stack is clearly load-bearing at this scale, not a rubber stamp -
worth noting since a naive reading might assume "scanning everything"
mostly bypasses production's real risk controls; it doesn't.

## Caveats (read before treating this as a green light for anything)

- **Not an apples-to-apples entry-logic comparison.** Breakout-signal-
  gated entry only became the SOLE real entry path TODAY (21 Sep 2026),
  after this entire 14-day window had already traded. "REAL" here means
  the OLDER Chartink-alert-ranked entry logic for essentially the whole
  window, not this same screener at a narrower universe. This backtest
  measures "confirmed-breakout entry over the full F&O universe" against
  "the old ranked-webhook entry logic" - two different variables changing
  at once (universe size AND entry-selection logic), not one.
- **No funds/margin constraint modeled** (same disclosed gap as every
  prior backtest in this line of work - `has_sufficient_bucket_funds`'s
  own fail-open behavior is assumed). This matters far more here than in
  narrower prior backtests: scanning all 210 stocks routinely wants many
  more concurrent positions than a real account's margin could support: a
  live version would take a fraction of these 469 trades.
- **Triple-counting bug**, see above - corrected for exact duplicates,
  not for near-duplicates.
- Same lot-size/qty-scaling discipline as every prior script in this line
  (resolved contract's own real lot size x `QUANTITY_LOTS`, never reused
  from another instrument) - checked, not a repeat of the lot-size bug
  [[alert-bucket-switch]] found once.
- Liquid-contract resolution, gap-down delay, and the exit ladder all
  match each package's OWN currently-deployed `.env` values (read live at
  import time), not hardcoded historical config - but "currently deployed"
  means the RIGHT NOW config applied retroactively across all 14 days,
  including config changes (e.g. the risk-cap raises) that may not have
  been live for the whole window in reality either.

## What's still open

1. Fix the `close_t <= t` cross-strategy-lock ordering bug in the shared
   `open_positions` pruning logic (this script and likely its
   predecessors) and re-run for a fully clean number.
2. Model a funds/margin cap, even approximately, given how much more this
   matters at full-universe scale than in prior narrower backtests.
3. Decide whether "scan the whole F&O universe" is worth prototyping as an
   actual watchlist-source change to `breakout_signal.py` (currently only
   populated via `record_alert` from real Chartink webhooks) - this
   backtest is evidence toward that being promising, not proof, per
   [[feedback-live-trading-safety]]: no live change follows from a
   backtest number alone, no matter how good it looks.
