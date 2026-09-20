Status: BACKTEST ONLY, high-fidelity real-gate AND real-exit-stack
replication (updated 20 Sep 2026). Real exits corrected the delta DOWN
from +Rs71,479.70 (naive target/stop only) to **+Rs32,649.10** - real
`PROFIT_PROTECTION_HIT` cuts most winners short well before the naive
+20% target, which materially overstated the earlier number. 18 trades -
directional evidence, not a deployment case. Nothing wired live.

# Luxury-only signal-gated entry, simulated against real production gates

## What was asked

Take the breakout-screener signal-gated hypothesis
([[breakout-scanner-vs-real-pnl]]) and run it "as a live version" scoped
to Luxury's own real alerts only (not the shared cross-strategy bucket),
replicating Luxury's REAL production entry gates - daily re-entry cap,
loss-repeat block, cross-strategy claim, funds, and the opening-burst
extra CE slot already deployed live - rather than the earlier version's
bare signal-confirm-then-enter simulation with no gates at all.

## Scope correction made before running

Luxury's own Chartink scan_names (Kaashvi, Krishvi, laxmi, longTerm,
longTermOptions) are ALL CE in this dataset - confirmed empirically (426
CE symbol-days, 0 PE, across the same 14-day window). Luxury genuinely
never receives a PE alert here; this is real, not a bug.

`Options'` config values were WRONG for Luxury and got corrected: Luxury's
real deployed TARGET_PCT/STOP_LOSS_PCT are **0.20/0.16**, not Options'
0.10/0.03 that [[breakout-scanner-vs-real-pnl]]'s original (non-Luxury-
scoped) run used for every signal regardless of strategy. Also: Luxury's
`ENABLE_SQUARE_OFF` is **False** in production (unlike Options/Futures) -
positions are NOT force-closed at 15:15, so this run's exit ladder walks
forward across real calendar days (via Dhan's option intraday data,
multi-day) until target/stop is hit or the backtest's own data horizon
(today) is reached, rather than forcing an EOD exit.

## Real gates replicated (with Luxury's actual live .env values, read at
runtime via `from Luxury import config`, not hardcoded)

1. **Daily re-entry cap** = 3 (`MAX_DAILY_ENTRIES_PER_SYMBOL`).
2. **Loss-repeat block**, COUNT=**2** (live value, not the code default 1)
   - hard-blocks after 2 real losses on the same symbol same day.
   `LOSS_REENTRY_TREND_CHECK_ENABLED`'s real ADX(>=20)/ER(>=0.3) check
   (via `reversal_filters._compute_adx`/`_efficiency_ratio_at`, real
   historical 5-min data) genuinely gates the 1-prior-loss case here,
   unlike Options' own COUNT=1 (where that path is dead code).
3. **Cross-strategy / broker-wide open-position check**, approximated as
   an interval overlap against Options/Futures/Swing's own REAL trading
   (unaffected by this hypothesis) plus this simulation's own currently-
   open Luxury positions.
4. **Capacity with the real opening-burst slot**: CE cap 3 (+1 = 4 during
   09:15-10:00), PE cap 1 - Luxury's actual live `MAX_LIVE_POSITIONS_CE/
   _PE`, not the code defaults (2/2).
5. **Liquid-contract resolution** (`LIQUID_CONTRACT_GATE_ENABLED`): ATM
   strike must show no 4-consecutive-zero-volume-bar streak AND >=500
   summed daily volume over the last 7 calendar days, else search up to 5
   strikes outward each side for a substitute - real historical option
   data, not assumed.
6. **Option-liquidity entry gate** - same zero-volume-bar check,
   reapplied to whichever contract step 5 resolved.

**Not replicated, deliberately** (see the script's own module docstring
for the full reasoning): the **funds check** needs a real historical
account-balance snapshot that doesn't exist - treated as always-
sufficient, which is exactly production's OWN fail-open behavior when
this check can't complete for real, not a new simplification. A general
"RSI/trend" gate and a "volume floor" gate were NOT invented for Luxury
either - grep-confirmed neither exists in Luxury's real code (RSI/trend
only ever runs after a same-day loss, already covered by gate 2; volume
floor is Options/Futures/Swing-only).

## Result

| | Trades | Wins | Win Rate | Total PnL |
|---|---|---|---|---|
| REAL Luxury (14 days) | 125 | 45 | 36.0% | -Rs17,909.35 |
| SIGNAL-GATED SIMULATION | 18 | 17 | 94.4% | **+Rs53,570.35** |
| **Delta** | | | | **+Rs71,479.70** |

19 raw signals found before gating; only 1 was skipped (`claimed_or_
already_open`) - real production gates barely constrained this
particular sample (capacity/daily-cap/loss-repeat never fired once),
plausible given the low signal density (~1.4/day) rarely collides with a
3-4 slot CE cap.

**Verified, not just trusted**: re-fetched MAHABANK's real 09-03 09:15
option candle directly (`MAHABANK 29 SEP 85 CALL`) after this run showed
several signals resolving to `TARGET_HIT` in the SAME minute as entry,
which looked suspicious at first. Confirmed genuine: that exact 1-min
candle opened at 1.95, high 2.35, against a target of exactly 1.95*1.20=
2.34 - a real, volume-driven single-minute spike (162,500-929,500 volume
in the following minutes), not an off-by-one entry-timing bug. Zero
fetch failures logged across the entire run (checked via grep against the
raw log) - the liquidity/daily-volume gates ran cleanly throughout, no
silent fail-opens masking a broken check.

## Caveats

- **18 trades over 14 days** - directional at best, same as every other
  finding in this line of work so far.
- Funds check untested (see above) - a real capital constraint at the
  time could have blocked some of these.
- Cross-strategy claim is an interval-overlap proxy, not the real
  momentary claim mechanism - could be mildly more conservative
  (blocking cases the real millisecond-scoped claim wouldn't have) or
  mildly more permissive (missing a real settlement-window collision the
  interval approximation can't see), direction unclear.
- Same idealized-fill caveats as [[breakout-scanner-vs-real-pnl]] - no
  slippage modeled, several trades are on very cheap sub-Rs10 premiums.
- The multi-day (no-EOD-forced) exit ladder was not needed in this
  sample - every one of the 18 resolved same-day, so `ENABLE_SQUARE_OFF=
  False`'s effect on this specific result is untested even though the
  mechanism was built to handle it correctly.

## Real exit-stack replication (added 20 Sep 2026, user request)

The result above originally used a naive fixed target(+20%)/hard-stop(
-16%) pair only. Rebuilt `simulate_luxury_real_exit()` to faithfully
replicate Luxury's actual `_exit_reason_for`, checked in the SAME
priority order every tick, with every threshold read live from `Luxury.
config`/`Options.config` (not hardcoded):

1. **MAX_LOSS_HIT** - absolute rupee cap, Rs4,500 CE before 11:30 IST /
   Rs2,100 after (`ENABLE_MAX_LOSS_HIT_BEFORE_CUTOFF=True` live, so this
   can fire any time of day, not just after the cutoff).
2. **TARGET_HIT** - entry x 1.20.
3. **PROFIT_PROTECTION_HIT** - once peak profit exceeds Rs2,000 (before
   11:30) / Rs1,500 (after), exits the moment price dips more than 3%
   (`PROFIT_PROTECTION_GIVEBACK_PCT`, live value) below that peak.
4. **TRAILING_SL_HIT / STOP_LOSS_HIT** - dynamic stop that ratchets up
   1% tighter for every 7% (CE) the position has moved in its favor
   (`ENABLE_TRAILING_SL` itself is off for Luxury - only the dynamic step
   mechanism applies).
5. **SUPERTREND_EXIT** - real 10-period/3.0-multiplier Supertrend on
   5-min underlying candles (`bt_common.supertrend_state_at`, the same
   pure function this repo's other backtests already validated against),
   gated by a real >=0.10% minimum-underlying-move confirmation.
6. **EMA_CROSS_EXIT** - real 9/12 EMA cross on 5-min underlying candles.
   Approximated (disclosed): the exact internal cache semantics of
   `dhan_client`'s `get_cached_ema_cross_crossed` aren't fully knowable
   without reading its live implementation, so this walks back through
   the EMA history to find the most recent fast/slow flip - a reasonable,
   not verified-identical, reconstruction.
7. **LIQUIDITY_GUARD_ZERO_VOLUME** - same 4-consecutive-zero-volume-bar
   check as the entry-side gate, now checked continuously while holding.

All rupee-based caps (MAX_LOSS_HIT, PROFIT_PROTECTION_HIT's giveback
floor, the dynamic trailing SL) are converted to a PRICE level and
checked against each 1-min candle's own high/low, filled at that level
(or the candle's open if price gapped past it) - same convention as
every other exit simulation in this line of work.

### Result: real exits cut the naive version's edge by more than half

| | Trades | Wins | Win Rate | Total PnL |
|---|---|---|---|---|
| REAL Luxury (14 days) | 125 | 45 | 36.0% | -Rs17,909.35 |
| Naive target/stop-only simulation | 18 | 17 | 94.4% | +Rs53,570.35 |
| **Real full exit-stack simulation** | 18 | 11 | 61.1% | **+Rs14,739.75** |

Exit reason breakdown (18 trades): **PROFIT_PROTECTION_HIT 10**,
TARGET_HIT 6, MAX_LOSS_HIT 1, LIQUIDITY_GUARD_ZERO_VOLUME 1.

**The real finding**: `PROFIT_PROTECTION_HIT` dominates and usually exits
near breakeven (6 of the 10 protection-hits closed at pnl=Rs0, or very
close to it) - for the ultra-cheap, huge-quantity contracts this signal
tends to pick (IDEA @0.62-0.68, PAYTM, OIL), the Rs1,500-2,000 profit
threshold is crossed by a tiny few-paise move given the large lot size,
and the 3% giveback then triggers on what's often just normal short-term
noise for a penny-priced option - locking in a wash long before the real
move (several of these same signals went on to hit the +20% target in
the naive version) ever develops. The one confirmed `MAX_LOSS_HIT`
(PAYTM, 16 Sep) landed at EXACTLY -Rs4,500, the configured cap to the
rupee - strong evidence the price-level conversion and time-of-day gating
are computing correctly, not coincidentally close.

**This is exactly why the user's request to replicate real exits mattered
- the naive +Rs71,479.70 delta was a real overstatement.** The corrected
+Rs32,649.10 (Real minus REAL Luxury: 14,739.75 - (-17,909.35)) is still
solidly positive, but the mechanism producing it is now visibly different
and much less clean: 6 clean target hits plus a handful of small giveback
wins/breakevens, not a near-uniform sweep of +20% targets.

## Backtest functions vs. the real deployed functions (asked + verified 20 Sep 2026)

**The backtest does NOT call the actual deployed global functions** - it
replicates their LOGIC and reads their EXACT config thresholds, but as
separate implementations. Grep-confirmed none of these appear anywhere
in `backtest_luxury_signal_gated_live.py`:
- `Luxury/position_store.py`'s `_cap_for()` / `_in_burst_window()` (the
  real burst-slot capacity function)
- `dhan_client.get_liquid_atm_option()` (the real global liquid-contract
  resolver, shared by every live-trading package)
- its internal helpers `_is_contract_liquid_and_active()` /
  `_nearby_option_candidates()`
- `refresh_liquidity_signal()` / `get_cached_illiquid()` (the live
  caching layer those helpers depend on)

**Why they couldn't be called directly**: all of them are built for LIVE
operation - they read `datetime.now()` and/or an in-process cache
(`get_cached_illiquid` only holds a value if something already queried
that exact contract right now) with no "as of this past timestamp"
parameter. A backtest walking through 2 September as if it were "now"
has no way to ask a wall-clock-bound function about a moment three weeks
in the past, so `resolve_liquid_contract()` and the inline burst-window
capacity check in `run()` reconstruct the same decision from real
historical Dhan data instead.

## If this were ever actually deployed live

This matters for scoping future work, not just as trivia: **almost none
of the entry-gate logic above would need to be newly written**. A live
version only needs ONE new thing - a background loop that watches
Luxury's own bucket symbols' real 5-min candles and evaluates the 8
rules, calling Luxury/trading_engine.py's existing `_process_one_entry(
symbol, "CE")` the moment a signal confirms, exactly like the real
Chartink-webhook handler already does today. Everything downstream of
that one call is the REAL, unmodified production pipeline, inherited for
free:
- `position_store.reserve_symbol()` -> internally calls the REAL
  `_cap_for()`/`_in_burst_window()` - the burst slot is automatic, no
  reimplementation needed.
- `cross_strategy_registry.try_claim()` - the REAL momentary claim, not
  this backtest's interval-overlap proxy.
- `dhan_wrapper.has_open_position_for_underlying()` - REAL broker check.
- `_enter_single_position()` calling `dhan_wrapper.get_liquid_atm_option(
  )` - the REAL liquid-contract resolver, with its REAL
  `_is_contract_liquid_and_active()`/`_nearby_option_candidates()`/
  `refresh_liquidity_signal()`/`get_cached_illiquid()` machinery, live
  and current rather than historically reconstructed.
- `reversal_filters.check_option_liquidity()` - REAL entry-time
  liquidity re-check.
- `fund_allocation.has_sufficient_bucket_funds()` - REAL, and would
  actually work correctly live (a genuine real-time balance/margin
  check), unlike this backtest which could only assume it always passes.
- `count_opened_today`/`loss_count_today`/`loss_exit_count_today`
  (trade_history.py), `reversal_filters.check_trend_strength()`,
  `dhan_wrapper.is_rsi_loss_reentry_blocked()` - all REAL, unmodified.

**UPDATE (same day): the exit side has since been replicated too** - see
"Real exit-stack replication" above. At the time this was first written,
only a naive target/stop pair had been tested and the prediction below
was speculative; it turned out directionally correct (real exits DO
change the result materially) but the backtest itself no longer has this
gap. A live position, once created via `_process_one_entry`, would still
be managed by the REAL `monitor_loop`/`_check_one_position`/`_exit_
reason_for` rather than this script's own reconstruction of it - the
remaining difference is the EMA-cross-state approximation disclosed
above (the one exit path not verified byte-identical to production's own
caching), plus that a live position can react to a genuine live LTP tick
mid-candle, not just a candle's own OHLC.

## What's still open

Backtest evidence only, per [[feedback-live-trading-safety]] - nothing
wired live regardless of how this reads. See [[breakout-scanner-vs-real-
pnl]]'s own "what's still open" for the same standing next-step options
(more data, a new parallel signal source, or leave documented). If a live
version is ever built, the real integration point is a single new
`_process_one_entry()` call from a new 5-min signal-watcher loop - not a
rewrite of any existing gate.
