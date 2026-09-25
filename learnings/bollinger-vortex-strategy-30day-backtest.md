# "bollinger" strategy - Bollinger ribbon + Vortex breakout, 30-day backtest

**Date:** 26 Sep 2026
**Source:** YouTube video "ALMOST NEVER LOSES! - VORTEX BREAKOUT Trading
Strategy" (Trader DNA), https://www.youtube.com/watch?v=cd1C9TuK99c -
no downloadable transcript file existed, so the rules were pulled by
seeking the video's `<video>` element second-by-second and reading the
live caption overlay via injected JS (`.ytp-caption-segment`).
**Scope, confirmed with the user:** backtest script only
(`traderBoy/backtest_bollinger_vortex_9symbols_30day.py`), NOT wired into
Swing/trading_engine.py, not a new ENTRY_STRATEGY_VERSION - a standalone
strategy, sharing only candle-fetch/option-resolution/P&L plumbing with
the rest of this repo's backtests.
**Symbols:** same 9-symbol NSE-equity Swing watchlist (BANDHANBNK,
TORNTPHARM, DLF, ZYDUSLIFE, SONACOMS, CIPLA, ASHOKLEY, VEDL, SOLARINDS),
OPTIONS basket, last 30 trading days.

## The rules, as stated in the video

1. **Trend filter** - 5 Bollinger Bands on one chart, all period=20,
   deviations 0.5/0.4/0.3/0.2/0.1 (a tight "ribbon" around the 20-SMA,
   NOT a standard 2.0-deviation band). Price in the upper part of the
   ribbon = bullish bias; lower part = bearish.
2. **Trend confirmation** - Vortex Indicator (VI+/VI-, period never
   stated in the video). VI+ > VI- = bullish; VI- > VI+ = bearish.
3. Trend only "valid" when both agree.
4. **Entry** - never on the breakout itself. Wait for a genuine
   multi-candle pullback against the confirmed trend (a single opposite
   candle explicitly does NOT qualify, per the video's own example),
   then place a stop order at the swing point that existed just before
   the pullback - it fires only if price breaks back through in the
   trend direction.
5. **Stop-loss/trailing** - nearest swing point, then trail at 1/3 of
   the initial stop distance with a 1/5 step. **No profit target is ever
   mentioned** - pure trend-following trailing-stop exit.

## Interpretation calls made (full detail in the script's own docstring)

- "Upper/lower part of the zone" -> persisting state, close vs SMA(20)
  (the ribbon is so tight at 0.1-0.5 deviations that this is
  functionally identical to "which side of the zone").
- Vortex period -> 14 (universal default, never stated).
- "Genuine pullback" -> >=2 consecutive closes against the confirmed
  trend, after a confirmed 2-bar fractal swing point, with the BB+Vortex
  filter staying unbroken throughout (cancels the pending order the
  instant it flips).
- No volume-floor gate, no Day Range branch, no regime-EMA leg - this is
  NOT a hybrid with Swing's own v1-v4 formula.
- Swing distance (on the underlying) converted to a PERCENTAGE at entry,
  applied to the option premium via the same `hard_stop_for` helper used
  everywhere else in Swing - the one place this script's risk math is a
  per-trade DYNAMIC percentage rather than a fixed config constant.

## Result

| Symbol | Trades | Wins | Losses | Win rate | Net P&L |
|---|---|---|---|---|---|
| BANDHANBNK | 32 | 21 | 10 | 65.6% | +Rs 27,648 |
| TORNTPHARM | 36 | 24 | 12 | 66.7% | +Rs 12,269 |
| DLF | 34 | 18 | 15 | 52.9% | +Rs 7,220 |
| ZYDUSLIFE | 31 | 18 | 12 | 58.1% | +Rs 28,935 |
| SONACOMS | 35 | 17 | 17 | 48.6% | +Rs 10,474 |
| CIPLA | 28 | 13 | 13 | 46.4% | +Rs 2,593 |
| ASHOKLEY | 29 | 21 | 7 | 72.4% | +Rs 8,000 |
| VEDL | 33 | 21 | 7 | 63.6% | +Rs 5,635 |
| SOLARINDS | 29 | 12 | 17 | 41.4% | +Rs 3,467 |
| **COMBINED** | **287** | **165** | **110** | **57.5%** | **+Rs 106,240** |

All 9 symbols net positive. Combined day-wise P&L climbed on 15 of 19
active trading days, with the two worst days (09-16: -Rs 3,100, 09-18:
-Rs 2,974) both small relative to the best days (09-22: +Rs 17,145,
09-01: +Rs 9,968).

## Real caveat found while building this, not just a generic disclaimer

**277 of 287 entries (96.5%) hit the script's own 1% stop-distance
floor** (`stop_pct = max(stop_pct, 0.01)`), meaning the ACTUAL
2-bar-fractal swing distance on 5-min candles for these liquid large/
mid-cap names was almost always tighter than 1% of the underlying price.
So this backtest is really testing "BB+Vortex pullback-continuation
entry with a near-uniform ~1% underlying stop and proportional
trailing," not the video's genuinely variable swing-based stop - the
video's own risk-sizing logic barely engaged. This is very likely WHY
trade frequency is so high (287 trades/30 days across 9 symbols, ~1/
symbol/day, several times the pace of Swing's own v2/v3/v4) and why win
rate sits in the 41-72% range per symbol rather than clustering tighter -
tight stops on a pullback-continuation entry produce exactly this
profile (many quick stop-outs, offset by trailing-stop winners that run).
A 2-bar fractal on 5-min candles may simply be too tight a swing
definition for this instrument set - worth re-testing with a wider
fractal lookback (e.g. 3-5 bars) or a coarser signal timeframe (15-min)
before reading this result as representative of the video's actual
system.

**Other standard caveats** (same as every backtest in this repo): no
slippage/brokerage modeled, and at ~1 entry/symbol/day this frequency
would be far more sensitive to real execution costs than Swing's own
lower-frequency v2/v3. One 30-day window is one sample.

**Outcome:** informational only - not deployed, not wired into Swing.
Purely a standalone backtest artifact per the user's explicit scope.
