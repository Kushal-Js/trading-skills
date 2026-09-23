# 2026-09-15: JSWENERGY orphaned broker-side stop-loss + ICICIPRULI stuck entry order

Two real, distinct incidents on the same live trading day, both found via
user-reported symptoms and both root-caused and fixed the same day in
`traderBoy`.

## Incident 1: JSWENERGY's real stop-loss order left resting with no position behind it

**Symptom** (user-reported, screenshot of the broker's own Pending Orders
list): a SELL trigger-limit order for `JSWENERGY 29 SEP 520 PUT` (qty
1075, trigger 6.45) was still showing as pending at the broker, even
though the underlying PE position had already closed ~40 minutes earlier
(`LIQUIDITY_GUARD_ZERO_VOLUME`, real market SELL, TRADED, +₹1,075).
Real-money risk: if price ever fell back to the trigger, this stray order
could fire and open an unintended naked short.

**Root cause, precisely**: the position had been reconciled from the
broker at a restart earlier that session (a real, still-open position at
that time). Reconciliation (`reconcile_broker_positions()`) has no way to
discover a pre-existing resting stop-loss order just from `/positions`
data, so the reconciled `Position.stop_loss_order_id` came back empty.
When the position later closed via a *different* exit path
(`_exit_position`'s own `LIQUIDITY_GUARD_ZERO_VOLUME` branch), the only
remaining safety net - `get_pending_order_id`'s live broker-order-list
scan - matched on the `tradingSymbol` **string** alone, and missed the
genuinely-resting order. Two independently plausible reasons this scan
can miss a real match: (a) this codebase's own `trading_symbol` values
are always `SEM_CUSTOM_SYMBOL` format, but Dhan's order-list API can echo
`tradingSymbol` in a different format (`SEM_TRADING_SYMBOL`), so a
string-equality check can fail even for the identical real contract; (b)
the scan's old status allow-list explicitly included `"TRIGGER_PENDING"`
- a string that isn't actually one of DhanHQ's own documented order
statuses (see `OrderStatus`'s own enum, which only has
TRANSIT/PENDING/REJECTED/CANCELLED/PART_TRADED/TRADED/EXPIRED) - so a
real intermediate status Dhan actually uses but that guess-list didn't
happen to include would also be silently skipped.

**Fix** (commit `5e73d1b` in traderBoy): two parts, in the ONE shared
`Options/dhan_client.py` (Futures/Luxury re-export this same singleton,
so the fix applies to all three automatically):
1. `get_pending_order_id` now ALSO resolves `trading_symbol` to its real
   `security_id` (via the already-existing `_instrument_meta`, which
   already matches on EITHER symbol format) and matches broker orders on
   EITHER the tradingSymbol string OR that security_id.
2. The status check switched from an allow-list to
   `OrderStatus.TERMINAL_STATUSES` as a deny-list - any status that isn't
   definitively terminal is now treated as "still there."

Additionally, `reconcile_broker_positions()` in Options/Futures/Luxury
now proactively looks up and populates `stop_loss_order_id` for any
reconciled position (when `BROKER_STOP_LOSS_ENABLED`) - defense in depth,
so the fallback path always has a real value even if the scan somehow
misses again. Verified live on its very first real test after deploy:
the very next reconciled position (`INDIGO 29 SEP 4900 PUT`) had its
resting stop order correctly discovered and tracked.

**Lesson**: this is the third time this exact "match by the trading-
symbol string" pattern has caused a real bug this session (see also the
COPPER cross-exchange instrument-master collision, and the
`_instrument_meta`/`_instrument_meta_by_security_id` split that already
existed for this reason) - `security_id` is Dhan's only reliable cross-
endpoint identifier; any NEW code that matches broker data by symbol
string should be treated as a latent bug, not a convenience.

**Update, same day (commit `d54f270`)**: the `reconcile_broker_positions()`
fix above was applied to Options/Futures/Luxury only at first. Swing has
its own separate `reconcile_broker_positions()` with the identical gap
(no resting-stop-loss discovery on restart), closed the same day before
`SWING_V2_BROKER_STOP_LOSS_ENABLED` was turned on live for the first
time - Swing had a real open COPPER position at the time that would
otherwise have been exposed to this exact class of bug on its very next
restart. One difference from the other three packages: Swing is
side-aware (a SHORT position's resting stop order is a BUY, not a SELL),
so it uses `exit_transaction_type(side)` rather than a hardcoded
`"SELL"`.

## Incident 2: ICICIPRULI's real BUY order stuck PENDING for 10+ minutes

**Symptom** (user-reported): a real BUY market order for `ICICIPRULI 29
SEP 465 PUT` (qty 925) placed immediately on a real Chartink alert never
reached TRADED or REJECTED - `_sync_pending_orders` correctly re-checked
it every monitor tick (as designed for AMO orders), but simply re-logged
the same PENDING status forever, with no timeout or remediation. The
order was still stuck when an unrelated restart (deploying the Incident 1
fix) wiped the in-memory tracking for it entirely - since it had never
become a filled position, reconciliation had nothing to recover either.
User manually cancelled it at the broker.

**Root cause**: `_sync_pending_orders` had no notion of "this has been
stuck too long, do something" for a plain (non-AMO) order placed during
live market hours - only AMO orders are *supposed* to sit non-terminal
for a long time (until the next session), and the existing logic didn't
distinguish the two cases.

**Fix** (commit `a15160c`): `_sync_pending_orders` (Options/Futures/
Luxury, each a near-verbatim copy needing its own change) now times out
a plain market-hours BUY order once it's been non-terminal for
`config.STALE_ENTRY_ORDER_TIMEOUT_SECONDS` (default 300s):
1. Cancel it.
2. Re-verify broker truth (`get_broker_net_quantity`) in case it filled
   in the exact instant the cancel raced against it - promote to a real
   Position using the broker's real quantity/LTP if so.
3. Otherwise, exactly ONE retry: place a fresh market order for the
   identical contract/quantity (a market order always fills at whatever
   the CURRENT price is, so re-submitting IS the "adjust to current
   price" retry - there's no separate limit price to change on a market
   order).
4. If that retry ALSO times out, abandon the entry (release the
   reservation) rather than retrying forever - capped via a new
   `OrderRecord.retry_count` field.

**Related, distinct gap - NOT yet fixed** (lower priority after the fix
above, since most stuck orders now self-resolve within 5-10 minutes
without needing a restart to intervene): a restart that happens to land
*within* that stale-order window still loses all tracking of a genuinely
stuck entry order the same way it did for ICICIPRULI - `orders_today` is
in-memory only, and an order with no filled position yet has nothing for
`reconcile_broker_positions()` to recover. A full fix would need startup
reconciliation to also scan the broker's own pending-order list (not
just filled positions) - not built, since the exposure window is now
narrow (restart AND a stuck order both landing in the same &lt;5 minute
span) and the risk is bounded (worst case: a real order sits unmanaged
until the next restart or until it resolves on its own at the broker).

## Test-isolation bug found + fixed while adding regression coverage

While writing `tests/test_get_pending_order_id_security_id_match.py` for
Incident 1's fix, discovered it failed only when run in the same pytest
session as `test_futures_broker_stop_loss.py` (bisected via running the
two files together). Root cause: `test_futures_broker_stop_loss.py`'s
own `test_6/test_8/test_9/test_10` (and the identical copies in
`test_options_broker_stop_loss.py`/`test_luxury_broker_stop_loss.py` -
the same bug had been copy-pasted into all three) each did
`real_get_pending = odc.dhan_wrapper.get_pending_order_id` **after**
already calling `install_all_dhan_mocks()` - but that helper itself
mocks `get_pending_order_id` as part of its own setup, so `real_get_
pending` was actually capturing the MOCK, not the true original. Each
test's own `finally: odc.dhan_wrapper.get_pending_order_id = real_get_
pending` then permanently overwrote the real method with that stale mock
for every test collected afterward in the same session - the exact same
CLASS of bug as the `Swing.signals` cross-test-file leak found earlier
this session (2026-09-14): a monkeypatch captured/restored in the wrong
order relative to when it was actually applied. Fixed by moving the
`real_*` capture lines to before `install_all_dhan_mocks()` in all 12
affected spots (4 tests x 3 files).

**Lesson (reinforcing the earlier one)**: whenever a test captures "the
real function" to restore later, capture it as the FIRST thing in the
test, before any mocking helper runs - never assume a helper you're
calling hasn't already touched the thing you're about to save.
