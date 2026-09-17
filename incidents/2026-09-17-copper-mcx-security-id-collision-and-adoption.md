# 2026-09-17: COPPER manual-trade adoption into Swing, a real MCX `security_id` collision, and the live incident it caused

## Background: adopting a manually-placed real position into Swing

The user manually placed a real COPPER 23 SEP 1400 CALL (MCX, 1 lot,
entry 9.0, ₹2500/point multiplier) directly at the broker - untracked by
any strategy. They asked to have it "considered as SWING placed trade and
keep track of it by monitoring also."

**Approach**: reused the existing, already-proven `attribute_open_broker_
position` + `reconcile_broker_positions()` mechanism rather than
inventing new live-mutation logic - wrote ONE `position_opened` JSONL
record (strategy="Swing", the position's real fields, real order_id) via
`trade_history.append_jsonl`, verified `attribute_open_broker_position`
now resolved to "Swing", then restarted. Confirmed live via `/swing/
positions`: `reconciled: true`, correct entry_price/pnl_multiplier/
target_price/hard_stop_loss.

The user then asked to place a real stop-loss for it, matching Swing's
standard sizing formula (`broker_stop_trigger_and_limit`). Computed
trigger=7.2/limit=7.11 from live config (`MAX_LOSS_PROTECTION_RS=4500`,
gap multiple 0.05), placed via `place_mcx_stop_loss_limit_order`, got a
real order_id (`23826091716609`), confirmed genuinely resting via
`refresh_order_status` (`PENDING`, `remark='EXCH:Order Added'`).

## The bug: `get_pending_order_id` resolves MCX symbols to the wrong `security_id`

Checking whether reconciliation had picked up this resting SL order
(`Position.stop_loss_order_id`) found it still `null`. Diagnosed
precisely with direct scripts against the live account, not guessed:

- `_instrument_meta("COPPER 23 SEP 1400 CALL")` (no exchange hint) →
  `security_id='123831'` - **wrong**, an unrelated NSE instrument.
- `_instrument_meta("COPPER 23 SEP 1400 CALL", expected_exchange="MCX")`
  → `security_id='574836'` - matches the order's own real `securityId`
  field exactly.
- Dhan's own `get_order_list()` echoes this order's `tradingSymbol` as
  `"COPPER-23Sep2026-1400-CE"` (hyphenated) - never equal to this
  codebase's own canonical `"COPPER 23 SEP 1400 CALL"` (space-separated).

So BOTH of `get_pending_order_id`'s match paths (string equality, and
security_id equality) were failing for this contract - the string
never matches MCX's differently-formatted echo, and the security_id
lookup itself resolved to the wrong instrument because the call site
never passed an exchange hint. This is the exact same "cross-exchange
instrument-master collision" class already fixed for ATM resolution (12
Sep 2026, `c7ffaa3`) and independently rediscovered for the *stale-order
string-match* case two days earlier (see
[[2026-09-15-jswenergy-orphaned-stop-loss-and-icicipruli-stuck-order]]) -
but `get_pending_order_id`'s own `_instrument_meta` call was never given
an exchange hint, since the function predates Copper/MCX support
entirely.

**This wasn't cosmetic** - it affects reconciliation's stop-loss
discovery AND every future Copper exit's stale-order-cancel step, for
every Copper trade with a broker-side stop, not just this one position.

**Fix** (`traderBoy` commit `da6feb6`): added `expected_exchange:
Optional[str] = None` to `get_pending_order_id`, threaded through to
`_instrument_meta`. Defaults to `None` (today's exact behavior) since
Options/Futures/Luxury only ever call this for NSE symbols - zero risk
there. Swing's 3 call sites (entry dup-guard, `_exit_position`'s stale-
order-cancel, `reconcile_broker_positions`' SL discovery) now pass
`"MCX"`/`"NSE"` from the position's own `exchange_segment`.

## The live incident this bug was actively causing, found while verifying the fix

While re-verifying the fix (deploying it to the droplet and about to
restart), a routine `/swing/positions` check turned up something much
more urgent than the original tracking gap: the position showed
`exit_failure_count: 11` and a `next_exit_retry_at` a few minutes out -
Swing's own automated exit had been trying and failing to close COPPER
for **~40 minutes straight**, retrying every ~5 minutes.

**Root cause, read directly from `_exit_position`'s own logic**: before
placing a fresh exit order, it tries to cancel any stale/resting order
for the same contract+side - first via `get_pending_order_id` (broken,
per above), then falling back to `position.stop_loss_order_id` (also
`null`, for the same underlying reason). With both paths empty, the
cancel step never ran, so `_exit_position` placed a brand-new SELL
market order **on top of** the still-resting SL-L order
(`23826091716609`). Dhan's RMS saw two live sells against a 1-lot
holding and rejected each one as if opening a fresh naked short:
`"You have insufficient funds. Please add Rs.259660.57 to trade."` - 11
consecutive rejections, retrying every ~5 min, confirmed via
`journalctl` (first attempt 11:32:22, still retrying at 12:06:12).

The exit signal itself was `PROFIT_PROTECTION_HIT` (price had drifted up
toward target) - not a losing position being mismanaged, but a real,
valid exit that the bot could not execute because of this same bug.

**Resolution**: deployed the already-committed fix, restarted
`dhanboy.service` (explicit user confirmation obtained first, per
standing live-trading-restart practice). On the very next tick after
restart: reconciliation correctly populated `stop_loss_order_id:
"23826091716609"`; the stuck `PROFIT_PROTECTION_HIT` exit retried
immediately, this time correctly found and cancelled the resting SL
order via the fixed scan, and placed a clean exit - SELL 1 COPPER
**TRADED** at 12:13:07, fill price 10.14 (entry 9.0, ~₹2,850 profit on
the lot). Zero manual intervention needed once the fix was live; the
bot's own retry loop resolved itself on the first correct tick.

## Lessons

1. **A currently-resting broker-side stop-loss order is itself a hazard
   to the exit path that's supposed to protect the same position**, if
   the exit path's own stale-order discovery is broken. Placing the SL
   order was necessary and correct, but it's what turned a previously-
   silent tracking bug into an active, repeatedly-failing real order
   rejection loop - the SL order gave the bug something to collide with.
   When adding a broker-side stop to a position, immediately verify (as
   this session should have, and now will as standing practice) that the
   position's OWN eventual exit path can actually discover and cancel
   that same order - not just that the SL order itself is resting
   correctly.
2. **Cross-exchange `security_id` collisions keep recurring wherever a
   NEW MCX-aware call site is added without an exchange hint** - this is
   now the third distinct place this exact class of bug has been found
   in `traderBoy` (ATM resolution 12 Sep, JSWENERGY's string-match gap 15
   Sep, and this function 17 Sep). Any future function that calls
   `_instrument_meta`/resolves a `trading_symbol` and might ever be
   called for an MCX contract should default-audit whether it passes an
   `expected_exchange` hint, rather than waiting for another live
   incident to find the next omission.
3. **A routine verification step (re-checking a just-deployed fix against
   real data) surfaced a live, actively-failing production incident that
   wasn't otherwise being alerted on** - `exit_failure_count` climbing
   and `next_exit_retry_at` are visible in `/swing/positions` today but
   nothing pages on them. Worth considering a lightweight alert (or at
   minimum, checking this field as a matter of course) whenever
   inspecting a live position for an unrelated reason.
