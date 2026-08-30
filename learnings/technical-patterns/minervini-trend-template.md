# Minervini's Trend Template — the "is this stock even worth watching" filter

Source: Mark Minervini's *Think & Trade Like a Champion* trend-template
criteria, cross-referenced across multiple trading-education sites
(TraderLion, Deepvue, ChartMill, sharpely.in) — a widely-taught, standard
methodology, not one site's invention. Researched 30 Aug 2026.

## What it is

A checklist of 8 conditions that together define a stock in a confirmed
**"Stage 2" uptrend** — the phase of a stock's cycle where institutional
accumulation is underway and price has structural support to keep
advancing. It is a *pre-filter*, not an entry signal: it answers "is this
stock in the kind of trend worth building a setup on at all," before any
pattern (like VCP, see `vcp.md`) or intraday trigger gets applied.

## The 8 criteria (a stock must pass ALL of them)

1. Current price is above both the 150-day (30-week) and 200-day (40-week)
   moving averages.
2. The 150-day MA is above the 200-day MA.
3. The 200-day MA has been trending up for at least 1 month (not flat or
   declining).
4. Price is above the 50-day MA.
5. The 50-day MA is above both the 150-day and 200-day MAs (short-term
   trend confirms the longer-term one).
6. Current price is at least 30% above the 52-week low (rules out stocks
   still basing near multi-month lows).
7. Current price is within 25% of the 52-week high (rules out stocks that
   have already fallen far from their peak — a genuine Stage 2 stock stays
   relatively close to its highs).
8. (Often included as supporting context rather than a hard gate) Relative
   strength versus the broader market/index is strong — the stock should be
   outperforming, not just moving with the market.

**Why all 8 together, not any one alone:** each individual criterion on its
own produces a lot of false positives (e.g. "above the 50-day MA" is true
for a huge fraction of stocks on any given day). It's the *simultaneous*
combination — price structure, moving-average alignment, and proximity to
both the low and the high — that narrows the universe down to stocks
genuinely in a structurally strong uptrend.

## Adapting this for DhanBoy's context — an important translation

Minervini designed this for **swing/position trading** (holding periods of
weeks to months), using **daily and weekly charts**. DhanBoy's Options
strategy holds positions for **minutes to at most a day or two** — a very
different timeframe. The trend template doesn't need to be discarded for
that reason, but its *role* changes:

- **Don't** try to use 150/200-day MA alignment as an intraday entry
  trigger — that's not what it's for, and at DhanBoy's holding-period scale
  it's far too slow a signal.
- **Do** use it as a **daily-chart context filter**, run once per day (or
  cached and refreshed periodically), to answer "which F&O underlyings are
  in a strong enough structural uptrend to be worth watching for an
  intraday CE entry today at all" — narrowing the ~208-stock F&O universe
  down before any of the faster, intraday-scale signals (5-min RSI/
  Supertrend/ROC, already in `designs/fno-daily-screener.md`) get applied.
- The natural place for this in the F&O screener design is a **Stage 0**,
  ahead of the existing Stage 1 liquidity floor — see that file for the
  update.

This mirrors a distinction already learned the hard way in this repo: a
signal computed on the wrong timeframe for the strategy's actual holding
period produces misleading conclusions (see `learnings/exit-mechanics.md`'s
finding that `on_price_tick` reacts on arrival regardless of any interval
setting — timeframe mismatches between what a signal measures and what a
strategy needs are a recurring category of mistake to watch for).
