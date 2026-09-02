# Design + backtest: Relative Strength (vs NIFTY) momentum ranking

**Status: Backtested, 2 Sep 2026 — NO usable edge found; if anything a
mild signal in the OPPOSITE direction (see "Honest read" below).** Not
wired into production (`traderBoy`'s `Swing/relative_strength.py` holds
the implementation, backtest-only). Built as the first of two "different
signal ideas" the user asked to explore after putting
`designs/hhhl-momentum-continuation.md` on hold.

## The idea

The classic, well-documented momentum-persistence effect (academic
"momentum factor" literature; IBD's own "Relative Strength Rating"): a
stock that has already been outperforming the broad market over a recent
trailing window tends, on average, to keep doing so over the near term.
Deliberately a DIFFERENT KIND of signal from the HH/HL approach - no
chart-pattern geometry, no fractal swing detection, just one plain number
(trailing excess return vs NIFTY) with ONE tunable parameter (the
lookback window), chosen specifically to be lower-overfitting-risk than
HH/HL's four-weight composite, given what that signal's own tuning sweep
already showed about trusting a many-parameter score on a modest sample.

## Implementation

`Swing/relative_strength.py` (pure, unit-tested - `tests/test_swing_
relative_strength.py`, 6 scenarios): `align_series_by_date` (keeps only
dates present in BOTH the stock's and NIFTY's own daily series - protects
against a data gap like the MOTHERSON one found during the HH/HL backtest
silently misaligning two different days' closes against each other) and
`relative_strength_score` (excess % return over `lookback_days` trading
days, ending at "yesterday" relative to the scoring point - never
today's own still-forming day, matching production's own point-in-time
discipline for daily context).

## Backtest methodology (`traderBoy/backtest_relative_strength.py`)

Reused the IDENTICAL 782 real Swing entry-signal events already found and
cached for the HH/HL backtest (same entry rule, same forward-return
definition) - this result is directly, apples-to-apples comparable to
that one. Only one new fetch: NIFTY's own daily OHLC (security_id 13,
IDX_I/INDEX segment - confirmed working, same endpoint the "false alarm"
investigation already cleared). Swept `lookback_days` in {10, 20, 40}.

## Results

| Lookback | n | Correlation | Low avg return | High avg return | Horse race |
|---|---|---|---|---|---|
| 10 days | 657 | −0.061 | +0.16% | −0.00% | 30.8% (n=52) |
| 20 days | 535 | −0.026 | +0.16% | −0.01% | 42.9% (n=42) |
| 40 days | 272 | −0.003 | +0.38% | +0.21% | 36.4% (n=22) |

Every single lookback window shows a **negative or ~zero correlation**,
and the same-day horse race is **below 50% at every window** (30.8% /
42.9% / 36.4%) - the LOWER-RS candidate outperformed the higher-RS one
MORE often than not, consistently across all three windows, not just one
noisy outlier. The "Low" tercile beats the "High" tercile on average
return at every single lookback too.

## Honest read

**No usable edge, and the consistency of the inverse direction across
all three independent lookback windows makes this look like more than
pure noise** (unlike the HH/HL tuning sweep's single 67% outlier out of
36 scans, which had every hallmark of a multiple-comparisons artifact -
this result points the same wrong-for-momentum direction at 10, 20, AND
40 days, with the correlation weakening monotonically as the window
lengthens toward zero).

**A plausible, sourced explanation, not proven, but worth recording
rather than shrugging off as unexplained noise**: this measures RS at the
EXACT moment Swing's own Supertrend-crossover entry signal fires - i.e.
right as a stock is ALREADY turning up. A stock that had already been the
market's strongest recent outperformer by the time that crossover fires
may be more "extended" (a similar idea to `momentum_signal.py`'s own
extension-penalty component), while a stock that had been LAGGING and is
only now catching up may have more room left to run. This lines up with
the well-documented short-term reversal literature (Jegadeesh 1990 and
others - momentum's own well-known "sibling" effect operates at these
same few-week horizons in the opposite direction from the multi-month
momentum effect RS is usually built to capture) rather than a genuine
data or methodology bug - the 10/20/40-day windows tested here sit
squarely in that shorter reversal-prone zone, not the 3-12 month
horizon most momentum-factor research actually validates.

**Practical takeaway**: don't build this in as a re-ranking signal as
tested. If pursued further, the interesting next test isn't a parameter
retune of THIS window range (the monotonic weakening toward 40 days
already suggests going even LONGER, toward the 3-12 month range the
literature actually validates, rather than shorter) - but that's a
genuinely different backtest, not a tweak of this one, and NIFTY-relative
momentum over 3-12 months isn't obviously the right frame for a strategy
whose own holding period is days, not months.
