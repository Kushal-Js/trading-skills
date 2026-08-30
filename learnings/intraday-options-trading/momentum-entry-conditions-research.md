# Intraday momentum entry conditions - external research, and a real diagnostic that prompted it

Source: web research (31 Aug 2026), prompted directly by diagnosing why
K01 v2's first real backtest (`../../designs/k01-smart-money-flow-
revision.md`) produced only 3 trades, all PE, none CE, over the 3-day
window. Read that diagnostic first (below) - it's the reason this research
was worth doing, not a generic literature dump.

## The diagnostic that started this: CE wasn't broken, the strength gate just never let one through

Re-ran the same 8-stock/3-day dataset with the `strength_min=0.4` gate
removed entirely, counting raw retest geometry only:

- **Regime flips**: 13 CE vs 5 PE (CE more common, as expected - Stage 0's
  Trend Template selects only stocks in a daily uptrend, so their
  intraday regime skews bullish).
- **Raw retests (no strength filter)**: 44 CE vs 48 PE - genuinely
  balanced, no directional bias in the retest geometry itself.
- **Retests clearing `strength_min=0.4`**: 0 CE, 5 PE. Every CE retest's
  strength topped out around 0.24-0.29 across the whole sample; several
  PE retests reached 0.40-0.42.

So the all-PE result was real, but it was the **strength gate**, not the
retest mechanic, doing the filtering - and specifically, in this one
3-day/8-stock sample, PE-side money-flow readings happened to run higher
than CE-side ones. The natural next question: is that a real market
pattern (down-moves genuinely carry more volume/conviction) or small-
sample noise? That's what the research below actually settles - and the
answer complicates the intuitive story.

## Correction to an intuitive but wrong assumption: "down moves are panickier" doesn't reliably hold for India

The obvious hypothesis - "selling pressure is naturally sharper/higher-
volume than buying pressure, so PE retests will always look stronger" -
**is not well-supported for the Indian market specifically**. Cross-market
academic research comparing asymmetry indices across ten stock markets
found that in most markets, price falls faster than it rises - **but China
and India are the documented exceptions, where price RISES are generally
faster than price FALLS**. The behavioral mechanism cited elsewhere
("panic" selling after down-moves being ~10x stronger than the buying
reaction to up-moves) is a general-market finding, not one that transfers
cleanly to NSE.

**Practical consequence: don't explain away K01 v2's CE=0/PE=5 result as
"of course, down-moves are always more violent" - the actual published
evidence argues that explanation is backwards for this specific market.**
The honest conclusion stays what the design doc already says: this is a
3-day sample, too small to know if it's real, and the fix is a longer
backtest, not a market-structure story that happens to sound plausible.

## Concrete, sourced entry-condition ideas worth testing against K01 v2

### 1. Relative Volume (RVOL) as a possibly-better confirmation than raw money-flow strength

RVOL = current volume ÷ the historical average volume for that SAME bar
interval at the SAME clock time (time-of-day-adjusted, not a flat
lookback-window average) - common thresholds: **RVOL ≥ 2.0 for a genuine
momentum candidate**, **RVOL ≥ 1.5 specifically on the breakout/retest
bar** as confirmation. Explicitly flagged as a *confirmation filter to
pair with a price trigger*, not a standalone entry signal.

**Why this might fix the CE/PE imbalance specifically**: RVOL is
direction-agnostic by construction (just "was this bar's volume unusual
for this time of day") - it can't develop the kind of one-sided skew the
CLV-weighted, signed money-flow-strength calculation showed in the 3-day
sample, since it never encodes which direction the volume moved, only how
much there was. Worth backtesting as either a *replacement* for the
strength gate or an *additional* confirmation alongside it.

### 2. Opening Range Breakout (ORB) + VWAP + volume - a three-layer confluence pattern

A commonly-cited "good" ORB entry requires ALL of: a candle closing beyond
the opening range (not just an intra-bar poke through it), price above
VWAP (confirms the day's average participant is in profit, i.e. broad
participation not just a spike), and volume on the breakout bar ≥1.5x the
recent average. Each added confirmation layer trades fewer setups for
higher quality per setup - the same "fewer, better-confirmed conditions"
principle already validated in this repo's own DanDanaDan-2 vs Kaashvi-28
backtest comparison (`../screener-analysis/dandanadan-2.md`).

Not identical to K01's own adaptive-band regime mechanic, but the VWAP
layer specifically is worth considering as an additional confirmation
K01 v2 doesn't currently have at all (it has RSI band + adaptive-band
regime + money-flow strength, no VWAP check anywhere).

### 3. First pullback only - directly refines K01 v2's own retest design

"The first pullback has the highest probability of continuation - only
trade the first pullback, not the second or third" is a repeated, specific
claim across pullback-trading sources, framed as: a pullback retest is
higher-probability than a fresh breakout because it's a *second*
confirmation of the same move, but that edge specifically degrades on the
second/third pullback within the same trend leg.

**This is a real gap in the current K01 v2 prototype**
(`k01_v2_indicators.py`'s `find_retest_entries`, scratchpad only, not yet
in either repo's tracked code): it currently allows every qualifying
retest after a cooldown, with no "only the first one per regime" limit.
Per this research, that should be **the first retest since the current
regime began, full stop** - not cooldown-gated repetition. This also
directly bears on `k01-smart-money-flow-revision.md`'s Option C (scale-
ins on a second retest): if the second pullback is genuinely lower-
probability than the first, a scale-in triggered by a second retest needs
a STRICTER bar to clear (not the same threshold as the first entry), or
should be reconsidered - this research doesn't support treating a second
retest as an equally-good opportunity to add exposure.

### 4. A market-regime filter K01 has no equivalent of at all

One sourced intraday-momentum approach requires the broad index (Nifty)
to be trading above its own 50-day SMA as a precondition for taking ANY
individual-stock momentum trade - filtering out days where the broader
market environment itself is unfavorable, regardless of how good a single
stock's own setup looks. **K01 has no index-level filter anywhere in its
pipeline** - Stage 0 (Trend Template) is per-stock only, Stage 1
(liquidity) is per-stock only, Stage 3 (momentum) is per-stock only.
Worth considering as a new Stage -1/Stage 0-adjacent gate: skip the whole
daily screen, or bias toward CE-only/PE-only for the day, based on
Nifty's own trend state - directly relevant to the "why so few CE trades"
question if the 26-28 Aug window happened to be a broadly weak-for-Nifty
stretch (not checked yet - worth doing before the next backtest).

## Checked, not left as a "worth checking": Nifty was actually down 2 of the 3 backtest days

Pulled Nifty's real daily OHLC for the exact backtest window:

| Date | Open | Close | Change |
|---|---|---|---|
| 26 Aug 2026 | 24,341.95 | 24,207.75 | **-0.55%** |
| 27 Aug 2026 | 24,277.60 | 24,090.85 | **-0.77%** |
| 28 Aug 2026 | 24,122.60 | 24,175.65 | +0.22% |

**This is a materially better explanation for the CE-shy result than
either the strength-gate mechanics or market-microstructure folklore.**
Two of the three backtest days were genuinely broad-market-down days -
individual stocks passing Stage 0's *daily* Trend Template (a slower,
weeks-long signal) can still have a weak *intraday* session on a day the
whole market sells off, which would plausibly produce more genuine
selling-side volume/conviction even on names in a longer-term uptrend.
This doesn't prove causation from n=3 days alone, but it's a real,
checked fact about the market during this window - not a story invented
to explain a result after the fact, and it directly argues for building
the Nifty-regime filter (idea #4) sooner rather than later, since it
would have been directly informative for interpreting this exact result.

## What to actually do with this before the next K01 v2 iteration

1. **Add the Nifty-regime filter (idea #4) before the next backtest** -
   given it's now a confirmed, not hypothetical, factor in this exact
   result, it should be built alongside (not after) a longer backtest
   window, so the longer run's own CE/PE split can be interpreted against
   Nifty's own daily direction each day, not read blind.
2. Try RVOL (idea #1) as either a replacement or a parallel run alongside
   the existing money-flow-strength gate on the SAME dataset - cheap to
   test since the underlying 5-min volume data is already fetched, and
   directly answers whether the CE/PE imbalance was ALSO an artifact of
   the specific strength formula, independent of the Nifty-direction
   explanation above (both can be true at once).
3. Restrict `find_retest_entries` to the first qualifying retest per
   regime only (idea #3) before the next backtest run - a small code
   change, and directly grounded in real trading-strategy literature
   rather than an assumption.
4. Still don't change all of these at once - per this repo's own
   methodology (`../backtest-methodology.md`), change one thing at a time
   so a result's cause is actually isolable. Given #1 is now confirmed
   relevant (not speculative), it's the reasonable one to prioritize
   first if only one change fits before the next run.

Sources: [ORB + VWAP + volume](https://www.sahi.com/blogs/orb-trading-strategy-explained), [TrendSpider on RVOL](https://trendspider.com/learning-center/relative-volume-rvol-trading-strategies/), [Trade Ideas on RVOL thresholds](https://www.trade-ideas.com/learning-center/trading-strategies/relative-volume-trading-strategy/), [Bulls on Wall Street - first pullback](https://www.bullsonwallstreet.com/post/first-pullback-trading-strategy), [Strike.money - pullback trading](https://www.strike.money/technical-analysis/pullback-trading), [cross-market price-fall/price-rise asymmetry](https://arxiv.org/pdf/1903.05322), [Bogleheads discussion on the general asymmetry](https://www.bogleheads.org/forum/viewtopic.php?t=351754), [intraday momentum + Nifty-SMA filter](https://app.prorsi.com/upload/doc/638232910029521295momentum-trading.pdf).
