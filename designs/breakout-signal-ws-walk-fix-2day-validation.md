# WS-walk breakout-signal fix - 2-day validation (23 Sep 2026)

## Purpose

Before deploying the WS-walk fix (see [[2026-09-23-breakout-signal-single-snapshot-missed-signals-and-ws-walk-fix]]),
the user asked for a replay showing what it would actually have done, and - if it looked like a
net loss - a backtest across both today (23 Sep) and yesterday (22 Sep) before deciding whether
to take it live.

## Methodology

Standalone, read-only replay using a user-supplied out-of-band Dhan access token (never
persisted) via direct REST calls - not the droplet's own live session, to avoid adding rate-limit
pressure to the live bot during market hours. For each day:

1. Reconstructed the REAL, FULL alert universe from the droplet's own real logs (not assumed):
   - 23 Sep: union of `history/2026-09-23_breakout_signal_{universedispatcher,luxury,futures,
     options}_{CE,PE}.json` - 89 CE + 61 PE = **150 distinct real-alerted symbols**.
   - 22 Sep: reconstructed from `history/2026-09-22_webhook_alerts.log` using the same empirically-
     built (strategy, scan_name)->option_type table `backtest_bucket_switch.py` already uses
     (Krishvi/Kaashvi/Range Breakout->CE, Sell Range Breakout->PE) - 68 CE + 23 PE = **91 distinct
     real-alerted symbols**. (22 Sep predates the UniverseDispatcher/universe_bucket rollout -
     confirmed no Simply Bull/Bear alerts that day - so Options/Luxury/Futures each ran their own
     independent per-package watchlist that day, same single-snapshot bug applying to each.)
2. For every symbol, walked EVERY completed 5-min candle on the target date (continuous multi-day
   series, matching the standing continuous-candles rule) against the live-deployed loosened
   thresholds (clearance=0.15%, body>=0.5%, relvol>=0.8x, avg_daily_vol>=300k, consolidation
   range<=12%, price-level<=10% from 20d/50d extreme, trend vs 20d/50d SMA), returning the FIRST
   candle that confirms - this is what a near-real-time WS-walk evaluation converges to for a
   symbol alerted at/near market open (see the fix's own doc for why the production stale-candle
   reversal guard is not separately modeled here: it only matters for a late-joining symbol, and
   nearly everything in both universes was alerted within the first ~10-20 minutes of the day).
3. For each confirmed signal, resolved the real ATM option contract and simulated entry (next
   real 1-min candle open at/after the signal) against the **actually-deployed live ladder for
   ALL THREE packages** (corrected from an earlier, less careful pass that used code-default
   10%/3%/25%/16% figures): `.env` shows `TARGET_PCT=0.20`, `STOP_LOSS_PCT=0.16` uniformly for
   Options/Luxury/Futures - EOD square-off 15:15, target/stop-LEVEL fills (gap-open aware), same
   convention `backtest_breakout_screener_vs_real.py` already established for this universe's
   thin, sub-Rs20 option premiums.

## Results

### 23 Sep 2026 (today)

| | Trades | Wins | Net PnL |
|---|---|---|---|
| **REAL** (Options+Luxury+Futures, breakout-signal path, actual fills today) | 5 | 2 | **+Rs264.00** |
| **SIMULATED** (fix, full 150-symbol real universe, 1 lot/signal, unconstrained) | 12 | 11 | **+Rs15,836.95** |

Real trades today: BANDHANBNK -Rs2,376 (late entry, see incident doc), MOTILALOFS +Rs4,068.75,
MAHABANK +Rs3,185, MCX -Rs2,295, IDFCFIRSTB -Rs2,318.75 (late entry, second real incident -
see the fix doc's own addendum). **Both real losses (BANDHANBNK, IDFCFIRSTB) were confirmed by
their own live production logs to be exactly the "missed the true early candle, entered late and
extended" failure mode this fix closes** - the simulation independently confirms both would have
been TARGET_HIT winners (+Rs1,044 and +Rs1,521.10 respectively) if caught at their true 09:15/
10:25 confirmation instead of 09:22/11:00.

Full per-symbol detail: `replay_2026-09-23.json` (12 signals, all TARGET_HIT except PERSISTENT PE
which hit STOP_LOSS_HIT at -Rs1,020).

### 22 Sep 2026 (yesterday)

| | Trades | Wins | Net PnL |
|---|---|---|---|
| **REAL, breakout-signal-sourced only** (matched via `breakout_signals.log`'s own `entered` status to `real_trades.log`) | 7 | 2 | **-Rs4,552.25** |
| **REAL, all Options+Luxury+Futures trades that day** (any source - broader baseline, not all attributable to this fix) | 16 | 5 | **-Rs8,256.00** |
| **SIMULATED** (fix, full 91-symbol real universe, 1 lot/signal, unconstrained) | 29 | 29 | **+Rs46,167.50** |

Full per-symbol detail: `replay_2026-09-22.json`.

## Important caveats before trusting these numbers

1. **29/29 (100%) simulated win rate yesterday is implausibly clean and should NOT be read as a
   reliable forecast.** It is directionally consistent with a genuinely strong-momentum-
   continuation morning session (most `best_case_pct` figures show the underlying continuing
   30-300% further in the option's favor past the point needed for a 20% target), and manual
   inspection of one trade's raw 1-min option data (BANDHANBNK) shows real, noisy, non-synthetic
   prices - not a data bug. But no real strategy has a true 100% hit rate, and this warrants
   treating the +Rs46,167.50 figure as a soft **upper bound**, not an expectation.
2. **Unconstrained capacity**: both totals assume every confirmed signal gets its own 1-lot
   entry. Real concurrent-position caps (Options CE=1/PE=3, Luxury CE=3/PE=1, Futures CE=3/PE=1,
   per the live `.env` - 12 total concurrent slots across all three) would have blocked some of
   the 29 yesterday / 12 today from all being open AT THE SAME INSTANT - though since most
   simulated trades resolve (target/stop) within single-digit minutes, a fast capacity-aware
   system could still cycle through most signals sequentially over the session. Not modeled here
   in full (would need a full event-driven joint simulator, matching this repo's own
   `backtest_deployed_today_breakout_signal_updated_params.py` methodology) - treat the totals
   above as directionally right, not precisely achievable.
3. **Simplified exit ladder**: flat 20%/16% target/stop only - the REAL production exit stack
   (`MAX_LOSS_HIT -> TARGET_HIT -> PROFIT_PROTECTION_HIT -> TRAILING_SL_HIT/STOP_LOSS_HIT ->
   SUPERTREND_EXIT -> EMA_CROSS_EXIT -> LIQUIDITY_GUARD_ZERO_VOLUME`) is materially more complex
   (dynamic/stepped SL, profit protection, trend-based exits) and can both outperform (real
   MOTILALOFS today rode to +31% vs. simulation's flat +20% target) and underperform (thin-book
   slippage on illiquid sub-Rs10 premiums) this simplified model - same disclosed limitation
   every prior backtest in this repo already carries.
4. **Yesterday's real -Rs8,256 baseline mixes trades NOT sourced from breakout_signal.py at all**
   (e.g. HDFCBANK, DRREDDY, PGEL - some via the separate `alert_bucket.py` loss-triggered switch
   feature, which this fix does not touch) - the fairer comparison for isolating THIS fix's own
   effect is the -Rs4,552.25 / 7-trade breakout-signal-only baseline.

## Bottom line

Both days show the fix would have been a **large net improvement** over what actually happened -
today by directly converting 2 real losses (-Rs4,694.75 combined) into simulated wins
(+Rs2,565.10 combined) on the exact same underlying signals, and yesterday by finding 29 real,
confirmed, currently-undetected momentum continuations the old single-snapshot scan structurally
could not have caught in time. Even heavily discounting the yesterday figure for the 100%-win-
rate implausibility and capacity constraints, there is no scenario in this data where the fix
looks like a net negative - the only real losses in the SIMULATED data are 1 trade today
(PERSISTENT PE, -Rs1,020) out of 41 combined signals across both days.

**Recommendation**: the data supports deploying, but given caveats 1-3 above (soft upper bound,
unconstrained capacity, simplified ladder), consider running the fix live in shadow/paper mode
for one more session (or watching the first live day closely) rather than trusting the exact Rs
figures - the qualitative case (catch the early candle, don't chase an extended one) is strong
and directly evidenced by two real losses today; the precise magnitude is less certain.
