# ASHOKLEY futures strategy v2: 1-hour RSI, 5-min timed entries/exits (24 Sep 2026, user request)

**Status: exploratory backtesting only - nothing here is deployed live.**
ASHOKLEY is not on any package's live watchlist; the script below is a
standalone backtest in `traderBoy` (repo root, not inside a package dir),
paper/simulation only, no real orders placed at any point.

See [[ashokley-futures-strategy-exploration-orb-v1]] for v1 (Opening
Range Breakout) and the full 15-variant comparison this session already
ran before v2 was found. v2 is a genuinely different signal family from
everything in that table (multi-timeframe RSI, not a crossover-only or
range-breakout design) and, on the same 30-day window, has a BETTER
avg-PnL/trade than v1 (+2,386 vs +357) with less tail-risk concentration
(no single loss anywhere near v1's -18,600 stop-outs).

## Strategy (user's own framing: "1 hour RSI has to be used against 5 min candles")

RSI(14) is computed on the 1-HOUR close series (NOT the 5-min series).
The 5-min series is the master clock/decision grid - at every 5-min bar,
the most recently FULLY-CLOSED 1-hour bar's RSI value is looked up (never
a still-forming hour, to avoid lookahead) and checked against the levels.
Since the 1-hour RSI only actually changes once per hour, a "cross" only
fires on the first 5-min bar after a new hourly RSI value has pushed it
past a level - this gives 5-min timing precision without pretending the
underlying indicator updates faster than once an hour.

  BUY (LONG): 1-hour RSI crosses ABOVE 50.
  SELL (SHORT): 1-hour RSI crosses BELOW 50.
  Square off on WHICHEVER hits first:
    - LONG: RSI reaches/crosses above 70, OR MAX_HOLD_MINUTES elapsed.
    - SHORT: RSI reaches/crosses below 30, OR MAX_HOLD_MINUTES elapsed.

No entry/exit-level ambiguity (unlike the earlier MACD attempt this
session) - 50 (entry) and 70/30 (exit) are genuinely different levels.

**Critical finding: the 70/30 exit levels NEVER fired, at any hold time
tested (30/45/60 min), across the full 30-day sample.** RSI(14) on the
1-hour timeframe essentially never reaches 70 or 30 intraday on this
stock in this window - EVERY SINGLE exit across all 63 trade-instances
(21 trades x 3 hold-time variants) was `MAX_HOLD_TIME_EXIT`. In practice
this strategy IS "enter on a 1h RSI(50) cross, hold exactly N minutes,
exit" - a pure time-boxed trade with no actual RSI-based profit-taking or
stop-loss. Flagged to the user; the 70/30 levels are currently dead code
in terms of real effect on this backtest, not a design flaw in the
implementation.

Script: `backtest_ashokley_futures_1hour_rsi_5min_timing.py`, CLI args
`test_days_back max_hold_minutes` (default 10/30).

## Hold-time sweep (30-day window, 2026-08-12 to 2026-09-23, 21 identical
entries each - only exit TIMING differs, since RSI never hits 70/30)

| Hold (min) | Trades | Wins | Losses | Win Rate | Net P&L | Avg P&L/trade |
|---|---:|---:|---:|---:|---:|---:|
| **30 (best)** | 21 | 12 | 9 | 57.1% | **+45,050*** | **+2,386** |
| 45 | 21 | 13 | 8 | 61.9% | +45,450 | +2,164 |
| 60 | 21 | 10 | 11 | 47.6% | +28,850 | +1,374 |

(*30-min total P&L is +50,100 per the original backtest message - the
45,450/28,850 figures above are the 45/60-min re-runs' own totals,
recorded verbatim from their run output.)

30 minutes is the best hold on total P&L; 45 minutes has a slightly
higher win rate (61.9% vs 57.1%) but a lower total because its winners
average smaller and at least one loser (09/07, -7,950) is worse than the
30-min version's equivalent trade (-3,750) - longer holds let both
winners and losers run further, and on this sample the losers ran
further on average past 45 minutes. 60 minutes is clearly worse on every
metric (lowest win rate, lowest total, lowest per-trade average) -
holding a full hour gives the market enough time to round-trip several
of these trades from profit back to loss.

## 30-day result at the winning hold time (30 min): full trade log

Also compare 10-day-only sample (3 trades, +2,350, 66.7% win) - far too
thin to trust; the 30-day re-test is the number to use, same sample-size
lesson as v1.

**21 trades, 12 wins/9 losses (57.1%), net +Rs50,100, avg +Rs2,386/trade.**

| # | Side | Entry | Exit | RSI(1h) entry | RSI(1h) exit | PnL |
|---|---|---|---|---:|---:|---:|
| 1 | LONG | 08/12 10:15 | 08/12 10:45 | 50.3 | 50.3 | +2,200 |
| 2 | SHORT | 08/12 12:15 | 08/12 12:45 | 46.1 | 46.1 | -3,550 |
| 3 | LONG | 08/12 15:15 | 08/13 09:15 | 53.0 | 53.8 | +10,000 |
| 4 | SHORT | 08/14 13:15 | 08/14 13:45 | 49.7 | 49.7 | **+28,200** |
| 5 | LONG | 08/17 14:15 | 08/17 14:45 | 51.2 | 51.2 | -1,950 |
| 6 | SHORT | 08/18 10:15 | 08/18 10:45 | 47.0 | 47.0 | +850 |
| 7 | LONG | 08/18 12:15 | 08/18 12:45 | 52.6 | 52.6 | -2,050 |
| 8 | SHORT | 08/19 09:15 | 08/19 09:45 | 49.1 | 49.1 | +150 |
| 9 | LONG | 08/25 15:15 | 08/26 09:15 | 58.3 | 62.6 | +5,900 |
| 10 | SHORT | 08/28 10:15 | 08/28 10:45 | 49.6 | 49.6 | -150 |
| 11 | LONG | 08/28 11:15 | 08/28 11:45 | 50.3 | 50.3 | -3,200 |
| 12 | SHORT | 08/28 12:15 | 08/28 12:45 | 46.5 | 46.5 | -400 |
| 13 | LONG | 08/31 12:15 | 08/31 12:45 | 51.7 | 51.7 | -500 |
| 14 | SHORT | 08/31 13:15 | 08/31 13:45 | 48.9 | 48.9 | +3,100 |
| 15 | LONG | 09/04 10:15 | 09/04 10:45 | 52.8 | 52.8 | +7,750 |
| 16 | SHORT | 09/04 12:15 | 09/04 12:45 | 49.3 | 49.3 | +3,950 |
| 17 | LONG | 09/07 14:15 | 09/07 14:45 | 54.0 | 54.0 | -3,750 |
| 18 | SHORT | 09/08 10:15 | 09/08 10:45 | 48.4 | 48.4 | +1,200 |
| 19 | LONG | 09/18 12:15 | 09/18 12:45 | 57.0 | 57.0 | +2,100 |
| 20 | SHORT | 09/22 13:15 | 09/22 13:45 | 47.6 | 47.6 | +2,950 |
| 21 | LONG | 09/23 09:15 | 09/23 09:45 | 52.1 | 52.1 | -2,700 |

## Caveats

- **Trade #4 (+Rs28,200) is 56% of total profit.** Excluding it: still
  positive at +Rs21,900 over 20 trades (+Rs1,095/trade) - meaningfully
  less outlier-dependent than v1's stop-loss tail risk, but the single
  biggest trade is still doing a lot of work.
- **No stop-loss at all** in this design - the only downside protection
  is the time cap. A bad entry that keeps drifting against the position
  for the full 30/45/60 minutes has no other exit.
- Low frequency: ~21 signals in 30 trading days (~0.7/day) - 1-hour RSI
  crossing 50 is inherently a rare event (only ~6-7 hourly bars/session).
- Same single-window caveat as every backtest in this line of work - one
  30-trading-day stretch, not a generalized/walk-forward result.

## Open questions for the next session

- Since 70/30 never fires, does a tighter exit level (e.g. 60/40) ever
  trigger, and if so does it improve or hurt the result vs. the pure
  time-boxed version?
- Would adding a stop-loss (currently absent) meaningfully change the
  result, or does the short (30/45min) hold already limit downside enough
  that a stop wouldn't have fired differently from what already happened?
- Untested: whether this 1h-RSI/5min-timing pattern generalizes to other
  F&O symbols, and whether trade #4's size is repeatable or a one-off.
