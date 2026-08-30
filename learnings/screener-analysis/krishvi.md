# Screener analysis: Krishvi (chartink.com/screener/krishvi)

Fetched directly from the live Chartink page, 30 Aug 2026. Author: Kushal
Gaur (the user). Evaluated on the futures segment.

## Full filter logic (all ANDed unless noted)

| # | Condition | What it checks |
|---|---|---|
| 1 | Daily % Change > 1 | Stock already up >1% for the day |
| 2 | Daily Drawn pattern(Daily Close, 80,22,85,15) = 1 *(appears twice, identically, inside an "any 1 of" sub-group — see below)* | A hand-drawn chart-shape match via Chartink's pattern tool |
| 3 | 5-min RSI(14) < 80 | Excludes only extreme overbought — a loose ceiling, rarely binding |
| 4 | 5-min Close ≥ 5-min Supertrend(**7,3**) | Price in a bullish 5-min trend regime |
| 5 | 1-min Close **crossed above** 1-min Supertrend(**7,3**) | The actual entry trigger — edge-detected, not a state check |
| 6 | 5-min ROC(9, Close) **crossed above** 0 | Momentum just turned positive |
| 7 | 5-min ROC(9, Close) > 0 | Same fact restated as a separate clause |
| 8 | 5-min Close > 1-day-ago Close | Above yesterday's close |
| 9 | 5-min Close > Daily Open | Above today's own open |

Structurally: #1 sits at the top level. #2's two identical clauses are
wrapped in their own "any 1 of" sub-group (visually nested/indented under
#1). Everything else (#3–#9) sits at the same outer indentation as #1 — i.e.
ANDed with it, not further nested.

## Assessment

This is a genuinely well-built multi-timeframe trend-confirmation scan, not
a naive "stock is up X%" screener: a daily momentum gate (#1), a mid-term
trend filter (#4), a precise edge-detected entry trigger (#5), a momentum-
turning confirmation (#6/#7), and two basic intraday-strength checks
(#8/#9). More disciplined than most retail Chartink scans.

**Two design issues worth knowing about, neither breaking, both worth fixing
for clarity:**

1. **Redundant clauses.** #7 is functionally implied by #6 — "ROC crosses
   above 0" already means "ROC > 0" at that same bar; #7 can only ever
   independently matter on a later bar without a fresh crossover, but #5's
   own crossover-based entry trigger already requires a fresh signal on that
   bar anyway. Similarly, #2's "any 1 of" wraps two *identical* conditions,
   so the OR has zero effect. Neither is wrong, just untrimmed — worth
   cleaning up only for scan-maintenance clarity, not because it changes
   behavior.
2. **Not something I can verify from outside Chartink:** the "Drawn
   pattern(Daily Close, 80,22,85,15)" condition is a hand-drawn shape match
   — the numbers are the pattern's stored coordinates/scale, not something
   interpretable without seeing the actual drawn shape rendered on
   Chartink's own charting tool. Treat this clause as a black box unless
   someone opens the scan editor and looks at the drawn line directly.

## Parameter mismatch against the bot's own Supertrend — the important one

Krishvi's entry trigger (#5) uses **Supertrend(period=7, multiplier=3)** on
both the 1-min and 5-min timeframes. DhanBoy's own exit-side Supertrend
check (`Options/config.py`: `SUPERTREND_PERIOD=10`, `SUPERTREND_
MULTIPLIER=3.0`, `SUPERTREND_INTERVAL_MINUTES=5`) uses **period=10**, not 7.

Period 7 is meaningfully more sensitive/faster than period 10 — it flips
direction sooner on the same price action. This means: **the screener enters
on a quicker trend-change signal than the signal the bot later uses to
decide when to exit.** Not necessarily wrong (an entry can reasonably want
to be more reactive than an exit), but it's an asymmetry worth being
deliberate about rather than accidental. Worth asking: was this intentional,
or should the bot's exit-side Supertrend period be brought down to 7 to
match the entry logic it's actually paired with?

## One coincidental validation

The `ROC(9)` used in Krishvi's own entry logic (#6/#7) matches the ROC
period independently chosen for the SAGILITY momentum chart built this
session — good confirmation that period-9 ROC is a sensible, non-arbitrary
default for this kind of intraday-momentum read, at least for this
screener's own design intent.

## Note on "futures segment"

The scan evaluates its technical conditions against the futures contract's
price series, not equity cash — this is a Chartink data-source choice for
the screener's own calculations, not an instruction to trade futures. The
bot itself trades ATM *options* on the underlying equity in response to
these alerts; the futures price closely tracks equity spot with only a
minor basis difference, so this doesn't materially change what the alert
means for the bot's own strategy.
