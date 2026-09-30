# WS-built candles can miss a trigger touch that REST candles show (open question, 30 Sep 2026)

**Observation.** RBLBANK (Bollinger paper) had a pending BULLISH resting trigger at 409.2, armed at the 09:30 bar close.

| Source | High | Touch? |
|---|---|---|
| Dhan REST 1-min | **409.25 at 09:38** (46k volume, then back to 408) | Yes |
| `Swing.candle_feed`, WS-built 09:35 5-min bar | **408.45** | No |

So the live resting check never saw the touch and didn't enter. The signal replay (REST bars) marked it `fired`.

**Why it matters.** Super Bollinger's REAL tick entries use the same WS ticks, while every backtest uses REST bar highs. Live can therefore skip spike-only touches that the backtest counts. Spike touches often reverse, as this one did, so the effect on P&L isn't obviously negative, but it is a live-vs-backtest gap.

**Also seen the same morning (not a bug).** `_replay_pending_order_loop` can arm and fire a pending order on the **same** bar (NIFTY, 09:30 bar). The live resting mode can't take those. Check whether the Bollinger and Super Bollinger backtests counted such same-bar fires as trades; if they did, the backtests are optimistic.

**To do.** After a close, compare WS 5-min bars (`history/<date>_swing_candles_*.log`) with REST 5-min highs/lows for every watchlist symbol. Count the pending triggers REST would have filled and WS missed, and re-run the Super Bollinger backtest without same-bar arm+fire entries.
