# 2026-09-22: MCX symbols' Supertrend/regime fetch failures are DH-904 rate-limit contention, not an MCX-segment data problem

## What was asked

After [[2026-09-22-swing-signal-cache-never-throttled-on-failure]] fixed
the amplification bug (unthrottled retry storm), a live check ~20 minutes
into market open found the throttled failures concentrated almost
entirely on NATURALGAS/COPPER (MCX) versus a clean COALINDIA/ASHOKLEY/
ANGELONE (NSE equity) in the same Swing watchlist - user asked why.

## Finding: NOT an MCX-segment reliability problem - it's account-wide DH-904 rate-limit contention

`Options/dhan_client.py`'s `fetch_continuous_intraday` got a diagnostic
log line the same day (22 Sep) that captures Dhan's actual raw response
on an empty-data return, not just "empty" - checking it directly against
a 20-minute live window (04:08-04:28 UTC) settled the question with hard
evidence instead of inference:

```
fetch_continuous_intraday(security_id=568245, segment=MCX_COMM, instrument=FUTCOM, ...)
  raw response: {'status': 'failure', 'remarks': {'error_code': 'DH-904',
  'error_type': 'Rate_Limit', 'error_message': 'Too many requests on
  server from single user breaching rate limits. Try throttling API calls.'}}
```

**Every single failure in the window (134/134) was `DH-904 Rate_Limit`** -
zero were a genuine empty-data/out-of-range/other response. Broken down
by segment:

| Segment | Failures | Share |
|---|---:|---:|
| MCX_COMM | 87 | 65% |
| IDX_I (index - NIFTY/BANKNIFTY, shared gap-down check) | 37 | 28% |
| NSE_EQ | 9 | 7% |
| NSE_FNO | 1 | <1% |

MCX is disproportionately represented, but **NSE_EQ and NSE_FNO calls hit
the exact same DH-904 too, in the exact same window** - this rules out
"Dhan's MCX data feed for this segment is specifically unreliable" as the
explanation. It's the same account-wide REST budget (`DH-904`'s own
wording is account-scoped, not segment-scoped) under real contention from
the combined call volume of everything now running concurrently: Swing's
own regime/Supertrend polling, Options/Luxury/Futures' own breakout
scanners (3 loops), the NEW UniverseDispatcher (a 4th, added overnight -
see [[2026-09-21-breakout-signal-sole-entry-path-deploy]]'s own
follow-up), and - as of this session - the live-tick underlying-feed
observation for 8 symbols (WS-only, not REST, but still new).

## Why MCX still bears the disproportionate share

By security_id, not just segment:

| security_id | Symbol | Failures |
|---|---|---:|
| 568245 | NATURALGAS | 72 |
| 25 | (IDX_I, likely NIFTY) | 27 |
| 571298 | COPPER | 13 |
| 13 | (IDX_I, second index) | 10 |
| 27066 | (unidentified) | 4 |
| 1333 | (NSE_EQ) | 3 |
| 881 | (NSE_EQ) | 2 |
| 101886 | (NSE_FNO) | 1 |

**NATURALGAS alone is 54% of every failure recorded, system-wide, across
every package and every segment** - more than DOUBLE the entire NSE_EQ
category. COPPER, the OTHER MCX symbol in the exact same watchlist with
the exact same polling cadence (regime 60s, Supertrend 15s, both fast+
slow intervals), shows a much lower failure count (13) - not proportional
to "MCX segment" being the driver, since both are MCX_COMM/FUTCOM calls
through the identical code path (`Swing/signals.py`'s `_underlying_
reference`).

**Not fully explained by this session's investigation** - NATURALGAS'S
specific over-representation relative to COPPER (same segment, same
polling pattern) suggests either a Dhan-side per-contract or per-security
quirk not visible from this account's own logs, or some other call
source hitting NATURALGAS's specific security_id that wasn't identified
in this pass. Flagged as an open question, not resolved.

## Why this matters, and why it's not urgent

Fails open by design (last-good cached value, confirmed already in
[[2026-09-22-swing-signal-cache-never-throttled-on-failure]]) - no
incorrect trade results directly. Zero Swing positions were open during
this whole investigation window. The real cost is Supertrend/regime
signal staleness specifically for NATURALGAS (and to a lesser extent
COPPER) at whatever moment Swing might actually try to evaluate an entry
into either.

**This is the real-world materialization of the "watch closely for
throttling once market opens" risk flagged in the morning's own pre-
market review** (see that session's own evaluation, same day) - the
UniverseDispatcher's added concurrent REST load was explicitly called out
as untested at real market-hours volume; this is the first confirmed
evidence of it contributing to real rate-limit pressure, though NIFTY/
BANKNIFTY's own 28% share and NATURALGAS's outsized 54% share suggest the
dispatcher isn't the ONLY or even necessarily the dominant contributor -
this needs more targeted investigation (e.g., temporarily disabling the
dispatcher and re-measuring) to actually attribute cause, not just
correlate timing.

## Not done this session (disclosed)

- Did not attempt to isolate the dispatcher's own contribution by
  temporarily disabling `UNIVERSE_DISPATCHER_ENABLED` and re-measuring -
  would need explicit user go-ahead given it changes live Luxury/Futures
  entry behavior, not something to toggle mid-session for a diagnostic.
- Did not investigate why NATURALGAS specifically outpaces COPPER within
  the same MCX segment/polling pattern - real, open question.
- Did not check whether Dhan enforces genuinely separate per-segment rate
  limits (vs. one account-wide bucket) - `DH-904`'s own error text reads
  as account-wide ("single user"), which is what this write-up assumes,
  but this isn't independently confirmed against Dhan's own (undocumented)
  API behavior beyond this one observation window.
