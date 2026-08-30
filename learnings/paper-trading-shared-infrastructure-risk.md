# "Paper trading only" doesn't mean "zero risk to real trading"

Found while asking, before K01's first live session (30 Aug 2026): "would
this paper trading impact real time trading tomorrow?" The honest answer
had two parts, and the second one is the generalizable lesson.

## The guarantee "paper trading only" actually gives you

A strategy asserting `PAPER_TRADING_ONLY = True` at startup, with no code
path to `dhan_wrapper.place_market_order`/`.client.order_placement`
anywhere in its file, genuinely cannot place a real order. This part is a
hard, verifiable guarantee — confirmed by code inspection, not just intent.

## The guarantee it does NOT give you

If the paper strategy shares **infrastructure** with a real-money
strategy — the same authenticated broker connection, the same process,
the same rate-limit budget — then the paper strategy's own activity can
still *indirectly* affect the real one, even though it never touches an
order. Concretely for this codebase: every strategy package
(`Options/`, `Futures/`, `IndexScalping/`, `CopperOptions/`, `K01/`) reuses
one shared `dhan_wrapper` singleton and runs in one shared process
(`main.py`'s combined lifespan). A paper strategy making its own REST
calls against that shared connection competes for Dhan's undocumented
rate limit (`traderBoy/NOTES.md` bug #5) alongside the real strategies'
own exit-monitoring calls (LTP checks, Supertrend refreshes). If that
contention ever caused a real position's exit check to be delayed or
retried at the wrong moment, that's a real-money consequence with a
paper-only strategy as its root cause.

This is not a new risk *mechanism* — it's the same one already documented
in `learnings/exit-mechanics.md`'s stale-LTP finding — but a new paper
strategy is a new *source* of load feeding that same shared bottleneck.

## The check to run before any new paper strategy's first live session

1. Does it reuse the same broker connection/process as anything placing
   real orders? (In this codebase: yes, always — that's the established
   architecture, see `main.py`'s docstring.)
2. How much REST call volume does it add, and at what cadence, relative to
   what the real strategies already generate?
3. Is there a known, already-hit rate limit on that shared resource? (Yes
   here — bug #5.)

If 1 and 3 are both true, treat the new strategy's own polling cadence/
call volume as a real-money-adjacent decision, not just a "how responsive
do we want the paper results to be" tuning knob — reason it through (and
default conservative, e.g. off, or throttled) *before* the first live
session, not after noticing a problem.
