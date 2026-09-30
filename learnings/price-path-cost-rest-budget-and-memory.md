# Price reads: what each one costs, why the REST budget runs out, and memory headroom

Audit of the live bot on the night of 30 Sep → 1 Oct 2026 (traderBoy `0c5ded5` deployed). Scope: the shared
Dhan connection that every package uses (`Options/dhan_client.py`), Super Bollinger's price and hedge paths,
and the droplet (1 vCPU, 961 MB RAM, 1 GB swap). Follow-up to
[`incidents/2026-09-30-paper-positions-saturate-rest-ltp-quote-budget.md`](../incidents/2026-09-30-paper-positions-saturate-rest-ltp-quote-budget.md).

## 1. The real book alone runs out Dhan's price budget

The 30 Sep morning fix (`da4942c`, 10:12 IST) stopped paper positions using REST prices. The afternoon was still
worse than the morning:

| 30 Sep (IST) | failed REST LTP calls (`Exception at calling ltp`) |
|---|---|
| 09:15–10:30 | ~1,400 |
| 10:30–13:30 | ~835 |
| 13:30–15:30 | ~2,900 |

After 10:12, most of the failures were on the REAL contracts. Retry warnings by contract: MOTILALOFS 1000 CE 1,023,
APLAPOLLO 2240 CE 798, GLENMARK 2380 CE 745, APLAPOLLO 2220 CE 234, APLAPOLLO 2200 PE (hedge) 119. Super Bollinger
logged 75 checks where a real position had no price at all after 3 attempts (`[SuperBollinger] could not fetch LTP`).
65 of them came between 13:30 and 15:30, when 3 real CEs and 2 real hedges were open.

**Why:** each real position is read 2–3 times per 2-second cycle:

- the monitor loop's `_check_real`;
- the supervisor's `check_ce` via `_ltp`;
- the disaster brake's `_day_real_pnl` via `_ltp`, every supervisor tick.

A thin option's WS tick is usually older than `LTP_STALE_AFTER_SECONDS` (5 s). `/feed-stats` on the evening of
30 Sep showed 4,195 stale cache reads vs 359 fresh ones, so almost every read falls back to REST. The monitor loop and
the supervisor can both miss the cache at the same moment and both call REST. `ltp_rest_fallback_semaphore` is 2,
and Dhan allows about 1 request/s, so one of the two calls fails.

Five thin real contracts × 2–3 reads per 2 s is well over 1/s, whatever paper is doing.

**REST LTP adds nothing while the WS feed is alive.** LTP only changes when the contract trades, and the WS pushes
every trade it sees. For a contract that has not traded, REST returns the same last price the cache already has. What
a quiet contract needs is the order book (bid/ask), not a fresher LTP. The fallback only helps when the feed itself is
down. Staleness is judged per contract, so the code cannot tell "no trade" from "feed dead".

## 2. The quote and LTP calls share one budget but are paced separately

`pricing.py` (live from 30 Sep 16:01; 1 Oct is its first market day) adds Dhan quote calls (`get_option_quote`,
order book):

- the order-book fallback in `live_price` for real positions;
- `mark_for_loss` near the hedge trigger;
- `price_for_entry`;
- every order retry (`retry_unfilled_buy`).

Quote calls are spaced 1.1 s apart among themselves (`_quote_lock`). REST LTP calls are not paced at all, and neither
kind waits for the other. The order-book fallback runs exactly when LTP calls are failing, which is when the budget is
already used up. So expect the fallback to fail most in the minutes it matters most.

Note: the 30 Sep LTP failures were `status: failure` with every `remarks` field `None` and `data: ''`, not DH-904. The
most likely cause is the quote API's rate limit reported through dhanhq's generic failure shape. This was inferred
from the pattern, not confirmed with Dhan: failures cluster when many contracts are read at once (09:11 UTC:
10 contracts, 37 failures in one minute) and hit liquid names too. The earlier 1/s observation is in
`SuperBollinger/pricing.py`'s docstring.

## 3. Each price read costs CPU, measured on the droplet

Measured 1 Oct 00:20 IST on the droplet itself, from the local instrument-master CSV, with no Dhan calls:

| operation | cost | where it runs |
|---|---|---|
| `_instrument_meta(trading_symbol)`: full pandas scan of 204,649 rows, **not memoized** | **~69 ms** | every `get_cached_option_ltp`, `get_recent_cached_option_ltp`, `note_rest_ltp`, `get_option_quote`, subscribe/unsubscribe, order helpers |
| lookup by security id (`_instrument_meta_by_security_id`) | ~99 ms | reconciliation, orphan sweep |
| Tradehull `get_ltp_data`: `instrument_df.copy()` | ~38 ms + about 27 MB allocated per call | every REST LTP call |
| Tradehull `get_ltp_data`: then 1–2 scans, then a hardcoded `time.sleep(0.4)` | ≥ 0.4 s of a worker thread | every REST LTP call |
| Tradehull `order_placement` | same `instrument_df.copy()` + scans | every order |

The instrument master holds **132 MB** in memory (deep size, 17 columns).

Consequences:

- **A read of the WS cache is not cheap.** Most of its cost is the 69 ms symbol lookup, not the dict read. It runs in
  the default executor (`EXECUTOR_MAX_WORKERS=5`).
- The comparisons run on object-dtype columns and hold the GIL. While one runs, the WS tick thread and the event loop
  wait.
- On 30 Sep the bot used about 48–56% of the single core during market hours (systemd CPU totals per run: 46.5 min in
  83 min, 70.5 min in 147 min).
- A cycle with a handful of real, hedge and paper positions does roughly 10–25 lookups, which is about 0.7–1.7 s of CPU
  per 2 s. This is an estimate from the code paths; the live process was not profiled.

**Fix:** memoize `_instrument_meta` by `(trading_symbol, expected_exchange)`. Clear the cache when `instrument_df`
changes; the process restarts daily at 08:00 anyway. This is the cheapest large latency win available. Avoid
Tradehull's `get_ltp_data` on hot paths: call `Dhan.ticker_data` with the security id you already have, which skips
the copy, the scans and the 0.4 s sleep.

## 4. The worker pool can run out of threads

The default executor has 5 threads. The Super Bollinger additions that hold a thread while waiting:

- `get_option_quote` sleeps AND makes the HTTP call inside a `threading.Lock`. Each queued caller holds a worker; with
  3 attempts and a 12 s HTTP timeout, the worst case is tens of seconds.
- `wait_for_order_result` uses `time.sleep` between polls: 6 × 1 s for an entry, `entry_retry_wait_seconds` for a
  retry, plus `order_safety.cancel_unfilled`'s wait.
- Tradehull's 0.4 s sleep on every REST LTP call.
- History fetches paced by `_throttle_market_data_call`'s sleep.

A bar boundary with 2 tick entries, a hedge, and a quote-based re-price can park every worker. Every package's exit
checks then wait, because the plain cache reads also go through that pool.

Fixes:

- make cache reads direct, not `run_in_executor`, once `_instrument_meta` is memoized;
- move the quote pacing and the order polling to `asyncio.sleep` between REST calls, as `get_option_ltp_async`
  already does.

Do not raise `EXECUTOR_MAX_WORKERS` on a closed market; see
[`incidents/2026-09-13-executor-sizing-ws-storm-on-closed-market.md`](../incidents/2026-09-13-executor-sizing-ws-storm-on-closed-market.md).

## 5. Memory headroom

- Bot RSS is 316–337 MB right after a restart, with nothing open.
- The one full-day run on record (29 Sep, 08:00 IST to past the MCX close) peaked at **420.6 MB and pushed 92.6 MB to
  swap**. Every 30 Sep run was cut short by restarts, so none of them shows a full-day peak.
- The droplet had 327 MB available with the bot idle. Other resident processes: `fwupd` 40 MB, the `uv run` wrapper
  32 MB (the parent of the real python process), `multipathd` 27 MB, `watchdog.py` 35 MB, journald 32 MB.
- Swap on this box means pauses of hundreds of milliseconds for every thread.
- Of the 30 Sep additions, no new code path grows memory without bound. Small per-contract/per-symbol dicts
  (`pricing._quotes`, supervisor `_series_checked`) are cleared by the daily restart.
- The big consumers are the 132 MB instrument master and the ~27 MB copies Tradehull makes of it on each LTP or order
  call. Two concurrent calls mean two copies.
- Highest risk: the Friday 00:00 IST watchlist job (`weekly_watchlist_refresh.py`, first run 2 Oct). It starts a
  second Python process that loads its own instrument master plus daily/5-min series for the F&O universe, while the
  bot is at its end-of-day size.

## 6. Checked and found safe (Super Bollinger races)

- **Duplicate entries.** The tick path (WS thread) and the 5 s scan can both start on the same symbol, because
  `_entry_inflight`'s check-then-add is not atomic across threads. Only one can win: `PROFILE.consumed` is checked and
  set with no `await` in between, and real entries also go through `cross_strategy_registry.try_claim` and
  `position_store.reserve_symbol`. Worst case is a wasted evaluation.
- **Capacity.** `open_count()` (real + paper) is checked before awaits, so concurrent entries can overshoot the
  combined count. Real positions are capped atomically in `reserve_symbol`. Paper index positions (NIFTY/BANKNIFTY,
  paper since 30 Sep 20:19 IST) still count toward the 5 slots, so they can block a real stock entry. That is by
  design, but worth knowing.
- **State files.** `live_state`, `best_price_memory` and `position_memory` are written only from the event loop, one
  file per strategy, with tmp + `os.replace`. No write races.
- **Orphan sweep.** It skips orders that an in-flight intent owns, skips everything while any intent has no order id
  yet, and ignores orders younger than 20 s.

## 7. WS candles vs REST candles (price lag on the trigger side)

From `research_results/2026-09-30_ws_vs_rest_candle_gap.txt` (23–30 Sep, 27 symbols, 4,178 bars):

- the live bar's high is below the REST high in 31% of bars, by more than 0.05% in 5.4% of them;
- 142 REST bars had no live bar at all;
- of 23 BULLISH trigger touches, 20 were seen in the same bar, 1 was 1–3 bars late, 1 was missed and 1 had no live
  bar.

See [`ws-candle-vs-rest-candle-trigger-miss.md`](ws-candle-vs-rest-candle-trigger-miss.md).
