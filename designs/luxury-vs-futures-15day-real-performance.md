# Luxury vs Futures — real trade performance, last 15 days (31 Aug – 18 Sep 2026)

Source: droplet's own `history/*_real_trades.log` (every closed real trade,
both strategies, `reconciled: true` where applicable). 13 trading days have
data in this window (no file for 2 Sep, 14 Sep, 19–20 Sep — weekends/no
trades logged); 21 Sep had no trades yet at pull time (market not open).

All numbers are **before** the 21 Sep risk-config raise and breakout-signal
threshold change on Luxury (see [[luxury-signal-gated-live-simulation]] and
[[luxury-breakout-detection-parameter-sweep]]) — this is what actually
happened under the *older* config, not a projection.

## Headline comparison

| Metric | Luxury | Futures |
|---|---:|---:|
| Trades | 125 | 50 |
| Wins / Losses / Flat | 45 / 78 / 2 | 19 / 31 / 0 |
| Win rate | 36.0% | 38.0% |
| Total PnL | **-17,909.35** | **-11,943.55** |
| Gross win | 61,250.15 | 25,413.75 |
| Gross loss | -79,159.50 | -37,357.30 |
| Profit factor | 0.77 | 0.68 |
| Avg win | 1,361.11 | 1,337.57 |
| Avg loss | -1,014.87 | -1,205.07 |
| Avg PnL/trade | -143.27 | -238.87 |
| Days traded | 10 of 13 | 7 of 13 |
| Positive days | 3 | 3 |
| Negative days | 7 | 4 |
| Best trade | ADANIENSOL +4,286.25 (TARGET_HIT, 17 Sep) | LTM +3,697.50 (TARGET_HIT, 15 Sep) |
| Worst trade | POLYCAB -2,881.25 (STOP_LOSS_HIT, 18 Sep) | PAYTM -4,603.75 (MAX_LOSS_HIT, 16 Sep) |

Both strategies lost money over this window. **Futures lost less in
absolute terms (-11.9k vs -17.9k) but was proportionally worse per trade**
(-238.87 avg/trade vs -143.27) and had the lower profit factor (0.68 vs
0.77) — Futures loses more per losing trade on average and wins slightly
less often, it just traded less than half as often as Luxury (50 vs 125
trades) so the total drawdown looks smaller.

## Exit-reason breakdown

**Luxury (125 trades):**

| Exit reason | Trades | PnL |
|---|---:|---:|
| PROFIT_PROTECTION_HIT | 34 | +43,517.15 |
| TARGET_HIT | 6 | +14,623.75 |
| MAX_LOSS_HIT | 25 | -34,508.50 |
| STOP_LOSS_HIT | 13 | -16,539.00 |
| LIQUIDITY_GUARD_ZERO_VOLUME | 12 | -9,140.50 |
| SUPERTREND_EXIT | 25 | -6,259.50 |
| EMA_CROSS_EXIT | 5 | -5,042.75 |
| EOD_SQUARE_OFF_FRIDAY | 4 | -2,320.00 |
| RECONCILED_ALREADY_FLAT | 1 | -2,240.00 |

**Futures (50 trades):**

| Exit reason | Trades | PnL |
|---|---:|---:|
| PROFIT_PROTECTION_HIT | 10 | +13,403.75 |
| TARGET_HIT | 4 | +10,262.50 |
| MAX_LOSS_HIT | 5 | -12,393.25 |
| STOP_LOSS_HIT | 4 | -9,942.80 |
| LIQUIDITY_GUARD_ZERO_VOLUME | 14 | -7,637.50 |
| SUPERTREND_EXIT | 13 | -5,636.25 |

Same pattern in both: **PROFIT_PROTECTION_HIT is the single biggest
profit contributor** (Luxury +43.5k across 34 trades, Futures +13.4k
across 10), and **MAX_LOSS_HIT + STOP_LOSS_HIT together are the biggest
drag** (Luxury -51k combined, Futures -22.3k combined). Futures leans
much more heavily on LIQUIDITY_GUARD_ZERO_VOLUME (14 of 50 trades, 28%)
than Luxury (12 of 125, 9.6%) — Futures contracts are getting caught by
the zero-volume liquidity guard proportionally 3x more often, worth a
closer look at Futures' liquidity/strike-selection behavior separately.

## Day-wise PnL

| Date | Luxury Trades | Luxury PnL | Futures Trades | Futures PnL |
|---|---:|---:|---:|---:|
| 31 Aug | 0 | 0.00 | 2 | +55.50 |
| 01 Sep | 7 | -3,240.00 | 2 | +2,880.00 |
| 03 Sep | 25 | -10,229.00 | 0 | 0.00 |
| 04 Sep | 12 | -361.50 | 0 | 0.00 |
| 08 Sep | 4 | +1,951.25 | 0 | 0.00 |
| 09 Sep | 8 | -3,857.50 | 0 | 0.00 |
| 10 Sep | 6 | -1,402.50 | 0 | 0.00 |
| 11 Sep | 1 | -1,338.75 | 2 | -3,622.50 |
| 15 Sep | 0 | 0.00 | 2 | +5,227.50 |
| 16 Sep | 3 | +6,139.25 | 4 | -8,603.30 |
| 17 Sep | 26 | +5,130.00 | 23 | -5,741.75 |
| 18 Sep | 33 | -10,700.60 | 15 | -2,139.00 |
| **Cumulative** | | **-17,909.35** | | **-11,943.55** |

Notable: Luxury and Futures rarely have overlapping bad/good days —
16 Sep was Luxury's best day (+6.1k) and Futures' worst day (-8.6k);
17 Sep was the reverse direction but both traded heavily that day
(26 and 23 trades respectively, likely a shared high-volume Chartink
alert day). 3 Sep and 18 Sep are Luxury's two worst days and account
for -20.9k of its -17.9k total on their own (i.e., every other day
combined was net positive) — Luxury's loss is concentrated in two
outlier days, not a steady bleed.

## Caveats

- 13 trading days of data, not 15 — no trades logged on 2 of the 15
  calendar days in this window, plus weekends.
- This reflects the **pre-21-Sep** Luxury config (before the risk-cap
  raise and breakout-threshold change) — not comparable going forward
  without re-pulling after those changes have had time to accumulate
  their own trade history.
- Futures traded far less often (7 of 13 days vs Luxury's 10 of 13) —
  its smaller total loss is partly a function of fewer opportunities,
  not necessarily a better hit rate per opportunity (its avg loss/trade
  and profit factor are both worse than Luxury's).
- No Options/Swing comparison here — scoped to Luxury vs Futures only,
  per the request.
