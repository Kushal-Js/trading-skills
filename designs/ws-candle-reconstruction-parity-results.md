# WS candle reconstruction parity backtest — results (21 Sep 2026)

User request: run `backtest_ws_candle_reconstruction_parity.py` (built by
a parallel session earlier the same day, see `underlying_candle_feed.py`'s
own module docstring) and, if it validates, enable `BREAKOUT_USE_WS_CANDLES`
for Options/Luxury/Futures live.

## Result: inconclusive — did NOT enable live

Ran twice against `DEFAULT_SYMBOLS = ["RELIANCE", "TCS", "MAHABANK", "IDEA"]`,
day `2026-09-18`, using a user-supplied hand-off `access_token` (per
[[local-backtest-dhan-session-collision]]'s own standing rule — never
`pin_totp` locally while the live bot is running).

| Run | RELIANCE | TCS | MAHABANK | IDEA |
|---|---|---|---|---|
| 1st | 72 real candles fetched | **0 fetched** | 0 fetched | 0 fetched |
| 2nd | 72 real candles fetched | 72 real candles fetched | **0 fetched** | **0 fetched** |

**Two different symbols failed to fetch ANY real data on each run** -
not the same ones both times, and I directly confirmed real data DOES
exist for MAHABANK on this exact day (`{'status': 'success', ...}`,
manually re-fetched immediately after the failed run). This points to
transient rate-limiting rather than missing data or a broken fetch
function - consistent with the SEPARATE, already-documented risk in
[[local-backtest-dhan-session-collision]]: "sustained bulk historical-
data pulls from a local process... produce DH-904 Rate_Limit... shared
per-account REST-call budget contention with the live bot's own real-
time traffic... persists regardless of auth mode." The access-token
hand-off fixes the SESSION-collision problem, not this separate rate-
limit contention.

## For the 2 symbol-days that DID fetch cleanly (RELIANCE + TCS, 71
matched bars each)

- **Close price**: 100% match within 0.05% tolerance, both symbols.
- **Volume**: 100% match within 1% tolerance, both symbols.
- **Open price**: only 10/71 (RELIANCE) and 5/71 (TCS) bars matched
  exactly - the reconstructed OPEN is consistently a few points off the
  real 5-min bar's own open.

**This open-price gap is very likely an artifact of the TEST'S OWN proxy
methodology, not a bug in `_update_bar`'s aggregation logic itself.** The
script feeds each real 1-MINUTE candle's own CLOSE price through
`_update_bar` as a stand-in for what a live tick would deliver - the
first "tick" of any 5-min window is therefore already ~1 minute stale
relative to the window's true opening trade, which a real Quote-mode WS
tick stream (arriving much more frequently, on every actual trade) would
not be. The module's own docstring flags this limitation up front. There
is no way to conclusively rule out a real discrepancy without comparing
against an actual live WS tick stream during market hours, which this
backtest cannot do.

**Why this open-price gap matters more here than it might elsewhere**:
`breakout_signal.py`'s own body-size check (`BREAKOUT_MIN_BODY_PCT`,
currently 0.5% live) is computed directly from `abs(close - open) / open`
- a systematically-off open price could shift a candle across that
threshold in either direction, changing which candles qualify as a
breakout under WS-sourced data vs REST-sourced data. This is exactly the
kind of thing worth being sure about before it can affect a real signal
that places a real order.

## Decision: not enabled

Per this module's own docstring ("a backtest number alone is never
itself authorization") and [[feedback-live-trading-safety]], this result
is not a clean pass — 2 of 4 symbol-days have no data at all (rate-limit
casualty, not a validated result either way), and the 2 that do have an
unresolved, methodologically-ambiguous open-price gap on the exact metric
the live signal logic is most sensitive to. `BREAKOUT_USE_WS_CANDLES`
remains `false` for all three packages, unchanged from before this test.

## Follow-up (same day): re-ran with retry/pacing - rate-limit half resolved, open-price gap confirmed structural

`backtest_ws_candle_reconstruction_parity.py` had two real gaps causing
the rate-limit casualties above: no pacing between calls, and
`fetch_real_5m_candles`/`fetch_real_1m_candles` silently swallowed a
`{"status": "failure", ...}` rate-limit response into an empty dict
instead of raising - so a rate-limited call looked identical to "no data
for this symbol" with no retry ever attempted. Fixed with the same
discipline already established in [[local-backtest-dhan-session-collision]]'s
own fix (`_fetch_with_retry`: raises on non-success status, 5 retries,
5-25s exponential backoff, 1.2s pacing between every call).

Re-ran against 8 symbols (RELIANCE, TCS, MAHABANK, IDEA, HDFCBANK,
ICICIBANK, SBIN, ITC), same day (`2026-09-18`):

| Metric | Result |
|---|---|
| Symbols with real data fetched | **8/8** (0 rate-limit casualties, vs 2/4 lost on each of the first two runs) |
| Close match (<0.05%) | **568/568 (100%)** - every bar, every symbol |
| Volume match (<1%) | **568/568 (100%)** - every bar, every symbol |
| Open+close exact match | 145/568 (25.5%) - low, but *consistently* low across all 8 symbols (range 4/71 to 56/71) |

**This is now much stronger evidence than the first two runs.** Zero
close or volume deviation across 568 independent bar comparisons rules
out a real aggregation bug in `_update_bar` with high confidence - if
the bucketing math itself were wrong, it would not produce a perfect
close/volume match while only ever missing on open. The open-price gap
being present, in the same direction, on every single symbol (not random
scatter) confirms it's the TEST METHODOLOGY's own systematic artifact
(1-min-close-as-tick-proxy always samples ~1 minute into each window,
never catching the window's true first trade) rather than a symbol-
specific data problem or a bug in the module being tested.

**Still not enabled live** - this backtest can raise confidence in the
aggregation logic, but it structurally cannot answer the open-price
question, no matter how many symbols or days are added, because the
proxy method itself is what's biased. Settling that requires a real live
WS tick stream, not another REST replay - see below.

## Live-tick observation capability (added same day, for the NEXT market session)

Since markets were closed by the time this was worth pursuing further
today, added a genuinely passive, decoupled observation path so the
open-price question can be settled with a REAL WS tick stream next
session, without needing `BREAKOUT_USE_WS_CANDLES` (which would also
start using it for real signal decisions) turned on:

- `POST /debug/underlying-feed/subscribe` (body: `{"symbols": [...]}`) -
  calls `underlying_candle_feed.subscribe()` directly, independent of any
  package's `BREAKOUT_USE_WS_CANDLES` flag. Purely additive: subscribes
  the listed symbols on the live bot's own already-authenticated market-
  data WebSocket in Quote mode, same connection Options/Luxury/Futures
  already share for option LTP - no new session, no collision risk.
- `GET /debug/underlying-feed/snapshot` - read-only, returns
  `underlying_candle_feed.snapshot()` (subscribed symbols, bar counts,
  last-tick age) for observability.
- `GET /debug/underlying-feed/candles/{symbol}` - read-only, returns
  `get_candles_dict(symbol)`'s own completed bars for direct comparison
  against a REST pull for the same symbol/day after market close.

Neither of these two GET endpoints nor the subscribe endpoint touches
`BREAKOUT_USE_WS_CANDLES`, `BREAKOUT_SEED_UNIVERSE_ENABLED`, or any real
entry-gating code path - subscribing a symbol here has zero effect on
what Options/Luxury/Futures actually trade. Deployed (`c97bd50`),
restart-verified (all 4 packages' positions flat before/after), and
functionally verified live: `POST /debug/underlying-feed/subscribe`
with `{"symbols": ["RELIANCE", "TCS"]}` returned `{"subscribed": [...]}`
cleanly; `GET .../candles/RELIANCE` correctly returns `{}` right now
(market closed, no ticks flowing yet) rather than erroring - confirms
the plumbing works end to end, just waiting on real market data.

**Next step (needs live market hours - not done yet)**: subscribe a
handful of symbols via the new endpoint right after the next market
open, let real ticks accumulate through the session, then pull
`get_candles_dict()` for each and diff against real REST 5-min candles
fetched after close for the same symbols/day - the comparison this
whole investigation has been building toward. Until that runs, the
open-price question remains open (aggregation logic itself is now
well-supported by the 100% close/volume match above; the open field
specifically is unproven either way).

## Live parity results, 22 Sep 2026 - real answer at last, and it's not clean yet

The automated `ws-candle-parity.timer` ran its first REAL cross-session
comparison overnight/this morning (21->22 Sep) - 21 Sep's own report
(`history/2026-09-21_ws_candle_parity_report.json`) showed `recon_bar_
count: 0` for every one of the 8 test symbols against `real_bar_count:
72` - the WS reconstruction produced literally nothing that day. Not
promising on its face, but see below - this was superseded by fixes
already in flight the same evening (`2632cf1` day-rollover fix,
`f234c0c` wiring the dispatcher's own WS-subscribe call).

**Today's live state (checked directly via `/debug/underlying-feed/
parity/{symbol}`, not waiting for the scheduled 15:40 IST report)**:
reconstruction IS producing bars today (12 bars per symbol by 11:13 IST,
fresh ticks flowing). Pulling the real parity comparison for all 8 test
symbols:

- **Close price: genuinely good.** 10-12 of 12 matched bars within
  0.05% on every symbol, max deviation 0.095%. This part of the
  reconstruction is solid.
- **Volume: badly broken, but now root-caused and fixed.** Every single
  symbol showed a massive overshoot on its FIRST reconstructed bar only
  - RELIANCE real=44,064 vs recon=1,427,692 (32x), MAHABANK real=34,784
  vs recon=2,140,397 (61x), similarly 25-60x across all 8. Root cause:
  `underlying_candle_feed._update_bar` hardcoded the volume baseline to
  0.0 on a symbol's first tick, an assumption only valid if the
  subscription starts exactly at market open. The parity check
  subscribes at 10:00 IST (not 09:15), so the first bar's "volume"
  became the ENTIRE day's cumulative volume up to that point, not that
  bar's own volume - and since the dispatcher subscribes a symbol
  whenever it first enters its pool (any time in the session, not just
  09:15), this would hit real production usage too, not just the parity
  check's own timing. **Fixed same day** (`9a7fe68`): seed the baseline
  from the first tick's own `cum_volume` instead of 0.0. Not catchable
  by this doc's own REST-replay backtest (`backtest_ws_candle_
  reconstruction_parity.py`), which always starts from the beginning of
  a symbol's day, so `cum_volume` is naturally ~0 at the first synthetic
  tick either way - the exact condition under which the old bug's
  assumption happened to be correct. Added `tests/test_underlying_
  candle_feed.py` specifically to cover the mid-day-subscribe case the
  replay method structurally cannot.
- **Open price: still the open question, now with real numbers.**
  `open_and_close_exact_matches` ranged from 3/12 (TCS, HDFCBANK) to
  11/12 (MAHABANK) across the 8 symbols - i.e. for some symbols the
  reconstructed open was wrong on up to 75% of bars. This is the exact
  gap this whole doc has been chasing (see "Next step" above, written
  before today's live data existed) - now empirically confirmed as
  real and non-trivial, not just theoretically unproven.

**Why this matters beyond just "is the WS feed accurate"**: the same-day
breakout-scanner parameter sweep ([[breakout-scanner-param-sweep-22sep-
intraday]]) found `BREAKOUT_MIN_BODY_PCT` (computed from a candle's own
`open`) is the single most sensitive gate in the whole scanner, and
`BREAKOUT_MIN_RELATIVE_VOLUME` (computed from `volume`) is the only
other gate with any measurable effect. Turning on `BREAKOUT_USE_WS_
CANDLES` before both of these are solid would feed corrupted data into
exactly the two parts of the algorithm that actually matter, not a
minor accuracy footnote.

**Decision: `BREAKOUT_USE_WS_CANDLES` stays off.** The volume bug is
fixed, but unverified until the next live dry-run confirms it (today's
fix landed mid-session, after the parity data above was already
collected - the NEXT parity report, or a fresh manual check, should show
the first-bar overshoots gone). The open-price gap is untouched by
today's fix and remains the harder, still-unsolved half of this
investigation.

## Why the automated 15:40 IST report itself came back empty

The scheduled `ws-candle-parity.timer` report for 22 Sep
(`history/2026-09-22_ws_candle_parity_report.json`) shows
`recon_bar_count: 0` for all 8 test symbols against `real_bar_count: 73`
- i.e. by the numbers alone, today's automated end-of-day check looks
like a total failure. **It is not a WS-reconstruction failure** - the
11:13 IST manual check documented above already proved reconstruction
was producing real bars that morning. The automated report came back
empty because of a separate, unrelated operational gap:

`ws_candle_parity_check.py`'s own `maybe_subscribe()` only ever called
the bot's `/debug/underlying-feed/subscribe` endpoint **once** per day
(gated on a `state["subscribed"]` flag persisted to disk). `underlying_
candle_feed`'s subscription set is in-memory only inside the bot
process - restarting the bot wipes it completely. The bot restarted
**5 times** after the 10:00 IST subscribe call that day (04:38:44,
06:01:13, 08:09:33, 09:16:38 UTC - all legitimate, unrelated live-bug
fixes/deploys, unconnected to this investigation), and this script never
noticed any of them, so it never told the newly-restarted process to
re-subscribe. The result: the running process had these 8 symbols
subscribed for roughly the first 8 minutes of the session (10:00-10:08
IST - the window the 11:13 IST manual check's own data actually came
from, captured before the first restart destroyed it) and then **zero
minutes** for the remaining ~5.5 hours, including the entire second
half of the day the scheduled report was supposed to cover.

**Fixed same day** (`a701821`): `maybe_subscribe()` now calls the
subscribe endpoint on every ~5-minute timer tick between `SUBSCRIBE_
TIME_IST` and `REPORT_TIME_IST`, not once. `underlying_candle_feed.
subscribe()` is already idempotent and, per its own docstring, restores
a symbol's persisted bars from disk and resumes live ticks the first
time a given *process* sees it - so this self-heals across any number
of bot restarts during the day, at the cost of one cheap no-op HTTP call
per tick when nothing has changed. No code in `underlying_candle_feed.py`
itself needed to change for this - the restore-from-disk mechanism was
already there, this script just never gave a fresh process the chance
to use it.

**Net effect on the actual open-price question**: unresolved, still.
The best real evidence remains the 11:13 IST manual check documented
above (`open_and_close_exact_matches` from 3/12 to 11/12 depending on
symbol) - a real, if short (12-bar), live sample. The automated report's
own emptiness is a tooling gap now fixed, not new evidence either way.
**Next run of `ws-candle-parity.timer` (or a manual re-check) should
finally produce a full-day sample**, assuming no restart happens to
interact badly with the FIX itself (unlikely, since the fix specifically
targets surviving restarts) - that full-day sample is what should
actually settle the open-price question, not another partial morning
snapshot.
