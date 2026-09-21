Status: 5-DAY (Simply Bull) COMPARISON COMPLETE, 21 Sep 2026 - zero
measurable backlog effect at this signal volume. 210-STOCK/14-DAY
COMPARISON STARTED THEN PAUSED (user request, "hold it off for some time
then") before completing - not yet re-run as of this write-up.

# Isolating the capacity backlog's own PnL effect

## What this tests

Whether [[universe-bucket-dispatcher-design]]'s capacity backlog (queue a
signal both Luxury and Futures decline purely on
`duplicate_or_capacity_full`, retry it once a slot frees, gated by a real
momentum-reversal check) actually changes outcomes versus the dispatcher
alone (round-robin + sequential fallback, no backlog). Both scenarios use
IDENTICAL round-robin rotation state, so any difference between them is
attributable to the backlog alone, not to dispatch-order differences.

## Method

`traderBoy/backtest_dispatcher_capacity_backlog_5day.py`. Same signal set
as [[simply-bull-screener-5day-backtest]] (Simply Bull CSV, last 5
distinct dates, CE-only, Luxury+Futures), run twice: Scenario A
(dispatched, no backlog) and Scenario B (dispatched + capacity backlog),
via an event-driven simulation (a merged, chronologically-sorted queue of
signal arrivals AND position-close events - a backlog retry is attempted
at every such event, mirroring the live dispatcher's own per-tick backlog
drain).

## Result: ZERO difference

| Scenario | Trades | Wins | Win Rate | PnL |
|---|---|---|---|---|
| A) Dispatched, no backlog | 31 | 26 | 83.9% | +Rs47,604.85 |
| B) Dispatched + capacity backlog | 31 | 26 | 83.9% | +Rs47,604.85 |

Not a bug - the backlog mechanism itself was separately verified correct
via targeted unit tests (synthetic ticks/fake entry functions: momentum-
reversal detection correctly drops stale entries, a still-valid-but-
capacity-blocked entry correctly survives one cycle and succeeds once a
slot frees on the next). It simply never got exercised on this dataset:
capacity was never actually the binding constraint at Simply Bull's
signal volume (avg ~6 signals/day against 6 total CE slots across both
packages) - where a package DID decline, it was almost always Futures'
own `volume_floor_gate` (which Luxury doesn't have), and the fallback-to-
Luxury-within-the-same-dispatch-attempt already resolves that instantly,
with nothing left to backlog.

## What's still open

The 210-stock/14-day dataset is the one where capacity contention is
real - the earlier racing simulation
([[all-fno-universe-breakout-signal-15day-backtest]]) hit 890
`duplicate_or_capacity_full`-shaped skips out of 2,184 attempts, several
orders of magnitude more than Simply Bull's scale ever produced. A
`backtest_dispatcher_capacity_backlog_210stock.py` run was started
(confirmed all 728 raw signals found, matching the earlier backtest's own
count) but paused by the user before Scenario A/B could run - queued to
resume, not abandoned. That result is what would actually show whether
the backlog earns its keep at meaningful scale; this 5-day result alone
doesn't settle that question either way.
