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

## Measured, 30 Sep 2026 (added 2 Oct)

Live WS-built 5-min candles (`Swing/candle_feed` logs) against Dhan's official candles. The official candles were
taken from the REST 1-min cache aggregated to 5-min. The sample is 1,416 candles on 23 stocks; bars that overlap a
restart were left out.

| | WS = official |
|---|---|
| whole candle (O, H, L, C) | 25% |
| high | 74% (when different, WS lower in 366/368; median 0.017%, max 0.39%) |
| low | 68% (when different, WS higher in 448/450; median 0.020%, max 0.65%) |
| close | 73% |

The snapshot feed misses spikes, so the extremes come out one-sided.

What matters for the strategy is the pending order computed from each candle. I replayed `Bollinger/signals._replay`
with the newest candle official versus WS (1,393 candles):

- **Identical side, trigger and stop: 96.6%.**
- Side differs in 1.1%, mostly bearish.
- The trigger never differs when the side matches.
- The stop differs in 2.3%.

Meanwhile 23% of the Unified Momentum backtest's engine-A entries touch in the first minute of the candle. That is
the window in which live is still waiting for the official candle download. A WS-first pending order, corrected by
the official candle afterwards, would trade timing back for a ~1-3% chance of a briefly different order.
