# 2026-09-15: Swing/COPPER - MCX session-hours misdetection placed 4 duplicate real BUY orders

**Symptom**: a Swing structure-break BUY signal for COPPER fired at 20:10
IST - well within MCX's live evening commodity session - and the bot
placed 4 duplicate real BUY orders for the same contract in under a
minute before Dhan's own margin engine started hard-rejecting further
attempts with `DH-906` (insufficient funds). All 4 were independently
confirmed CANCELLED at the broker (Dhan's own margin engine, not this
codebase) before the fix was deployed - no manual broker-side cleanup
was needed, but the duplicate-order mechanism itself was real and would
have kept firing on any persistent failure, not just this specific one.

**Root cause, precisely**: `dhan_client.is_market_open()` checked only
NSE F&O hours (09:15-15:30 IST) regardless of which exchange segment the
order was actually for. MCX runs a materially longer session (09:00 to
23:30 most of the year, later during the Nov-Mar US-daylight-saving
window) - so a genuinely-live MCX order placed at 20:10 IST was wrongly
tagged as an AMO (after-market order) anyway. Swing v2's own strict
TRADED-only fill discipline (the MAHABANK-phantom-exit lesson - see
`designs/structure-break-indicator.md` and NOTES.md bug history)
immediately treats any non-TRADED result, including a legitimately-
queued AMO, as a FAILED entry rather than a pending one. With no
per-symbol retry cooldown anywhere in Swing at the time, the very next
`MONITOR_INTERVAL_SECONDS` (5s) monitor tick re-evaluated the same
still-true structure-break signal and retried the "failed" entry -
placing another real order every ~5 seconds until Dhan's margin engine
started rejecting them outright.

**Fix** (commit `2023e09`), two independent layers:
1. `is_market_open()` now takes `exchange_segment` (default `"NSE_FNO"`,
   every pre-existing caller unchanged) - pass `"MCX_COMM"` to check new
   `MCX_MARKET_OPEN_TIME`/`MCX_MARKET_CLOSE_TIME` (09:00-23:30,
   independently configurable via env) instead of the NSE hours.
   `place_mcx_market_order` now passes `exchange_segment="MCX_COMM"`
   through, so an MCX order is correctly tagged live vs AMO by its own
   session, not NSE's.
2. Swing now tracks a per-symbol entry-retry cooldown
   (`ENTRY_RETRY_COOLDOWN_SECONDS`, default 180s, `Swing/config.py`): any
   non-entered outcome from `enter_position_for_stock` starts the
   cooldown for that symbol, and the watchlist scan skips a symbol still
   in cooldown instead of hot-retrying it every tick. This is a second,
   independent safeguard against the same CLASS of rapid-duplicate-order
   risk even for a future, unrelated failure mode (not just this
   specific MCX-hours bug) - a genuinely transient one-off failure (a
   single rate-limited LTP call, say) just waits out the 180s and tries
   again; only a hot, tick-by-tick retry loop is blocked.

New tests: `tests/test_mcx_market_hours.py` (4 tests),
`tests/test_swing_entry_retry_cooldown.py` (5 tests). Full suite: 233
passed, same pre-existing 17-test baseline failures, zero new
regressions from this change.

**Lesson**: any AMO/live-order-tagging logic that hardcodes NSE hours
will silently mis-tag every other exchange segment traded in this
codebase - MCX was the first, but the same class of bug would recur for
any future segment with its own session calendar. Separately: Swing's
deliberately strict "non-TRADED = failed, not pending" entry discipline
(the right call for the MAHABANK lesson it came from) has a real
downside if nothing else bounds the retry cadence - a correctness fix in
one place (fill discipline) created a latent duplicate-order risk that
only a second, independent fix (the cooldown) actually closed. Worth
remembering when adding a similarly strict "treat ambiguous as failed"
rule anywhere else: pair it with a cooldown/backoff, don't assume the
caller already has one.

**Related**: `reconcile_broker_positions()` was also given the ability to
proactively discover a pre-existing resting broker-side stop-loss order
on restart, applied to Swing the same day (commit `d54f270`) just ahead
of turning `SWING_V2_BROKER_STOP_LOSS_ENABLED` on live for the first
time - see
[[2026-09-15-jswenergy-orphaned-stop-loss-and-icicipruli-stuck-order]]'s
own "Incident 1" fix, which this is the Swing-specific application of.
