# Incident: Swing regime/Supertrend fetch failures never throttled, ran unthrottled all day 21 Sep

## What happened

A routine pre-market review (22 Sep, ~07:15 IST, before market open) of the
previous day's full logs found `swing_signals` "could not fetch regime
state" / "could not fetch Supertrend state - keeping last cached value"
logged **22,198 times** during 21 Sep's real market hours (03:45-10:00 UTC
/ 09:15-15:30 IST, a 6h15m window) - roughly 59 times a minute, essentially
continuous - versus only 52 times across the same length of non-market
overnight hours. All for the same small, fixed set of symbols: Swing's own
watchlist (COPPER, COALINDIA, NATURALGAS, ASHOKLEY, ANGELONE).

## Relationship to [[local-backtest-dhan-session-collision]]

That incident (same day, 21 Sep) already identified and fixed the likely
**trigger**: several local backtest/diagnostic scripts re-authenticating
via `pin_totp` during the day each briefly invalidated the live bot's own
Dhan session, and separately, bulk local historical-data pulls contended
for the same per-account REST rate-limit budget as the live bot's own
calls (`DH-904 Rate_Limit`). Both are real and were fixed same-day
(`c61e82e` blocks non-systemd `pin_totp` auth; heavy pulls deferred to
after close).

**What that incident's writeup didn't yet explain**: why the resulting
`swing_signals` fetch failures kept firing at a sustained ~1/second rate
for hours, well past any single collision event or bulk-pull window,
instead of a handful of transient blips that self-healed within the
intended 60s/15s cache refresh window. This incident is that missing
piece - a separate, genuine code bug that turned each transient failure
(from whatever originally triggered it - a collision, a `DH-904`, or
Dhan's already-documented "intermittent bare generic failure envelope on
back-to-back calls", see `_retry`'s own docstring in `dhan_client.py`)
into a **self-sustaining storm** that never throttled down on its own.

## Root cause

`Swing/signals.py`'s `get_regime_state` and `get_supertrend_state` are
both meant to be cached and throttled (`REGIME_REFRESH_SECONDS=60`,
`SUPERTREND_REFRESH_SECONDS=15`) so that even though `_monitor_tick` runs
every `MONITOR_INTERVAL_SECONDS=5`, the actual Dhan REST call only happens
once per refresh window per symbol. Both functions stamped the cache
tuple `(timestamp, value)` **only on a successful fetch**:

```python
try:
    state = await loop.run_in_executor(None, _fetch_regime_state_once, symbol)
except Exception:
    logger.exception(...)
    return cached[1] if cached else None   # <-- cache timestamp NOT updated
_regime_cache[symbol] = (_now_ist(), state)
return state
```

On failure, the "is this cache fresh enough" check at the top of the next
call (`(_now_ist() - cached[0]).total_seconds() < REFRESH_SECONDS`) reads
the SAME stale timestamp as before - which is already older than the
refresh window (that's *why* the fetch was attempted in the first place) -
so it never evaluates true again. Every subsequent 5-second monitor tick
therefore re-attempts the fetch immediately instead of waiting out the
intended window. Per symbol this is up to 4 real REST calls per tick
(regime fast+slow, Supertrend 5min+15min) x 6 watchlist symbols x 12
ticks/minute = up to ~288 calls/minute once any symbol starts failing -
easily enough to keep a real Dhan-side rate limit engaged indefinitely,
independent of whether the original trigger (a token collision, a bulk
pull) was still happening.

## Real risk impact

Fail-open by design - `_check_one_position`/`_evaluate_entry_signal` fall
back to the last good cached `RegimeState`/`SupertrendState` rather than
crashing or reading `None` as a false signal, so no incorrect order was
placed as a direct result. The real impact is **staleness**: for however
long a given symbol was stuck in the failure loop, Swing's regime/
Supertrend reads for that symbol were whatever was last successfully
computed, potentially hours old, not the live 5-min/15-min state a
trend-following strategy is supposed to be reading. Swing carries 85% of
fund allocation ([[project-fund-allocation-system]]), so this is the
highest-impact latency/staleness bug found in this review. Compounded
Dhan-side impact: hammering the same REST endpoint at up to ~288 calls/min
all day is very plausibly what kept `DH-904`-style responses coming long
after the actual bulk-pull contention from the other incident had ended.

## Fix

Stamp the cache with the current outcome - the recovered stale value (or
`None`) on failure, a fresh value on success - **unconditionally**, so a
failing symbol is retried at the intended cadence (60s/15s) instead of on
every 5s tick:

```python
try:
    state = await loop.run_in_executor(None, _fetch_regime_state_once, symbol)
except Exception:
    logger.exception(...)
    state = cached[1] if cached else None
_regime_cache[symbol] = (_now_ist(), state)   # stamped either way
return state
```

Identical fix applied to `get_supertrend_state`. Verified in isolation
(mocked fetch always raising): 2 immediate calls produced only 1 real
fetch attempt (correctly throttled); a 3rd call after the refresh window
elapsed correctly retried. Deployed commit `783925f`, restart 01:55:45 UTC
22 Sep (0 live positions before/after, confirmed via `/positions` on all
4 packages), `/health` OK post-restart.

## What's still open

This fixes the amplification, not necessarily every possible trigger -
`fetch_continuous_intraday` can still occasionally return empty data from
a genuine transient Dhan-side issue (that's the whole reason `_retry`
exists). That's expected and now correctly bounded to the intended
refresh cadence instead of an unbounded storm. Worth watching this
morning (22 Sep market open) for whether the same 5 Swing symbols show
any `could not fetch` lines at all, and if so, confirming they're spaced
~60s/~15s apart rather than every 5s.
