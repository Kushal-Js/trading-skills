# Breakout-signal loosened params, backtested against today's real alerts, then deployed (21 Sep 2026)

Status: **DEPLOYED AND LIVE** (21 Sep 2026, same day) - straight off the
backtest below, user request.

## Context

Same day as [[2026-09-21-breakout-signal-sole-entry-path-deploy]] (the
breakout-signal scanner's promotion to the SOLE real entry path for
Options/Luxury/Futures). After that deploy, the user first backtested
the deployed thresholds (clearance=0.3%, body=0.5%, relvol=1.2x,
avg_daily_volume>=500,000 - the same combo carried over from
[[luxury-breakout-detection-parameter-sweep]]) against TODAY's real
alerts specifically (not a historical multi-day window) - see
[[2026-09-21-breakout-signal-sole-entry-path-deploy]]'s own final
verification section for that run. Result: only **5 qualifying signals
all session** out of 58 real alerts across the 3 packages by ~12:10 PM
IST - a much lower signal rate than any prior multi-day backtest in this
series saw, prompting the user to ask for a loosened-parameter re-test
the same day.

## Backtest methodology (same script family as every other entry in this
series, extended to run jointly)

New script: `backtest_deployed_today_breakout_signal.py` (deployed-
threshold baseline) and `backtest_deployed_today_breakout_signal_
updated_params.py` (the loosened-params copy, differs only in the
threshold-setting block). Unlike every PRIOR backtest in this line of
work (which ran one package at a time, checking cross-strategy locking
only against the OTHER packages' real non-gated trade history), this one
runs Options (CE+PE) + Luxury (CE) + Futures (CE) **JOINTLY in one
shared timeline** - since all three now genuinely run the same
breakout-signal gate concurrently in production, a single shared
`open_positions` dict (keyed by underlying symbol) is the more faithful
cross-strategy-claim model. Swing (still its own non-gated real path)
locks symbols too, via the same interval-overlap proxy every prior
script uses for "other real strategies."

Every real entry gate modeled per-package using that package's own live
config (daily re-entry cap, RSI-loss-reentry block, loss-repeat block +
trend check, volume-floor gate where it exists - NOT Luxury, confirmed
grep-absent same as always - liquid-contract resolution, capacity + the
CE opening-burst slot, square-off/trading-window cutoffs, gap-down CE
delay). Real exit ladder per package's own `_exit_reason_for` priority
order, walked on real 1-min option candles up to the moment each run was
executed (today's session was still in progress both times - open
positions would show `STILL_OPEN`, though none did in either run).

Data source: today's REAL `webhook_alerts.log`/`real_trades.log`
scp'd fresh from the droplet immediately before each run (not the
locally-cached multi-day history/ files used by every other backtest in
this series, which only go up to 18 Sep).

## Result

| | Deployed (clearance 0.3%, relvol 1.2x, vol>=500k) | Loosened (clearance 0.15%, relvol 0.8x, vol>=300k) |
|---|---:|---:|
| Raw signals | 5 | 10 (11 on a later re-run with more alerts synced in - the 11th was filtered by the volume-floor gate, no new trade) |
| Entered | 5 | 9 |
| Win rate | 80.0% | 77.8% |
| Total PnL | +Rs5,291.00 | +Rs11,091.55 |
| Adjusted for a duplicate leg* | +Rs3,561.00 | +Rs9,361.55 |

Body% (0.5%) was UNCHANGED in the loosened run - not part of this
tuning pass, only clearance/relvol/avg-daily-volume were loosened, per
the user's explicit request.

*Both runs show Options CE and Luxury CE both firing on INDHOTEL at the
identical minute (09:25:00) - a real limitation of the interval-overlap
cross-strategy-claim proxy (see [[luxury-signal-gated-live-simulation]]'s
own note on why the real momentary `cross_strategy_registry` claim isn't
reconstructable after the fact): in production only ONE of the two would
likely have won that race, so the adjusted total (excluding one
duplicate INDHOTEL leg, since both are identical trades) is probably
closer to the real outcome than the raw total.

### Trade-wise (loosened params, 9 trades)

| Strategy | Type | Symbol | Signal | Entry | Entry Px | Exit | Exit Px | Reason | Qty | PnL |
|---|---|---|---|---|---:|---|---:|---|---:|---:|
| Options | PE | LICI | 09:15:00 | 09:17:00 | 3.80 | 09:26:00 | 3.23 | TRAILING_SL_HIT | 1,400 | -798.00 |
| Luxury | CE | LAURUSLABS | 09:15:00 | 09:15:00 | 26.80 | 09:16:00 | 32.16 | TARGET_HIT | 850 | +4,556.00 |
| Luxury | CE | CGPOWER | 09:15:00 | 09:15:00 | 12.50 | 09:16:00 | 15.00 | TARGET_HIT | 850 | +2,125.00 |
| Luxury | CE | BANDHANBNK | 09:15:00 | 09:22:00 | 1.85 | 09:33:00 | 1.99 | LIQUIDITY_GUARD_ZERO_VOLUME | 3,600 | +504.00 |
| Luxury | CE | SONACOMS | 09:15:00 | 09:15:00 | 11.30 | 09:52:00 | 9.72 | TRAILING_SL_HIT | 1,225 | -1,937.95 |
| Options | CE | INDHOTEL | 09:25:00 | 09:25:00 | 8.65 | 09:25:00 | 10.38 | TARGET_HIT | 1,000 | +1,730.00 |
| Luxury | CE | INDHOTEL | 09:25:00 | 09:25:00 | 8.65 | 09:25:00 | 10.38 | TARGET_HIT | 1,000 | +1,730.00 |
| Luxury | CE | TORNTPHARM | 10:30:00 | 10:30:00 | 62.00 | 10:31:00 | 74.40 | TARGET_HIT | 125 | +1,550.00 |
| Luxury | CE | MANKIND | 12:15:00 | 12:15:00 | 32.65 | 12:15:00 | 39.18 | TARGET_HIT | 250 | +1,632.50 |

Exit reasons: TARGET_HIT 6, TRAILING_SL_HIT 2, LIQUIDITY_GUARD_ZERO_VOLUME 1.

## Caveats

- **Single-day sample (9 or 5 trades)** - by far the smallest sample in
  this entire line of work (every prior multi-day backtest had 8-125
  trades over 14 days). Directional at best, more so than usual.
- The deployed-vs-loosened comparison is confounded the same way every
  parameter change in this series is - not a clean isolate, since the
  underlying set of qualifying symbols differs entirely between the two
  runs (loosened finds MORE signals, not just different prices on the
  same signals).
- Same disclosed gaps as every other backtest here: funds check not
  modeled (fails open), cross-strategy lock is an interval-overlap
  proxy not the real momentary claim (see the INDHOTEL duplicate above
  for a concrete instance of this proxy's own limitation).
- This is ONE trading day of loosened-threshold behavior, re-verified
  once (an 11th raw signal appeared on a later re-sync but was filtered
  by the volume floor, no change to the trade list) - not the kind of
  multi-day validation the ORIGINAL clearance=0.3/relvol=1.2 combo got
  (the 27-combo sweep against 15 days of Luxury history). Deployed
  anyway on explicit user instruction, same day, fully informed.

## Deployed (21 Sep 2026, same session)

`.env` updated for all three packages (`LUXURY_BREAKOUT_CLEARANCE_PCT`/
`OPTIONS_BREAKOUT_CLEARANCE_PCT`/`FUTURES_BREAKOUT_CLEARANCE_PCT` all
0.3->0.15, `*_BREAKOUT_MIN_RELATIVE_VOLUME` all 1.2->0.8, new
`*_BREAKOUT_MIN_AVG_DAILY_VOLUME=300000` for all three - Options/Futures
previously had no explicit override, relying on the code default of
500,000). Body% left at 0.5 for all three, unchanged. scp'd to droplet,
service restarted per the standing [[project-dhanboy-deployment]]
checklist (fresh position check immediately before restart).
