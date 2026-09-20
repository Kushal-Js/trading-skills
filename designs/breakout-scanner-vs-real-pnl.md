Status: TESTED, RESULT IS NULL (zero signals) - the breakout-scanner's
literal 8-rule spec does not fire at all on DhanBoy's own NSE alert-bucket
universe over 14 days. Not deployed, not adopted; a recalibration pass
would need explicit user sign-off before treating any relaxed-threshold
number as comparable.

# Breakout-scanner screening rules vs real DhanBoy PnL

## What was asked

Apply the 8-rule breakout screener from `~/Desktop/projects/breakout-
scanner` (built from "Claude Code Built a FREE Breakout Stock Screener",
The AI University, youtube.com/watch?v=vZ8usGRL_h8 - see that project's
own README for the full spec) to DhanBoy's own real CE/PE alert buckets
([[alert-bucket-switch]]) over the last 14 days, and compare a
hypothetical "adopted every confirmed breakout" portfolio against the
bot's REAL Options+Futures+Luxury PnL over the same window.

## Method

`traderBoy/backtest_breakout_screener_vs_real.py`. Reuses
`backtest_bucket_switch.py`'s bucket reconstruction (same empirically-
built scan_name -> option_type table, same 14-day window: 31 Aug - 18 Sep
2026, the only days with `webhook_alerts.log` data - "last 15 days" was
asked for but only 14 exist). For every distinct symbol in a day's CE or
PE bucket (611 symbol-days total), fetched REAL 1h Dhan candles (not
Yahoo - this bot trades off Dhan data, so scoring it against a different
vendor's numbers would be apples-to-oranges), aggregated to 4h exactly
like `server.js`'s `aggregateTo4h`, and walked every 4h candle that closed
on the scan day checking: consolidation range <=12% over the prior 10
candles, breakout/breakdown clearance of 2%, candle body >=5%, relative
volume >=1.5x, then (from a separate daily series, look-ahead-safe -
fetched only through the day BEFORE the scan day) liquidity, price
within 10% of the 20d/50d high (CE) or low (PE), and trend vs both SMAs.
Market cap (original check 6) was dropped - no equivalent to Yahoo's
`quoteSummary` in the Dhan SDK used here, and every bucket symbol is
already a large/mid-cap NSE name where this would never have been
binding anyway. PE used a bearish mirror of the bullish rules (same
relationship as this repo's own `ribbon_score.score_ribbon_expansion` /
`score_ribbon_breakdown` pair - unproven the same way that pair's PE side
is unproven, just the reasonable symmetric assumption).

## Result: zero signals, 611/611 symbol-days

Every single symbol-day was rejected - not close to passing, and not a
data/fetch problem (confirmed 84 real hourly bars / 21 real 4h candles for
a sampled symbol, full 20-day lookback present). Sampled 12 candles across
known high-momentum bucket names on their own alert days (BLUESTARCO,
ETERNAL, NHPC, ADANIPOWER, ASHOKLEY, SIEMENS, TIINDIA - several of these
are the same names that scored highest in [[alert-bucket-switch]]'s ribbon
backtest, i.e. NOT an unlucky/quiet sample):

| Symbol | Day | close vs consolidation-high | range% | candle body size% | rel. volume |
|---|---|---|---|---|---|
| BLUESTARCO | 17 Sep | -1.33% | 6.62% | 2.34% | 0.68x |
| ETERNAL | 17 Sep | +0.06% | 3.00% | 2.66% | 1.88x |
| ADANIPOWER | 17 Sep | -4.90% | 6.23% | 0.31% | 0.63x |
| ASHOKLEY | 18 Sep | -5.92% | 8.36% | 0.62% | 0.73x |
| TIINDIA | 17 Sep | -4.92% | 7.24% | 0.07% | 0.72x |

Required to pass: close >= +2% above the consolidation high, AND candle
body >= 5%. **Max candle-body size seen across the sample was 2.66% - not
one candle came within reach of the 5% requirement, and only one
(ETERNAL) was even marginally above its own 10-candle high at all.**

**Conclusion: the screener's specific numeric thresholds (5% single-4h-
candle body move, 2% breakout clearance) are calibrated for a materially
more volatile universe than NSE large/mid-cap names moving within one
4-hour bar** - plausibly the momentum/small-cap US names (PLTR, NVDA-style
single names) the original video's presenter had in mind, not the kind of
large-cap NSE stocks that dominate DhanBoy's own Krishvi/laxmi/Range-
Breakout alert buckets. This is the same category of finding the video
itself opened with ("2 of my 8 rules were completely wrong" for their own
context) - the rules that survived their fix still don't transfer to a
different market's volatility regime without recalibration.

No comparison against real PnL is meaningful with zero signals on one
side - real Options+Futures+Luxury PnL over the same 14 days was
-Rs 61,827.65 (284 trades, 36.6% win rate), for reference only.

## What would be needed to get a real comparison

Recalibrate the breakout-size/clearance thresholds against this specific
universe's own actual candle-size distribution (a full 611-symbol-day
sweep of raw `close_vs_consolidation_high%`/`breakout_size%`/`relative_
volume` values, then pick thresholds off that distribution rather than
carrying over the original video's numbers) before re-running the
adoption simulation. Not done yet - would change the screener's own
identity (no longer literally the video's 8 rules) and is a strategy-
tuning decision for the user to make, not something to do unprompted.
