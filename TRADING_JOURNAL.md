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

### 21 Sep 2026

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| Breakout-signal scanner deployed (flag-enabled, feature-flagged) | Futures | [[futures-breakout-signal-gated-live-full-real-gates]] - 8-trade sample, 87.5% win rate, +Rs13,359.10 vs real Futures' -Rs11,943.55 over the backtest window (same caveats: tiny sample, 3 variables changed at once) | Not yet - deployed today, no real trades under it yet |
| Breakout-signal thresholds loosened to clearance=0.3%/body=0.5%/relvol=1.2x (the 27-combo sweep's #1) | Luxury | [[luxury-breakout-detection-parameter-sweep]] - 72 signals/59 entered/93.2% win rate/+Rs175,346.60 vs real Luxury's -Rs17,909.35 over the same 14-day window | Not yet |
| Risk config raised: MAX_LOSS 5,500/3,100 (CE), PROFIT_PROTECTION_THRESHOLD 5,000/2,500, giveback 8% | Futures | Mirrors the same-day Luxury raise, applied without its own backtest at request time; backtested afterward - [[futures-updated-risk-config-15day-backtest]] (near-breakeven -Rs714.75 vs real -Rs11,943.55, but that backtest was later found to be missing 3 real gates - see the full-real-gates doc above for the corrected number) | Not yet |
| Risk config raised: MAX_LOSS 5,500/3,100 CE / 4,500/2,600 PE, PROFIT_PROTECTION_THRESHOLD 5,000/2,500, giveback 8% | Luxury | Same raise, Luxury side - see [[luxury-signal-gated-live-simulation]]'s "Config raise" section (+Rs73,999.10 delta on the same 18-trade signal-gated sample) | Not yet |
| **Breakout-signal scanner made the SOLE real entry path** (not an additional gate) - `_handle_chartink_webhook` no longer calls `enter_positions_for_stocks` at all for any of the 3; a raw alert only records into the watchlist now. Options widened from PE-only to full CE+PE parity, `OPTIONS_BREAKOUT_SIGNAL_ENABLED` defaulted to true. Same commit fixed the WS market-feed thread-death bug for real (watchdog + exponential backoff, see [[market-feed-thread-death-on-429]]). | Options, Luxury, Futures | Deployed via a separate, parallel Claude Code session on the user's own instruction (commit `b889608`, hotfixed by `db07226` for an accidental `alert_bucket` import break) - explicitly **not backtested as a sole-gate config**, only ever backtested as an additional layer (the two rows above, and [[futures-breakout-signal-gated-live-full-real-gates]]). | **Measured same-day**: 19 trades opened before the 10:07 IST cutover, **0 trades opened in the 2+ hours after** (through 12:26 IST) - the normal alert-driven path's removal, combined with the scanner firing only once all session (see 21 Sep monitoring log above), has visibly collapsed trade frequency. Worth a deliberate decision on whether this is the intended tradeoff before relying on it further. |
| **Swing NSE volume-floor gate lowered 1.2x -> 0.6x** (`SWING_NSE_VOLUME_FLOOR_RATIO_MIN`) - straight off today's real observation that ADANIPORTS and ANGELONE kept qualifying on price/trend (a genuine Supertrend+regime entry signal, repeating every ~7s) but got blocked purely on volume ratio (~0.60-0.69x). MCX's own floor (COPPER/CRUDEOIL/NATURALGAS) left untouched at 1.2x. | Swing | No backtest run before deploying - straight config change on direct user instruction. **Important caveat**: this exact gate was originally promoted from shadow-mode to a live block specifically because of a real ANGELONE 29 SEP 295 PUT loss tied to a low-volume entry (18 Sep 2026) - halving the floor meaningfully reduces that same protection for every NSE watchlist symbol, not just today's two. Worth backtesting or watching closely rather than assuming today's two rejected candidates were false positives. | Not yet - by the time this deployed (~14:14 IST), both ADANIPORTS' and ANGELONE's triggering entry signals had already lapsed (last events 10:00 and 12:30 IST respectively), so this hasn't yet been observed taking a real trade it wouldn't have before. |

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

### 20 Sep 2026

| Change | Strategy | Backtest evidence | Real PnL since |
|---|---|---|---|
| Breakout-signal live entry trigger deployed (flag-enabled), separate CE/PE watchlists with daily refresh | Luxury | [[breakout-scanner-vs-real-pnl]] (original recalibration work) + [[luxury-signal-gated-live-simulation]] (real-gate replication, +Rs32,649.10 on the real exit stack before the config raise above) | 0 real trades attributed to the signal path through 21 Sep (feature just launched; normal alert-driven trading continued in parallel and is what's reflected in the 17/18 Sep rows above) |

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
| Opening-burst extra CE capacity slot (+1 slot, 09:15-10:00) | Options/Futures/Luxury | [[opening-burst-slot-and-sl-target-sensitivity]] | Reflected in the 17/18 Sep rows above (still net negative for Options/Futures those days) |
| Global liquid-contract-resolution gate (`get_liquid_atm_option`) | All 4 (Options/Futures/Luxury/Swing) | [[liquid-contract-resolution]] - built after the ATHERENERG broker-stop-rejection incident | Reflected in every row from 17 Sep onward |

### 14 Sep 2026

- Nifty gap-down max-delay ceiling lowered 120→30 minutes
  (`GAP_DOWN_MAX_DELAY_MINUTES`) - shared across Options/Futures/Luxury.
- PE-specific MAX_LOSS_PER_TRADE_RS caps introduced (tighter than CE) for
  Options/Futures/Luxury.
- Swing v2 entry strategy version set explicitly (`v2`).
- `EXECUTOR_MAX_WORKERS` held at 5 pending a live-hours observation
  window - see [[executor-sizing-ws-storm-on-closed-market]] (incident,
  13 Sep) for why this is deliberately not yet raised to 10.

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
| 21 Sep | [[local-backtest-dhan-session-collision]] | A local backtest script's own Dhan re-authentication invalidated the live bot's session, forcing one real, unplanned `dhanboy.service` restart. No open positions affected (confirmed via `/health`/`/positions` before and after). Fixed for future local scripts: use a user-supplied hand-off `access_token` instead of a competing local `pin_totp` login; defer bulk historical-data pulls to after market close (separate shared-rate-limit issue, not fixed by the access-token change). |

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
