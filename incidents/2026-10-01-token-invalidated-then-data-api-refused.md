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

## Follow-up, 2 Oct 06:20-07:30 IST
- **The retry timer itself knocked the bot off its session.** The 06:20 IST retry ran after 05:30 IST = UTC
  midnight. Tradehull names its token file by the UTC date, found no file for "today", logged in with PIN+TOTP and
  minted a new token - Dhan allows one active token per account, so the running bot got DH-906 from 06:20 until
  the 08:00 morning refresh (which reuses the job's token). Retry window narrowed to 00:00-05:25 IST any day +
  08:15-23:59 IST at weekends (timer changed on the droplet ~07:15; script check in `0bc237e`).
- **Both login weaknesses fixed in traderBoy `0bc237e`** (user: "Fix both login issues"):
  - startup: 6 PIN+TOTP rounds in-process (backoff 30/60/120/240/300 s, each in a fresh TOTP window) instead of
    2 tries then exit;
  - session guard: GET /v2/profile every 60 s; on a definite "invalid" the bot first adopts a newer token another
    droplet process already saved, else mints one (15 s timeout, PIN never logged), swaps it in place on the
    existing client, reconnects both WebSockets; max 3 an hour, then a loud error.
  Verified with fake-Dhan scripts only (26/26); deploy pending the user's go-ahead.

## Lessons (added)
- **Every login mints a session that evicts the previous one.** Anything that can log in (a timer job, a local
  script) must reuse the live bot's token, or run only when a fresh login cannot hurt.
- **Date-keyed caches + a UTC server clock = a daily 05:30 IST cliff.** Check what changes at UTC midnight before
  scheduling anything between 05:30 and 08:00 IST.

## Deployed (2 Oct, user: "yes deploy now and fix both leaks too")
- 07:27:49 IST `0bc237e` (login retries + session guard): reused the 06:20 token, no TOTP; 10 checks, 0 re-logins
  in its first 10 minutes.
- 07:38:42 IST `167b2ec` - two credential leaks closed:
  - PIN: in pin_totp mode the bot no longer hands the PIN to Tradehull/dhanhq (their login path logs, prints and
    tracebacks the request error - on a network error that text is the URL with PIN and TOTP). It reuses the cached
    token or mints one itself (network errors by class name, chained context suppressed, malformed PIN without its
    value, urllib3's URL-bearing DEBUG line kept off), then starts Tradehull in access_token mode. Checked first:
    0 occurrences of `pin=` in the journal (since 28 Sep) or Tradehull's log files - it had never leaked.
  - Token: dhanhq's OrderUpdate printed `Sent subscribe message: {...Token...}` on every connect - 50 journal lines
    since 28 Sep. All those tokens are dead except the current one (expires 3 Oct 06:20 IST). The bot now runs its
    own order-update session without the print.
  - After deploy: Tradehull in ACCESS TOKEN mode reused the cached token, order-update WS connected, 0 token
    prints, 0 `pin=`, 0 DH-906, session guard checking. Fake check 31/31; full suite = unchanged code.

## 2 Oct 08:00 IST - the plan is active, data is still refused (Dhan-side)
- The user renewed the Data API plan (auto-renewal on). `4447bdd` adds Dhan's own plan status to `/feed-stats`
  (data_plan / data_validity / token_validity, from the 60 s profile check) and logs a warning when the plan is
  not Active or ends within 3 days.
- Deployed 08:00:32 IST with a forced fresh login (new token minted by the PIN-safe code on the first try).
  Dhan's profile on that new token: **dataPlan "Active", dataValidity 2026-10-13 17:55:21**. Every data call on the
  same token: **DH-902 / HTTP 451** "User has not subscribed to Data APIs or does not have access to Trading APIs";
  market-data WebSocket HTTP 429; trading APIs fine.
- So the cause is neither the plan nor a stale token - it is on Dhan's side (something changed for the account
  around 1 Oct 22:47 IST, when every token was invalidated). Raised to the user to take up with DhanHQ support.

## Public reports and Dhan's limits (researched 2 Oct ~08:45 IST)
- **Multi-user Dhan-side outage:** MadeForTrade "DH-902 ERROR on active subscription" (posted 1 Oct 17:48 IST,
  errors since ~15:48; >= 4 users, one plan also valid to 13 Oct) and "Dhan Data API suddenly stopped working
  despite active subscription" (1 Oct 20:54 IST; NIFTY quote returning all-null error fields - our bot logged the
  same at 23:59). No Dhan staff reply at the time of reading. Our timeline fits: the 06:48 token kept data until
  Dhan invalidated it at 22:47; every token minted after that lacks data. Read-only test 2 Oct 08:20: a web.dhan.co
  SELF token is refused the same way from the droplet AND another network -> not token type, not IP.
- **Lockout:** Dhan staff on MadeForTrade (thread 56441): TOTP login errors appear "usually when 5 consecutive failed
  attempts happen at the server". No published lock duration. Other limits: 25 consents/day (API-key flow), token
  24 h, static IP only for order placement. 805 "Too many requests or connections. Further requests may result in
  user being blocked" is a warning (thread 60693 cleared by throttling, no block).
- **Our exposure:** 4 consecutive "Invalid TOTP" at 2 Oct 00:00 - one short of 5. The retry series built today (6
  tries) plus the fallback's 15-min primary retries could cross 5 -> proposed: a persisted cap of 4 consecutive
  failed tries, then pause PIN+TOTP and run on the fallback.

## Resolution (2 Oct 2026)
- **Dhan restored data access between 08:44:40 IST (last DH-902) and 09:09 IST** - no action on our side changed it
  (fresh tokens, a web token and another network had all been refused). Outage on our account: 1 Oct ~22:47 IST
  (token invalidated) / 00:01 (DH-902) -> 2 Oct ~08:45 IST; other users from ~15:48 IST 1 Oct.
- **4-try PIN+TOTP cap deployed `2c17a1d` 09:09 IST** (user request): one count for every process on the droplet,
  no 5th consecutive failed login ever sent (Dhan locks at 5), 6 h pause then one try per pause, bot runs on the
  fallback token or waits in-process meanwhile; `dhan_fallback_token.py login-reset` clears.
- **Weekly watchlist refresh completed 09:14 IST** (8 failed attempts since 00:00): 213/213 scored; Unified
  Momentum's real-money list +ABCAPITAL/ETERNAL/OBEROIRLTY/PAYTM/TVSMOTOR, -GLENMARK/KALYANKJIL/MANAPPURAM/PAGEIND/
  SAIL; clean safe restart.
- **What we have now that we lacked on 1 Oct:** in-process login retries, a 60 s token check with in-place re-login,
  a stored fallback access token, Dhan's data-plan status + expiry warning in /feed-stats, a data probe + retry
  timer for the weekly job, the 4-try cap, and no PIN / token in any log.

## Lesson (added)
- **Read what third-party SDKs print.** Two credentials reached logs through vendor code (an exception text that
  is a URL with the PIN in it; a debug print of the login message), not through ours.
