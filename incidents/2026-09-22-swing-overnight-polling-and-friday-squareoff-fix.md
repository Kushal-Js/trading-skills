# 2026-09-22/23: Swing polled Dhan 24/7 with no market-hours gate - fixed, plus a new weekly Friday square-off

## What was found

A post-market-close system check (triggered by the user, following the
day's [[2026-09-22-luxury-loosened-breakout-params-cost-analysis]]
rollback work) found `Swing/trading_engine.py`'s `monitor_loop` - by
design, runs forever with no EOD/weekend gate, since Swing carries
positions across days - was ALSO running its *entry*-evaluation path
(`_evaluate_entry_signal` -> `get_regime_state`/`get_supertrend_state`)
completely unconditionally, every `MONITOR_INTERVAL_SECONDS` (5s), 24/7.

Confirmed live: **39 DH-904 rate-limit hits in under an hour**, well
after midnight, entirely from this - zero other scanner was active
(breakout scanners/UniverseDispatcher all correctly stop at their own
`market_end=15:35`). Zero possible benefit either way: no new candle can
form outside real trading hours, and no order could fill even if a
signal somehow fired.

Isolated a second, sharper data point in the same pass: with almost no
other traffic overnight, **ASHOKLEY failed 39/40 times vs. NATURALGAS's
1**, both through the identical code path/account - ruling out "shared
contention from other processes" as ASHOKLEY's specific cause (the
messier multi-process picture from that morning's own
[[2026-09-22-mcx-vs-nse-fetch-failures-are-shared-rate-limit-not-segment-specific]]
couldn't rule this out). Still not root-caused - flagged as an open
question, sharper than before.

## Fix 1: per-symbol market-hours gate

`Swing/signals.py` gained `_symbol_market_open(symbol)`: weekday check
(Sat/Sun always closed) + the RIGHT session per symbol - `MCX_COMM` hours
(09:00-23:30, materially longer) for anything in `config.MCX_SYMBOLS`,
NSE hours (09:15-15:30) otherwise - reusing the already-existing
`dhan_wrapper.is_market_open(exchange_segment=...)`, not a new
implementation. A blanket NSE-hours cutoff would have wrongly blocked a
live MCX symbol for hours it's genuinely still open (the exact mirror of
the incident `is_market_open`'s own MCX-awareness was built for, 15 Sep
2026); a blanket MCX-hours cutoff would have left the same overnight-
polling problem mostly unfixed for NSE symbols.

Wired into two places:
- `Swing/trading_engine.py`'s `_monitor_tick` - the watchlist entry-
  evaluation loop only, gated per-symbol. **Exit-checking on an already-
  open position is deliberately untouched** - out of scope for this fix,
  not requested, and changes real risk-management behavior for a held
  position, which needs its own explicit go-ahead if ever revisited.
- `Swing/signals.py`'s `structure_break_refresh_loop` (COPPER's dedicated
  background refresh, independent of `monitor_loop`) - same class of
  always-on overnight polling, same fix applied for consistency.

## Fix 2: weekly Friday square-off (new behavior, explicit user request)

Added `config.FRIDAY_SQUARE_OFF_ENABLED`/`_TIME` (default `true`/`15:25`)
and `trading_engine._is_friday_square_off_time()`. Once Friday 15:25 IST
passes, `_monitor_tick` calls the already-existing (previously only
manually-triggered) `_square_off_all("FRIDAY_SQUARE_OFF")` and returns
early - no ordinary exit-check, no new-entry evaluation - for the rest of
Friday. Deliberately earlier than MCX's own much-later 23:30 close, since
the point is avoiding the weekend gap entirely, not squeezing out MCX's
last few evening hours. Swing still carries positions across ordinary
weekdays by design; this only stops it carrying across a WEEKEND.

## Verification

Full existing Swing test suite (13 files, all standalone `python
tests/test_X.py` scripts - NOT pytest-collectible, see each file's own
"HOW TO RUN" docstring) re-run before and after: two pre-existing,
unrelated failures confirmed via `git stash` (an LTP-staleness broker-SL-
cancel assertion in both `test_swing_v2_entry_exit.py` and
`test_swing_v2_mcx_entry_exit.py` - flaky/timing-sensitive, reproduces
identically with this fix's changes stashed out, later passed again on a
clean re-run with no code change - not investigated further, out of
scope). One genuine regression found and fixed: `test_swing_entry_retry_
cooldown.py::test_5` called `_monitor_tick()` directly against a real NSE
symbol without controlling wall-clock time - broke because the new gate
correctly skips NSE entry-evaluation outside 09:15-15:30, and the test
happened to run well after midnight. Fixed by stubbing `_symbol_market_
open` open for that test (it's testing the cooldown gate in isolation,
not this one).

New dedicated test file: `tests/test_swing_overnight_gate_and_friday_
squareoff.py` (6 tests) - per-symbol NSE-vs-MCX-vs-weekend gating,
`_monitor_tick` skipping/resuming entry-evaluation across the gate, the
Friday cutoff's exact boundary (15:24 false / 15:25 true / stays true
rest of Friday / Thursday-same-time false), and `_monitor_tick` calling
`_square_off_all` + skipping ordinary flow once active. All 3 modules
that independently read `datetime.now(IST)` (`Swing.signals`,
`Options.dhan_client`'s `is_market_open`, `Swing.trading_engine`) needed
patching together for a single frozen "now" to take effect end-to-end -
same reason `tests/test_mcx_market_hours.py` patches `Options.dhan_
client.datetime` on its own.

## Deployed

Committed `cc70363`, pushed, pulled on droplet, dry-run import check
passed, restarted after market close with zero open positions (safest
possible timing - no live-position risk at all). Verified post-restart:
both `monitor_loop` and `structure_break_refresh_loop` started cleanly,
**zero DH-904/error hits in the new process** vs. the old process's
steady ~1-2 minute failure cadence beforehand.

## Not done this session (disclosed)

- ASHOKLEY-vs-NATURALGAS asymmetry still not root-caused - sharper
  evidence now (near-isolated overnight window), still open.
- Exit-checking's own Supertrend-reversal fetch (`_evaluate_exit_signal`)
  is NOT market-hours-gated - only matters when a position is actually
  held overnight/through a weekend, which the new Friday square-off
  should make rare going forward, but not impossible on an ordinary
  weeknight. Not requested, not touched.
- Did not investigate whether Dhan enforces genuinely separate per-
  segment or per-symbol rate-limit buckets - still assumed account-wide
  per `DH-904`'s own wording, same caveat as the morning's own writeup.
