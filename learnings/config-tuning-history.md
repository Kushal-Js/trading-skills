# Config-tuning history

Measured effects of specific Swing/Options config changes, backed by
backtests run against the actual live entry/exit logic (not a generic
strategy simulation) - see `backtest-methodology.md` for how these are
kept faithful to production.

## Swing Supertrend period/multiplier: 10/3.0 (current) vs 5/5.0 - SONACOMS 30-day

**Date:** 25 Sep 2026
**Symbol:** SONACOMS (NSE equity, OPTIONS basket, not an index symbol - v3
Day Range branch never fires for it either way)
**Question:** does tightening the Supertrend to period=5/multiplier=5.0
(from the live default period=10/multiplier=3.0) improve P&L?

**Method:** `traderBoy/backtest_sonacoms_supertrend_5_5_vs_current_30day.py`
- byte-for-byte port of `Swing/trading_engine.py`'s live v2/v3 entry filter
(5m Supertrend cross + 15m Supertrend/regime/gap-widening filter leg) and
exit ladder (MAX_LOSS_HIT -> TARGET_HIT -> PROFIT_PROTECTION_HIT ->
STOP_LOSS_HIT -> SUPERTREND_REVERSAL), replayed candle-by-candle over the
last 30 trading days (2026-08-14 to 2026-09-25) on continuous multi-day
candles. Both variants share the same regime/EMA/volume-floor computation
and the same option contract/price data - only the Supertrend
period/multiplier (applied to BOTH the 5-min entry/exit signal and the
15-min filter leg, since config.py only has one such value reused across
both timeframes) differs between runs.

**Result:**

| Variant | Trades | Wins | Losses | Win rate | Net P&L |
|---|---|---|---|---|---|
| CURRENT (period=10, mult=3.0) | 10 | 8 | 2 | 80.0% | **+Rs 17,946** |
| NEW (period=5, mult=5.0) | 8 | 3 | 5 | 37.5% | **-Rs 8,146** |

Delta: **-Rs 26,092** for the tighter (5/5.0) setting.

**Why it's worse, mechanically:** a shorter period (5 vs 10) makes the
Supertrend react to fewer bars, so it whipsaws inside the same trend far
more - 5 of the 8 NEW-variant trades exited via SUPERTREND_REVERSAL
(all losses) vs only 3 of 10 for CURRENT, and none of those 3 CURRENT
reversals were as costly. The larger multiplier (5.0 vs 3.0) does widen
the band, which should cut *some* noise, but not enough to offset the
much shorter lookback - net effect across this window was a clearly worse
entry/exit rhythm, not a wash.

**Scope of this finding:** one symbol (SONACOMS), one 30-day window,
OPTIONS basket. Not evidence either way for other watchlist symbols, MCX
symbols (different volume-floor gate), or the v3 Day Range branch (index
symbols only, never exercised here). Treat as a data point against
lowering the Supertrend period this aggressively for equity swing entries,
not a general verdict on all period/multiplier combinations.

**Outcome:** live `Swing/config.py` SUPERTREND_PERIOD/SUPERTREND_MULTIPLIER
left unchanged (10/3.0) - user asked for the comparison, not a deploy, and
the backtest itself argues against the change.
