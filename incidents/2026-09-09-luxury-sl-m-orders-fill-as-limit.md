# Incident: Luxury's real broker-side STOP_LOSS_MARKET orders were filling instantly as LIMIT sells, 9 Sep 2026

## What happened

The day after Luxury's own real broker-side stop-loss feature went live
(see NOTES.md entry #98, added 8 Sep 2026 - a real SELL STOP-LOSS MARKET
order placed at Dhan immediately after every entry, meant to fire
independently of our own poll/tick cadence), several real entries that
morning (COALINDIA, GVT&D, PAYTM) were closed within seconds of fill,
well before any real price move could plausibly have crossed the
intended stop trigger.

The user, watching live, diagnosed it exactly right without seeing any
code: "orders are getting closed too early without max loss limit ever
being hit, while placing the stop loss order, the price should be
calculated based on current price and the price at which the trade is
brought. I also doubt that a real Stop Loss order is being placed. I
think it is a normal Sell order right after a trade is placed."

## Root cause: Dhan converted/executed our real STOP_LOSS_MARKET orders as LIMIT sells

A direct `get_order_by_id` check (via the droplet's own cached, already-
authenticated session - never from a local machine, see "Safe live-
testing discipline" below) on that morning's actual "stop-loss" orders
showed every one of them recorded as:

```json
{"orderType": "LIMIT", "price": <our computed trigger>, "triggerPrice": 0.0, "orderStatus": "TRADED"}
```

Not `STOP_LOSS_MARKET`, not a resting conditional order - an ordinary,
immediately marketable LIMIT sell that filled the moment it reached the
exchange.

### Client-side code was ruled out first, by reading actual installed source

Before assuming this was a Dhan platform quirk, the full call chain was
traced by reading the real source of the exact package versions
installed on the droplet (not assumed from memory or from PyPI docs):

- `Dhan_Tradehull.Tradehull.order_placement()`'s own mapping:
  `{'STOPMARKET': self.Dhan.SLM}` - correct, and passed straight through
  to `self.Dhan.place_order(order_type=order_type, ...)` unmodified.
- `dhanhq`'s own constants: `LIMIT='LIMIT'`, `MARKET='MARKET'`,
  `SL='STOP_LOSS'`, `SLM='STOP_LOSS_MARKET'` - all distinct strings, no
  collision.
- `dhanhq.dhanhq.place_order()`'s payload builder:
  `"orderType": order_type.upper()`, POSTed verbatim via
  `DhanHTTP.post()` with zero client-side remapping or fallback logic
  anywhere in the chain.

This conclusively rules out a client-library bug. The outgoing request
genuinely carried `orderType: "STOP_LOSS_MARKET"` with the correct
trigger price. Whatever converts it happens server-side, at Dhan.

### A controlled live test isolated the exact behavior (real money, ~Rs.600-750 spent, with the user's explicit go-ahead each step)

Official Dhan v2 docs were thin on SL-M specifics for F&O options (a
recurring theme - see other notes in this repo about Dhan's docs being
generally thin), so the only way to get a real answer was a live,
controlled test:

1. Confirmed zero live positions across every strategy first.
2. Bought 1 lot COALINDIA ATM PE (a real position - needed to test a
   genuine protective SELL; naked-selling an option never held is a
   different order class with different margin semantics that wouldn't
   isolate the same bug).
3. Placed a real SELL "STOPMARKET" order with the trigger deliberately
   far below the current LTP (so it could not realistically fire during
   the test) - Variant A with `price=0` (production's actual code at
   the time), Variant B with `price=trigger_price` (the leading
   hypothesis for a fix, since Dhan's own stop-loss support docs
   describe SELL SL mechanics as "Market Price > Trigger Price > Order
   Price," implying a real reference price might matter).

**Result: both variants came back `orderType: "LIMIT"` and both TRADED
INSTANTLY at the prevailing market price (~Rs.7.5-7.6), nowhere near the
~Rs.3.6-3.8 trigger.** This ruled out "it's about what value we pass for
`price`" conclusively - the behavior is identical either way. It also
meant a further test bypassing `Tradehull.order_placement()` to call
`dhanhq.place_order()` directly was skipped once the source-reading
above showed it would build an IDENTICAL payload (Tradehull is a thin,
unmodified pass-through here) - that would have cost another real trade
for zero new information.

### A real mistake in the test script itself, caught and fixed immediately

The test script fired Variant A's SELL, then Variant B's SELL, without
checking in between whether the position was still open. Variant A's
SELL correctly closed the long; Variant B's SELL then had nothing left
to close and went out as an unintended naked short (confirmed via
Dhan's own `get_positions`: `COALINDIA-Sep2026-435-PE, netQty=-1350`).
Caught immediately by checking real broker positions right after the
test, and covered with a real BUY before any further exposure. Net
realized cost of the whole round trip (buy 7.6 → sell 7.5 → naked-short
7.55 → cover-buy 7.85, ×1350 qty each leg) was approximately -Rs.472,
plus normal brokerage/STT/exchange charges - roughly Rs.600-750 total.
This was an own-script bug (a missing position-state guard between two
order calls), not a finding about Dhan's API, and is called out here so
the same mistake isn't repeated in a future controlled test: **always
verify current position/order state between two live order calls in a
test script, never assume the first call's outcome.**

## Fix

No client-side "fix" exists for this - the request was already correct
and Dhan's own execution behavior for SL-M SELL orders on NSE_FNO
options (`productType=MARGIN`) does not match documented conditional-
stop semantics on this account/segment. Decision (the user's own call):
**abandon broker-side SL-M for options entirely**, rather than keep
spending real money chasing a fix that may not exist from the client
side at all.

- `LUXURY_BROKER_STOP_LOSS_ENABLED=false` was already deployed as an
  emergency mitigation the moment the bug was suspected, before this
  investigation even started.
- `Luxury/config.py`'s `BROKER_STOP_LOSS_ENABLED` code DEFAULT was
  flipped from `"true"` to `"false"` (not just `.env`) so this confirmed
  -broken feature can't silently re-enable itself from a fresh
  environment or a rebuilt `.env`.
- The feature's code and its 7 tests were kept, not deleted (this
  repo's own convention - see SAFETY.md-adjacent practice in traderBoy's
  own NOTES.md). While fixing the default, found that
  `tests/test_luxury_broker_stop_loss.py`'s tests 3/4/5 never explicitly
  pinned `BROKER_STOP_LOSS_ENABLED = True` the way tests 1/2/6/7 already
  did - they were silently relying on the (now-flipped) ambient default
  to reach the code paths they claim to test. Same "ambient default
  quietly defeats a test's actual intent" pattern as traderBoy NOTES.md
  entries #92/#94. Fixed by pinning all three explicitly.

Luxury's real protection stack going forward: the pre-existing
poll/tick-driven `MAX_LOSS_HIT` check (event-driven per real WebSocket
tick via `on_price_tick`, plus a 2-second REST-poll fallback via
`monitor_loop`), `LIQUIDITY_GUARD_ENABLED` (exits early on a thinly-
traded option going quiet - the actual precursor pattern behind the
CHOLAFIN-style overshoot-via-price-gap case broker SL-M was originally
built to backstop, see `2026-09-03-cholafin-overshoot-and-luxury-
corrective-actions.md`), and `LOSS_REPEAT_BLOCK_ENABLED`.

## Open question / not yet true for other instrument types

This finding is specifically about **options** SL-M SELL orders under
`productType=MARGIN`. Swing's own uncommitted broker-stop-loss work
(built 9 Sep 2026, targeting FUTURES legs via the same
`place_stop_loss_market_order` primitive, never deployed) has **not**
been shown broken by this - futures is a different instrument type and
may behave differently on Dhan's side. If that Swing feature is ever
revived, it needs its own independent live verification before being
trusted - don't assume it's safe just because the primitive is shared,
and don't assume it's broken just because options SL-M was.

## Lessons

1. **A user's plain-language diagnosis from watching live behavior can
   out-perform documentation research.** The user correctly identified
   "it's probably just a normal Sell order" purely from observing
   instant post-entry exits, before any code was read. Trust and verify
   specific, mechanism-level user hypotheses with hard evidence rather
   than defaulting to "let me check the docs first."
2. **Rule out the client library by reading its actual installed
   source, not by trusting past assumptions or third-party docs about
   it.** This investigation didn't stop at "the mapping dict looks
   right" - it read `place_order()`'s full body on the exact installed
   version to confirm zero remapping exists anywhere in the chain,
   which is what made "bypass Tradehull" provably redundant before
   spending more real money on it.
3. **A live, controlled, real-money test is sometimes the only way to
   get a definitive answer when a broker's own docs are thin** - but
   scope it tightly (smallest liquid instrument, trigger deliberately
   unreachable, immediate close-out) and get explicit confirmation
   before every real trade it requires, especially once the real
   capital number is known (not just "a test," but "~Rs.10,000 notional
   for this specific contract").
4. **Even a carefully-scoped live test needs its own safety checks
   between steps.** A script placing multiple real orders in sequence
   must verify position/order state after each one, not just at the
   end - the naked-short mistake here happened precisely because the
   script assumed the first SELL's outcome instead of checking it.
5. Same "ambient config default silently defeats a test's actual
   intent" pattern keeps recurring (see #92/#94 and the test-suite
   real-auth-leak incident) - whenever a test's whole point is to
   exercise behavior gated by a flag, pin that flag explicitly in the
   test itself, never rely on whatever the code/env default currently
   happens to be.
