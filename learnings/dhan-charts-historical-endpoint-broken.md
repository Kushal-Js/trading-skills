# CORRECTED 2 Sep 2026: `/v2/charts/historical` was NOT broken — a stale local token produced a misleading error

**Update, same day, a few hours later**: the original finding below (found
2 Sep 2026 while backtesting `designs/hhhl-momentum-continuation.md`) was
**wrong about the root cause**. Verified directly against the live
`traderBoy` droplet — `Swing/trading_engine.py._fetch_daily_closes_once`
(the exact production function) returned a clean, correct 63-daily-candle
result for RELIANCE, and a raw call to the same endpoint from the droplet
returned `status: success` with real OHLCV data. **The endpoint itself is
fine, and Swing's live daily watchlist prune is NOT affected** — kept
below as the corrected record rather than deleted, per this repo's own
style guide ("update what changed and why, don't just delete the
history").

**What actually caused the original symptom**: re-testing locally with a
FRESH token reproduced success immediately — the local session's cached
access token had gone stale (`pin_totp` mode's own token cache) between
the original test run and this recheck. The stale token produced
`DH-905 Input_Exception: "Missing required fields, bad values for
parameters etc."` on this specific endpoint — **a misleading error
message for what was actually an auth problem** (compare: an expired
token on other Dhan calls, and even a bad-TOTP retry seen elsewhere this
session, surface a clear `DH-906 Invalid Token` instead). That's the
genuinely reusable finding here: **if `/charts/historical` ever returns
DH-905 again, check token freshness FIRST before assuming a payload
problem** — this endpoint doesn't necessarily report auth failures the
same way the rest of the API does. This is exactly why several payload
variations (`instrument_type` casing, `expiryCode`/`oi` presence,
different date ranges) all failed identically in the original
investigation below — none of them were the actual variable; the token
was already stale before any of that testing started.

**Practical consequence for future investigation**: don't conclude a Dhan
endpoint is "broken" from a local ad-hoc script's failure alone,
especially one whose own auth session has been sitting idle across a long
work session — re-authenticate fresh (or check `Token validity:` in the
login log line against the current time) before trusting a payload-level
diagnosis, and cross-check directly against the live droplet's own
process when a finding would otherwise get flagged as a live-production
risk.

---

## Original finding (2 Sep 2026, root cause was wrong — see correction above)

Found while fetching daily-context data for the HH/HL momentum-
continuation backtest. `historical_daily_data()` (→ POST
`/v2/charts/historical`) appeared to return `DH-905 Input_Exception` for
every request tried against this account, regardless of
`instrument_type`, date range, or whether `expiryCode`/`oi` were included
— confirmed (at the time) via both the SDK and a raw `requests.post`
directly against Dhan's own endpoint with the identical payload, which
seemed to rule out an SDK-level bug. It did not — it ruled out a
*payload* bug specifically, while the real cause (a stale token) was
never controlled for across those tests.

`intraday_minute_data()` (`/charts/intraday`) and `get_ohlc_data()`
worked fine throughout the original investigation — consistent with the
corrected finding, since neither of those happened to be exercised with
the same stale token in the same window.
