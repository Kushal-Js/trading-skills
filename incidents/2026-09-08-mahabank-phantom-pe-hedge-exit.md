# Incident: MAHABANK phantom PE_HEDGE exit — real position left open and unmonitored for ~2.5 hours, 8 Sep 2026

## What happened

User asked "check status of recent alert" mid-morning, which led to
routine status checks, then "why is Mahabank not exited yet?" once the
underlying's own Supertrend reversal fired. The bot's own bookkeeping
(`GET /swing/basket-hedge-positions`) showed MAHABANK's PE_HEDGE had
already exited cleanly at **04:55:08 UTC (10:25 IST)** via
`PE_SUPERTREND_REVERSAL_EXIT`, flat at 2.18 (same as entry, zero net
P&L). The user then said **"why I can see MAHA bank put still open
then?"** — they were looking at the real Dhan app, which showed the
position still fully open. A direct, read-only query against Dhan's own
real `/positions` confirmed the user was right: `MAHABANK 29 SEP 85 PUT,
quantity=6500, avg_price=2.18` — completely unchanged from before the
"exit."

## Root cause: the exit SELL order never actually filled, but the bot's own success check didn't require a fill

`journalctl` for the exact moment:

```
04:55:02 MAHABANK: PE hedge Supertrend reversal HIT - 5-min close 85.27 crossed above Supertrend 84.35
04:55:02 Placing SELL order: MAHABANK 29 SEP 85 PUT x6500 (product=MARGIN)
04:55:08 Order 222260908224307 still not in a terminal status after 6 retries (last status=PENDING)
04:55:08 BasketHedge PE_HEDGE CLOSED (back to watching): MAHABANK reason=PE_SUPERTREND_REVERSAL_EXIT exit=2.18
```

A direct `get_order_by_id` check (hours later, while investigating)
showed the order was still sitting exactly where it started:

```python
{'orderStatus': 'PENDING', 'orderType': 'LIMIT', 'price': 2.04,
 'filledQty': 0, 'remainingQuantity': 6500, 'averageTradedPrice': 0.0}
```

Two things stacked to cause this:

1. **Dhan/Tradehull silently converted our "MARKET" order into a
   protected LIMIT order** (price 2.04) rather than a true market fill -
   our own code (`Options/dhan_client.py:place_market_order`) explicitly
   passes `order_type="MARKET"` to Tradehull's `order_placement()`, but
   the broker's own order record shows `orderType: 'LIMIT'`. This
   appears to be Dhan/Tradehull's own "market protection" behavior for
   F&O options (a raw market order on a single-stock option can get a
   terrible fill on a wide spread) - not something our code chose. The
   underlying kept moving after the order was placed; the PE's own LTP
   drifted to 1.87, below the 2.04 limit sell price, so nobody would
   trade at 2.04 and the order sat genuinely unfilled.

2. **The real bug, entirely ours**: `Swing/trading_engine.py:_place_leg`'s
   own success check was `ok = result.status not in
   OrderStatus.REJECTED_STATUSES and result.status !=
   OrderStatus.CANCELLED` - this only ever excluded REJECTED/CANCELLED.
   Everything else - PENDING, TRANSIT, PART_TRADED, a stuck order that
   never resolved within `wait_for_order_result`'s own 6-retry/1s-delay
   budget - silently counted as `ok=True`, a successful fill. The bot
   then genuinely believed the SELL succeeded, recorded the PE_HEDGE
   position CLOSED, released `MAX_LIVE_BASKETS` capacity, and moved on -
   while the REAL position sat fully open at the broker, completely
   unmonitored (no more loss-cap/profit-lock/reversal checks would ever
   run for it again, since the bot's own in-memory state showed nothing
   live).

**This exact same `ok` pattern (`status in REJECTED_STATUSES or status
== CANCELLED` as the ONLY failure check) exists identically in
`Options/trading_engine.py`, `Futures/trading_engine.py`, and
`Luxury/trading_engine.py`** - flagged to the user as a systemic,
codebase-wide exposure. Not yet fixed in those three packages as of this
writeup; Swing's own `_place_leg` was fixed first since it's what this
incident actually happened in.

## Impact

- Real, live, ~2.5 hours (04:55 to ~07:32 UTC when discovered) of a
  completely unmonitored open MAHABANK PUT position (qty 6500) - no exit
  condition of any kind would have fired for it again on its own.
- Unrealized loss at time of discovery: ≈ Rs 2,015 (LTP 1.87 vs entry
  2.18), though this number is somewhat academic since the position
  itself was never actually being watched.
- No NEW basket was blocked by MAHABANK's own (phantom) capacity release
  in this window, since no fresh alert happened to compete for the slot
  during those 2.5 hours - but this was luck, not a safeguard. If an
  alert HAD landed, a second real basket could have opened while the
  first (unmonitored) MAHABANK exposure was still live, silently
  exceeding the user's own intended `MAX_LIVE_BASKETS=1` real risk
  budget.

## Recovery (manual, same session)

Diagnosed and fixed live, with the user's explicit go-ahead:

1. Cancelled the stale PENDING limit order (`222260908224307`) -
   confirmed `CANCELLED` at the broker.
2. Placed a fresh real market SELL for the full qty 6500 - filled
   immediately (`TRADED`, avg fill **1.85**).
3. Confirmed the broker now shows zero open FNO positions.

**Real economics, corrected**: entry 2.18 → real exit 1.85 → a genuine
**Rs 2,145 loss** on this leg (`(2.18 - 1.85) * 6500`) - NOT the 2.18
flat/zero-P&L the original phantom "exit" had recorded. The bot's own
`history/*_real_trades.log`/`swing_events.log` for this leg's exit still
carry the WRONG (2.18) numbers, since these are append-only logs and
weren't rewritten - anyone pulling historical P&L for MAHABANK on 8 Sep
2026 needs to know the true exit was 1.85, not what the log says.

## Fix (Swing only so far)

`Swing/trading_engine.py:_place_leg` now requires the ACTUAL terminal
`TRADED` status for `ok=True` - a stuck PENDING order, TRANSIT,
PART_TRADED (a partial fill Swing has no mechanism to track separately -
conservatively treated as not-done), and a genuinely queued AMO (Swing
has no promotion path for one, unlike Options/Futures/Luxury's own
`_sync_pending_orders`) are all now `ok=False`, so every existing caller's
own "leg left open, will retry on the next tick" handling applies
uniformly instead of a false success. New regression suite
(`tests/test_swing_place_leg_fill_confirmation.py`) proves this both at
the unit level and end-to-end through the real `_exit_pe_hedge_to_
watching` call site (a stuck-PENDING exit now correctly leaves the
position OPEN and capacity RESERVED, nothing recorded as closed).

## Still open (flagged, not yet fixed)

The identical `ok` bug pattern in `Options/`, `Futures/`, and `Luxury/`'s
own `trading_engine.py` files - every one of them uses `if result.status
in OrderStatus.REJECTED_STATUSES or result.status ==
OrderStatus.CANCELLED:` as their own only failure check, meaning ANY
order that gets converted to a protected limit order and then drifts
away from fillable (the exact mechanism here) could produce the same
phantom-success outcome in any of those three packages too - a position
silently recorded as opened/closed with a bogus (likely zero) fill_price
while the real broker state diverges. This needs the same fix, package
by package, checked against each package's own AMO-handling code path
(unlike Swing, Options/Futures/Luxury already have a proper
`is_queued_amo` -> `_sync_pending_orders` promotion path, so their own
fix needs to distinguish "genuinely still resolving, handled by that
path" from "stuck and should be treated as failed" more carefully than
Swing's own blunter fix could get away with).

## Lesson

**A "market" order isn't guaranteed to actually fill just because the
broker didn't reject it.** `wait_for_order_result`'s own docstring
already said this explicitly ("Always returns whatever the last-seen
status was; callers MUST check `.status`... rather than assuming the
order filled just because this returned") - but that warning wasn't
followed at every call site that determines trading-logic success/
failure. A "not-yet-terminal after N retries" order needs its own
explicit, distinct handling (retry later, alert, or at minimum refuse to
update internal state) - it is a definitively different case from both
"filled" and "rejected," and treating it as either one is wrong in a way
that has real financial consequences. Found this time by a user directly
comparing the bot's own dashboard against the real Dhan app and noticing
they disagreed - worth periodically doing that kind of cross-check
manually, since the bot's own internal state is not self-verifying.
