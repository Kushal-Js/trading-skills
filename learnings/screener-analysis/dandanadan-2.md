# Screener analysis: DanDanaDan-2 (chartink.com/screener/dandanadan-2)

Fetched directly from the live Chartink page, 30 Aug 2026. Author: Kushal
Gaur (the user). Evaluated on the futures segment. See `krishvi.md` for
the sibling analysis this one is directly comparable to.

## Full filter logic (all ANDed unless noted)

| # | Condition | What it checks |
|---|---|---|
| 1 | Daily % Change > 1 | Stock already up >1% for the day |
| 2 | Daily Drawn pattern(Daily Close, 80,22,85,15) = 1 *(appears twice, in an "any 1 of" sub-group — the recurring redundant-OR pattern, see `krishvi.md`)* | Hand-drawn chart-shape match |
| 3 | 5-min RSI(14) < 85 | Exhaustion ceiling — looser than Krishvi's `<80` |
| 4 | 5-min Close ≥ 5-min Supertrend(**7,3**) | Bullish 5-min trend regime |
| 5 | **A second "any 1 of" group**: [0] 1-min Drawn pattern(...)=1 *(twice, again redundant)* OR Daily Close > Daily Supertrend(7,3) OR 5-min ROC(9, Close) crossed above 0 | The entry trigger — but structured as an OR, not an AND like Krishvi's stacked conditions |

## The structural difference from Krishvi — this is the important finding

Krishvi's entry logic **ANDs** three independent confirmations together (1-min Supertrend crossover AND 5-min ROC crossing above 0 AND the implicit 5-min-Supertrend-regime check from clause 4) — every one of them has to agree.

DanDanaDan-2's final group is an **OR**: it fires if *any one* of {a 1-min drawn-pattern match, Daily Close above Daily Supertrend, 5-min ROC crossing above 0} is true. This is a meaningfully looser trigger — a single one of three independent, unrelated conditions is sufficient, rather than requiring agreement across signals. Expect **more alerts, lower average conviction per alert** compared to Krishvi, all else equal — this is a real, structural, not cosmetic, difference between two screeners built by the same author.

## Everything else — same as Krishvi

Daily % Change floor, the (functionally inert, duplicated) drawn-pattern OR-group, and the RSI ceiling are all structurally identical in form (just a slightly looser RSI ceiling: 85 vs. Krishvi's 80). The Supertrend period-7 vs. the bot's own period-10 mismatch (see `krishvi.md`) applies here too, on both the 5-min regime check and the new Daily-Supertrend clause in the OR-group.

## Backtest correlation

This screener's real 3-day backtest (`03 DanDanaDan-1 Day.csv`, 26–28 Aug 2026) produced 31 closed trades, 77.4% win rate, +32,944.70 — the cleanest win rate of any CSV backtested this session, with only 2 MAX_LOSS_HIT out of 31. Whether that's attributable to the looser OR-trigger catching genuinely different (and better-performing, in this sample) setups than Krishvi's AND-trigger, or is just a small-sample coincidence, isn't established — would need a much larger backtest sample directly comparing the two screeners' alert quality to say anything confident. Worth flagging as an open question rather than a conclusion.
