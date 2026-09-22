Status: LIVE-DEPLOYED config change (21 Sep 2026) - Luxury's real
MAX_LOSS_PER_TRADE_RS/PROFIT_PROTECTION_THRESHOLD_RS/GIVEBACK_PCT were
raised on the live bot (.env, both local and droplet) straight off this
backtest's own finding that `PROFIT_PROTECTION_HIT` was cutting most
winners short. Result: delta improved from +Rs32,649.10 (old config) to
**+Rs73,999.10** (new config) on the same 18-trade sample, now with ZERO
PROFIT_PROTECTION_HIT exits at all - 16/18 trades ride clean to
TARGET_HIT. See "Config raise" section below for the exact before/after
numbers and caveats. The breakout-SIGNAL feature itself remains flag-
enabled and live (see [[breakout-scanner-vs-real-pnl]]); this update is
about Luxury's own risk-parameter config, which affects every Luxury
position (not just breakout-signal-sourced ones).

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
   unlike Options' own COUNT=1 (where that path is dead code). See
   [[reversal-trend-strength-filter-arc]] for where these two thresholds
   actually came from - a 5-round backtest arc the following day.
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

## Config raise, deployed live (21 Sep 2026, user request)

Straight off the "PROFIT_PROTECTION_HIT dominates and cuts winners short"
finding above, the user raised Luxury's real risk parameters - deployed
to BOTH the local repo's `.env` and the droplet's `.env` (gitignored,
never a git commit - scp'd directly), then the live service restarted
(positions confirmed flat immediately before and after, per the standing
safety checklist):

| Setting | Old | New |
|---|---|---|
| MAX_LOSS_PER_TRADE_RS (CE) before/after 11:30 | Rs4,500 / Rs2,100 | **Rs5,500 / Rs3,100** |
| MAX_LOSS_PER_TRADE_RS (PE) before/after 11:30 | Rs3,500 / Rs1,600 | **Rs4,500 / Rs2,600** |
| PROFIT_PROTECTION_THRESHOLD_RS before/after 11:30 | Rs2,000 / Rs1,500 | **Rs5,000 / Rs2,500** |
| PROFIT_PROTECTION_GIVEBACK_PCT | 3% | **8%** |

Re-ran the exact same 18-signal backtest (same 14-day window, same
entries) against the new config - nothing else about the methodology
changed.

### Result: PROFIT_PROTECTION_HIT stopped firing entirely

| | Trades | Wins | Win Rate | Total PnL |
|---|---|---|---|---|
| REAL Luxury (unaffected historical baseline) | 125 | 45 | 36.0% | -Rs17,909.35 |
| Old config (Rs2,000/1,500 threshold, 3% giveback) | 18 | 11 | 61.1% | +Rs14,739.75 |
| **New config (Rs5,000/2,500 threshold, 8% giveback)** | 18 | 16 | **88.9%** | **+Rs56,089.75** |
| **Delta vs. real (new config)** | | | | **+Rs73,999.10** |

Exit reasons, new config: **TARGET_HIT 16**, MAX_LOSS_HIT 1 (PAYTM,
16 Sep - landed at exactly -Rs5,500, the new cap, same precise price-
level-conversion validation as before), LIQUIDITY_GUARD_ZERO_VOLUME 1
(MAHABANK, 11:55 - breakeven, unrelated to the risk-parameter change).
**Zero PROFIT_PROTECTION_HIT exits** - every trade that would have been
cut short before now had room to reach its own natural TARGET_HIT
instead, since the much higher threshold (Rs5,000/2,500 vs Rs2,000/1,500)
and wider giveback (8% vs 3%) together mean a position has to build a
much larger real peak profit, and give back a lot more of it, before this
exit path can fire at all - closing most of the gap to the fully naive
(no-profit-protection-at-all) simulation from the "Real exit-stack
replication" section above (+Rs71,479.70) without actually disabling the
mechanism.

### Day-wise (new config, 18 trades)

| Day | Real Trades | Real PnL | Sim Trades | Sim PnL | Delta |
|---|---|---|---|---|---|
| 1 Sep | 7 | -3,240.00 | 0 | 0.00 | +3,240.00 |
| 2 Sep | 0 | 0.00 | 1 | 9,720.60 | +9,720.60 |
| 3 Sep | 25 | -10,229.00 | 3 | 8,101.25 | +18,330.25 |
| 4 Sep | 12 | -361.50 | 4 | 17,216.40 | +17,577.90 |
| 8 Sep | 4 | 1,951.25 | 1 | 3,317.50 | +1,366.25 |
| 9 Sep | 8 | -3,857.50 | 2 | 9,190.25 | +13,047.75 |
| 10 Sep | 6 | -1,402.50 | 1 | 2,912.00 | +4,314.50 |
| 11 Sep | 1 | -1,338.75 | 1 | 3,870.00 | +5,208.75 |
| 16 Sep | 3 | 6,139.25 | 2 | -3,952.00 | -10,091.25 |
| 17 Sep | 26 | 5,130.00 | 0 | 0.00 | -5,130.00 |
| 18 Sep | 33 | -10,700.60 | 3 | 5,713.75 | +16,414.35 |
| **Total** | | **-17,909.35** | | **56,089.75** | **+73,999.10** |

Only one day (16 Sep) is net negative for the sim vs real - the single
MAX_LOSS_HIT (PAYTM, -5,500) landed that day. 17 Sep has zero sim signals
at all (real Luxury still traded 26 times for -PnL that day) - the
breakout screener simply found no qualifying setup, not a config effect.

### Trade-wise (new config, 18 trades)

| Day | Symbol | Signal Time | Entry | Exit | Exit Reason | Qty | PnL |
|---|---|---|---|---|---|---|---|
| 2 Sep | IDEA | 09:50 | 0.68 | 0.82 | TARGET_HIT | 71,475 | +9,720.60 |
| 3 Sep | MAHABANK | 09:15 | 1.95 | 2.34 | TARGET_HIT | 6,500 | +2,535.00 |
| 3 Sep | SBICARD | 09:15 | 10.80 | 12.96 | TARGET_HIT | 800 | +1,728.00 |
| 3 Sep | GODREJPROP | 10:15 | 59.05 | 70.86 | TARGET_HIT | 325 | +3,838.25 |
| 4 Sep | RELIANCE | 09:15 | 20.95 | 25.14 | TARGET_HIT | 500 | +2,095.00 |
| 4 Sep | IDEA | 11:00 | 0.62 | 0.74 | TARGET_HIT | 71,475 | +8,862.90 |
| 4 Sep | TATASTEEL | 13:40 | 3.88 | 4.66 | TARGET_HIT | 2,750 | +2,134.00 |
| 4 Sep | SWIGGY | 14:30 | 11.30 | 13.56 | TARGET_HIT | 1,825 | +4,124.50 |
| 8 Sep | GVT&D | 09:25 | 132.70 | 159.24 | TARGET_HIT | 125 | +3,317.50 |
| 9 Sep | COALINDIA | 09:15 | 4.95 | 6.50 | TARGET_HIT | 1,350 | +2,092.50 |
| 9 Sep | PAYTM | 09:15 | 48.95 | 58.74 | TARGET_HIT | 725 | +7,097.75 |
| 10 Sep | OIL | 09:15 | 10.40 | 12.48 | TARGET_HIT | 1,400 | +2,912.00 |
| 11 Sep | MCX | 10:35 | 86.00 | 103.20 | TARGET_HIT | 225 | +3,870.00 |
| 16 Sep | PAYTM | 09:15 | 84.15 | 76.56 | **MAX_LOSS_HIT** | 725 | **-5,500.00** |
| 16 Sep | PATANJALI | 13:35 | 7.20 | 8.64 | TARGET_HIT | 1,075 | +1,548.00 |
| 18 Sep | BHEL | 09:15 | 6.85 | 8.22 | TARGET_HIT | 2,625 | +3,596.25 |
| 18 Sep | APLAPOLLO | 10:15 | 30.25 | 36.30 | TARGET_HIT | 350 | +2,117.50 |
| 18 Sep | MAHABANK | 11:55 | 1.34 | 1.34 | LIQUIDITY_GUARD_ZERO_VOLUME | 6,500 | 0.00 |

### Caveats specific to this change

- **This is a real, live risk-parameter change affecting EVERY Luxury
  position**, not just breakout-signal-sourced ones - a normal Chartink-
  alert-driven Luxury trade now also rides to a much wider Rs5,000/2,500
  profit-protection threshold and 8% giveback, and can lose up to
  Rs5,500/3,100 (CE) or Rs4,500/2,600 (PE) before MAX_LOSS_HIT instead of
  the old, tighter caps. That is a deliberate, larger risk-per-trade
  trade-off in exchange for letting winners run - not free upside.
- **Still the same 18-trade sample** - the config change was validated by
  re-running the identical historical signals, not new data. The absence
  of any PROFIT_PROTECTION_HIT in this specific sample doesn't guarantee
  none will ever fire again at the new, higher threshold - it means none
  of these particular 18 trades' peaks crossed it.
- The MAX_LOSS_HIT cap is now also correspondingly larger (Rs5,500/3,100
  vs Rs4,500/2,100) - the SAME kind of bad trade that used to cap out at
  -Rs4,500 now caps out Rs1,000 worse per occurrence before/after 11:30.
  This wasn't separately backtested against a scenario where a real trade
  would have kept losing past the OLD cap - only the one MAX_LOSS_HIT
  case in this sample (PAYTM) is visible, and it hit the cap in the very
  first candle, so there's no evidence here of how much further it might
  have fallen without a cap at all.

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
