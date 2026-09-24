# NIFTY options strategy v3: Swing's real v2-combined entry logic, 1-min fast layer (24 Sep 2026, user request)

**Status: exploratory backtesting only - nothing here is deployed live.**
No Swing code or config was touched - this is a standalone what-if
backtest in `traderBoy` (repo root), paper/simulation only, no real
orders placed. Named "v3" by the user, following the naming convention
started with [[ashokley-futures-strategy-exploration-orb-v1]] (v1) and
[[ashokley-futures-1hour-rsi-5min-timing-v2]] (v2). See also
[[nifty-options-v1-v2-adaptation]] for how v1/v2 were separately adapted
to NIFTY options, and its documented expiry-ceiling constraint, which
applies here too (see below).

## What v3 is

Swing's REAL DEPLOYED v2-combined entry logic (the actual live rule, not
a simplified stand-in), extended with two conditions the user specified
new for this exercise (Trend-aware Filter, Day Range Bull/Bear), with the
FAST timeframe moved from Swing's native 5-min down to 1-min, applied to
NIFTY-50 index options instead of Swing's real stock watchlist
(ADANIPORTS, COPPER, ANGELONE, COALINDIA). The exit ladder is Swing's
REAL exit ladder, called via the REAL `Swing.position_store` functions
(not a reimplementation) - same MAX_LOSS_HIT / TARGET_HIT /
PROFIT_PROTECTION_HIT / STOP_LOSS_HIT / SUPERTREND_REVERSAL priority
order as production, with SUPERTREND_REVERSAL now checked on the 1-min
Supertrend (was 5-min) per the user's explicit instruction.

Full interpretation-call documentation (regime-as-persisting-state vs.
one-bar-event, what "Day Range Bull" bundles together, RSI phrasing) is
in the script's own docstring - not repeated here, see
`backtest_nifty_options_swing_v2_1min.py`.

**Entry formula, exactly as specified:**
```
BULLISH: ((15min ST green) OR (Trend-aware Filter Bullish) OR (Regime Bullish))
         AND (1min ST just crossed above)
         OR (Day Range Bull AND Trend-aware Filter Bullish)

BEARISH: mirror (red / Bearish / crossed-below / Day Range Bear)
```
Signal computed off NIFTY's own SPOT index (1-min fast + 15-min slow,
regime EMA200, Supertrend(10,3.0) on both, RSI(14) for Day Range).

## Result: 30-day window (2026-08-12 to 2026-09-23)

**140 trades (96 skipped at the far edge, 08/12-08/21 - see Expiry
Ceiling below), 139 closed + 1 still open, 50 wins/89 losses (36.0%),
net +Rs3,929, avg +Rs28/trade.**

**Exit-reason breakdown - the real story of this result:**

| Exit reason | Trades | Total PnL | Avg PnL |
|---|---:|---:|---:|
| TARGET_HIT | 16 | +43,680 | +2,730 |
| SUPERTREND_REVERSAL | 122 | -34,843 | -286 |
| MAX_LOSS_HIT | 1 | -4,908 | -4,908 |

16 target-hits carry the entire strategy. 122 Supertrend-reversal exits
(88% of closed trades) bleed small losses on average - the strategy only
survives because winners are ~10x the size of the average loser. Strip
out the 16 target-hits and the other 123 trades net **-Rs39,751**.

**Date-wise (22 resolved trading days, 08/24-09/23):**

| Date | Trades | W/L | Net PnL |
|---|---:|---|---:|
| 08/24 | 3 | 2/1 | +2,213 |
| 08/25 | 6 | 2/4 | -2,028 |
| 08/26 | 6 | 2/4 | -2,135 |
| 08/27 | 5 | 3/2 | +2,984 |
| 08/28 | 4 | 1/3 | +1,189 |
| 08/31 | 4 | 2/2 | +201 |
| 09/01 | 5 | 3/2 | +3,793 |
| 09/02 | 8 | 2/6 | -2,304 |
| 09/03 | 9 | 2/7 | -1,765 |
| 09/04 | 6 | 2/4 | +1,895 |
| 09/07 | 3 | 3/0 | +2,418 |
| 09/08 | 7 | 2/5 | +3,305 |
| 09/09 | 7 | 5/2 | +3,894 |
| **09/10** | 7 | 1/6 | **-6,376** |
| **09/11** | 9 | 0/9 | **-5,996** |
| 09/15 | 6 | 2/4 | +3,000 |
| 09/16 | 8 | 2/6 | -3,516 |
| 09/17 | 8 | 5/3 | +4,917 |
| 09/18 | 10 | 2/8 | -2,945 |
| 09/21 | 5 | 2/3 | +1,030 |
| 09/22 | 6 | 2/4 | +604 |
| 09/23 | 7 | 3/4 | -449 |

09/10-09/11 was a brutal two-day stretch: -Rs12,372 combined across 16
trades, only 1 win. Same whipsaw-cluster failure mode as the VWAP+RSI
scalper's 09/22 loss cluster (see
[[ashokley-futures-1hour-rsi-5min-timing-v2]]'s own earlier finding of
this exact pattern on a different symbol/strategy).

Full 139-trade log saved: `results_nifty_options_swing_v2_1min_30day.json`.
Script: `backtest_nifty_options_swing_v2_1min.py`, CLI arg
`test_days_back` (default 30).

## Expiry-ceiling constraint (same issue as v1/v2's NIFTY adaptation, more severe here)

**96 of the ~236 signals this rule generated over the full 30 days were
SKIPPED** - the resolver could only find usable option data from
2026-08-24 onward, not the full 30-day window. Every skip was in
08/12-08/21 (the oldest ~8 trading days). This is the same expiry-
ceiling issue documented in [[nifty-options-v1-v2-adaptation]] (Dhan
delists expired weekly contracts), but it bites HARDER here because v3
generates far more signals per day (~6-10x v1/v2's frequency) - more
signals means more chances to land in the unresolvable window.

**Same "wrong expiry cycle" distortion also applies, and matters more
here**: every trade in this backtest, including the earliest resolved
ones (08/24), trades the 2026-09-29 expiry - a contract that was ~5 weeks
from its own expiry at that point, not a genuine near-week option.
Because v3 trades far more frequently than v1/v2, a much larger share of
its total trade count is affected by this distortion. Treat this result
as a directional signal at best, not a real economic estimate of what
weekly-options scalping on this rule would actually produce.

## Open questions for the next session

- Whether restricting to the cleanly-resolved 08/24-onward window (22
  trading days, no skips) changes the picture materially from the full
  30-day number - not yet re-run with that restriction.
- Whether a same-day-loss circuit breaker (flagged as working well for
  the VWAP+RSI scalper, see [[ashokley-futures-1hour-rsi-5min-timing-v2]])
  would prevent the 09/10-09/11 cluster here too.
- The strategy's 88%-of-trades-are-small-losers profile means it's
  extremely sensitive to the 16 target-hit trades being real/repeatable -
  worth checking whether they cluster in a few volatile days or are
  genuinely spread out (date-wise table above suggests they're fairly
  spread: 08/24, 08/27, 08/28, 09/01 x2, 09/04, 09/08, 09/09, 09/15 x2,
  09/17 x3, 09/22, 09/23 - 13 different days out of 22, so not a single-
  day artifact, a mild positive sign).
- Untested: whether the "Regime Bullish/Bearish as persisting state" vs.
  "one-bar-event" interpretation call (see script docstring) materially
  changes results if implemented the other way.
