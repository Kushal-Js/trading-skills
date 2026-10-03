# Scalper rules on CRUDEOIL options (MCX): 30-day backtest (3 Oct 2026)

**Result: the strategy does not carry over to CRUDEOIL.** It lost before costs, and costs then made it heavily negative. Do not add CRUDEOIL to the Scalper (the live code refuses MCX symbols anyway).

User request: "Backtest Scalper strategy on CRUDEOIL for last 30 days and show me daywise PnL".

## Setup

- **Signal:** the live Scalper signal = Swing v3 on CLOSED 1-min candles + the 15-min layer (`research_swing_index_1min_vs_5min.signals`), computed on the CRUDEOIL Oct-2026 future (569900). The Sep future (expired 21 Sep) has no intraday data any more. 15-min candles are anchored at 09:00 (MCX open); the series is continuous from 10 Aug.
- **Option:** ATM CE/PE of the nearest monthly, rolled on expiry day: Sep options (17 Sep) until 16 Sep, Oct options (15 Oct) from 17 Sep. 1 lot = 100 bbl (Dhan's master says lot 1; `SWING_MCX_PNL_MULTIPLIER_CRUDEOIL=100`).
- **Exits (live 1-lot limits):** max loss Rs 1,500; target off; profit protection above Rs 3,000 with 2% giveback; hard stop -20%; Supertrend exit on a closed 1-min candle; fresh-formation re-entry; daily stop Rs 3,000 realised.
- **MCX timing:** entries 09:01-23:24, square-off 23:25.
- **Fills:** price exits on the option's 1-min OHLC.
- **Costs:** Rs 20/order, CTT 0.05% sell, MCX options charge 0.0418%, SEBI, stamp, GST, plus 0.5 pt slippage per fill (about Rs 210 per trade).
- **Option prices:** Dhan's expired-options API **works for MCX**: `POST /v2/charts/rollingoption` with `exchangeSegment MCX_COMM`, `instrument OPTFUT`, **`securityId 294`** (CRUDEOIL underlying - NOT the future's id, which returns empty), `expiryFlag MONTH`, `expiryCode 1/2`, `strike ATM+/-k`. Multi-day ranges work in one call. 103 calls in total. Only 4 of 5,536 held minutes lacked a price.
- Script: scratch `crude_scalper.py`.

## Results (3 Sep - 1 Oct 2026, 21 MCX days, 1 lot)

| | |
|---|---|
| Trades | 312 (15 per day), 91 won (29%) |
| Gross before costs | **-Rs 3,142** (no edge) |
| Costs + slippage | Rs 65,639 |
| **Net** | **-Rs 68,781** |
| Winning days | 4 of 21 (best +10,551 on 17 Sep, worst -10,216 on 4 Sep) |
| Max drawdown | -63,408 |
| Daily stop hit | 13 of 21 days |

Exit mix, net after costs: Supertrend close 224 trades -61.4k; max loss 54 trades -98.7k; profit protection 32 trades +91.7k; square-off 2 trades -0.4k.

## Why it fails

- **Too many trades:** the 14.5-hour MCX session plus a 1-min Supertrend gives ~15 trades a day (BANKNIFTY: ~5). At ~Rs 210 per round trip, costs alone are ~Rs 3,100 a day.
- **Stops too tight for crude:** the BANKNIFTY rupee limits on a 100-bbl lot make max loss only 15 points on a ~Rs 416 premium (3.6%), so crude's normal 1-min noise hits it.
- **No signal edge:** even before costs, the 312 trades net to about zero.

## Follow-up: same logic on 5-min candles (user, 3 Oct)

Same script with `FAST_MIN=5`: signals, Supertrend line and re-entry all use closed 5-min candles anchored at 09:00. The 15-min layer, limits and option-price exits are unchanged. Cache only.

| | 1-min (live logic) | 5-min |
|---|---:|---:|
| Trades | 312 | **68** |
| Won | 91 (29%) | 24 (35%) |
| Gross before costs | -3,142 | **+2,496** |
| Costs | 65,639 | 14,300 |
| **Net** | **-68,781** | **-11,802** |
| Positive days | 4/21 | 6/21 |
| Best / worst day | +10,551 / -10,216 | +6,634 / -4,473 |
| Max drawdown | -63,408 | -20,933 |

**5-min exit mix:** max loss 42 trades -77.2k, profit protection 22 trades +61.9k, square-off 2 trades +4.6k, Supertrend close 2 trades -1.1k. The Rs 1,500 (15-point) stop decides almost every losing trade; the 5-min Supertrend exit barely gets a chance.

**Takeaway:** 5-min cuts the trade count by 4.6x and turns gross slightly positive, but net is still negative after costs. The max-loss level, not the signal, is now the binding rule. Any next test (a crude-sized stop) is in-sample on 21 days.

## Follow-up 2: wider limits - max loss Rs 3,000, daily stop Rs 5,000 (user, 3 Oct)

Profit protection (above Rs 3,000, 2% giveback), the Supertrend candle-close exit and everything else unchanged. Cache only (`ML_RS=3000 DAILY_RS=5000`).

| | 5-min | 1-min |
|---|---:|---:|
| Trades | 71 | 363 |
| Won | 37 (52%) | 111 (31%) |
| Gross | +14,307 | +2,133 |
| Costs | 15,007 | 75,957 |
| **Net** | **-703** | **-73,824** |
| Positive days | 11/21 | 4/21 |
| Best / worst day | +8,056 / -6,665 | +14,141 / -12,288 |
| Max drawdown | **-28,586** | -65,894 |
| Daily stop hit | 4 days | 10 days |

**5-min exit mix:** max loss 24 trades -80.3k; profit protection 34 trades +93.6k; Supertrend close 7 trades -14.3k; square-off 5 trades +3.2k; hard stop 1 trade -2.9k.

**Halves:** 3-16 Sep +2,531; 17 Sep - 1 Oct -3,234.

**Gap risk:** the worst single trade was -6,403 despite the 3,000 max loss (the price gapped through the stop inside a minute).

**Takeaway:** 5-min with a crude-sized stop is about break-even (gross edge ~ costs) and is flat in both halves. The wider stop deepens the drawdown (-28.6k over 11-21 Sep). 1-min stays deeply negative on costs. No version tested so far makes money on CRUDEOIL.
