# Incident: UniverseDispatcher crashed every cycle from market open, blocking all dispatcher-sourced entries

## What happened

The UniverseDispatcher (added 21 Sep, the sole path for turning universe_
bucket's webhook-sourced CE/PE alerts into Luxury/Futures entries) began
raising an unhandled `AttributeError` on its very first backlog-cycle
call after market open and continued failing on **every single 60s scan
cycle** from ~09:10 IST through at least 09:18 IST - caught only when the
user asked why "lots of alerts" (universe_bucket had grown to 27 CE / 26
PE symbols from real webhook traffic) hadn't produced a single trade.

```
AttributeError: module 'Luxury.config' has no attribute
'BREAKOUT_CAPACITY_BACKLOG_MAX_AGE_MINUTES'
```

## Root cause

`universe_dispatcher_loop` (`breakout_signal.py`) reads shared dispatcher
settings off whichever target happens to be first in the list `main.py`
passes it:

```python
dispatcher_task = asyncio.create_task(breakout_signal.universe_dispatcher_loop([
    ("Luxury", luxury_config, luxury_main._breakout_entry_fn),
    ("Futures", futures_config, futures_main._breakout_entry_fn),
]))
...
primary_cfg = targets[0][1]  # = luxury_config, by this ordering
...
await _dispatch_backlog_cycle(primary_cfg, targets)
```

`BREAKOUT_CAPACITY_BACKLOG_MAX_AGE_MINUTES` was added (fbd11f9, 21 Sep) to
`Options/config.py` only, documented there as "the shared master flag" -
but `primary_cfg` was never actually Options' own config, it was whichever
package happened to be `targets[0]`. Every other flag `_dispatch_backlog_
cycle`/`_dispatch_scan_cycle` reads (`BREAKOUT_SCAN_MAX_PER_CYCLE`,
`BREAKOUT_SCAN_PACE_SECONDS`, `BREAKOUT_SIGNAL_ENABLED`, etc.) happened to
already exist per-package in both Luxury's and Futures' own configs, so
this one gap went unnoticed until the backlog cycle actually ran with a
non-empty backlog for the first time today - the previous session's
testing apparently never exercised this exact code path live.

## Real trading impact - real, not hypothetical

For the ~8-10 minutes this was broken, **every** universe_bucket-sourced
alert for both Luxury and Futures was silently dropped: the dispatcher
never got past the backlog check to reach `_dispatch_scan_cycle` at all
(the whole `try` block's exception took down both calls for that cycle).
universe_bucket kept accumulating real Chartink webhook alerts the whole
time (19 -> 27 CE, 0 -> 26 PE) with **zero** of them ever reaching an
entry attempt. Exact forgone-trade count/PnL is not recoverable (no
record of which of those symbols would have passed the entry gates), but
this was live money-on-the-table during real market hours, not a
backtest or paper-mode gap.

**Not fully blocked, worth being precise about**: Luxury's (and
presumably Futures') OWN independent `breakout_signal.signal_scanner_loop`
- a separate loop, running its own static watchlist, calling the exact
same `_breakout_entry_fn` the dispatcher calls - was never affected by
this crash and continued operating normally. That's how GVT&D and
CGPOWER (Luxury, both real entries, GVT&D closed +Rs1,775 at TARGET_HIT
before this was even found) got placed during the outage window. Only
the universe_bucket/dispatcher path was down.

## Fix

Defined `BREAKOUT_CAPACITY_BACKLOG_MAX_AGE_MINUTES` identically (same env
var name, not per-package-prefixed, since it's meant to be one shared
value) in `Luxury/config.py` and `Futures/config.py` too, so the
dispatcher's behavior no longer depends on which target ends up
`primary_cfg`. Verified every other `cfg.*`/`primary_cfg.*` attribute the
dispatcher touches already existed in both configs - this was the only
gap in that specific code path. Deployed commit `02b515a`, restart
03:53:23 UTC (09:23 IST) - by the user's own action, with 2 real open
Luxury positions and 1 real open Options position at restart time, all
confirmed reconciled correctly post-restart (`reconciled: true`,
correct entry price/highest price/stop-loss order id preserved).
Futures picked up a new HDFCBANK CE entry within 35 seconds of the
restart - first live confirmation the dispatcher path is actually
dispatching again.

## Broader audit done same session (user asked to check for other silent/hidden bugs)

Given this bug's exact shape - a config attribute assumed present
everywhere but only actually defined in one package - ran a systematic
diff of every config variable name across Options/Luxury/Futures
`config.py`, cross-referenced against every `cfg.*`/`primary_cfg.*`/
`config.*` attribute referenced by the shared cross-package modules
(`breakout_signal.py`, `universe_bucket.py`, `breakout_paper_engine.py`).

Found one more asymmetry: `VOLUME_FLOOR_RATIO_MIN`/`VOLUME_FLOOR_GATE_
ENABLED` exist in Options/Futures but not Luxury - **verified safe**,
not fixed: `breakout_paper_engine.py`'s only reference is `if
getattr(cfg, "VOLUME_FLOOR_GATE_ENABLED", False): ... cfg.VOLUME_FLOOR_
RATIO_MIN`, so a missing attribute on Luxury's config correctly defaults
the gate to disabled rather than crashing - the inner (unsafe) access is
unreachable when the attribute is absent. No fix needed; noted here so
it isn't re-investigated from scratch later, and so it's remembered if
`breakout_paper_engine.py` is ever enabled for Luxury and someone expects
this gate to actually be active (it currently can't be, silently).

Also verified: `_breakout_entry_fn` (both Luxury's and Futures') is
explicitly the SOLE real entry path for its package, called identically
by the dispatcher and by that package's own independent
`signal_scanner_loop` - both converge on the same atomic `reserve_symbol`
claim in `_process_one_entry`, so there is no duplicate-order risk
between the two concurrent signal sources. No exception types other than
the known regime/Supertrend DH-904 pattern (already documented,
[[2026-09-22-swing-signal-cache-never-throttled-on-failure]]) appeared in
logs since the restart. Memory stable at ~341M/961M, no swap growth.

## Lesson

A flag documented as "the shared master flag, defined once" is only
actually shared if every code path that might read it agrees on WHERE to
read it from. `primary_cfg = targets[0][1]` quietly turned an intended
single-source-of-truth value into "whichever package happens to be
listed first" - correct today only by coincidence (Luxury and Futures'
per-package copies of every OTHER flag this loop reads happened to
already match). Any future flag added to only one of Options/Luxury/
Futures' configs but consumed by code that might run under any of them
needs either (a) defined identically in all three, or (b) read
explicitly from a named module (`options_config.X`), never through a
positional `targets[0]`-style indirection.
