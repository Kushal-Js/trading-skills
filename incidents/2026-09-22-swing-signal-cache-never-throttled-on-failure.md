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
refresh cadence instead of an unbounded storm.

**Update, same morning, ~08:56-09:00 IST (pre-open)**: watched the fix live
post-deploy - throttle spacing was confirmed clean (~60-65s apart, not
every 5s), but 3 of Swing's 6 watchlist symbols (COPPER, COALINDIA,
NATURALGAS) were still failing their regime-state fetch on literally
every attempt, continuously, for 50+ minutes straight. Added one
diagnostic-only log line to `fetch_continuous_intraday`
(`Options/dhan_client.py`, commit `38de294`) to capture Dhan's actual raw
response instead of just "empty" - deployed immediately (restart
03:29:42 UTC, 0 positions before/after, auth clean, `/health` OK). First
log line on the very next tick answered it definitively:

```
DH-904 Rate_Limit: "Too many requests on server from single user
breaching rate limits. Try throttling API calls."
```

**Root cause, confirmed**: a genuine account-wide Dhan REST rate limit,
not anything specific to these 3 symbols individually - `DH-904` is
per-account, not per-instrument. With Options/Luxury/Futures' breakout
scanners (up to 10 symbols/cycle each), the UniverseDispatcher, the
universe_bucket sync (19 CE symbols), and Swing's own regime/Supertrend
polling for 6 symbols all sharing the same account-wide call budget, the
aggregate REST volume across all 4 packages is now tight enough that
Swing's fetches are sometimes the ones that get throttled out - COPPER/
COALINDIA/NATURALGAS just happened to be whichever calls landed in the
"too many requests" window each cycle, not a property of those symbols.
My earlier MCX-contract-age hypothesis was wrong - glad this got settled
by evidence instead of shipped as a guess.

**Action taken same morning**: per user request, reduced Swing's
watchlist from 6 symbols to 2 (COPPER, NATURALGAS only - dropped
ADANIPORTS, ANGELONE, COALINDIA, ASHOKLEY) via `POST /swing/watchlist/
replace`, effective immediately (no restart needed - the watchlist file
is re-read every monitor tick). This directly cuts Swing's own
contribution to the shared REST budget by two-thirds as an immediate,
zero-risk mitigation ahead of market open, buying time to look at the
budget question properly.

**Still open, deliberately deferred to after market close (22 Sep)**:
whether/how to reduce the account-wide call volume more structurally -
options include pacing Swing's regime/Supertrend fetches further apart,
reducing `INTRADAY_CONTINUOUS_LOOKBACK_DAYS`/`REGIME_EMA_LOOKBACK_DAYS`
call frequency, or reviewing whether the dispatcher/universe_bucket
scan cadence (10 symbols/60s) is itself worth trimming. Not touched
today - no code/config change to the shared budget itself, just the
watchlist-size mitigation above.
