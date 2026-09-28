# Bollinger backtests had a lookahead bias in entry timing; with live-faithful timing the edge disappears

**Date:** 28 Sep 2026
**Outcome:** Bollinger switched from REAL to PAPER before market open the same morning (see TRADING_JOURNAL.md, 28 Sep).

## The bug

Every Bollinger backtest script (`backtest_bollinger_vortex_9symbols_30day.py`, `backtest_bollinger_exact_slippage_9symbols_30day.py`, the MCX/index variant) walks 1-min bars and, at each minute `t`, maps `t` to the 5-min bar CONTAINING it (`idx_at_or_before(fast_ts, t)` on bar START timestamps). The pending-stop "fire" test uses that bar's full `high`/`low`, which are only known once the bar closes, yet the entry is filled at minute `t` inside the bar, before price has actually reached the trigger. In effect the backtest buys the breakout knowing it will happen.

Live cannot do this. `Bollinger/signals.py` only acts on the newest **closed** bar: `candle_feed.get_candles_dict` (WebSocket path) never returns a still-forming bucket, and `_get_intraday_series` explicitly trims the forming bar from the REST fallback. So a live entry happens at or after the fire bar's close.

## Evidence

`traderBoy/walkforward_selector_eval.py` ports the entry/exit logic as a signal stream plus a simulator with a `lag` switch. With the OLD timing it reproduces `backtest_bollinger_exact_slippage_9symbols_30day.py` exactly on the current 15-stock watchlist: identical trade count on every symbol, ₹155,652 vs ₹155,654 (rounding). Changing ONLY the entry moment to the bar close:

| Current 15-stock watchlist | Old timing | Live-faithful timing |
|---|---|---|
| Real ATM option premiums, Aug 28 - Sep 25 | +₹155,652 | **-₹66,254** |
| Synthetic premium tier, ~90 days, ~1,340 trades | +₹2,50,099, 49% win | **-₹2,43,044, 26% win** |

Largest swings: LAURUSLABS +₹51,414 -> -₹15,012; MCX +₹18,459 -> -₹11,790; ZYDUSLIFE +₹25,732 -> +₹3,229. Why it's so large: under the old timing the median trade lasted 4 minutes and 67% exited within 5 minutes. The 1% MIN_STOP_PCT is applied to the option PREMIUM (not the underlying), so stops and trailing stops sit a few ticks away; most of the "profit" was the in-bar move the backtest already knew about.

## Implications

- Every Bollinger backtest result recorded before this date is optimistic by a large, systematic margin: the original video backtest, the exact-slippage comparison vs Swing V3, the MCX/index run, the 27 Sep ₹2,10,985 max-loss-cap sweep, and the 28 Sep FNO-universe stock search. Their RELATIVE comparisons (stock A vs stock B) are also suspect, since the bias scales with how much each stock moves within a 5-min bar.
- Stock selection can't fix a negative-expectancy entry. The open question is whether ANY selection rule gives this entry a positive edge with correct timing: `walkforward_selector_eval.py` answers that walk-forward (ATH vs ATH+gates vs strategy-fit vs hybrid vs random), full-universe run pending.
- **Standing rule for any future intraday backtest here:** a signal computed from bar `i`'s OHLC may only be acted on at or after bar `i`'s close. Check this explicitly whenever a backtest steps a finer series (1-min) inside a coarser signal series (5-min) using bar START timestamps.
