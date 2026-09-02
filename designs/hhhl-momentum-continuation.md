# Design + backtest: HH/HL momentum-continuation re-ranking signal

**Status: Backtested AND parameter-tuned, 2 Sep 2026 — tuning did NOT
rescue this into a deployable signal; if anything it surfaced a reason
for MORE caution, not less (see "Parameter sweep" section below).** Not
wired into production (`traderBoy`'s `Swing/momentum_signal.py` holds the
implementation, backtest-only, no import from `trading_engine.py` yet).
User's own framing throughout: "show me results first... then we will
think of deploying it or not based on profits or higher profitability
signal detection" / "tune the parameters and re-test."

## The idea (user's own words, 2 Sep 2026)

"There is a higher high, higher low kind of a formation over a five
minute candle than a stock price is rising over daily time frame... before
rising, there would be a kind of a consolidation period" — proposed as an
**additional pre-filter/context layer ahead of Swing's existing Supertrend
crossover entry trigger, re-ranking (soft signal) rather than gating**.

This maps onto standard, already-researched technical-analysis concepts,
not a novel pattern: Dow Theory's own definition of an uptrend (ascending
swing highs + ascending swing lows), preceded by a tightening range —
conceptually the same as a **flag/pennant continuation** pattern (see
`learnings/technical-patterns/classic-chart-patterns.md`) or an intraday-
scale analogue of VCP's "volume/range dry-up" (`learnings/technical-
patterns/vcp.md`), just measured via ATR contraction instead of
Minervini's own multi-week contraction-depth sequence (that pattern is
explicitly daily-chart-only per that file's own guidance).

## Implementation (backtest-only)

`Swing/momentum_signal.py` — pure, unit-tested functions
(`tests/test_swing_momentum_signal.py`, 16 scenarios, all passing):

- **Fractal swing detection**: `find_fractal_swing_highs`/`_lows(series,
  k)` — a bar is a swing point if it's strictly the highest/lowest of the
  `k` bars on EACH side. `k` is a free, sweepable parameter (user's own
  note: "this can further be optimized also") — this backtest used `k=2`
  throughout, not yet swept.
- **Breakout detection**: `detect_hh_hl_breakouts` — walks the series
  chronologically, respecting each swing point's own `k`-bar confirmation
  lag (a live system reading this rule has the identical lag — you can't
  know bar i was a swing high until k bars later print without breaking
  it), and fires the FIRST bar price closes above a pivot where the last
  two confirmed swing highs are ascending AND the last two confirmed
  swing lows are also ascending.
- **Four soft-score components**, each `[0,1]` (or None if not enough
  history):
  - `coil_tightness_score` — ATR in the 12 bars before breakout vs a
    60-bar baseline; 1.0 = maximally tight coil.
  - `rvol_score` — breakout bar's volume ÷ the historical average volume
    for that SAME 5-min bar-of-day (time-of-day normalized, per the
    sourced approach in `learnings/intraday-options-trading/momentum-
    entry-conditions-research.md`).
  - `freshness_score` — 1.0 right at breakout, linearly decaying to 0 by
    one trading day later (same "just happened beats an hour ago"
    principle `traderBoy`'s own live `_entry_candidate_rank_key` already
    uses).
  - `extension_score` — penalizes a candidate that's already run far
    above its own pivot by the time it's scored (measured in ATR
    multiples, decaying to 0 by 2 ATRs above pivot).
- `composite_momentum_score` — a weighted sum (defaults: coil 0.30,
  freshness 0.25, RVOL 0.25, extension 0.20 — untuned starting point, NOT
  a fit result).

## Backtest methodology (`traderBoy/backtest_momentum_signal.py`)

Read-only, real Dhan data, no order placement — same discipline
`bt_common.py` already follows.

- **Universe**: all 22 symbols in `data/watchlist` as of 2 Sep 2026 (the
  actual live Swing watchlist). MOTHERSON returned zero candles from Dhan
  (security-ID resolution issue, not investigated further — 1 of 22
  symbols, doesn't change the overall read).
- **Data**: real 5-min AND 1-min intraday candles, ~90 calendar days back
  — confirmed empirically to be Dhan's actual limit for this account for
  BOTH intervals (the `dhanhq` SDK's own docstring claim that 1-min data
  only goes back "5 trading days" is WRONG — it goes back the same ~90
  days as 5-min, confirmed via direct testing, ~23,295 1-min candles for
  a 90-day RELIANCE pull).
- **Entry trigger replicated EXACTLY as `traderBoy`'s own production rule**
  (`Swing/trading_engine.py._evaluate_watchlist_entry_signal`): price
  confirmed at/above previous day's close (latches for the day), 5-min
  close CROSSED ABOVE its Supertrend, 1-min close at-or-crossed-above its
  own Supertrend — same `SUPERTREND_PERIOD=10`/`MULTIPLIER=3.0` Swing
  actually runs live. This isolates the ONE variable being tested (does
  the new score correctly rank real entry-signal candidates) rather than
  inventing a different, easier-to-detect entry rule.
- **Forward return** = the underlying's own % move from entry to the same
  Supertrend-reversal exit rule (or a 150-bar cap) — a proxy for setup
  quality via the underlying's price move, NOT a full options/futures P&L
  simulation (margin, ATM strike, slippage, the real PE-hedge swap
  mechanics `basket_hedge` mode uses). Deliberately kept simple for this
  first exploratory pass.
- **No lookahead** anywhere — every score at bar t only uses swing points
  already confirmed (respecting the k-bar lag) by bar t.

## Results (n=792 real entry signals across 21 symbols, ~90 days)

| Score bucket | n | Avg score | Avg forward return | Median return | Win rate |
|---|---|---|---|---|---|
| Low tercile | 238 | 0.243 | −0.06% | −0.32% | 35.7% |
| Mid tercile | 238 | 0.397 | +0.18% | −0.29% | 37.4% |
| High tercile | 240 | 0.551 | +0.22% | −0.26% | 38.3% |

- **Pearson correlation (score vs forward return), n=716 scoreable
  entries: +0.061** — a weak positive relationship, not a strong one.
- 90.4% of all real entry signals had SOME recent HH/HL breakout context
  behind them (k=2 is quite permissive) — the 9.6% with no context at all
  performed about the same on average (+0.00%) as the full population, so
  the presence/absence of the pattern alone isn't very informative; only
  the composite score's own MAGNITUDE shows the (weak) gradient above.
- **Same-day "horse race"** (on the 57 days where 2+ scored candidates
  fired): the higher-scored candidate had the equal-or-better forward
  return **33/57 times (57.9%)** — better than a coin flip, but a modest
  edge on a modest sample, directly answering the actual intended use
  case ("which of these competing candidates should Swing pick first").
- Win rates are low across every bucket (36-38%) regardless of score —
  this reflects the base entry signal's own hit-rate characteristics
  (typical trend-following shape: below-50% win rate, presumably bigger
  average winners than losers, though this backtest did not separately
  measure winner/loser magnitude — worth doing before any further
  decision) rather than something this new signal changes.

## Parameter sweep (`traderBoy/backtest_momentum_signal_tune.py`, same day, user request: "tune the parameters and re-test")

Reused the exact same 782 real entry-signal events already found (fully
independent of any momentum-score parameter — only the SCORING varies),
and swept 36 configurations: fractal `k` ∈ {2, 3, 4}, coil/baseline
window ∈ {(8,40), (12,60), (16,80)}, and 4 weight profiles (default,
coil-heavy, RVOL-heavy, freshness-heavy).

**Two findings, and they point in opposite directions — reported both
rather than picking whichever looks better:**

1. **A genuine, non-noisy pattern**: `k=2` beat every single `k=3`/`k=4`
   configuration on population-wide correlation (12/12 — every k=2 row
   outranked every k=3/k=4 row). Best: `k=2, coil/baseline=8/40,
   weights=rvol_heavy` → correlation **+0.071** (up from the original
   pass's +0.061, using default weights and k=2 already) — a small
   improvement, still a weak correlation in absolute terms. Its same-day
   horse race at this exact configuration: **50.9%** — a coin flip.

2. **The metric that matters most for the actual use case (same-day
   "which candidate should Swing pick") does NOT track correlation at
   all.** The single best horse-race result across all 36 configs was
   **67.3%** (`k=3, coil/baseline=16/80, weights=fresh_heavy`, n=55
   days) — but that same configuration's population correlation is
   **NEGATIVE (−0.018)**, and its own high-tercile average return
   (+0.06%) is WORSE than its low-tercile average (+0.21%) — i.e. by the
   bucket-average metric, this "best horse race" config's own high scores
   would have picked the WORSE performer on average, exactly backwards
   from what a working signal should do.

**This divergence is itself the headline finding, and it's a caution
sign, not a green light.** Scanning 36 configurations and reporting
whichever one hits 67.3% on a 55-day sample is exactly the shape of a
multiple-comparisons/overfitting artifact (out of 36 tries, SOME
configuration hitting notably above/below 50% by chance alone is
expected even with zero real underlying skill) — not evidence of a real,
generalizable edge. A genuinely robust signal should show the SAME
horse-race-favoring config also correlating positively and showing a
sensible bucket-average ordering; this one does neither.

## Honest read

Tuning did not rescue this into a deployable signal. The one robust
finding (k=2 outperforms k=3/4 on correlation, consistently) is real but
small (+0.061 → +0.071, still a weak correlation by any standard) and
doesn't come with a correspondingly strong same-day horse-race result at
that same setting (50.9%, a coin flip). The config that DOES show a
strong horse-race number fails the correlation and bucket-ordering checks
badly enough to read as noise from scanning many configurations on a
modest (55-57 day) sample, not a real edge. **Still not compelling enough
to recommend production deployment, and the tuning pass itself is a
caution against trusting a single flattering metric out of a parameter
sweep without checking it holds up under a second, independently
meaningful metric too.**

## Before concluding further

1. ~~Sweep `k` (3, 4) and the coil/baseline window sizes~~ **— done, see
   "Parameter sweep" above.** Result: no configuration cleared the bar on
   both correlation AND same-day horse race at once.
2. ~~Separately measure average winner size vs average loser size per
   bucket~~ **— done.** At the best-by-correlation config: low tercile
   avg_winner +1.12%/avg_loser −0.64%, high tercile avg_winner
   +1.79%/avg_loser −0.73% — the high tercile's winners ARE meaningfully
   bigger (not just a marginally-better win rate), which is the one
   genuinely encouraging data point in the whole exercise, but it rides
   on the same weak (+0.071) correlation as everything else here, so
   weight it accordingly rather than as decisive on its own.
3. This was ONE historical window (last ~90 days) — same caution
   `learnings/backtest-methodology.md` already gives for any single-CSV
   backtest result generalizes here too. Still not done, and now the
   single most likely next step to actually move this forward (a longer
   window, or a different market regime) given tuning alone didn't
   settle it.
4. Investigate the MOTHERSON data gap before treating the 21-symbol
   result as fully representative of the 22-symbol live watchlist. Still
   not done.

## Unrelated discovery made getting this backtest's daily context data — CORRECTED, was a false alarm

See `learnings/dhan-charts-historical-endpoint-broken.md` for the full
story. Original suspicion: Dhan's `/v2/charts/historical` (daily candles)
endpoint looked like it was rejecting every request for this account with
`DH-905`. **Verified directly against the live `traderBoy` droplet a few
hours later and it was fine** — the real cause was a stale local access
token producing a misleading `DH-905` instead of a clear auth error on
this one endpoint. Swing's live daily watchlist prune
(`_fetch_daily_closes_once`) was never actually affected. The resampled-
from-5-min approach this backtest used for daily context (point 3 above)
remains a fine, deliberate choice on its own merits — just not one forced
by a broken endpoint.
