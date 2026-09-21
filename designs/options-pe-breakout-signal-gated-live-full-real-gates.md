# Options PE + breakout-signal gate, full real-gate replication (21 Sep 2026, user request)

Status: **BUILT, not yet deployed** (21 Sep 2026, same day as the
backtest below) - the live version was built as a feature-flagged,
PE-only wiring of the unchanged `breakout_signal.py` module into
Options, alongside the [[2026-09-21-market-feed-thread-death-on-429]]
backoff fix, for a single combined deploy per the user's own
instruction. See "Live build" section near the bottom for exactly what
was wired and how to arm it.

Follow-up to the user flagging that Options' real PE PnL over the
recent window looked weak. Same treatment already given to
[[luxury-signal-gated-live-simulation]] and
[[futures-breakout-signal-gated-live-full-real-gates]]: replace Options'
real alert-driven PE entries with the SAME breakout-screener signal gate
currently deployed live on Luxury (clearance=0.3%, body=0.5%,
relvol=1.2x - the sweep's #1 combo, see
[[luxury-breakout-detection-parameter-sweep]]), with every one of
Options' own real production PE entry/exit gates applied - liquidity
guard, gap-down CE delay (structurally a no-op for PE, see below), and
cross-strategy lock.

New script: `backtest_options_pe_breakout_signal_gated_live.py`. Read
`Options/trading_engine.py`'s real `_process_one_entry`/`_exit_reason_for`
and `Options/option_main.py`'s webhook handler directly (not assumed/
copied from an older script) to get the exact gate order and logic.
Options is architecturally different from Futures/Luxury here: it has NO
per-package env-var prefix - its own `config.py` IS the shared module
every other package's numeric thresholds (liquidity guard, RSI-reentry,
EMA-cross periods, etc.) already read from - so this script needed only
ONE config import (`ocfg = Options.config`), not a package config plus a
shared-config pair.

## Gates modeled

| Gate | Modeled | Notes |
|---|---|---|
| Square-off cutoff for new entries | Yes | `ENABLE_SQUARE_OFF=false` live - a no-op every day except Friday (`FRIDAY_SQUARE_OFF_TIME=15:20`) |
| Gap-down/sharp-fall Nifty CE cool-off | **N/A, verified structural** | `option_main.py` gates this behind `option_type == "CE"` explicitly (line 268) - it can never fire for a PE signal. Not modeled as a live check (would be dead code that's always False) |
| Daily re-entry cap | Yes | `MAX_DAILY_ENTRIES_PER_SYMBOL=3` |
| RSI-gated loss re-entry block | Yes | Only evaluated after a same-day `MAX_LOSS_HIT` |
| Loss-repeat block | Yes | `LOSS_REPEAT_BLOCK_COUNT=2` (live value) |
| Loss-reentry trend-strength check | Yes | ADX>=20 or ER>=0.3, gates the 1-prior-loss case |
| Volume-floor entry gate | Yes | `VOLUME_FLOOR_RATIO_MIN=1.2` |
| Cross-strategy claim / broker-wide open-position check | Yes (interval-overlap proxy) | Against Futures/Luxury/Swing's own REAL trade history + this sim's own open Options PE positions |
| Capacity | Yes | `MAX_LIVE_POSITIONS_PE=3` live, no burst slot (burst is CE-only, `BURST_EXTRA_SLOTS_CE`) |
| Liquid-contract resolution (`get_liquid_atm_option`) | Yes | Zero-volume-bar streak + prior-session-volume check, with strike substitution |
| Funds check | **No** | Same disclosed gap as every other backtest in this line of work - fails open, matching production's own fail-open behavior |

## Exit ladder

Options' own real deployed `TARGET_PCT=0.20`/`STOP_LOSS_PCT=0.16` and the
current live risk caps, checked in `_exit_reason_for`'s own priority
order: `MAX_LOSS_HIT` (PE caps Rs3,500/1,600 before/after 11:30,
`ENABLE_MAX_LOSS_HIT_BEFORE_CUTOFF=true` live so this is unconditional
all day) -> `TARGET_HIT` -> `PROFIT_PROTECTION_HIT` (threshold
Rs2,000/1,500, `GIVEBACK_PCT=0.03`) -> `TRAILING_SL_HIT`/`STOP_LOSS_HIT`
(`ENABLE_TRAILING_SL=false`, only the dynamic-step mechanism applies,
`DYNAMIC_SL_STEP_PCT_PE=0.09`) -> `SUPERTREND_EXIT` -> `EMA_CROSS_EXIT`
(`ENABLE_EMA_CROSS_EXIT=true` live) -> `LIQUIDITY_GUARD_ZERO_VOLUME`.
`ENABLE_SQUARE_OFF=false` live (same as Luxury, unlike Futures) - a PE
position is NOT force-closed daily at 15:15; the exit walk continues
across real calendar days until something fires, Friday's 15:20
square-off, or this backtest's own data horizon (today) is reached.

## Result

Window: 2026-08-31 to 2026-09-18 (14 trading days - the full real-trade
history currently logged locally; no gap in that window except the
already-known missing 12/13/14 Sep dates also absent from every other
backtest in this series).

|                          | Trades | Wins | Win Rate |     Total PnL |
|--------------------------|-------:|-----:|---------:|---------------:|
| REAL Options PE          |     50 |   17 |    34.0% |  -Rs19,918.00 |
| **SIGNAL-GATED SIMULATION** |  **17** | **13** | **76.5%** | **+Rs21,254.87** |
| **Delta**                |        |      |          | **+Rs41,172.87** |

Raw breakout signals found across Options-PE-alerted symbol-days: **37**.
Of those, **20 were filtered out**: 14 by capacity (`MAX_LIVE_POSITIONS_
PE=3` filled), 6 by the volume-floor gate. **17 entered, 13 won.** No
signals were blocked by the daily re-entry cap, RSI-loss-reentry block,
loss-repeat block, cross-strategy claim, or liquid-contract resolution in
this sample.

### Day-wise

| Day | Real Trades | Real PnL | Sim Trades | Sim PnL | Delta |
|---|---:|---:|---:|---:|---:|
| 31 Aug | 5 | -6,762.50 | 3 | 4,021.25 | +10,783.75 |
| 15 Sep | 21 | 2,639.50 | 5 | 5,533.50 | +2,894.00 |
| 16 Sep | 11 | -6,052.50 | 2 | 3,814.75 | +9,867.25 |
| 17 Sep | 8 | -3,918.75 | 1 | 2,015.00 | +5,933.75 |
| 18 Sep | 5 | -5,823.75 | 6 | 5,870.37 | +11,694.12 |
| **Total** | **50** | **-19,918.00** | **17** | **21,254.87** | **+41,172.87** |

No signals at all on 1/2/3/4/7/8/9/10/11 Sep - not a config effect, the
breakout screener simply found no qualifying PE setup among that day's
Options-alerted symbols on those dates (real Options PE still traded 0
times those days too, in this specific sample - the strategy was quiet
for both real and simulated on that whole stretch).

### Trade-wise (17 trades)

| Day | Symbol | Signal Time | Entry | Exit | Exit Reason | Qty | PnL |
|---|---|---|---:|---:|---|---:|---:|
| 31 Aug | LTF | 09:15 | 9.10 | 9.00 | LIQUIDITY_GUARD_ZERO_VOLUME | 2,250 | -225.00 |
| 31 Aug | ADANIPORTS | 09:15 | 25.75 | 30.90 | TARGET_HIT | 475 | +2,446.25 |
| 31 Aug | NTPC | 09:15 | 5.65 | 6.85 | TARGET_HIT | 1,500 | +1,800.00 |
| 15 Sep | MAZDOCK | 09:15 | 42.30 | 50.76 | TARGET_HIT | 225 | +1,903.50 |
| 15 Sep | VEDL | 09:15 | 5.70 | 6.84 | TARGET_HIT | 1,150 | +1,311.00 |
| 15 Sep | JSWENERGY | 09:15 | 4.80 | 4.80 | LIQUIDITY_GUARD_ZERO_VOLUME | 1,075 | 0.00 |
| 15 Sep | KAYNES | 11:05 | 115.00 | 115.00 | LIQUIDITY_GUARD_ZERO_VOLUME | 150 | 0.00 |
| 15 Sep | INDIGO | 14:15 | 77.30 | 92.76 | TARGET_HIT | 150 | +2,319.00 |
| 16 Sep | FORTIS | 09:15 | 14.80 | 17.76 | TARGET_HIT | 775 | +2,294.00 |
| 16 Sep | RVNL | 09:20 | 5.42 | 6.21 | PROFIT_PROTECTION_HIT | 1,925 | +1,520.75 |
| 17 Sep | PNBHOUSING | 13:30 | 15.50 | 18.60 | TARGET_HIT | 650 | +2,015.00 |
| 18 Sep | INFY | 09:15 | 9.70 | 11.64 | TARGET_HIT | 400 | +776.00 |
| 18 Sep | KPITTECH | 09:15 | 10.50 | 8.92 | **TRAILING_SL_HIT** | 775 | **-1,220.63** |
| 18 Sep | TCS | 09:15 | 30.00 | 36.00 | TARGET_HIT | 225 | +1,350.00 |
| 18 Sep | HCLTECH | 09:15 | 17.75 | 21.30 | TARGET_HIT | 400 | +1,420.00 |
| 18 Sep | PERSISTENT | 09:15 | 85.80 | 102.96 | TARGET_HIT | 125 | +2,145.00 |
| 18 Sep | NESTLEIND | 12:55 | 14.00 | 16.80 | TARGET_HIT | 500 | +1,400.00 |

Exit reasons: **TARGET_HIT 12**, LIQUIDITY_GUARD_ZERO_VOLUME 3 (2 of
which closed at exactly pnl=0.00 - the guard caught a contract that
never moved after entry, not a real loss), PROFIT_PROTECTION_HIT 1,
TRAILING_SL_HIT 1 (the only real loss, KPITTECH -Rs1,220.63).

## Why this differs so much from real Options PE performance

Same mechanism seen in every other backtest in this series: the
breakout-screener signal is a much smaller, more selective source (37
raw signals across 14 days vs whatever fed the 50 real PE trades) that
already requires a confirmed, volume-backed breakdown before an entry is
even attempted. `capacity_full` did most of the filtering here (14 of 20
skips) - `MAX_LIVE_POSITIONS_PE=3` filled up quickly once several
qualifying signals clustered at the same 09:15 open across multiple
days (31 Aug, 15 Sep, 18 Sep all show 3+ raw signals at the open),
meaning several genuine signals were never even attempted purely on
capacity, not a quality filter. A higher PE cap would very likely change
this result - not tested here.

## Caveats

- **17 trades over 14 days** - directional at best, same as every other
  finding in this line of work. A single trade (KPITTECH's -Rs1,220.63)
  is the only real loss; 3 more trades closed at exactly pnl=0.00
  (illiquidity catches, not genuine losses) - the result is dominated by
  12 clean TARGET_HIT wins.
- **Zero signals for 9 of the 14 days** (1-11 Sep) - the breakout
  screener found nothing on Options' own PE-alerted symbol set that
  stretch. Worth checking against a longer window before trusting the
  headline win rate.
- Funds check not modeled (fails open, matching production's own
  fail-open behavior - not a new simplification).
- Cross-strategy lock is the same interval-based proxy every script in
  this series uses (real trade history of the OTHER strategies), not the
  real momentary `cross_strategy_registry` claim.
- Capacity (`MAX_LIVE_POSITIONS_PE=3`) was the single biggest filter
  (14/20 skips) - unlike the Futures/Luxury versions of this same
  hypothesis where capacity barely bound. A capacity-relaxed re-run would
  answer whether the missed 14 signals were similarly high-quality or not
  - not done here, flagged as the natural next step if this is pursued
  further.
- Was a HYPOTHETICAL when this backtest was written - now BUILT (see
  "Live build" below) but still **not deployed/armed**. Nothing has
  changed any live config yet.

## Live build (21 Sep 2026, same day, user request)

Wired `breakout_signal.py` into Options **unchanged** - no scanner-logic
changes, exactly the same module Luxury/Futures already import. Three
touch points, all in the `dhanBoy` branch:

1. **`Options/config.py`** - new `BREAKOUT_*` block, same shape as
   Luxury/Futures' own, env-prefixed `OPTIONS_BREAKOUT_*`. Defaults match
   this backtest's own thresholds exactly (clearance=0.3%, body=0.5%,
   relvol=1.2x, `OPTIONS_BREAKOUT_MIN_AVG_DAILY_VOLUME`=500000, etc. - see
   the file itself for the full list, all overridable via env var).
   **`BREAKOUT_SIGNAL_ENABLED` (`OPTIONS_BREAKOUT_SIGNAL_ENABLED`)
   defaults `false`** - deliberately different from Luxury/Futures' own
   flags (both default `true`, but those were already-reviewed-live
   features when added; this is new/unarmed until the user explicitly
   flips it on post-deploy, per [[feedback-live-trading-safety]]).
2. **`Options/option_main.py`** - `_breakout_entry_fn` (byte-for-byte the
   same pre-entry-gate wrapper pattern as Luxury's own), a new
   `signal_scanner_loop("Options", config, _breakout_entry_fn)` task
   started unconditionally in `lifespan` (matches breakout_signal.py's
   own design - the loop's daily watchlist refresh housekeeping needs to
   run regardless of whether scanning itself is flag-enabled), and a new
   read-only `GET /breakout-signal` status endpoint mirroring Luxury's.
3. **PE-only scoping, the one real design choice here**: `breakout_
   signal.record_alert(...)` is called ONLY from inside `_handle_chartink_
   webhook` when `option_type == "PE"` (i.e. only from `/chartink/
   webhook-sell`) - never from the CE path (`/chartink/webhook`). Since
   nothing ever feeds the CE watchlist, it stays permanently empty and
   the generic per-package module's own CE-side machinery is structurally
   inert for Options - a deliberate scope match to what was actually
   backtested (PE only), without needing any CE/PE conditional inside
   `breakout_signal.py` itself.

**Verified before trusting it**: full `tests/` suite run before and
after this change (193 failed/182 passed both times, identical - every
failure is pre-existing pinned-config-value staleness, confirmed by
diffing against a `git stash`ed baseline run) - zero regressions. Also
confirmed via direct import (`Options.option_main`, and the combined
`main.app`) that `/breakout-signal`, `/chartink/webhook-sell`, and every
other existing route still resolve correctly post-change.

**To arm it once deployed**: set `OPTIONS_BREAKOUT_SIGNAL_ENABLED=true`
in both the local and droplet `.env` (gitignored, scp separately per
[[project-dhanboy-deployment]]), then restart - same checklist as any
other live-affecting config change. Until then, the code ships inert:
the scanner loop runs (harmless daily housekeeping only) but never
scans or enters anything.

## Bot resource usage (checked same session, user request)

Live droplet (`ubuntu-s-1vcpu-512mb-10gb-blr1`, 961Mi RAM, 1GB swap),
checked directly via SSH while this backtest ran (backtest itself runs
locally, makes read-only Dhan REST calls, no load on the droplet):

- `dhanboy.service` memory: **206MB RSS (21.7% of 961Mi)**, cgroup peak
  244MB since last restart (~20 min uptime at check time).
- CPU: **7.6%** on the single vCPU, system load average 0.33/0.15/0.11 -
  effectively idle.
- Swap: 50MB/1024MB used (4.9%) - healthy headroom, not under memory
  pressure.
- Disk: 8.3G/24G used (36%).
- **No OOM kills found** in the full available journal history (back to
  2 Sep 2026, ~19 days, 2.2GB of retained logs) - `journalctl -k` grepped
  for "out of memory"/"oom-kill"/"killed process", zero matches.
- Frequent `Main process exited, code=exited, status=143` entries (clean
  SIGTERM, i.e. deliberate restarts - deploys, `.env` changes, the daily
  08:00 IST token-refresh timer) throughout 14-17 Sep, consistent with
  an active dev/deploy period, not crashes - no `status=137`
  (SIGKILL/OOM) entries anywhere in the same window.

**Conclusion: the live bot has comfortable memory/CPU headroom today.**
Nothing about this specific PE breakout-signal hypothesis was deployed
or would by itself change the live bot's resource profile - it's a
local-only backtest. If this WERE deployed live per the "if ever
deployed" note above, the added resource cost would be one more
`signal_scanner_loop` (one more `intraday_minute_data`/
`historical_daily_data` REST poll per not-yet-signaled PE-alerted symbol
per `BREAKOUT_SCAN_INTERVAL_SECONDS`, same shape as Luxury's own already-
live loop) - negligible against the current 21% memory / 8% CPU
utilization, based on Luxury's own loop running today with no measurable
resource complaint.
