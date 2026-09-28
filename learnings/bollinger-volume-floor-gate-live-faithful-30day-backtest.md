# Bollinger volume-floor gate: live-faithful 30-day portfolio backtest (28 Sep 2026)

**Question (user, 28 Sep 2026):** what does Bollinger's last 30 days look like
with the newly deployed volume-floor gate and every real condition/check?

**Script:** traderBoy `backtest_bollinger_live_faithful_volume_gate_30day.py`
(results cached at `history/bt_bollinger_live_faithful_20260928/results.json`).
Window 29 Aug - 28 Sep 2026 (28 Sep up to ~12:50 IST), the live 17-symbol
watchlist (15 stocks + NIFTY/BANKNIFTY).

## What it models (and doesn't)

Modeled: resting-order entry (pending trigger from bar i-1, filled on the first
1-min touch in bar i, same day, once per order); MAX_CONCURRENT_TRADES=5 across
all symbols (full book = symbol not evaluated, order stays live for the rest of
the bar); volume floor (NSE 1.2x / index 0.6x, last closed 5-min candle vs prior
20, fail-open); NIFTY expiry-day skip; Rs 5 premium gate; real 1-min option OHLC
(stocks: 29 SEP contract; NIFTY weekly / BANKNIFTY monthly: Dhan
`/v2/charts/rollingoption` at ATM-5..+5, tracked at the fixed entry strike);
broker SL-L via `broker_stop_trigger_and_limit`, tick-rounded, filling AT the
limit if the bar reaches it; software trailing (arms at stop_pct/3, trails
stop_pct/3, 1/5 steps); MAX_LOSS; Friday and index-daily 15:25 square-offs;
inverse-to-premium slippage on entry and every market exit.

Not modeled: funds check; `get_liquid_atm_option`'s liquidity walk (it picked a
different strike on 1 of 4 checked live entries - APLAPOLLO 2180 PE vs the
backtest's 2200 PE); WS staleness, restarts, API failures; the capacity
backlog's re-dispatch. Intraminute order is unknown, so adverse moves are
checked first each minute.

## Results

Two trailing-stop variants bracket the intraminute uncertainty: "high" updates
the trailing high-water mark from each 1-min high (ticks do see it, but the
print may be at the ask), "close" from 1-min closes only.

| Variant | Trades | Win % | Net Rs | PF | Avg win | Avg loss | Max DD | Green days |
|---|---|---|---|---|---|---|---|---|
| Gate OFF, best=high | 346 | 57% | +71,033 | 1.66 | 906 | -721 | -5,684 | 14/20 |
| **Gate ON (live), best=high** | **99** | **62%** | **+25,432** | **2.24** | 754 | -541 | **-3,616** | 12/19 |
| Gate OFF, best=close | 346 | 56% | +84,095 | 1.70 | 1,051 | -801 | -9,402 | 14/20 |
| **Gate ON (live), best=close** | **99** | **59%** | **+22,355** | **1.81** | 862 | -674 | -6,452 | 11/19 |

Gate ON skips 274 entries. Median hold is about 4 minutes. Index contributes
+Rs 4-5k in every variant.

## Findings

1. **The gate improves trade quality but cuts total profit by roughly 60-75%.**
   Win rate +3-5 pts, PF 1.66 -> 2.24 and max drawdown -5.7k -> -3.6k (high
   variant). But the ~247 entries it removes netted +Rs 45.6k (high) / +Rs 61.7k
   (close). On this window, Bollinger's low-volume entries were profitable on
   average. That is the opposite of the Options/Swing evidence the 1.2x
   threshold came from.
2. **Validated against live on 28 Sep.** The backtest's volume ratios for the two
   entries the live gate skipped (APLAPOLLO 12:39 = 1.01, ZYDUSLIFE 12:43 =
   0.74) match the live logs exactly; both lose in the backtest (-473, -580). The
   real PHOENIXLTD trade reproduces closely: backtest -Rs 385 vs real -Rs 367.50
   (vol 0.66 - the gate would have blocked it).
3. **Much more optimistic than earlier Bollinger research, and not reconciled
   yet.** `bollinger-backtest-lookahead-bias-entry-timing.md` found -Rs 66k with
   bar-close timing; `bollinger_research.py` found resting + 5% stop at about
   +Rs 11k (15 stocks, closes only, no premium gate, no broker SL-L, no
   capacity). The likely drivers are the SL-L fill model (fills near the
   trigger instead of the next minute's close), the Rs 5 premium gate, and
   capacity. Treat the absolute rupee figures with caution until an
   attribution run isolates them. The gate-ON vs gate-OFF comparison is the more
   robust result, because both runs share every assumption.
4. **One window only (20 trading days).** Don't tune the gate threshold on it.
   If revisited, test the threshold on a separate window.
