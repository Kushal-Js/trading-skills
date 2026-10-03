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

## Follow-up: as a trend filter inside Unified Momentum (user, 3 Oct)

The user believed it works best on short timeframes (5-min or lower).

Setup: deployed UM config, walk-forward lists, 3 Aug - 1 Oct (43 sessions), cache only (scratch `um_ha.py`). A call (engine A) is entered only if the stock's last CLOSED HA candle is green; a put (engine B) only if it is red.

| Filter | Total | vs deployed | Aug | Sep | Signals blocked (A / B) |
|---|---:|---:|---:|---:|---:|
| None (deployed) | +149,360 | - | +57,927 | +91,434 | - |
| SHA 1-min | +82,382 | **-66,978** | -17.7k | -49.3k | 45 / 69 |
| SHA 3-min | +121,718 | -27,642 | -3.8k | -23.8k | 5 / 68 |
| SHA 5-min | +156,429 | +7,069 | +2.2k | +4.9k | 2 / 68 |
| SHA 5-min, calls only | +158,436 | +9,076 | +2.2k | +6.9k | 2 / 0 |
| SHA 5-min, puts only | +147,352 | -2,008 | 0 | -2.0k | 0 / 68 |
| SHA 1-min, puts only | +151,969 | +2,609 | 0 | +2.6k | 0 / 69 |
| Fast SHA (6/2) 1-min | +106,881 | -42,479 | +5.7k | -48.2k | 29 / 68 |
| Fast SHA (6/2) 5-min | +130,410 | -18,950 | +1.0k | -19.9k | 19 / 67 |
| Classic HA 5-min | +68,843 | -80,517 | -25.0k | -55.5k | 52 / 69 |
| Classic HA 1-min | +138,207 | -11,153 | +4.1k | -15.3k | 25 / 64 |

**Why:**
- **Shorter is worse, not better.** Engine A buys the breakout right after a pullback. During that pullback the 1-min HA is red, so the filter blocks the best entries: 29 traded calls worth +67k, including the +26,288 / +16,817 / +15,399 winners.
- **On 5-min the SHA is almost always already green when A triggers** - it blocked only 2 calls in 43 sessions (HAL -2,225, PAYTM -6,851). That +9k is two trades: chance, not evidence.
- **For engine B** the filter blocks over half the put signals but moves P&L by only about +-2.5k (no information).

**Verdict:** do not add an HA filter to UM. At most, log the 5-min SHA colour at each real entry (shadow) to gather out-of-sample evidence.
