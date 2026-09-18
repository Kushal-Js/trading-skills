# 2026-09-18: Running a Swing MCX test file standalone triggers a real Dhan login attempt

## Symptom

Running `tests/test_swing_v2_mcx_entry_exit.py` as a standalone script
(`uv run python tests/test_swing_v2_mcx_entry_exit.py`, the "HOW TO RUN"
convention this repo's test files document in their own docstrings) - or
even a single test extracted from it and run alone via `asyncio.run(...)` -
triggers **real, live authentication attempts against Dhan's actual login
endpoint**, using the REAL `DHAN_PIN`/`DHAN_TOTP_SECRET` from `.env`
(loaded via this test file's own `load_dotenv(REPO_ROOT / ".env")` at
import time - the same `.env` the live droplet uses). Confirmed via a real
traceback: `Dhan_Tradehull.Dhan_Tradehull.get_login` /
`Tradehull.__init__` raising `"Login failed. Please retry with valid
credentials."`, with "PIN + TOTP login failed once, retrying with a fresh
TOTP in N seconds" retry cycles (13s and 30s delays observed) burning real
wall-clock time and making real requests against Dhan's servers - two of
these runs hit `"Too many attempts. Please try again after sometime"`
(Dhan's own rate-limit response).

**Confirmed pre-existing, not caused by this session's work**: reproduced
on the clean, already-deployed `9f8390d` baseline (before this session's
own `_get_ltp`/`get_last_historical_close` MCX-fallback work), running
`test_1_copper_options_entry_uses_mcx_routing` completely alone. The test
itself still PASSES despite the real auth attempts in the middle - the
auth failures are somehow non-fatal to the test's own outcome, meaning
they're happening in a path whose own failure gets caught/swallowed
somewhere and doesn't propagate into the assertion.

## Confirmed NOT the cause (ruled out by direct code reading)

- `get_liquid_atm_option`'s own "not authenticated" bypass
  (`self._client is None`) - correctly returns the mocked `get_atm_option`
  result before ever calling `_is_mcx_commodity`. `_client` genuinely
  stays `None` throughout (a failed `Tradehull()` constructor never
  reaches the `self._client = tsl` assignment), so this bypass keeps
  firing correctly - it's why the test's OWN assertions still pass.
- `fund_allocation.has_sufficient_bucket_funds` - both real Dhan calls it
  makes (`get_margin_required`, `get_fund_limits` via
  `get_bucket_available_funds`) are already mocked by this test file's own
  `install_mocks()`, and the whole function fails open on any exception
  regardless.
- `Swing.signals.get_supertrend_state` - already mocked by this file's own
  `_set()` helper.

## NOT yet root-caused

The exact trigger is still unknown - ruled out the obvious candidates but
didn't find the actual one before stopping (production safety was the
priority, not a full root-cause). Worth investigating before anyone next
runs a `tests/test_swing_v2*.py` (or similar Swing MCX) file standalone -
candidates not yet checked: `_record_swing_event`, `resolve_instrument_
side`/`resolved_option_type_for` (unlikely, pure functions), a
module-level side effect in `Swing/signals.py` or `Swing/config.py`
triggered purely by import, or something in `position_store.reserve_
symbol`/`__init__` that isn't obviously broker-related but transitively
touches `dhan_wrapper`.

## Why this didn't show up in any of today's several full pytest-suite runs

`uv run --with pytest --with pytest-asyncio pytest tests/ -q
--asyncio-mode=auto` (the full-suite command used as this session's
deploy gate, run 5+ times today) has never shown this symptom - no
"Attempting authentication"/"PIN + TOTP" noise in any of those runs'
output, and all of them completed in their usual ~13 minutes. Whatever
the trigger is, something about running many test files together in one
pytest process (vs. this one file alone in a fresh process) avoids it -
possibly an earlier-collected test file's own mock/fixture leaves
`dhan_wrapper` in a state that happens to protect this file's tests too,
masking the gap rather than fixing it. This means **the full pytest
suite remains a safe, validated way to test changes** - the risk is
specifically standalone single-file execution of this one file's test
suite via `uv run python tests/test_swing_v2_mcx_entry_exit.py` (its own
documented "HOW TO RUN"), or any hand-extracted single test from it run
outside pytest.

## Confirmed: production was never at risk

The live droplet's own `dhanboy.service` process is completely separate -
its own already-authenticated session, its own IP, its own process.
Checked before, during, and after these test runs: `/health` stayed
`{"status":"ok"}`, all 3 real Swing positions (ANGELONE/COPPER/
NATURALGAS) stayed correctly tracked, and `journalctl` on the droplet
showed zero auth failures or restarts during this window. A local test's
real auth attempt failing with "Unauthorized"/"Too many attempts" did not
propagate to or affect the droplet's own session in any observable way
this time - but see `incidents/2026-09-08-test-suite-real-auth-leak.md`
(and its recurrence) for a REAL case where a local test's auth attempt
DID invalidate a live session. This is a real, standing risk class this
codebase has been burned by before, not a hypothetical one - treat any
new "PIN + TOTP" log noise during local testing as a stop-and-investigate
signal, not background noise.

## Practical takeaway for future sessions

- **Never run `tests/test_swing_v2_mcx_entry_exit.py` (or likely its
  siblings) as a bare standalone script** until this is root-caused and
  fixed - use `uv run --with pytest --with pytest-asyncio pytest
  tests/test_swing_v2_mcx_entry_exit.py -q --asyncio-mode=auto` instead
  (also not fully verified safe in isolation - only proven safe as part
  of the FULL `tests/` suite run so far).
- If a new test needs to exercise something in this file, prefer adding
  it to the full suite's own tracked coverage and validating via the full
  suite run, not a standalone script invocation, until the actual trigger
  here is found and fixed.
- If "Attempting authentication using PIN + TOTP" or "Auto-generated
  TOTP" ever appears in local test output, STOP immediately, check the
  live droplet's health (`/health`, `/positions` across all 4 packages,
  and `journalctl` for auth failures) before doing anything else.
