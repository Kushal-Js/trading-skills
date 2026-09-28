# Structure-break (COPPER's Swing rule) on the Bollinger watchlist: 30-day backtest (28 Sep 2026)

**Question (user):** backtest the Bollinger watchlist over the last 30 days using
the structure-break strategy COPPER trades live, with real entry, exit and
slippage conditions; show day-wise and trade-wise P&L.

**Script:** traderBoy `backtest_structure_break_bollinger_watchlist_30day.py`
(results: `history/bt_bollinger_live_faithful_20260928/structure_break_results_t35.json`).
Window 29 Aug - 28 Sep 2026 (28 Sep up to ~12:50 IST), 17 symbols (15 stocks + NIFTY/BANKNIFTY).

## Rules modeled (Swing's live COPPER path, applied to each symbol)

- **Signal:** `structure_break.compute_structure_break` on 5m, 15m and 1h. The 15m and
  1h bars are resampled from 5m on the 09:15 grid. Enter only when all three
  last-closed regimes agree, using the consumed rule from
  `structure_break_entry_signal`.
- **Entry:** market buy of the ATM CE/PE, at the close of the minute after the 5m bar
  closes.
- **Gates:** one position per symbol, `SWING_MAX_CONCURRENT_TRADES=5`, Swing volume
  floor (NSE 1.2x / index 0.6x), NIFTY expiry-day skip. A blocked entry is retried
  while the agreement lasts.
- **Exits:**
  - Broker SL-L: tighter of the 20% hard stop and the Rs 4,500 cap, limit gap 0.05.
  - MAX_LOSS Rs 4,500.
  - PROFIT_PROTECTION: exit on a 2% giveback once peak profit is above Rs 3,000.
  - TARGET +35% (`TARGET_PCT_NON_COPPER`).
  - Structure square-off or reversal when the agreement breaks.
  - Friday and index-daily 15:25 square-offs.
  - Stock positions carry overnight.
- **Slippage:** inverse-to-premium, on entry and every market exit.
- **Option prices:** real 1-min option OHLC. NIFTY/BANKNIFTY use the Dhan rollingoption
  data.

## Result: clearly negative

| | Trades | Win % | Net Rs | PF | Avg win | Avg loss | Max DD |
|---|---|---|---|---|---|---|---|
| Target 35% (live non-COPPER rule) | 256 | 38% | **-1,20,478** | 0.70 | 2,926 | -2,509 | -1,20,854 |
| Target 20% (COPPER's value) | 258 | 40% | -1,16,204 | 0.70 | 2,659 | -2,550 | -1,16,794 |

- **Green days:** only 5 of 20 (best: 15 Sep +24.7k; worst: 31 Aug -25.7k).
- **By exit:** PROFIT_PROTECTION +2.16L (73) and TARGET +61k (18) did not cover
  STOP_LOSS -1.53L (59), STRUCTURE_BREAK_SQUARE_OFF -1.51L (69) and MAX_LOSS
  -81k (17, avg -4,794; gaps overshoot the cap, worst -6,665).
- **Holds:** median 68 min; 42 trades held overnight.
- **By symbol:** only 7 of 17 were positive (OBEROIRLTY +10.7k best, MOTHERSON
  -38.8k worst). Index: 38 trades, -24.6k.
- **Consistent with COPPER's own real record:** -8.9k over 14 trades, 15-25 Sep. All
  5 of COPPER's STRUCTURE_BREAK_SQUARE_OFF exits lost on 24 Sep.

## Caveats

- **Indicator window:** it is computed once over the full 90-day 5m history.
  Live recomputes on rolling REST windows (5m 15d / 15m 30d / 1h 90d), so the
  EMA seeds differ slightly.
- **Not modeled:** funds check, liquidity-walk strike choice, restarts and API
  failures.
- **Sample:** a single 20-trading-day window.

## Takeaway

Don't roll structure-break out to the Bollinger watchlist. Its weak exit is
the "agreement breaks" square-off, which lands after about an hour of adverse
drift. The 3-timeframe agreement arrives late, so entries are near the end of a
move.
