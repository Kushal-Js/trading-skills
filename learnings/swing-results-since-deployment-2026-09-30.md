# Swing: real and paper results since deployment (as of 30 Sep 2026 ~10:20 IST)

**Sources:**
- `history/*_real_trades.log` (strategy Swing) and `history/*_swing_paper_trades.log` from the droplet.
- MCX P&L before the 21 Sep logging fix is corrected by the lot multiplier (COPPER 2500, NATURALGAS 1250). This matches TRADING_JOURNAL's corrected 1–18 Sep Swing total of −10,639.
- Six early 28 Sep paper rows had no `pnl_modeled`; it was recomputed with the bot's own slippage model.

## Real: 61 trades, 22 wins (36%), −Rs 45,526 (1–29 Sep)

| Phase | Days | Trades | Wins | P&L | Stocks | Index | MCX |
|---|---|---:|---:|---:|---:|---:|---:|
| v1 basket (futures + PE) | 1–8 Sep | 8 | 4 | −7,794 | −7,794 | – | – |
| v2 / v2.1 rewrite | 15–23 Sep | 21 | 10 | −13,482 | −12,182 (7) | – | −1,300 (14) |
| v3 default, 35% target, index added | 24–25 Sep | 18 | 4 | −10,562 | −2,764 (4)* | −610 (7) | −7,188 (7) |
| Stocks paper, MCX + index real | 28–29 Sep | 14 | 4 | −13,688 | – | −1,450 (5) | −12,238 (9) |

\*25 Sep stock trades placed while Swing was wrongly still live (paper-mode override incident, fixed 27 Sep).

**By day:**

| Date | P&L | Running total |
|---|---:|---:|
| 1 Sep | −6,930 | −6,930 |
| 3 Sep | +38 | −6,892 |
| 4 Sep | +2,219 | −4,674 |
| 7 Sep | −3,120 | −7,794 |
| 8 Sep | 0 | −7,794 |
| 15 Sep | −435 | −8,229 |
| 17 Sep | −3,248 | −11,476 |
| 18 Sep | +838 | −10,639 |
| 21 Sep | −6,288 | −16,926 |
| 22 Sep | −2,900 | −19,826 |
| 23 Sep | −1,450 | −21,276 |
| 24 Sep | −300 | −21,576 |
| 25 Sep | −10,261 | −31,838 |
| 28 Sep | −4,824 | −36,662 |
| 29 Sep | −8,864 | −45,526 |

**By segment:**

| Segment | Trades | P&L |
|---|---:|---:|
| Stocks | 19 | −22,740 |
| MCX | 30 | −20,725 |
| Index | 12 | −2,061 |

Average win +1,819 vs average loss −2,194.

**By exit reason:**

| Direction | Exit | P&L | Trades |
|---|---|---:|---:|
| Profit | Profit protection | +22,438 | 10 |
| Profit | Target | +10,984 | 7 |
| Loss | Max loss | −21,272 | 5 |
| Loss | Stop loss | −15,658 | 9 |
| Loss | COPPER structure break | −15,362 | 7 |
| Loss | Supertrend reversals / 5-min exits | about −24,700 | – |

## Paper: 62 trades, 23 wins (37%), −6,973 raw / −21,052 modeled (28–30 Sep)

| Date | Trades | Wins | Raw | Modeled |
|---|---:|---:|---:|---:|
| 28 Sep | 29 | 8 | −11,354 | −15,489 |
| 29 Sep | 22 | 9 | −11,300 | −18,273 |
| 30 Sep | 11 | 6 | +15,681 | +12,711 |

On 30 Sep, five quick PE wins on a bearish open (AUROPHARMA ×2, DIVISLAB ×2, ZYDUSLIFE) made about +16k.

**Reading.** Swing has never had a positive real week. MCX (COPPER max-loss hits of about −4.6k to −5.3k each) and stocks carry the losses; index is roughly flat. One strong paper morning doesn't change the picture. Re-evaluate on several weeks of modeled paper P&L before any stock/MCX Swing goes back to real.
