# Incident: the same test-mock gap recurred a third time — real Dhan login attempts from local test runs, 11 Sep 2026

## What happened

While running the local regression suite after deploying today's config
changes (EMA-cross exit turned on for Options/Luxury, plus the session's
new RSI-loss-reentry block and Nifty gap-down CE delay features),
`tests/test_options_corrective_actions.py` printed real Dhan login
activity mid-run: `Attempting authentication using PIN + TOTP` /
`Auto-generated TOTP` / `Login failed: Access token not found in
response: {'message': 'Unauthorized Request', 'status': 'error'}` /
`Login failed. Please retry with valid credentials.` — a genuine,
uncached PIN+TOTP login attempt against the real Dhan account, made from
a local test process on the Mac.

This is the exact same failure class as
[[2026-09-08-test-suite-real-auth-leak]] and the MAHABANK phantom-exit
incident the same day — a local process authenticating fresh can
invalidate the droplet's own live session (Dhan appears to allow only
one valid token per account/type at a time).

## Why it happened again despite the 8 Sep fix

The 8 Sep incident's fix was **reactive and manual**: patch the specific
files caught missing a mock, plus proactively grep for the one sibling
method pattern known at the time. It did not add any **structural**
guard - so every NEW Dhan-network-touching method added since (this
session alone added three: `refresh_ema_cross_signal`/
`get_cached_ema_cross_candle_start`, `is_rsi_loss_reentry_blocked` + 3
getters, `should_delay_ce_entry`) had to be manually remembered and
back-filled into every test file's own `install_all_dhan_mocks()` by
hand, with no automated check to catch a miss. Options'/Luxury's own
`ENABLE_EMA_CROSS_EXIT` had been off in every local test run until today
(only Futures had it on, since 10 Sep), so `_capture_supertrend_entry_
candle`'s always-present call to `dhan_wrapper.refresh_ema_cross_signal`
(gated only on "is EITHER Supertrend or EMA-cross on", not per-flag) had
never actually executed un-mocked in a local run before - the exact same
"never triggered until the feature actually turns on" shape as the 8 Sep
incident.

Investigating this one also turned up that **Futures' own test file had
carried this exact gap since 10 Sep** (when its EMA-cross flag first
went live) without ever surfacing - it just happened not to reach the
vulnerable code path in a way that printed a visible error before now.

## Impact

- **No actual damage**: the droplet's own live session was confirmed
  unaffected afterward (token validity unchanged, service active,
  positions flat) - the local login attempts *failed* both times
  (`Unauthorized Request`), so no competing session was ever actually
  established. This is different from the 8 Sep incident, where the
  local attempt apparently *succeeded* and broke the droplet's session.
- Purely by luck of how Dhan's TOTP/session handling responded this
  time, not because of anything protective in the code.

## Fix (11 Sep 2026)

Audited and patched **14 test files'** `install_all_dhan_mocks()` (or
equivalent) to mock all three of this session's new methods, matching
the existing `refresh_supertrend_signal`/`get_cached_supertrend_candle_
start` no-op pattern:
- `refresh_ema_cross_signal`, `get_cached_ema_cross_candle_start`
- `is_rsi_loss_reentry_blocked`, `get_cached_rsi`, `get_cached_prev_rsi`,
  `rsi_loss_reentry_reason`
- `should_delay_ce_entry` (only in the files that call the real webhook
  handler directly - `_process_one_entry`-only tests never reach it)

Files: `test_choppy_stocks.py`, `test_cross_strategy_registry.py`,
`test_daily_reentry_cap.py`, `test_deep_integration.py`,
`test_fund_allocation_integration.py`, `test_futures_broker_stop_loss.py`,
`test_futures_corrective_actions.py`, `test_luxury_allowed_trading_time.py`,
`test_luxury_broker_stop_loss.py`, `test_luxury_corrective_actions.py`,
`test_luxury_package.py`, `test_nifty_gap_down_ce_delay.py`,
`test_options_broker_stop_loss.py`, `test_options_corrective_actions.py`.
Every file re-run individually afterward, watched live for "Attempting
authentication"/"Login failed" - all clean (traderBoy commit `705f799`).

## Second recurrence, same day: `test_fund_allocation.py` missed by the sweep above

While testing an unrelated change (turning on `BROKER_STOP_LOSS_ENABLED`/
`FUTURES_BROKER_STOP_LOSS_ENABLED` and lowering `LOSS_REPEAT_BLOCK_COUNT`,
later the same day), a full local suite run surfaced the exact same real
PIN+TOTP login activity again - this time from `test_fund_allocation.py`.
This file has its **own** separate `install_dhan_mocks()` (distinct from
`test_fund_allocation_integration.py`, which the sweep above did patch)
and was missed entirely - a plain naming collision (`fund_allocation` vs
`fund_allocation_integration`) let it slip through the audit. Patched
with the same 6 mocks (`refresh_ema_cross_signal`/
`get_cached_ema_cross_candle_start`, `is_rsi_loss_reentry_blocked`/
`get_cached_rsi`/`get_cached_prev_rsi`/`rsi_loss_reentry_reason`; no
`should_delay_ce_entry` needed - this file never reaches the real webhook
handler). Re-run clean, no auth activity, all 7 checks still pass
(traderBoy commit `3b09310`). No damage - droplet token/session/positions
confirmed unaffected both times.

This is now a **third** occurrence of the identical failure class in one
week (8 Sep, 11 Sep, 11 Sep again), and the second time in one day that a
"complete" manual sweep still missed a file. That's strong evidence the
manual-audit approach itself doesn't scale - even a careful grep-and-fix
pass, run twice in one day, missed a file both times. Reinforces the
Lesson below: this needs to stop depending on a human (or Claude)
remembering to check every file by name.

## Lesson (the 8 Sep lesson didn't hold - this one needs to be structural)

A per-incident manual patch has now failed to prevent a recurrence
twice. The durable fix isn't "remember to grep for the new method next
time" - it's an **automated regression guard**, the same pattern this
repo already uses elsewhere (e.g. `tests/test_continuous_intraday.py`'s
`test_4`, which inspects source to catch a reintroduced today-only
fetch). Proposed for a future session: a test that inventories every
`dhan_wrapper` method touched by `_process_one_entry`/the webhook
handlers in each package's `trading_engine.py`/`*_main.py` (via
`inspect.getsource` or an explicit allowlist) and asserts every test
file's own `install_all_dhan_mocks()` mocks all of them - so adding a
new network-touching signal without updating every test's mocks fails
the SUITE itself, loudly, instead of silently attempting a real Dhan
login the next time that flag happens to flip on in a local `.env`.
