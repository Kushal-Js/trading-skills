# Dual-EMA-band (external idea) vs Swing v3 (live default) — day-wise NIFTY comparison

User request 24 Sep 2026: "compare this v3 Swing and show me PnL
differences day wise for both strategies." Companion to
[[dual-ema-band-directional-strategy-animesh-k]] — read that first for what
the Dual-EMA-band strategy actually is and how its rupee figures were
produced.

## What's being compared, and why it's NOT apples-to-apples

- **Swing v3** (`Swing/config.py`'s promoted default, 24 Sep 2026 — see
  `designs/nifty-options-swing-v2-1min-v3.md`): the actual live-default
  NIFTY/BankNifty options entry rule (5-min fast layer, 0.6x volume-floor
  gate, Day-Range Bull/Bear OR-branch, Supertrend-reversal/target/max-loss
  exit ladder). Its 30-day backtest (`results_nifty_options_swing_v2_5min_
  volgate0.60_30day.json`) used **real quoted option premiums** from a
  single still-listed contract (the 2026-09-29 expiry, which happened to
  already have real trading history stretching back into August) — genuine
  data, not modeled.
- **Dual-EMA-band**: an external YouTube strategy idea (Animesh K), never
  deployed anywhere, backtested here using **Black-Scholes-estimated**
  premiums (real spot + real India VIX, but a modeled option price, not a
  real quote) — see the companion file for why real premiums were
  unfetchable for its window.

Comparing them is useful for shape/direction, **not** as a fair head-to-
head — one side has real execution-grade data, the other has a
theoretical proxy known (from the companion file) to be dominated by a
single outsized day.

## Day-wise PnL (Rs, per lot — NIFTY lot size 65 both sides)

| Day | v3 Swing | DualEMA 5-min | DualEMA 15-min |
|---|---:|---:|---:|
| 2026-08-13 | — | -1,954 | -2,050 |
| 2026-08-14 | — | -1,162 | -2,398 |
| 2026-08-17 | — | +423 | -1,348 |
| 2026-08-18 | — | +86 | +4,141 |
| 2026-08-19 | — | +1,075 | +425 |
| 2026-08-20 | — | -469 | — |
| 2026-08-21 | — | -1,210 | +448 |
| 2026-08-24 | +2,096 | -680 | -2,067 |
| 2026-08-25 | -1,144 | +9,297 | +7,664 |
| 2026-08-26 | +2,428 | -3,715 | -4,401 |
| 2026-08-27 | +751 | -1,194 | — |
| 2026-08-28 | +156 | -946 | -951 |
| 2026-08-31 | -1,505 | -1,772 | -1,495 |
| 2026-09-01 | +2,928 | -1,332 | -3,145 |
| 2026-09-02 | -1,160 | -2,601 | -2,527 |
| 2026-09-03 | +981 | +591 | +1,036 |
| 2026-09-04 | +3,061 | +1,011 | 0 |
| 2026-09-07 | +374 | +1,783 | +1,598 |
| 2026-09-08 | +5,597 | +2,216 | -138 |
| 2026-09-09 | +1,313 | +867 | +2,382 |
| 2026-09-10 | -4,059 | -1,351 | -1,389 |
| 2026-09-11 | -1,986 | -2,287 | -3,919 |
| 2026-09-15 | +2,486 | **+16,086** | **+13,162** |
| 2026-09-16 | -1,898 | -1,889 | -578 |
| 2026-09-17 | +1,053 | -1,816 | -1,693 |
| 2026-09-18 | +84 | — | — |
| 2026-09-21 | +2,450 | +486 | +34 |
| 2026-09-22 | +744 | -1,232 | -1,373 |
| 2026-09-23 | — (1 trade still open, excluded) | -227 | +1,106 |
| **Total** | **+14,752** | **+8,085** | **+2,522** |

## What the day-wise view actually shows

- **v3 only trades on days its own signal fires** (22 of 29 days, some
  multi-trade) — Dual-EMA-band trades on almost every single day by
  construction (its rule always looks for a band-break once biased). This
  alone means v3's day-wise series is sparser and each entry is more
  selective.
- **Restricting to the 20 days BOTH strategies traded** (2026-08-24
  onward, v3's window start): v3 nets **+14,667** on those days,
  Dual-EMA-5min nets **+11,523** — closer than the full-window totals
  suggest, since Dual-EMA's early Aug 13-21 losing stretch (before v3 ever
  traded) is excluded.
- **Directional agreement is weak**: only 12 of those 20 overlap days have
  the same sign (both win or both lose) — 60%, barely better than a coin
  flip. Pearson correlation of daily P&L on the overlap set: **0.213**
  (weak positive, not a shared driver). The two systems are reading
  meaningfully different signals from the same instrument, not variations
  on the same idea.
- **2026-09-15 is the standout day for BOTH**, but at wildly different
  scale — v3 +2,486 (in line with its other winning days) vs Dual-EMA
  +16,086 on 5-min / +13,162 on 15-min (as documented in the companion
  file, this single day is the difference between Dual-EMA's headline
  "net positive" result and a net loss). v3's P&L is already confirmed
  spread across 13 different days out of 22 (see the design doc's own
  "Open questions" section) — a materially more distributed, less
  single-day-dependent result than Dual-EMA-band's.
- **2026-09-10/09-11** is a shared bad stretch for both (v3 -4,059/-1,986,
  Dual-EMA-5min -1,351/-2,287) — the one place both systems agree in the
  same direction with real magnitude, worth a look if either strategy
  ever gets a same-day-loss circuit breaker (already flagged as an open
  question for v3 in the design doc).

## Bottom line

Not a fair contest — v3 is the live-validated default built on real
quoted premiums; Dual-EMA-band is an unvetted external idea priced with a
theoretical model and (per the companion file) dependent on one lucky day
for its entire net-positive result. The useful takeaway here is the low
correlation (0.213) and weak directional agreement (60%), which says the
two systems aren't redundant with each other — not that Dual-EMA-band is
comparably reliable to v3.
