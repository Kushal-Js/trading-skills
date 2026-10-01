# Super Bollinger: bypassing the 1-hour filter on "strong momentum" lets through mostly losers (1 Oct 2026)

**Question (user, 1 Oct 2026 ~10:20 IST, after PAGEIND's 10:07 trigger was skipped by the filter):** should the 1-hour-green entry filter be bypassed when upside momentum is very strong?

**Test.** traderBoy `c9ede6e`, `research_super_bollinger_1h_momentum_override.py`, results `research_results/2026-10-01_super_bollinger_1h_momentum_override.txt`.

- Setup: cache-only, in-sample, HYBRID walk-forward 3 Aug–29 Sep, 5 slots, calls only (and calls + the live hedge).
- Rule tested: "last closed 60-min candle green **OR** override".
- Every override uses only data known at the trigger touch.

| Rule | Net (calls only) | Max drawdown | Let through | Won | P&L of the let-through trades |
|---|---:|---:|---:|---:|---:|
| Live: 1-hour green | **+94,945** | **−22,773** | – | – | – |
| No filter | +63,495 | −45,752 | – | – | – |
| OR day up ≥ 1.0% and above open | +93,519 | −29,599 | 8 | 2 | −6,610 |
| OR day up ≥ 1.5% and above open | +95,403 | −29,853 | 4 | 1 | +458 |
| OR volume surge (green bar, ≥ 2× 20-bar avg) | +95,200 | −22,519 | 1 | 0 | −4,929 |
| OR ADX(5m) ≥ 25 and 15m Supertrend up | +81,941 | −28,674 | 11 | 1 | −10,935 |
| OR efficiency ratio(10) ≥ 0.6 | +86,470 | −22,773 | 5 | 0 | −8,759 |
| OR forming hour up ≥ 0.5% (the PAGEIND case) | +83,491 | −29,853 | 10 | 1 | −11,737 |
| OR above yesterday's high and the 30-min high | +90,116 | −27,602 | 1 | 0 | −4,829 |
| OR strict: ADX ≥ 25, ER ≥ 0.5, above open | +83,303 | −29,853 | 5 | 0 | −11,642 |

**Reading.** In every definition, the trades that the filter refused but "strong momentum" would have allowed won 0–2 times. The two best versions (day up ≥ 1.5%, volume surge) change only 1–4 trades: that's noise, and the drawdown still gets worse. A strong move while the last hour closed red is usually a spike into resistance that fades, and that's exactly what the filter screens out. This agrees with the 1 Oct alignment test: a rolling last-60-minute candle is almost always green at a breakout, and the result was worse (+61.6k vs +94.9k).

**Caveat.** This is in-sample, and the samples behind each override are small. To gather out-of-sample evidence without risk, log the momentum features on each live `ENTRY_SKIPPED_1H_RED` event (shadow only) and re-check after a few weeks.
