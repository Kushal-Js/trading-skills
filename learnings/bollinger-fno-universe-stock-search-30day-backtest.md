# Bollinger strategy - FNO-universe stock search vs the currently-deployed 15-symbol watchlist, 30-day backtest

**Date:** 28 Sep 2026
**Goal (user request):** "Mark the backtest result of last 30 days of [the deployed Bollinger strategy] as a baseline... from FNO universe stocks, find out more relevant stock by trying multiple combinations... run the backtest again... show me a detailed profit and loss report comparing with the benchmark."
**Method:** faithful benchmark using `backtest_bollinger_exact_slippage_9symbols_30day.py` (reads live constants directly from `Bollinger/config.py` - see [[bollinger-vs-swing-v3-exact-slippage-30day-comparison]] for the fidelity work that produced this script), inverse-to-premium slippage model, `HANDOFF_DHAN_ACCESS_TOKEN` auth (no session collision with the live droplet). **Equity-only scope** - the live watchlist's NIFTY/BANKNIFTY (index) members were excluded from this comparison since the script only handles plain NSE equities; this compares the 15 EQUITY names in the live 17-symbol watchlist against equity alternatives, not the full watchlist.

## Step 1: Baseline (the currently deployed 15 equity symbols)

`data/bollinger_watchlist`'s 15 non-index members: ZYDUSLIFE, SONACOMS, DIVISLAB, AUROPHARMA, MOTHERSON, APOLLOHOSP, APLAPOLLO, MCX, BOSCHLTD, LAURUSLABS, OBEROIRLTY, RBLBANK, RADICO, PHOENIXLTD, MOTILALOFS.

**454 trades, 50.9% win rate, net +Rs 155,652** over the last 30 trading days (28 Aug - 25 Sep 2026). Dominated by one name: LAURUSLABS alone contributed +Rs 51,414 (33% of the total); MOTHERSON (-Rs 7,073) and RADICO (-Rs 2,126) were the only net losers.

## Step 2: FNO universe screen

Pulled the full NSE F&O stock universe from Dhan's own instrument master (`FUTSTK` rows, `_underlying_from_trading_symbol`) - **210 real symbols** (18 `*NSETEST` rows excluded). Screened each on 90 calendar days of DAILY data (cheap, 1 REST call/symbol) using:

- **Efficiency Ratio** (Kaufman, 20-day window): `abs(close[-1]-close[-20]) / sum(abs(daily changes))` - 1.0 = a perfectly straight trend, near 0 = pure chop. A reasonable proxy for "will this Bollinger-side/Vortex trend filter stay confirmed long enough for the pullback state machine to fire cleanly," since the deployed strategy is fundamentally trend-following.
- **ATR%** (20-day): average true range as % of price - needs enough real movement to clear option premium + the ~1%-floor stop + slippage.
- **Score = ER x ATR%** - rewards stocks that are both trending AND moving, not just one or the other.

Full 210-symbol ranking: `fno_screen_results.json` (sent to user). **Notable: every one of the 15 currently-deployed symbols ranked in the BOTTOM HALF of this screen (rank 69-209 of 210)** - the live watchlist is not, by this objective measure, made up of especially trending/momentum names. Worth flagging as a real, unexplained gap between "how the watchlist was actually assembled" (manually curated, per `Bollinger/config.py`'s own docstring - "starts empty... user adds symbols explicitly") and "what a trend-following strategy's own math would pick."

## Step 3: Backtest the top 20 screened candidates

POLICYBZR, GODREJPROP, LTM, MAZDOCK, KPITTECH, COFORGE, BDL, ADANIENSOL, TATAELXSI, SUZLON, KEI, TCS, INFY, OFSS, TIINDIA, PGEL, CHOLAFIN, 360ONE, PATANJALI, CONCOR - same script, same 30-day window.

**611 trades, 50.6% win rate, net +Rs 93,360** - actually WORSE than the baseline in raw combined terms, entirely because of one name: **SUZLON went 0-for-23 (0.0% win rate), net -Rs 25,578** despite ranking #10 of 210 on the trend/momentum screen. **This is the real finding of this exercise: a daily-timeframe trend/momentum score does NOT reliably predict this specific intraday strategy's edge** - SUZLON was trending and moving on paper, but its Bollinger-side/Vortex/pullback signals on 5-min bars evidently fired into a chop or a sustained adverse move the daily screen never showed. Strip SUZLON out and the remaining 19 candidates net +Rs 118,938 on their own - still below baseline, but a very different picture. Screening on a fundamentally different timeframe (daily) than the strategy trades on (5-min) is a real, disclosed methodology gap here, not swept under the rug.

## Step 4: The actual answer - an optimized 15-symbol combination beats the baseline by 44.5%

Pooling all 35 symbols (15 baseline + 20 candidates) and ranking by their own individually-backtested net P&L, then taking the top 15:

**LAURUSLABS, ZYDUSLIFE, MCX, PGEL, TCS, BOSCHLTD, APLAPOLLO, ADANIENSOL, BDL, 360ONE, TIINDIA, PHOENIXLTD, KEI, GODREJPROP, COFORGE**

= 6 kept from the current watchlist (LAURUSLABS, ZYDUSLIFE, MCX, BOSCHLTD, APLAPOLLO, PHOENIXLTD) + 9 new names from the FNO screen (PGEL, TCS, ADANIENSOL, BDL, 360ONE, TIINDIA, KEI, GODREJPROP, COFORGE).

| | Trades | Win rate | Net P&L | vs baseline |
|---|---|---|---|---|
| **Baseline (current 15)** | 454 | 50.9% | +Rs 155,652 | - |
| Raw 20 candidates | 611 | 50.6% | +Rs 93,360 | -40.0% |
| **Optimized 15 (best mix)** | 511 | 55.8% | **+Rs 224,934** | **+44.5%** |

Day-wise: the optimized-15 combination was net-positive on **19 of 20 trading days**, the one exception (4 Sep, -Rs 563) essentially flat - a materially smoother equity curve than either the baseline or the raw candidate list, not just a higher total. Full day-wise/trade-wise CSVs for all three (baseline, raw-20-candidates, optimized-15) sent to the user.

**What this optimization actually is, stated plainly**: this is IN-SAMPLE selection - the top 15 were chosen because they were the best performers over this exact same 30-day window being reported on, so the "+44.5%" figure is close to a best-case, not an out-of-sample validation. The honest, still-useful signal here is narrower than "+Rs 224,934 is what you'd get": it's that **9 real, liquid FNO names outside the current watchlist (PGEL, TCS, ADANIENSOL, BDL, 360ONE, TIINDIA, KEI, GODREJPROP, COFORGE) backtested profitably against this exact deployed strategy over the same window the current watchlist was tested on**, and 9 of the current watchlist's own members (SONACOMS, DIVISLAB, AUROPHARMA, MOTHERSON, APOLLOHOSP, OBEROIRLTY, RBLBANK, RADICO, MOTILALOFS) underperformed the alternatives available. A genuine "is this actually better" answer needs the same combination re-tested on a DIFFERENT, later window before any live watchlist change - same standard caveat as every backtest in this repo, doubly important here since the selection itself was fit to this window.

**Other standard caveats** (same as [[bollinger-vs-swing-v3-exact-slippage-30day-comparison]]): ATM-strike liquidity filtering, the funds check, the duplicate-pending-order guard, and live `MAX_CONCURRENT_TRADES=5` capacity/freshness-priority backlog are NOT modeled - every symbol here was backtested as if it could trade with unlimited capacity in isolation, which the real 5-slot-wide live book cannot do. A 15-symbol watchlist backtested this way will systematically overstate what the real live book would have captured, for baseline AND candidates equally (same bias both sides, so the RELATIVE comparison holds better than either total in isolation).

**Outcome:** informational only. Not deployed, `data/bollinger_watchlist` not touched. If the user wants to act on this, the natural next step is a fresh out-of-sample window (not the one used to pick these 15) before touching the live watchlist file.
