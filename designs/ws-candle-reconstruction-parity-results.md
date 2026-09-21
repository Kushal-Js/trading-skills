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

## What would actually resolve this

- Re-run the parity check with proper call pacing/backoff and a longer
  gap between symbols, ideally well after any live-bot activity window,
  to get all 4 (or more) symbol-days fetched cleanly without rate-limit
  casualties.
- The open-price question can only be truly settled by comparing the
  live `underlying_candle_feed` module's own real WS-tick-derived bars
  against REST candles for the SAME real trading session, once
  subscribed - not a REST-vs-REST-proxy replay. This would need running
  the feed live (still without `BREAKOUT_USE_WS_CANDLES` gating any real
  entry) during a market session and diffing its `get_candles_dict()`
  output against real REST candles fetched after the fact for the same
  symbols/day.
