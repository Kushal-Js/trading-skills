# Smoothed Heikin Ashi 4-step pullback strategy - tested, no edge (3 Oct 2026)

Source: YouTube, Trader DNA, "(NON-REPAINT) How To Make 6 Figures Per Trade Using Only This Indicator!" (6cpFCLErKks, Oct 2025). The user asked to study it as a possible "winning strategy".

## Rules (from the transcript)

1. A clear trend: a run of big smoothed-HA candles of one colour.
2. The candles shrink and flip colour. Do NOT enter on the flip (the video calls it late).
3. Wait for price to pull back to the smoothed-HA candles (support/resistance).
4. Confirmation = an engulfing candle. Enter there, stop below the swing low, target the "next rejection level".

Shorts are the mirror image. No timeframe, smoothing lengths or exact target were given.

## Implementation (scratch `ha_strategy.py`)

- sHA = EMA10 of O/H/L/C -> Heikin Ashi -> EMA10 of the HA O/H/L/C.
- Trend = at least 8 same-colour sHA candles before the flip; the setup stays valid for 20 bars while the sHA colour holds.
- Pullback = a candle's low (high for shorts) trades into the sHA body; then a regular-candle engulfing (prev opposite colour, body engulfs prev body) triggers entry at its close.
- Stop = lowest low (highest high) since the flip. Exit at 2R, or the variant "exit when sHA flips back". Max hold 50 bars.
- Costs 0.10% of notional (stocks) / 0.07% (NIFTY futures) per round trip.
- Baseline = enter at the close of the flip candle.

## Results

**F&O stocks, 1-hour candles, Aug 2025 - Oct 2026 (Yahoo), 211 stocks with trades:**

| Variant | Trades | Win | Gross / trade | Net PF | Net, 1 lot each |
|---|---:|---:|---:|---:|---:|
| Video method, 2R | 3,874 | 38% | +0.10% | 1.00 | -9.5L (-245/trade) |
| Video method, exit on flip | 3,868 | 29% | +0.09% | 0.99 | -11.9L |
| Enter on flip, 2R | 8,182 | 40% | +0.10% | 1.00 | -7.0L |
| Enter on flip, exit on flip | 9,194 | 32% | +0.08% | 0.98 | -21.3L |

About half of the stocks are net positive (106/211), as chance would give.

**Parameter sweep** (smoothing 10/10, 6/2, 20/10; min trend 5/8/12; target 1.5R/2R/3R): **net PF 0.90-1.01 in all 8 settings**, all net negative.

**NIFTY (1 lot = 65):**

| Data | Video method, 2R | Video method, exit on flip | Enter on flip, 2R |
|---|---|---|---|
| Daily 2007-2026 | 24 trades, PF 0.51, -80k | 24 trades, -137k | 95 trades, PF 0.71, -6.9L |
| 1-hour, Oct 2023 - Oct 2026 | 34 trades, PF 0.47, -1.56L | 36 trades, PF 1.03, +7.5k | 94 trades, PF 0.93, -52k |
| Fut 5-min, 8 Sep - 1 Oct 2026 | 13 trades, +12.9k | 14 trades, -25k | 21 trades, -33k |
| Fut 15-min, same days | 3 trades, -17.7k | 3 trades, -16.6k | 8 trades, -20.9k |

The 5-min win (13 trades) is too few to mean anything.

## Takeaways

- **Smoothed HA is a doubly smoothed moving average of price, so it is a lagging trend display.** The pullback + engulfing timing does not beat entering on the flip, and neither has an edge after costs.
- **The "6 figures per trade" claim is not supported.**
- **Possible use:** as a chart readability aid only. Any use as a filter inside an existing strategy would need its own test (past filter add-ons here mostly cut profit).
