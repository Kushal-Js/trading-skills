# ASHOKLEY futures strategy exploration - ORB "v1" is the best result so far (24 Sep 2026, user request)

**UPDATE:** a later strategy, 1-hour-RSI-timed-on-5min-candles, was named
**v2** and on the same 30-day window has a BETTER avg-PnL/trade than v1
(+2,386 vs +357) with less tail risk - see
[[ashokley-futures-1hour-rsi-5min-timing-v2]] for the full writeup and
hold-time sweep (30/45/60 min).

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

## UPDATE 24 Sep 2026: 30-day re-test confirms the sample-size caveat was
## warranted - the picture is materially different at scale

User asked to extend the window per the "test longer for a larger sample"
option raised above. Re-ran the identical v1 script/params
(`opening_range_minutes=15 rr_multiple=2.0`) over the last 30 trading
days (2026-08-12 to 2026-09-23, i.e. the original 10-day window PLUS 20
earlier days never backtested before):

**27 trades, 13 wins/14 losses (48.1% - down from 87.5%), net +Rs9,650
(down from +Rs44,800), avg +Rs357/trade (down from +Rs5,600/trade).**

The earlier 20 days (08/12-09/08, not in the original 10-day sample)
included a brutal stretch of **6 consecutive-ish STOP_HIT losses between
08/13 and 08/24**: -18,200 / -10,500 / -18,600 / -6,700 / -6,450 / -7,500
(sum -67,950, avg -11,325/loss) - this is exactly the tail risk the
original 10-day sample never surfaced, because in that window every
single trade happened to exit at EOD instead. One TARGET_HIT trade
(+14,700, 09/04) also only shows up in the extended window.

**Exit-reason breakdown (27 trades, full 30-day window):**

| Exit reason | Trades | Total PnL | Avg PnL |
|---|---:|---:|---:|
| EOD_SQUARE_OFF | 20 | +62,900 | +3,145 |
| STOP_HIT | 6 | -67,950 | -11,325 |
| TARGET_HIT | 1 | +14,700 | +14,700 |

**Read: EOD_SQUARE_OFF and TARGET_HIT trades are strongly profitable on
average; STOP_HIT trades are what's dragging the whole thing down to
barely-positive.** The stop is placed at the FULL OPPOSITE side of the
opening range, which on a wide-range day is a large price distance - when
the breakout fails, the loss is correspondingly large (up to -18,600 on a
single trade, more than triple the biggest EOD winner in the original
10-day sample). This is the clear next lever: the entry/direction
selection looks genuinely decent (48% win rate with winners averaging far
more than losers on the EOD side), but the RISK MANAGEMENT (stop
placement) is what needs tightening - not the signal itself.

**Revised verdict: v1 is still net profitable at scale (+Rs9,650 / 27
trades over 30 days, positive expectancy), but the original 10-day number
was a favorable sample that avoided this strategy's real failure mode
(wide-range days that fail and hit a wide stop). Do not treat the
+Rs5,600/trade figure as representative going forward - use +Rs357/trade
(30-day) as the more honest current estimate until a tighter/ATR-based
stop is tested.**

## Open questions for the next session (not yet resolved)

- **[RESOLVED by the 30-day re-test above]** Does the 2:1 target ever get
  hit over a longer window, or is EOD-square-off structurally the
  dominant exit? - EOD_SQUARE_OFF still dominates (20/27), but STOP_HIT
  (6/27) and TARGET_HIT (1/27) both occur at scale; the target rarely
  hits but the wide stop hits often enough to matter a lot.
- **[RESOLVED, sample-size warning confirmed]** Is the +26,250 day
  repeatable / representative? - No: the 10-day sample was a favorable
  stretch; the 20 earlier days added six large STOP_HIT losses that
  dragged the 30-day average down to +357/trade.
- **STILL OPEN:** a tighter/ATR-based stop (instead of full opposite-
  range-side) to cut the -11,325 average STOP_HIT loss size without
  losing the apparently-real edge on the EOD/target side - the most
  promising next lever per the exit-reason breakdown above.
- **STILL OPEN:** a different/shorter opening-range window, and whether
  ORB's edge is ASHOKLEY-specific or general across the F&O universe.

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
