# "ema_cci_macd" strategy - 50/110/250 EMA regime + CCI/MACD-histogram trigger, 30-day backtest

**Date:** 26 Sep 2026
**Source:** YouTube video "The Most Accurate EMA Settings Ever - Almost
ALWAYS WINS!" (Trader DNA), https://www.youtube.com/watch?v=_Wr57vS9ADM -
UNLIKE the bollinger/liquidity-sweep backtests this session, this video's
own pinned comment links a written article
(traderversity.com/the-most-accurate-ema-settings-ever-ema-cci-macd-indicators.html)
that states the mechanics precisely in prose, so the rules below are
quoted from that page rather than reconstructed from caption fragments.
**Scope:** backtest script only
(`traderBoy/backtest_ema_cci_macd_swing_watchlist.py`), NOT wired into
Swing/trading_engine.py - same 9-symbol NSE-equity Swing watchlist
(BANDHANBNK, TORNTPHARM, DLF, ZYDUSLIFE, SONACOMS, CIPLA, ASHOKLEY, VEDL,
SOLARINDS), OPTIONS basket, last 30 trading days - identical scope to
[[bollinger-vortex-strategy-30day-backtest]] and
[[liquidity-sweep-prev-hour-strategy-30day-backtest]], explicit user
request to test all three video-derived strategies on "the same data."

## The rules, as stated in the source article

1. **Three EMA tiers** as layered dynamic support/resistance: 50-period
   (primary), 110-period (secondary, watched if price breaks below 50),
   250-period ("the final line of defense," chosen over the conventional
   200 EMA - "if the price breaches even the 250-period EMA and fails to
   recover, that often signals a major shift in trend"). The article is
   explicit these exact numbers are an EXAMPLE from one backtest, not
   universal - the real lesson is the METHOD (sweep 20-200 and keep
   whichever period the market visibly respects).
2. **Entry - CCI + MACD histogram combo:**
   BUY: CCI drops below -100 while MACD histogram stays ABOVE zero
   throughout the dip ("despite the pullback, market momentum remains
   bullish"), then CCI crosses back above zero while the histogram is
   positive -> BUY signal.
   SELL: CCI rises above +100 while MACD histogram stays BELOW zero
   throughout, then CCI crosses back below zero while histogram is
   negative -> SELL signal.
   "The MACD histogram acts as a trend filter, confirming the overall
   direction remains intact."
3. **No exit rule anywhere in the source** - the article only ever
   describes the entry signal.

## Interpretation calls made (full detail in the script's own docstring)

- EMA periods used exactly as stated (50/110/250) - the video's own
  per-symbol "sweep 20-200 and keep what's respected" meta-search was
  NOT re-run per symbol (a materially larger, separate backtest).
- Regime = EMA STACK ORDER (EMA50>EMA110>EMA250 = bullish, reverse =
  bearish, mixed = no trade) - operationalizes "bullish/bearish market
  scenario" as the trend-alignment filter for the CCI/MACD trigger, not
  a literal "price touching one specific EMA line at that exact bar"
  condition (the source text never explicitly joins the two into one
  single-bar rule).
- CCI period -> never stated, universal default 20. MACD -> never
  stated, universal default (12,26,9).
- No exit rule -> ported Swing's own live-system convention instead
  (buy/sell, exit on the OPPOSITE signal) - a LONG CE exits on a fresh
  SELL trigger, a LONG PE exits on a fresh BUY trigger. `MAX_LOSS_
  PROTECTION_RS` and 15:15 IST square-off applied as safety nets, same
  as every backtest here.
- **No stop-loss/swing-distance rule exists in this video at all** (a
  first for this session's 3 video-derived backtests) - there's no
  underlying distance to convert into a premium percentage the way the
  other two scripts do. Used a bare `FIXED_PREMIUM_STOP_PCT=5%` capital-
  protection floor instead (this script's own choice, not the video's) -
  most trades are expected to exit via the opposite signal or EOD
  square-off, not this floor.

## Result

| Symbol | Trades | Wins | Losses | Win rate | Net P&L |
|---|---|---|---|---|---|
| BANDHANBNK | 2 | 1 | 1 | 50.0% | -Rs 720 |
| TORNTPHARM | 0 | - | - | - | Rs 0 |
| DLF | 0 | - | - | - | Rs 0 |
| ZYDUSLIFE | 1 | 0 | 1 | 0.0% | -Rs 1,305 |
| SONACOMS | 1 | 1 | 0 | 100.0% | +Rs 4,226 |
| CIPLA | 0 | - | - | - | Rs 0 |
| ASHOKLEY | 3 | 2 | 1 | 66.7% | +Rs 1,750 |
| VEDL | 1 | 0 | 1 | 0.0% | -Rs 345 |
| SOLARINDS | 0 | - | - | - | Rs 0 |
| **COMBINED** | **8** | **4** | **4** | **50.0%** | **+Rs 3,606** |

Day-wise (only 6 active days out of 30): 09-02 +72, 09-04 -792, 09-09
+2,000, 09-15 +4,226, 09-16 +350, 09-17 -1,305, 09-21 -945 (2 trades).

Full trade list (all 8):

| Entry | Symbol | Side | Entry | Exit | Qty | Reason | PnL |
|---|---|---|---|---|---|---|---|
| 09-02 15:00 | BANDHANBNK | SELL | 3.64 | 3.66 | 3600 | EOD_SQUARE_OFF | +72 |
| 09-04 14:35 | BANDHANBNK | BUY | 5.19 | 4.97 | 3600 | EOD_SQUARE_OFF | -792 |
| 09-17 09:35 | ZYDUSLIFE | SELL | 20.50 | 19.05 | 900 | FIXED_STOP_HIT | -1,305 |
| 09-15 14:25 | SONACOMS | SELL | 20.25 | 23.70 | 1225 | EOD_SQUARE_OFF | +4,226 |
| 09-09 14:40 | ASHOKLEY | SELL | 3.85 | 4.25 | 5000 | EOD_SQUARE_OFF | +2,000 |
| 09-16 15:10 | ASHOKLEY | SELL | 3.52 | 3.59 | 5000 | EOD_SQUARE_OFF | +350 |
| 09-21 12:10 | ASHOKLEY | BUY | 2.21 | 2.09 | 5000 | FIXED_STOP_HIT | -600 |
| 09-21 12:10 | VEDL | BUY | 5.05 | 4.75 | 1150 | FIXED_STOP_HIT | -345 |

## Real finding: the signal is extremely rare, and the "positive" result is one trade

**4 of 9 symbols (TORNTPHARM, DLF, CIPLA, SOLARINDS) never fired a single
signal in the full 90-day fetch window (30-day test + 60-day warmup),
let alone the 30-day test window itself.** Requiring EMA50/110/250 to be
in strict, clean order (a real sustained trend) AND a CCI excursion past
+-100 AND MACD histogram staying on the trend's side for the ENTIRE
excursion is a highly selective triple-confluence filter - 8 entries
across 9 symbols x 30 days is roughly 1 trade per symbol per month, two
orders of magnitude less frequent than either the bollinger (287 trades)
or liquidity-sweep (522 entries) backtests on the identical watchlist/
window.

**The one SONACOMS trade (+Rs 4,226) is 117% of the combined total** -
without it, the other 7 trades net to -Rs 620. This is not a converged
result; it is one large winning trade riding an EOD square-off, on top of
a roughly coin-flip small sample otherwise. **Do not read "+Rs 3,606,
50% win rate" as this strategy being profitable** - with n=8 trades,
that framing is essentially meaningless; it would take very little
(a different 30-day window, a different symbol set) to flip the sign
entirely.

**Other standard caveats** (same as every backtest in this repo): no
slippage/brokerage modeled, `FIXED_PREMIUM_STOP_PCT` is this script's own
invention (the video states no stop at all), one 30-day window is one
sample, and here specifically an unusually small one at that.

**Outcome:** informational only - not deployed, not wired into Swing.
Purely a standalone backtest artifact, matching the user's own explicit
scope for this class of request (see
[[bollinger-vortex-strategy-30day-backtest]] for the identical scope
precedent). Given the sparse signal, this strategy is a weaker candidate
for further iteration than the bollinger strategy on this same watchlist
unless tested over a much longer window to build a meaningful sample size.
