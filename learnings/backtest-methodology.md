# Backtest methodology: what `bt_common.py` actually replicates, and its real limits

Applies to: any CE ATM options backtest run against a Chartink CSV export
using `traderBoy/bt_common.py` (repo-root, versioned as of 30 Aug 2026 —
commit 20498cf) plus a per-CSV runner script.

## What's faithfully real, not simulated

- **Exit decisions run through the actual production functions** —
  `Options.trading_engine._exit_reason_for` and `Options.position_store.
  Position` are imported and called directly, not reimplemented. Target,
  stop-loss, dynamic-SL, Supertrend-exit, MAX_LOSS_HIT, and PROFIT_
  PROTECTION_HIT all use the real logic.
- **Ranking and capacity are enforced identically to live**:
  `MAX_LIVE_POSITIONS_CE` from the live `Options/config.py` caps concurrent
  positions (`if len(open_positions) >= MAX_POS: break`), and candidate
  selection (`pick_candidates`) replicates `rank_and_pick_top_stocks()`
  exactly — including respecting `SELECT_BOTTOM_N_STOCKS`. Verify this
  actually bound something in a given run by sweeping entry/exit timestamps
  for periods where concurrency hit the cap — if it never did, the capacity
  setting wasn't actually tested by that particular CSV.
- **All tunable values are read live from `Options/config.py` at import
  time** — re-running an existing backtest script after a config deploy
  automatically reflects the new values, no script changes needed. This is
  also why re-running the *same* CSV after a config change is a valid, cheap
  way to measure that change's effect (see `capacity-and-ranking.md`'s
  CE=2→4 comparison).
- **Historical option and underlying price data is fetched live from Dhan**
  (`intraday_minute_data`) for the exact backtest date range — not
  synthetic or interpolated.

## What's simplified or approximate

- **Entry timing assumes the alert is actionable the instant it appears in
  the CSV** — it does not model any live webhook-delivery delay, and
  (per `capacity-and-ranking.md`) the CSV's timestamps may not even
  correspond to when a live webhook alert for that symbol actually arrived.
  A backtest answers "what would this strategy logic do against this alert
  sequence," not "what actually happened live on this date."
- **1-minute option OHLC candles are the finest resolution available** —
  there is no tick-level historical data. For thin/illiquid contracts this
  data can disagree with real fill prices by a few paise (see
  `exit-mechanics.md`'s SAGILITY finding) — trade-wise precision on very
  thin names should be read with that caveat, though day-level and
  multi-trade aggregate P&L is generally trustworthy.
- **REST-fallback/staleness timing during entry capture is idealized** —
  `_capture_supertrend_entry_candle`'s cache-refresh throttling isn't
  modeled with real-world timing jitter; the backtest assumes the "latest
  closed candle as of this instant" reading is always accurate, which is a
  reasonable approximation but not a perfect one (see the Supertrend-lag
  analysis in `exit-mechanics.md` for how small real-world lag actually is).

## Standard workflow for backtesting a new Chartink CSV

1. Read the CSV, confirm its date range and row format matches the expected
   `"Date","Symbol","Marketcapname","Sector"` schema with
   `"DD-MM-YYYY H:MM am/pm"` timestamps.
2. Copy an existing per-CSV runner script (e.g. the most recent one), change
   only `CSV_PATH`, `DAILY_FROM`/`DAILY_TO`, the output JSON filename, and
   the summary title string.
3. Run in the background — larger CSVs (more unique symbols) take
   proportionally longer since each symbol needs its own Dhan REST fetch;
   don't assume a quiet output log means it's stuck (stdout is
   block-buffered when redirected to a file — nothing appears until the
   process finishes or flushes).
4. Move the resulting `*_results.json` out of the repo root into scratch
   space immediately — the script writes it to cwd, which pollutes the git
   repo if left there. Verify `git status --short` is clean afterward.
5. For day-wise and trade-wise reports: recompute P&L directly from each
   trade's `entry_price`/`exit_price`/`quantity` in the JSON
   (`(exit_price - entry_price) * quantity` for a long CE) rather than
   trusting only the script's own printed summary — this is what lets you
   regroup by day, by symbol, or by exit reason after the fact without
   re-running the backtest.

## Sim-clock trap: Dhan 5-min equity candles stop at the 15:10 bar

Found 10 Sep 2026 (Futures EMA-cross backtest). `intraday_minute_data` at
`interval=5` for `NSE_EQ`/`EQUITY` returns bars only through **15:10**
(72 bars, 09:15→15:10) — no 15:15/15:20/15:25 5-min bar, even though the
equity session runs to 15:30. Any backtest that drives its simulation
clock off the 5-min underlying timestamps goes blind after ~15:14 each day
and skips the entire 15:15–15:30 window:

- the **15:15 / Friday 15:20 square-off never fires** — a position entered
  in the 14:00–15:28 zone-2 window "carries" overnight when it would
  really have been force-closed;
- **end-of-day PP / MAX_LOSS / trailing-SL are never checked** on the real
  15:15–15:39 prints (the option 1-min series *does* run to ~15:39);
- the first observed price becomes next morning's open, so a benign Friday
  close + weekend gap gets booked as a huge MAX_LOSS_HIT that never
  happened (real case: HINDALCO −₹5,180 / VEDL −₹4,715, both flat-to-
  profitable at Friday's real close).

**Fix, mandatory for every new backtest script:** build the sim clock as a
full 1-minute grid `09:15..15:29` for every trading day with data, not
from the 5-min timestamps. `bt_common.py` predates this finding — check
its `master_ts` construction before trusting late-day / carried-position
P&L from an older run.
