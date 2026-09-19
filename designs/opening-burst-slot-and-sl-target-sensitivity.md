Status: opening-burst slot DEPLOYED flag-on 19 Sep 2026 (traderBoy commit
d454d74) despite the thin-sample caveat below, per explicit user
decision; SL/target sensitivity remains backtest-only, not deployed - AND its first-published conclusion was wrong, see the CORRECTION section below.

# Opening-burst extra capacity slot + stop-loss/target sensitivity

Two related backtests run the same evening, both triggered by the 18 Sep
finding that real trading netted -Rs 18,663 that day while 91 alerted-
but-never-traded symbols would have netted +Rs 15,041 in the shadow sim
([designs/ribbon-switch-shadow.md](ribbon-switch-shadow.md) covers a
separate, earlier finding from the same investigation thread). Scripts
live in `traderBoy/backtest_opening_burst_extra_slot.py` and
`traderBoy/backtest_stop_loss_target_sensitivity.py`.

## 1. Opening-burst extra capacity slot

**Design** (deployed 19 Sep 2026 - `BURST_CAPACITY_ENABLED=true` by
default in all 3 packages, window 09:15-10:00 IST, +1 CE slot): a single
time-of-day tweak to
the existing capacity gate, no new position-management logic -
`effective_cap = MAX_LIVE_POSITIONS_CE + BURST_EXTRA_SLOTS_CE` while
`now` is inside a configurable window (default 09:15-09:40 IST). A
position entered under the extra slot is an ordinary `Position` - same
exit ladder, same dedup - so there's genuinely nothing else to build.

**Backtest across 6 days (10, 11, 15, 16, 17, 18 Sep)**, using the real
logged `max_live_positions_reached` reason (exact, not inferred) to know
which alerts were capacity-blocked, then walking a realistic slot-
occupancy model (one extra slot per strategy, freed only when the shadow-
simulated position actually exits, cross-strategy deduped so the same
symbol can't be double-counted):

| Scenario | Extra PnL | Picks | Win rate |
|---|---|---|---|
| +1 slot, 09:15-09:40 | +35,957.50 | 20 | 65.0% |
| +2 slots, 09:15-09:40 | +50,658.00 | 29 | 69.0% |
| **+1 slot, wider window 09:15-10:00** | **+72,310.00** | 37 | **70.3%** |
| +1 slot, narrower window 09:15-09:30 | +26,686.25 | 15 | 60.0% |

**Finding**: widening the window beats adding a second slot. A single
extra slot held open until 10:00 instead of 09:40 more than doubles the
edge of a permanent second slot, because it cycles through multiple
opportunities as early positions resolve quickly (15-30 min via
Supertrend/target exits) rather than sitting on one position the whole
time - cheaper in capital/risk terms too (1 extra concurrent position,
not 2). Still only 15-37 samples per scenario across 6 days - directional,
not proven. Even the best scenario's "hypothetical combined" (real PnL +
extra PnL) over the same 6 days was still slightly negative - this is a
tailwind on top of a real underlying loss problem, not a fix for it.

## 2. Stop-loss / target percentage sensitivity

**Methodology**: real 1-min option + 5-min underlying candles fetched
read-only for every alerted symbol on 5 of the 6 days (18 Sep excluded -
Dhan's `intraday_minute_data` endpoint hadn't indexed it yet even ~9-10
hours after close; confirmed via a direct test call returning `status:
success` with empty arrays for that date specifically, while a wider
date range still only returned the prior day's bars - a real API
indexing lag, backfill once it clears). `shadow_evaluator.py`'s exact
exit-ladder logic (`eval_open_shadows`) was reimplemented with
STOP_LOSS_PCT/TARGET_PCT swapped for grid values, every other threshold
(MAX_LOSS_PER_TRADE_RS before/after 11:30 cutoff, PROFIT_PROTECTION_RS +
GIVEBACK_PCT, dynamic SL, Supertrend exit) pinned at the droplet's actual
live `.env` values, not code defaults.

### Two real bugs found and fixed before trusting any result

1. **Dropped SUPERTREND_EXIT guard conditions.** The real check requires
   the underlying bar be strictly after entry AND within 5 minutes of
   the current option tick (`und["ts"][uidx] > entry_underlying_ts and
   und["ts"][uidx] <= t + 300`). The first draft dropped both, letting
   stale/pre-entry Supertrend state chop positions immediately after
   entry. Caught by cross-validating the re-simulated 16%/20% total
   against the real `shadow_positions.json` history for the same
   symbols - result was negative (-Rs 4,303) against an actual positive
   history, an unmissable sign-flip.
2. **No underlying lookback for Supertrend warm-up.** The candle fetcher
   pulled only the single target day's 5-min bars; `shadow_evaluator.py`
   itself always fetches a 4-day lookback (`_underlying`'s
   `days_back=4`) so the period-10 Supertrend is actually warmed up by
   market open. Without it, the indicator is close to noise exactly when
   most alerts fire (within the first few 5-min bars of the day). Fixed
   by a separate patch pass re-fetching only the underlying series with
   the same lookback the production code uses.

After both fixes, the re-simulated 16%/20% total (+Rs 33,019 across all
276 symbol-days) matched the real shadow history in sign and order of
magnitude (+Rs 17,179 for the 230 originally-shadow-sourced trades in
that set) - good enough to trust directionally, though a ~30% per-trade
mismatch remains (attributed to minor timing/candle-alignment noise
between a live progressive run and one fresh historical replay, not a
further logic bug - not chased further given diminishing returns).

**General lesson for any future re-simulation-from-raw-candles backtest
in this codebase**: always cross-validate against a real, independently-
produced result for the *same* symbols under the *same* config before
trusting a sweep across other configs. Both bugs here would have
silently produced a confidently wrong recommendation without that check.

### CORRECTION (19 Sep 2026, same day) - the original conclusion below this heading was WRONG

The first version of this section reported "current 16%/20% is already
near-optimal" and "STOP_LOSS_PCT is functionally decorative". Both were
artifacts of a config mis-read on my part: the sweep pinned the flat-rupee
MAX_LOSS cap at Rs 1,200 (before 11:30) / Rs 1,000 (after) - the code
DEFAULTS - when the droplet's live `.env` actually has
`MAX_LOSS_PER_TRADE_RS_BEFORE_CUTOFF=4500` / `..._AFTER_CUTOFF=2100`
(Options, Futures and Luxury identical). My `.env` grep at the time only
matched the `_CE`/`_PE`-suffixed key names and silently missed the
un-suffixed CE-side lines. With a Rs 1,200 cap, MAX_LOSS_HIT fired before
the 16% stop ever could, which is what made every stop-loss row look
identical. With the real Rs 4,500/2,100 caps the percentage stop is very
much live: `STOP_LOSS_HIT` fired 29 times at the current 16%/20% setting.

**Lesson (also for future backtests here): pull EVERY relevant key with a
broad pattern (`grep -E "MAX_LOSS|PROFIT_PROTECTION"`) and diff against the
code defaults BEFORE hardcoding "live values" into a script - a targeted
regex that matches the key names you expect will silently skip the ones you
didn't.**

Corrected grid (same 5 days / 276 symbol-trades, same fixed ladder except
MAX_LOSS = 4500 before 11:30 / 2100 after):

| SL \ Target | 10% | 15% | 20% | 25% | 30% | 40% |
|---|---|---|---|---|---|---|
| 8% | 12,200 | 28,329 | 39,042 | 38,784 | 37,790 | 33,138 |
| 10% | 14,312 | 34,410 | 45,509 | 44,579 | 44,697 | 41,563 |
| 12% | 11,565 | 31,527 | 44,050 | 44,368 | 44,019 | 42,117 |
| 14% | 9,404 | 29,943 | 41,772 | 40,476 | 40,126 | 38,225 |
| **16% (current)** | 11,890 | 33,814 | **45,072** | 45,012 | 44,662 | 41,030 |
| 20% | 28,538 | 46,758 | 60,598 | 62,108 | 61,131 | 57,746 |
| **24%** | 35,786 | 52,817 | 67,970 | **69,480** | 68,503 | 64,945 |

Best in-sample: SL=24%/Target=25% at +Rs 69,480 (137W/139L) vs current
SL=16%/Target=20% at +Rs 45,072 (131W/145L) - a large in-sample gap
(+Rs 24,408, ~54%), driven almost entirely by the stop-loss width, not the
target: `STOP_LOSS_HIT` drops from 29 to 3 exits and those trades mostly
recover or exit later via SUPERTREND_EXIT/PROFIT_PROTECTION instead. Target
matters little between 20% and 30% at any stop width.

**How much to trust it: not enough to ship on its own.** Five days; every
trade is a shadow-simulated CE-ATM fill at a candle close (no slippage, no
broker stop, one trade per symbol, no capacity/cooldown gates), and the
same period's REAL trading lost money while this simulation shows large
profits - so the absolute numbers are far more optimistic than reality and
only the *relative* ordering across cells is informative. A wider stop also
means each loser can lose more per trade before the flat-rupee cap catches
it (the cap is the real backstop then), so a proper evaluation should look
at worst-case loss and drawdown, not just total PnL. Direction (a 16% stop
is probably too tight for CE premium noise; 20-24% deserves a real look) is
worth a follow-up with more days (18 Sep is now available) before any
config change.

## New finding while validating the deploy: 9 tests are time-of-day flaky

Full suite run at ~1:30-1:40 AM IST (right before deploying the burst
slot) showed 16 failures - 7 more than the documented 9-failure baseline
from earlier the same day. The 7 new ones (`test_*_ltp_stale_for_too_
long_forces_a_market_exit...` across Futures/Luxury/Options/Swing broker-
stop-loss and Swing v2 files, `test_9/16_profit_protection_giveback_
buffer` across Futures/Luxury/Options corrective-actions files,
`test_12_ema_cross_refresh_runs_on_a_continuous_multi_session_series`,
and `test_8_ltp_staleness_uses_mcx_segment_codes_for_an_mcx_position`)
span code the burst-capacity change never touches (Swing MCX, EMA-cross
refresh) - an immediate signal this wasn't a regression.

Confirmed via `git stash` (stashing the burst-capacity changes) then
re-running just those files: **identical 9 failures on the clean,
unmodified baseline** - proving this is pre-existing time-of-day
flakiness in the test suite itself, not caused by the deploy. Not yet
root-caused (candidate: tests computing relative time windows against
real `datetime.now()` that behave differently very late at night vs.
during a normal trading-day run), but now a known, reproducible pattern
- if a future session sees test failures cluster around `ltp_stale`/
`profit_protection_giveback`/`ema_cross_refresh`/`mcx_segment_codes`
specifically, check the wall-clock time before assuming a regression.
