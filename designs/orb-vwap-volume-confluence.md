# Design + backtest: ORB + VWAP + Volume confluence re-ranking signal

**Status: Backtested, 2 Sep 2026 — NO usable edge found** (third signal
in a row - see the cross-signal synthesis in
`designs/hhhl-momentum-continuation.md`'s sibling files for the full
picture). Not wired into production (`traderBoy`'s
`Swing/orb_vwap_signal.py` holds the implementation, backtest-only). The
second of two "different signal ideas" explored after putting HH/HL on
hold (Relative Strength was the first, also no edge - see
`designs/relative-strength-momentum-ranking.md`).

## The idea

Sourced, cited in `learnings/intraday-options-trading/momentum-entry-
conditions-research.md`, never built anywhere in this codebase until
now: a commonly-cited "good" intraday momentum confirmation requires ALL
of - a candle CLOSING beyond the day's own opening range (not just an
intra-bar poke through it), price above VWAP (broad participation, not
just a spike), and volume meaningfully above its own recent average. A
third, genuinely different mechanism from both other signals tried: no
chart-pattern swing geometry (unlike HH/HL), no cross-sectional benchmark
comparison (unlike Relative Strength) - a purely intraday, single-day,
session-anchored confluence check.

## Implementation

`Swing/orb_vwap_signal.py` (pure, unit-tested - `tests/test_swing_orb_
vwap_signal.py`, 7 scenarios): `day_boundaries`/`opening_range` (the
first N 5-min bars of each trading day), `vwap_series` (session-anchored,
resets every day - NOT a running average across days), `orb_breakout_
score` (0 until price genuinely CLOSES beyond the range's high, then
scales up in ATR units), `vwap_confluence_score` (same shape, smaller ATR
cap since VWAP is a same-session average), and `rvol_score` reused
as-is from `momentum_signal.py`. Combined via `orb_vwap_composite_score`.

## Backtest methodology (`traderBoy/backtest_orb_vwap_signal.py`)

Reused the IDENTICAL 782 real Swing entry-signal events already used for
BOTH prior backtests - directly, apples-to-apples comparable to both.
Swept the opening-range window: 3/6/12 5-min bars (15/30/60 minutes).

## Results

| OR window | n | Correlation | Low avg return | High avg return | Horse race |
|---|---|---|---|---|---|
| 15 min | 737 | −0.031 | +0.11% | +0.10% | 40.7% (n=54) |
| 30 min | 737 | −0.024 | +0.12% | +0.10% | 40.0% (n=55) |
| 60 min | 737 | −0.013 | +0.13% | +0.16% | 47.4% (n=57) |

Negative correlation at every window tested, and the same-day horse race
is below 50% at every window too (though closer to a coin flip at the
60-minute window than the other two, unlike Relative Strength's much
stronger inversion).

## Honest read

No usable edge, same verdict as Relative Strength. Weaker in magnitude
than that signal's own inversion (correlation closer to zero, horse race
closer to 50%) but pointing the same general direction (slightly
negative/no better than chance) rather than the positive-but-weak
direction HH/HL showed. See the cross-signal synthesis below for what
three consistent null-to-negative results across three genuinely
different signal families actually implies.

## Cross-signal synthesis (all three signals tested against the IDENTICAL 782 entries)

| Signal | Best correlation | Best-correlation horse race | Direction |
|---|---|---|---|
| HH/HL momentum-continuation | +0.071 (k=2) | 50.9% (coin flip) | weak positive |
| Relative Strength vs NIFTY | −0.003 (40-day) | 36.4% | negative, consistent |
| ORB + VWAP + Volume | −0.013 (60-min OR) | 47.4% | negative, weak |

**None of the three cleared a bar that would justify production
deployment.** The one directionally-positive result (HH/HL) didn't
survive its own parameter-tuning sweep with a robust horse-race number at
the same setting that produced the best correlation. Taken together, this
is a real, useful negative result about the specific dataset/period
tested (last ~90 days, 22-symbol watchlist, forward return measured to
the Supertrend-reversal exit) - it does NOT prove no such signal could
ever help Swing's own ranking, but it does mean none of these three
reasonably-well-sourced, honestly-implemented approaches found one here.
**Recommended before trying a FOURTH signal idea**: question whether the
bottleneck is the signal ideas themselves or the evaluation setup (a
longer/different backtest window, a less noisy forward-return definition,
or accepting that Swing's existing freshness+volume ranking may already
be adequate for its own purposes) - see `learnings/backtest-methodology.md`
for the standing caution about drawing conclusions from one narrow window.
