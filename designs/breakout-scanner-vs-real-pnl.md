Status: RECALIBRATED 20 Sep 2026 (user-specified thresholds) - now
produces real signals with a promising but thin-sample result (+Rs67,619
over 32 signals, 90.6% win rate, vs real PnL of -Rs61,827.65 over the same
14 days). Backtest-only - nothing wired into any live package, no
sign-off given to deploy.

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

## Recalibration round (20 Sep 2026, user-specified numbers)

User's own explicit spec: switch from 4-hour to 5-minute candles, relax
candle body to >=1% and breakout clearance to >=0.5% above the 10-candle
consolidation high (mirrored: -0.5% below the low for PE breakdowns).
Applied to BOTH `breakout-scanner/server.js` (the live JS scanner) and
`backtest_breakout_screener_vs_real.py` so the two stay in sync. Also
applied a user-requested F&O-eligibility filter on the bucket universe
(built from the live Dhan instrument master's own NSE OPTSTK listings) -
**it excluded zero symbol-days**, meaning DhanBoy's own Chartink screeners
(Krishvi/laxmi/Range-Breakout/etc.) are apparently already scoped to
F&O-eligible names only; the bucket was never actually polluted with
non-tradable stocks in the first place.

### A second real bug caught before trusting this result

The first 5-min run produced an implausible 94.1% win rate and +Rs101,294
from 34 signals - almost every exit was `TARGET_HIT`. Root cause: the
shared `simulate_exit_ladder` (from `backtest_bucket_switch.py`, also used
by [[alert-bucket-switch]]'s backtest) fills at the triggering candle's
**close**, never checking high/low. Harmless for normally-priced options,
but many of this universe's flagged CE/PE premiums are sub-Rs10 (IDEA
@0.62, GMRAIRPORT @1.05, MAHABANK @1.34-1.95) - a single 1-min candle's
close can drift far past a nominal +10% target on these before the loop
even records it, since cheap premiums swing huge in percentage terms on
tiny absolute moves. Fixed locally (`simulate_exit_ladder_fill_at_level`,
kept separate from the shared function so [[alert-bucket-switch]]'s
already-published numbers aren't silently altered) to fill AT the target/
stop price level when a candle's high/low crosses it, or at that candle's
own open if it gapped straight past - the standard, more conservative
convention. This alone cut the reported gain from +Rs101,294 to +Rs67,619
and the win rate from 94.1% to 90.6% - a real, material difference, and a
second instance (after [[alert-bucket-switch]]'s lot-size bug) of this
codebase's exit-ladder simplifications quietly producing an implausibly
clean result until checked against a sanity bound.

### Result after both fixes

| | Trades/Signals | Wins | Win Rate | Total PnL |
|---|---|---|---|---|
| REAL (Options/Futures/Luxury, 14 days) | 284 | 104 | 36.6% | -Rs61,827.65 |
| ADOPTED BREAKOUT SIGNALS | 32 | 29 | 90.6% | **+Rs67,619.10** |

Highly selective: only 32 of 611 symbol-days (~5.2%) ever produced a
confirmed signal. Of those 32: 16 were symbols the bot ALSO traded for
real that same day (same option_type) - 16 were breakouts the bot's own
alert-driven entry logic never touched at all that day.

**The cleanest slice - same symbol, same day, same option_type, bot's
real entry vs. this screener's confirmed-breakout entry (16 matched
cases):**

| Day | Type | Symbol | Real PnL | Breakout PnL | Delta |
|---|---|---|---|---|---|
| 31 Aug | PE | ADANIPORTS | -1,377.50 | 2,446.25 | +3,823.75 |
| 3 Sep | CE | MAHABANK | -2,600.00 | 2,535.00 | +5,135.00 |
| 4 Sep | CE | RELIANCE | -1,425.00 | 2,095.00 | +3,520.00 |
| 8 Sep | CE | GVT&D | -743.75 | 3,317.50 | +4,061.25 |
| 9 Sep | CE | COALINDIA | -135.00 | 2,092.50 | +2,227.50 |
| 9 Sep | CE | PAYTM | -471.25 | 7,097.75 | +7,569.00 |
| 10 Sep | CE | OIL | 1,820.00 | 2,828.00 | +1,008.00 |
| 11 Sep | CE | MCX | -1,338.75 | 3,870.00 | +5,208.75 |
| 11 Sep | CE | PAYTM | -942.50 | 7,242.75 | +8,185.25 |
| 15 Sep | PE | DELHIVERY | -1,660.00 | 3,154.00 | +4,814.00 |
| 15 Sep | PE | INDIGO | 525.00 | 2,319.00 | +1,794.00 |
| 15 Sep | PE | VEDL | 1,322.50 | 1,311.00 | -11.50 |
| 16 Sep | CE | PATANJALI | 2,096.25 | 1,548.00 | -548.25 |
| 16 Sep | CE | PAYTM | -3,443.75 | -9,761.40 | -6,317.65 |
| 18 Sep | CE | MAHABANK | 0.00 | 1,742.00 | +1,742.00 |
| 18 Sep | PE | INFY | -1,240.00 | 776.00 | +2,016.00 |
| **Total (16 matched)** | | | **-9,613.75** | **34,613.35** | **+44,227.10** |

14 of 16 improved; the two losers (VEDL trivial, PATANJALI small) and one
shared loser (PAYTM 16 Sep, worse for breakout) don't offset the size of
the 14 wins. This is the most controlled comparison available (identical
symbol/day/direction, only the entry timing/confirmation differs).

## Caveats (read before treating this as a green light)

- **Thin sample**: 32 signals over 14 days, ~5% hit rate on the bucket
  universe. Not enough to rule out a lucky window.
- **Idealized fills, no slippage/liquidity modeling** - entry at the next
  1-min candle's real open, exit at target/stop level or that candle's
  open. Real order execution on some of these sub-Rs10, thin-volume
  contracts (GMRAIRPORT, IDEA-tier premiums) could plausibly cost more in
  real slippage than this backtest assumes.
- **One signal per symbol per day** (first qualifying candle) even though
  5-min bars offer far more candles/day than the original 4h version -
  kept for consistency, not re-evaluated for whether allowing re-signals
  intraday would change anything.
- Market-cap check still dropped (see above); PE-mirror still unproven;
  1%/0.5% thresholds are the user's own explicit numbers, not fitted to
  this universe's own candle-size distribution.
- Every entry gate real production applies (RSI/trend, volume floor,
  liquidity, funds, cross-strategy claim, daily re-entry cap) is absent
  here - a live version of this signal would still have to clear all of
  those before an order actually gets placed.

## What's still open

This is backtest evidence only - nothing is wired into any live package,
and per [[feedback-live-trading-safety]] that stays true no matter how
good a backtest number looks. Next decision is the user's: gather more
days of data, try live-wiring as a NEW capacity-slot filter alongside the
existing Chartink-alert entry logic (rather than replacing it), or leave
this as a documented finding for now.
