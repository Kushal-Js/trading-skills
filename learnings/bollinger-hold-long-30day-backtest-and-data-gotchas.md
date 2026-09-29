# Bollinger Hold-Long - 30-day backtest (31 Aug - 29 Sep 2026) + two data gotchas

Script: `traderBoy/backtest_bollinger_hold_long_30day.py` (live HOLD_LONG rules: BULLISH-only
resting entry, ATM CE, roll within 2 trading days, Rs 5 premium gate, MAX_LOSS the only early
exit, 15:15 square-off NSE, MCX exempt / Friday 23:25). Real 1-min option prices; PnL "modeled"
= paper_book slippage on both legs. Validated against 29 Sep live paper trades (APLAPOLLO /
NIFTY / BANKNIFTY entries and exits within a few rupees).

| MAX_LOSS_PROTECTION_RS | trades | win % | net modeled | net raw | MAX_LOSS exits | max DD |
|---|---|---|---|---|---|---|
| 3000 | 151 | 36.4 | +9,452 | +48,384 | 54 (-1,78,800) | -57,498 |
| 4500 | 142 | 40.8 | +37,445 | +74,475 | 26 (-1,27,377) | -39,512 |

Tighter cap is worse: 3000 cuts ~28 more trades that would have recovered by 15:15.
Slippage eats ~Rs 260-270/trade (raw vs modeled gap). Stocks carry it; indices ~flat; MCX negative
but only 4 trades priceable (29 MCX entries skipped - Sep MCX options delisted).

## Gotcha 1 - NSE equity 1-min series ends at 15:14
`dhan_wrapper.fetch_continuous_intraday` for NSE_EQ returns 360 bars/day (09:15-15:14); IDX_I and
option series run to 15:29. A backtest that squares off "when a bar >= 15:15 appears" NEVER fires
on stocks - positions silently carry for weeks. First run of this backtest showed +1 lakh LAURUSLABS
from exactly this. Square off on the day's LAST underlying bar, priced from the option's 15:15 minute,
and assert every NSE trade closes on its entry day.

## Gotcha 2 - pricing expired NIFTY weeklies
Dhan `/charts/rollingoption` (`dhanhq.expired_options_data`) works for expired index options.
`expiry_code` 1 = nearest expiry, 2 = next; on an expiry Tuesday code 1 is THAT DAY's contract.
Verified 29 Sep vs listed contracts (within Rs 0.30-0.60). It is ATM-relative: rebuild a fixed
strike minute-by-minute from the ATM+/-k series whose `strike` equals it. Monthly stock / BANKNIFTY
options are only directly fetchable until their expiry day - fetch them that evening at the latest.

## Follow-up (29 Sep, same evening) - adding the short side (BEARISH -> buy ATM PE)
Same rules, resting sell-stop on the BEARISH trigger, one position per symbol. Separate "both" variant.

| variant | trades | net modeled | long leg | short leg | max DD |
|---|---|---|---|---|---|
| long-only, 3000 | 151 | +9,452 | +9,452 | - | -57,498 |
| long-only, 4500 | 142 | +37,445 | +37,445 | - | -39,512 |
| both, 3000 | 270 | -23,374 | +14,598 | -37,972 | -83,385 |
| both, 4500 | 250 | +5,840 | +55,587 | -49,747 | -56,554 |

Short side loses on stocks (-40k at 4500) and MCX; small positive on NIFTY/BANKNIFTY (+6k, 15 trades).
Even raw (pre-slippage) the short leg is negative (-22k). Confirms the SIDES="long" research: the
BEARISH trigger has no hold-to-close edge. Rolling-option PUT series priced by matching the strike field.
