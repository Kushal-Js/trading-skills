# "liquidity_sweep_prev_hour" strategy - previous-hour box sweep+reclaim, 30-day backtest

**Date:** 26 Sep 2026
**Source:** YouTube video "My Incredibly Easy Scalping Strategy To Grow
Small Accounts FAST" (The Secret Mindset),
https://www.youtube.com/watch?v=_F57FxTEVB0 - no downloadable transcript
file existed (YouTube's own timedtext endpoint returned an empty body for
every signed URL pulled from `ytInitialPlayerResponse`), so the rules were
pulled the same way as the bollinger strategy: seeking the video's
`<video>` element and reading the live caption overlay at ~10 points
across its 23:35 runtime.
**Scope:** backtest script only
(`traderBoy/backtest_liquidity_sweep_prev_hour_swing_watchlist.py`), NOT
wired into Swing/trading_engine.py, not a new ENTRY_STRATEGY_VERSION - a
standalone strategy, sharing only candle-fetch/option-resolution/P&L
plumbing with the rest of this repo's backtests (same pattern as
[[bollinger-vortex-strategy-30day-backtest]]).
**Symbols:** same 9-symbol NSE-equity Swing watchlist (BANDHANBNK,
TORNTPHARM, DLF, ZYDUSLIFE, SONACOMS, CIPLA, ASHOKLEY, VEDL, SOLARINDS),
OPTIONS basket, last 30 trading days (28 Aug - 25 Sep 2026). The live
watchlist also carries COPPER/NATURALGAS/NIFTY/BANKNIFTY (13 symbols
total as of this session) - not covered here, same exclusion reason as
the bollinger MCX/index companion script (different security-id/exchange-
segment resolution, no OPTSTK universe for an index or MCX futures
underlying).

## The rules, as stated in the video

1. **Levels** - the previous clock-hour's high/low become a "box" - the
   only two price levels that matter for the next hour ("when the next
   hour starts, those two edges become the levels I watch").
2. **No-trade zone** - do nothing while price sits inside the box
   ("I prepare for a possible sweep").
3. **Sweep** - price must actually BREAK the box edge, not just touch it.
4. **Failure + reclaim = the entry** - if price breaks below the box low
   and then CLOSES back inside the box, that failure is the long entry
   (symmetric for a short on a failed break above the box high). Enter
   after the reclaim, stop below (above) the wick. Explicitly NOT the
   continuation breakout - "if price breaks below and holds outside the
   box" (no reclaim), that's a real breakdown, not this strategy's trade.
5. **Risk management** - 1R = the stop distance. At 1R, take 50% off and
   move the stop to breakeven for the rest. No video-stated final target
   for the remaining half.
6. **Quality filter** - skip if the sweep is too big relative to the box
   ("if price gives them room to escape cleanly, I skip the scalp - I
   made this mistake a lot of times").

## Interpretation calls made (full detail in the script's own docstring)

- "Previous hour" -> discrete wall-clock session blocks from market open
  (09:15, 10:15, ..., 15:15-15:30 shorter final block), not a rolling
  60-minute window - resets every day (NSE closes overnight, unlike the
  video's own continuous gold/forex chart); the first block of each day
  is never traded (no prior box).
- "Holds outside the box" -> at most ONE reclaim decision (taken or
  skipped) evaluated per block per side, no re-arming on repeated sweeps
  of the same level within the hour. The video's own follow-on idea (a
  later retest of the broken level, now flipped support/resistance, as a
  separate continuation trade) is NOT implemented here.
- "Skip if it gives them room to escape cleanly" -> quantified as: skip
  if the resulting stop distance exceeds 1.5x the box's own height
  (`MAX_STOP_TO_BOX_RATIO`) - never stated numerically in the video, a
  documented judgment call.
- Underlying-to-premium translation, partial-profit bookkeeping - same
  convention as the bollinger script: underlying stop distance ->
  percentage of underlying entry price (floored at 1%) -> applied to the
  option premium via `Swing.position_store.hard_stop_for`/
  `unrealized_pnl_rs`. Each entry that reaches 1R produces two trade rows
  (`1_partial_1R` for the 50% booked at target, `2_runner` for the rest,
  closed by the breakeven stop / `MAX_LOSS_PROTECTION_RS` safety net /
  15:15 IST square-off). `Swing.config.MAX_LOSS_PROTECTION_RS` rupee
  circuit-breaker applied as a safety net, same as every backtest here -
  not from the video.

## Result

| Symbol | Trade-legs | Wins | Losses | Win rate | Net P&L |
|---|---|---|---|---|---|
| BANDHANBNK | 74 | 28 | 42 | 37.8% | -Rs 11,764 |
| TORNTPHARM | 103 | 45 | 56 | 43.7% | -Rs 10,694 |
| DLF | 93 | 39 | 47 | 41.9% | -Rs 2,916 |
| ZYDUSLIFE | 89 | 37 | 50 | 41.6% | -Rs 9,729 |
| SONACOMS | 72 | 25 | 41 | 34.7% | -Rs 11,494 |
| CIPLA | 108 | 48 | 57 | 44.4% | -Rs 1,769 |
| ASHOKLEY | 84 | 40 | 38 | 47.6% | -Rs 1,533 |
| VEDL | 110 | 43 | 40 | 39.1% | -Rs 1,494 |
| SOLARINDS | 59 | 22 | 35 | 37.3% | -Rs 11,015 |
| **COMBINED** | **792** | **327** | **406** | **41.3%** | **-Rs 62,410** |

("Trade-legs" not entries - 522 real entries total, 270 of which reached
1R and so produced 2 rows each; see the leg breakdown below for the real
per-entry picture.) All 9 symbols net negative, unlike the bollinger
strategy's clean sweep of all-positive on the same watchlist/window -
these two video-derived strategies land on opposite sides of breakeven
against the identical symbol set and lookback, a useful contrast (see
Comparison section below).

Combined day-wise P&L was negative or flat on 17 of 19 active trading
days; only 2026-08-28 (+Rs 1,710) and 2026-09-01 (+Rs 1,887) were net
positive, both small relative to the worst day (2026-09-15: -Rs 14,701).
Losses accumulated steadily rather than from one or two outlier days -
running total crossed -Rs 50,000 by 09-21 and drifted to -Rs 62,410 by
09-25 without a single day recovering meaningfully.

## Real diagnostic found while building this: the asymmetry that sinks it

Broken down by leg (792 rows = 522 entries):

| Leg | Count | Exit reasons | Net P&L |
|---|---|---|---|
| Full loser (never reached 1R) | 252 | STOP_LOSS_HIT (248), EOD_SQUARE_OFF (3), MAX_LOSS_HIT (1) | **-Rs 125,392** |
| `1_partial_1R` (the 50% booked at target) | 270 | TARGET_1R_PARTIAL (270) | +Rs 20,538 |
| `2_runner` (the remaining 50% after 1R) | 270 | BREAKEVEN_STOP_HIT (213), EOD_SQUARE_OFF (57) | +Rs 42,444 |

**51.7% of entries (270/522) do reach 1R** - the reclaim signal itself
isn't rare or badly timed. The problem is sizing asymmetry: the 48.3%
that never reach 1R lose the FULL position (-Rs 125,392 total, averaging
-Rs 498/loser), while winners only ever book HALF size at 1R (+Rs 20,538
on 270 partials, averaging +Rs 76 each - tiny, because 1R in premium terms
is itself small once the 1% stop floor applies, see below) plus whatever
the other half's runner adds (+Rs 42,444, but 213 of 270 runners just
breakeven out near Rs 0, only 57 actually ride to EOD square-off for real
size). Net: the strategy is paying full-size losses against half-size-at-
1R-plus-mostly-flat-runner wins - a structurally negative expectancy at
this win rate (41.3% overall, but only 51.7% even reach the FIRST
milestone) unless the runner's occasional big win (EOD square-off) is
large enough to offset it, which this 30-day sample says it isn't.

**Same 1% stop-floor caveat as the bollinger backtest** (see that file's
own write-up) very likely applies here too - a previous-hour box on these
liquid large/mid-caps is typically tight, so `stop_pct = max(stop_pct,
0.01)` is doing a lot of the same normalizing work it did there. Not
independently re-verified with a floor-hit percentage for this run, but
the same underlying-to-premium translation code path is used unchanged.

## Comparison with the bollinger strategy (same symbols, same window)

| | bollinger (trend pullback) | liquidity_sweep_prev_hour (mean-reversion) |
|---|---|---|
| Combined net P&L | +Rs 106,240 | -Rs 62,410 |
| Combined win rate | 57.5% | 41.3% |
| Trade count | 287 | 522 entries (792 legs) |
| All-symbols-positive? | Yes (9/9) | No (0/9) |

Read as informational, not a verdict on trend-following vs. mean-reversion
in general - both are single 30-day samples on the same 9 symbols, with
their own independent set of interpretation calls and both subject to the
1%-stop-floor normalization. The contrast is worth keeping in mind before
treating either result as a converged answer: a strategy this repo backtests
favorably on one video source can flip sharply negative on another, even
on the identical symbol set and window.

**Other standard caveats** (same as every backtest in this repo): no
slippage/brokerage modeled, ATM-strike/1-min option premium is the finest
resolution available, index-options-style expiry-ceiling caveats don't
apply here (NSE-equity monthly options only), one 30-day window is one
sample.

**Outcome:** informational only - not deployed, not wired into Swing.
Purely a standalone backtest artifact, matching the user's own explicit
scope for this class of "watch a video, backtest it" request (see
[[bollinger-vortex-strategy-30day-backtest]] for the identical scope
precedent).
