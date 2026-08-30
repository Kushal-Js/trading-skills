# VCP — Volatility Contraction Pattern

Source: Mark Minervini's VCP methodology, cross-referenced across multiple
trading-education sites (TraderLion, TrendSpider, Deepvue, tradingsim.com,
finermarketpoints.com) — a widely-taught, standard pattern, not one site's
invention. Researched 30 Aug 2026. See also an existing open-source
reference implementation: [tradermonty/claude-trading-skills' VCP screener
skill](https://github.com/tradermonty/claude-trading-skills) (built for
US/S&P 500 equities via a different data provider — useful as a structural
reference for how to implement pattern-detection logic as a Claude skill,
not directly reusable for NSE data without adaptation).

## What it is

A price-action base that forms **within an existing uptrend** (it only
means something on top of a Minervini Trend Template pass — see that file)
where a stock consolidates through a series of pullbacks, each one
**tighter (smaller %) and on lower volume** than the last. The pattern
reflects supply drying up — sellers running out, in the specific,
observable form of shrinking pullback depth and shrinking volume on each
successive contraction — right before institutional buying pressure tips
the stock into a breakout.

## The three things that must all be true

1. **Sequential contractions, each smaller than the last.** Typically
   2–4 contractions (labeled T1, T2, T3...), with depths tightening in a
   pattern like ~18% → ~12% → ~6%. The final contraction (T3 or later)
   should be under ~10% deep — the tighter the final contraction, the
   higher-quality the setup.
2. **Volume declining through each contraction, reaching its lowest point
   of the entire base on the final contraction** ("volume dry-up") —
   Minervini's own term for this specific signal. This is what separates a
   genuine VCP from an ordinary sideways congestion zone that just happens
   to look similar on price alone.
3. **Each contraction's low is higher than the prior contraction's low** —
   the base is stepping upward even while volatility shrinks, not just
   getting quieter at the same level.

## The pivot point and the entry

The **pivot point** is the high of the right side of the final (tightest)
contraction. The pattern confirms — and the entry trigger fires — when
price **breaks above the pivot on increased volume** (commonly cited
threshold: volume at least ~40% above its own recent average on the
breakout bar). Stop-loss is typically placed just below the low of that
final contraction, giving a tight, well-defined risk on the trade — this is
part of the pattern's actual appeal: contractions this tight mean a
correspondingly tight, honest stop-loss distance.

## Reported effectiveness (context, not a guarantee)

Research cites a ~90% success rate for VCP breakouts meeting all criteria
*when the broader market/index is itself above its own monthly 10-EMA* —
i.e. the pattern's own reported edge is explicitly conditional on a
favorable broader-market regime, not a standalone guarantee independent of
market conditions. Treat any single-number "success rate" claim about a
pattern with the same skepticism this repo already applies to trading
claims generally (see `README.md`'s style guide: cite evidence and scope,
don't just repeat a number).

## Adapting this for DhanBoy's context

Same translation as `minervini-trend-template.md`: VCP as originally taught
is a **multi-week, daily-chart base** — far slower than DhanBoy's own
intraday holding period. The right way to use it here:

- **Detect VCP on daily charts**, across the F&O universe, as a
  **pre-market or once-daily context signal**: "this underlying has built a
  genuine tightening base and sits near a pivot — worth watching closely
  today for an intraday breakout attempt."
- **Don't** expect or try to detect a "VCP" on 5-minute intraday candles —
  the pattern's whole edge (institutional accumulation over weeks) doesn't
  exist at that timescale; a superficially similar-looking tightening
  sequence on a 5-minute chart is just short-term noise, not the same
  phenomenon.
- The actual **intraday entry trigger** stays what's already designed in
  `designs/fno-daily-screener.md`'s Stage 3 (5-min/1-min Supertrend
  crossover, ROC) — VCP's role is upstream of that, narrowing down *which*
  stocks are worth running the intraday trigger logic on, alongside (or
  combined with) the Trend Template's own pre-filter.
- **Practical implementation note for later, when this gets built**: VCP
  detection needs an algorithm to find swing highs/lows in daily price
  data, measure each contraction's % depth and volume, and check the
  sequence is monotonically tightening — this is meaningfully more complex
  than the indicator-based signals (RSI/Supertrend/ROC) already built this
  session, and is real, non-trivial code to write and test carefully rather
  than something to approximate loosely. Worth its own design pass before
  building, same as the F&O screener got.
