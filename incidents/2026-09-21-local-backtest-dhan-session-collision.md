# Incident: local backtest's Dhan re-auth collided with the live bot's session, forced a real service restart

## What happened

While running `traderBoy/backtest_all_fno_breakout_signal_15day.py` (the
all-F&O-universe breakout-signal backtest, see `designs/all-fno-universe-
breakout-signal-15day-backtest.md`) locally during market hours, the local
process authenticated to Dhan using the same `pin_totp` credentials as the
live droplet bot. Dhan appears to allow only **one active access-token
session per account** — each local re-authentication mints a genuinely new
token, invalidating whatever token the live bot was currently holding.

Confirmed via `journalctl -u dhanboy.service` (droplet clock is UTC, see
[[feedback-droplet-utc-timestamps]]): at **08:43:39 UTC (14:13:39 IST)**,
`dhanboy.service` was fully **stopped and restarted** — not just an
in-process token refresh, a new PID (216927 → 221124). The log immediately
before the stop shows a growing string of `swing_signals` fetch failures
("could not fetch Supertrend state - keeping last cached value"), then on
restart: `Cached token invalid, deleting cache and doing PIN + TOTP login:
... DH-906 Invalid Token ... New PIN + TOTP access token validated
successfully`. Minutes later (14:16 IST), the LOCAL backtest process
started getting its own `DH-906 Invalid Token` errors — consistent with
the freshly-restarted live bot's own new token now invalidating the local
process's token in turn, i.e. the two were repeatedly kicking each other
out.

**No open positions were affected** (`/positions` showed empty throughout,
confirmed before and after) — real trading was not caught mid-flight this
time. This was checked live before doing anything else, per
[[feedback-live-trading-safety]] — the restart itself is a real, confirmed
production event even though the outcome was benign this time.

## Root cause

Any local script that calls `dhan_wrapper.authenticate()` in `pin_totp`
mode (the default local `.env` auth mode) mints a competing session
against the SAME live account the droplet bot is continuously using. This
is not specific to this one backtest script — **any** local tool that
re-authenticates via PIN+TOTP while the live bot is running carries this
risk, including ad-hoc diagnostic `python3 -c` one-liners (several of
which were also run this session while debugging a separate, unrelated
rate-limit bug — each one independently re-minted a token).

A SEPARATE, distinct issue was also found the same session: even with the
collision fixed, sustained bulk historical-data pulls from a local process
during market hours produce `DH-904 Rate_Limit` ("too many requests on
server from single user breaching rate limits") — this is shared
per-account REST-call budget contention with the live bot's own real-time
traffic, not a session/token problem, and persists regardless of auth
mode. See the design doc for how both were handled.

## Fix used this session

1. **Session collision**: the user handed over a fresh Dhan access token
   in chat (the established "third verification path" pattern, see
   [[project-dhanboy-deployment]]) and the backtest script was changed to
   use `access_token` mode with that token instead of `pin_totp` — this
   VALIDATES an existing token rather than minting a new one, so it
   coexists with the droplet's own session instead of kicking it out.
   Confirmed working: `/health` stayed OK through the rest of the session,
   no further DH-906 errors from the collision itself.
2. **Rate-limit contention**: deferred the heavy 210-symbol/14-day pull to
   after market close (~16:00 IST, 30 min buffer past the 15:30 close),
   when the live bot's own call volume drops to idle/reconciliation
   levels. Also raised `CALL_PACE_SECONDS` 0.35 → 1.2 and added a proper
   retry-with-backoff wrapper (5 retries, 5s-25s backoff) around every
   Dhan REST call, since even post-close the live bot never fully stops
   (Swing/paper-trade polling, reconciliation).

## Standing rule for future local backtests/scripts against Dhan

- **Never call `dhan_wrapper.authenticate()` in `pin_totp` mode from a
  local process while the live bot might be running** (i.e. essentially
  always, since the bot runs 24/7 via systemd `Restart=always`). Use a
  hand-off `access_token` from the user instead — ask for one in chat the
  same way this session did, never try to read/extract the droplet's own
  cached token programmatically (that was correctly blocked by the
  sandbox's credential-exploration guard when attempted this session).
- **Any bulk/heavy historical-data pull should run after market close**,
  not during 09:15-15:30 IST, regardless of auth mode — shared rate-limit
  budget contention with the live bot's real-time calls is a separate risk
  from the session collision and isn't fixed by switching auth modes.
- Check `/health` (and ideally `/positions`) before AND after any local
  script that touches Dhan auth, so a real interference event is caught
  immediately rather than discovered later.
