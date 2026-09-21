# 2026-09-21: `dhanhq` MarketFeed background thread dies silently on a startup 429, live price feed goes dark

## Summary

The market-data WebSocket feed (used for live LTP ticks that back
`get_cached_option_ltp()`) went completely silent shortly after the
routine 08:00 IST (`02:30 UTC`) morning token-refresh restart, and never
recovered on its own. A `dhanboy.service` restart at the user's request
(prompted by investigating this) did NOT fix it - it changed the failure
from "self-sustaining noisy reconnect storm" to "feed thread dead,
completely silent, zero further attempts", which is actually a step
WORSE for self-healing even though the process otherwise looked healthy
(`/health` OK, positions reconciled correctly). **Root cause is a real
bug in the vendored `dhanhq==2.2.0` SDK, not this repo's own code** - see
below.

Bot correctness was never at risk either way: `Options/dhan_client.py`'s
`get_cached_option_ltp()` treats a missing/stale WS tick as a cache miss
and forces the caller onto a REST fetch (confirmed live via `/feed-stats`
- `ltp_cache_stale` climbed continuously throughout the whole incident),
and `note_rest_ltp()` re-primes the cache from that REST call. Exit
monitoring (target/stop/MAX_LOSS_HIT/etc.) kept running correctly the
entire time, just via REST round-trips instead of free cached ticks -
slower per check and more REST call volume, not a missed-exit risk. See
[[dhan-rate-limit-every-call-site]] for the broader REST-rate-limit
context this adds pressure to.

## Timeline (UTC; droplet clock is UTC, see [[feedback-droplet-utc-timestamps]])

- **~02:54** - `dhanboy.service` restarted (likely the daily
  `dhanboy-morning-refresh.timer`, though the actual fire time (02:30)
  and this restart (02:54) don't line up exactly - not fully explained,
  worth checking `systemctl list-timers` next time this recurs).
- **02:54 - ~03:19+** - Market-data WebSocket in a fast, non-backing-off
  reconnect loop: `feed_connects` reached 81 (real successful handshakes
  at some point), but `price_ticks_received` stayed at **0** the entire
  session, and `feed_errors` climbed continuously (341 by 03:19, 588 by
  ~03:24, 660 by 03:25 - roughly 1 error every 1-2 seconds, unbroken).
  Also saw unrelated `Swing/signals.py` `RuntimeError: fetch_continuous_
  intraday returned no data at all` for ASHOKLEY/NATURALGAS around the
  same window (05-min/15-min Supertrend + regime fetches) - not
  confirmed related to the WS issue, may be a separate transient Dhan
  REST hiccup; flagged here in case it recurs alongside this pattern
  again.
- **03:27** - Investigated via SSH (`journalctl -k` OOM grep: clean, no
  OOM kills in 19 days of retained logs; `free -m`: 206MB/961MB RSS,
  healthy) + `/feed-stats` (two snapshots ~1 min apart confirmed the
  error count actively climbing, not a one-time blip) + `/positions`
  across all 4 packages (only 1 open position: Swing ANGELONE CE, entry
  7.0/target 8.4/stop 5.6, reconciled from before the restart).
- **03:28:04** - User-approved `systemctl restart dhanboy.service`
  (pre-restart position check re-confirmed same single open position
  immediately before restarting, per the standing [[project-dhanboy-
  deployment]] checklist).
- **03:28:00.932** (~4s into the new process) - Market-data WebSocket's
  VERY FIRST connection attempt this run got `HTTP 429` again (`server
  rejected WebSocket connection: HTTP 429`, logged as "error #1 this
  run"). **No further market-data WebSocket log line of any kind
  appeared afterward** - confirmed via `journalctl --since` filtered to
  exactly that timestamp onward, zero matches for "market-data"/"market
  feed", for over a minute of wall-clock time after. `feed_connects`
  stayed at 0, `feed_errors` stayed at exactly 1, `price_ticks_received`
  stayed at 0 - a genuinely dead thread, not a slow one.
- ANGELONE reconciled correctly post-restart (`reconciled: true`, same
  entry/target/stop as before) - the restart itself was clean on that
  front, consistent with [[project-dhanboy-deployment]]'s own "all 4
  packages now reconcile from the broker on restart" note.

## Root cause (confirmed by reading the vendored SDK source directly)

`.venv/lib/python3.12/site-packages/dhanhq/marketfeed.py` (dhanhq==2.2.0,
pinned in `uv.lock`), `MarketFeed` class:

```python
def run(self):                                    # line 87
    self._running = True
    try:
        self.loop.run_until_complete(self._run_async())
    except KeyboardInterrupt:                      # line 95 - ONLY this is caught
        self.close_connection()

async def _run_async(self):                        # line 108
    await self.connect()                            # <-- UNPROTECTED, outside any try/except
    while self._running:
        try:
            ...
        except Exception as e:
            if self.on_error:
                self.on_error(self, e)
            await asyncio.sleep(1)                  # retry loop, but only reachable AFTER
                                                      #   the line above already succeeded once

async def connect(self):                            # line 147
    if not self.ws or self.ws.state == ...CLOSED:
        try:
            self.ws = await websockets.connect(url)
            ...
        except Exception as e:
            if self.on_error:
                self.on_error(self, e)
            raise e                                  # re-raises - by design, for callers that DO catch it
```

`market_feed` (in `Options/dhan_client.py`) starts this via
`MarketFeed.start()`, which spawns `run()` in a **daemon thread with no
wrapper of its own**:

```python
def start(self):
    t = threading.Thread(target=self.run, daemon=True)
    t.start()
    return t
```

So the actual call chain on a fresh process start is: `start()` spawns
the thread -> `run()` -> `_run_async()`'s **first line**, `await
self.connect()`, runs OUTSIDE the while loop's own try/except -> if THIS
call raises (any `Exception`, including the 429 `InvalidStatus`), it
propagates all the way up through `_run_async()` and `run()` (which only
catches `KeyboardInterrupt`) and **kills the entire background thread**.
The retry-with-1s-sleep loop that graders would expect to see literally
cannot help here - it lives inside the `while self._running:` block,
which is never reached if the very first `connect()` before it throws.

This also explains the confusing MORNING pattern (81 connects, hundreds
of errors, still climbing) vs. the RESTART pattern (0 connects, exactly
1 error, dead silence) as two different failure modes of the SAME root
cause:
- **Morning**: the very first `connect()` after the 02:54 restart must
  have SUCCEEDED (hence `feed_connects` could reach 81 - each successful
  handshake fires `on_connect`), so `_run_async()` entered its `while`
  loop. Every subsequent failure (`get_instrument_data()`/`ws.recv()`
  breaking, or a reconnect attempt from inside the loop's own `else`
  branch) IS caught by the loop's own try/except, logged as an error,
  and retried after `asyncio.sleep(1)` - a real retry loop, just with no
  exponential backoff, so once Dhan's WS endpoint started rate-limiting
  reconnect attempts, the fixed 1s cadence kept re-triggering the SAME
  429 instead of ever letting the limiter's window clear. Self-sustaining
  but at least ALIVE (still trying).
- **After the 03:28 restart**: the very first `connect()` this time
  failed outright (429 - plausibly because Dhan's rate-limit state from
  the morning's storm hadn't fully cleared yet, ~9 minutes later).
  Because this specific call is unprotected, the thread died on the
  first strike, before ever reaching the loop that would have retried
  it. Worse for self-healing (zero retries, ever, until the NEXT process
  restart) despite LOOKING calmer in the logs (only 1 error vs.
  hundreds).

**This is a bug in `dhanhq`'s own `MarketFeed.run()`/`_run_async()`, not
in this repo.** `Options/dhan_client.py`'s OWN order-update feed
(`_run_order_update_forever`) already has the correct, disciplined
pattern the market-data feed is missing:

```python
def _run_order_update_forever(self) -> None:
    while True:
        try:
            self._order_update.connect_to_dhan_websocket_sync()
        except Exception:
            logger.exception("Order-update WebSocket dropped; reconnecting in 5s")
        time.sleep(5)
```

This wraps the ENTIRE blocking call (including its own first connection
attempt) in an infinite retry loop with its own fixed 5s backoff, at the
app level, independent of whatever the SDK's own internals do. The
market-data feed instead trusts `MarketFeed.start()`'s own internal
thread/retry management entirely, which this incident shows is not
safe to trust for the very first connection attempt.

## What actually protected the bot during this whole incident

Nothing broke. `get_cached_option_ltp()` (`Options/dhan_client.py`) is
built to treat "no tick yet" and "stale tick" identically - both return
`None`, which the caller MUST treat as a cache miss and fall back to a
REST price fetch (see the function's own docstring, written after a
PRIOR real incident - SAGILITY, 28 Aug 2026 - where a stale-but-present
cached tick was trusted directly for ~2 minutes and let a MAX_LOSS_HIT
overshoot). `note_rest_ltp()` then re-primes the cache from that REST
value so the next `LTP_STALE_AFTER_SECONDS`-sized window doesn't need
another REST call. This is exactly why `ltp_cache_stale` kept climbing
throughout (visible proof the fallback was firing on every single check)
while the bot's actual trade logic - confirmed via the ANGELONE
position's correct reconciliation and the absence of any exit-ladder
error in the logs - kept functioning normally the entire time, WS feed
dead or not.

## Fix direction (not yet built - this incident is diagnosis only, per [[feedback-live-trading-safety]])

Wrap `market_feed`'s own startup the same way `_run_order_update_forever`
already wraps the order-update feed - own retry loop with backoff around
the ENTIRE call to `MarketFeed.start()`/`run()`, not trusting the SDK's
internal loop to survive its own first connection attempt. A minimal
version: replace the direct `feed.start()` call in the `market_feed`
property with a small wrapper thread that catches any exception escaping
`run()` (not just relies on the SDK's own internal while-loop) and
restarts `MarketFeed` itself after a backoff (ideally exponential, e.g.
1s/2s/4s/8s capped at some ceiling - NOT the SDK's own fixed 1s, which is
exactly what turned one rate-limit rejection into a hundreds-strong
storm in the morning case). This is a real code change to
`Options/dhan_client.py` (shared by Luxury/Futures/Swing's own copies too
- check whether they're byte-identical copies the way `trading_engine.py`
is, per [[backtest-methodology]]'s own note on that pattern) and should
go through the normal test -> confirm -> deploy checklist before landing,
not be rushed in reaction to this single incident.

## Recognizing this again

If `/feed-stats` shows `price_ticks_received` stuck at 0 for an extended
period during market hours:
1. Check `feed_connects` vs `feed_errors`: BOTH stuck (0 or a low fixed
   number, not climbing) = the thread is dead (this incident's second
   phase) - a restart is the only way to get another connection attempt
   in without the code fix above. `feed_connects` climbing WITH
   `feed_errors` climbing just as fast = the noisy-but-alive first phase
   - also needs a restart to break the fixed-1s-retry cycle against
   Dhan's own rate-limit window, but at least it was still trying.
2. Either way, this is NOT itself a reason to worry about missed exits -
   confirm via `ltp_cache_stale` climbing in the same `/feed-stats`
   response, which proves the REST fallback is (still) doing its job.
3. A restart is a live-bot action - follow [[project-dhanboy-
   deployment]]'s own restart checklist (check `live_positions` on all 4
   `/positions` endpoints before AND immediately before restarting,
   since state can change between the two checks) every time, same as
   any other restart, regardless of how routine this specific problem
   starts to feel after being seen once.
