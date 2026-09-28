# Swing's broker-side stop-loss order sent with a NEGATIVE trigger price (open - check later)

**Status: RESOLVED - already fixed on 25 Sep 2026, same day as the incident.** This note was first written on 28 Sep as OPEN without checking the code; that was wrong. `Swing/position_store.broker_stop_trigger_and_limit` gained a `hard_stop_pct` argument on 25 Sep that takes the TIGHTER of the rupee-cap trigger and the percentage stop, so the trigger can no longer go negative; its docstring cites this exact VEDL incident. Both call sites pass it (Swing: `HARD_STOP_LOSS_PCT`; Bollinger: the trade's `stop_pct`, or a 95% backstop in hold_to_close mode since 28 Sep). Kept for the record.

## What happened (25 Sep 2026, while Swing was accidentally live-real)

Two real VEDL 29 SEP 265 CALL entries (qty 1150) each tried to place the broker-side SL-L order with a negative price, and Dhan rejected both (`order_placement returned no order id`):

| Entry (IST) | Fill | SL-L sent |
|---|---|---|
| 09:30 | 2.95 | trigger -0.95, limit -1.15 |
| 14:05 | 2.25 | trigger -1.65, limit -1.85 |

The code logged the failure and carried on with only the poll/tick-driven MAX_LOSS_HIT check, so both positions had no resting stop at the broker. (They closed at -Rs 747.50 via STOP_LOSS_HIT and -Rs 460 via SUPERTREND_REVERSAL.)

## Likely cause (numbers match exactly, code not yet read)

`Swing/trading_engine.py` ~line 815 calls `broker_stop_trigger_and_limit(side, fill_price, pnl_multiplier, MAX_LOSS_PROTECTION_RS, ...)`. The rupee cap spread across the lot is `4500 / 1150 = 3.91` per unit, which is bigger than the whole premium: `2.95 - 3.91 = -0.96` and `2.25 - 3.91 = -1.66`, the exact pre-rounding triggers in the log. For a cheap option with a big lot, the max-loss-derived stop sits below zero and nothing floors it (the `hard_stop_pct` argument evidently isn't taking over in that case).

## To check when picking this up

- Read `broker_stop_trigger_and_limit`: which of the rupee cap vs `HARD_STOP_LOSS_PCT` wins, and why a non-positive trigger isn't rejected or floored before sending.
- Bollinger uses the same broker stop mechanism (`BOLLINGER_BROKER_STOP_LOSS_ENABLED=true`) with the same 4500 cap, so it likely has the same gap on low-premium contracts.
- Any fix should clamp to a valid positive tick (or fall back to the percentage stop) and alert loudly when the broker stop can't be placed, rather than continuing silently unprotected.
