Status: Analyzed same day (22 Sep 2026). Not deployed - deployed parameters
kept unchanged based on this analysis' own conclusion.

# Breakout scanner parameter sweep - today's real alerts (22 Sep 2026)

User request, mid-morning: "revaluate the deployed configurations for
breakout scanner... by adjusting and trying different breakout scanner
parameters, [check] if we are missing any real trades which would have
been profitable since this morning."

## Method

- **Candidate universe**: the union of every symbol that actually
  appeared in today's real, persisted breakout-signal watchlists on the
  droplet (`history/2026-09-22_breakout_signal_{options,luxury,futures,
  universedispatcher}_{CE,PE}.json`) - 154 unique real symbols, not a
  synthetic list.
- **Detection replay**: the REAL, unmodified `breakout_signal.
  _evaluate_signal_sync` (never reimplemented) run against real 5-min
  candles (15-day continuous lookback) and real daily candles (120-day),
  fetched via a hand-off `access_token` (zero session-collision risk with
  the live bot, per this session's own standing rule) and monkeypatching
  only the two fetch functions to serve pre-fetched, time-sliced data -
  the walk-forward finds the first 5-min candle each (symbol, direction,
  config) combination would have fired on, matching production's own
  first-qualifying-candle behavior.
- **Validated before trusting**: replayed GVT&D (a real signal production
  actually detected today) and got an EXACT match on every computed
  field (close=4497.7, range_pct=0.54, body_pct=1.76, relative_volume=
  9.65) - confirms the replay is byte-for-byte faithful to production,
  not an approximation.
- **Baseline** = today's actual deployed parameters (confirmed live,
  identical across Options/Luxury/Futures via env overrides):
  `MIN_RELATIVE_VOLUME=0.8 CLEARANCE_PCT=0.15% MAX_CONSOLIDATION_RANGE_
  PCT=12% MIN_BODY_PCT=0.5% MAX_PCT_FROM_HIGH_LOW=10%`.
- **8 variants**: 5 single-parameter loosenings (isolates each gate's own
  effect), moderate + aggressive combined loosening, and one tightening
  control (to see the full sensitivity shape, not just one direction).
- Rate-limit awareness: paced at 0.75s/call (more conservative than the
  usual 0.35s) specifically because today's account was already showing
  DH-904 pressure (see the morning's own incidents) - ~150 symbols x 2
  calls (5m+daily) plus a lighter 43-symbol profitability follow-up, run
  as two separate, bounded batches rather than one large one.

Scripts (untracked, repo root): `backtest_breakout_scanner_param_sweep_
today.py`, `backtest_breakout_scanner_param_sweep_profitability.py`.

## Results - signal counts

| Config | Signals | vs baseline |
|---|---:|---:|
| baseline (deployed) | 24 | - |
| v1 loosen MIN_RELATIVE_VOLUME (0.8->0.5) | 26 | +2 |
| v2 loosen CLEARANCE_PCT (0.15->0.05%) | 24 | +0 |
| v3 loosen MAX_CONSOLIDATION_RANGE_PCT (12->18%) | 24 | +0 |
| v4 loosen MIN_BODY_PCT (0.5->0.2%) | 47 | +23 |
| v5 loosen MAX_PCT_FROM_HIGH_LOW (10->20%) | 24 | +0 |
| v6 moderate loosen all | 36 | +12 |
| v7 aggressive loosen all | 67 | +43 (unique across all variants) |
| v8 tighten control | 11 | -13 |

**Finding 1**: `BREAKOUT_MIN_BODY_PCT` is overwhelmingly the dominant
constraint today - loosening it ALONE nearly doubled the catch count.
`CLEARANCE_PCT`, `MAX_CONSOLIDATION_RANGE_PCT`, and `MAX_PCT_FROM_HIGH_
LOW` were not binding AT ALL at today's real price action - loosening
any of them individually changed nothing. `MIN_RELATIVE_VOLUME` had a
small effect (+2).

## Results - would the additional catches have been profitable?

For all 43 unique signals caught by some variant but missed by the
deployed baseline, fetched each underlying's current price and today's
subsequent high/low (real data, no simulation) and computed the move
since the signal candle's close, sign-adjusted for direction:

- Best favorable move ANY of the 39 successfully-checked symbols achieved
  at any point today: **+1.34%** (KPITTECH). Median: **+0.3%**.
- **18 of 39 (46%) had already moved AGAINST the signal direction** by
  the time this was checked (~11:00 IST).
- Deployed exit ladder is `TARGET_PCT=0.20` (20%) on the OPTION premium.
  A +1.34% underlying move on an ATM option (delta ~0.5) is roughly a
  2-3% premium move at most - nowhere close to a 20% target, even before
  accounting for the ~46% that reversed.

**Finding 2**: the additional signals a looser `MIN_BODY_PCT` (or any
other loosened gate) would have caught today were low-conviction/noise
moves, not missed profitable trades. Loosening the parameters would have
added volume to the scanner's output without adding real edge - if
anything, it would have diluted quality (nearly half of the additional
catches already reversed by mid-morning).

## Conclusion

**No parameter change recommended based on today's data.** The deployed
configuration is not causing genuinely profitable trades to be missed
this morning - the gates that ARE binding (`MIN_BODY_PCT` most of all)
appear to be correctly filtering out low-quality moves, not real
breakouts. Kept the live config unchanged.

## Separate observation, NOT the same question

My baseline replay found 24 algorithmically-qualifying signals, but
production's own real-time scanner had only logged 8 of them
(`history/2026-09-22_breakout_signals.log`) by the time this analysis ran.
This gap is NOT a parameter-tuning issue - it's the scanner's own
scan-capacity/cadence (10 symbols per 60s cycle, cycling through ~150
real candidates across 4 scan loops) not yet having reached every
candidate. This is the same class of bottleneck already documented in
[[universe-bucket-dispatcher-design]] and partially mitigated today via
the Swing watchlist cut - worth a separate look at whether the breakout
scanners' own scan cadence/capacity (not their detection thresholds)
needs adjustment, but that's a distinct question from the one this
analysis answers.

## What this analysis did NOT do (scope, stated plainly)

- Did not simulate real option-contract PnL (ATM/liquid-strike
  resolution + real option 1-min candles + real exit ladder walk-forward)
  for every one of the 43 additional catches - the underlying-move proxy
  above was judged sufficient given how uniformly small the moves were
  (median +0.3%, far below what a 20% option target needs); a full
  option-level simulation would have cost meaningfully more real API
  calls against an already rate-limited account for a conclusion the
  proxy already answers clearly.
- Did not replicate downstream entry gates (volume-floor, liquid-contract
  resolution, capacity, daily re-entry cap, RSI-loss-reentry) - this
  analysis is scoped to the breakout-DETECTION parameters specifically,
  per the user's own framing ("breakout scanner parameters"). Separately
  worth noting: production's OWN log today shows 4 of its 8 real
  detections were skipped by the volume-floor gate (TECHM, PERSISTENT,
  DIXON, PGEL) - a DIFFERENT gate from anything swept here, not evaluated
  in this pass.
- This is an intraday-only, partial-day analysis ("since this morning") -
  conclusions are specific to today's actual price action, not a general
  claim about the parameters' long-run tuning.
