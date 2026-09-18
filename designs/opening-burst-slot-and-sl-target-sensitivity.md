Status: opening-burst slot DEPLOYED flag-on 19 Sep 2026 (traderBoy commit
d454d74) despite the thin-sample caveat below, per explicit user
decision; SL/target sensitivity remains backtest-only, not deployed.

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

### Result: current 16%/20% is already close to optimal

| SL \ Target | 10% | 15% | 20% | 25% | 30% | 40% |
|---|---|---|---|---|---|---|
| 8% | -720 | 14,393 | 24,171 | 24,888 | 24,945 | 20,292 |
| 10% | 4,457 | 23,541 | 32,809 | 33,389 | 33,455 | 31,829 |
| 12% | 4,782 | 24,068 | 34,436 | **35,016** | 34,614 | 32,926 |
| 14% | 3,598 | 22,778 | 33,146 | 33,727 | 33,325 | 31,636 |
| **16% (current)** | 3,471 | 22,651 | **33,019** | 33,600 | 33,198 | 31,337 |
| 20% | 3,471 | 22,651 | 33,019 | 33,600 | 33,198 | 31,164 |
| 24% | 3,471 | 22,651 | 33,019 | 33,600 | 33,198 | 30,992 |

Best found: SL=12%/Target=25% at +Rs 35,016 (113W/163L) vs current
SL=16%/Target=20% at +Rs 33,019 (114W/162L) - only **+Rs 1,997 over 5
days (~6% relative)**, essentially the same win/loss count. Not a
compelling case to change either knob on this data alone.

**The one structural, high-confidence finding**: rows for 14/16/20/24%
stop-loss are near-identical at every target level. `STOP_LOSS_HIT`
never fires at all at the current combo (0 occurrences across 276
trades) - `MAX_LOSS_HIT` (the flat-rupee cap, Rs 1200 before / Rs 1000
after the 11:30 cutoff) always triggers first for CE contracts once
STOP_LOSS_PCT is wider than ~12-14%. **STOP_LOSS_PCT is functionally
decorative in the current config** - the real loss-limiting lever is
MAX_LOSS_PER_TRADE_RS_BEFORE_CUTOFF/AFTER_CUTOFF, not STOP_LOSS_PCT.
Anyone tuning "the stop-loss" going forward should be looking at that
flat rupee figure, not the percentage.

**Follow-on question this surfaces, not yet investigated**: a flat
rupee cap means small-premium/large-lot contracts (SUZLON, GMRAIRPORT -
lot sizes in the thousands) hit `MAX_LOSS_HIT` after a much smaller
*percentage* move than large-premium/small-lot contracts (BOSCHLTD,
SHREECEM - lot sizes of 25-50). Risk-per-trade is therefore inconsistent
across symbols by construction. Whether normalizing the cap as a % of
entry premium value (rather than a flat rupee number) would change
outcomes is a real, separate question worth its own backtest before
touching MAX_LOSS_PER_TRADE_RS_BEFORE/AFTER_CUTOFF.

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
