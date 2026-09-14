# Backtest methodology: the real entry/exit checklist, and what's simplified

Two generations of backtest tooling exist in `traderBoy`. This file covers
both, but the **checklist below is the authoritative, currently-maintained
one** — built 14 Sep 2026 after finding that ad-hoc, per-script copies of
the exit ladder had silently drifted from the real code (EMA_CROSS_EXIT and
the liquidity guard went unmodeled across four backtests before this was
caught). Read the "Keeping this in sync" section before writing or trusting
any new backtest.

## Generation 2 (current): standalone `backtest_*.py` scripts + shared helpers

Applies to: `backtest_options_chartink_*.py`, `backtest_futures_chartink_*.py`
at the `traderBoy` repo root, each built per-CSV but sharing two real,
versioned helper modules (also at repo root, not copy-pasted per script):

- **`nifty_gap_down_backtest_helper.py`** — reconstructs the Nifty50 open
  gap-down/sharp-fall CE cool-off from real NIFTY 1-min candles, including
  the live code's one-shot-per-day caching quirk. CE-only, never applied to
  PE backtests (see the module's own docstring for why).
- **`exit_ladder_backtest_helper.py`** — the full exit-priority ladder,
  ported from `Options/trading_engine.py._exit_reason_for` directly, plus
  `build_full_minute_grid()` (the sim-clock fix, see below). This is the
  ONE place the exit ladder is implemented for backtesting — every script
  calls `evaluate_exit_reason(...)` rather than reimplementing its own
  copy inline, specifically so a future change to the live ladder only
  needs to be ported here once.

### Entry-condition checklist (verified against `Options/trading_engine.py` and `Options/dhan_client.py`, 14 Sep 2026)

| Condition | Modeled? | Notes |
|---|---|---|
| Trading-window/cutoff gate (`ENABLE_TRADING_TIME_LIMIT`/`ALLOWED_TRADING_TIME`) | Yes | Per-package value, read live |
| Nifty gap-down/sharp-fall CE cool-off | Yes (CE only) | `nifty_gap_down_backtest_helper.py`; per-package `ENABLE_GAP_DOWN_CE_DELAY` on/off, shared thresholds from `Options/config.py` |
| Capacity check (`MAX_LIVE_POSITIONS_CE`/`_PE`) | Yes | Whole batch skipped if full, not partially filled — matches live |
| Batch ranking (`rank_and_pick_top_stocks`, `SELECT_BOTTOM_N_STOCKS`) | Yes | `prefer_highest=True` for CE, `False` for PE — direction matters, verify per script |
| `MAX_DAILY_ENTRIES_PER_SYMBOL` | Yes | |
| RSI-gated loss re-entry block | Yes | Shared RSI thresholds always read from `Options/config.py`, never per-package |
| `LOSS_REPEAT_BLOCK` | Yes | |
| ATM strike resolution | Yes | Nearest CE/PE to entry-moment underlying price, nearest expiry |
| Funds check | **No** | Assumes unlimited buying power — a real capital-constrained day could see fewer live entries than the backtest shows |
| `cross_strategy_registry` (Options/Futures/Luxury mutual symbol lock) | **No** | Each backtest runs one strategy in isolation; can't know what another strategy claimed that same minute in real life |

### Exit-condition checklist (verified against `Options/trading_engine.py._exit_reason_for`, 14 Sep 2026 — priority order matters, this IS the order)

| # | Condition | Modeled? | Notes |
|---|---|---|---|
| 1 | `MAX_LOSS_HIT` | Yes | Gated by `ENABLE_MAX_LOSS_HIT_BEFORE_CUTOFF` (per-package; deployed `true` for all three as of 14 Sep 2026, contrary to code's own default-False comment — **always check the live `.env`, never trust a code comment's stated default**) |
| 2 | `TARGET_HIT` | Yes | Gated by `ENABLE_TARGET_EXIT` (per-package; deployed `true` for all three) |
| 3 | `PROFIT_PROTECTION_HIT` | Yes | `giveback_pct` passed explicitly per script (argv-overridable for PP sweeps, not baked into the shared config snapshot) |
| 4 | `TRAILING_SL_HIT`/`STOP_LOSS_HIT` | Yes | Dynamic-SL step logic, computed per script (unchanged, not part of the shared helper) |
| 5 | `SUPERTREND_EXIT` | Yes | Direction-aware: bearish crossover exits CE, bullish exits PE |
| 6 | `EMA_CROSS_EXIT` | **Yes, added 14 Sep 2026** | Was missing from every backtest before this date despite being live-enabled (`ENABLE_EMA_CROSS_EXIT=true` deployed for all three packages) — see the incident note below |
| 7 | `LIQUIDITY_GUARD_ZERO_VOLUME` | **Yes, added 14 Sep 2026** | Needs the OPTION's own 1-min **volume** series, not just close — `fetch_option_1min_cached` now caches `volumes` too; a cache file written before 14 Sep 2026 lacks this field and is correctly treated as stale (re-fetched, not silently run with missing data) |
| — | Broker-side SL-L (redundant safety net) | **No** | Can only tighten a real loss, never worsen it — deliberately low priority to model |

**Per-package flags are NOT shared from Options** — Futures and Luxury each
carry their own `FUTURES_`/`LUXURY_`-prefixed override for
`ENABLE_TARGET_EXIT`, `ENABLE_MAX_LOSS_HIT_BEFORE_CUTOFF`,
`ENABLE_SUPERTREND_EXIT`, `ENABLE_EMA_CROSS_EXIT`, `LIQUIDITY_GUARD_ENABLED`,
and `RISK_THRESHOLD_CUTOFF_TIME`. Only the numeric EMA-cross periods
(`EMA_CROSS_FAST_PERIOD`/`_SLOW_PERIOD`) and the liquidity-guard bar count
(`LIQUIDITY_GUARD_ZERO_VOLUME_BARS`) are genuinely shared, undeclared in
Futures/Luxury's own `config.py`. `exit_ladder_backtest_helper.ExitLadderConfig`
takes the correct package's config module as `pkg_config` for exactly this
reason — an earlier draft hardcoded `Options.config` for all three, which
would have silently used Options' flags for a Futures/Luxury backtest.

### Sim-clock trap: Dhan 5-min equity candles stop at the 15:10 bar

Found 10 Sep 2026 (original Generation-1 finding, `bt_common.py`), **still
true when re-verified live 14 Sep 2026**. `intraday_minute_data` at
`interval=5` for `NSE_EQ`/`EQUITY` returns bars only through **15:10** — no
15:15/15:20/15:25 bar, even though the equity session runs to 15:30. A
simulation clock built by extending 5-min underlying timestamps (`ts + 60*k
for k in range(5)`) inherits this gap and goes blind after ~15:14 every
day — end-of-day exits never get checked on the real 15:15-15:39 prints, and
a position still open at that point "jumps" straight to the next trading
day's first checked minute, which can misattribute an overnight/weekend move
as a single-tick MAX_LOSS_HIT that never really happened that way.

This was **re-discovered as a live bug in every Generation-2 script this
session** (four backtests already delivered before this was caught) when
`nifty_gap_down_backtest_helper.py`/`exit_ladder_backtest_helper.py` were
being added and this file was re-read as part of that work — the standalone
scripts had independently reinvented the exact same flawed `master_ts`
construction Generation 1 had already documented, because nobody checked
this file before writing them.

**Fix, now in `exit_ladder_backtest_helper.build_full_minute_grid()`**:
build the sim clock as a full 1-minute grid `09:15..15:29` for every trading
day present in the data, independent of what the 5-min series itself
contains. Every Generation-2 script must call this for `master_ts`, not
derive it from candle timestamps. Verified effect on the "03 Range
Breakout.csv" backtest: trade count went 60→81 and win rate 83.3%→70.4%
(more real late-day entries/exits now visible, a more realistic risk
picture), while aggregate net P&L barely moved (+₹121,027→+₹121,768) — the
sim-clock trap doesn't necessarily change your headline number, but it can
substantially change trade count and win-rate, which matter for judging
whether a config's risk profile is being read correctly.

### NOT modeled (Generation 2, disclosed standing gaps)

- Broker-side SL-L order (can only improve on the modeled exit, never
  worsen it — low priority).
- `cross_strategy_registry` (Options/Futures/Luxury mutual symbol lock) —
  needs a joint multi-strategy backtest sharing one claim/release state
  across all three packages' CSVs simultaneously, a real structural effort
  beyond any single-strategy backtest.
- Funds check (assumes unlimited capital).
- REST-fallback/staleness timing during entry capture is idealized (see
  Generation 1's note below — applies equally here).
- 1-minute option OHLC is the finest resolution available — no tick data,
  so very thin/illiquid contracts can disagree with real fills by a few
  paise.

## Keeping this in sync (the actual discipline, not just a list)

The EMA_CROSS_EXIT/liquidity-guard gap and the reinvented sim-clock bug both
happened the same way: a backtest script was built by copying an *existing
backtest script*, not by re-reading the live trading code or this file.
Copying compounds errors instead of catching them — Generation 2 shipped a
bug Generation 1 had already found and documented.

**Before trusting a NEW backtest's results, or before building one for a
CSV that hasn't been tested before:**
1. Re-read this file's two checklists above. If either the entry or exit
   checklist doesn't match what the new script actually does, the script is
   wrong, not the checklist — fix the script.
2. If it's been a while (a live deploy could have changed a flag), grep
   `Options/trading_engine.py._exit_reason_for` and the entry-gate call
   chain in `option_main.py`/`trading_engine.py` directly, and diff what
   you find against the tables above. Update this file if anything changed.
3. Check the ACTUAL deployed `.env` for every flag referenced above, not
   just the code's own default — this session found three separate flags
   (`ENABLE_TARGET_EXIT`, `ENABLE_EMA_CROSS_EXIT`,
   `ENABLE_MAX_LOSS_HIT_BEFORE_CUTOFF`) where the live `.env` override
   contradicts what the code comment says the default/intent is.
4. New backtest scripts should IMPORT `nifty_gap_down_backtest_helper.py`
   and `exit_ladder_backtest_helper.py` from the `traderBoy` repo root, not
   reimplement any part of either. If a script needs something the shared
   helpers don't do yet (a new live exit condition, a new entry gate),
   add it to the shared helper and update this file's tables in the same
   pass — never add it inline to just the one script.
5. When you find a real gap (something live-enabled that no backtest
   modeled, or a data characteristic like the sim-clock trap), write it up
   as its own dated note here (like the two sections above), not just a
   silent code fix — the NEXT gap will be found the same way this one was:
   someone actually reading this file before writing new code.

## Generation 1 (original): `bt_common.py` + per-CSV runner scripts

Applies to: any CE ATM options backtest run against a Chartink CSV export
using `traderBoy/bt_common.py` (repo-root, versioned as of 30 Aug 2026 —
commit 20498cf) plus a per-CSV runner script. Superseded in practice by
Generation 2 for every backtest since 12 Sep 2026, but the file/pattern
still exists in the repo and the findings below remain true of it.

### What's faithfully real, not simulated

- **Exit decisions run through the actual production functions** —
  `Options.trading_engine._exit_reason_for` and `Options.position_store.
  Position` are imported and called directly, not reimplemented. Target,
  stop-loss, dynamic-SL, Supertrend-exit, MAX_LOSS_HIT, and PROFIT_
  PROTECTION_HIT all use the real logic.
- **Ranking and capacity are enforced identically to live**:
  `MAX_LIVE_POSITIONS_CE` from the live `Options/config.py` caps concurrent
  positions (`if len(open_positions) >= MAX_POS: break`), and candidate
  selection (`pick_candidates`) replicates `rank_and_pick_top_stocks()`
  exactly — including respecting `SELECT_BOTTOM_N_STOCKS`.
- **All tunable values are read live from `Options/config.py` at import
  time**.
- **Historical option and underlying price data is fetched live from Dhan**
  (`intraday_minute_data`) for the exact backtest date range.

### What's simplified or approximate

- **Entry timing assumes the alert is actionable the instant it appears in
  the CSV** — no webhook-delivery delay modeled.
- **1-minute option OHLC candles are the finest resolution available**.
- **REST-fallback/staleness timing during entry capture is idealized**.

### Standard workflow for backtesting a new Chartink CSV (Generation 1)

1. Read the CSV, confirm its date range and row format matches the expected
   `"Date","Symbol","Marketcapname","Sector"` schema with
   `"DD-MM-YYYY H:MM am/pm"` timestamps.
2. Copy an existing per-CSV runner script, change only `CSV_PATH`,
   `DAILY_FROM`/`DAILY_TO`, the output JSON filename, and the summary title.
3. Run in the background — larger CSVs take proportionally longer.
4. Move the resulting `*_results.json` out of the repo root into scratch
   space immediately.
5. For day-wise and trade-wise reports: recompute P&L directly from each
   trade's `entry_price`/`exit_price`/`quantity` in the JSON rather than
   trusting only the script's own printed summary.
