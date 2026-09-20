Status: BACKTEST ONLY, high-fidelity real-gate replication. +Rs71,479.70
delta vs real Luxury PnL over 14 days, but only 18 trades - directional
evidence, not a deployment case. Nothing wired live.

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

**The one place a live version would genuinely differ from this
backtest's own numbers - the EXIT side.** This backtest only modeled a
fixed target(+20%)/hard-stop(-16%) pair. A real position, once created,
is managed by Luxury's existing `monitor_loop`/`_check_one_position`/
`_exit_reason_for` - which checks, in order: `MAX_LOSS_HIT` (an absolute
rupee cap, tighter before `RISK_THRESHOLD_CUTOFF_TIME`), `TARGET_HIT`,
`PROFIT_PROTECTION_HIT` (locks in profit above a rupee threshold with a
giveback buffer), a **dynamic trailing stop-loss** that ratchets tighter
as price rises (`TRAILING_SL_HIT`/`STOP_LOSS_HIT`), `SUPERTREND_EXIT`,
and `LIQUIDITY_GUARD_ZERO_VOLUME` (exits early on a thinly-traded
contract going quiet). None of these are modeled here. A live-deployed
version of this signal would very plausibly exit earlier and differently
than this backtest's clean target/stop pair suggests - **this backtest's
+Rs71,479.70 delta should not be read as what a live version would
actually produce on the exit side, only as evidence the ENTRY signal
itself finds genuinely strong setups.**

## What's still open

Backtest evidence only, per [[feedback-live-trading-safety]] - nothing
wired live regardless of how this reads. See [[breakout-scanner-vs-real-
pnl]]'s own "what's still open" for the same standing next-step options
(more data, a new parallel signal source, or leave documented). If a live
version is ever built, the real integration point is a single new
`_process_one_entry()` call from a new 5-min signal-watcher loop - not a
rewrite of any existing gate.
