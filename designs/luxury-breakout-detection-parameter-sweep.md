Status: BACKTEST ONLY - a 27-combination parameter sweep over the
breakout-signal DETECTION rules (not entry gates or exit stack, both
unchanged and real). Best combination found is dramatically better than
the current live thresholds on this 14-day sample (+Rs175,346.60 vs the
live baseline's +Rs56,089.75), but nothing has been changed live -
awaiting the user's decision on whether/what to adopt.

# Luxury breakout-signal detection parameter sweep

## What was asked

Tweak/vary the breakout-signal DETECTION parameters (the 7 rules on
5-min candles - see [[breakout-scanner-vs-real-pnl]] for their origin)
and find out whether looser/tighter values produce more or better real
trades for Luxury, trying all reasonable combinations, with a detailed
report across every version tested.

## Scope

Swept the 3 most impactful "detection sensitivity" thresholds, 3 values
each = 27 combinations (current live values included as one of them):

- `BREAKOUT_MIN_BODY_PCT`: 0.5 / 1.0 (live) / 1.5
- `BREAKOUT_CLEARANCE_PCT`: 0.3 / 0.5 (live) / 0.75
- `BREAKOUT_MIN_RELATIVE_VOLUME`: 1.2 / 1.5 (live) / 2.0

`BREAKOUT_MAX_CONSOLIDATION_RANGE_PCT` (12%) and `BREAKOUT_MAX_PCT_FROM_
HIGH_LOW` (10%) were held fixed at their live values - a full 5-parameter
grid would be 5x the size (135+ combos) for two dials that are more
structural/secondary than "how sensitive is the trigger," per the
earlier diagnostic work in [[breakout-scanner-vs-real-pnl]].

PE not included - Luxury's own scan_names are all CE-tagged in this
dataset (confirmed empirically, see [[luxury-signal-gated-live-
simulation]]).

**Everything else held IDENTICAL to the just-updated live config**: same
real entry gates (daily re-entry cap, loss-repeat block, cross-strategy
claim, capacity + burst slot, liquid-contract resolution), same real
exit stack including the 21 Sep risk-parameter raise (MAX_LOSS_PER_TRADE
5500/3100 CE, 4500/2600 PE; PROFIT_PROTECTION_THRESHOLD 5000/2500;
GIVEBACK_PCT 8%). Only the signal-DETECTION thresholds vary between
combos - `traderBoy/backtest_luxury_signal_sweep.py`.

## Methodology note: how 27 combos ran efficiently

Fetched every Luxury CE-bucket symbol's real 5-min + daily candle data
ONCE (reusing [[breakout-scanner-vs-real-pnl]]'s own per-(symbol,day)
caching, which never depends on the swept thresholds), then re-evaluated
the 7-rule check against that SAME cached data 27 times via monkey-
patching the threshold globals before each pass - the detection phase
itself cost nothing extra after the first fetch. The downstream real-
gate + real-exit simulation (which does need fresh option-contract data)
reused the same per-(symbol,day,contract) caching `backtest_luxury_
signal_gated_live.py` already has, so overlapping signals across nearby
parameter combos were priced only once.

## Full sweep results (ranked by total PnL)

| Body% | Clear% | RelVol | Signals | Entered | Wins | Win Rate | Total PnL | vs Real |
|---|---|---|---|---|---|---|---|---|
| 0.50 | 0.30 | 1.20 | 72 | 59 | 55 | 93.2% | **175,346.60** | 193,255.95 |
| 0.50 | 0.50 | 1.20 | 70 | 57 | 53 | 93.0% | 174,385.20 | 192,294.55 |
| 0.50 | 0.30 | 1.50 | 69 | 58 | 54 | 93.1% | 173,634.85 | 191,544.20 |
| 0.50 | 0.50 | 1.50 | 66 | 55 | 52 | 94.5% | 172,673.45 | 190,582.80 |
| 0.50 | 0.50 | 2.00 | 60 | 52 | 49 | 94.2% | 159,531.95 | 177,441.30 |
| 0.50 | 0.30 | 2.00 | 62 | 54 | 50 | 92.6% | 158,723.35 | 176,632.70 |
| 0.50 | 0.75 | 1.20 | 43 | 36 | 32 | 88.9% | 111,939.70 | 129,849.05 |
| 0.50 | 0.75 | 1.50 | 40 | 34 | 31 | 91.2% | 103,813.95 | 121,723.30 |
| 0.50 | 0.75 | 2.00 | 37 | 32 | 29 | 90.6% | 98,380.20 | 116,289.55 |
| 1.00 | 0.30 | 1.20 | 21 | 20 | 18 | 90.0% | 63,809.75 | 81,719.10 |
| 1.00 | 0.50 | 1.20 | 20 | 19 | 17 | 89.5% | 62,039.75 | 79,949.10 |
| 1.00 | 0.30 | 1.50 | 20 | 19 | 17 | 89.5% | 57,859.75 | 75,769.10 |
| **1.00** | **0.50** | **1.50** | **19** | **18** | **16** | **88.9%** | **56,089.75** | **73,999.10** *(current live)* |
| 1.00 | 0.30 | 2.00 | 18 | 17 | 15 | 88.2% | 53,994.75 | 71,904.10 |
| 1.00 | 0.50 | 2.00 | 18 | 17 | 15 | 88.2% | 53,994.75 | 71,904.10 |
| 1.00 | 0.75 | 1.20 | 17 | 16 | 14 | 87.5% | 52,757.25 | 70,666.60 |
| 1.00 | 0.75 | 1.50 | 16 | 15 | 13 | 86.7% | 46,807.25 | 64,716.60 |
| 1.00 | 0.75 | 2.00 | 16 | 15 | 13 | 86.7% | 46,807.25 | 64,716.60 |
| 1.50 | 0.30 | 1.20 | 6 | 5 | 4 | 80.0% | 8,107.75 | 26,017.10 |
| 1.50 | 0.50 | 1.20 | 6 | 5 | 4 | 80.0% | 8,107.75 | 26,017.10 |
| 1.50 | 0.75 | 1.20 | 6 | 5 | 4 | 80.0% | 8,107.75 | 26,017.10 |
| 1.50 | 0.30 | 1.50 | 5 | 4 | 3 | 75.0% | 1,675.00 | 19,584.35 |
| 1.50 | 0.30 | 2.00 | 5 | 4 | 3 | 75.0% | 1,675.00 | 19,584.35 |
| 1.50 | 0.50 | 1.50 | 5 | 4 | 3 | 75.0% | 1,675.00 | 19,584.35 |
| 1.50 | 0.50 | 2.00 | 5 | 4 | 3 | 75.0% | 1,675.00 | 19,584.35 |
| 1.50 | 0.75 | 1.50 | 5 | 4 | 3 | 75.0% | 1,675.00 | 19,584.35 |
| 1.50 | 0.75 | 2.00 | 5 | 4 | 3 | 75.0% | 1,675.00 | 19,584.35 |

Real Luxury baseline over the same 14 days: -Rs17,909.35 (125 trades,
36.0% win rate).

## The pattern: candle-body threshold dominates everything else

`BREAKOUT_MIN_BODY_PCT` alone explains almost the entire spread - every
0.5%-body combo beats every 1.0%-body combo, which beats every 1.5%-body
combo, REGARDLESS of what clearance% or relative-volume is set to. Going
from the live 1.0% down to 0.5% roughly **triples** both signal count
(19->72 raw, 18->59 entered) and total PnL (Rs56k->Rs175k) while the win
rate actually IMPROVED slightly (88.9%->93.2%) rather than degrading -
loosening this specific threshold did not add worse trades on this
sample, it found more of the same quality. Clearance% and relative-
volume matter far less within the ranges tested - moving from 0.5% to
0.3% clearance, or 1.5x to 1.2x relative volume, shifts the total by only
single-digit percentages, not multiples.

## Best combination: candle body >=0.5%, clearance >=0.3%, relative volume >=1.2x

### Sanity-checked before trusting it

- Exit reasons across the 59 trades: TARGET_HIT 49, LIQUIDITY_GUARD_
  ZERO_VOLUME 4, PROFIT_PROTECTION_HIT 4, EMA_CROSS_EXIT 1, MAX_LOSS_HIT
  1 - a healthy, varied mix, not dominated by one mechanism.
- No single outlier trade - the largest individual trade (RBLBANK,
  +Rs9,366.25) is only 5.5% of the combo's total PnL.
- The one MAX_LOSS_HIT (PAYTM, 16 Sep) landed at exactly -Rs5,500 - the
  current live cap, same precision check that's validated every prior
  backtest in this line of work.

### Day-wise (best combo vs. real Luxury, same 14 days)

| Day | Real N | Real PnL | Sim N | Sim Wins | Sim PnL | Delta |
|---|---|---|---|---|---|---|
| 1 Sep | 7 | -3,240.00 | 1 | 1 | 4,560.00 | +7,800.00 |
| 2 Sep | 0 | 0.00 | 4 | 4 | 26,989.85 | +26,989.85 |
| 3 Sep | 25 | -10,229.00 | 10 | 9 | 27,220.45 | +37,449.45 |
| 4 Sep | 12 | -361.50 | 10 | 9 | 32,583.05 | +32,944.55 |
| 8 Sep | 4 | 1,951.25 | 2 | 2 | 6,317.50 | +4,366.25 |
| 9 Sep | 8 | -3,857.50 | 5 | 5 | 16,859.50 | +20,717.00 |
| 10 Sep | 6 | -1,402.50 | 2 | 2 | 5,759.00 | +7,161.50 |
| 11 Sep | 1 | -1,338.75 | 1 | 1 | 3,870.00 | +5,208.75 |
| 16 Sep | 3 | 6,139.25 | 3 | 2 | -561.75 | -6,701.00 |
| 17 Sep | 26 | 5,130.00 | 10 | 10 | 30,507.75 | +25,377.75 |
| 18 Sep | 33 | -10,700.60 | 11 | 10 | 21,241.25 | +31,941.85 |
| **Total** | | **-17,909.35** | | | **175,346.60** | **+193,255.95** |

Every day is net positive for the sim except 16 Sep (the one MAX_LOSS_
HIT day) - the most consistent day-by-day result of any backtest in this
line of work so far.

### Trade-wise (best combo, all 59 trades, chronological)

| Day | Symbol | Time | Entry | Exit | Exit Reason | Qty | PnL |
|---|---|---|---|---|---|---|---|
| 1 Sep | ATHERENERG | 09:40 | 60.80 | 72.96 | TARGET_HIT | 375 | 4,560.00 |
| 2 Sep | IDEA | 09:50 | 0.68 | 0.82 | TARGET_HIT | 71,475 | 9,720.60 |
| 2 Sep | MAHABANK | 10:40 | 2.76 | 3.39 | TARGET_HIT | 6,500 | 4,095.00 |
| 2 Sep | RBLBANK | 11:15 | 10.05 | 13.00 | TARGET_HIT | 3,175 | 9,366.25 |
| 2 Sep | OIL | 12:50 | 13.60 | 16.32 | TARGET_HIT | 1,400 | 3,808.00 |
| 3 Sep | INDIANB | 09:15 | 25.00 | 30.00 | TARGET_HIT | 1,000 | 5,000.00 |
| 3 Sep | DLF | 09:15 | 18.00 | 21.60 | TARGET_HIT | 950 | 3,420.00 |
| 3 Sep | LICHSGFIN | 09:15 | 32.00 | 29.25 | LIQUIDITY_GUARD_ZERO_VOLUME | 1,000 | -2,750.00 |
| 3 Sep | RBLBANK | 09:15 | 9.70 | 11.64 | TARGET_HIT | 3,175 | 6,159.50 |
| 3 Sep | MAHABANK | 09:15 | 1.95 | 2.34 | TARGET_HIT | 6,500 | 2,535.00 |
| 3 Sep | BANKINDIA | 09:15 | 3.63 | 4.36 | TARGET_HIT | 5,200 | 3,775.20 |
| 3 Sep | SAGILITY | 09:20 | 1.70 | 1.83 | LIQUIDITY_GUARD_ZERO_VOLUME | 12,000 | 1,560.00 |
| 3 Sep | COALINDIA | 09:25 | 4.85 | 5.82 | TARGET_HIT | 1,350 | 1,309.50 |
| 3 Sep | GODREJPROP | 10:15 | 59.05 | 70.86 | TARGET_HIT | 325 | 3,838.25 |
| 3 Sep | BHEL | 11:15 | 14.00 | 14.90 | PROFIT_PROTECTION_HIT | 2,625 | 2,373.00 |
| 4 Sep | LICHSGFIN | 09:15 | 18.65 | 19.00 | LIQUIDITY_GUARD_ZERO_VOLUME | 1,000 | 350.00 |
| 4 Sep | RELIANCE | 09:15 | 20.95 | 25.14 | TARGET_HIT | 500 | 2,095.00 |
| 4 Sep | PRESTIGE | 09:15 | 50.00 | 49.55 | EMA_CROSS_EXIT | 450 | -202.50 |
| 4 Sep | CDSL | 09:15 | 41.05 | 49.26 | TARGET_HIT | 475 | 3,899.75 |
| 4 Sep | PAYTM | 09:30 | 47.00 | 56.40 | TARGET_HIT | 725 | 6,815.00 |
| 4 Sep | PNBHOUSING | 10:45 | 36.00 | 44.85 | TARGET_HIT | 650 | 5,752.50 |
| 4 Sep | IDEA | 11:00 | 0.62 | 0.74 | TARGET_HIT | 71,475 | 8,862.90 |
| 4 Sep | SWIGGY | 11:25 | 10.75 | 11.82 | PROFIT_PROTECTION_HIT | 1,825 | 1,956.40 |
| 4 Sep | COCHINSHIP | 11:55 | 43.45 | 45.75 | PROFIT_PROTECTION_HIT | 400 | 920.00 |
| 4 Sep | TATASTEEL | 13:40 | 3.88 | 4.66 | TARGET_HIT | 2,750 | 2,134.00 |
| 8 Sep | HAL | 09:15 | 100.00 | 120.00 | TARGET_HIT | 150 | 3,000.00 |
| 8 Sep | GVT&D | 09:25 | 132.70 | 159.24 | TARGET_HIT | 125 | 3,317.50 |
| 9 Sep | COALINDIA | 09:15 | 4.95 | 6.50 | TARGET_HIT | 1,350 | 2,092.50 |
| 9 Sep | PAYTM | 09:15 | 48.95 | 58.74 | TARGET_HIT | 725 | 7,097.75 |
| 9 Sep | OIL | 09:15 | 10.10 | 12.12 | TARGET_HIT | 1,400 | 2,828.00 |
| 9 Sep | ADANIPORTS | 10:20 | 28.15 | 33.78 | TARGET_HIT | 475 | 2,674.25 |
| 9 Sep | TATASTEEL | 10:20 | 3.94 | 4.73 | TARGET_HIT | 2,750 | 2,167.00 |
| 10 Sep | OIL | 09:15 | 10.40 | 12.48 | TARGET_HIT | 1,400 | 2,912.00 |
| 10 Sep | PNBHOUSING | 14:05 | 21.90 | 26.28 | TARGET_HIT | 650 | 2,847.00 |
| 11 Sep | MCX | 10:35 | 86.00 | 103.20 | TARGET_HIT | 225 | 3,870.00 |
| 16 Sep | YESBANK | 09:15 | 0.45 | 0.54 | TARGET_HIT | 31,100 | 2,799.00 |
| 16 Sep | PAYTM | 09:15 | 84.15 | 76.56 | **MAX_LOSS_HIT** | 725 | **-5,500.00** |
| 16 Sep | PATANJALI | 11:15 | 9.95 | 11.94 | TARGET_HIT | 1,075 | 2,139.25 |
| 17 Sep | PNB | 09:15 | 1.64 | 1.97 | TARGET_HIT | 8,000 | 2,624.00 |
| 17 Sep | LICHSGFIN | 09:15 | 8.85 | 10.62 | TARGET_HIT | 1,000 | 1,770.00 |
| 17 Sep | LAURUSLABS | 09:15 | 35.00 | 42.00 | TARGET_HIT | 850 | 5,950.00 |
| 17 Sep | PAYTM | 09:25 | 42.20 | 50.64 | TARGET_HIT | 725 | 6,119.00 |
| 17 Sep | MAHABANK | 09:35 | 1.80 | 2.16 | TARGET_HIT | 6,500 | 2,340.00 |
| 17 Sep | MCX | 09:35 | 78.30 | 93.96 | TARGET_HIT | 225 | 3,523.50 |
| 17 Sep | HDFCLIFE | 10:00 | 9.35 | 11.22 | TARGET_HIT | 1,100 | 2,057.00 |
| 17 Sep | HYUNDAI | 11:50 | 37.25 | 44.70 | TARGET_HIT | 275 | 2,048.75 |
| 17 Sep | MFSL | 12:40 | 28.60 | 34.32 | TARGET_HIT | 400 | 2,288.00 |
| 17 Sep | TATASTEEL | 14:40 | 3.01 | 3.66 | TARGET_HIT | 2,750 | 1,787.50 |
| 18 Sep | CGPOWER | 09:15 | 11.80 | 14.16 | TARGET_HIT | 850 | 2,006.00 |
| 18 Sep | GVT&D | 09:15 | 84.80 | 101.76 | TARGET_HIT | 125 | 2,120.00 |
| 18 Sep | BHEL | 09:15 | 6.85 | 8.22 | TARGET_HIT | 2,625 | 3,596.25 |
| 18 Sep | ZYDUSLIFE | 09:15 | 17.00 | 20.40 | TARGET_HIT | 900 | 3,060.00 |
| 18 Sep | JUBLFOOD | 09:30 | 7.30 | 8.76 | TARGET_HIT | 1,250 | 1,825.00 |
| 18 Sep | SAGILITY | 09:50 | 0.75 | 0.90 | TARGET_HIT | 12,000 | 1,800.00 |
| 18 Sep | INDHOTEL | 09:50 | 8.60 | 10.32 | TARGET_HIT | 1,000 | 1,720.00 |
| 18 Sep | APLAPOLLO | 10:15 | 30.25 | 36.30 | TARGET_HIT | 350 | 2,117.50 |
| 18 Sep | PATANJALI | 11:20 | 7.10 | 8.52 | TARGET_HIT | 1,075 | 1,526.50 |
| 18 Sep | MAHABANK | 11:55 | 1.34 | 1.34 | LIQUIDITY_GUARD_ZERO_VOLUME | 6,500 | 0.00 |
| 18 Sep | SONACOMS | 12:50 | 14.00 | 15.20 | PROFIT_PROTECTION_HIT | 1,225 | 1,470.00 |

## Caveats before adopting anything

- **Still 14 days of data, now spread across 59 trades instead of 18** -
  a bigger sample within the same short window, not a longer window.
  The consistent day-by-day positivity is reassuring but this remains a
  single historical stretch.
- **A looser candle-body threshold (0.5% vs 1.0%) means triggering on
  smaller, less decisive single-candle moves.** It happened to still find
  quality setups in THIS 14-day window (which included several genuinely
  strong trending days), but a smaller minimum move is inherently closer
  to noise in a choppier regime this sample may not represent. The clean
  monotonic result here is not automatically evidence it generalizes.
- Every other real production gate/exit is unchanged and validated
  elsewhere in this repo - this sweep isolates the DETECTION thresholds
  specifically, nothing else was varied.

## Deployed (21 Sep 2026)

User reviewed this table and asked to adopt the #1 combo -
`clearance=0.3, body=0.5, relvol=1.2` (the loosest of the 27, 59
trades / 93.2% win rate / +Rs175,346.60). Deployed as env overrides
(code defaults of 0.5/1.0/1.5 are left untouched, per this repo's
existing convention of changing `.env` rather than code when only the
live-deployed *value* is changing):

```
LUXURY_BREAKOUT_CLEARANCE_PCT=0.3
LUXURY_BREAKOUT_MIN_BODY_PCT=0.5
LUXURY_BREAKOUT_MIN_RELATIVE_VOLUME=1.2
```

Applied to local `.env`, scp'd to the droplet's `.env`, and the Luxury
service restarted to pick it up (positions checked empty before/after,
per the standing restart-safety checklist). Given the caveat above
about a looser body threshold trending closer to noise, this is worth
re-running the same sweep methodology against fresh alert history in
a few weeks to confirm the ranking holds outside this original 14-day
window.

## What's still open

Re-validate this ranking against a later, non-overlapping stretch of
Luxury alert history once enough new alerts accumulate, since the
caveats above (single 14-day window, looser body threshold trending
closer to noise) were never fully resolved - just accepted as the
tradeoff for adopting the best combo found so far.
