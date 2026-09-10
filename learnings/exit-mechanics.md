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

## PROFIT_PROTECTION_HIT: a give-back buffer is net-negative across the real trade population

Investigated 10 Sep 2026 after a real Luxury OIL trade (entered 09:16:15
IST, right into the opening spike, exited 13s later on PROFIT_PROTECTION_HIT
@ 15.85 for +Rs.2,030) where OIL 505 CE then ran 14.40 -> 16.75 -> 18.9 ->
21.2 within 4 minutes. The PP rule has zero drawdown tolerance: once peak
unrealised profit crosses the Rs.1,500/1,000 threshold, it exits on the
FIRST tick below the running peak (`ltp < highest_price`), even a 10-paise
sub-minute wiggle.

Built `PROFIT_PROTECTION_GIVEBACK_PCT` (Options + Luxury, env, default 0.0 =
bit-identical to old) - PP fires only once `ltp < highest_price * (1 - pct)`.
Backtested buffers 0/3/5/8/10% two ways:

1. **Against only the 23 real PROFIT_PROTECTION_HIT trades** (Aug 31-Sep 10):
   looked positive, ~+Rs.7-8k, because that subset is self-selected to
   cases where holding longer tends to help.
2. **Against all 87 real Options+Luxury closed trades**, using the
   drift-immune metric `sum(replay[buffer] - replay[0])` (same 1-min engine
   + data on both sides, so replay-vs-reality error cancels): **every buffer
   level LOSES** - 3% -Rs.5,975, 5% -Rs.7,109, 8% -Rs.1,234, 10% -Rs.4,839.

**Mechanism:** the buffer converts ~5 fast-momentum trades into TARGET hits
(+Rs.2-5k each: MPHASIS, MAZDOCK, both FORCEMOT, the 10-Sep OIL) but gives
back on ~18 trades where the option peaked then reversed (CAMS 1402->165,
GVT&D 1494->-125, BLUESTARCO, DIVISLAB, PERSISTENT, MAHABANK, several
supertrend trades). Option premiums mean-revert hard (theta + OTM decay),
so the reversal give-backs outweigh the continuation captures.

**Lesson:** the zero-tolerance PP is doing its job - locking small/medium
gains before they evaporate. Don't add a blanket give-back buffer. If the
opening-spike miss (OIL-style) needs addressing, use a TARGETED rule
instead (e.g. "within X% of target -> let it run to target, skip PP" or
"don't arm PP in the first 2-3 min after entry") that leaves the
reversal-heavy midday trades alone. The feature/knob exists in code
(commit 0645c1b in traderBoy, left at 0.0) for future experiments.

Methodology caveat: the 1-min replay only reproduced 26/87 real exits
within tolerance - real PP exits are largely sub-minute events invisible to
1-min candles, and 1-min Supertrend recompute + entry-candle alignment is
rough. This is why the pure `replay[b] - replay[0]` delta (not
replay-vs-real) is the trustworthy number here.

## Broker SL-L vs bot tick-driven MAX_LOSS exit: bot wins on healthy days

Tracked live across a full trading day (10 Sep 2026, `BROKER_STOP_LOSS_
ENABLED=false` for Options - so pure tick-driven - with an SL-L equivalent
computed for every real stop-out). All **8** MAX_LOSS_HIT/STOP_LOSS_HIT
trades that day: the bot's tick-driven SELL got a **better price than a
broker SL-L limit would have, on every single one**. SL-L total vs bot:
**−₹4,175** (SL-L worse), i.e. the 3% `BROKER_STOP_LOSS_LIMIT_BUFFER_PCT`
slippage consistently costs more than the poll/tick lag gives up.

Matches the earlier `backtest_options_chartink_laxmi_02_sll.py` pessimistic
result (−₹5,700 / −25% on that dataset's stop trades).

The SL-L's only edge is disaster insurance - feed down, process wedged,
premium gaps through the level between ticks. But 10 Sep's ICICIPRULI
incident ([[2026-09-10-icicipruli-unmonitorable-position]]) shows that even
when the feed *does* die, the SL-L wouldn't fire either - it needs the same
live LTP the monitor lost.

**Verdict: keep `BROKER_STOP_LOSS_ENABLED=false` for Options.** It's a net
cost when monitoring is healthy and doesn't cover the case where it isn't.
Re-evaluate only if a real "bot monitoring was down and a position ran
away" incident happens.

## ENABLE_TARGET_EXIT: the fixed +TARGET_PCT exit is now a per-package toggle

Added 10 Sep 2026 (traderBoy commit `fcbf1a5`). `config.ENABLE_TARGET_EXIT`
(default `true`, env: `ENABLE_TARGET_EXIT` / `LUXURY_` / `FUTURES_` prefix)
gates the `if ltp >= position.target_price: return "TARGET_HIT"` branch in
`_exit_reason_for`. Present in all three option packages so the
Options/Futures engine copies stay byte-identical.

**Deployed: `FUTURES_ENABLE_TARGET_EXIT=false`** (user request: "disable
TARGET_HIT for Futures, rest to remain same"). Options and Luxury keep it
`true`.

What "off" does: a winning Futures position is never closed just for
touching `entry * (1 + TARGET_PCT)` (25%). It rides on to whatever fires
next in `_exit_reason_for`'s order - `PROFIT_PROTECTION_HIT` (peak >
₹1,500/1,000 then any dip past the give-back buffer), the trailing +
dynamic + hard SL, `SUPERTREND_EXIT`, the liquidity guard, or the 15:15 EOD
square-off. `target_price` is still computed and stored (reconciliation /
`/positions` display use it) - it's just no longer an exit trigger. The
loss side (`MAX_LOSS_HIT`, all SLs) and every entry gate are untouched.

Net effect to watch for as real Futures data accumulates: with the fixed
target gone, **`PROFIT_PROTECTION_HIT` becomes Futures' primary profit-
taking exit**. The zero-drawdown-tolerance PP finding above (give-back
buffer is net-negative) now matters more for Futures than for Options -
worth re-checking whether the ₹1,500/1,000 PP threshold is the right level
once there's a Futures trade population to measure. This is effectively a
live test of "let winners run vs. lock the +25%" on the second independent
Options copy while the original keeps the fixed target.

## EMA_CROSS_EXIT: a second trend-reversal exit alongside Supertrend (Futures)

Added 10 Sep 2026 (traderBoy commit `9591411`), **deployed
`FUTURES_ENABLE_EMA_CROSS_EXIT=true`**; Options and Luxury keep it off
(default). Fires `EMA_CROSS_EXIT` when the fast EMA of the underlying's
5-min close *crosses* the slow EMA against the position's direction -
`EMA_CROSS_FAST_PERIOD=9` below `EMA_CROSS_SLOW_PERIOD=12` for a CE, the
reverse for a PE - on a fully-closed candle **later than the entry
candle** (same entry-candle skip as the Supertrend exit, for the same
reason: don't cut a trade flat on the bar it opened on). Checked right
after `SUPERTREND_EXIT`, before the liquidity guard.

Mechanics (`dhan_client.refresh_ema_cross_signal`, shared cache, poll-loop
refresh only - tick path reads it synchronously, exactly like the
Supertrend signal):

- EMA is SMA-seeded (`_compute_ema`), computed on a continuous
  multi-session 5-min series (see the "Continuous intraday history"
  section below), still-forming candle dropped.
- It's a **crossover edge, not a standing state**: the cache stores
  `crossed = (sign of fast-slow flipped between the last two closed
  candles)`. `EMA_CROSS_EXIT` needs `crossed and (fast<slow for a CE)` -
  so entering a CE while EMA9 is *already* below EMA12 does NOT trigger it
  (no flip on the latest bar); only an actual cross after entry does. The
  series runs continuously across the overnight gap, so a flip on the
  day's first bar (vs the prior session's last) IS a crossover and fires
  - the entry-candle skip is the only thing protecting a fresh intraday
  entry.
- **The only remaining wait** is for the 5-min candle the cross happens
  in to actually close - unavoidable for "EMA of the 5-min *close*". Once
  it closes, the exit fires on the next monitor tick (~2s) / signal
  refresh (~15s cap). If sub-5-min reaction is ever wanted, that needs an
  intra-candle variant (act on the forming candle's live price vs the
  EMAs) - deliberately not built; it trades noise for speed and
  contradicts "of the 5-min close".

Not backtested before enabling - it's a plain trend-follower exit on the
same 5-min grid as the (backtested, kept) Supertrend exit, and it's on
the Futures copy specifically so it can be measured against the Options
original without touching that. Watch: does it exit good Futures trades
early (same failure mode the Supertrend entry-candle skip was added for),
and how often does it beat Supertrend to the exit vs just duplicate it.

## Continuous intraday history — every indicator, every strategy (no daily warm-up lag)

10 Sep 2026 (traderBoy commits `75a60d3` + `d1573e2`), user request: "no
lag across ALL strategies, calculations run continuously across sessions,
not with a fresh day start like a charting platform".

**The problem it fixed:** every intraday indicator used to fetch
`intraday_minute_data` with `from_date=today, to_date=today`. A recursive
indicator (Supertrend, EMA, RSI, ATR) seeded from *today's* candles only
is either uncomputable or unreliable until enough bars have closed:

| Indicator | period | first value | reliable |
|---|---|---|---|
| 5-min Supertrend (`SUPERTREND_PERIOD=10`) | 10 | ~10:10 IST | later still — bands need history, "reads bearish on ~everything" early |
| EMA(9/12) 5-min | 12 | ~10:20 IST | ~10:20 |

i.e. the entire 09:15–11:00 morning entry window was largely unprotected
by these exits.

**The fix:** one chokepoint — `dhan_wrapper.fetch_continuous_intraday()` —
pulls `INTRADAY_CONTINUOUS_LOOKBACK_DAYS` (7 calendar days ≈ 5 trading
sessions) of candles *through* today, wrapped in `_retry`. Every fetch
site routes through it: `refresh_supertrend_signal`,
`refresh_ema_cross_signal`, `refresh_liquidity_signal` (Options/Futures/
Luxury), Swing's `_fetch_supertrend_state_once`, and the IndexScalping /
CopperOptions / K01 paper engines. Result: bands/EMAs fully warm from the
**session's first bar**, exactly like a charting platform's continuous
intraday chart. `tests/test_continuous_intraday.py` guards against a
regression reintroducing a today-only fetch.

**Consequences worth knowing:**
- Dhan `intraday_minute_data` returns a genuinely continuous
  multi-session series (verified: 1-min 7d → ~2200 bars / ~5 sessions;
  5-min 7d → ~430 bars / 6 sessions). No synthetic overnight bars.
- The 5-min-multi-day endpoint *intermittently* returns
  `status=failure` under rapid back-to-back calls (rate limit) — `_retry`
  + the 15s signal cache absorb it in production; a failed refresh just
  means "no signal this cycle", same degradation the today-only fetch had.
- Crossover edges (`EMA_CROSS_EXIT`, Swing's `SupertrendState.crossed_*`)
  now span the overnight gap: a flip between the prior session's last bar
  and today's first IS a crossover and will fire. This is intentional
  ("like a charting platform") and correct for overnight-carried NRML
  positions — the trend genuinely flipped. The per-position entry-candle
  skip still prevents a *fresh* intraday entry being whipsawed on its own
  entry bar.
