# Unified Momentum watchlist (HYBRID) review - 3 Oct 2026

User: "stock selection is the most deciding factor ... check the watchlist updating function for Unified Momentum ... is it the best method ... are the stocks bound to rise or to have continued momentum?"

**Answer: HYBRID is the best of everything tested, but it does NOT pick stocks that are bound to rise.** It picks stocks where engine A's intraday pullback entries have recently worked. Those stocks hold up somewhat better than the market, but about half still fall the following week.

## Method (stock_selection.hybrid_select, weekly Friday job)

1. Liquidity gate: ATM premium >= Rs 5 and 20-day turnover >= Rs 50 cr.
2. Top 40 by the ATH composite score.
3. Super Bollinger / engine A rules replayed on each stock's 5-min bars over the last 20 sessions (plain fit, no 1-hour filter).
4. Top 15 by fit (>= 3 trades), filled from the ATH pool if short.

## Tests

**1. Price follow-through.** 9 walk-forward weeks (picks 31 Jul - 25 Sep, `history/bt_walkforward_long/selection_walkforward.json`), daily Yahoo closes, average % per stock vs all F&O stocks:

| List | Next 5 days | Universe | Picks up | Next 10 days | Universe | Beat universe (10d) |
|---|---:|---:|---:|---:|---:|---:|
| HYBRID (live, 15) | -0.14 | -0.44 | 47% | -0.06 | -1.01 | 6/7 |
| HYBRID top 10 | +0.22 | -0.44 | 50% | +0.08 | -1.01 | 7/7 |
| HYBRID top 20 | -0.13 | -0.44 | 45% | -0.10 | -1.01 | 6/7 |
| MOMENTUM | -0.34 | -0.44 | 44% | -0.54 | -1.01 | 4/7 |
| TREND_ATH (ATH only) | -0.80 | -0.44 | 36% | -1.28 | -1.01 | 3/7 |
| Random (10 lists) | -0.43 | -0.44 | 38% | -0.88 | -1.01 | 34/70 |

In the last two weeks (11 and 18 Sep) the picks did WORSE than the universe (-0.76 / -1.14 pp over 5 days).

**2. UM backtest with list variants** (deployed config, 3 Aug - 1 Oct, 43 sessions):

| List | Total | Max dd |
|---|---:|---:|
| Top 15 (live) | **+149,360** | -13,712 |
| Sticky (enter top 15, leave below top 20; 15-17 stocks) | +144,415 | -13,712 |
| Top 20 | +126,212 | -17,327 |
| Top 10 | +125,238 | -15,387 |

Live is best in both months.

**3. Tenure and churn.**
- Stocks staying from earlier weeks earned Rs 1,358 per stock-week vs Rs 635 in a stock's first week.
- 4-8 of 15 stocks are replaced each week.
- Week-to-week P&L rank correlation per stock is +0.12 (weak).
- The sticky rule still did not help (holdovers ranked 16-20 add little).

**4. Regime.** Prior 14-week research (`research_results/2026-09-30_selection_rules_longer.txt`): HYBRID beat 98% of random lists overall, 100% when NIFTY was below its 20-day average, 54% (= random) when above. On 1 Oct NIFTY was 3.6% below its 20-day average (22,422 vs 23,269).

## Takeaways

- Keep the live rule.
- The only candidate that ever scored higher (40-session fit with the 1-hour filter) is being shadow-scored weekly since 2 Oct. Decide after 6-8 scored weeks.
- Untested: the fit uses only engine A's rules, yet engine B's puts trade the same list (a separate engine B fit is untested); fewer slots in "NIFTY above 20-day average" weeks (backtested earlier, waiting for more data).
