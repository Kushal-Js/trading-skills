# Unfilled entry order left resting at the broker (SONACOMS, 30 Sep 2026)

**Strategy:** Super Bollinger (real). **Found by:** the user, in the Dhan order book ("I can see it is still open").
**Real impact:** no loss - the user cancelled the order by hand. ~Rs 34k of funds were blocked for ~11 minutes
(available balance Rs 4,218), which would have made later entries fail the funds check.

## What happened (IST)
- 13:22:06 SONACOMS BULLISH trigger 825.95 touched; tick entry starts. ATM lookup needed a retry ("No CE leg found
  for SONACOMS at strike 0").
- 13:22:07 market BUY placed: SONACOMS 27 OCT 830 CALL x1225, order 322260930176507.
- 13:22:14 `wait_for_order_result` (6 x 1s) returns **PENDING / CONFIRMED** - not TRADED. The engine's "TRADED-only
  fill discipline" treated the entry as failed, unsubscribed the option and released the symbol - **and left the
  order resting at Dhan**. No event was written (only a log line), so the day's event log showed nothing.
- 13:22:17 the Bollinger PAPER book "filled" the same contract at 28.10 (its LTP also took 3 REST attempts): the
  option was thin at that moment.
- ~13:33 user cancels the order in the Dhan app; funds back to Rs 38.5k.

## Why it is dangerous
A resting BUY that fills later becomes a real position nobody manages: no stop, no breakeven, no 15:15 square-off,
and restart reconciliation will not adopt it either (it only adopts contracts whose OPEN is in the strategy's own
trade history). The same code shape exists in the supervisor's hedge order, and in Bollinger's and Swing's entry
paths (both paper-only on the day).

## Fix (traderBoy `6478e3f`, deployed 13:33:58 IST with 3 real CEs open, all re-adopted)
`SuperBollinger/trading_engine.settle_unfilled_order()`: when an entry or hedge order is still open
(TRANSIT/PENDING/PART_TRADED) at the end of the wait -> cancel it, then read the final status once more. A fill
that raced the cancel returns TRADED and continues as a normal fill (SL placed, position tracked). Events:
`ORDER_UNFILLED_CANCELLED`, `ORDER_FILLED_DURING_CANCEL`, `ORDER_STILL_RESTING_AT_BROKER` (manual cancel needed).

## Still open
- Same gap in `Bollinger/trading_engine.py` and Swing's entry path - fix before either goes real again.
- Why a MARKET order stayed PENDING: likely Dhan's market-price protection turning it into a limit on a thin
  contract. Check the order's details in Dhan's order history.
- Separate misses the same afternoon: PNBHOUSING (12:45) and LAURUSLABS (12:46) entries were skipped as
  `ENTRY_SKIPPED_LOW_PREMIUM` with premium = null - the REST LTP call returned nothing three times (the known
  REST-quote weakness), i.e. two real entries lost to a pricing failure, not to a low premium.
- A partial fill that ends CANCELLED is only logged, not adopted (cannot happen at 1 lot).
