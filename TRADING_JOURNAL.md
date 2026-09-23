# Trading Journal

A single, chronological record of every strategy/risk-config change
deployed to the live bot, tied to what actually happened to real PnL
before and after it, plus the issues that came up along the way. Purpose
(user request 21 Sep 2026): give future strategy-tuning decisions a
complete, honest before/after record to learn from, instead of
re-deriving history from scattered `.env` comments and 280+ git commits
every time.

**This is a living document — update it, don't replace it.** See
"How to maintain this" at the bottom before making any live deployment.

## How to read this file

- **Real PnL trajectory**: the actual, empirical day-by-day PnL per
  strategy from the bot's own `history/*_real_trades.log` files — ground
  truth, not backtest.
- **Deployment timeline**: every dated strategy/risk/config change,
  compiled from `.env`'s own dated comments (traderBoy's established
  practice of documenting every change inline) and git commit history.
  Each entry notes the evidence behind it and links to the fuller
  backtest/design doc or incident writeup where one exists.
- **Known issues**: chronological index of `incidents/` — anything that
  went wrong for real, independent of whether it was caused by a
  deliberate strategy change.
- Real trade history only goes back to **31 Aug 2026** (earlier days
  have no `real_trades.log` files) — changes dated before that have no
  directly-attributable before/after PnL here; use the design docs'
  own backtest evidence for those instead.

---

## Real PnL trajectory (all 4 strategies, real trades, 31 Aug – 18 Sep 2026)

Source: `history/<date>_real_trades.log` on the droplet, pulled 21 Sep
2026. 13 trading days have data (no alerts/trades on 2/14/19/20 Sep -
weekends or genuinely zero activity).

| Date | Options T | Options PnL | Futures T | Futures PnL | Luxury T | Luxury PnL | Swing T | Swing PnL | **All PnL** |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 31 Aug | 10 | -9,432.50 | 2 | +55.50 | 0 | 0.00 | 0 | 0.00 | **-9,377.00** |
| 01 Sep | 0 | 0.00 | 2 | +2,880.00 | 7 | -3,240.00 | 2 | -6,930.00 | **-7,290.00** |
| 03 Sep | 0 | 0.00 | 0 | 0.00 | 25 | -10,229.00 | 2 | +37.50 | **-10,191.50** |
| 04 Sep | 0 | 0.00 | 0 | 0.00 | 12 | -361.50 | 1 | +2,218.75 | **+1,857.25** |
| 07 Sep | 0 | 0.00 | 0 | 0.00 | 0 | 0.00 | 2 | -3,120.00 | **-3,120.00** |
| 08 Sep | 0 | 0.00 | 0 | 0.00 | 4 | +1,951.25 | 1 | 0.00 | **+1,951.25** |
| 09 Sep | 0 | 0.00 | 0 | 0.00 | 8 | -3,857.50 | 0 | 0.00 | **-3,857.50** |
| 10 Sep | 28 | +4,809.50 | 0 | 0.00 | 6 | -1,402.50 | 0 | 0.00 | **+3,407.00** |
| 11 Sep | 18 | -11,470.75 | 2 | -3,622.50 | 1 | -1,338.75 | 0 | 0.00 | **-16,432.00** |
| 15 Sep | 28 | +3,957.00 | 2 | +5,227.50 | 0 | 0.00 | 2 | -435.00 | **+8,749.50** |
| 16 Sep | 12 | -10,095.50 | 4 | -8,603.30 | 3 | +6,139.25 | 0 | 0.00 | **-12,559.55** |
| 17 Sep | 8 | -3,918.75 | 23 | -5,741.75 | 26 | +5,130.00 | 8 | -3,247.50 | **-7,778.00** |
| 18 Sep | 5 | -5,823.75 | 15 | -2,139.00 | 33 | -10,700.60 | 2 | +837.50 | **-17,825.85** |
| **Total** | | **-31,974.75** | | **-11,943.55** | | **-17,909.35** | | **-10,638.75** | **-72,466.40** |

**Every strategy is net negative over this window.** No config change
made in this period had turned the real trajectory positive by 18 Sep -
this is the baseline every deployment below should be measured against
once enough real trades accumulate under each new config to judge it
fairly (a handful of days is not enough, per the caveat every backtest
in this repo already carries).

**Correction (21 Sep 2026)**: the Swing column and totals above are
CORRECTED from the original version of this table. The bot's own
`real_trades.log` understated every Swing MCX trade's (COPPER/
CRUDEOIL/NATURALGAS) PnL by orders of magnitude due to a logging bug
(see "PnL logging bug" under 21 Sep in the Deployment timeline, and the
new Known Issues row) - e.g. 17 Sep's real MAX_LOSS_HIT on COPPER was
logged as -Rs1.89 but was actually **-Rs4,725.00**. Recomputed directly
from each trade's real entry/exit price and MCX_PNL_MULTIPLIER, not
from the logged `pnl` field. Non-MCX trades (Options/Futures/Luxury,
and Swing's own non-MCX symbols) were never affected - only Swing's 3
MCX symbols were.

*(This table should be re-pulled and appended to periodically - see
"How to maintain this" below.)*

---

## Deployment timeline

Compiled from traderBoy's `.env` (which already documents every change
inline with a date and rationale, per that repo's own established
convention) and `git log`. Grouped by date, most recent first. "Backtest
evidence" links to the `designs/` doc where a real backtest exists;
"Real PnL since" is filled in once enough real trades have accumulated
under that specific config to judge it (many recent rows are too new to
judge yet - flagged explicitly).

### 23 Sep 2026

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| **Swing: gated entry-evaluation to real market hours, added a weekly Friday square-off** (`Swing/trading_engine.py`/`signals.py`/`config.py`, commit `cc70363`) - a post-market-close system check found `monitor_loop`'s entry-evaluation path (`get_regime_state`/`get_supertrend_state`, the structure-break refresh loop) ran completely unconditionally every 5s, 24/7, for zero possible benefit (no candle forms and no order can fill outside real trading hours). Confirmed live: 39 `DH-904` rate-limit hits in under an hour, well after midnight. Added `_symbol_market_open` (MCX-vs-NSE aware, since MCX's session runs materially longer - reuses the `is_market_open(exchange_segment=...)` split from the 15 Sep MCX-hours fix below) to gate both entry-evaluation paths; exit-checking on an already-open position is untouched, by design (Swing carries positions across ordinary days). Also added a new weekly Friday square-off (`FRIDAY_SQUARE_OFF_TIME`, default 15:25 IST) - every open Swing position, NSE and MCX alike, force-closed by Friday close with no new entries for the rest of the week, to avoid weekend gap risk. See [[2026-09-22-swing-overnight-polling-and-friday-squareoff-fix]]. **Backfilled into this journal 23 Sep** - originally deployed same night, its own entry was missed at the time; caught by a follow-up "scan for gaps across all 4 packages" audit. | Swing | Live rate-limit-hit evidence (39 DH-904s/hour), not a backtest - infra/scheduling fix plus a new risk-management rule. | N/A - deployed 23 Sep 00:11 IST (well after market close, 0 live positions to disturb). Not yet independently re-verified against a real Friday close (next Friday after this deploy will be the first live test of the square-off path itself). |
| **Added a read-only `/swing/structure-break` debug endpoint for COPPER** (`Swing/signals.py`/`swing_main.py`, commit `5e023f3`) - direct follow-up to confirming Swing's overnight-polling fix works: `/swing/signals` can't see COPPER's structure-break state at all (it bypasses the classic regime/Supertrend path entirely while `COPPER_STRUCTURE_BREAK_ENABLED` is on), so the only prior way to check it was inferring from log silence. Caught in testing before committing: the first draft called the real `structure_break_entry_signal` directly, which has a side effect (clears the consumed-suppression marker on an observed break) - a GET endpoint mutating live entry-gating state on its own polling cadence. Fixed with a separate read-only predictor function; verified via a manual repro that repeated debug calls leave the real state untouched. | Swing | No backtest - pure introspection/tooling, no signal or risk-parameter change. Verified via `tests/test_swing_v2_signals.py` (unchanged, all 6 pass) plus a manual mutation-safety repro. | N/A - committed, **not yet deployed** (droplet restart deferred to later the same day per user instruction). |
| **Luxury CE dynamic-SL trigger raised 0.07 -> 0.20** (`LUXURY_DYNAMIC_SL_STEP_PCT_CE`, `.env`, staged not yet deployed) - real incident: BANDHANBNK CE spiked past the 7% dynamic-SL trigger then reversed to a -15.6% net loss in 70 seconds. Checked against 6 days of real trade history (102 shadow-log-matched trades): ALL 7 `TRAILING_SL_HIT` exits in the window (any package, any option type) were net losses, and all 4 CE-side ones (BLUESTARCO/SWIGGY/PATANJALI/BANDHANBNK) lost -15.6% to -17.3% - essentially matching or beating Luxury's own plain 16% hard stop, meaning the dynamic mechanism never once protected real profit in this sample. Also checked (and rejected) the entry-side hypothesis: BANDHANBNK's own shadow log showed RSI=89 at entry, which looked like an obvious red flag, but segmenting all 102 trades by `rsi_extreme_alone_blocks` showed no meaningful win-rate difference (30% vs 29%) - doesn't generalize, stays shadow-only, same conclusion pattern as [[reversal-trend-strength-filter-arc]] and [[copper-breakout-strength-filter-rejected]]. See [[2026-09-23-bandhanbnk-dynamic-sl-over-sensitivity]] for the full data and reasoning, including the honest caveat that no peak-price data is persisted per trade, so 0.20 is a principled default (tighten only once ahead by more than the stop's own width) rather than a swept-optimal value. Options showed the identical 100%-loss pattern on its own PE side (MAZDOCK/HCLTECH) - not changed, flagged as a follow-up candidate. | Luxury (CE only; Options/Futures unchanged) | Real 6-day trade-history analysis (102 matched trades), not a forward backtest - see the incident doc's full breakdown table. | N/A - staged in `.env`, not yet deployed (same pending restart as the debug endpoint above). |
| **Generalized the 22 Sep monitor-loop-freeze fix to every blocking Dhan order/LTP call, in all four packages** (`Swing/Futures/Luxury/Options/trading_engine.py`, commit `5d49c49`) - a live-code audit (user request: "scan live code for any bugs or bot induced latency or signal lag or race conditions") found the 22 Sep fix (`asyncio.wait_for` around Swing's own `_get_ltp`, see [[2026-09-22-swing-signal-cache-never-throttled-on-failure]]'s related incident) only ever covered that one call site. Every other `run_in_executor(None, dhan_wrapper.X)` call in all four packages - `wait_for_order_result` (entry AND exit, every real order placement), `refresh_order_status`, `check_if_order_filled`, `cancel_order`, plus Futures/Luxury/Options' own `_get_ltp` (which never got the 22 Sep fix at all) - still had no asyncio-level timeout, inheriting dhanhq's ~60s default HTTP timeout compounded by `wait_for_order_result`'s own internal 6-attempt retry loop. Added `_LTP_FETCH_TIMEOUT_SECONDS=10s`, `_ORDER_STATUS_TIMEOUT_SECONDS=10s`, `_ORDER_RESULT_TIMEOUT_SECONDS=30s` to all four files. On a `wait_for_order_result` timeout, synthesizes `OrderResult(status=TRANSIT)` rather than letting the exception propagate raw, so it flows into each package's existing "not yet terminal after poll budget" handling (the already-proven PAGEIND 17 Sep 2026 recovery path) instead of a new, untested failure mode. Also confirmed (not fixed, documented as lower-severity-now): all four packages share ONE `ThreadPoolExecutor` sized by `EXECUTOR_MAX_WORKERS` (`main.py`, live value 5) - a hang in any package's REST call occupies a shared worker thread every other package also depends on; bounded to 10-30s now instead of open-ended. | Options, Futures, Luxury, Swing | No backtest - pure infra/timeout fix, no signal or risk-parameter change. Verified via full test-suite regression (`git stash`/restore comparison, identical 18 pre-existing failures before and after, root-caused to a wall-clock-time-sensitive market-open guard in `_handle_ltp_staleness` unrelated to this change - confirmed via isolated repro, not assumed). | N/A - deployed same session (restart ~08:42 IST, before market open, 0 live positions across all 4 packages before/after, `/health` OK, Dhan auth clean). Nothing to reconcile since nothing was open. Worth a spot-check next time any package's `wait_for_order_result`/`refresh_order_status` path actually hits a slow/hung Dhan response, to confirm the timeout fires as designed rather than assuming from the code alone. |

### 22 Sep 2026

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| **Root-caused and fixed the WS candle open-price mismatch** (`Options/dhan_client.py`, commit `a6e026e`) - ticks were bucketed into 5-min windows by local receipt time, never by `LTT` (Last Trade Time, the exchange's own timestamp - already parsed by the SDK, never read anywhere in this codebase). A trade processed just after a 5-min boundary got misbucketed into the next bar, corrupting that bar's open specifically. Verified LTT's timezone live before trusting it (10 min behind receipt time, not 5.5 hours - confirms no UTC conversion needed). Added `_tick_time_from_ltt` with safe fallback, 5 new tests. See [[2026-09-22-ws-candle-open-price-root-cause-ltt]]. | Cross-cutting (any future `BREAKOUT_USE_WS_CANDLES` use) | Live diagnostic data (LTT vs receipt-time comparison), not a backtest. | N/A - deployed (0 positions before/after, health OK). **Still not live-validated as of 22 Sep 18:20 IST**: a same-day scheduled re-check ran after 15:30 IST market close (past the trading window) and hit two `dhanboy` service restarts ~90 min apart during the check itself, each wiping the in-memory WS feed's subscription state - `recon_bar_count` came back 0-1 vs `real_bar_count: 73` for all 8 test symbols, zero usable samples. Genuinely needs the next live trading session (subscribed near 09:15 IST open, before any restart) to answer the TCS/ICICIBANK exact-match-rate question. See [[ws-candle-reconstruction-parity-results]]'s "22 Sep evening" update. |
| Also: `UNIVERSE_BUCKET_WINDOW_TRADING_DAYS_CE` brought back down to 1 (matching PE), commit `4bdedfe` - user request, same scan-cadence/capacity reasoning as the original 3->1 cut. | Luxury, Futures (dispatcher targets) | User request, operational config change. | N/A - forward-looking. |
| **Fixed WS candle reconstruction's mid-day-subscribe volume overshoot** (`underlying_candle_feed._update_bar`, commit `9a7fe68`) - the volume baseline hardcoded to 0.0 on a symbol's first tick, correct only if subscribed exactly at market open. Confirmed live via `/debug/underlying-feed/parity`: all 8 test symbols showed 25-60x volume overshoots on their first reconstructed bar (subscribed at 10:00 IST, not 09:15). Added `tests/test_underlying_candle_feed.py` covering the exact mid-day-subscribe case `backtest_ws_candle_reconstruction_parity.py`'s REST-replay method structurally can't catch. See [[ws-candle-reconstruction-parity-results]] for the full live numbers, including the still-open open-price gap (up to 75% of bars wrong on some symbols) this fix does NOT address. | Cross-cutting (Swing, and any future `BREAKOUT_USE_WS_CANDLES` use) | Live parity data, not a backtest - real WS ticks vs real REST candles, today. | N/A - `BREAKOUT_USE_WS_CANDLES` stays off everywhere; this was in response to the user asking whether WS candles could be turned on now given the scan-cadence bottleneck - answer was no, not yet, and here's the concrete bug found while checking. |
| **No change** - breakout scanner parameter sweep against today's real alerts found the deployed config isn't missing profitable trades. Replayed the real `_evaluate_signal_sync` (validated byte-for-byte against a real production signal, GVT&D) against 154 real candidate symbols under the deployed params + 8 variants. `MIN_BODY_PCT` is the dominant constraint (loosening it alone: 24->47 signals); `CLEARANCE_PCT`/`MAX_CONSOLIDATION_RANGE_PCT`/`MAX_PCT_FROM_HIGH_LOW` weren't binding at all today. But the 43 additional signals any variant caught showed only marginal real price action since (best case +1.34%, median +0.3%, 46% already reversed) - far short of what the 20% option target needs. See [[breakout-scanner-param-sweep-22sep-intraday]]. | Options, Luxury, Futures | Real-data replay, today's actual alerts/candles, not synthetic. | N/A - deployed config kept unchanged based on this analysis' own conclusion. |
| **Fixed UniverseDispatcher crashing every cycle since market open** (`AttributeError: module 'Luxury.config' has no attribute 'BREAKOUT_CAPACITY_BACKLOG_MAX_AGE_MINUTES'`) - `primary_cfg = targets[0][1]` resolved to Luxury's config, which never had this flag (only Options' did). Blocked EVERY universe_bucket-sourced entry to Luxury+Futures from ~09:10 IST until the fix (27 CE/26 PE real alerts accumulated with zero entry attempts). Luxury's own independent scanner was unaffected (that's how GVT&D/CGPOWER got entered during the outage - GVT&D already closed +Rs1,775 TARGET_HIT). See [[2026-09-22-dispatcher-crash-loop-market-open]]. | Luxury, Futures | Found live during a user-requested monitoring session, not a backtest. | N/A - deployed commit `02b515a`, restart 03:53:23 UTC (by user's own action, 2 real Luxury + 1 real Options position open at restart time, all confirmed reconciled correctly). Futures picked up a new HDFCBANK CE entry within 35s of restart, confirming the dispatcher path works again. |
| **Broader "any other hidden bugs" audit** (user request, same session) - systematic diff of every config variable across Options/Luxury/Futures against everything the shared cross-package modules reference. Found one more asymmetry (`VOLUME_FLOOR_RATIO_MIN`/`GATE_ENABLED` missing from Luxury) but verified it's safely `getattr`-guarded, not a live risk - not fixed, just documented. Also verified no duplicate-order risk between the dispatcher and each package's own scanner (both converge on the same atomic `reserve_symbol` claim). No other exception types found in logs since the dispatcher fix's restart. | Options, Luxury, Futures | Direct code/log audit, not a backtest. | N/A - diagnostic pass, no further changes made. |
| **Fixed `get_regime_state`/`get_supertrend_state` never throttling on fetch failure** (`Swing/signals.py`) - the cache timestamp was only stamped on success, so once a fetch failed it retried on every 5s monitor tick instead of the intended 60s/15s window. Found during a pre-market review: 22,198 fetch-failure log lines during 21 Sep's market hours alone (~59/min, essentially continuous) for Swing's 5-symbol watchlist. See [[2026-09-22-swing-signal-cache-never-throttled-on-failure]] for the full writeup and its relationship to the 21 Sep session-collision incident. | Swing | User asked for a general pre-market health check (code issues/latency/throttling/race conditions); found via direct log analysis, not a backtest - this is an infra/caching bug, not a strategy-logic change. | N/A - deployed same morning (commit `783925f`, restart 01:55:45 UTC, 0 live positions before/after all 4 packages, `/health` OK). Verified live post-restart: COPPER's retry cadence went from every ~1-2s to ~60-65s apart, matching the intended throttle. |
| **Added diagnostic logging to `fetch_continuous_intraday`** (`Options/dhan_client.py`, commit `38de294`) - logs Dhan's raw response whenever a fetch returns empty, since the plain `{}` return was indistinguishable between a rate limit, an out-of-range request, or anything else. Immediately revealed the real cause of the above: a genuine account-wide `DH-904 Rate_Limit` from aggregate cross-package REST volume (Options/Luxury/Futures scanners + dispatcher + universe_bucket sync + Swing's own polling all sharing one budget) - not anything specific to COPPER/COALINDIA/NATURALGAS as symbols. See the same incident doc's update section. | Swing (diagnostic), account-wide (finding) | User asked to dig into why those 3 specifically kept failing; answered by evidence on the very first post-deploy log line, not guessed. | N/A - diagnostic-only, no behavior change. Deployed same morning (restart 03:29:42 UTC, 0 positions before/after, `/health` OK, auth clean). |
| **Fixed scan-cycle starvation in `_scan_cycle`/`_dispatch_scan_cycle`** (`breakout_signal.py`, commit `182bea1`) - both rebuilt their "pending" symbol list fresh every cycle in stable dict-insertion order, then capped evaluation at `BREAKOUT_SCAN_MAX_PER_CYCLE` (10). A checked-but-not-signaled symbol just landed back in the same front-of-list position next cycle, so anything past the first ~10 was never reached at all - not slow, genuinely never scanned. Confirmed live: UniverseDispatcher had 38 CE + 45 PE tracked symbols, only ~10 CE ever evaluated all session, PE literally 0/45 ever checked (CE alone always exhausted the budget). Luxury's own CE scanner (42 pending) and Futures' (18 pending) were independently hitting the same cap. Fix: track the last symbol examined per strategy and rotate the pending list to resume right after it each cycle - genuine round-robin instead of a fixed front-loaded batch. Verified via simulation before deploying: 83-symbol backlog reaches full first-pass coverage in 9 cycles (~9 min) instead of never finishing. | Options, Luxury, Futures (UniverseDispatcher) | User asked "are all these stocks continuously monitored" - answered by evidence (pending/signaled counts from the live watchlist files), not assumed; fix verified deterministically via simulation, not backtested against real data. | N/A - deployed same session (restart by user's own confirmation), HDFCBANK closed for real profit (+Rs780, TARGET_HIT) within seconds of the restart, confirming the dispatcher path still works correctly post-fix. |
| **Added exponential backoff to Swing's regime/Supertrend fetch retries on consecutive failure** (`Swing/signals.py`, commit `f915d98`) - the "throttle on failure" fix earlier the same day (783925f) stopped a persistently-failing symbol from being retried on every 5s tick, but it still retried at the exact same fixed cadence (60s regime / 15s Supertrend) forever regardless of streak length. Confirmed live: 113 DH-904 failures in 16 minutes post-restart, 60 of them on NATURALGAS alone - Supertrend's 15s base meant up to 4 retry attempts/min on an endpoint failing for account-wide reasons unrelated to that symbol. Fix: double the effective wait on each consecutive failure per (symbol) / (symbol, interval), capped at a 5-min ceiling, resetting to the base interval the instant a fetch succeeds again. | Swing | Live incident, direct log evidence (exact failure timestamps/counts), not a backtest. | N/A - deployed same session (restart 04:38:44 UTC, all 4 packages' positions flat before/after). Backoff widening itself not yet independently re-verified against a fresh consecutive-failure streak post-deploy - worth a spot-check next time NATURALGAS/COPPER hit a sustained failure run. |
| **Diagnosed why 22 Sep's automated WS-candle parity report came back empty** (`recon_bar_count: 0` for all 8 test symbols vs `real_bar_count: 73`) - NOT a WS-reconstruction failure (an earlier 11:13 IST manual check that same morning already proved reconstruction was producing real bars). Root cause: `ws_candle_parity_check.py`'s `maybe_subscribe()` only called the bot's subscribe endpoint ONCE per day; `underlying_candle_feed`'s subscription state is in-memory only and gets wiped on every bot restart. The bot restarted 5 times after the 10:00 IST subscribe that day (all unrelated, legitimate fixes), and the script never noticed or re-subscribed - so the 8 test symbols were only actually subscribed for the first ~8 minutes of the session, not the full day the 15:40 IST report was supposed to cover. Fixed (`a701821`): re-subscribe on every ~5-min tick instead of once - cheap and idempotent, self-heals across any number of restarts, since `underlying_candle_feed.subscribe()` already restores a symbol's persisted bars from disk the first time a fresh process sees it. See [[ws-candle-reconstruction-parity-results]]'s new "Why the automated 15:40 IST report itself came back empty" section for the full writeup. | Cross-cutting (validation tooling, not strategy logic) | User asked to check the scheduled report; the empty result was itself the finding, root-caused and fixed same session. | N/A - `BREAKOUT_USE_WS_CANDLES` stays off everywhere. The open-price question (the whole point of this investigation) remains unresolved - the 11:13 IST manual check is still the best real evidence (3/12 to 11/12 open+close exact matches depending on symbol). Next timer run should finally produce a full-day sample now that the subscribe gap is fixed. |
| **Reduced Swing watchlist 6 -> 2 symbols** (COPPER, NATURALGAS only - dropped ADANIPORTS, ANGELONE, COALINDIA, ASHOKLEY) via `POST /swing/watchlist/replace`, effective immediately, no restart needed. User's own immediate mitigation for the DH-904 finding above - cuts Swing's own share of the account-wide REST call budget by two-thirds ahead of market open. Old watchlist backed up (`data/watchlist.bak.20260922_033307`) by the replace endpoint itself. | Swing | Direct response to the rate-limit finding above, not a backtest - a capacity mitigation, not a strategy change. | N/A - just deployed; whether ADANIPORTS/ANGELONE/ASHOKLEY dropping out changes anything for Swing's own trade count/PnL is something to check once enough days pass under the 2-symbol watchlist. **Revisit after market close today**: whether/how to reduce the account-wide call volume more structurally (Swing fetch pacing, dispatcher/universe_bucket scan cadence) - not addressed yet, this was only the immediate mitigation. |
| `universe_bucket.py`'s `WINDOW_TRADING_DAYS` lowered 3->1 (`UNIVERSE_BUCKET_WINDOW_TRADING_DAYS`) + fixed `universe_dispatcher_loop` to actually WS-subscribe symbols (it never did - only an older, now dispatcher-bypassed code path did) | Luxury, Futures (dispatcher targets) | User request, straight off the ~13.4-min-per-full-pass latency finding in [[ws-candle-reconstruction-parity-results]] (134 symbols / 10-per-60s scan cadence). No backtest - operational config change. | N/A - the window change is forward-looking only; see the data-quality finding below for why it didn't shrink 21 Sep's own count. |
| `ws_candle_parity_check.py` subscribe time moved 11:00 -> 10:00 IST | Cross-cutting | User request | N/A - infra timing only |
| `universe_bucket.py` window split per option type: `WINDOW_TRADING_DAYS_CE=2` (yesterday now trustworthy after the CSV cross-check below), `WINDOW_TRADING_DAYS_PE=1` (never individually verified, stays today-only per user instruction) | Luxury, Futures (dispatcher targets) | User request, off the same CSV cross-check finding | N/A - forward-looking config change |
| Droplet CE/PE bucket files corrected to today-only real counts using the user's own CSV exports; stale `UniverseDispatcher` PE watchlist (90 symbols from a post-close 23:52 IST sync) truncated | Luxury, Futures | Direct data correction, not a code change | N/A - historical-record fix, no restart needed (date had already rolled) |
| **New `BREAKOUT_PAPER_MODE_ENABLED` flag added** (default `false`, per package: `OPTIONS_/LUXURY_/FUTURES_BREAKOUT_PAPER_MODE_ENABLED`) + new shared `breakout_paper_engine.py` module. When true for a package, **replaces** (not augments) real trading for that package - every breakout-scanner signal is simulated with the exact same real entry gates (daily cap, RSI-loss-reentry, loss-repeat + trend check, volume-floor, capacity, liquid-contract resolution, option-liquidity gate) and the exact same real exit ladder, using that package's own `Position` class (`product_type="PAPER"`, `order_id=""`) and `trading_engine.py` functions directly - never a reimplementation. Closed paper trades logged to `history/<date>_breakout_paper_trades.log`, same shape as real trade records, tagged `mode=paper`. Shared monitor loop started once from `main.py`'s own combined lifespan. | Options, Luxury, Futures | User request. Follows Paper01's already-established real-conditions-paper-twin pattern, generalized via a per-strategy dispatch table. No backtest - new capability, inert until explicitly enabled. | N/A - flag defaults false everywhere; deployed and verified live (commit `af87803`, droplet restarted clean, 0 live positions before/after, `breakout_paper_engine` monitor loop confirmed started in logs) but not yet turned on for any package. |

**Data-quality finding while investigating the above**: the user uploaded real Chartink CSV exports ("01 Simply Bull.csv"/"01 Simply Bear.csv", the CE/PE screeners feeding `universe_bucket.py`) and asked to cross-check them against the bot's own 21 Sep bucket (44 CE / 90 PE at the time). The real 21-Sep-only counts were **19 CE / 57 PE** - the bot's persisted `2026-09-21_universe_bucket_{CE,PE}.json` files had an `alert_names` field reading `"Simply Bull - manual backfill (last 2 days: 2026-09-18, 2026-09-21)"` - confirming a manual backfill (from a parallel session, not this one) had merged BOTH 18-Sep's and 21-Sep's real alerts into the SAME 21-Sep-dated file. Verified exactly: CSV's (18-Sep unique ∪ 21-Sep unique) = 44 CE / 90 PE, matching the contaminated file precisely; every real 21-Sep symbol was already present (nothing missing, only 25 CE + 33 PE extras from 18-Sep to remove). Corrected both files to the true 19/57 on the droplet. **No restart needed and no live effect** - by the time this was found, the calendar date had already rolled to 22 Sep and `WINDOW_TRADING_DAYS=1` means the live bot's active window no longer reads 21 Sep's file at all; this was purely a historical-record accuracy fix, not an operational change. Lesson: a "manual backfill" that back-dates alerts across multiple real days into one file's own date silently defeats any window-based sizing control on that file specifically, no matter how the window itself is configured - worth remembering if another backfill is ever done this way again.

### 21 Sep 2026

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| Breakout-signal scanner deployed (flag-enabled, feature-flagged) | Futures | [[futures-breakout-signal-gated-live-full-real-gates]] - 8-trade sample, 87.5% win rate, +Rs13,359.10 vs real Futures' -Rs11,943.55 over the backtest window (same caveats: tiny sample, 3 variables changed at once) | Not yet - deployed today, no real trades under it yet |
| Breakout-signal thresholds loosened to clearance=0.3%/body=0.5%/relvol=1.2x (the 27-combo sweep's #1) | Luxury | [[luxury-breakout-detection-parameter-sweep]] - 72 signals/59 entered/93.2% win rate/+Rs175,346.60 vs real Luxury's -Rs17,909.35 over the same 14-day window | Not yet |
| Risk config raised: MAX_LOSS 5,500/3,100 (CE), PROFIT_PROTECTION_THRESHOLD 5,000/2,500, giveback 8% | Futures | Mirrors the same-day Luxury raise, applied without its own backtest at request time; backtested afterward - [[futures-updated-risk-config-15day-backtest]] (near-breakeven -Rs714.75 vs real -Rs11,943.55, but that backtest was later found to be missing 3 real gates - see the full-real-gates doc above for the corrected number) | Not yet |
| Risk config raised: MAX_LOSS 5,500/3,100 CE / 4,500/2,600 PE, PROFIT_PROTECTION_THRESHOLD 5,000/2,500, giveback 8% | Luxury | Same raise, Luxury side - see [[luxury-signal-gated-live-simulation]]'s "Config raise" section (+Rs73,999.10 delta on the same 18-trade signal-gated sample) | Not yet |
| **Breakout-signal scanner made the SOLE real entry path** (not an additional gate) - `_handle_chartink_webhook` no longer calls `enter_positions_for_stocks` at all for any of the 3; a raw alert only records into the watchlist now. Options widened from PE-only to full CE+PE parity, `OPTIONS_BREAKOUT_SIGNAL_ENABLED` defaulted to true. Same commit fixed the WS market-feed thread-death bug for real (watchdog + exponential backoff, see [[market-feed-thread-death-on-429]]). | Options, Luxury, Futures | Deployed via a separate, parallel Claude Code session on the user's own instruction (commit `b889608`, hotfixed by `db07226` for an accidental `alert_bucket` import break) - explicitly **not backtested as a sole-gate config**, only ever backtested as an additional layer (the two rows above, and [[futures-breakout-signal-gated-live-full-real-gates]]). | **Measured same-day**: 19 trades opened before the 10:07 IST cutover, **0 trades opened in the 2+ hours after** (through 12:26 IST) - the normal alert-driven path's removal, combined with the scanner firing only once all session (see 21 Sep monitoring log above), has visibly collapsed trade frequency. Worth a deliberate decision on whether this is the intended tradeoff before relying on it further. |
| **Swing NSE volume-floor gate lowered 1.2x -> 0.6x** (`SWING_NSE_VOLUME_FLOOR_RATIO_MIN`) - straight off today's real observation that ADANIPORTS and ANGELONE kept qualifying on price/trend (a genuine Supertrend+regime entry signal, repeating every ~7s) but got blocked purely on volume ratio (~0.60-0.69x). MCX's own floor (COPPER/CRUDEOIL/NATURALGAS) left untouched at 1.2x. | Swing | No backtest run before deploying - straight config change on direct user instruction. **Important caveat**: this exact gate was originally promoted from shadow-mode to a live block specifically because of a real ANGELONE 29 SEP 295 PUT loss tied to a low-volume entry (18 Sep 2026) - halving the floor meaningfully reduces that same protection for every NSE watchlist symbol, not just today's two. Worth backtesting or watching closely rather than assuming today's two rejected candidates were false positives. | Not yet - by the time this deployed (~14:14 IST), both ADANIPORTS' and ANGELONE's triggering entry signals had already lapsed (last events 10:00 and 12:30 IST respectively), so this hasn't yet been observed taking a real trade it wouldn't have before. |
| **Permanent guard against local pin_totp session collision** (`dhan_client.authenticate()` now refuses pin_totp auth from any non-systemd process) + **WS-candle parity backtest fixed and re-run** (retry/pacing added, 8/8 symbols clean, 100% close+volume match, open-price gap confirmed structural not a bug) + **passive underlying-feed observation endpoints added** (`/debug/underlying-feed/*`, decoupled from any real entry decision, ready to capture real ticks next market session). | Cross-cutting (Options/dhan_client.py, main.py) | See [[ws-candle-reconstruction-parity-results]] for the full investigation. `BREAKOUT_USE_WS_CANDLES` still **not** enabled for any package - the aggregation math now has strong evidence behind it, but the open-price question specifically needs a real live tick comparison, not another REST replay. | N/A - none of this changes any real trading decision; purely diagnostic/observability work. |

**Backtest of the sole-entry-path config against today's own real alerts**
(user request, same day): replayed every real Chartink alert received
today (09:15 IST onward) through the breakout-only logic for all 3
strategies together (one shared cross-strategy lock, each strategy's
own real risk config) - see `backtest_today_sole_entry_all_strategies.py`.
Result: only **5 signals qualified all morning** across Options/Luxury/
Futures combined (Futures: zero), all 5 entered - CGPOWER (Luxury CE,
+Rs2,125, TARGET_HIT), INDHOTEL (Options CE, +Rs1,730, TARGET_HIT),
BANDHANBNK (Luxury CE, +Rs504, LIQUIDITY_GUARD_ZERO_VOLUME), LICI
(Options PE, -Rs798, TRAILING_SL_HIT), plus a likely-phantom duplicate
INDHOTEL entry on Luxury (same candle, same fill, same instant TARGET_HIT -
the simulation's interval-based cross-strategy lock let both hypothetical
strategies claim it since the position round-tripped within one candle,
which the real momentary lock would not have allowed). Total **+Rs5,291.00
as simulated, more realistically ~+Rs3,561 excluding the phantom
duplicate.** Notably, CGPOWER's signal fired for real too (09:21:03) but
was skipped live (`duplicate_or_capacity_full` - Luxury's real CE
capacity was already full from the now-removed normal-entry path) -
in this counterfactual, that capacity was free instead. One morning is
too small a sample to draw a hit-rate conclusion from (4W/1L here) - see
the caveat on every other backtest in this repo.

**Issue**: first restart attempt for the Futures risk-config deploy hit
a transient PIN+TOTP login failure (`Invalid TOTP`) - systemd's
`Restart=always` auto-recovered on the 2nd attempt within seconds, no
live-position impact. Separately, a Dhan Marketfeed WebSocket incident
(SDK bug, thread dies silently on a startup 429) occurred in roughly
this same window - see [[market-feed-thread-death-on-429]] below; not
caused by these deployments, and bot correctness was never at risk
(REST fallback covered every exit check throughout).

**PnL logging bug fixed** (found during a user-requested live loss
analysis): `trade_history.py`'s `record_closed_trade()` multiplied by
`pos.quantity` unconditionally when computing the PnL to log. Correct
for Options/Futures/Luxury (quantity IS the real rupee multiplier for
them), but wrong for Swing's MCX positions - `quantity` there is just
the lot count (1), while the real rupee-per-point value lives in
`pos.pnl_multiplier` (e.g. 2500 for COPPER), exactly as Swing's own
live exit-decision code already uses. A real -Rs4,600 COPPER loss
today was logged as -Rs1.84. Fixed with a `getattr` fallback so
Options/Futures/Luxury (no `pnl_multiplier` attribute) are unaffected -
committed (`ca7a173`) and pulled to the droplet, but **not yet
restarted** (deliberately deferred by the user - the running process
still has the old buggy logic in memory until the next restart). The
"Real PnL trajectory" table above has already been corrected using the
true entry/exit prices, independent of whether the running bot has
picked up the code fix yet.

**All-F&O-universe breakout-signal backtest, 14 days** (user request:
"what if the scanner watched every F&O stock, not just alerted symbols"):
full detail in [[all-fno-universe-breakout-signal-15day-backtest]], report
artifact https://claude.ai/artifact/X7Adfi1Er4spdDzsDGjbfD. All 210
F&O-eligible symbols, both directions, all 14 real trading days
(2026-08-31 to 2026-09-18), replayed through each package's own real entry
gates and exit ladder (verified-live shared thresholds: clearance=0.15%,
body>=0.5%, relvol>=0.8x). Result: **469 sim trades, 78.0% win rate,
+Rs756,644.71 raw / +Rs688,954.19 with 20 confirmed exact-duplicate trades
(Rs67,690.52) removed**, vs real (Options+Luxury+Futures) 284 trades,
36.6% win rate, -Rs61,827.65 over the same 14 days. **Big caveats, not a
green light**: (1) real PnL here is almost entirely under the OLDER
alert-ranked entry logic, not this same screener - breakout-signal-gated
entry only became the sole real path earlier today, so this compares two
different entry-logic generations, not just two universe sizes; (2) the
same same-instant triple-counting pattern already seen in the "5 signals"
backtest just above this one (the "likely-phantom duplicate INDHOTEL"
case) shows up here at real scale - a `close_t <= t` non-strict-inequality
bug in the shared cross-strategy-lock pruning, still open, not yet fixed
in code; (3) no funds/margin cap modeled, which matters far more at
210-symbol scale than in any narrower prior backtest. Also produced two
real operational findings while running it (local Dhan session collided
with the live bot's own session, forced one real `dhanboy.service`
restart with no open-position impact; separately hit shared Dhan
rate-limit contention with the live bot during market hours) - see
[[local-backtest-dhan-session-collision]].

**Simply Bull screener, 5-day backtest (CE-only, racing logic - pre-
dispatcher)**: user's own real Chartink screener export
(`~/Desktop/future/01 Simply Bull.csv`) used as the per-day curated
universe instead of the full 210-stock scan. Full detail:
[[simply-bull-screener-5day-backtest]]. Last 5 distinct CSV dates
(2026-09-11, 09-16, 09-17, 09-18, 09-21 - 09-15 has zero rows, confirmed
not a gap, the screener genuinely found nothing that day), Luxury+Futures
only. Result: **35 sim trades, 82.9% win rate, +Rs47,257.35**, vs real
(Luxury+Futures, same window) 121 trades, 34.7% win rate, -Rs31,157.40.
Same "compares new filter vs. old entry logic, not apples-to-apples"
caveat as the 210-stock backtest above.

**DEPLOYED LIVE (commit `fbd11f9`, restart 15:05 UTC / 20:35 IST)**:
cross-package universe_bucket signal dispatcher with a capacity backlog,
for Luxury + Futures only, per explicit user go-ahead given AFTER I laid
out the risk explicitly (same-day-built code, zero prior live validation,
requires a restart) and asked for separate confirmation - see
[[universe-bucket-dispatcher-design]] for the full design and
[[all-fno-universe-breakout-signal-15day-backtest]]/[[simply-bull-screener-5day-backtest]]
for the backtests behind it.

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| `universe_bucket.py` deployed - rolling 3-trading-day CE/PE bucket, fed by two new webhooks (`POST /universe-bucket/webhook` bullish, `/webhook-sell` bearish) matching Chartink's own payload shape | Shared (not per-package) | New module, no dedicated backtest of the bucket mechanics itself - see the two backtests above for the strategies that read from it | N/A - bucket is EMPTY until the user's own new Chartink screener is pointed at the webhook ("after I integrate it" - not yet done as of this entry). Live code, no real trading effect yet. |
| `UNIVERSE_DISPATCHER_ENABLED=true` - dispatcher detects each universe_bucket signal ONCE (not once per package, closing the race the earlier independent-per-package-sync design had), offers it to Luxury then Futures round-robin+sequential-fallback, with a capacity backlog retrying a `duplicate_or_capacity_full`-only rejection once a slot frees, gated by a real momentum-reversal check (`reversal_filters.check_underlying_move_confirms_exit`, the same already-backtested 0.10% threshold) every retry | Luxury, Futures | [[dispatcher-capacity-backlog-comparison]] - 5-day Simply Bull backtest found ZERO measurable backlog effect (31 trades both with/without - capacity was never the actual binding constraint at that signal volume); 210-stock/14-day comparison queued to test the scale where capacity contention is real (890 `capacity_full` skips confirmed in the racing simulation) | Not yet - bucket empty, see above |
| `BREAKOUT_USE_WS_CANDLES` - deliberately NOT enabled in this deploy | Options, Luxury, Futures | [[all-fno-universe-breakout-signal-15day-backtest]]'s own WS-candle parity check found close/volume reconstruction reliable but open-price accuracy unverified under a genuine live tick stream (the REST-replay test method can't validate that) | N/A - stays REST-only for now, scoped out deliberately |

**Pre-existing, uncommitted, unrelated changes found in the working tree
during this deploy** (NOT from this session, NOT included in the deploy):
`Options/Luxury/Futures trading_engine.py`/`position_store.py` and a new
`alert_bucket.py` implementing the loss-triggered bucket-switch feature
(see [[alert-bucket-switch]]) reference `config.BUCKET_SWITCH_ENABLED`/
`BUCKET_SWITCH_LOSS_RS`/`BUCKET_SWITCH_MIN_SCORE`, none of which are
defined in any config.py - a real `AttributeError` waiting to fire on
every position-monitoring tick once BUCKET_SWITCH is reached. Deliberately
excluded from this deploy's commit (`main.py`'s `import alert_bucket` +
router-mount lines were also removed from what got committed, for the
same reason) - left untouched, uncommitted, in the local working tree for
whoever finishes that work. Worth fixing (define the 3 missing config
keys) before that feature is ever deployed on its own.

**Post-deploy verification** (all done before/immediately after restart,
no open positions on any package throughout): `/positions`,
`/futures/positions`, `/luxury/positions`, `/swing/positions` confirmed
empty pre-restart; `journalctl` confirmed clean startup - "UniverseDispatcher
started for targets=['Luxury', 'Futures']", both CE/PE watchlists and both
universe_bucket CE/PE buckets initialized with 0 symbols, zero tracebacks
from any of the new modules (the only tracebacks in the startup window are
the pre-existing, unrelated `swing_signals` "could not fetch Supertrend
state" pattern, self-healing via cached values); `/health` and
`/universe-bucket` (returning the correct rolling window
`["2026-09-21","2026-09-18","2026-09-17"]`, weekend correctly skipped) both
responding; droplet RAM 521Mi available post-restart, swap barely touched,
`systemctl is-active` = active.

**Simply Bear screener, 5-day backtest (PE-only, first run using the
LIVE dispatcher+backlog logic)**: PE counterpart to the Simply Bull
backtest above, but run AFTER the dispatcher deploy, using the actual
deployed dispatch mechanism rather than a reconstruction of the older
racing logic. Full detail: [[simply-bear-screener-5day-backtest]]. Last 5
distinct CSV dates (2026-09-15, 09-16, 09-17, 09-18, 09-21 - a genuinely
consecutive run this time, unlike Simply Bull's own gap-at-09-15),
dropped NIFTY/BANKNIFTY (indices, not F&O stocks, included as plain rows
in this CSV). Result: **34 sim trades, 82.4% win rate, +Rs50,159.14**, vs
real (Luxury+Futures, same window) 120 trades, 36.7% win rate,
-Rs20,968.65. Zero same-instant duplicate captures this run (unlike
Simply Bull's 3) - the round-robin dispatch cleanly split load with no
collisions in this dataset.

**Bucket ARMED with real candidates, 21:19 IST** - user's own two Chartink
screener exports (`~/Desktop/future/01 Simply Bull.csv` -> CE,
`01 Simply Bear.csv` -> PE), last 2 distinct CSV dates (2026-09-18,
2026-09-21) unioned and deduplicated, POSTed directly to the LIVE
`/universe-bucket/webhook` (44 symbols) and `/webhook-sell` (90 symbols) -
same endpoints a real Chartink scan would hit, per explicit user
instruction ("push it in a droplet as this was live data from the two
webhooks I have integrated... make sure this data is inserted into both
these two buckets in droplet"). All 134 symbol-pushes F&O-eligible
(checked locally first via the hand-off-token pattern, zero dropped).
Confirmed via `GET /universe-bucket` and `journalctl` (clean webhook log
lines, only the same pre-existing unrelated `swing_signals` tracebacks in
the surrounding window, `/health` OK).

**Real consequence, stated plainly**: this is the bucket the ALREADY-LIVE
dispatcher (`UNIVERSE_DISPATCHER_ENABLED=true`) reads every scan cycle.
Pushed at 21:19 IST, well after market close, so nothing fired tonight
(`_market_hours_now()` blocks it) - but the bucket now has 44 CE + 90 PE
real candidates sitting in it, and at the NEXT market open (09:15 IST,
2026-09-22) the dispatcher will autonomously evaluate every one of them
for a real breakout/breakdown and can place REAL Luxury/Futures orders on
any that qualify, with NO further per-trade confirmation - exactly the
behavior already authorized when `UNIVERSE_DISPATCHER_ENABLED` was set
true, now with real symbols in the bucket for the first time instead of
an empty/inert one. One caveat on the mechanism itself: the webhook
always records into the CALLING day's own file (today, 09-21) regardless
of which original CSV date a symbol came from - the 09-18 and 09-21 rows
both landed in today's file, not two separate day-buckets. No effect on
the ACTIVE window today (today is always included regardless), but worth
knowing if the daily-file audit trail is ever inspected later.

**Broker SL-L bug found and fixed (user request: "investigate into it") +
Friday square-off moved 15:20->15:25 (user request), commit `e70804d`,
restart 16:30 UTC / 22:00 IST**: investigated a real error from this
morning (LICI, Options, 09:19 IST) - full writeup
[[2026-09-21-lici-negative-broker-sl-trigger]]. Root cause: for a
cheap-premium, large-quantity position whose entire notional value is
already below the rupee MAX_LOSS cap, `_place_broker_stop_loss_if_
enabled`'s trigger-price formula computes a NEGATIVE price (LICI:
fill=0.70, qty=1400, cap=Rs3,500 -> trigger=-1.80), which Dhan correctly
rejects. Same bug confirmed in all 3 packages (identical formula). Real
risk impact was low - the percentage-based `STOP_LOSS_PCT` hard stop
(poll/tick-driven, unaffected) still protected the position; only the
redundant broker-side layer was missing for this one trade. Fixed
identically in all 3: skip broker SL placement cleanly (INFO, not ERROR)
when `trigger_price <= 0`.

Also moved `FRIDAY_SQUARE_OFF_TIME` 15:20->15:25 across all 3 packages
per explicit user request ("not carry forward any position after Friday
15:25 PM, square them off") - this rule already existed
(`ENABLE_FRIDAY_SQUARE_OFF=true` since 26 Aug 2026), only the cutoff time
itself changed. **Real deploy mistake caught and corrected same session**:
the first restart only changed the CODE DEFAULT, but `.env` had explicit
`FRIDAY_SQUARE_OFF_TIME=15:20`/`FUTURES_FRIDAY_SQUARE_OFF_TIME=15:20`
overrides that silently shadowed it - Options and Futures were still
running 15:20 after that restart, only Luxury (no override) picked up
the change. Caught by explicitly checking the EFFECTIVE runtime value on
the droplet after the first restart, not just trusting the code diff;
fixed `.env`, re-synced, restarted again, and re-verified all three
packages' actual runtime `FRIDAY_SQUARE_OFF_TIME` read 15:25 before
calling it done. Lesson: a code-default change to a value that also has
a live `.env` override does NOT take effect until the override is found
and changed too - always verify the EFFECTIVE value post-deploy, not
just that the code changed.

Pre-restart (both restarts): confirmed zero open positions on all 4
packages. Post-restart (both): clean startup, dispatcher restarted
correctly, universe_bucket's 44 CE/90 PE symbols survived intact,
`/health` OK, only the same pre-existing unrelated `swing_signals`
tracebacks in the log.

### 20 Sep 2026

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| Breakout-signal live entry trigger deployed (flag-enabled), separate CE/PE watchlists with daily refresh | Luxury | [[breakout-scanner-vs-real-pnl]] (original recalibration work) + [[luxury-signal-gated-live-simulation]] (real-gate replication, +Rs32,649.10 on the real exit stack before the config raise above) | 0 real trades attributed to the signal path through 21 Sep (feature just launched; normal alert-driven trading continued in parallel and is what's reflected in the 17/18 Sep rows above) |

### 19 Sep 2026

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| **Opening-burst extra CE capacity slot, actual code deploy** (`_cap_for()` in each package's `position_store.py` adds `BURST_EXTRA_SLOTS_CE` to `MAX_LIVE_POSITIONS_CE` while inside the configured 09:15-10:00 IST window - wired through the real `reserve_symbol()`/`remaining_capacity()` enforcement path, not a cosmetic flag; CE only, the backtest never modeled PE) (commit `d454d74`, `tests/test_burst_capacity.py`) - the backtest (17 Sep, see row above) and this deploy are 2 days apart; the journal previously conflated them into one 17-Sep row. Deployed flag-on by default per explicit user request despite the backtest's own thin-sample caveat (15-37 picks over 6 days) - user chose to ship live rather than flag-off first. **Backfilled into this journal 23 Sep** - the deploy itself was missed at the time (only the earlier backtest got logged); caught by a follow-up cross-package documentation audit. | Options, Futures, Luxury | [[opening-burst-slot-and-sl-target-sensitivity]] (same backtest as the 17 Sep row above) | Reflected in the 20/21/22 Sep rows above once enough days accumulate under it specifically - not yet separately isolated from the base capacity config. |

### 18 Sep 2026 (bundle - multiple commits same day)

Option-liquidity entry gate added (SOLARINDS-style illiquidity
prevention), MA-ribbon-expansion ranking turned on for Futures CE+PE,
RSI-loss-reentry / loss-repeat-block broadened to count ANY real loss
(not just MAX_LOSS_HIT/STOP_LOSS_HIT - ATHERENERG incident), minimum
underlying-move confirmation gate added before SUPERTREND_EXIT/
EMA_CROSS_EXIT, market-hours guard added to LTP-staleness forced exit.
See [[liquid-contract-resolution]] and NOTES.md (traderBoy repo) for
the per-commit detail - this was a dense bug-fix/hardening day more
than a single strategy pivot.

### 17 Sep 2026

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| Opening-burst extra CE capacity slot backtested (+1 slot, 09:15-10:00 IST, 6 days of real alert/shadow data - a single extra slot held open the full window beat a second permanent slot by cycling through positions as they resolve) | Options/Futures/Luxury | [[opening-burst-slot-and-sl-target-sensitivity]] | Backtest only on this date - see 19 Sep below for the actual code deploy (this row previously read as if deployed here; **corrected 23 Sep** after a cross-package audit found the real commit is dated 19 Sep, 2 days after this backtest) |
| Global liquid-contract-resolution gate (`get_liquid_atm_option`) | All 4 (Options/Futures/Luxury/Swing) | [[liquid-contract-resolution]] - built after the ATHERENERG broker-stop-rejection incident | Reflected in every row from 17 Sep onward |
| Kaufman Efficiency Ratio added as a shadow-mode-only logging field (never blocks, threshold 0.3); Futures ported Options' live volume-floor gate (1.2 ratio); Swing got ADX/RSI/volume-ratio/ER shadow logging on every real entry | Options/Futures/Luxury (ER logging), Futures (volume floor), Swing (shadow logging only) | [[reversal-trend-strength-filter-arc]] - rounds 3/4 of the same arc (`b43a873`, `416b3ff`); ER's standalone evidence (+Rs10,925.75/143 trades, +Rs5,561.25 incremental over the live volume floor) judged "thin and lumpy" (85% from one day), so kept shadow-only | Reflected in every row from 17 Sep onward (volume floor); ER logging is diagnostic only, no real-trade effect |
| `LOSS_REENTRY_TREND_CHECK_ENABLED`: live ADX(>=20)/ER(>=0.3) re-entry gate (either one passing is enough) for Options/Futures/Luxury, gating a re-entry on a symbol that already lost money today | Options/Futures/Luxury | [[reversal-trend-strength-filter-arc]] (round 1-4 findings, `d758e6b`) - real incident: ATHERENERG 29 SEP 1540 PUT (17 Sep) whipsawed out via SUPERTREND_EXIT with a logged ADX of 13.24 (well below the 20 threshold), then a same-day re-entry lost again; see also the 18 Sep bundle row below for this same commit's loss-repeat-block broadening | Reflected in every row from 18 Sep onward (also see 18 Sep bundle below) |

### 16 Sep 2026

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| Shadow-mode-only logging of ADX/RSI/volume-ratio/RSI-extreme-plus-volume-spike "climax combo"/post-SUPERTREND_EXIT cooldown for every real entry (`reversal_filters.py`, never blocks) | Options/Futures/Luxury | [[reversal-trend-strength-filter-arc]] - rounds 1+2 of a 5-round arc triggered by that morning's PAYTM (22s, -Rs4,603.75) and YESBANK (61s, -Rs4,043.00) near-instant reversals (`5291efa`) | Diagnostic only, no real-trade effect |
| Volume floor promoted from shadow-mode to a **live blocking gate** (entry-candle volume < 1.2x its 20-bar average blocks the entry), Options + Swing/MCX only | Options, Swing (MCX symbols only) | [[reversal-trend-strength-filter-arc]] - same rounds 1+2, "the single strongest individual filter across two backtest rounds" (+Rs7,131.50 on 37 trades/2 days, +Rs17,123.00 on 139 trades/15 days) (`19c551d`) | Reflected in every row from 16 Sep onward |

### 15 Sep 2026

**Backfilled into this journal 23 Sep** - all three rows below were missed
at the time despite each having a real commit (and, for the first two, a
real incident); caught by a follow-up cross-package documentation audit.

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| **Swing: PROFIT_PROTECTION_RS/GIVEBACK_PCT split by basket_type for OPTIONS** (`PROFIT_PROTECTION_RS_OPTIONS`/`_GIVEBACK_PCT_OPTIONS`, commit `3ddc59a`) - user request straight off switching `BASKET_TYPE` to options: the existing flat 5000/2% thresholds were tuned against FUTURES-notional P&L, and an option's own premium swings represent a much smaller absolute rupee move for the same underlying move, so the same threshold armed far later relative to a typical options trade's real profit potential. Falls back to the shared value when unset (FUTURES/EQUITY baskets unaffected), evaluated fresh off `position.basket_type` on every exit check, not baked in at entry. Deployed: `SWING_PROFIT_PROTECTION_RS_OPTIONS=2000`, `SWING_PROFIT_PROTECTION_GIVEBACK_PCT_OPTIONS=0.02`. Same pattern later extended by exchange segment rather than basket_type - see the 23 Sep MCX-specific `PROFIT_PROTECTION_RS_MCX=4000` change elsewhere in this repo's `.env` history. | Swing | User request, no backtest - risk-parameter retune off a structural observation (options premium vs futures notional), not a signal change. Full `test_swing_v2_*` suite (30 tests) re-verified unchanged. | N/A - too new to judge at the time; superseded in practice once COPPER structure-break went live 22 Sep under its own separate config. |
| **Swing/COPPER placed 4 duplicate real BUY orders** - `is_market_open()` checked only NSE hours regardless of segment, wrongly AMO-tagging a genuinely-live 20:10 IST MCX order; Swing's strict TRADED-only fill discipline then hot-retried the "failed" entry every 5s with no cooldown until Dhan's margin engine started rejecting with `DH-906`. Fixed same day (`2023e09`): MCX-aware `is_market_open(exchange_segment=...)`, plus a new 180s per-symbol `ENTRY_RETRY_COOLDOWN_SECONDS` closing the underlying retry-storm risk independently of this specific trigger. See [[2026-09-15-swing-copper-mcx-hours-duplicate-entry-orders]] and its Known-issues row above. | Swing | Real incident, direct evidence (4 confirmed-cancelled duplicate orders, DH-906 rejections) - not a backtest. New tests: `test_mcx_market_hours.py` (4), `test_swing_entry_retry_cooldown.py` (5). | N/A - all 4 duplicate orders confirmed cancelled at the broker before this fix deployed; no residual exposure. |
| **Swing: proactively discover resting broker stop-loss on reconciliation** (commit `d54f270`) - closed the same `reconcile_broker_positions()` gap the JSWENERGY incident found in Options/Futures/Luxury earlier the same day, applied to Swing specifically ahead of turning `SWING_V2_BROKER_STOP_LOSS_ENABLED` on live for the first time (Swing had a real open COPPER position at the time). Side-aware (a SHORT's resting order is a BUY, not a SELL) via `exit_transaction_type(side)`, unlike the other three packages' hardcoded `"SELL"`. See [[2026-09-15-jswenergy-orphaned-stop-loss-and-icicipruli-stuck-order]]'s "Update, same day" section. | Swing | Direct defense-in-depth fix ahead of a live risk-parameter flip, not a backtest. | N/A - preventive; no incident traced to this specific gap for Swing (unlike JSWENERGY itself, which was the Options/Futures/Luxury instance of the same bug). |

### 14 Sep 2026

- Nifty gap-down max-delay ceiling lowered 120→30 minutes
  (`GAP_DOWN_MAX_DELAY_MINUTES`) - shared across Options/Futures/Luxury.
- PE-specific MAX_LOSS_PER_TRADE_RS caps introduced (tighter than CE) for
  Options/Futures/Luxury.
- Swing v2 entry strategy version set explicitly (`v2`).
- `EXECUTOR_MAX_WORKERS` held at 5 pending a live-hours observation
  window - see [[executor-sizing-ws-storm-on-closed-market]] (incident,
  13 Sep) for why this is deliberately not yet raised to 10.

### 13 Sep 2026

**Backfilled into this journal 23 Sep** - missed at the time, caught by a
follow-up cross-package documentation audit.

- **Swing v2: Copper always trades OPTIONS, independent of the global
  `BASKET_TYPE`** (commit `18c0856`) - user correction (12 Sep, deployed
  just after midnight 13 Sep) in two parts: Copper no longer SKIPS
  entirely when `BASKET_TYPE` happens to be "futures" (now always enters
  as OPTIONS regardless), and the override is scoped to a new, separate
  `MCX_OPTIONS_ONLY_SYMBOLS` set (default `COPPER`) rather than a blanket
  rule for every `MCX_SYMBOLS` member, so a future MCX symbol could still
  route through the plain futures path if that's ever wanted. Computed
  as an `effective_basket_type` used everywhere (side resolution,
  instrument-branch dispatch, funds-check payloads, the stored
  `Position`) instead of the raw global config. User request, no
  backtest - a scoping/behavior correction, not a signal change. Full
  test suite re-verified (`tests/test_swing_v2_mcx_entry_exit.py` test 2
  rewritten, new test 5 added as the scoping regression guard).

### 12 Sep 2026

- **Swing v2 complete rewrite** (old v1 basket/sequential/basket_hedge
  design fully discarded) - `BASKET_TYPE=futures` live again for
  ADANIPORTS+COALINDIA, `PROFIT_PROTECTION_RS=5000`/`giveback=2%` per a
  10-day sweep (+Rs44,040 across the PP-threshold values tried, giveback
  was the real lever). `MAX_CONCURRENT_TRADES` raised 1→2.
- PROFIT_PROTECTION_THRESHOLD_RS raised 1,500/1,000 → 2,000/1,500 and
  giveback 2%→3% for all 3 packages - backtest-driven for Luxury only
  (+Rs22,750 vs +Rs19,262 on a real 21-trade sample), applied to
  Options/Futures without their own backtest at the time (explicitly
  flagged in the `.env` comment as a gap worth reviewing once real
  trades accumulate - the 15/16/17/18 Sep rows above are that review).
- `BROKER_STOP_LOSS_LIMIT_GAP_MULTIPLE` tied directly to the MAX_LOSS_HIT
  cap in rupees (replacing a flat 3%-of-price buffer).

### 11 Sep 2026

- "Idea 1": broker-side SL-L stop-loss order turned on for Options/
  Futures (already live on Luxury since 8-9 Sep) - backtest against 14
  real MAX_LOSS_HIT trades found ~Rs2,777 of that day's losses were pure
  polling overshoot this closes.
- "Idea 4": `LOSS_REPEAT_BLOCK_COUNT` behavior discussed (INDUSTOWER
  double-loss case, would have saved +Rs1,615 at a stricter count) - **the
  currently deployed value is 2** for all 3 packages; the `.env`
  narrative comment describes tightening to 1, so this is worth a
  direct check next time re-entry blocking is reviewed (flagged as an
  open question below, not resolved here).
- Luxury's own MAX_LOSS_PER_TRADE_RS raised 1,500/1,100 → 4,500/2,100
  (Options/Futures left at 1,500/1,000).

### 9-10 Sep 2026

- Real broker-side stop-loss mechanism root-caused: the "STOP_LOSS_MARKET"
  order type added 8 Sep does **not** place a real conditional stop at
  Dhan (NSE bans SL-M for options exchange-wide; Dhan silently fills it
  as an instant LIMIT sell instead of rejecting it) - see
  [[luxury-sl-m-orders-fill-as-limit]]. Replaced with a genuine SL-L
  (stop-loss-limit) order, confirmed via two controlled live tests,
  enabled for real production entries 9 Sep.
- Swing **paused entirely** (9 Sep) after basket_hedge changes still
  showed poor real results.
- Multi-window entry schedule (`09:15-14:00,14:00-15:28`, superseding the
  single-cutoff pair) turned on for Options/Futures/Luxury (10 Sep).
- Fixed +TARGET_PCT exit turned ON for all 3 packages; EMA-cross exit
  turned on for all 3 (10 Sep).

### 31 Aug - 8 Sep 2026

- Swing package created (31 Aug, deployed disabled), then taken live
  1 Sep after explicit risk confirmation.
- 2-bucket fund allocation (85% Swing / 15% Options+Futures+Luxury
  shared) added 1 Sep; secondary bucket raised 15%→20%→25%→30% across
  8-9 Sep after a real skipped SOLARINDS entry for insufficient margin.
- Luxury's target/stop matched to Options' own deployed 0.20/0.16 (1 Sep;
  Luxury's code default of 0.10/0.03 had drifted).
- Risk-threshold cutoff (`RISK_THRESHOLD_CUTOFF_TIME=11:30`) introduced,
  splitting MAX_LOSS_PER_TRADE_RS/PROFIT_PROTECTION_THRESHOLD_RS into
  before/after pairs (31 Aug).
- MAX_LIVE_POSITIONS_CE/PE tightened to 1 each for Options/Futures/Luxury
  (1 Sep) after several rounds of 2↔3↔4 tuning across 27-31 Aug.

### Earlier (24-27 Aug 2026)

Foundational risk-cap tuning: MAX_LOSS_PER_TRADE_RS lowered
1500→1200/1000 after a CSV backtest; MONITOR_INTERVAL_SECONDS lowered
5→2 and LTP_STALE_AFTER_SECONDS added after the SAGILITY overshoot
incident (28 Aug - see [[sagility-gap]]); NRML/carry-forward mode
(`ENABLE_SQUARE_OFF=false` + Friday-only carve-out) adopted after
backtest evidence; a re-entry-escalation multiplier was tried and
reverted same-day in favor of the current flat cap. No real
`real_trades.log` data exists for this period (starts 31 Aug) - these
are documented in `.env`'s own comments and NOTES.md (traderBoy repo)
but have no directly-attributable real before/after PnL here.

---

## Known issues (chronological index of `incidents/`)

| Date | Incident | Real trading impact |
|---|---|---|
| 28 Aug | [[sagility-gap]] | MAX_LOSS_HIT overshot its cap by ~Rs600 due to a stale WS-cached LTP - root cause of the MONITOR_INTERVAL_SECONDS/LTP_STALE_AFTER_SECONDS changes above |
| 2 Sep | [[ohlc-outage-zero-trades]] | Zero trades for a session |
| 3 Sep | [[cholafin-overshoot-and-luxury-corrective-actions]] | MAX_LOSS_HIT overshoot on a real Luxury trade |
| 8 Sep | [[mahabank-phantom-pe-hedge-exit]] | Swing v1 phantom exit |
| 8 Sep | [[test-suite-real-auth-leak]] | Test infra leaked a real auth attempt |
| 9 Sep | [[luxury-sl-m-orders-fill-as-limit]] | Root cause of the SL-M→SL-L broker-stop fix above |
| 10 Sep | [[icicipruli-unmonitorable-position]] | A position went unmonitorable |
| 11 Sep | [[test-suite-real-auth-leak-recurrence]] | Same test-infra leak recurred |
| 13 Sep | [[executor-sizing-ws-storm-on-closed-market]] | Why EXECUTOR_MAX_WORKERS is still held at 5 |
| 15 Sep | [[jswenergy-orphaned-stop-loss-and-icicipruli-stuck-order]] | Orphaned broker-side stop-loss |
| 17 Sep | [[copper-mcx-security-id-collision-and-adoption]] | MCX security-id collision |
| 17 Sep | [[pageind-orphaned-exit-and-swing-signal-caching]] | Orphaned exit + signal caching bug |
| 18 Sep | [[abb-ltp-blackout-max-loss-overshoot]] | Another MAX_LOSS_HIT overshoot, different root cause than 28 Aug |
| 18 Sep | [[standalone-test-real-auth-attempt-swing-mcx]] | Test infra, Swing/MCX auth leak variant |
| 21 Sep | [[market-feed-thread-death-on-429]] | Dhan SDK bug (vendored `dhanhq` MarketFeed thread dies silently on a 429), not this repo's own code. No trading-correctness impact - REST fallback covered every exit check throughout. Fix not yet built. |
| 21 Sep | Swing MCX PnL logging bug (this journal, no separate incident file) | `record_closed_trade()` understated every Swing MCX trade's logged PnL by orders of magnitude (used lot-count `quantity` instead of `pnl_multiplier`). No trading-decision impact - only the historical log was wrong, live exit decisions always used the correct multiplier. Fixed (`ca7a173`), pulled to droplet, restart deliberately deferred by user. |
| 21 Sep | [[local-backtest-dhan-session-collision]] | A local backtest script's own Dhan re-authentication invalidated the live bot's session, forcing one real, unplanned `dhanboy.service` restart. No open positions affected (confirmed via `/health`/`/positions` before and after). Fixed for future local scripts: use a user-supplied hand-off `access_token` instead of a competing local `pin_totp` login; defer bulk historical-data pulls to after market close (separate shared-rate-limit issue, not fixed by the access-token change). **Correction**: this is also the real TRIGGER behind the recurring `swing_signals` "could not fetch Supertrend/regime state" errors seen throughout the day across COPPER/NATURALGAS/ASHOKLEY/ANGELONE - repeatedly (and incorrectly) treated as a standalone Swing/WebSocket data issue in earlier analysis this same day, before this incident's own writeup connected it to the many local backtest scripts (mine included) re-authenticating via pin_totp during market hours. **Permanently fixed** (`c61e82e`): `dhan_client.authenticate()` now refuses pin_totp auth from any process not running under systemd. **Further correction (22 Sep)**: fixing the trigger wasn't the whole story - a separate caching bug ([[2026-09-22-swing-signal-cache-never-throttled-on-failure]]) meant each triggered failure never self-healed within the intended 60s/15s window and instead retried every 5s indefinitely, which is why the error count stayed this high (22,198 in one day) well past any single collision event. Both are now fixed. |
| 22 Sep | [[2026-09-22-swing-signal-cache-never-throttled-on-failure]] | `get_regime_state`/`get_supertrend_state` (`Swing/signals.py`) only stamped their cache timestamp on a successful fetch, so a failing symbol retried on every 5s monitor tick instead of the intended 60s/15s refresh window - up to ~288 calls/min once triggered. Explains why the 21 Sep session-collision incident's fetch failures ran essentially continuously for hours instead of a few isolated blips. No incorrect order placed (fail-open design, falls back to last good cached value) - real impact was hours of stale regime/Supertrend reads for Swing (85% of fund allocation) plus needless Dhan API load. Fixed and deployed same morning before market open (commit `783925f`), verified live: retry cadence now correctly ~60-65s apart. **Update same morning**: throttle fix didn't stop 3 of 6 Swing symbols (COPPER/COALINDIA/NATURALGAS) from still failing every attempt - added diagnostic logging (`38de294`) which immediately revealed the real cause: genuine account-wide `DH-904 Rate_Limit` from aggregate cross-package REST volume, not anything specific to those symbols. Immediate mitigation: Swing watchlist cut 6 -> 2 symbols (COPPER, NATURALGAS only) via `/swing/watchlist/replace`. Structural fix for the shared call-budget question **deliberately deferred to after market close 22 Sep**. |
| 22 Sep | [[2026-09-22-dispatcher-crash-loop-market-open]] | UniverseDispatcher crashed every 60s scan cycle from ~09:10 IST market open (`AttributeError` on a config flag only defined in Options' config, while the dispatcher read it off Luxury's via `primary_cfg = targets[0][1]`). Blocked every universe_bucket-sourced entry to Luxury+Futures for ~13 minutes real market time (27 CE/26 PE real alerts accumulated, zero entry attempts) - real trading impact, not hypothetical. Luxury's own independent scanner was unaffected (GVT&D/CGPOWER entries during the outage came through that path; GVT&D already closed +Rs1,775 TARGET_HIT). Fixed same session (`02b515a`), deployed with 3 real open positions across 2 packages, all confirmed reconciled correctly post-restart. |
| 22 Sep | [[2026-09-22-ws-candle-open-price-root-cause-ltt]] | WS candle reconstruction's open-price mismatch (first found via live parity checks earlier the same day) root-caused: ticks bucketed by local receipt time instead of the exchange's own `LTT` (Last Trade Time), so a trade processed just after a 5-min boundary got misattributed to the wrong bar - corrupting that bar's open specifically while close/volume stayed accurate. No incorrect order placed (`BREAKOUT_USE_WS_CANDLES` has been off throughout this entire investigation). Fixed (`a6e026e`), deployed, unit-tested (5 new tests). **Still open as of 22 Sep evening**: a same-day re-check attempt ran after 15:30 IST market close and caught two mid-check `dhanboy` restarts, each wiping the WS feed's in-memory subscription state - zero usable post-fix bars collected (0-1 recon bars vs 73 real REST bars, all 8 test symbols). Live re-validation still needs the next trading session, subscribing at/near 09:15 IST open before any restart can wipe state. **Update 23 Sep 09:27 IST**: tried again right at market open - 3 `dhanboy` restarts hit in the first 90 minutes of the day (08:00 known timer, 08:42 and 09:21 unexplained), the last one landing mid-bar and wiping the subscription again. Resulting sample (0/8 exact opens, 79-99% volume deviation) looked worse than the pre-fix baseline at first glance, but both sampled bars are individually explained by subscription-continuity gaps (subscribed 1 min after the 09:15 bar's own start; the 09:21:46 restart truncating the 09:20 bar to its last ~1 minute) - not a regression in the LTT fix, which addresses tick-to-bar bucketing, not subscription continuity. Still no clean, fully-covered bar to fairly judge the fix on - waiting on a bar with no restart interrupting it. **Update 23 Sep 09:30 IST - first clean bar, strong positive result**: no restart occurred 09:24-09:30, so the 09:25 bar was fully covered by continuous subscription. Result: 6/8 exact open matches, 5/8 exact volume matches, and the 2 open misses were fractional (0.6 and 0.05 rupees) - nothing like the multi-rupee gaps pre-fix. TCS and ICICIBANK, yesterday's worst performers (50%/25% exact-match), were BOTH exact on this bar. Still n=1 clean bar, not a full-session sample, but the first genuinely uncontaminated evidence this investigation has produced, and it's clearly positive. **FINAL 23 Sep 09:44 IST - confirmed with a real sample, 15-min clean monitoring window, no restart**: 26 clean bars across all 8 symbols - 18/26 (69.2%) exact open matches, vs 25.5% pre-fix (~2.7x improvement), remaining misses all fractional (max 0.9 rupees, none multi-rupee). Volume on clean bars similarly much improved. ITC shows a small, unusually consistent 0.05-rupee miss on all 3 of its clean bars - distinct pattern worth a separate look, but negligible size (~0.02%, far below the 0.5% BREAKOUT_MIN_BODY_PCT threshold). **Verdict: LTT fix confirmed working** with real evidence, not a fluke. `BREAKOUT_USE_WS_CANDLES` stays off - enabling it is a deliberate user decision, not automatic from this result. Separately, the 3 restarts in today's first 90 minutes (1 known timer, 2 unexplained) remain an open operational concern independent of this fix. |
| 21 Sep | [[2026-09-21-lici-negative-broker-sl-trigger]] | LICI (Options PE) broker-side SL-L order rejected - trigger price went negative for a cheap-premium/large-quantity position whose notional value was already below the rupee MAX_LOSS cap. Same bug in all 3 packages, fixed same day (commit `e70804d`). Real risk impact low - percentage-based STOP_LOSS_PCT hard stop was unaffected the whole time. |
| 15 Sep | [[2026-09-15-swing-copper-mcx-hours-duplicate-entry-orders]] | Swing/COPPER placed 4 duplicate real BUY orders in under a minute after `is_market_open()` wrongly tagged a genuinely-live 20:10 IST MCX order as AMO (checked only NSE hours), which Swing's strict TRADED-only fill discipline then treated as a failed entry and hot-retried every 5s with no cooldown. All 4 confirmed cancelled at the broker by Dhan's own margin engine (`DH-906`) before the fix - no manual cleanup needed, but the retry mechanism itself was real. Fixed same day (`2023e09`): MCX-aware `is_market_open(exchange_segment=...)` split, plus a new `ENTRY_RETRY_COOLDOWN_SECONDS` (180s) closing the underlying retry-storm risk independent of this specific trigger. **Backfilled into this journal 23 Sep** - missed at the time, caught by a follow-up cross-package documentation audit. |
| 23 Sep | [[2026-09-22-swing-overnight-polling-and-friday-squareoff-fix]] | Swing's entry-evaluation path (`get_regime_state`/`get_supertrend_state`, structure-break refresh) polled Dhan unconditionally every 5s, 24/7 - 39 real `DH-904` rate-limit hits confirmed in under an hour overnight, zero possible benefit (nothing can fill outside trading hours). Fixed same night (`cc70363`): MCX-vs-NSE-aware market-hours gate on entry-evaluation only (exit-checking on an open position untouched by design); also added a new weekly Friday square-off. No incorrect order placed - pure rate-limit/resource-waste impact, not a trading-correctness bug. **Backfilled into this journal 23 Sep** - missed at the time despite having its own incident doc from the night it happened; caught by a follow-up cross-package documentation audit. |
| 23 Sep | [[2026-09-23-bandhanbnk-dynamic-sl-over-sensitivity]] | Luxury BANDHANBNK CE entered on a climactic RSI-89 spike, then dynamic-SL triggered on a 7%+ premium move and reversed to a -15.6% net loss inside 70 seconds. 6-day data pull found the pattern was universal, not a one-off: all 7 real `TRAILING_SL_HIT` exits across Options/Luxury in the window were net losses, all 4 CE-side ones lost -15.6% to -17.3% - the dynamic mechanism never once protected real profit in this sample. Fixed (`LUXURY_DYNAMIC_SL_STEP_PCT_CE` 0.07->0.20, staged in `.env`) - also checked and rejected the entry-side "RSI-89 should have blocked this" hypothesis (no win-rate discrimination across 102 matched trades). No incorrect order placed - a risk-parameter tuning finding, not a code-correctness bug. |
| 23 Sep | [[2026-09-23-breakout-signal-single-snapshot-missed-signals-and-ws-walk-fix]] | `breakout_signal.py` only ever checked the single most-recently-completed candle at the instant a symbol's turn came up in the ~90-symbol/60s scan rotation, so an earlier qualifying candle that wasn't "current" at check time was gone for good. Two confirmed real-money hits the same day: BANDHANBNK entered at Rs4.22 instead of the missed 09:15 candle's Rs1.45 (-15.64%/-Rs2,376, TRAILING_SL_HIT), and IDFCFIRSTB entered at the 10:55-11:00 candle instead of the missed ~10:25 candle (~1.4% better underlying, ~35 min earlier) - entry Rs1.27 at RSI=79.92/ADX=69.68 (already flagged by the bot's own shadow reversal filter as over-extended), STOP_LOSS_HIT 20 min later for -Rs2,318.75/-19.69%. WS-walk fix written and locally validated (walks every unchecked candle since last check, with a stale-candle reversal guard) but **deliberately NOT deployed** per [[feedback-live-trading-safety]] - awaiting explicit user go-ahead since it changes live signal-detection logic and was written during market hours. |
| 23 Sep | [[2026-09-23-copper-mcx-orphaned-position-reconciliation-gap]] | Swing COPPER CE (entry Rs5.69, target Rs6.83, opened 22 Sep 13:45 IST) had its broker-side SL cancelled 90s after entry (unexplained), then the bot went down for 3h16m right after (14:03-17:19 IST, deliberate restart that didn't come back up for hours), leaving the position fully unmanaged. Worse: `reconcile_broker_positions()` ran clean (zero exceptions) on ~9 restarts across the next 22 hours but never once surfaced this position - no error, no skip-warning, nothing, unlike other unattributed positions which DO log a skip reason. Root cause of that specific silent gap is unresolved (Dhan's `get_positions()` REST response apparently excluded it even though the broker app UI still showed it open). User found it manually on the Dhan app 23 Sep and closed it by hand - **realized profit Rs416.25**, but the incident is about ~22h with nothing watching the stop-loss, not about this particular outcome. Also found in the same trace: a separate log-spam bug (the SL-cancelled warning re-fired every ~6-7s for 17 minutes instead of latching once). No code fix deployed yet - highest-priority follow-up is making reconciliation log explicitly even when it finds zero positions, so a gap like this is loud instead of silent. |

## Live monitoring sessions

Ad-hoc real-time monitoring windows (user request), checking `/health`,
all 4 packages' positions, new ERROR/CRITICAL log lines, and alert/trade
counts at a fixed interval - read-only, never restarts anything itself.
Logged here so a "how noisy is market open, really" baseline builds up
over time, separate from the dated `incidents/` writeups (which are for
specific notable failures, not routine confirmation that everything's
fine).

### 21 Sep 2026, 09:15-09:45 IST (market open, 30 min, 90s interval)

**Result: no crash, bot healthy throughout.** 20/20 health checks
returned 200; `/positions` never went unresponsive on any of the 4
packages.

- **Alerts**: 58 webhook alerts logged in the first ~25 minutes (all
  arrived by 09:38 IST, none after - normal, not a stall).
- **Trades**: 17 real trades closed by the end of the window - Luxury 7
  (2 wins, -Rs7,817.50), Options 3 (0 wins, -Rs6,315.00), Futures 5
  (2 wins, +Rs348.00), Swing 2 (0 wins, -Rs7,475.00 - MCX-corrected).
  Net: **-Rs21,259.50** for the session so far. End-of-window open
  positions: Luxury 2 (GODREJPROP, BLUESTARCO), Options/Futures/Swing
  flat.
- **Errors observed** (none fatal, all self-recovered):
  - **137x** `Exception at calling ltp/OHLC as {'status': 'failure', ...}`
    concentrated in a burst around 09:22-09:27 IST - Dhan REST
    congestion right at market open (a very common pattern, see
    [[dhan-rate-limit-every-call-site]]). The bot's own retry logic
    (`_get_option_ltp_once failed (attempt 1/3)... retrying in 1.5s`)
    and documented fallback (`live LTP unavailable - using last
    historical close ... instead of going blind`) both fired correctly -
    no trade decision was made blind, and `/feed-stats` confirmed the
    WebSocket feed itself was alive and receiving ticks throughout
    (`price_ticks_received: 2734`, `feed_errors: 0` as of this check) -
    the earlier same-morning [[market-feed-thread-death-on-429]]
    incident had already resolved by this window.
  - **17x** `swing_signals` "could not fetch Supertrend/regime state -
    keeping last cached value" for COPPER/ASHOKLEY/ANGELONE/ADANIPORTS/
    NATURALGAS - the same pre-existing, already-known
    `fetch_continuous_intraday returned no data` pattern seen earlier
    this session, not a new issue.
- **Nothing required a fix.** No restart, no code change, no manual
  intervention during this window.

## Open questions worth resolving (found while compiling this journal)

- **`LOSS_REPEAT_BLOCK_COUNT`**: the 11 Sep `.env` narrative describes
  tightening re-entry blocking to trigger after just ONE loss (count=1),
  but the currently deployed value for all 3 packages is **2**. Either
  it was deliberately raised back afterward without a matching comment
  update, or the comment is stale. Worth a direct check next time this
  gate is reviewed - not resolved in this pass.
- **Options/Futures' 12 Sep PROFIT_PROTECTION_THRESHOLD_RS/giveback
  raise was never separately backtested** against their own real trade
  history (only Luxury was) - the real PnL trajectory table above now
  has enough real days (15-18 Sep) under that config to do that
  comparison properly; not done in this pass.

---

## How to maintain this

1. **Every time a strategy/risk-config change goes live** (not every bug
   fix - use judgment: would this plausibly move real PnL?), add a row
   to the current date's section in "Deployment timeline" with: what
   changed, which strategy, the backtest evidence (link the `designs/`
   doc), and "Real PnL since: not yet" if too new to judge.
2. **Periodically** (e.g., every time this journal is opened for a new
   entry, or every ~1-2 weeks), re-pull `history/*_real_trades.log` from
   the droplet, extend the "Real PnL trajectory" table with the new
   days, and go back and fill in "Real PnL since" for any deployment
   that now has enough real days under it to judge fairly.
3. **New incidents** go in `incidents/` as their own file per this
   repo's existing convention (see README.md) - then add one row to the
   "Known issues" table here linking it.
4. Keep entries evidence-based, not narrated from memory - cite the real
   log line, the `.env` comment, or the backtest number. If a past entry
   turns out wrong, correct it and say what changed, don't silently
   delete history (same standing style rule as the rest of this repo).
5. **Cross-check against git log, not just memory of what was worked on
   this session** - added 23 Sep 2026 after an audit found 6 real
   commits (5 of them Swing-specific, clustered 13-15 Sep and 23 Sep)
   that never made it into this journal despite being real deploys, one
   with its own orphaned incident doc nobody had linked in. The failure
   mode: logging happens from what the CURRENT session remembers doing,
   which silently misses anything a different session/agent deployed.
   Concretely, before considering a deployment-related task done, or at
   least every ~1-2 weeks alongside step 2's real-PnL pull: run
   `git log --oneline --since=<last journal date> -- Options/ Futures/
   Luxury/ Swing/` in `traderBoy`, and for each commit not already
   referenced by hash anywhere in this journal, decide (same judgment
   call as step 1) whether it's worth a row. Pay particular attention to
   whichever package got the LEAST session time recently - that's
   exactly where a gap accumulates unnoticed, since nobody's actively
   looking at it to write about.
