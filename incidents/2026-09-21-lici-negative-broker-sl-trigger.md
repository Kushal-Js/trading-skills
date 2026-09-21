# Incident: LICI broker-side SL-L order rejected with negative trigger price

## What happened

09:19:39 IST (03:49:39 UTC), Options package. A real PE entry on LICI
(Chartink scan "Sell Range Breakout"), fill=0.70, qty=1400. The bot
attempted to place the usual broker-side SELL stop-loss LIMIT order
immediately after fill, and it failed:

```
ERROR | trading_engine | LICI: could not place the broker-side stop-loss
order for LICI 29 SEP 380 PUT (trigger would have been -1.80, limit
-1.93) - proceeding without it, the existing poll/tick-driven MAX_LOSS_HIT
check still protects this position exactly as before
RuntimeError: order_placement returned no order id for STOPLIMIT SELL
LICI 29 SEP 380 PUT - check Tradehull's console/log output for the
underlying error.
```

Only this one occurrence today - confirmed via a full-day journalctl
grep, not a repeating pattern.

## Root cause

`_place_broker_stop_loss_if_enabled` (identical in all three packages)
computes:

```python
max_loss_cap = current_max_loss_per_trade_rs(option_type)
trigger_price = fill_price - (max_loss_cap / quantity)
```

For LICI: `max_loss_cap` = Rs3,500 (Options PE, before the 11:30 cutoff),
`quantity` = 1,400. `3500 / 1400 = 2.50`. `0.70 - 2.50 = -1.80` - a
negative price, which Dhan correctly rejects (no order can have a
negative price).

**Why this happens**: the formula assumes the rupee MAX_LOSS cap is
always smaller than the position's own notional value (`fill_price *
quantity`). That's usually true, but for a cheap-premium, large-lot-size
name like LICI, the position's ENTIRE notional value
(`0.70 * 1400 = Rs980`) was already below the Rs3,500 cap - meaning even
a 100% loss (the option going to zero) can never reach Rs3,500. The
rupee cap was never going to bind for this trade in the first place, so
there's nothing for a rupee-based broker order to enforce - the formula
just doesn't handle that boundary case and produces a nonsense negative
price instead of recognizing it.

## Real risk impact: low, not zero

The position was NOT left unprotected in any meaningful sense - the
regular percentage-based `STOP_LOSS_PCT` hard stop (computed separately,
`entry_price * (1 - STOP_LOSS_PCT)` = `0.70 * 0.84` = `~0.59`, a sane
positive price) is enforced by the normal poll/tick-driven
`_check_one_position` loop regardless of whether the broker-side SL-L
order exists. What WAS missing: the broker-side redundant layer that
survives even if the bot process itself goes down - and, worth being
precise about, the RUPEE-based `MAX_LOSS_HIT` check specifically could
never have fired for this position either way (its own internal
threshold price is the same negative number), but that's not a gap in
practice since the position's worst possible loss (Rs980, full wipeout)
was always below the Rs3,500 cap regardless of whether that check could
fire.

## Fix

`Options/Luxury/Futures/trading_engine.py`'s `_place_broker_stop_loss_
if_enabled`, identically: if the computed `trigger_price <= 0`, skip
broker SL placement cleanly with an INFO log (not an ERROR - this isn't
a failure, it's an expected edge case for cheap/large-lot positions)
explaining that the position's own notional value is already below the
MAX_LOSS cap, so the cap can never bind and the percentage-based
STOP_LOSS_PCT hard stop is what actually protects the trade. Deployed
same day, commit `e70804d`, restart 16:30 UTC.

## What's still open

This only guards the NEGATIVE-price case. A trigger price that's
positive but still below the instrument's own minimum tick size (or
below Dhan's own minimum order price) could still theoretically fail the
same way with a different error - not observed today, not specifically
guarded against. Worth watching for.
