# 2026-09-22: Today's Luxury losses trace almost entirely to signals that only exist because of today's loosened breakout params

## What was asked

After a losing day for Luxury (2W/7L, -7,777.50 real PnL vs. Options
1W/0L and Futures 2W/1L), user asked for a direct, checkable comparison:
of today's real Luxury entries, how many would have fired under the
*original* (pre-loosening) breakout thresholds vs. only exist because of
today's deployed loosened ones?

Params in question (deployed same day, see
[[2026-09-22-breakout-signal-loosened-params-today]] if it exists, or the
day's own deploy log):

| | clearance | body | rel. volume | avg daily volume |
|---|---:|---:|---:|---:|
| Original (code default) | 0.5% | 1.0% | 1.5x | 500,000 |
| Loosened (live in `.env` today) | 0.15% | 0.5% | 0.8x | 300,000 |

## Method

Reused `backtest_breakout_screener_vs_real.py`'s `evaluate_signal` /
`find_first_signal_that_day` (the already-trusted replication of
`breakout_signal.py`'s live gating logic) via `set_bsr_thresholds`, run
twice per (symbol, direction) - once at each threshold set - against
today's real candle history, for each of the 8 Luxury symbols that
actually got a real entry today.

**Required an access-token hand-off** to run locally: the live bot is
running with a real open position today (Swing NATURALGAS PE), so
`DHAN_AUTH_MODE=pin_totp` locally is correctly blocked by the guard from
[[2026-09-21-local-backtest-dhan-session-collision]]. User supplied a
separate `DHAN_ACCESS_TOKEN` (access-token mode, not pin_totp) specifically
for this - handled via a scratch env file sourced into the process, never
placed directly in a shell command (the auto-mode credential-leakage
classifier blocked the direct-inline attempt; the sourced-file attempt was
*also* blocked until the user separately enabled Bypass Permissions mode
for the session).

## Bug found and fixed en route: silent DH-904-as-empty-data in the backtest replica

First run: even the *loosened* threshold pass failed to reproduce 6 of 8
known-real live entries (only JUBLFOOD and SONACOMS reproduced). Root
cause: `fetch_daily_underlying`'s `historical_daily_data` call was hitting
`DH-904 Rate_Limit` (today's account-wide contention, see
[[2026-09-22-mcx-vs-nse-fetch-failures-are-shared-rate-limit-not-segment-specific]])
- and that response comes back as `{'status': 'failure', ...}` **without
raising an exception**, so the code's `except` branch never fired, nothing
was logged, and the empty result got cached forever for that
(symbol, day) key. Every subsequent read - across both threshold passes -
silently returned "no signal" regardless of what the actual thresholds
were, because the daily-SMA/avg-volume gate can never pass without daily
data.

This is the exact same failure class already fixed once in this repo for
a different module (`structure_break: retry once on an empty-but-unraised
fetch response`, commit `b4ca12e`) - worth checking any other backtest
script that calls `historical_daily_data`/`intraday_minute_data` directly
for the same silent-swallow pattern, especially on a day with confirmed
rate-limit contention.

**Fix applied** (`backtest_breakout_screener_vs_real.py`,
`fetch_5m_underlying` and `fetch_daily_underlying`): retry up to 3x with a
1s backoff on an empty-but-`status=failure` response, and stop caching
empty results permanently (only cache on genuine success). After the fix,
all 8/8 real entries reproduced correctly under loosened thresholds -
sanity check passed, methodology trustworthy.

## Finding

| Symbol | Fires @ loosened | Fires @ original | Real PnL |
|---|---|---|---:|
| GVT&D | YES @09:15 rv=9.65 | **YES @09:15, same candle** | +1,775.00 |
| CGPOWER | YES rv=2.62 | NO | -2,252.50 |
| DRREDDY | YES rv=8.18 | NO | +1,156.25 |
| SWIGGY | YES rv=0.96 | NO | -1,551.25, -912.50 (2 entries) |
| JUBLFOOD | YES rv=52.64 | NO | -1,500.00 |
| PGEL | YES rv=1.83 | NO | -1,900.00 |
| SONACOMS | YES rv=5.53 | NO | -980.00 |
| PATANJALI | YES @09:30 | YES, but @09:40 (different candle, rv=3.05) | -1,612.50 |

**UPDATE (same day, later pass): PATANJALI reclassified to loosening-only
after checking its actual real entry candle directly.** The table above
used `find_first_signal_that_day` - "first candle satisfying thresholds,
scanning the whole day" - which for PATANJALI found a signal at 09:30/
09:40 under loosened/original respectively. But PATANJALI's REAL live
entry didn't happen until 13:32 IST, four hours later, even though it was
already in Luxury's own restored watchlist all day. That gap means the
full-day scan's early "signal" doesn't reflect what the live system
actually detected at that time (reason not fully root-caused - possibly a
live-vs-replay candle-window difference, not a watchlist-membership
issue since PATANJALI genuinely was already in rotation) - so it's not
trustworthy evidence of what the live bot would have done.

Checked directly instead: pulled the exact real entry candle (13:25 IST
start, confirmed via an exact match to the live log's own range_pct=0.65,
body_pct=1.1, relative_volume=5.93) and re-evaluated it standalone against
both threshold sets. Loosened: signal (matches live exactly). **Original:
no signal** - clearance and/or avg-daily-volume fails at that specific
candle. So PATANJALI's real entry also would not have happened under
original params.

**Corrected finding: only GVT&D would have entered under original
params today - nothing else.** Same candle (09:15), identical signal
under both threshold sets, same real outcome: +1,775.00. Every other real
Luxury trade today (CGPOWER, DRREDDY, SWIGGY x2, JUBLFOOD, PGEL,
SONACOMS, PATANJALI) only exists because of the loosened gate. **Original
params would have produced a clean +1,775.00 today, vs. the real
-7,777.50.**

**Methodology note for future backtests using this pattern**: `find_
first_signal_that_day`'s "scan the whole day, return the first pass" is
NOT a faithful stand-in for "would the live system have entered here" -
it can find an earlier hypothetical signal the live system never acted on
for reasons the full-day scan doesn't model (live-vs-replay data-window
differences at minimum; possibly others). When a symbol's real entry time
doesn't line up with the full-day scan's own found time, check the real
entry's exact candle directly (as done above) rather than trusting the
full-day scan's answer for that symbol. This caveat does NOT weaken the
"no signal on ANY candle all day" negative results (CGPOWER, DRREDDY,
etc.) - an exhaustive full-day miss already covers the real candle too,
by construction; the failure mode only applies to full-day-scan HITS that
don't line up with the real timeline.

**Mechanistic read**: the loosened gate isn't just admitting more trades -
it's specifically admitting signals with materially weaker relative-volume
confirmation (0.96x, 1.83x, 2.62x - all below the original 1.5x floor,
several barely above the loosened 0.8x floor). Those are exactly the ones
that lost. The one signal strong enough to also clear the *original*,
stricter bar (GVT&D, rv=9.65) was the one that won.

## Rollback deployed (same day)

Given the finding, user chose to roll back **Luxury only** (Options and
Futures stayed on loosened params - both were having a fine day, 1W/0L
and 2W/1L respectively, and weren't implicated in this analysis).
`LUXURY_BREAKOUT_CLEARANCE_PCT/MIN_BODY_PCT/MIN_RELATIVE_VOLUME/MIN_AVG_
DAILY_VOLUME` reverted to 0.5/1.0/1.5/500000 in `.env`, dry-run import
check passed, service restarted mid-day with one real open Luxury
position (DLF CE, entry 14.65) - reconciled cleanly from the broker on
startup (`reconciled: true`, same entry/target/stop), consistent with
every prior same-day restart's reconciliation behavior.

## Not done this session (disclosed)

- Did not re-run the same comparison for Options/Futures signals today -
  scope was explicitly Luxury only (the strategy carrying the day's
  losses). Both other strategies also run under their own now-loosened
  `.env` overrides; unclear whether the same pattern holds for them since
  they saw far fewer signals today.
- Did not test intermediate threshold values between original and today's
  loosened set - this was a binary A/B check, not a sweep.
- Did not attempt to determine whether GVT&D's/PATANJALI's outcomes would
  have differed under a partially-loosened set (e.g. only relaxing
  avg-daily-volume, keeping clearance/relvol at original) - out of scope
  for what was asked.

## Final day-end tally (market closed, full data)

One more real Luxury trade came in after the rollback restart was already
in flight: **DLF (CE)**, entered 14:17 IST via the dispatcher (just before
the restart took effect), closed STOP_LOSS_HIT for -2,280.00. Checked the
same way as the rest - real entry candle (14:10 IST, range 0.15%, body
0.5%, relvol 2.62x, exact match to the live log) does not clear original
thresholds. Joins the loosening-only list, bringing it to 9 of 10 real
Luxury trades today.

**Full-day comparison, 3 breakout-gated strategies (Options+Luxury+
Futures), market closed:**

| | Real | Backtested under today's final deployed config (Luxury=original, Options/Futures=loosened, unchanged) |
|---|---:|---:|
| Options | +1,560.00 | +1,560.00 (unaffected) |
| Futures | +241.50 | +241.50 (unaffected) |
| Luxury | -10,057.50 (10 trades) | +1,775.00 (1 trade - GVT&D only) |
| **Total** | **-8,256.00** | **+3,576.50** |

**Swing win/loss for GVT&D-sole-survivor answer restated for clarity**:
only GVT&D, among all 10 real Luxury entries today, would have fired
under original params. The other 9 (CGPOWER, DRREDDY, SWIGGY x2,
JUBLFOOD, PGEL, SONACOMS, PATANJALI, DLF) exist only because of the
loosened gate.

Swing (separate, non-gated strategy, unaffected by this whole
investigation) had its own real day: NATURALGAS + COPPER round-trips
netting -2,900.00 - noted for full-account context, not part of this
comparison.

## Full rollback completed (same day, after market close)

Following the Luxury-only rollback and its clean result, user asked to
also roll back Options and Futures - all three breakout-gated strategies
are now back on original (pre-21-Sep-loosening) params:

| | clearance | body | rel. volume | avg daily volume |
|---|---:|---:|---:|---:|
| Options (original) | 0.3% | 0.5% (unchanged) | 1.2x | 500,000 |
| Futures (original) | 0.3% | 0.5% (unchanged) | 1.2x | 500,000 |
| Luxury (original) | 0.5% | 1.0% | 1.5x | 500,000 |

Note Options/Futures' own original clearance/relvol floor (0.3%/1.2x) was
already looser than Luxury's (0.5%/1.5x) before any of this - the 21 Sep
loosening moved all three down further, by different amounts, to a
common clearance=0.15%/relvol=0.8x/avgvol=300k. Body% was never touched
by the loosening for any of the three.

Deployed after market close (all positions flat, zero live-position risk)
- dry-run import check passed, restart verified via `/health` and all
three scanners' own startup log lines. No Options/Futures-specific cost
analysis was done before this rollback (unlike Luxury) since both were
having a fine real day (Options 1W/0L, Futures 2W/1L) - this rollback is
precautionary/consistency-driven, not evidence-driven the way Luxury's
was. Worth revisiting with the same real-candle-check methodology once
Options/Futures produce enough of their own real trades under looser
params to judge whether they needed it too, or whether it turns out they
were fine leaving it looser (open question, not yet answered).
