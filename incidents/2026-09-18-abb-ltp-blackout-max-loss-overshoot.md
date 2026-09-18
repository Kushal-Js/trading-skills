# 2026-09-18: ABB 29 SEP 7200 CALL - MAX_LOSS_HIT overshoot from a name-based LTP quote blackout on a genuinely liquid contract

## Symptom

Two Futures CE trades on ABB today, both real losses:

| | Entry | Exit | Reason | PnL |
|---|---|---|---|---|
| Trade 1 | Rs 147.35 @ 12:31 IST | Rs 125.45 @ 12:38 IST | `MAX_LOSS_HIT` | **-Rs 2,737.50** (-14.86%) |
| Trade 2 | Rs 134.80 @ 13:04 IST | Rs 120.15 @ 13:30 IST | `SUPERTREND_EXIT` | **-Rs 1,831.25** (-10.87%) |

Trade 1's configured cap at that time of day (after `RISK_THRESHOLD_CUTOFF_TIME=11:30`) was **Rs 2,100**. The realized loss overshot it by ~30%.

## Investigation

Initial hypothesis (wrong): the option was genuinely illiquid, and the
existing `LIQUIDITY_ENTRY_GATE_ENABLED` gate (added 17 Sep 2026 for the
SOLARINDS incident, checks `LIQUIDITY_GUARD_ZERO_VOLUME_BARS` consecutive
zero-volume 1-min bars) missed it. **Disproved by direct evidence**: a
read-only fetch of the option's own 1-min candles for today showed
genuine, substantial volume every single minute through the incident
window (1750-4750 contracts/minute, 06:55-07:10 UTC) - this option was
actively trading. The liquidity-guard hypothesis does not apply here.

**Real root cause**: `journalctl` showed **144 separate `"No LTP returned
for ABB 29 SEP 7200 CALL"`** warnings from `dhan_client.get_option_ltp`
across the two trades' combined holding time - both `_get_ltp`'s WS cache
AND its REST fallback (`get_option_ltp`, which calls Tradehull's
**name-based** `get_ltp_data(names=[trading_symbol])`) were failing
persistently, **despite the contract genuinely trading**. This is a
different failure class from illiquidity: a name-based live-quote lookup
can apparently go dark on a contract that a **security_id-based** read
(the same `fetch_continuous_intraday` call `refresh_liquidity_signal`
and `get_last_historical_close` already use) has no trouble with at all.

With the poll loop only getting a valid price sporadically (once every
10-40+ seconds instead of every `MONITOR_INTERVAL_SECONDS=2`), `MAX_LOSS_
HIT` could only evaluate against whatever price happened to be available
at each successful read - by the time one finally succeeded, the loss had
already run from the Rs 2,100 cap to Rs 2,737.50.

**The broker-side stop-loss (SL-L) backstop also failed on both trades**,
independently of the LTP problem:
- Trade 1: SL-L (trigger 130.55 / limit 129.70) never filled - still
  resting, unfilled, 7 minutes later when the bot force-exited via a
  fresh market SELL. Classic SL-Limit gap-through: price fell through the
  tight Rs 0.85 trigger-to-limit buffer before any buyer would take it at
  129.70.
- Trade 2: SL-L (trigger 118.00 / limit 117.15) was **cancelled by the
  broker/exchange without ever firing** (`ended as CANCELLED without
  firing`), logged as a repeating warning on every poll tick from
  07:51:57 UTC onward, with no self-healing/re-arm logic - the position
  fell back to relying solely on the poll loop, which was itself impaired
  by the same LTP problem.

**Why `LTP_STALE_FORCED_EXIT` (the existing "feed has gone dark" safety
net) didn't catch it either**: it only escalates after `_get_ltp` fails
*continuously* for `LTP_STALE_FORCE_EXIT_MINUTES` (5). On trade 2, 9
individual calls fully exhausted all 3 internal retries and raised - but
each raise was followed by just enough of a stray successful read to
reset `_ltp_failure_since` back to zero before 5 minutes accumulated.
The feed was flaky, not fully dead - exactly the gap between what this
threshold catches and what actually happened.

## Fix (same day)

`_get_ltp` (Options/Futures/Luxury `trading_engine.py`, identical code in
each) now falls back to `dhan_wrapper.get_last_historical_close` -
security_id-based, already proven reliable during a live-quote outage
(see the 2026-09-10 ICICIPRULI incident this function was originally
built for) - whenever `get_option_ltp` fails, before raising up to
`_handle_ltp_staleness`. The fallback price is deliberately **not**
written back via `note_rest_ltp` (one-tick reading only, so the very next
poll always retries the primary path fresh rather than treating a
stale-by-design historical close as an authoritative live price). This
keeps `MAX_LOSS_HIT`/every other exit check evaluating on *some* real
price every 2 seconds instead of going dark for minutes at a stretch.
`get_last_historical_close`'s own docstring was updated to reflect this
broader usage (previously scoped to forced-exit logging only).

Verified against the exact incident shape: a new test
(`tests/test_ltp_historical_close_fallback.py::test_3_real_position_with_dead_live_ltp_still_hits_max_loss_via_fallback`)
builds a real Futures position, makes `get_option_ltp` always raise (as
it did ~144 times that day), sets the historical-close fallback to the
real 125.45 exit price, and confirms `_check_one_position` still fires a
correct `MAX_LOSS_HIT` at that price instead of going blind.

**Not fixed today** (flagged, not addressed - scope was the LTP/cap
overshoot specifically): the broker-side SL-L failure mode itself (no
self-healing re-arm after a broker-side cancellation-without-firing).
Worth a follow-up if this contract-liquidity/quote-quirk combination
recurs.

## Separate, related change same day: `LOSS_REPEAT_BLOCK_COUNT` 2 -> 1

User request following this investigation: block re-entry into a symbol
for the rest of the day after its **very first** same-day loss-exit, not
the second (`loss_count >= LOSS_REPEAT_BLOCK_COUNT` in
`trading_engine._process_one_entry`, all 3 packages). Applied as both the
code-level default and the deployed `.env` value
(`LOSS_REPEAT_BLOCK_COUNT`/`FUTURES_LOSS_REPEAT_BLOCK_COUNT`/
`LUXURY_LOSS_REPEAT_BLOCK_COUNT`, all previously `2`). New regression
tests added per package (`test_7b`/`test_14b` in each
`test_*_corrective_actions.py`) confirm a single real `MAX_LOSS_HIT`
loss now blocks the very next same-day entry attempt for that symbol,
while a different symbol is unaffected. The existing COUNT=2 tests are
unaffected - they explicitly set `config.LOSS_REPEAT_BLOCK_COUNT = 2`
for their own duration rather than relying on the ambient default.
