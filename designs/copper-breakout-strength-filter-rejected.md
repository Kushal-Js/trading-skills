Status: EXPLORED AND REJECTED (23 Sep 2026) - backtested, not deployed.
Code removed from traderBoy after the finding below; this doc is the
durable record. See [[structure-break-indicator]] for the parent
strategy this was an attempted refinement of.

# COPPER structure-break entries: a price-action "breakout strength" filter

## What triggered this

22 Sep 2026 was a mostly-sideways day for COPPER. The 3-timeframe
structure-break signal (see [[structure-break-indicator]]) still fired
twice, and both trades lost (-Rs 8,000 and -Rs 4,125, -Rs 12,125
combined). The user asked whether any of the existing breakout-scanner's
own filters (`Options/breakout_signal.py`) would have caught this.

A real-data diagnostic (`check_breakout_filters_vs_copper_trades.py`,
since removed - see below) confirmed both losing entries shared a
specific shape: decent-to-strong relative volume (1.17x and 2.8x - volume
was never the problem), but near-doji candle bodies (0.04% and 0.02% vs
breakout_signal.py's own 0.5% minimum) and neither entry actually cleared
the recent 10-bar consolidation high with real conviction.

## What was built

`breakout_strength_filter.py` - a standalone pure function reusing
breakout_signal.py's own range%/clearance%/body% formulas verbatim
(ported, not reimplemented) against a single OHLCV index, supporting both
directions. Relative volume was deliberately left out - Swing's own MCX
volume-floor gate already covers that, and the real incident showed
volume was never the actual problem.

Wiring this into `backtest_swing_structure_break_mtf.py` required a real
structural change: the backtest previously only re-evaluated an entry
candidate on the bar the combined signal first changed (nothing else
could block an entry, so a persisting signal had nothing new to check).
Once an entry could be blocked by something OTHER than the signal itself,
the loop needed to retry every bar a valid signal was present while
flat - matching how Swing's live MCX volume-floor gate already behaves
(a blocked signal keeps firing on every monitor tick until either it
passes or the underlying agreement breaks). This retry-every-bar
mechanism is itself relevant to any FUTURE filter idea in this space, not
just this specific rejected one.

## Backtest results (10-day COPPER window, 22 Sep 2026 data)

| Config | Trades | Win rate | Total PnL |
|---|---|---|---|
| No filter (existing behavior) | 13 | 69% | **+Rs 2,46,875** |
| Filter, body>=0.5% (breakout_signal.py's own default) | 1 | 100% | +Rs 73,000 |
| Filter, body>=0.25% (half threshold) | 3 | 67% | +Rs 71,750 |

Loosening the threshold did NOT recover the lost profit - it let in one
more winner (+Rs 15,125) but also a new loser (-Rs 16,375), netting
slightly WORSE than the stricter threshold.

## Why it was rejected - two distinct failure modes, not one

1. **Over-exclusion**: at either threshold, the filter excluded roughly
   Rs 175,000 of genuinely profitable trades across the window (09-09,
   09-11, 09-16, 09-18) that happened to have modest-but-not-explosive
   candle bodies. The structure-break signal fires on 3-timeframe REGIME
   AGREEMENT, not on a single dramatic candle - most legitimate entries
   simply don't carry the doji-vs-explosive distinction the two losing
   trades happened to exhibit. "Small candle body" correlates with "bad
   trade" far more weakly than the original two-trade diagnostic
   suggested.

2. **Entry-delay degradation (the more interesting failure)**: the
   retry-and-wait mechanism can turn a WINNING trade into a LOSING one,
   not just reject bad ones. Concrete example, 09-15: the real
   (unfiltered) trade was PE entered 12:20 @ 1360.75, exited 15:30 @
   1358.50, +Rs 5,625. With the filter on, that same underlying
   agreement didn't produce a "strong enough" candle until 13:35 (over an
   hour later) @ 1351.95 - same exit, same time, but now -Rs 16,375. The
   filter didn't reject this trade; it waited for a better-looking
   candle, and by the time one appeared the best part of the move was
   already over and price had started reversing. This is a structural
   risk of ANY "wait for a stronger signal" filter layered on top of an
   already-lagging entry condition (3-timeframe agreement is itself not
   instant) - the wait can cost more than the filter saves.

## Takeaway for future filter ideas in this space

The diagnosis that motivated this ("yesterday's losses had weak
candles") was CORRECT for those two specific trades - it just doesn't
generalize into a good blanket rule. A blanket price-action strength gate
applied to every entry isn't selective enough to separate good trades
from bad ones here, and any filter with a "wait for confirmation"
mechanism needs to itself be evaluated for entry-delay risk, not just
false-positive/false-negative rates on the original signal. If this gets
revisited, it would need to be something more targeted (e.g. distinguish
which SPECIFIC circumstances predict a bad structure-break entry, rather
than a universal candle-strength floor) or bounded (e.g. give up on a
formation if it hasn't qualified within N bars, rather than waiting
indefinitely) - neither was attempted here.

## Code status

`breakout_strength_filter.py` and `check_breakout_filters_vs_copper_
trades.py` were deleted from traderBoy after this conclusion - this
document is the durable record, not the code. `backtest_swing_
structure_break_mtf.py` was reverted to its pre-filter state (no
`--breakout-filter` flag, no retry-every-bar restructure) - if this idea
is revisited later, the retry-every-bar mechanism described above is
worth re-deriving from this doc rather than assuming the current
backtest script still supports it.
