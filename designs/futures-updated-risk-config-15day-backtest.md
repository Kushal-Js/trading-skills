# Futures risk-config raise, deployed live + 15-day backtest (21 Sep 2026, user request)

Same risk-parameter raise already applied to Luxury the same day (see
[[luxury-signal-gated-live-simulation]]'s "Config raise, deployed live"
section), applied to Futures too per explicit user request:

| Setting | Old | New |
|---|---|---|
| MAX_LOSS_PER_TRADE_RS (CE) before/after 11:30 | Rs4,500 / Rs2,100 | **Rs5,500 / Rs3,100** |
| MAX_LOSS_PER_TRADE_RS (PE) before/after 11:30 | Rs3,500 / Rs1,600 | **Rs4,500 / Rs2,600** |
| PROFIT_PROTECTION_THRESHOLD_RS before/after 11:30 | Rs2,000 / Rs1,500 | **Rs5,000 / Rs2,500** |
| PROFIT_PROTECTION_GIVEBACK_PCT | 3% | **8%** |

(Futures' PE caps have no live effect today - `futures_main.py` only
exposes a CE webhook - kept for parity per `Futures/config.py`'s own
existing note, same as before this raise.)

Deployed as `.env` overrides on BOTH the local repo and the droplet
(gitignored, scp'd directly, never a git commit), then `dhanboy.service`
restarted. The first restart attempt was blocked by the Claude Code
production-deploy classifier; re-confirmed explicitly by the user before
retrying. Positions checked immediately before and after (Options/
Futures/Luxury empty both times; Swing's ANGELONE reconciled cleanly
with `exit_failure_count` reset to 0 both times, per the standing
restart-safety checklist).

## Backtest methodology

New script: `backtest_futures_updated_risk_config_15day.py`, adapted
from `backtest_futures_chartink_range_breakout_06.py` (17 Sep 2026) -
same modeled entry/exit mechanics, but pulls **real Futures Chartink
webhook alerts** (`history/<day>_webhook_alerts.log`, strategy=="Futures")
across the same 14-trading-day window used elsewhere this session
(31 Aug - 18 Sep 2026) instead of a static CSV, matching how
`backtest_luxury_signal_gated_live.py` sources Luxury's real alerts.

Modeled, all read live from `Futures/config.py` at import time (so this
run picked up the NEW values automatically once `.env` was updated):
multi-window entry gate (09:15-11:00, 14:00-15:28), batch ranking
(TOP_N_STOCKS=4, SELECT_BOTTOM_N_STOCKS, one simulated batch per real
webhook call), MAX_LIVE_POSITIONS_CE capacity (3), daily re-entry cap,
RSI-gated loss re-entry block, LOSS_REPEAT_BLOCK, volume-floor entry
gate, and the full CE exit ladder in production's real priority order:
MAX_LOSS_HIT -> TARGET_HIT -> PROFIT_PROTECTION_HIT (with giveback) ->
TRAILING_SL_HIT/STOP_LOSS_HIT (dynamic SL) -> SUPERTREND_EXIT ->
EMA_CROSS_EXIT. Underlying and option-premium candles fetched as one
continuous series per symbol/contract (30 days back from today, never
re-fetched per day), per the standing continuous-candles rule.

**Not modeled** (disclosed, same gaps as the range-breakout script this
is adapted from): LIQUIDITY_GUARD_ZERO_VOLUME, SL-L broker-side stop
(can only help), ENABLE_GAP_DOWN_CE_DELAY, cross_strategy_registry, and
EOD/Friday square-off.

## Result: new config vs. the real (old-config) Futures baseline

| | Trades | Wins | Win Rate | Total PnL | Profit Factor |
|---|---:|---:|---:|---:|---:|
| REAL Futures (old config, unaffected historical baseline) | 50 | 19 | 38.0% | -Rs11,943.55 | 0.68 |
| **Simulated, new config, real alerts** | 45 closed (+2 still open) | 20 | 44.4% | **-Rs714.75** | **0.98** |
| Delta vs. real | | | | **+Rs11,228.80** | |

Both are still net negative, but the new config's simulation comes in
almost exactly breakeven, a large improvement over the real -Rs11,943.55.
**Caveat up front: this is not a clean isolate of "config change only."**
The real baseline is REAL trades (real entry timing/fills, real
liquidity guard, real gap-down delay, real cross-strategy lock); the new-
config number is a SIMULATION missing several of those real gates (see
Not Modeled above) AND uses the new risk caps - so part of this delta is
the config change, and part of it is the simulation's own gaps (a real
run at the new config would very likely underperform this simulated
number for the same reasons the naive/real-exit-stack comparison showed
for Luxury). Treat this as the same order of caveat as Luxury's own
config-raise backtest.

### Exit-reason breakdown (new config, 45 closed trades)

| Exit reason | Trades | PnL |
|---|---:|---:|
| TARGET_HIT | 10 | +30,026.25 |
| SUPERTREND_EXIT | 20 | -7,547.25 |
| TRAILING_SL_HIT | 3 | -7,006.25 |
| EMA_CROSS_EXIT | 9 | -6,910.00 |
| STOP_LOSS_HIT | 2 | -5,181.25 |
| MAX_LOSS_HIT | 1 | -4,096.25 |

Zero `PROFIT_PROTECTION_HIT` exits in this sample, same pattern as
Luxury's own config-raise result - the higher threshold (Rs5,000/2,500)
and wider giveback (8%) mean fewer positions ever get cut short by this
path; TARGET_HIT alone contributes more gross profit (+30,026.25) than
every other exit reason's losses combined lose.

### Day-wise (new config, 47 trades incl. 2 still open)

| Day | Trades | W/L | New-Config PnL | Real Futures PnL (old config) |
|---|---:|---|---:|---:|
| 1 Sep | 1 | 0W/1L | -2,380.00 | +2,880.00 |
| 11 Sep | 4 | 2W/2L | -2,684.75 | -3,622.50 |
| 15 Sep | 3 | 1W/2L | +1,858.75 | +5,227.50 |
| 16 Sep | 10 | 5W/5L | -2,614.00 | -8,603.30 |
| 17 Sep | 19 | 8W/11L | +6,501.50 | -5,741.75 |
| 18 Sep | 10 | 4W/4L, 2 open | -1,396.25 | -2,139.00 |
| **Total** | | | **-714.75** | **-11,943.55** |

(31 Aug, 3/4/8/9/10 Sep had zero entries in both the real trades and
this simulation - the simulation's ranking/volume-floor gates blocked
the same days Futures happened to sit out for real, not a bug; verified
against the raw run log.)

### Trade-wise (new config, 45 closed trades)

| Day | Symbol | Entry Time | Entry | Exit | Exit Reason | Qty | PnL |
|---|---|---|---:|---:|---|---:|---:|
| 01 Sep | HCLTECH | 09:25 | 39.45 | 33.50 | TRAILING_SL_HIT | 400 | -2,380.00 |
| 11 Sep | PAYTM | 09:20 | 51.95 | 52.20 | SUPERTREND_EXIT | 725 | +181.25 |
| 11 Sep | MCX | 09:20 | 110.95 | 115.70 | SUPERTREND_EXIT | 225 | +1,068.75 |
| 11 Sep | OIL | 09:20 | 15.90 | 13.60 | SUPERTREND_EXIT | 1,400 | -3,220.00 |
| 11 Sep | IDEA | 13:35 | 0.69 | 0.68 | SUPERTREND_EXIT | 71,475 | -714.75 |
| 15 Sep | OFSS | 09:20 | 260.60 | 325.45 | TARGET_HIT | 100 | +6,485.00 |
| 15 Sep | LTM | 09:20 | 114.10 | 94.55 | TRAILING_SL_HIT | 150 | -2,932.50 |
| 15 Sep | TATAELXSI | 09:20 | 93.00 | 79.45 | TRAILING_SL_HIT | 125 | -1,693.75 |
| 16 Sep | PAYTM | 09:20 | 60.10 | 62.55 | SUPERTREND_EXIT | 725 | +1,776.25 |
| 16 Sep | ADANIPORTS | 09:20 | 35.50 | 34.40 | SUPERTREND_EXIT | 475 | -522.50 |
| 16 Sep | YESBANK | 09:20 | 0.54 | 0.65 | TARGET_HIT | 31,100 | +3,421.00 |
| 16 Sep | PAYTM | 09:25 | 50.45 | 45.50 | SUPERTREND_EXIT | 725 | -3,588.75 |
| 16 Sep | PAYTM | 12:40 | 45.00 | 39.35 | **MAX_LOSS_HIT** | 725 | **-4,096.25** |
| 16 Sep | YESBANK | 13:10 | 0.73 | 0.73 | SUPERTREND_EXIT | 31,100 | 0.00 |
| 16 Sep | ADANIPORTS | 13:35 | 33.80 | 34.20 | EMA_CROSS_EXIT | 475 | +190.00 |
| 16 Sep | YESBANK | 14:00 | 0.75 | 0.75 | SUPERTREND_EXIT | 31,100 | 0.00 |
| 16 Sep | COLPAL | 14:50 | 29.35 | 29.75 | SUPERTREND_EXIT | 275 | +110.00 |
| 16 Sep | COLPAL | 15:00 | 29.20 | 29.55 | SUPERTREND_EXIT | 275 | +96.25 |
| 17 Sep | BANKINDIA | 09:20 | 2.95 | 3.62 | TARGET_HIT | 5,200 | +3,484.00 |
| 17 Sep | SRF | 09:20 | 54.55 | 43.80 | STOP_LOSS_HIT | 200 | -2,150.00 |
| 17 Sep | BAJFINANCE | 09:20 | 19.25 | 18.70 | SUPERTREND_EXIT | 750 | -412.50 |
| 17 Sep | MARICO | 09:50 | 12.05 | 10.50 | EMA_CROSS_EXIT | 1,200 | -1,860.00 |
| 17 Sep | BSE | 11:25 | 84.35 | 82.10 | EMA_CROSS_EXIT | 200 | -450.00 |
| 17 Sep | TATACONSUM | 11:30 | 15.05 | 19.35 | TARGET_HIT | 550 | +2,365.00 |
| 17 Sep | SHRIRAMFIN | 11:40 | 18.60 | 18.65 | SUPERTREND_EXIT | 825 | +41.25 |
| 17 Sep | SHRIRAMFIN | 11:50 | 19.30 | 19.50 | SUPERTREND_EXIT | 825 | +165.00 |
| 17 Sep | GODREJPROP | 11:55 | 38.80 | 36.30 | EMA_CROSS_EXIT | 325 | -812.50 |
| 17 Sep | TIINDIA | 12:50 | 59.40 | 56.10 | EMA_CROSS_EXIT | 200 | -660.00 |
| 17 Sep | NAUKRI | 14:00 | 19.60 | 19.90 | SUPERTREND_EXIT | 550 | +165.00 |
| 17 Sep | NAUKRI | 14:10 | 20.50 | 20.50 | SUPERTREND_EXIT | 550 | 0.00 |
| 17 Sep | TATASTEEL | 14:15 | 3.68 | 4.80 | TARGET_HIT | 2,750 | +3,080.00 |
| 17 Sep | HEROMOTOCO | 14:15 | 90.05 | 108.90 | TARGET_HIT | 150 | +2,827.50 |
| 17 Sep | TMPV | 14:40 | 4.90 | 6.20 | TARGET_HIT | 1,600 | +2,080.00 |
| 17 Sep | JINDALSTEL | 14:45 | 16.90 | 16.65 | SUPERTREND_EXIT | 625 | -156.25 |
| 17 Sep | IEX | 14:50 | 2.96 | 2.91 | EMA_CROSS_EXIT | 4,350 | -217.50 |
| 17 Sep | JINDALSTEL | 14:55 | 27.35 | 26.65 | SUPERTREND_EXIT | 625 | -437.50 |
| 17 Sep | TATACONSUM | 15:00 | 16.65 | 15.65 | EMA_CROSS_EXIT | 550 | -550.00 |
| 18 Sep | ETERNAL | 09:20 | 5.70 | 4.45 | STOP_LOSS_HIT | 2,425 | -3,031.25 |
| 18 Sep | JUBLFOOD | 09:30 | 7.00 | 9.00 | TARGET_HIT | 1,250 | +2,500.00 |
| 18 Sep | KFINTECH | 09:30 | 10.65 | 13.00 | TARGET_HIT | 575 | +1,351.25 |
| 18 Sep | ABB | 12:35 | 141.00 | 123.40 | SUPERTREND_EXIT | 125 | -2,200.00 |
| 18 Sep | POLYCAB | 12:40 | 150.05 | 143.40 | EMA_CROSS_EXIT | 125 | -831.25 |
| 18 Sep | LODHA | 12:50 | 26.90 | 24.15 | EMA_CROSS_EXIT | 625 | -1,718.75 |
| 18 Sep | PHOENIXLTD | 13:25 | 31.75 | 38.70 | TARGET_HIT | 350 | +2,432.50 |
| 18 Sep | SHREECEM | 15:00 | 382.00 | 386.05 | SUPERTREND_EXIT | 25 | +101.25 |

Plus 2 still-open at end of available data: POWERINDIA (18 Sep 13:55)
and ABB (18 Sep 14:30) - Saturday/Sunday aren't trading days and Monday
21 Sep's session hadn't started yet when this backtest ran, so there's
simply no further price data past Friday's close to resolve them; not a
bug.

## Caveats specific to this change

- **Real, live risk-parameter change affecting every Futures position**,
  not a hypothetical - a normal Futures trade can now lose up to
  Rs5,500/3,100 before MAX_LOSS_HIT instead of Rs4,500/2,100, and rides a
  much wider profit-protection band before locking in gains. Deliberate
  larger risk-per-trade in exchange for letting winners run.
- **The delta vs. real is NOT a clean before/after of the config change
  alone** - see the caveat under Result above. A genuine apples-to-apples
  comparison would need the same simulation methodology run against the
  OLD config too (not done here, since the user asked for the update +
  15-day backtest of the new config specifically, mirroring the Luxury
  request's exact scope).
- Same single-window, non-generalized-sample caveat as every other
  backtest in this line of work - 14 trading days, one historical stretch.
