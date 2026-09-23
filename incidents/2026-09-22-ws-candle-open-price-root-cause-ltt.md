# WS candle open-price mismatch, root-caused: ticks bucketed by receipt time, not trade time

## Background

The 22 Sep live parity checks ([[ws-candle-reconstruction-parity-results]])
confirmed a real, non-artifact open-price mismatch in `underlying_
candle_feed.py`'s 5-min bar reconstruction - up to 75% of bars wrong on
some symbols (TCS, ICICIBANK worst), while close (10-12/12 bars within
0.05% on every symbol) and volume (after the separate baseline fix,
[[2026-09-22-swing-signal-cache-never-throttled-on-failure]] adjacent)
stayed accurate. That specific pattern - one field wrong, everything
built from aggregating many ticks fine - pointed at a boundary-timing
bug rather than a bug in the aggregation math itself.

## Root cause

`Options/dhan_client.py`'s `_on_market_tick` bucketed every Quote/Full
WebSocket tick by `datetime.now(IST)` - the moment our own process
received and handled the packet - never reading `LTT` (Last Trade Time),
a field the vendored `dhanhq` SDK already parses out of the Quote packet
(`process_quote`/`process_full`, both include `open`/`high`/`low`/`close`
day-level fields too, none of which were being read - correctly, those
are day-level not per-5-min-bar) but that nothing in this codebase ever
consumed.

A trade whose true exchange timestamp was, say, 10:29:59.8 but that our
process didn't finish handling until 10:30:00.1 (ordinary network/
processing latency, no error involved) got bucketed by receipt time into
the **10:30 bar** instead of the 10:25 bar it actually belonged to.
Since a bar's `open` is set from whichever tick is FIRST attributed to
that window, this specifically and disproportionately corrupts open -
one misplaced tick near a boundary flips the open of the bar it lands
in, while volume/close (built from accumulating dozens of ticks across
the whole bar) barely move from one tick shifting between adjacent bars.

## Timezone verification (done live, not guessed)

The SDK's own `utc_time()` helper formats `LTT` via `datetime.
utcfromtimestamp(epoch).strftime('%H:%M:%S')` - named "utc_time", which
raised a real question: does the resulting string need a further
+5:30 UTC->IST conversion before use, or is it already IST-equivalent?
Guessing wrong here would trade one bucketing bug for a worse one (every
tick off by 5.5 hours instead of by a boundary-adjacent ambiguity).

Added temporary diagnostic logging (deployed, one log line per symbol,
removed once resolved) and captured real ticks post-close (22 Sep,
~16:09 IST): local receipt time read `16:09:12`, `LTT` read `15:59:33`-
`15:59:59` across 4 symbols - ~10 minutes BEHIND receipt time, not ~5.5
HOURS behind. If the string needed the +5:30 conversion, it would have
read something like `21:29` (nonsensical for a market that closed at
15:30). The ~10-minute gap is explained by post-close staleness (no new
trades since ~16:00, Quote packets keep re-pushing the last known
LTP/LTT) - not a timezone bug. **Confirmed: the string is already
correct IST wall-clock time-of-day, no conversion needed.**

## Fix

Added `_tick_time_from_ltt(raw_ltt, received_at)`: parses `LTT`,
combines it with `received_at`'s own calendar date (NSE/MCX trading
hours never span a real midnight, so this is safe), and falls back to
`received_at` itself (the old behavior) if `LTT` is missing or
malformed - this runs on the MarketFeed's own background thread, where
an unhandled exception would be far more damaging than a slightly-stale
bucket on a rare bad packet.

Wired into `_on_market_tick`'s quote-tick-subscriber path only - the
`tick_time` argument passed to `underlying_candle_feed`'s `_on_tick`
callback now uses trade time, not receipt time. Deliberately does NOT
touch `now`/`self._ltp_cache_ts` (used elsewhere for "how fresh is our
own knowledge of the price", a genuinely different question from "which
5-min bar does this trade belong to").

5 new tests (`tests/test_tick_time_from_ltt.py`): verbatim use of a
well-formed LTT (no conversion), correct date-combining, both fallback
paths (missing/malformed LTT), and the exact boundary scenario this
fixes (a 10:29:59 trade processed 87ms after the 10:30:00 boundary still
buckets to its own true 10:25-10:30 window). All passing, both locally
and re-run directly on the droplet's deployed code.

Deployed same session (`a6e026e`), restart with 0 open positions across
all 4 packages before and after, health OK.

## What's still open

**Still not validated against fresh live data post-fix.** The market
went fully quiet (13+ minutes with zero new ticks) before a fresh bar
could complete and be compared against a real REST candle right after
this fix landed. A same-day follow-up attempt (~16:45-18:20 IST, after
15:30 IST market close) also came back with zero usable samples -
`recon_bar_count` 0-1 vs `real_bar_count: 73` on all 8 test symbols,
because the check ran outside trading hours and two unrelated
`dhanboy` service restarts during the check (11:16 UTC and 12:46 UTC)
each wiped the WS feed's in-memory subscription state before any real
trade could be captured. See [[ws-candle-reconstruction-parity-results]]'s
"22 Sep evening re-check attempt" section for the full detail. No live
positions or config were touched by this check.

The fix's correctness is established by code-level reasoning + the
empirical LTT-timezone verification + comprehensive unit tests, but the
actual open-price accuracy IMPROVEMENT (does the exact-match rate
actually go up on TCS/ICICIBANK specifically) still needs a live check
during actual NSE trading hours (09:15-15:30 IST), ideally subscribing
right at 09:15 IST open before any restart can wipe accumulated bars,
using `/debug/underlying-feed/parity/{symbol}` the same way this whole
investigation has throughout the day.

`BREAKOUT_USE_WS_CANDLES` stays off until that live re-validation
confirms the fix actually closes the gap, not just that it should in
theory.

## 23 Sep market-open attempt - confounded by 3 restarts in ~90 minutes, still no clean sample

Scheduled a one-time task for 09:20 IST market open to do exactly the
live re-validation described above (disabled once handled directly in
the live session instead - same investigation, just done interactively
rather than via the scheduled task).

Subscribed the same 8 symbols at 09:16 IST. The droplet restarted 3
times between 08:00-09:22 IST this morning: 08:00:01 (the known daily
`dhanboy-morning-refresh.timer`), 08:42:14 (unexplained, clean SIGTERM -
a deliberate restart, not a crash), and 09:21:46 (also unexplained,
clean SIGTERM) - the last one landing MID-BAR, ~1m46s into the 09:20-
09:25 window, wiping the WS subscription before that bar could complete.

Parity check at 09:27 IST (2 bars per symbol) showed **0/8 symbols with
an exact open match** and volume off by 79-99% on every bar - at first
glance far WORSE than yesterday's pre-fix baseline (TCS 50%, ICICIBANK
25%, others 75-100%). **This is NOT evidence the fix failed - both
sampled bars are individually explainable by subscription-continuity
gaps, unrelated to LTT-based bucketing:**

- **09:15 bar**: subscribed at 09:16:07, ~1 minute after this bar's true
  09:15:00 start. No bucketing logic, however correct, can recover a
  bar's true open when the subscription itself didn't exist yet when
  the bar opened - the first tick WE saw was never the market's true
  first trade of that window.
- **09:20 bar**: the 09:21:46 restart hit while this bar was still
  forming, wiping the subscription; re-subscribed at ~09:24, meaning
  the reconstructed version of this bar only reflects its LAST ~1
  minute (09:24-09:25), not its true full 09:20-09:25 window - hence
  the severe volume undercount and an "open" that's really just
  whatever LTP happened to be first observed after re-subscribing, not
  the bar's true 09:20:00 open.

Both are real, inherent limits of subscribing/re-subscribing mid-bar,
not a regression in the LTT fix itself - the fix addresses "bucket a
tick by when the trade happened, not when we received the packet"; it
cannot and was never meant to address "we have a genuine gap in tick
coverage because the subscription wasn't continuous through this bar's
whole window." Still no clean, fully-covered bar to fairly judge the
fix on. Waiting on the current (09:25-09:30) bar, which - if no further
restart interrupts it - would be the first bar fully covered by
continuous subscription since the 09:24 re-subscribe, and would be a
fair test.

**Separately worth noting**: the DH-904 rate-limit pattern from
yesterday ([[2026-09-22-swing-signal-cache-never-throttled-on-failure]])
is already recurring this morning too (ASHOKLEY regime/Supertrend
fetches failing at market open) - the account-wide call-budget question
deferred yesterday is still open and appears to still be live today.

## First clean bar (09:25-09:30 IST) - strong positive result

No restart occurred between the 09:24 re-subscribe and 09:30, so the
09:25 bar is the first one fully covered by continuous subscription -
the fair test this whole investigation has been waiting for.

| Symbol | Real open | Recon open | Open exact? | Real vol | Recon vol | Vol exact? |
|---|---:|---:|---|---:|---:|---|
| RELIANCE | 1244.8 | 1244.8 | YES | 72,147 | 69,332 | no (4% off) |
| TCS | 2093.6 | 2093.6 | YES | 98,568 | 98,568 | YES |
| MAHABANK | 83.32 | 83.32 | YES | 314,261 | 314,261 | YES |
| IDEA | 14.31 | 14.31 | YES | 5,341,599 | 5,285,210 | no (~1% off) |
| HDFCBANK | 735.2 | 735.2 | YES | 611,720 | 611,720 | YES |
| ICICIBANK | 1339.7 | 1339.7 | YES | 86,273 | 86,273 | YES |
| SBIN | 989.6 | 989.0 | no (0.6 off) | 72,237 | 72,237 | YES |
| ITC | 266.4 | 266.35 | no (0.05 off) | 105,107 | 113,112 | no (7.6% off) |

**6/8 exact open matches, 5/8 exact volume matches** - and the 2 open
misses are fractional (0.6 rupees on SBIN, 0.05 on ITC), nothing like
the multi-rupee gaps seen on yesterday's pre-fix data or on today's own
confounded 09:15/09:20 bars above. Most notably: **TCS and ICICIBANK -
yesterday's worst performers at 50% and 25% exact-match rates - are
both exact on this clean bar.**

Still n=1 clean bar - real evidence, not yet a large sample - but this
is the first genuinely uncontaminated data point this investigation has
produced, and it strongly supports the LTT fix working as intended.
Worth accumulating more clean bars (ideally an uninterrupted stretch of
several, which needs the restart pattern to settle down) before treating
this as fully confirmed, but the direction and magnitude of the result
are a clear positive signal.

## 15-minute clean monitoring window (09:29-09:44 IST) - confirmed with a real sample

No restart occurred in this 15-minute window (confirmed: service uptime
unchanged throughout, same 03:51:46 UTC start). All 8 symbols
accumulated 5-6 bars each, giving **26 clean bars total** (every bar
except each symbol's own confounded 09:15/09:20) - a real sample, not a
single lucky data point.

| Symbol | Clean bars | Exact open matches | Worst miss (rupees) |
|---|---:|---:|---:|
| RELIANCE | 3 | 2/3 | 0.2 |
| TCS | 3 | 2/3 | 0.9 |
| MAHABANK | 3 | 3/3 | - |
| IDEA | 3 | 1/3 | 0.01 |
| HDFCBANK | 3 | 3/3 | - |
| ICICIBANK | 4 | 4/4 | - |
| SBIN | 4 | 3/4 | 0.6 |
| ITC | 3 | 0/3 | 0.05 (every miss identical) |
| **Total** | **26** | **18/26 (69.2%)** | - |

**69.2% exact open match on clean bars, vs 25.5% (145/568) pre-fix** -
a real ~2.7x improvement, and every remaining miss is a fraction of a
rupee (max 0.9), not the multi-rupee gaps seen before the fix or on
today's own confounded bars. Volume on clean bars is similarly much
improved (mostly within 1%, several exact) vs the 79-99% deviations on
confounded bars.

**ITC shows a small, unusually CONSISTENT miss** - exactly 0.05 rupees
off on every one of its 3 clean bars, always in a way that doesn't
resolve. This is a distinct pattern from the other symbols' occasional,
inconsistent misses and might indicate something ITC-specific (tick
granularity, a rounding quirk, or a very small residual timing lag) -
worth a separate look, though the magnitude (~0.02% on a ~266 rupee
stock) is far below `BREAKOUT_MIN_BODY_PCT`'s 0.5% threshold and
unlikely to matter for real signal detection.

## Verdict

**The LTT fix is confirmed working** with a real, if still single-
session, sample - not just one lucky bar. This is a genuine, large
improvement over the pre-fix baseline, not noise. Recommend: let this
run through a full session before treating it as fully proven, and
separately address the restart frequency issue (3 unplanned/explained
restarts in the first 90 min of today alone), which is an operational
concern independent of the fix's own correctness. `BREAKOUT_USE_WS_
CANDLES` remains off - enabling it is a deliberate decision for the
user to make explicitly once satisfied, not an automatic consequence of
this result, per this module's own standing "a backtest/live-check
number alone is never itself authorization" discipline.
