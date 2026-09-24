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

## Cross-symbol test: VEDL futures (24 Sep 2026, same 30-day window, 30-min hold)

Script now takes SYMBOL as a CLI arg (`symbol test_days_back
max_hold_minutes`, was hardcoded to ASHOKLEY before) specifically to run
this cross-symbol check.

**VEDL SEP FUT, lot_size=1,150: 13 trades, 10 wins/3 losses (76.9%), net
+Rs4,945, avg +Rs380/trade.**

| # | Side | Entry | Exit | RSI(1h) entry | RSI(1h) exit | PnL |
|---|---|---|---|---:|---:|---:|
| 1 | LONG | 08/20 10:15 | 08/20 10:45 | 58.0 | 58.0 | +862 |
| 2 | SHORT | 08/25 14:15 | 08/25 14:45 | 50.0 | 50.0 | -1,150 |
| 3 | LONG | 08/25 15:15 | 08/26 09:15 | 53.2 | 54.7 | +2,875 |
| 4 | SHORT | 08/31 10:15 | 08/31 10:45 | 42.3 | 42.3 | +345 |
| 5 | LONG | 09/04 14:15 | 09/04 14:45 | 51.6 | 51.6 | +345 |
| 6 | SHORT | 09/07 10:15 | 09/07 10:45 | 39.8 | 39.8 | +230 |
| 7 | LONG | 09/08 10:15 | 09/08 10:45 | 54.6 | 54.6 | -517 |
| 8 | SHORT | 09/08 12:15 | 09/08 12:45 | 47.0 | 47.0 | +690 |
| 9 | LONG | 09/09 10:15 | 09/09 10:45 | 50.8 | 50.8 | +575 |
| 10 | SHORT | 09/10 12:15 | 09/10 12:45 | 48.5 | 48.5 | +920 |
| 11 | LONG | 09/18 10:15 | 09/18 10:45 | 56.8 | 56.8 | +575 |
| 12 | SHORT | 09/21 15:15 | 09/22 09:15 | 46.7 | 47.6 | -2,242 |
| 13 | LONG | 09/22 10:15 | 09/22 10:45 | 58.2 | 58.2 | +1,438 |

**Cross-symbol comparison (identical window, identical params):**

| | Trades | Win Rate | Net P&L | Avg P&L/trade | Avg P&L/trade / lot_size (per-share edge) |
|---|---:|---:|---:|---:|---:|
| ASHOKLEY (lot 5,000) | 21 | 57.1% | +50,100 | +2,386 | Rs0.48/share |
| VEDL (lot 1,150) | 13 | **76.9%** | +4,945 | +380 | Rs0.33/share |

**Verdict: the strategy generalizes - net profitable on both symbols,
with VEDL actually posting a HIGHER win rate.** But the per-share edge is
smaller on VEDL (Rs0.33 vs Rs0.48) - ASHOKLEY's own +28,200 single trade
inflates its average; VEDL's best trade is only +2,875. Also confirms the
dead-code finding is not ASHOKLEY-specific: every single VEDL exit is
ALSO `MAX_HOLD_TIME_EXIT` - RSI(14) on 1-hour didn't reach 70/30 for VEDL
either in this window. And VEDL fires fewer signals (13 vs 21) over the
identical 30 trading days - its 1-hour RSI crosses 50 less often.

## Multi-symbol x multi-holdtime sweep (24 Sep 2026, user request: "run v2
## against few FnO using timeframes 30/45/60 mins")

Same 30-day window (2026-08-12 to 09-23) for every cell. Added
TATASTEEL, SBIN, HDFCBANK to the ASHOKLEY/VEDL pair already tested, and
filled in VEDL's missing 45/60min runs.

| Symbol | Hold | Trades | Win Rate | Net P&L | Avg/trade | Per-share edge |
|---|---:|---:|---:|---:|---:|---:|
| ASHOKLEY (lot 5,000) | 30 | 21 | 57.1% | +50,100 | +2,386 | +Rs0.477 |
| ASHOKLEY | 45 | 21 | 61.9% | +45,450 | +2,164 | +Rs0.433 |
| ASHOKLEY | 60 | 21 | 47.6% | +28,850 | +1,374 | +Rs0.275 |
| VEDL (lot 1,150) | 30 | 13 | 76.9% | +4,945 | +380 | +Rs0.330 |
| VEDL | 45 | 13 | 53.8% | +3,393 | +261 | +Rs0.227 |
| VEDL | 60 | 13 | 46.2% | +5,980 | +460 | +Rs0.400 |
| **TATASTEEL (lot 2,750)** | 30 | 33 | 42.4% | **-2,502** | -76 | -Rs0.028 |
| **TATASTEEL** | 45 | 33 | 39.4% | **-3,685** | -112 | -Rs0.041 |
| **TATASTEEL** | 60 | 33 | 36.4% | **-12,100** | -367 | -Rs0.133 |
| SBIN (lot 750) | 30 | 17 | 58.8% | +4,500 | +265 | +Rs0.353 |
| SBIN | 45 | 17 | 58.8% | +3,900 | +229 | +Rs0.305 |
| **SBIN** | 60 | 17 | 41.2% | **-8,025** | -472 | -Rs0.629 |
| HDFCBANK (lot 650) | 30 | 17 | 64.7% | +2,665 | +157 | +Rs0.242 |
| HDFCBANK | 45 | 17 | 58.8% | +5,785 | +340 | +Rs0.523 |
| **HDFCBANK** | 60 | 17 | 58.8% | **+8,808** | +518 | **+Rs0.797 (best of all 15 cells)** |

**Three findings that materially change the picture from the two-symbol
version of this section:**

1. **No universal best hold time - it's symbol-specific.** ASHOKLEY and
   SBIN both prefer 30min and degrade as hold time increases (SBIN flips
   net NEGATIVE at 60min). HDFCBANK does the OPPOSITE - monotonically
   IMPROVES from 30 to 45 to 60min, and its 60min per-share edge
   (+Rs0.797) is the best result in the entire 15-cell matrix, beating
   even ASHOKLEY's best. VEDL is non-monotonic (dips at 45, recovers at
   60). A single fixed MAX_HOLD_MINUTES tuned on ASHOKLEY does NOT
   transfer as "the right value" to every symbol.
2. **TATASTEEL breaks the "v2 generalizes" claim from the two-symbol
   version of this doc.** Net negative at ALL THREE hold times, and
   monotonically worse the longer it's held (-76 -> -112 -> -367/trade).
   This is the first symbol where v2 fails outright rather than just
   underperforming - should be EXCLUDED from any deployment of this
   strategy, not parameter-tuned into profitability.
3. **SBIN's 60min collapse (+265/trade at 30min -> -472/trade at 60min)
   is the sharpest single reversal in the matrix** - direct evidence
   against "longer hold = safer"; for some symbols it's the opposite.

**Practical implication for any future live use:** v2 works on 4 of 5
symbols tested here, but the profitable hold time isn't a fixed constant
across symbols - it would need per-symbol tuning, which is a real
weakness for a rule meant to run identically across a watchlist. Treat
"v2 with a single global MAX_HOLD_MINUTES" as unproven at watchlist scale
until this is investigated further (see open questions).

## Open questions for the next session

- Since 70/30 never fires (confirmed on ASHOKLEY and VEDL), does a
  tighter exit level (e.g. 60/40) ever trigger, and if so does it improve
  or hurt the result vs. the pure time-boxed version? Still untested on
  TATASTEEL/SBIN/HDFCBANK too.
- Would adding a stop-loss (currently absent) meaningfully change the
  result, or does the short (30/45min) hold already limit downside enough
  that a stop wouldn't have fired differently from what already happened?
  Particularly relevant for SBIN's 60min collapse and TATASTEEL's losses.
- **[RESOLVED, negatively]** whether a single global hold time is the
  right design - no, it isn't; see the multi-symbol sweep above. Worth
  investigating WHY HDFCBANK prefers long holds while ASHOKLEY/SBIN
  prefer short ones (volatility regime? trend persistence? needs its own
  investigation, not guessed at here).
- Whether TATASTEEL's failure is specific to this window or a durable
  property of the symbol/strategy combination - only one 30-day sample
  tested so far, same standing caveat as everything else in this repo.
