# Off-session ticks became candles (and an index's flat after-hours values fed every indicator)

**Found 2 Oct 2026** while validating the weekly watchlist refresh: Unified Momentum's NIFTY chop gate read an
efficiency ratio of 0.0 (= "fully choppy", gate CLOSED) from a candle stamped 1 Oct **18:10** IST.

**Cause.** The WebSocket candle builder (traderBoy Swing/candle_feed.py) turned EVERY tick into 5-min bars:
- NSE indices: the feed keeps repeating the closing value after 15:30 -> flat, zero-volume bars (NIFTY 33 to 18:10
  and BANKNIFTY 62 to 20:35 on 1 Oct; 44 each to 19:05 on 29 Sep);
- NSE stocks: one extra bar at 15:50 from the closing-price session (15:40-16:00), sometimes a pre-open bar.
The bars were persisted and reloaded on every restart, so between one session's close and the next open every
strategy's Supertrend / EMA / Bollinger / ATR / efficiency ratio read data the REST history - and every backtest -
never has (REST and the backtests use 09:15-15:25 bars only).

**Fix (traderBoy `514e31d`).** NSE equities and indices keep ticks of the regular session only (bar starts 09:15 ..
15:25): the first tick after 15:30 still closes the 15:25 bar but starts no bar, wakes no tick listener and does
not mark the feed fresh; bars restored from disk are filtered the same way. MCX keeps its long session. The UM gate
also reads session bars only. After the deploy the gate read ER 0.195 (open) from Thursday's 15:25 bar.

**Lessons**
1. A live candle builder must apply the same session calendar as the historical data the strategy was tested on -
   otherwise live indicators silently differ from the backtest.
2. Flat zero-volume bars are a red flag in any series: they shrink ATR / Bollinger width and drive an efficiency
   ratio to 0.
3. Check what the persisted bar files actually contain (time-of-day histogram per symbol), not just bar counts.
