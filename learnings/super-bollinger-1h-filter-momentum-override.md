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

## Follow-up the same day: 9 different stock lists (traderBoy `6f780ca`)

The user asked whether the result depends on the stock list, for example the curated shadow list. Script: `research_super_bollinger_1h_override_lists.py`; results: `research_results/2026-10-01_super_bollinger_1h_override_lists.txt`. Same setup as above, run on each weekly walk-forward list.

**Net P&L, calls only:**

| List | No filter | 1-hour filter (live) | OR day up ≥1.5% | OR day up ≥1.0% | OR forming hour up ≥0.5% | OR ADX≥25 + 15m ST |
|---|---:|---:|---:|---:|---:|---:|
| SHADOW (HYBRID_F40)* | **+88.9k** (dd −42.1k) | +56.5k (dd −32.1k) | +59.7k | +60.3k | +58.4k | +44.3k |
| HYBRID (live) | +63.5k (dd −45.8k) | **+94.9k (dd −22.8k)** | +95.4k (dd −29.9k) | +93.5k | +83.5k | +81.9k |
| HYBRID top 20 | +77.5k (dd −46.8k) | +77.2k (dd −29.1k) | +77.4k | +72.7k | +62.4k | +57.8k |
| Old ATH list | −30.3k | −19.3k | −11.3k | −4.1k | −14.5k | −20.5k |
| Strategy fit (whole universe) | +41.8k | +26.2k | +34.1k | +35.3k | +29.1k | +31.8k |
| Momentum | −67.9k | −25.4k | −17.3k | −25.3k | −32.7k | −46.9k |
| Random 1 / 2 / 3 | −78.4k / −33.5k / −30.2k | −29.1k / −20.4k / +7.5k | −30.5k / −21.1k / +2.6k | −46.6k / −23.3k / −12.9k | −47.3k / −19.3k / −17.9k | −52.9k / −10.3k / −20.4k |

\*SHADOW: 23–27 of its trades had no cached option prices and were skipped.

**Bypass trades summed across all nine lists** (trades overlap between lists):

| Bypass | Let through | Won | P&L |
|---|---:|---:|---:|
| Forming hour up ≥0.5% (the PAGEIND case) | 89 | 15 | −63k |
| ADX ≥25 + 15-min Supertrend up | 97 | 18 | −74k |
| Day up ≥1.0% | 76 | 21 | −27k |
| Day up ≥1.5% | 33 | 8 | **+22k** |

The ≥1.5% bypass is the only near-neutral one: better net on 6 of 9 lists, but a worse drawdown on HYBRID and SHADOW.

**Reading:**
- The list doesn't rescue the PAGEIND-type bypass.
- The 1-hour filter cuts drawdown on 8 of 9 lists, but it isn't universally profitable: on SHADOW and on the whole-universe fit list, it costs profit.
- The best combination is still HYBRID + the 1-hour filter.

**Candidates for paper/shadow evidence, not real money:**
1. A "day up ≥1.5%" bypass.
2. The shadow list without the filter.
