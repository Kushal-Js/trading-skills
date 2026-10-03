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
