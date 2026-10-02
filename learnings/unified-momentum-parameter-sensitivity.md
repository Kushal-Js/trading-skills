# Unified Momentum: parameter sensitivity (3 Oct 2026)

User request: "go through my deployed Unified Momentum strategy and give some suggestion on how to increase more profits and cut down bigger losses."

**Finding: none of 17 single-setting changes beat the deployed config in BOTH August and September.** The deployed settings sit on a plateau of this (in-sample) data. Tweaking exits, slots, gates or start times is not where more profit is. Read this before re-testing any of these knobs.

## Setup

- Deployed UM rules: engine A = 2 call slots with the PUT hedge; engine B = 2 put slots; NIFTY chop gate 0.10; premium >= Rs 10; Rs 1.2L cash.
- Lists: walk-forward weekly HYBRID picks (live-like, no hindsight).
- Window: 3 Aug - 1 Oct 2026, 43 sessions. Cache only, modelled slippage.
- Script: scratch `um_improve.py`. It patches `backtest_super_bollinger_universe_30day.simulate` (breakeven lock) and `research_unified_super_strategy.engine_b` (candle-close Supertrend exit), then reruns `U.portfolio`.
- Baseline reproduces the known numbers: +149,360 (Aug +57,927 / Sep+1 Oct +91,434), 129 trades, 38% won, worst day -5,964, max dd -13,712. The top 2 stocks give 65% of the profit; without them it is +52,644.

## Results (change vs deployed, Rs)

| Change | Total | Aug | Sep | Max dd | Worst day |
|---|---:|---:|---:|---:|---:|
| No PUT hedge on calls | +3,801 | +1,507 | +2,293 | 0 | -7,067 (worse) |
| Engine B off | -16,986 | -7,187 | -9,799 | +999 | -6,851 |
| Engine B entries from 09:45 / 10:00 | -12.3k / -14.7k | -8,812 | -3.5k / -5.9k | +346 | same |
| Engine A entries from 09:45 | -18,387 | -8,905 | -9,483 | 0 | same |
| A 3 slots (B 2) | -12,272 | -1,657 | -10,617 | -4,015 | -9,129 |
| A 3 slots, B 1 | -18,185 | -3,569 | -14,617 | -3,919 | -7,511 |
| Chop gate 0.15 | -16,749 | +2,315 | -19,066 | +995 | -5,749 |
| Chop gate off | -5,674 | -10,065 | +4,390 | -1,190 | -8,717 |
| Daily stop after -6,000 realised | -6,282 | -6,411 | +128 | +128 | same |
| Engine B Supertrend exit on 5-min close (not tick) | -1,964 | +621 | -2,586 | +192 | same |
| Engine A breakeven locks +300 | -14,991 | -18,170 | +3,178 | -431 | -4,798 |
| Engine A breakeven locks +600 | -61,121 | -21,915 | -39,207 | -7,884 | -7,838 |
| Engine A max loss 3,500 | -25,644 | +3,348 | -28,993 | -2,779 | -6,946 |
| Engine A max loss 5,500 | +225 | +1,338 | -1,113 | +376 | -5,114 |
| Breakeven armed at +1,000 | -21,918 | -14,938 | -6,981 | -5,601 | -7,417 |
| Breakeven armed at +2,000 | -3,455 | +1,721 | -5,178 | -6,328 | -11,479 |
| Cash Rs 1,07,751 (actual funds 1 Oct) | 0 | 0 | 0 | 0 | 0 |
| Cash Rs 90,000 | -9,845 | -17,097 | +7,251 | +3,475 | -5,743 |

## Takeaways

- **Hedge:** about neutral. It costs ~Rs 3.8k over 43 days and saves ~Rs 1.1k on the worst day. Keep it as cheap crash insurance; it does not add profit.
- **Engine B pays for itself:** +17k, with both halves positive. Its 09:20-09:45 entries are part of that, so keep its start time.
- **No change for:** tick vs candle-close Supertrend exit (noise), max loss 5,500 (neutral), chop gate 0.10 (0.15 and off each win one month and lose the other).
- **Breakeven-plus lock:** raises the win rate (38% -> 50-59%) but cuts the winners that dip and run, so it costs profit. Same lesson as the 29-30 Sep finding that the call must not be trailed.
- **More slots** add worse trades and a deeper drawdown.
- **Rs 1.08L of cash is enough;** below ~Rs 1L entries start to be skipped.
- **The 5 worst days were -5,964 / -5,114 / -4,819 / -4,489 / -4,430.** The 20k disaster brake never comes close. A 12-15k brake would cost nothing in this sample and halve the malfunction tail; check intraday open-P&L lows first.
- **Where profit can still come from:** stock selection (65% of profit from 2 stocks; the 2 Oct old-vs-new list test moved +-16k on list changes alone) and execution (the edge per trade, about Rs 1,160, is roughly the modelled slippage). Neither is a setting.

Caveat: this is all in-sample (the rules were designed on the same Aug-Sep data). A plateau here shows that nothing obvious is left on the table; it does not prove the edge holds out-of-sample.
