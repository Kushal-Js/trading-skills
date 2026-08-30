# Exit mechanics: what actually determines how fast and at what price a position closes

Applies to: DhanBoy's Options package (`Options/trading_engine.py`,
`Options/dhan_client.py`), and by extension Futures (same shared
`dhan_wrapper`/cache). Verified live against real 28 Aug 2026 trades and via
mocked unit tests, not just read from code.

## There are two independent exit-check paths, not one

1. **Event-driven (`on_price_tick`)** — fires the instant a new price arrives
   over the WebSocket market feed. Acts directly on the price it's handed;
   it does **not** consult the LTP cache or any staleness logic. This is the
   fast path, and for any option that's actually printing trades, it's what
   actually closes the position.
2. **Poll loop (`_check_one_position`, every `MONITOR_INTERVAL_SECONDS`)** —
   the fallback/heartbeat. Its own LTP lookup (`_get_ltp`) prefers the
   WebSocket-cached price and only falls back to a REST call if there's no
   cached tick at all, or (since 27 Aug 2026) if the cached tick is older
   than `LTP_STALE_AFTER_SECONDS`.

**Consequence:** lowering `MONITOR_INTERVAL_SECONDS` mostly does not make
exits faster for actively-ticking positions — they're already reacting
instantly via path 1. It only tightens the fallback heartbeat for the case
where path 1 isn't producing new ticks (see the feed-stall case below).

## MAX_LOSS_HIT / PROFIT_PROTECTION_HIT can overshoot their rupee cap — and it's usually not fixable

The rupee cap (`MAX_LOSS_PER_TRADE_RS`, `PROFIT_PROTECTION_THRESHOLD_RS`) is
a **trigger threshold checked against whatever price arrives**, not a
guaranteed fill price. Two distinct failure modes can cause the realized
P&L to blow past the intended cap:

1. **Genuine liquidity gap (not fixable by any polling/staleness setting).**
   If the real market has zero trades print between price A (above the
   threshold) and price B (below it), no code can trigger an exit at a price
   that never traded — the best any system can do is react to price B the
   instant it appears. Confirmed with real Dhan 1-minute option data on
   SAGILITY 29 SEP 46 CALL, 28 Aug 2026: entry 2.29, cap threshold 2.19,
   but the market went 2.25 (04:38 close) → 2.14 (04:39 open) with **no
   trade printing in between**. The bot's exit fired on literally the first
   tick past the gap (matching that minute's open, not its worse low) — it
   could not have done better. Realized loss: −Rs.1,800 against a
   Rs.1,200 cap. Verified by replaying the exact tick sequence through the
   real `on_price_tick` code with the staleness fix both on and off —
   **identical result either way**. This is a real, unpreventable cost of
   trading thin, low-premium, high-quantity-per-lot options.
2. **Stale cache during a genuine feed stall (fixable, and now fixed).**
   Before 27 Aug 2026, `get_cached_option_ltp` had no staleness check — once
   any tick arrived it was trusted forever, so if the WebSocket feed
   genuinely stopped delivering ticks for an option (not just "few real
   trades," but zero data), the poll loop's fallback would keep reading an
   arbitrarily old price and never notice a real move. Fixed via
   `LTP_STALE_AFTER_SECONDS` (default 5s): `get_cached_option_ltp` now
   treats a tick older than this as a miss, forcing a REST refetch, which
   re-primes the cache (`note_rest_ltp`) so a persistently-quiet option
   isn't hammered with a REST call on every single poll (throttled to
   roughly once per `LTP_STALE_AFTER_SECONDS` instead of once per poll —
   matters for Dhan's undocumented REST rate limit once more than one
   position goes stale at once).

**How to tell which one happened, after the fact:** pull real 1-minute
option OHLC for the exact entry/exit window. If the exit price matches a
candle's open (or the candle right after the threshold-crossing candle) and
the prior candle's low/close was already past the cap on the wrong side with
no intermediate print — it's a gap, not a monitoring bug. If the price was
sitting well past the cap for an extended period (minutes) with the exit
only firing much later, that's a genuine stale-cache/feed-stall case.

## Data-quality caveat: thin-option minute candles can disagree with real fills

For a very thin contract (SAGILITY's option, 28 Aug 2026), the 1-minute OHLC
Dhan returns for the *option itself* did not match the bot's own real order
fill prices to better than ~3 paise in some minutes (e.g. entry recorded as
2.19 in the fill log, but that minute's candle showed a flat 2.22 the whole
time). Likely cause: real fills reflect bid/ask quotes, while minute-candle
aggregation reflects only last-traded prints, and a thin contract can have
very few of those per minute. **Don't trust option-level minute candles for
sub-paise-precision reconstruction on thin names** — they're fine for
underlying-stock analysis (much more liquid) but not as a ground truth for
an illiquid option's exact intra-minute price path.

## Supertrend exit: real observed detection lag is much smaller than the theoretical worst case

`SUPERTREND_REFRESH_SECONDS` caps how often the underlying's 5-min candles
are re-fetched (independent of `MONITOR_INTERVAL_SECONDS` — the poll loop
calls `refresh_supertrend_signal` every tick, but it's a no-op unless the
cache has aged past this value). Measured against real SAGILITY 5-min
Supertrend flips and real SUPERTREND_EXIT trade timestamps on 28 Aug 2026
(under the *old* `SUPERTREND_REFRESH_SECONDS=60`): observed lag ranged 4–27s
across 9 trades, averaging 15.7s — far under the 60s theoretical ceiling,
because the 5s poll cadence in effect at the time already kept the cache
reasonably warm. Lowering the refresh ceiling to 15s (27 Aug 2026) tightens
the worst case but, based on the same data, likely only shaves a handful of
seconds off the already-fast outliers — not a dramatic change. Don't expect
config changes here to produce large P&L swings; the mechanism was already
fairly fast in practice.
