# Design/backtest: MA-ribbon-expansion trigger as a STANDALONE entry signal

**Status: backtested (25-symbol smoke test, 23 Sep 2026) — no edge found
when fired unfiltered/standalone. Not deployed, not recommended as-is.**
See "Next steps" below before spending more effort here without first
addressing the false-positive-rate problem this run surfaced.

## Where this came from

User showed a chart (tight multi-EMA ribbon consolidation collapsing into
a sharp breakout candle, ribbon then fanning out) on 23 Sep 2026 and asked
for a function that detects this pattern across stocks on 5-min candles
and finds "signals at the exact moment to trade". That scoring function
already existed — `ribbon_score.score_ribbon_expansion()` (see
[[ribbon-switch-shadow]] for its build history) — but its only live role
is **re-ranking candidates that already arrived via a Chartink webhook
alert** (`RIBBON_RANKING_ENABLED`). It had never been tested as an
independent entry source scanning a stock universe on its own. This
backtest is that test.

## What was built

`traderBoy/backtest_ribbon_trigger_standalone_entry.py` (full docstring
has all the caveats in detail, summarized here):

- Scans F&O-eligible NSE symbols on continuous 5-min candles.
- For every genuine "close just crossed the whole 9/20/50/100 EMA ribbon"
  event, calls the real, unmodified `score_ribbon_expansion()` at each bar
  within its own `DEFAULT_LOOKBACK_BARS` (15-bar) confirmation window and
  fires ONE signal at the first bar the already-deployed `MIN_ENTRY_SCORE`
  (50.0) threshold is cleared — the same bar the live re-ranker would call
  "worth entering".
- Simulates forward on the **underlying's own price** (not real option
  premium — a deliberate first-pass simplification, see script docstring
  caveat 1) with a configurable target/stop (default 1.0%/0.5%
  underlying), capped to the same trading day.
- CE/bullish only (`score_ribbon_breakdown`/PE mirror is still unbacktested
  per `ribbon_score.py`'s own docstring — out of scope here too).

Efficiency note for anyone extending this: the script does NOT call
`score_ribbon_expansion` on every bar (that would be O(n²) per symbol,
too slow at full-universe scale) — it precomputes the ribbon EMAs once
(causal, so mathematically identical to slicing) purely to cheaply locate
candidate trigger bars, then calls the real scoring function only inside
the narrow window after each candidate. The actual score for every
reported signal is still `score_ribbon_expansion()`'s own number,
unchanged.

## Results — 25-symbol smoke test, 30 calendar days (~20 trading days)

| Metric | Value |
|---|---|
| Signals | 785 (≈31/symbol over ~20 trading days — see "the real finding" below) |
| TARGET hit (1.0%) | 119 (15.2%) |
| STOP hit (0.5%) | 342 (43.6%) |
| EOD (neither) | 324 (41.3%) |
| Win rate (return>0 at exit) | 36.4% |
| Avg return at exit | **-0.037%** |
| Avg MFE / MAE | +0.422% / -0.397% |
| +15m / +30m / +60m / +120m avg return | -0.012% / +0.002% / -0.020% / -0.005% |
| +15m / +30m / +60m / +120m win-rate | 40.3% / 45.0% / 45.3% / 43.5% |

Top signal-count symbols: AXISBANK (54), APOLLOHOSP (44), ADANIPORTS (43),
ADANIENT (41), AUROPHARMA (41) — i.e. roughly one signal every ~1-2
trading days *per symbol*, across the board.

Raw results: `traderBoy/backtest_ribbon_trigger_standalone_entry_results.json`.

## The real finding: this isn't a rare/high-quality setup when unfiltered

785 signals from 25 symbols in ~20 days is far too frequent to be the
"textbook" squeeze-then-breakout the user's chart showed — that pattern
should be an occasional, high-conviction event, not something firing
roughly every 1-2 days per stock. Two things explain the gap between "the
chart that inspired this" and "what actually fired":

1. **`MIN_ENTRY_SCORE=50` was calibrated for a different job.** It answers
   "given this candidate already survived some OTHER filter (a Chartink
   screener match) and is being compared against its alert-batch-mates,
   is it still worth entering at all" — not "out of every unfiltered
   ribbon-cross in the whole market, is this one of the rare good ones".
   Used as the latter, it's far too permissive.
2. **Compression is only a 25%-weighted, soft component of the total
   score**, not a hard gate. A candidate can score >=50 from fan-out +
   confirmation alone even when `compression_score` was near zero — i.e.
   the ribbon never actually squeezed tight beforehand, so what fired
   wasn't really the "tight coil -> breakout" shape from the chart at
   all, just an ordinary bullish EMA stack.

Consistent with that read: net expectancy is **negative** at the default
1%/0.5% target/stop (2:1 reward:risk needs >33% target-hit rate to break
even before costs; only 15.2% actually hit target vs 43.6% stopped out),
and win rate stays below 50% at every forward horizon out to +120m. On
this sample, the *unfiltered* trigger has no standalone edge — this is
not a reason to distrust `score_ribbon_expansion()` itself (it's already
proven useful in its actual, narrower live role — re-ranking within a
pre-filtered alert batch, see [[ribbon-switch-shadow]]), it's a reason to
distrust firing it on raw, unfiltered candles across the whole market.

## Next steps (not yet done)

- **Require a real compression gate**, not just a soft-weighted component
  — e.g. only evaluate a candidate at all if `compression_score` (or the
  raw tightest-ribbon-width%) cleared some real minimum, matching what
  the user's chart actually showed (a GENUINE squeeze, not just "ribbon
  happens to be stacked").
- **Raise `MIN_ENTRY_SCORE`** substantially above the re-ranking-tuned 50
  for standalone use (e.g. sweep 60/70/80 against this same 25-symbol
  sample first — cheap, no new API calls needed, the fetched candle data
  is reusable).
- **Run the full ~210-symbol F&O universe** before drawing a firm
  conclusion — 25 symbols/785 signals is a reasonable smoke-test sample
  but not the final word.
- **Test it in its actually-intended role** — layered on top of an
  existing screener-filtered candidate list (what it already does live),
  not as a replacement for one. This backtest deliberately tested the
  more ambitious "can this replace a screener entirely" question first,
  since that was the literal ask; the answer here is "not without a real
  compression gate and a higher bar."
