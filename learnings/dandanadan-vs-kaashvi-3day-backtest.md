# DanDanaDan-2 vs Kaashvi-28: head-to-head backtest, 26–28 Aug 2026

Both screeners' Chartink CSV exports backtested against the exact same
3 trading days, through the same production engine and the same live
config, so this is a fair like-for-like comparison — not two backtests run
under different assumptions. See `backtest-methodology.md` for what this
methodology does and doesn't capture.

**Source CSVs**: `06 DanDanaDan - 1 day.csv`, `03 Kaashvi - 1 day.csv`
(both `~/Desktop/future/`). **Config at run time**: `TARGET_PCT=0.25`,
`STOP_LOSS_PCT=0.16`, `MAX_LOSS_PER_TRADE_RS=1200`,
`PROFIT_PROTECTION_THRESHOLD_RS=1500`, `MAX_LIVE_POSITIONS_CE=3`,
`TOP_N_STOCKS=4`, `SELECT_BOTTOM_N_STOCKS=true` — i.e. today's real
production values, not any historical config.

**Data-provenance note**: `06 DanDanaDan - 1 day.csv` turned out to be a
byte-for-byte duplicate of the already-analyzed `03 DanDanaDan-1 Day.csv`
(confirmed by diffing both files and both scripts' resulting trade JSON —
identical). So the DanDanaDan side of this comparison is a *reproduction*
of a previously-run backtest, not fresh data — useful as a determinism
check (the pipeline reproduces bit-for-bit given the same input, as
expected), but it doesn't add a new sample. The Kaashvi side
(`03 Kaashvi - 1 day.csv`) is genuinely new data, covering the same 3
calendar days for a fair comparison.

## Day-wise

| Date | DanDanaDan-2 (trades / P&L / win%) | Kaashvi-28 (trades / P&L / win%) |
|---|---|---|
| 26 Aug | 13 / +₹14,084.50 / 92.3% | 40 / +₹35,870.50 / 70.0% |
| 27 Aug | 6 / +₹3,674.95 / 66.7% | 54 / +₹5,267.05 / 48.1% |
| 28 Aug | 12 (+1 open) / +₹15,185.25 / 66.7% | 56 (+2 open) / +₹24,221.50 / 53.6% |
| **Total** | **31 / +₹32,944.70 / 77.4%** | **150 / +₹65,359.05 / 56.0%** |

## Trade-wise, by exit reason

| Exit reason | DanDanaDan-2 (n / P&L) | Kaashvi-28 (n / P&L) |
|---|---|---|
| PROFIT_PROTECTION_HIT | 17 / +₹23,799.45 | 53 / +₹96,616.30 |
| TARGET_HIT | 1 / +₹8,084.00 | 4 / +₹27,354.00 |
| SUPERTREND_EXIT | 11 / +₹3,530.00 | 56 / +₹3,410.15 |
| MAX_LOSS_HIT | 2 / −₹2,468.75 | 37 / −₹62,021.40 |
| **Net** | **+₹32,944.70** | **+₹65,359.05** |

Full per-trade JSON: `dandanadan_06_ce_results.json` /
`kaashvi_03_ce_results.json` (session scratchpad — not committed here,
regenerable from the CSVs via `bt_common.py` per the standard workflow).

## Which screener actually performed better

**Raw P&L favors Kaashvi** (₹65,359 vs ₹32,945) — but that's a volume
effect, not a quality effect: Kaashvi fired **150 trades vs DanDanaDan's
31** (nearly 5x), because its trigger condition is structurally looser
(see `screener-analysis/kaashvi-28.md`) and its underlying alert frequency
is much higher (up to 164 trigger timestamps/day vs DanDanaDan's ~30).

**By trade quality, DanDanaDan-2 is clearly the better screener:**
- **Win rate: 77.4% vs 56.0%** — barely better than a coin flip for
  Kaashvi.
- **MAX_LOSS_HIT drag: −₹2,469 across 2 trades vs −₹62,021 across 37
  trades** — Kaashvi's losers are frequent and expensive, eating nearly
  all of its PROFIT_PROTECTION gains.
- **P&L per trade: ₹1,063 (DanDanaDan) vs ₹436 (Kaashvi)** — DanDanaDan
  generates roughly **2.4x the P&L per unit of capacity/risk consumed**,
  which is the number that actually matters given `MAX_LIVE_POSITIONS_CE`
  is a shared, finite resource across whichever alerts happen to fire.

## Why this isn't a surprise — it confirms a structural finding, not a coincidence

`screener-analysis/kaashvi-28.md` already established, from reading the
filter trees directly on Chartink, that Kaashvi-28's entry trigger reduces
to a single hand-drawn-pattern match with no ROC or Daily-Supertrend
confirmation (the two extra OR-branches DanDanaDan-2's own entry trigger
has). This backtest is the first *quantitative* evidence for that
structural difference, on real market data, under real production exit
logic — not just filter-tree reasoning. Treat this as reasonably solid
(a 3-day, 181-combined-trade sample from the two screeners' own real
alert history), while still keeping in mind it's one specific 3-day window,
not a multi-week validation.

## Practical implication, if this ever informs a live decision

Not acted on yet (per this repo's `SAFETY.md` — analysis only, no live
config change follows automatically from this finding). But if the bot
were ever choosing between the two screeners' alert streams under a
shared, capacity-limited pool (`MAX_LIVE_POSITIONS_CE`), this result
argues for weighting DanDanaDan-2's alerts higher, or Kaashvi-28's lower,
rather than treating both webhook sources as equally trustworthy —
consistent with, not contradicting, the structural filter-tree analysis
already on record.
