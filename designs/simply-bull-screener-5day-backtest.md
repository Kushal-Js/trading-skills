Status: BACKTEST COMPLETE, 21 Sep 2026. Predates the universe_bucket
dispatcher (built and deployed later the same day) - this run used the
OLDER "racing" logic (each package independently syncs the same shared
symbol list into its own watchlist, both may attempt the same signal,
`cross_strategy_registry` breaks ties). Not wired into any live package.

# Simply Bull screener, 5-day backtest (racing logic, CE-only)

## What was asked

User's own real Chartink screener export
(`~/Desktop/future/01 Simply Bull.csv`, columns Date/Symbol/Marketcapname/
Sector) used as the per-day curated universe, instead of a hand-picked
static list or the full 210-stock F&O universe - the concrete "Option 2:
curated subset" from the earlier scoping conversation. Scope: Luxury +
Futures only (per standing user instruction), CE-only (the screener's own
name is unambiguously bullish).

## Method

`traderBoy/backtest_simply_bull_screener_5day.py`. Parsed the CSV's own
per-day symbol lists, took the last 5 DISTINCT dates present (2026-09-11,
09-16, 09-17, 09-18, 09-21 - note 09-15 has zero rows in this CSV even
though it's a real trading day with data in every other backtest's own
DAYS list; confirmed not a gap, the screener genuinely found nothing
bullish that one day), filtered to F&O-eligible (zero dropped - Chartink's
own screener output was already fully F&O-scoped). Real entry gates, real
exit ladder, real live-deployed thresholds - same machinery as
[[all-fno-universe-breakout-signal-15day-backtest]], reused by import
rather than reimplemented. Each raw signal was attempted by BOTH Luxury
and Futures (racing, pre-dispatcher design) - the shared `open_positions`
dict (keyed by symbol only) enforces the same-underlying exclusivity, but
whichever package's pending-list entry happened to be processed first
(Luxury, by fixed insertion order - no round-robin in this run) got first
crack.

## Result

| | Trades | Wins | Win Rate | Total PnL |
|---|---|---|---|---|
| REAL (Luxury+Futures, 5 days) | 121 | 42 | 34.7% | -Rs31,157.40 |
| SIM (Simply Bull universe, racing) | 35 | 29 | 82.9% | **+Rs47,257.35** |

31 raw signals, 35 entered (some symbols entered by both packages on
their own separate attempts across the window - a same-instant same-fill
duplicate pattern shows up 3 times: YESBANK/PAYTM 09-16, PATANJALI/
INDHOTEL 09-21 - a smaller version of the triple-counting artifact found
in the earlier 210-stock backtest, since Futures mostly got filtered out
here by its own `volume_floor_gate`, which Luxury doesn't have, leaving
little room for the two to actually collide). By package: Luxury 31
trades/+Rs47,604.85 (real: n/a broken out separately in this run);
Futures 4 trades/-Rs347.50.

## Caveats

Same standing caveats as every backtest in this line of work: real PnL
mostly predates any breakout-signal-gated deploy (old entry logic for
almost this whole window), no funds/margin cap modeled, small sample (35
trades). Additionally: this run used the OLDER racing dispatch logic, now
superseded live by [[universe-bucket-dispatcher-design]] - see
[[dispatcher-capacity-backlog-comparison]] for the isolated backlog-
specific re-run against this exact same signal set (found the backlog
made zero difference at this signal volume - capacity was never actually
the binding constraint).
