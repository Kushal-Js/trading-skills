# Token invalidated, then Dhan Data APIs refused - weekly watchlist job could not run (1-2 Oct 2026)

**Strategies:** all (REAL = Unified Momentum + Scalper). **Market:** closed (evening + Gandhi Jayanti holiday).
**Real impact:** none so far - no positions were open. Risk: from Mon 5 Oct 09:15 every strategy is blind (no
candles, no prices) until the Dhan Data API plan is renewed.

## What happened (IST)
- 1 Oct 22:47 - every Dhan call starts returning **DH-906 "Invalid Token"**. The token was valid until 2 Oct 06:48,
  so it did not expire; something invalidated it (cause unknown - no local Tradehull login after 18:40). The bot
  never re-logs in on its own, so it ran blind: Unified Momentum's supervisor could not read the order book (86
  errors), the market-data WebSocket was refused (HTTP 429) and reconnected every 60 s.
- 2 Oct 00:00 - the first scheduled weekly watchlist refresh: 442 MB free (< 450) -> safe-restarted the bot FIRST.
  The bot's cached-token check failed (DH-906) -> PIN+TOTP login -> **"Invalid TOTP" four times**; startup exited
  twice (status 3), systemd `Restart=always` brought it up at 00:01:18 with a new token.
- From 00:01 - trading APIs fine (funds Rs 1,07,751.72, orders), but **every data call: DH-902 / HTTP 451 "User has
  not subscribed to Data APIs"** -> the Data API plan most likely lapsed.
- 00:01:34 - the job scored 0 of 213 symbols, aborted (lists untouched, correct) **but exited 0**: systemd showed
  success and nothing would retry for a week. Meanwhile the Scalper's candle setup was re-started every 1 s monitor
  tick and forced both REST fetches -> ~2 rejected calls/s (~1,140 DH-902 per 10 min).

## Fix (traderBoy `2988c40`, deployed 2 Oct 00:23 IST, flat, safe_restart clean)
- Weekly job: one NIFTY daily-history probe right after login, before the memory-guard restart or any file change;
  a data failure (probe, login or the scored-count net) writes `data/weekly_watchlist_refresh_pending.json` and
  exits 75 (systemd shows "failed"). New droplet timer `dhanboy-weekly-watchlist-retry` re-runs it with `--retry`
  hourly, only 00:20-06:20 IST on any day and 07:20-23:20 IST at weekends, only while the pending file exists;
  a completed run deletes it. Verified on the droplet: retry 00:23:56 -> DH-902 -> exit 75, bot NOT restarted.
- Scalper: a failed candle setup waits 60 s (REST_RETRY_SECONDS) before the next attempt -> 2 calls/min.

## Lessons
- **A job that aborts must not report success.** Exit non-zero and leave a retry marker, or the schedule silently
  skips a week.
- **Probe the dependency before doing anything disruptive.** The job restarted the bot (and triggered a fragile
  re-login) for a run that could never succeed.
- **Every retry loop needs its own backoff** - a "once at a time" guard is not a rate limit when each attempt fails
  in milliseconds.
- **Read the error codes per time window before diagnosing.** "DH-902 since 22:47" was really DH-906 (token) until
  00:00 and DH-902 (data plan) after - two different problems with two different fixes.

## Still open (user's call)
- The bot never re-authenticates when its token goes invalid mid-run (in market hours UM could not read orders or
  exit) - needs an in-process re-login on DH-901/DH-906.
- Startup login gives up after 2 TOTP attempts and exits; systemd restarts paper over it. More in-process retries
  with backoff would avoid the crash-loop (and any TOTP lockout risk).
