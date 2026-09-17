# 2026-09-17: PAGEIND orphaned position, Swing signal-caching bug, COALINDIA double-fill, and SOLARINDS loss

Four real, distinct incidents/investigations on the same live trading day.
The first two share a common lesson (silent failure modes getting cached
or acted on as if they were legitimate results) and both got code fixes
the same day in `traderBoy`; the third was a manual remediation; the
fourth was a loss investigation that concluded no code change was
warranted.

## Incident 1: PAGEIND real position orphaned after a stalled exit order (root cause of tonight's reconciliation work)

**Symptom**: a real PAGEIND PE position (netQty=20) showed up unmanaged by
any strategy, despite Luxury's own trade history believing it had already
closed the position. User's resolution for the immediate situation: cancel
the stray SELL order and leave the position deliberately untracked
(manual intervention), rather than have the bot re-attribute or re-touch
it.

**Root cause, precisely**: a `LIQUIDITY_GUARD_ZERO_VOLUME` exit fired
because the option had gone quiet - but that same illiquidity then also
stalled the market SELL order placed to actually exit it. The SELL sat
`PENDING` (non-terminal, non-AMO) past `wait_for_order_result`'s own poll
budget (6 retries, 1s apart - see its own docstring, which already warned
"callers MUST check `.status`/`.is_queued_amo` rather than assuming the
order filled"). Every package's exit-order-resolution code (Options/
Futures/Luxury/Swing's own `_exit_position`-equivalent) checked for
REJECTED/CANCELLED (back off and retry) and `is_queued_amo` (defer to
sync), but had **no explicit branch for "still non-terminal, non-AMO,
retries exhausted"** - it fell straight through to unconditionally call
`close_position()`, recording a `closed_at`/`exit_price` for a position
that was NOT actually closed at the broker.

That false "closed" record then broke `trade_history.
attribute_open_broker_position` on the next restart: its logic (correctly,
given its inputs) compares `last_open_at > last_closed_at` per strategy
and returns the strategy name only when exactly one strategy still claims
the position as open. A `closed_at` this position never actually earned
made every strategy look like "already closed" - so the function correctly
concluded "no strategy currently owns this," orphaning a real, still-open
position with no owner. The `attribute_open_broker_position` logic itself
was verified correct by reading it in full; the bug was entirely upstream,
in the exit path feeding it bad data.

Notably, Options' own **entry-side** code already had the correct pattern
for this exact class of bug (its own "NOTES.md bug #22" - a slow-filling
market order caused a wrong LTP-guessed entry price): an explicit
`if result.status not in OrderStatus.TERMINAL_STATUSES:` branch that
defers to `_sync_pending_orders()` instead of assuming success. That fix
had simply never been mirrored onto any package's **exit** side, across
four packages that are all near-verbatim copies of each other.

**Fix** (uncommitted as of this write-up; see traderBoy's own git log for
the landing commit): the identical branch added to all four `_exit_
position`-equivalents (Options/Futures/Luxury/Swing, Swing adapted for
LONG/SHORT via its `exit_side` variable instead of a hardcoded `"SELL"`),
placed after the existing `is_queued_amo` check and before the
unconditional `close_position()` call:

```python
if result.status not in OrderStatus.TERMINAL_STATUSES:
    logger.warning(
        "SELL order %s for %s still %s after the poll budget - deferring to "
        "background sync instead of assuming it filled.",
        order_id, symbol, result.status,
    )
    return
```

This relies on each package's own **already-running** periodic pending-
order recheck (`_sync_pending_orders` in Options/Futures/Luxury,
`_sync_pending_exit_orders` in Swing - called every tick from each
package's own `monitor_loop()`) to resolve the deferred case once the
order actually reaches a terminal status. Verified by reading those
functions in full: they already iterate ANY position with a pending exit
order id set (regardless of true AMO-ness), already clear-and-retry on
REJECTED/CANCELLED, and already `close_position()` on genuine TERMINAL -
no changes needed there, only the upstream bug that fed them a false
"already closed" record needed fixing.

Regression tests added: `test_options_broker_stop_loss.py::
test_12_still_pending_non_amo_exit_defers_instead_of_closing`,
`test_futures_broker_stop_loss.py::test_12_...` (identical pattern),
`test_luxury_broker_stop_loss.py::test_12_...` (the file that names the
real incident directly), `test_swing_v2_entry_exit.py::test_13_...` - each
overrides `wait_for_order_result` to return a non-terminal, non-AMO
`OrderResult` (e.g. `PENDING`) for the exit call and asserts the position
stays in `live_positions` with `pending_exit_order_id` still set, and
nothing lands in `closed_positions_today`.

**Lesson**: when the same bug pattern gets fixed on one side of a
symmetric operation (entry vs. exit, open vs. close, buy vs. sell) in one
package, check whether the mirror-image side - and every other package
that copied the original code - has the same latent gap. This codebase's
"near-verbatim copy across 4 packages" architecture means a fix applied
to only one copy leaves the other three carrying the original bug
untouched and undetected until their own real incident.

## Incident 2: Swing signal-caching silently went null for the whole watchlist during a Dhan API instability window

**Symptom**: regime/Supertrend signals for ALL Swing watchlist symbols
went silently `null` at once during a live window of real Dhan API
instability (malformed JSON responses, failed expiry-list/LTP calls,
WebSocket reconnect errors, a stale access token - all observed within
roughly 02:30-03:14 UTC the same night). This matches an earlier, still-
unexplained CRUDEOIL/NATURALGAS "all signals went null" anomaly from a
prior session - now root-caused.

**Root cause, precisely**: `fetch_continuous_intraday` (`Options/
dhan_client.py`, shared by every package) returns `{}` - its own
documented failure mode - on an internal Dhan API failure, WITHOUT
raising. `_fetch_regime_state_once`/`_fetch_supertrend_state_once`
(`Swing/signals.py`) then compute `None` normally from that empty data,
which is **indistinguishable** from "a genuinely too-new symbol with
insufficient warm-up history." The caller (`get_regime_state`/
`get_supertrend_state`) unconditionally caches whatever came back:
`_regime_cache[symbol] = (_now_ist(), state)` - silently overwriting a
previously-good cached value with `None`, with **zero exception raised or
logged**, since nothing in the empty-data path actually raises. The
existing `try/except: logger.exception(...); return cached[1] if cached
else None` fallback in both getters is correct and was already there -
it just never got a chance to run, because nothing threw.

**Fix**: added an explicit check immediately after each `fetch_
continuous_intraday` call, before computing anything from its result:

```python
if not fast_data.get("close"):
    raise RuntimeError(
        f"{symbol}: fetch_continuous_intraday returned no data at all for the "
        f"{config.REGIME_FAST_INTERVAL_MINUTES}-min regime series - treating as a fetch "
        f"failure, not genuinely insufficient history"
    )
```

(mirrored for the slow regime series and for the Supertrend fetch). This
converts a silent "empty data" case into a raised exception that the
already-correct exception handler in `get_regime_state`/`get_supertrend_
state` now actually catches, preserving the last good cached value. A
genuinely-too-new symbol is unaffected: Dhan serves *however many* bars
actually exist for a real, currently-listed instrument, so it never comes
back COMPLETELY empty (`not data.get("close")`) - only a real fetch
failure does. That symbol still correctly falls through to `None` via the
pre-existing, unrelated warm-up-length check further down the function.

Regression test added: `tests/test_swing_v2_signals.py::
test_6_completely_empty_fetch_response_keeps_last_good_cached_value` -
establishes a good cached value, then simulates `fetch_continuous_
intraday` returning `{"status": "success", "data": {}}` (not an
exception) for both regime and Supertrend, and asserts the good value
survives.

**Lesson**: a function whose own docstring documents "returns `{}` on
failure" as a *normal* return path (not an exception) is a landmine for
every caller that treats an empty result as "just no data yet" rather
than "the fetch itself may have failed." Any caching layer sitting
downstream of such a function needs its own explicit check for the
completely-empty case before trusting a fetch result enough to cache it -
the presence of a correct fallback/retry mechanism (like the existing
try/except here) is worthless if the failure never gets a chance to raise
into it.

## Incident 3: COALINDIA doubled to netQty=2700 (manual remediation, not a code bug)

**Symptom**: a real COALINDIA position doubled to `netQty=2700` - an old
AMO order filled at market open on top of a fresh, guard-permitted entry
placed the same morning. Not caused by a duplicate-order bug (the
existing dedup guards were checked and are working as designed for the
case they cover); this was two independently-valid orders for the same
symbol landing in the same session.

**Resolution** (per explicit user instruction: "leave the full 2700 qty
and fix the tracking," not sell down): a one-off remediation script
cancelled the old consolidated SL order, computed a new trigger/limit via
`broker_stop_trigger_and_limit("LONG", 4.525, 2700, MAX_LOSS_PROTECTION_RS,
BROKER_STOP_LOSS_LIMIT_GAP_MULTIPLE)`, and placed a new broker-side SL
covering the full 2700 qty. Swing's own `Position` record was then
corrected to the true 2700 qty via `reconcile_broker_positions()` reading
the real broker net quantity directly on the next restart (verified: this
function already reads `quantity = abs(bp["quantity"])` straight from
Dhan's real `/positions` data, not from any previously-tracked value, so
a restart alone was sufficient to pick up the true amount once the SL was
fixed).

**Lesson**: not every "real position doesn't match expectations" incident
is a code bug - worth explicitly ruling out "two independently correct
decisions colliding" before assuming a guard failed. No code change was
made or needed here beyond the standard restart-driven reconciliation
already in place.

## Investigation: SOLARINDS 29 SEP 18750 PUT large loss - concluded not preventable by current filters

**Question investigated**: why did this specific trade suffer a large
loss - was it foreseeable or preventable with better filters?

**Finding**: option-contract illiquidity, not a directional or filter-
detectable risk. The underlying moved only about ±1% (18840 -> 18780)
during the trade window while the option premium collapsed ~15.5% from
its peak (580 -> 490.05) - a quote/liquidity phenomenon specific to a
deep, expensive 18750 strike on a thinly-traded underlying, not something
visible in the underlying's own price action. Cross-checked two
independent ways: (1) shadow-mode filter data for this trade showed
healthy readings across the board (vol_ratio=2.42, Efficiency Ratio=0.571,
no combo-block condition), and (2) the underlying's own 1-min price-action
data directly, which confirmed the ±1% move claim.

**Conclusion**: none of the current filters - vol_ratio, Efficiency Ratio,
reversal-prevention combo checks - operate on anything other than the
underlying's own price/volume data, so none of them could have caught a
purely option-side liquidity collapse on a deep strike. No code change
made; flagging as an open, currently-unaddressed risk category (option-
side illiquidity independent of underlying movement) rather than a bug -
a future mitigation would need a liquidity signal computed from the
option's OWN order book/quote data, not the underlying's, which nothing
in this codebase currently fetches or checks.
