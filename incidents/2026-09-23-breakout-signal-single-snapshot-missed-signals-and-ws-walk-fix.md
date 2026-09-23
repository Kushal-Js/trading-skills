# Breakout-signal scanner was missing real signals via a single-snapshot evaluation bug - WS-walk fix (23 Sep 2026)

## What was found

Replayed today's (23 Sep 2026) real UniverseDispatcher PE watchlist (60 symbols, Simply Bear
alerts) and CE watchlist (46 symbols, Krishvi/Range Breakout alerts) against the exact 7-check
`breakout_signal.py` logic, using the live-deployed loosened thresholds (clearance=0.15%,
body>=0.5%, relvol>=0.8x, avg_daily_vol>=300k, from Luxury's cfg as the dispatcher's primary),
via a standalone local script using a user-supplied out-of-band Dhan token (never persisted) -
not the droplet, to avoid rate-limit contention with the live bot during market hours.

- **PE: 2 of 60 symbols would have confirmed** (PERSISTENT 09:15, SUZLON 09:25) - both would
  have been strong winners (best intraday option premium moves +116%/+115%; Luxury 10%/3%
  ladder both TARGET_HIT; Futures 25%/16% ladder SUZLON TARGET_HIT, PERSISTENT STOP_LOSS_HIT).
  Neither was caught live - the real PE watchlist showed `signaled: {}` all day.
- **CE: 9 of 46 symbols would have confirmed** (BANDHANBNK, HINDZINC, IDFCFIRSTB, LICHSGFIN,
  MAHABANK, MCX, MOTILALOFS, POLICYBZR, SWIGGY) - best-case option moves ranged 24%-261%.
  Two of these (BANDHANBNK, MOTILALOFS) DID signal live - see the root cause below for why
  even those two were materially worse than they should have been.

## Root cause (confirmed via real logs, not inferred)

`breakout_signal.py`'s `_evaluate_signal_sync` only ever checked **the single most recently
completed candle** at the instant it was called. With `BREAKOUT_SCAN_MAX_PER_CYCLE=10` and
`BREAKOUT_SCAN_INTERVAL_SECONDS=60` against a combined ~90-symbol CE+PE dispatcher watchlist,
each individual symbol only got looked at roughly once every ~9 minutes - far coarser than the
5-minute candle cadence the strategy itself is built on. If the one qualifying candle for a
symbol wasn't the "current" one at the moment its turn came up, it was gone for the day,
permanently, even though the underlying candle data (especially once WS-fed) was sitting right
there the whole time.

**Direct, quantified proof - BANDHANBNK today:** the real qualifying candle was 09:15 IST
(would have entered the CE at ~Rs1.45). The live scanner's first actual check of BANDHANBNK
wasn't until ~09:22 IST (`signaled_at: "2026-09-23T09:22:02"`), by which point price had already
run and the real ATM CE entry cost Rs4.22 - a ~3x worse, already-extended entry - which then hit
`TRAILING_SL_HIT` at Rs3.56 for a real -15.64% / -Rs2,376 loss just over a minute later. The same
signal, taken at its actual 09:15 confirmation, would have been a clean +10% TARGET_HIT winner
in simulation. This is not a hypothetical inefficiency - it directly caused a real loss today.

## The fix (implemented in code, 23 Sep 2026, NOT yet deployed - see below)

`underlying_candle_feed.py` (WS candle reconstruction, `BREAKOUT_USE_WS_CANDLES` flipped on for
all three packages today) already keeps a free, local, persisted candle history per symbol
(`get_candles_dict`, up to `MAX_BARS_KEPT=120` bars) - no REST cost to read. The single-snapshot
design in `breakout_signal.py` simply never made use of that full history. Changes made to
`breakout_signal.py`:

1. **`_check_candle`**: the old 7-check body split out of `_evaluate_signal_sync` into a pure,
   reusable per-candle function (takes an explicit candle index + pre-fetched daily data).
2. **`_evaluate_ws_walk_sync`**: new WS-only evaluator that, for a WS-fresh symbol, walks
   EVERY unchecked completed candle since the watchlist item's own `checked_through_epoch`
   (persisted per-item, new field) - not just the latest one - returning the first one that
   confirms. A symbol added mid-session (`checked_through_epoch=None`) gets its FULL day's
   history examined on the very next scan, closing the exact gap BANDHANBNK hit today.
3. **Stale-candle reversal guard**: a walked candle that ISN'T the freshest one available (real
   time has passed since it closed) is re-validated against the latest close via
   `reversal_filters.check_underlying_move_confirms_exit` (the same real, already-backtested
   0.10%-move check the capacity backlog already trusts) before being treated as tradeable - a
   naive "just check every missed candle" walk would otherwise risk firing a real entry into a
   breakout that has since fully reversed hours later. Verified with a synthetic unit test: a
   reversed stale candle is correctly skipped (no phantom entry), a still-live one still fires.
4. **`_evaluate_signal_sync` (REST path) is UNCHANGED** - still single-snapshot, since walking
   many candles per symbol per cycle over REST would reintroduce the exact rate-limit risk
   `underlying_candle_feed.py` was built to remove. Only WS-fresh symbols get the new walk.
5. **`_fetch_daily_cached`**: 20d/50d daily closes/volumes don't change intraday - now cached
   per symbol per calendar date instead of refetched every cycle, since the WS-walk path can now
   touch a symbol many times per cycle where the old code touched it once every ~9 minutes.
6. **Scan cadence**: `_scan_cycle`/`_dispatch_scan_cycle` now split pending symbols into
   WS-fresh (evaluated in FULL every cycle, no `BREAKOUT_SCAN_MAX_PER_CYCLE` cap, no
   `BREAKOUT_SCAN_PACE_SECONDS` pacing except on the rare real daily-REST-fetch call) vs.
   REST-fallback (unchanged capped/paced behavior).

Validated: `py_compile` clean; a standalone synthetic-candle unit test (not committed - ad hoc,
run inline) confirmed (a) the walk correctly finds a qualifying candle missed on the first check
of a newly-joined symbol, (b) `checked_through_epoch` correctly advances and never re-examines a
cleared candle, (c) a stale-and-since-reversed candle is correctly skipped rather than fired,
(d) `_check_candle`'s math is byte-identical to the old single-snapshot behavior on the same
input (regression-safe - only the iteration wrapper changed, not the 7 checks themselves).

## NOT yet deployed

Per [[feedback-live-trading-safety]] this changes real, live signal-detection logic feeding
real order placement - code was written and locally validated but deliberately NOT pushed/
deployed/restarted on the droplet without the user's explicit go-ahead, especially since it was
written during live market hours. Next step before going live: ideally a WS-parity-style replay
of the new code path against a real day's data (similar rigor to
`backtest_ws_candle_reconstruction_parity.py`), then deploy+restart per the normal checklist.
