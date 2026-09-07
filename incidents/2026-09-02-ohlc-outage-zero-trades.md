# Incident: Dhan `get_ohlc_data` outage — zero real trades bot-wide for the full trading day, 2 Sep 2026

## What happened

**Wednesday 2 Sep 2026 had ZERO closed real trades across every package**
(Options, Futures, Luxury, Swing) — confirmed via `history/2026-09-
02_real_trades.log` simply not existing that day. User asked "why did
Swing not trigger on Wednesday?" - the investigation found the actual
cause was much bigger than Swing alone.

## Root cause: Dhan's `get_ohlc_data` (quote OHLC) call failed continuously, ALL DAY

`journalctl -u dhanboy.service` for 2 Sep 2026 shows **78,127** occurrences
of `Exception at calling OHLC as {'status': 'failure', 'remarks':
{'error_code': None, 'error_type': None, 'error_message': None}, 'data':
''}` — an entirely EMPTY failure envelope (no Dhan error code at all,
unlike a normal documented rejection), from the very first tick after
market open (confirmed failing by 03:43 UTC / 09:13 IST, right at open)
through to 23:59:53 IST, essentially the entire day. Zero occurrences in
the same window on 3 Sep (the very next day) - this was a genuinely
single-day, well-bounded event, not a standing bug.

**This one call is shared infrastructure two completely different things
depend on**, which is why it took down BOTH:

1. **Swing's own price-confirmation entry gate**
   (`_is_price_confirmed_above_prev_close` → `dhan_wrapper.
   get_today_open_and_prev_close` → `get_ohlc_data`) - every single call
   for every one of the 22 watchlist symbols raised `ValueError: No OHLC
   data returned for {symbol}` (after `_retry`'s own 2 retries also
   failed), so the gate returned `False` for every symbol, every tick,
   all day. Swing's own entry rule short-circuits on this gate FIRST
   (before even checking the 5-min/1-min Supertrend crossover) - so no
   symbol ever got far enough to be evaluated for an actual entry
   signal. This correctly fails SAFE (no entry on bad data), not a bug
   in Swing's own logic - the gate is exactly doing its job.
2. **Options/Luxury's own stock ranking** (`get_day_change_pct`, also
   built on `get_ohlc_data`) - confirmed via `history/2026-09-
   02_webhook_alerts.log`: **150 separate Luxury alerts** all resulted in
   `"status": "no_action", "reason": "could_not_rank_any_stock"` that
   day, and zero `"status": "entered"`/`"processed"` results anywhere in
   the log. Futures shares the same near-identical ranking code path.

**So this was never a Swing-specific issue** - it was a shared Dhan
OHLC-quote outage that silently zeroed out the ENTIRE bot's real-money
trading capability for one full day, across all four packages, via one
common dependency.

## Why nothing alerted on this

Every affected code path fails open/safe by design (Swing's gate returns
`False`/no-entry; ranking returns `could_not_rank_any_stock`/no-action) -
correct behavior for not trading on bad data, but it means a sustained,
day-long outage produces the exact same *symptom* as "the market just
didn't offer a good setup today" from the outside. Nothing distinguishes
"legitimately no good setups" from "the data pipeline was broken all
day" in any dashboard/status endpoint - only `journalctl`'s own raw
volume of one specific exception message revealed it. Worth checking
`journalctl` for `Exception at calling OHLC` (or a periodic alert on this
specific exception's frequency) on any future day that looks
suspiciously trade-free, rather than assuming "quiet market."

## What's NOT affected

`intraday_minute_data`/`historical_daily_data` (used for Supertrend/EMA
computation directly, and confirmed independently in the 2 Sep momentum-
signal backtest work) are a DIFFERENT Dhan call, unaffected by this - the
outage was scoped specifically to the OHLC quote snapshot endpoint
(`get_ltp_data`-family), not the whole Dhan integration. Order placement/
position management for whatever WAS already open that day also wasn't
implicated (no evidence of any stuck/mismanaged position - there simply
were no new entries to manage).

## Open, not yet done

- Not yet checked whether this is a known Dhan-side incident from their
  own status page/support channel for that date - worth checking if this
  recurs.
- No code change made or recommended yet - this reads as a genuine
  external outage that resolved on its own by the next day, not a bug in
  `traderBoy`'s own logic. A possible, not-yet-actioned enhancement: a
  circuit-breaker/longer-backoff after N consecutive `get_ohlc_data`
  failures, purely to cut log volume during a sustained outage like this
  one (78K near-identical ERROR lines in a day) - not a correctness fix,
  since the fail-safe behavior itself was already correct throughout.
