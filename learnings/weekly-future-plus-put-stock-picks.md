# Weekly long stock future + ATM put on 2 picked stocks (3 Oct 2026 idea check)

User idea: buy a stock future hedged with an ATM PE, hold about a week, at most 2 stocks, picked for the highest chance of rising.

**Result: lost money in every selection rule over the last year.** Buying the put costs more than the picks' weekly edge.

## Setup

- Universe: today's 231 NSE F&O stocks, daily Yahoo closes.
- Timing: enter on the week's last close, exit the next week's last close.
- Put pricing: Black-Scholes, IV = 1.1 x the 20-day realised vol (floor 18%), 20 -> 15 trading days to expiry (no skew - real puts usually cost more, so this flatters the result).
- Future = spot minus carry.
- Costs: Dhan-style charges (futures STT 0.05%, options STT 0.15%) plus slippage (futures 0.02%/side, options 1% of premium/side).
- Size: 1 lot per pick (today's lot sizes). Scripts: scratch `fut_put.py`, `fut_put_sim.py`.

## Last year (3 Oct 2025 - 25 Sep 2026, 52 weeks, 104 trades per rule)

| Rule | Stock rose | Trade profitable | Year, hedged (Rs) | Winning weeks | Max dd | Same picks unhedged |
|---|---:|---:|---:|---:|---:|---:|
| Top 2 by 3-month gain | 57% | 44% | -89,149 | 23/52 | -241k | +248,772 (worst week/lot -130k) |
| Clean uptrend (> 50 DMA > 200 DMA), top 2 by 3M | 57% | 45% | -84,417 | 23/52 | -237k | +252,923 |
| Last week's 2 biggest gainers | 51% | 38% | -185,548 | 21/52 | -310k | +172,656 |
| Near 52-week high, top 2 by 1M | 54% | 38% | -317,422 | 20/52 | -356k | -276,508 |
| Dip in uptrend (3M > 10%, last week down) | 50% | 36% | -329,245 | 19/52 | -364k | -146,339 |
| Random 2 (100 draws/week) | 47% | 33% | -227,972 | 14/52 | -231k | -198,021 |

Over 3 years (Sep 2023 - Sep 2026) every rule was also negative hedged (-0.16L to -0.31L); only 2023 was positive.

## Takeaways

- **The put is the problem.** For these volatile picks an ATM put costs about Rs 33-38k per lot (~5-6% of a ~Rs 6.4L lot) and loses roughly Rs 5k of time value per week. With charges, a pick must rise about 1% a week just to break even.
- **Momentum selection has some value:** 57% of weeks up vs 47% for random picks, and +2.5L unhedged. But the hedge cost (~Rs 3.3L a year) is larger than that edge.
- **Unhedged is not the answer:** a single week can lose Rs 1.3L per lot.
- **Capital:** a median F&O lot is ~Rs 6L. The future margin (~Rs 1-1.5L) plus the put means two stocks need roughly Rs 3L or more; Rs 1.2L covers one at a time.
- **Not tested yet:** a cheaper hedge (OTM put or put spread) and longer holds (less decay per rupee of move).
