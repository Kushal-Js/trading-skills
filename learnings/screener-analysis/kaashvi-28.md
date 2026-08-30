# Screener analysis: Kaashvi-28 (chartink.com/screener/kaashvi-28)

Fetched directly from the live Chartink page, 30 Aug 2026. Author: Kushal
Gaur (the user). Evaluated on the futures segment. Near-identical in
structure to `dandanadan-2.md` — this file focuses on the one real
difference, confirmed by direct comparison of both pages' filter trees.

## Full filter logic (all ANDed unless noted)

| # | Condition | What it checks |
|---|---|---|
| 1 | Daily % Change > 1 | Stock already up >1% for the day |
| 2 | Daily Drawn pattern(Daily Close, 80,22,85,15) = 1 *(twice, "any 1 of" sub-group — redundant, see `krishvi.md`)* | Hand-drawn chart-shape match |
| 3 | 5-min RSI(14) < 85 | Exhaustion ceiling |
| 4 | 5-min Close ≥ 5-min Supertrend(**7,3**) | Bullish 5-min trend regime |
| 5 | **"any 1 of" group**: [0] 1-min Drawn pattern(...)=1 *(twice, redundant)* | The entry trigger |

## Confirmed missing vs. DanDanaDan-2 — not a scraping artifact

Kaashvi-28's final "any 1 of" group contains **only the two duplicate
1-min drawn-pattern clauses** — it does **not** have DanDanaDan-2's extra
two OR-branches (`Daily Close > Daily Supertrend(7,3)` and `5-min ROC(9)
crossed above 0`). Verified by loading both pages directly and comparing
their filter trees side by side, including a screenshot of each — this
isn't a page-load or extraction glitch, the trees are genuinely different
lengths.

**Practical consequence: since the two duplicate drawn-pattern clauses are
functionally one condition (an OR of two identical things has no effect),
Kaashvi-28's entire entry trigger reduces to a single hand-drawn
chart-pattern match at the 1-minute timeframe** — no ROC confirmation, no
Daily-Supertrend confirmation, unlike its sibling screener. Whatever
edge this scan has comes almost entirely from clauses 1–4 (the daily
momentum gate + RSI ceiling + 5-min trend regime) plus that one pattern
match — it's the least redundantly-confirmed entry trigger of the three
screeners analyzed so far (`krishvi.md`, `dandanadan-2.md`, this one).

## Everything else

Identical to DanDanaDan-2: same Supertrend(7,3) vs. the bot's own (10,3)
mismatch applies to clause 4 here too (see `krishvi.md` for the full
reasoning on why that matters).

## Backtest correlation

The `02 Kaashvi.csv` backtests run this session (4-day and CE=4/TOP_N=4
config variants) showed solid but not exceptional results (~56–61% win
rate depending on config) — consistent with a screener whose entry trigger
leans on fewer confirming signals than its sibling. Same caveat as
`dandanadan-2.md`: not established as causal from this sample size alone.
