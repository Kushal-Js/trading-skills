Status: BACKTEST COMPLETE, 21 Sep 2026. Uses the CURRENTLY DEPLOYED
dispatcher + capacity-backlog logic (`UNIVERSE_DISPATCHER_ENABLED=true`,
live since 20:35 IST the same day - see
[[universe-bucket-dispatcher-design]]) - the PE-side counterpart to
[[simply-bull-screener-5day-backtest]], and the first backtest run
AFTER the dispatcher existed rather than the older racing logic.

# Simply Bear screener, 5-day backtest (dispatcher + backlog, PE-only)

## What was asked

The user's second real Chartink screener export
(`~/Desktop/future/01 Simply Bear.csv`, same column shape as Simply Bull)
- the bearish/PE counterpart, backtested with the "new logic" (the live
dispatcher + capacity backlog, not the older racing design Simply Bull's
own first backtest used). Scope: Luxury + Futures only, PE-only (the
screener's own name is unambiguously bearish).

## Method

`traderBoy/backtest_simply_bear_dispatcher_backlog_5day.py`. Last 5
distinct CSV dates: 2026-09-15, 09-16, 09-17, 09-18, 09-21 - a genuinely
consecutive run, unlike Simply Bull's own window (Simply Bear found 132
bearish picks on 09-15 alone; Simply Bull found zero bullish picks that
same day - the two screeners clearly disagree on some days, which is
expected and not a bug in either). Dropped 3 non-F&O-eligible symbol-days
(NIFTY, BANKNIFTY - this CSV includes index names as plain rows despite
them not being individual F&O stocks; bsr.fno_eligible_symbols() is
OPTSTK/per-stock only, so these were excluded the same way any other
non-F&O pick would be). Single scenario only (the live/deployed logic,
not a comparison) - each confirmed bearish breakdown detected ONCE,
offered to Luxury then Futures round-robin + sequential fallback, with
the capacity backlog retrying any signal both declined purely on
`duplicate_or_capacity_full`.

## Result

| | Trades | Wins | Win Rate | Total PnL |
|---|---|---|---|---|
| REAL (Luxury+Futures, 5 days) | 120 | 44 | 36.7% | -Rs20,968.65 |
| SIM (Simply Bear universe, dispatcher+backlog) | 34 | 28 | 82.4% | **+Rs50,159.14** |

93 raw PE signals (55 of them on 09-15 alone), 34 entered. By package:
Luxury 20 trades/+Rs33,753.00 (real: 71 trades/-Rs10,060.10); Futures 14
trades/+Rs16,406.14 (real: 49 trades/-Rs10,908.55). Exit mix: 25
TARGET_HIT, 5 LIQUIDITY_GUARD_ZERO_VOLUME, 3 TRAILING_SL_HIT, 1
PROFIT_PROTECTION_HIT. Unlike the Simply Bull run, **zero same-instant
duplicate captures** this time - the round-robin dispatch cleanly split
load across both packages with no collisions in this dataset.

## Caveats

Same standing caveats as every backtest in this line of work: real PnL
mostly predates the dispatcher's own live deploy (old entry logic for
nearly this whole window - the dispatcher went live at 20:35 IST on the
LAST day of this 5-day window), no funds/margin cap modeled, small sample
(34 trades). This IS the first backtest run using the actual deployed
dispatch logic rather than a reconstruction of it, so it's the closest
thing to "what the live system would have done" of any backtest in this
line of work so far - still not a live result, a historical replay.
