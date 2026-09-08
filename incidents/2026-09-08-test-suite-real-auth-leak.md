# Incident: a test file's missing mocks made real Dhan login attempts, breaking the live droplet's auth — 8 Sep 2026

## What happened

While building and testing two new Luxury features (repeat-loss same-day
block, broker-side stop-loss order), a routine full-suite regression run
was followed almost immediately by `GET /funds/buckets` on the live
droplet returning a real `Internal Server Error`. This is the exact same
symptom as the MAHABANK phantom-exit incident's own root cause earlier
the same day (see `incidents/2026-09-08-mahabank-phantom-pe-hedge-exit.md`
for that one) — a local process doing a fresh PIN+TOTP login can
invalidate the live droplet's own session token, since Dhan appears to
allow only one valid token per account/type at a time.

## Root cause

`tests/test_choppy_stocks.py`'s own `install_all_dhan_mocks()` helper was
missing four mocks present in every OTHER test file's own equivalent
helper: `get_option_ltp`, `get_margin_required`, `get_fund_limits`,
`has_open_position_for_underlying`. These four are called by the
proactive funds check and the already-open-at-broker check before every
real entry attempt — unmocked, they fell through to REAL Dhan network
calls, which (using this local machine's own `.env` credentials, with no
cached, still-valid token the way the droplet's own `pin_totp` cache
sometimes has) attempted a genuinely fresh login, complete with the
`_retry` wrapper's own 27-30 second backoff between attempts. This test
alone took over 2 minutes to run because of this — a big red flag in
hindsight that should have been investigated the first time it was
noticed, not attributed to "just a slow test."

## Impact

- The live droplet's own real Dhan session token was invalidated,
  breaking `/funds/buckets` (`Internal Server Error`) until the service
  was restarted to force a fresh re-authentication.
- No real trades were lost or mispriced this time — all packages were
  confirmed flat before the restart, and the restart itself is the
  established, low-risk recovery action for this failure mode.
- Real financial risk was avoided, but the SAME mechanism that caused the
  MAHABANK incident (a phantom real-position management gap while the
  session was broken) could have recurred had a live position been open
  at the time.

## Fix

Added the 4 missing mocks to `test_choppy_stocks.py`'s own
`install_all_dhan_mocks()`, matching every other file's established
pattern. Runtime dropped from 2+ minutes to ~1.4 seconds, and a fresh run
was confirmed to make zero real Dhan calls (grepped the output for
"Attempting authentication"/"Login failed" — neither appeared).

While fixing this, also proactively checked (rather than waiting for
another real incident to reveal it): the SAME new broker-stop-loss
feature that prompted this discovery needed its own two new Dhan wrapper
methods (`place_stop_loss_market_order`, `check_if_order_filled`) mocked
everywhere a real Luxury entry gets exercised in tests. Found and fixed
across 6 more files (`test_cross_strategy_registry.py`,
`test_daily_reentry_cap.py`, `test_fund_allocation.py`,
`test_fund_allocation_integration.py`, `test_luxury_corrective_actions.py`,
`test_luxury_package.py`) by grepping for existing `place_market_order`
mocks and adding the sibling methods alongside each one, rather than
waiting for a second real auth failure to surface the gap.

## Lesson

**A test that takes noticeably longer than its siblings to run is a real
signal, not noise.** This test's own ~2-minute runtime (versus ~1-2
seconds for comparable files) had apparently gone unnoticed for a while -
it should have been the first clue that something was reaching a real
network boundary instead of a mock. More generally: any new Dhan wrapper
method added to support a live feature needs a mock added to EVERY test
file's own `install_all_dhan_mocks()`-style helper that exercises the
code path calling it - checked this time by grepping for the sibling
method that's already mocked everywhere (`place_market_order`) rather
than relying on noticing a slow test or a live auth break after the fact.
