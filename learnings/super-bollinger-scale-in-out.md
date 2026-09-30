# Super Bollinger: scale-in / scale-out (add a lot on the winning side)

**Date:** 30 Sep 2026. **Status:** paper variant live (traderBoy `dc6fb1c`, `scale_mode=paper`); not validated out-of-sample.

## Question (user)
Add 1 more lot on whichever side the trade is going (the CE when it wins, the supervisor's PE hedge when the
stock keeps falling), then either sell one lot quickly and let the other run, sell both once the loss is
recovered, or exit both on a profit-protection trail.

## Test
`traderBoy/research_super_bollinger_scale_in_out.py` (results in `research_super_bollinger_scale_in_out_results.txt/.json`).
Sep HYBRID walk-forward trades (92 trades, 21 days, max 5 open), G' rules unchanged for the original lot,
live hedge rule (Rs 2,000 CE loss + 1x ATR drop, stop 1,500, trail 1,000/40%). Added lots use the same
contract; modeled slippage on every leg; conservative intra-minute ordering. **In-sample** - G' and the hedge
rule were tuned on this same window.

## Results (modeled Rs, 21 days)
CE side, base G' 1 lot = +46,378 (worst day -13,368, peak premium 1.35L):

| Variant | Total | Worst day |
|---|---|---|
| 2 lots at entry, book 2nd at +2k | +30,490 | -24,227 |
| Add at +1.5k, book the add at +1-1.5k | +35-37k | -14,752 |
| Add at +1.5k, add exits with the original (breakeven) | +96,788 | -15,741 |
| **Add at +1.5k, sell the add when Supertrend(10,3) 5m turns bearish** | **+101,741** | **-13,689** |
| Add at +1.5k, sell the add at a fixed -1,000 | +41,621 | -14,752 |
| Add at +3k, sell the add on Supertrend bearish | +96,514 | -13,368 |

PE side, live hedge alone adds +14,113 (35 hedges):

| Add trigger \ exit | Sell add quickly (+1k) | Sell both when CE+PE >= 0 | Pair profit trail |
|---|---|---|---|
| Hedge already +1k | +15,819 | +15,969 | +14,126 |
| **Supertrend bearish** | +20,304 | **+22,659** (worst day -8,993 vs -11,649) | +11,836 |
| Both | +16,386 | +21,929 | +7,493 |

## Learnings
1. **On the trend side, quick booking hurts; pyramiding into a winner helps.** Same pattern as the 29 Sep exit study:
   the edge is a few big trend days; anything that cuts them (quick booking, tight fixed stops) loses more than it saves.
2. **A trend-reversal exit for the added lot beats a rupee cap.** Supertrend on 5-min bars kept the add's upside and
   cut the worst day; fixed Rs 750/1,000 stops were hit by normal noise.
3. **With 2 CE lots the hedge almost never fires**: the add happens when breakeven arms, so a reversal exits both
   lots at breakeven (add loses ~Rs 1,500) before the CE can reach the Rs 2,000 hedge trigger (gaps excepted).
4. **Hedge-side add: indicator confirmation helps modestly**, and exiting both lots as soon as the pair is back to
   breakeven beat trailing them - but only 16 adds; treat as a hypothesis.
5. **Capital:** the CE add needs up to ~2.4L peak premium vs ~1.2L account; with a 1.2L funds limit the result is
   still positive but different trades get skipped, so live != backtest unless capital or max_concurrent changes.

## Next
August out-of-sample run (queued for after 15:30 on 30 Sep), then compare the paper variant's log
(`GET /super-bollinger/scale`) with the backtest for 1-2 weeks before any real-money version (which also needs
partial-quantity broker SL handling in the live engine).
