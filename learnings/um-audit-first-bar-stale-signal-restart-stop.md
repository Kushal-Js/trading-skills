# Unified Momentum pre-launch audit (2 Oct 2026): first bar, stale signals, double stops, coupled loops

Context: Unified Momentum (UM) was deployed for real money on 1 Oct 2026, with Mon 5 Oct its first trading day.
On 2 Oct (a market holiday) the user asked for a full re-test: "the loopholes, the silent bugs, the failed
mechanism, API limits, blockage, latency". The audit read every UM module plus the shared signal, order and data
code. It then checked each finding against the backtest code before changing anything. Fixed in traderBoy
(commit in the journal row for 2 Oct).

## 1. The session's first bar has no "just-closed" bar

Bar-start logic expects `expected = forming_bar_start - 5 min` to be the newest closed bar. At 09:15 that bar is
09:10, which never exists: the newest closed bar is the previous session's 15:25. Two pieces of code did not
allow for this.

- **REST refetch.** `Bollinger/signals._get_intraday_series` refetched the 60-day REST series whenever the cached
  base was older than `newest_closed_start`. It had no floor, so between 09:15 and 09:20 every call refetched.
  Swing's copy already had a 60 s floor.
- **Forced refresh.** The UM and Super Bollinger tick listeners forced a refresh whenever the cached state was not
  on the expected bar, retrying every 3 s per stock. At the open they never stopped.

Result: for five minutes every day the open ran up to about 2 heavy REST calls/s. The account pacing (0.5 s) was
the only limit, and each call was also a full replay. DH-904 never fired because the pacing held, so the cost was
invisible: a full shared REST queue and executor workers asleep in the pacing lock.

Nothing could have been entered in that bar anyway:

- `resting_trigger_hit` refuses a state from a previous day.
- The backtest also skips it: in `candidates_for`, `fts[bi-1].date() != dt.date()` leads to `continue`.

**Fix:**

- The REST base is requested once per bar, then at most every 60 s while still behind. WS bars extend it in the
  meantime.
- The listeners do nothing in the 09:15 bar.
- Fake replay: the first bar went from about 100 REST calls per stock to 5.

**Rule:** any code built around "the bar that just closed" needs an explicit case for the session's first bar, and
any refetch-until-current loop needs a floor.

## 2. A previous day's closed candle can look like a fresh signal at the open

UM engine B (momentum puts) read Swing's state for the last closed candle and entered on a bearish cross. It never
checked the candle's date. Entries stop at 14:30, so a cross on the 15:25 candle was never used that day. At 09:15
the next morning it was still "the last closed candle", and the bot would have bought a put in the first seconds
of the session, at the widest spreads of the day.

The backtest evaluates each candle at its own close and skips closes after 14:30, so a previous day's candle never
trades.

How often: a cache replay of 20 UM-list stocks, 3 Aug to 29 Sep, found 4 stale signals in 820 stock-days that
would have passed the volume and regime checks at the next open. With 15 stocks that is about once every three
weeks.

**Fix:** the signal candle must be from today's session. The first possible signal is the 09:15 candle, at 09:20.

**Rule:** every closed-candle signal needs a same-session check, the same as the resting-trigger rule. A holiday
or weekend in between makes it worse, not better.

## 3. Restart adoption must not add a second stop

Every real entry path places the broker SL-L order first and records the position second, with a write-ahead order
intent around both. If the process dies between the two steps, the restart finds the intent and the filled order.
It then adopted the position and placed a new SL-L. The first one was still resting, so there were now two SELL
stops for one lot. On a fall both trigger, and the second sells a lot the account does not hold.

`safe_restart` refuses to restart while intents are open, so only a crash, OOM or kill can hit this.

**Fix:**

- Before placing a backstop, look for a SELL order already resting on the contract and adopt it.
- If the broker cannot be asked, place nothing. The bot's own max-loss check still manages the position, and the
  restart report flags it for review.

## 4. One engine's exception must not skip another engine's exits

Engine B ran at the end of engine A's monitor tick. If anything earlier in A's tick raised (for example inside A's
own square-off), B was skipped for that cycle, including its 15:15 square-off poll.

**Fix:** engine B has its own loop task.

Before splitting the loops, I checked concurrency safety. Both engines claim the stock in the cross-strategy
registry and re-check the other engine under that claim, so they can never both enter one stock.

## 5. Measure before changing the executor (open item)

Order placement uses `run_in_executor(None, ...)`, which is the same 5-worker pool as every blocking Dhan call. The
history-data pacing floor sleeps inside a worker while holding a lock. At bar starts about 10 loops each queue a
REST refetch, so an order could wait seconds for a worker. I added counters rather than change the executor on a
closed market (rule from the 13 Sep incident):

- `/feed-stats` `market_data_calls_by_minute` and `market_data_pacing_wait_s`;
- an executor-lag probe: a no-op job every second, recorded as `executor_lag_ms_max_by_minute`, with a WARNING at
  2 s or more.

First numbers, from the 2 Oct holiday:

- A restart burst gave 2.5-3.0 s of lag.
- The holiday steady state was about 100 history calls/min, because `is_market_open()` has no holiday
  calendar and every base stays "behind" all day. Lag peaked at 0.2-1.5 s.
- That rate means about 66 refetches at each 5-min bar start and about 98 at 15-min ones on a trading day.
  That is 33-49 s of queue.
- Swing fetches two bases per instrument per interval: a 45-day regime base and a 7-day Supertrend base.

The fix is prepared but off: `HISTORY_EXECUTOR_WORKERS` (default 0 = unchanged) moves the 14 history call
sites to their own "history" pool, so orders, quotes and order status keep the default pool. Turn it on during
market hours with a before/after look at `executor_lag`.

## Also checked, no change needed

- Supervisor hedge open, exit and disaster brake.
- The pricing layer's shared reads.
- The 1-hour filter's 09:15-anchored buckets, which match the backtest.
- Engine A/B cross claims.
- `SWING_ENTRY_TIMING=bar_close`, so engine B has no tick trigger.
- The droplet clock, which is NTP-synced.
- Holiday handling (2 Oct): `_symbol_market_open` closed every entry path, and no UM event was written.
- Partial fills are only logged, which is harmless at 1 lot (NSE options fill in whole lots). This must be handled
  before `quantity_lots` > 1.
