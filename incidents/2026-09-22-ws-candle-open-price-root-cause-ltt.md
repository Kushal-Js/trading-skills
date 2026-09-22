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
