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

## Update 26 Sep 2026 - 3 more variants (wider swing lookback / coarser timeframe)

Direct follow-up to the floor caveat above. `SIGNAL_INTERVAL_MINUTES` and
`SWING_FRACTAL_LOOKBACK` were made env-var-overridable
(`BOLLINGER_SIGNAL_INTERVAL_MINUTES`/`BOLLINGER_SWING_FRACTAL_LOOKBACK`),
same script, same 9 symbols, same 30-day window. Everything else
(BB period=20/deviations, Vortex period=14, pullback rule, trailing
ratios, 1% stop floor) unchanged.

| Variant | Trades | Wins | Losses | Win rate | Net P&L | % entries hitting the 1% floor |
|---|---|---|---|---|---|---|
| 5min / lookback=2 (original) | 287 | 165 | 110 | 57.5% | **+Rs 106,240** | 96.5% (277/287) |
| 5min / lookback=5 | 159 | 70 | 84 | 44.0% | +Rs 18,261 | 93.7% (149/159) |
| 15min / lookback=2 | 97 | 59 | 33 | 60.8% | +Rs 39,914 | 83.5% (81/97) |
| 15min / lookback=5 | 54 | 23 | 30 | 42.6% | +Rs 15,175 | 83.3% (45/54) |

**Per-symbol, all 4 variants:**

| Symbol | 5m/L2 | 5m/L5 | 15m/L2 | 15m/L5 |
|---|---|---|---|---|
| BANDHANBNK | +27,648 (32t) | +3,816 (15t) | +7,560 (14t) | +7,884 (8t) |
| TORNTPHARM | +12,269 (36t) | +912 (24t) | +3,169 (12t) | -419 (4t) |
| DLF | +7,220 (34t) | +4,417 (19t) | +2,043 (7t) | -2,185 (7t) |
| ZYDUSLIFE | +28,935 (31t) | +13,500 (15t) | -495 (10t) | -90 (5t) |
| SONACOMS | +10,474 (35t) | -551 (21t) | +17,701 (13t) | +10,290 (7t) |
| CIPLA | +2,593 (28t) | -659 (16t) | +1,466 (11t) | -1,105 (7t) |
| ASHOKLEY | +8,000 (29t) | -100 (14t) | +5,000 (13t) | +150 (8t) |
| VEDL | +5,635 (33t) | +805 (22t) | +1,552 (9t) | +0 (4t) |
| SOLARINDS | +3,467 (29t) | -3,880 (13t) | +1,917 (8t) | +650 (4t) |

**Findings:**

1. **The 1% floor caveat only partly explains the original result.**
   Widening the swing lookback (5 bars) or the timeframe (15-min) does
   reduce the floor-hit rate (96.5% -> 83.3-93.7%), but it stays
   dominant in every variant - a 2-bar-vs-5-bar fractal, or 5-min-vs-
   15-min, isn't enough on its own to make these liquid names' genuine
   swing distances routinely exceed 1% of price. A materially different
   swing definition (a much longer lookback, or an ATR-based stop
   instead of a fractal one) would be needed to test the video's
   risk rule on its own terms.
2. **All 4 variants are net positive**, but P&L drops sharply as the
   floor-hit rate drops: the ORIGINAL (most floor-dominated) variant is
   also the most profitable by a wide margin (+Rs 106,240 vs +Rs
   15,175-39,914 for the other three). This is the opposite of what
   "the floor is masking the real strategy, fix it and see" might have
   hoped to find - the tighter, floor-driven version outperformed every
   attempt to make the stop more genuinely swing-based, at least in this
   30-day sample. Read this as inconclusive on which stop style is
   actually better, not as confirmation either way - trade counts differ
   by 5x across variants (54 to 287), so per-variant results are noisy.
3. **15min/lookback=2 has the best win rate (60.8%) of the 4**, and
   SONACOMS is the standout performer in the two 15-min variants
   specifically (+Rs 17,701 and +Rs 10,290 respectively) despite far
   fewer trades - worth a closer look if this strategy is ever revisited.
4. No variant here should be read as "the" bollinger strategy result -
   this is a parameter-sensitivity finding, not a converged answer.
