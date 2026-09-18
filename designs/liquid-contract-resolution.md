# Liquid contract resolution gate

Status: **Built and deployed** (18 Sep 2026)

## What it is

A single shared function, `dhan_client.get_liquid_atm_option(underlying_symbol,
option_type)`, that all 4 live-trading packages (Options, Futures, Luxury,
Swing) now call instead of `get_atm_option` directly before placing any real
order. It resolves the natural ATM strike, then requires it to pass two
checks before trading it:

1. **Current-session liquidity** — the existing zero-volume-bars signal
   (`refresh_liquidity_signal`/`get_cached_illiquid`,
   `config.LIQUIDITY_GUARD_ZERO_VOLUME_BARS`), unchanged.
2. **Prior-sessions' real trading activity** (new) — `get_daily_volume_sum`
   sums the real daily volume (via Dhan's `/charts/historical` daily-bar
   endpoint, a genuinely separate call from the 1-min/5-min intraday series
   the rest of the codebase uses) over the last `LIQUID_CONTRACT_LOOKBACK_
   DAYS` (7) calendar days, and requires at least `LIQUID_CONTRACT_MIN_
   PRIOR_SESSION_VOLUME` (500, a disclosed starting judgment call, not
   backtested) total.

If the ATM strike fails either check, the function walks outward to the
nearest strikes on both sides (up to `LIQUID_CONTRACT_MAX_STRIKE_SEARCH`, 5,
each way) for the SAME underlying/expiry/option_type, and returns the first
candidate that passes both — a genuine liquid substitute, not just a skip.
Returns `None` only if nothing in the search window qualifies; every caller
treats that as "skip this entry."

## Why it exists

Real incident: ATHERENERG 29 SEP 1540 PUT's broker-side stop-loss was
REJECTED with `EXCH:17181: Contract not traded. Market order not allowed`
because the natural ATM strike had never printed a single trade before the
position was already opened via AMO. With the broker-side backstop gone,
the position fell back to an unprotected poll loop that couldn't keep up
with a fast opening move, landing a -Rs 4,893.75 loss against a Rs 3,500
cap. See `incidents/` for the day's fuller writeup of that specific loss.

## MCX support (Copper)

Initially shipped NSE-only, deliberately: `refresh_liquidity_signal` and
the new `get_daily_volume_sum` had never been exercised against an MCX
exchange_segment/instrument_type, and guessing wrong risked silently
blocking every real Copper entry. Extended to MCX the same day on explicit
user request, after empirically confirming (not guessing) the real segment
codes by reading the actual instrument master:

- `SEM_EXM_EXCH_ID == "MCX"`, `SEM_EXCH_INSTRUMENT_TYPE == "OPTFUT"` for
  every real MCX option row (commodity options are options-on-futures, a
  genuinely different instrument type from NSE's OPTSTK/OPTIDX).
- `exchange_segment = "MCX_COMM"` — the same constant Swing's own order
  placement already uses (`dhanhq.MCX = 'MCX_COMM'`).
- The instrument master also carries `SM_SYMBOL_NAME` (a clean underlying-
  name column) for MCX rows — Tradehull's own `ATM_Strike_Selection` adds
  an extra `SM_SYMBOL_NAME == Underlying` filter only on its MCX branch,
  so `_nearby_option_candidates` mirrors that same extra check for MCX
  rather than inventing a new one.
- `refresh_liquidity_signal` gained optional `expected_exchange`/
  `exchange_segment`/`instrument_type` parameters, all defaulting to NSE's
  existing values — every pre-existing NSE call site (the exit-time
  `LIQUIDITY_GUARD_ENABLED` checks in Options/Futures/Luxury) is untouched.

## A real regression this caught before deploy

`_is_mcx_commodity(underlying_symbol)` touches `self.instruments()` ->
`self.client`, and `self.client` is a lazy property that calls
`self.authenticate()` for real if `self._client` is still `None`. The first
version of `get_liquid_atm_option` called `_is_mcx_commodity` to compute
`is_mcx` **before** checking `self._client is None` (the standard "not
authenticated" bypass every other gate in this codebase, e.g.
`check_option_liquidity_sync`, already uses to stay test-safe). Every unit
test exercising a real entry path never calls `authenticate()` — so this
ordering mistake made 26 unrelated tests across 4 different test files
attempt a genuine Dhan login and fail with `RuntimeError: Dhan login failed
(mode=pin_totp)`, caught by this session's own full-suite run before
anything shipped.

Fix: check `config.LIQUID_CONTRACT_GATE_ENABLED`/`self._client is None`
**immediately** after resolving the ATM pick (itself safe because
`get_atm_option` is always mocked directly in tests), and only call
`_is_mcx_commodity` after that bypass has already returned. `tests/
test_liquid_contract_resolution.py::test_9_not_authenticated_bypass` was
deliberately rewritten to use the REAL, unmocked `_is_mcx_commodity` (not a
stub) specifically so this exact ordering mistake can never regress
silently again — a mocked version would have hidden the very bug that
happened.

**Lesson for future work here**: any new shared gate function added to
`dhan_client.py` that touches `self.instruments()`/`self.client`
transitively (not just directly) needs its "not authenticated" bypass
checked *first*, before any other internal call — and the regression test
for that bypass should use the real, unmocked internal call, not a stub,
or the test can't actually catch this class of ordering bug.

## What's NOT covered

- No backtest was run to validate `LIQUID_CONTRACT_MIN_PRIOR_SESSION_
  VOLUME=500` against real volume distributions - it's a starting,
  disclosed judgment call. Revisit if it turns out too strict (skipping
  entries that should have gone through) or too loose (still letting a
  thin contract through) in practice.
- Swapping to a nearby strike changes the position's actual delta/risk
  profile slightly versus the "true" ATM - an accepted, disclosed
  tradeoff (the user explicitly asked for a substitute contract, not just
  a skip).
- The added latency (one extra REST call per candidate checked, in the
  worst case up to `2 * LIQUID_CONTRACT_MAX_STRIKE_SEARCH + 1` candidates)
  is accepted given the entry pipeline already tolerates multi-second/
  multi-minute latencies elsewhere (ranking, funds checks, RSI/loss-repeat
  guards all run before this point too).
