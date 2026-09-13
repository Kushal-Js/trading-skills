# 2026-09-13 — Raising the default executor pool triggered a WebSocket reconnect storm; confounded by testing on a closed market

## What happened

User asked whether the bot risks slowing down or crashing under real
parallel-trade load. Investigation found the whole process (Options,
Futures, Luxury, Swing) shares Python's default `ThreadPoolExecutor` for
every blocking Dhan/Tradehull call, sized `min(32, cpu_count()+4)` — just
**5 threads** on the droplet's 1 vCPU. `TOP_N_STOCKS=4` entries already
run concurrently via `asyncio.gather`, which alone can nearly saturate
that pool. User approved raising it to 10 (`loop.set_default_executor`,
set once at `main.py`'s `lifespan` startup, before any strategy
authenticates).

Change was unit-tested locally (isolated concurrency benchmark showed a
clean 2x speedup for a 20-call burst at 5→10 workers; full test suite
re-run before/after via `git stash` showed zero new failures) and
deployed. Immediately after the restart, the Dhan market-data WebSocket
entered a reconnect storm: **~150 reconnect attempts in under a minute**,
escalating to `HTTP 429` (Dhan's own rate limit) rejecting the connection
outright. The immediately-prior restart *that same day, before the
change* was completely clean (0 WS errors) — a same-day, same-market-
state before/after contrast that clearly implicates the change itself
for the *onset* of the storm.

Reverted the change (`git revert`, redeployed) as a precaution — real
position (ADANIPORTS Swing futures, NSE_FNO, carried overnight) was open
throughout. Restarted twice more to try to recover a clean WS connection.
Instability continued and even worsened after the revert (40 connects /
241 errors on the third restart, still zero price ticks received,
`best_price` frozen at entry price) — at this point indistinguishable
from the change itself vs. residual fallout vs. something else.

**The something else, surfaced by the user**: it was **Sunday — market
closed**. No live data flows into Dhan's feed on a non-trading day
regardless of anything the bot does; `get_ltp_data` returning
`{'status':'failure'}` for the position's contract had **798 prior
occurrences already logged before the executor change was ever
deployed**, going back to the very first restart of the day — i.e. it
was happening on a totally clean baseline too.

## Why this was hard to read in the moment

- The *first* WS storm's onset really was clean vs. broken on the same
  Sunday (both restarts equally market-closed) — so "it's the weekend"
  doesn't explain why restart N was fine and restart N+1 immediately
  weekend by itself. This part of the causal read holds up.
- But once one storm happens and gets Dhan to briefly rate-limit the
  connection, every *subsequent* restart that day inherits an ambiguous
  starting state: is continued instability the lingering rate-limit
  penalty, is it the closed-market feed behaving oddly with nothing to
  actually stream, or both? On a live trading day the feed would have
  real quotes flowing and either fully recover or fully fail in a way
  that's much easier to read. On a Sunday, "no ticks, lots of connect/
  disconnect churn" is ambiguous by construction, because that's also
  roughly what a dead feed on a live day would look like from the
  outside — the one distinguishing signal (does price data ever flow
  once reconnected) can't fire when there's no data to flow regardless.

## The lesson

**Don't judge a WS/live-feed-dependent infra change's health from
restarts made on a non-trading day.** A clean local unit test + full
regression suite pass is necessary but not sufficient for anything that
touches the shared Dhan connection/executor — the only environment that
can actually confirm or rule out a feed-side regression is a real
restart during market hours, where ticks either flow or they
observably don't. Testing on a closed market can produce a false
positive (looks like an incident, isn't) that's genuinely hard to tell
apart from a real one, and burns real investigation time and (mildly)
the account's own rate-limit headroom via repeated restarts chasing a
signal that was never going to resolve.

Practical takeaway for next time this class of change comes up
(anything touching `dhan_wrapper`'s WS/executor/connection lifecycle):
stage it, but hold the market-hours verification step for an actual
trading day rather than trying to confirm it live on a weekend just
because the code is ready.

## Status

Executor change fully reverted (`main.py` back to relying on Python's
default 5-worker pool). No real-money impact — market was closed the
entire time, so there was nothing for a blind exit-ladder to miss. User
will request re-attempting the change on a future trading day, where a
clean before/after WS-health comparison is actually possible.

Related: [[icicipruli-unmonitorable-position]] — same underlying "no LTP
→ no exit-ladder protection" mechanism, different trigger (there: a
genuinely dead live-market feed for one contract; here: no market open
at all). Both point at the same still-unbuilt gap noted there: no
staleness watchdog that alarms or acts when a position hasn't been
priced in N minutes.
