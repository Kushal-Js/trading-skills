# Composite breakout/momentum score + 27 Sep watchlist full replacement

User request: extend the `ATH()` function (see `52-week-high-screen-swing-
vs-bollinger-30day-backtest.md` for its origin) into a scored composite
combining Minervini's Trend Template, 52-week-high proximity, loosened
Stage-2 breakout criteria, and a few extra signals, then rank the full
F&O universe and rebalance both watchlists to the top 15.

## Why a score instead of another hard AND filter

Every strict multi-condition "Stage 2 breakout" attempt this session
(see `traderBoy/screen_fno_stage2_breakout.py`'s own v1-v4 history)
produced 0-2 hits on any given day - a real, expected property of
combining several genuinely rare conditions (a fresh EMA cross, a tight
prior base, AND a volume-confirmed breakout rarely all land on the same
day for the same stock). A SCORE lets the whole universe rank by "how
close/how good" instead of a binary pass, which is what actually answers
"which 15 stocks look best today."

## The formula (100 points, `traderBoy/screen_fno_top15_composite.py`)

1. **Minervini structural score (25 pts)** = `(criteria_passed/9) * 25`.
   Reuses a faithful Minervini Trend Template implementation (150/200-day
   SMA, 200-SMA required to be rising over the last month, 50-SMA
   alignment, 30%-above-52w-low/25%-of-52w-high) - see `minervini-trend-
   template.md`. This is the base "worth watching at all" gate.
2. **52-week-high proximity band (20 pts)** - user's own "1% to 10% from
   the high is fine" framing, implemented as a band: full 20 pts at/above
   the 52w high itself, linearly down to 0 at 10% below it. Rewards both
   an already-in-progress breakout and a stock closing in on one.
3. **Loosened Stage-2 criteria (25 pts)** = `(passed/7) * 25`, at the
   thresholds this session's own combo grid showed actually mattered
   (12% tightness, 15-day breakout window, 1.2x volume) - the 20/50 EMA
   cross is EXCLUDED from this gate and scored separately (see #4) since
   requiring both a fresh cross AND an already-loosened breakout
   simultaneously produced zero hits in the combo grid - a real
   structural tension (a stock rarely does both in the same short
   window).
4. **Fresh 20/50 EMA cross bonus (10 pts flat)** - genuinely rare (~2-4%
   of the universe on any run this session), rewarded as a bonus rather
   than gated so it doesn't zero out every already-established trend.
5. **ADX(14) trend-strength bonus (10 pts, scaled)** = `min(ADX,25)/25*10`.
6. **Momentum-consistency bonus (10 pts)** = 2 pts per timeframe (of
   1W/1M/3M/6M/1Y) where `ATH()`'s own regression-based robust momentum
   is positive - rewards broad-based momentum over one cherry-picked
   window.

## Result (27 Sep 2026 run, 199/210 F&O stocks scored)

All top 15 passed Minervini's full Trend Template (9/9) - the composite
naturally converges there since it's a quarter of the score. Only
**ZYDUSLIFE and MOTHERSON** passed all 7 loosened Stage-2 conditions
outright; everyone else in the top 15 is "structurally excellent, still
waiting on the actual breakout+volume trigger." **PHOENIXLTD** was the
only stock with a genuinely fresh 20/50 EMA cross - the "just starting"
profile vs. everyone else's "already established" one.

Top 15 by score: ZYDUSLIFE (85.6), SONACOMS (81.8), DIVISLAB (80.4),
AUROPHARMA (79.4), MOTHERSON (76.4), APLAPOLLO (73.3), MCX (73.2),
BOSCHLTD (73.0), LAURUSLABS (72.6), RBLBANK (72.0), RADICO (71.4),
PHOENIXLTD (69.3), MOTILALOFS (68.4), NYKAA (68.2), GLENMARK (66.7).

## 30-day backtest validation before deploying

Backtested all 15 against both Swing and Bollinger (options basket, real
Dhan candle data), then explicitly re-backtested the ACTUAL then-current
watchlist (16 equities, post-26-Sep-rebalance) for a fair comparison
rather than reusing the earlier pre-rebalance baseline:

| | Then-current watchlist (16) | Composite top-15 |
|---|---:|---:|
| Swing | +Rs 79,890 (60 trades, 60.0% WR) | +Rs 91,437 (59 trades, 62.1% WR) |
| Bollinger | +Rs 210,985 (498 trades, 56.9% WR) | +Rs 231,662 (440 trades, 56.8% WR) |

Composite top-15 ahead on both (+14.5% Swing, +9.8% Bollinger), but the
edge is smaller than the headline numbers suggest: 8 of the 16
then-current symbols were ALSO in the composite top-15 (ZYDUSLIFE,
SONACOMS, DIVISLAB, MOTHERSON, APLAPOLLO, MCX, BOSCHLTD, RBLBANK) - the
real comparison is the 7 non-overlapping names each side contributes.
**LAURUSLABS did most of the composite side's heavy lifting**: standalone
+Rs 58,140 under Bollinger (69.2% win rate, 39 trades) - by far the best
single-symbol result of any backtest run this session. Remove that one
name and the composite list's edge over the prior watchlist mostly
disappears.

**RADICO flagged and included anyway, per explicit user decision**: it
scores well on the composite (structurally sound, near its highs) but
lost money in the 30-day backtest under BOTH strategies (-Rs 1,200 Swing,
-Rs 930 Bollinger) - the one name in the top 15 where the forward-looking
technical score and the trailing 30-day P&L disagree. User's own call:
"trust the composite" (structural technical score, not a backtest-P&L
guarantee) rather than exclude it.

## Watchlist rebalance actually deployed

User's own instruction: full REPLACE (not incremental add) of the equity
portion of both watchlists with exactly these 15, explicitly keeping
COPPER/NATURALGAS/NIFTY/BANKNIFTY untouched (never scored by the
composite - a different asset class the ATH/Minervini/Stage2 machinery
doesn't cover) and explicitly including RADICO per the paragraph above.
CANBK, VBL, ASHOKLEY, BANDHANBNK, TORNTPHARM, DLF, LICHSGFIN (all present
on the prior 26-Sep watchlist, none in the top-15) were dropped.

See `TRADING_JOURNAL.md`'s 27 Sep entry for the deploy/restart timeline
and post-restart validation.
