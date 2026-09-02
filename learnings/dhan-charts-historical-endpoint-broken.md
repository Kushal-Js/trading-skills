# Dhan's `/v2/charts/historical` (daily candles) endpoint is currently broken for this account

Found 2 Sep 2026, while fetching daily-context data for the HH/HL
momentum-continuation backtest (`designs/hhhl-momentum-continuation.md`) —
a real, currently-live issue, not a backtest-only concern.

## What's broken

`dhanhq`'s `historical_daily_data()` (→ POST `/v2/charts/historical`) — the
DAILY-candle endpoint, distinct from `intraday_minute_data()` (→ POST
`/charts/intraday`, confirmed working fine, tested up to 90 days back at
both 1-min and 5-min intervals) — returns `DH-905 Input_Exception:
"Missing required fields, bad values for parameters etc."` for **every**
request tried against this account, regardless of:

- `instrument_type` (`EQUITY`, `Equity`, `equity`, `STOCK`, `EQ` all
  identical failure)
- date range (5, 10, 15, 30, 60, 90+ days back — all identical failure)
- whether `expiryCode`/`oi` are included, omitted, or set to their SDK
  defaults
- calling via the SDK vs. a raw `requests.post` directly against
  `https://api.dhan.co/v2/charts/historical` with the exact same payload
  (ruling out an SDK-level bug — confirmed genuine HTTP 400 from Dhan's
  own server)

Confirmed against a definitely-valid, liquid, actively-traded symbol
(RELIANCE, security_id 2885) — not a delisted/illiquid/wrong-ID issue.

## Why this matters for `traderBoy`

`Swing/trading_engine.py._fetch_daily_closes_once` calls this EXACT same
endpoint with this exact same parameter shape, and is the data source for
the daily watchlist prune feature (added 1 Sep 2026 — see `traderBoy`'s
own NOTES.md entry #77). That function's own docstring already documents
it fails open (`return None`) on any fetch error, which the daily prune
tick then treats as "nothing to prune this run" — **no exception, no log
line distinguishable from an ordinary transient hiccup, nothing that would
have surfaced this as an active problem.** Given this endpoint's failure
looks structural (not transient — every single parameter permutation
failed identically), it's plausible the daily trend-based watchlist prune
has been silently doing nothing since deployment.

**Not yet verified directly against the live `traderBoy` droplet** — this
was found via a separate local backtest script authenticating against the
same Dhan account, not by testing the droplet's own running process. The
natural next step (flagged to the user, not yet actioned) is checking
`traderBoy`'s own logs/behavior directly, or calling
`_fetch_daily_closes_once` against a real watchlist symbol from the
droplet itself, before concluding the live feature is actually affected.

## What still works (don't assume the whole historical/quote API is down)

- `intraday_minute_data()` (`/charts/intraday`) — both 1-min and 5-min,
  confirmed working normally up to ~90 days back.
- `get_ohlc_data()` (used by `get_today_open_and_prev_close` /
  `get_day_change_pct`) — a different, quote-style endpoint, not the
  charts/historical one, unaffected.

So this is specific to the `/charts/historical` (daily OHLC) endpoint
only, not a broader Dhan API outage or an account-wide auth problem.

## Open

Whether this is a genuine Dhan-side outage (worth retrying in a few days
before spending more time on it) or a permanent API change that needs a
different payload shape than the current `dhanhq` SDK version sends.
Given several straightforward payload variations were already tried
without success, the productive next step if this needs fixing is
checking Dhan's own current API docs/changelog for `/v2/charts/historical`
specifically, rather than more guess-and-check payload permutations.
