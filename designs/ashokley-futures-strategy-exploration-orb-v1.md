# ASHOKLEY futures strategy exploration - ORB "v1" is the best result so far (24 Sep 2026, user request)

**Status: exploratory backtesting only - nothing here is deployed live.**
ASHOKLEY is not on any package's live watchlist; every script below is a
standalone backtest in `traderBoy` (repo root, not inside a package dir),
paper/simulation only, no real orders placed at any point in this session.

## Why this file exists

The user asked for a series of paper-trading strategies on ASHOKLEY's
FUTSTK contract, iterated through ~10 variants across a single session
(24 Sep 2026), then explicitly asked to "note it down" once one variant
(Opening Range Breakout) clearly outperformed everything else, and named
it **v1**. This file is that record - what was tried, in what order, why
each iteration happened, and the full comparison table, so a future
session doesn't have to re-run 10 backtests to rediscover that ORB beat
every oscillator/crossover variant by a wide margin.

## Full strategy comparison (all on ASHOKLEY FUTSTK, lot_size=5000, 1 lot)

All continuous multi-day candle fetches, per this repo's standing rule
(see `learnings/backtest-methodology.md`). Windows vary per user request
at the time - noted per row.

| # | Strategy | Window | Trades | Win Rate | Net P&L | P&L/Trade | Script |
|---|---|---|---:|---:|---:|---:|---|
| 1 | MACD(12,26,9) + 5min Supertrend | 3d | 26 | 32.0% | +8,700 | +335 | `backtest_ashokley_futures_macd_supertrend_3day.py` |
| 2 | 1min ST + 5/15min ST filter (asymmetric) | 3d | 27 | 26.9% | -10,500 | -389 | `backtest_ashokley_futures_multiframe_supertrend_3day.py` |
| 3 | Same, symmetric (5min filter on both sides) | 3d | 26 | 26.9% | -10,500 | -404 | same file, updated |
| 4 | RSI(14) level-50 cross, no time exit | 5d | 11 | 60.0% | -18,200 | -1,655 | `backtest_ashokley_futures_rsi50_5day.py` (renamed) |
| 5 | Same + 15min max-hold | 3d | 24 | 56.5% | +5,950 | +248 | `backtest_ashokley_futures_rsi50_maxhold15_3day.py` |
| 5b | Hold-time sweep: 15/30/45/60 min | 3d | 24/18/15/15 | 56.5/52.9/42.9/42.9% | **+5,950/+16,150/-3,300/-14,350** | - | same file, `MAX_HOLD_MINUTES` CLI arg |
| 5c | Winner (30min hold) re-tested | 10d | 50 | 57.1% | +17,450 | +349 | same file |
| 6 | Triple EMA 50/100/200 cross | 10d | 15 | 20.0% | -16,000 | -1,067 | `backtest_ashokley_futures_triple_ema_10day.py` |
| 7 | Triple EMA 20/50/100 | 10d | 30 | 30.0% | +4,500 | +150 | same file, params changed |
| 8 | Triple EMA 12/20/50 | 10d | 75 | 27.0% | -51,450 | -686 | same file, params changed |
| 9 | EMA 20/50/100 (1min) + 5min EMA200 state filter | 10d | 17 | 23.5% | +9,950 | +585 | `backtest_ashokley_futures_triple_ema_5min_ema200_filter.py` |
| 10 | Same, everything on 5min (no 1min trigger) | 10d | 2 | 0% | -9,450 | -9,450 | `backtest_ashokley_futures_triple_ema_5min_all.py` |
| 11 | 5min EMA cross + 15min EMA200 filter | 10d | 2 | 0% | -9,450 | -9,450 | `backtest_ashokley_futures_5min_ema_15min_ema200_filter.py` |
| 12 | VWAP + RSI(9) midline(50) cross scalp | 10d | 284 | 21.9% | -15,200 | -54 | `backtest_ashokley_futures_vwap_rsi_scalper.py` (v1 of this file) |
| 13 | VWAP + RSI(9) oversold/overbought(30/70) bounce | 10d | 54 | 55.6% | +9,950 | +184 | same file, entry trigger changed |
| 14 | Same + cooldown (2 consecutive stops -> 30min pause) | 10d | 52 | 55.8% | **+10,900** | **+210** | same file, tuned default |
| **15** | **Opening Range Breakout (15min range, 2:1 RR, 1 trade/day)** | **10d** | **8** | **87.5%** | **+44,800** | **+5,600** | **`backtest_ashokley_futures_orb.py` - named "v1"** |

## v1: Opening Range Breakout - the winner

**Why it won by such a wide margin:** every oscillator/crossover strategy
(#1-14) takes MANY signals per day because the trigger (a level cross)
fires repeatedly through a choppy session - profit per trade stayed in the
Rs150-600 range even for the "good" ones, swamped by transaction-like
noise from frequent small losses. ORB is structurally different: it
defines ONE reference range per day (first 15 minutes) and takes AT MOST
ONE trade per day, in whichever direction breaks out first. This isn't a
parameter tweak on the same idea - it's a different trade-selection
mechanism, and it's what actually worked.

**Rules:**
- Opening range = high/low of the first 15 minutes (09:15-09:30) of 1-min
  bars, computed fresh per trading day (not a continuous multi-day
  indicator - deliberately, since an "opening range" spanning days isn't
  a coherent concept).
- First 1-min close to break the range wins the day: close > range_high
  -> LONG, close < range_low -> SHORT. No second entry that day even if
  the opposite breakout later occurs (avoids ORB's classic
  buy-breakout-stop-then-short-breakdown whipsaw).
- Stop = the OPPOSITE side of the range. Target = entry +/- 2.0x the risk
  (RR_MULTIPLE, CLI-configurable) implied by that day's own range size -
  not a flat percent like the scalper, since range size varies day to day.
- If neither target nor stop hits, square off at the day's last available
  1-min bar (no overnight carry).

**10-day result (2026-09-09 to 2026-09-23): 8 trades, 7 wins, 1 loss
(87.5%), +Rs44,800 total, +Rs5,600/trade average.**

| Date | Side | Range | Entry | Stop | Target | Exit (all EOD) | PnL |
|---|---|---|---:|---:|---:|---:|---:|
| 09/09 | SHORT | [168.01, 170.00] | 167.93 | 170.00 | 163.79 | 167.07 | +4,300 |
| 09/10 | - | [165.22, 167.35] | - no breakout, no trade - | | | | 0 |
| 09/11 | LONG | [160.21, 163.97] | 163.98 | 160.21 | 171.52 | 165.18 | +6,000 |
| 09/15 | SHORT | [163.06, 165.80] | 162.70 | 165.80 | 156.50 | 157.45 | **+26,250** |
| 09/16 | SHORT | [156.67, 158.65] | 156.50 | 158.65 | 152.20 | 157.20 | -3,500 |
| 09/17 | LONG | [156.96, 158.93] | 158.94 | 156.96 | 162.90 | 159.60 | +3,300 |
| 09/18 | LONG | [159.10, 160.90] | 161.26 | 159.10 | 165.58 | 162.11 | +4,250 |
| 09/21 | SHORT | [162.00, 163.22] | 161.80 | 163.22 | 158.96 | 161.74 | +300 |
| 09/22 | SHORT | [162.60, 164.25] | 162.50 | 164.25 | 159.00 | 161.72 | +3,900 |
| 09/23 | - | [161.64, 163.60] | - no breakout, no trade - | | | | 0 |

**Important caveat, flagged to the user, still open:** every exit is
`EOD_SQUARE_OFF` - not one trade actually hit its 2:1 target or its stop
intraday over these 10 days. That means this result is really "trade the
breakout direction and hold to close" (a trend-day-capture strategy), not
a strategy whose defined risk exits are doing anything - the -3,500 loss
on 09/16 was wherever price happened to sit at 15:39, not a stop working
as intended. The +26,250 09/15 trade alone is >58% of total profit - one
trending day, same single-outlier-trade pattern seen in several earlier
strategies (see #12's -19,350/-7,500 overnight trades, #6-8's big single
winners). 8 trades is also a small sample - 87.5% win rate on 8 trades is
encouraging, not strong statistical evidence.

## Open questions for the next session (not yet resolved)

- Does the 2:1 target ever get hit over a longer window, or is
  EOD-square-off structurally the dominant exit for this stock's typical
  daily range vs. its opening-15min range?
- Is the +26,250 day repeatable, or does removing it flip this to a much
  more modest (or negative) result, same as the overnight-trade pattern
  in earlier strategies?
- Untested: tighter RR_MULTIPLE (so trades can book profit intraday
  instead of riding to EOD), a different/shorter opening-range window,
  and whether ORB's edge is ASHOKLEY-specific or general across the F&O
  universe.

## Script inventory (all in `traderBoy` repo root, all standalone/no live impact)

`backtest_ashokley_futures_macd_supertrend_3day.py`,
`backtest_ashokley_futures_multiframe_supertrend_3day.py`,
`backtest_ashokley_futures_rsi50_maxhold15_3day.py` (CLI:
`max_hold_minutes test_days_back`),
`backtest_ashokley_futures_triple_ema_10day.py` (CLI: `test_days_back`,
EMA periods are constants edited per run),
`backtest_ashokley_futures_triple_ema_5min_ema200_filter.py`,
`backtest_ashokley_futures_triple_ema_5min_all.py`,
`backtest_ashokley_futures_5min_ema_15min_ema200_filter.py`,
`backtest_ashokley_futures_vwap_rsi_scalper.py` (CLI: `test_days_back
target_pct stop_pct max_hold_minutes consecutive_stops_trigger
cooldown_minutes`), `backtest_ashokley_futures_orb.py` (CLI:
`test_days_back opening_range_minutes rr_multiple`) - **the v1 script**.
