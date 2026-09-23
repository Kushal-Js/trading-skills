# COPPER MCX position orphaned for ~22h after broker-side SL cancelled + 3h16m downtime + silent reconciliation gap (23 Sep 2026)

## What happened

Real Swing position, `COPPER 23 SEP 1420 CALL`, qty 1, opened 22 Sep 2026
13:45:12 IST, entry Rs5.69, target Rs6.83, SL Rs4.55, order_id
`23826092226709`. User found it still open on the Dhan broker app on 23 Sep
(next day), past target, requiring a manual close - the bot never touched
it after entry. Full evidence chain from `journalctl` (droplet), not
inferred.

## Root cause (two compounding, independent bugs)

1. **Broker-side SL cancelled 90s after entry, position left unprotected
   longer than intended.** The SL-L order (trigger=3.89, limit=3.80) shows
   as CANCELLED without firing at 13:46:49 IST - reason not visible from
   our own logs (not a fill, not a rejection we logged). The engine
   correctly fell back to its documented behavior ("relies solely on the
   regular poll/tick-driven MAX_LOSS_HIT check"), but then re-logged that
   exact same fallback warning **~150 times, every 6-7 seconds, for 17
   minutes straight** (13:46:49-14:03:29) - a real, separate bug: the
   monitor loop re-detects "SL order ended CANCELLED" on every tick
   instead of latching the state once it's been handled. Cosmetic/log-spam
   impact only on its own, but it's a sign this code path treats a
   terminal order state as something to keep re-discovering rather than a
   one-time transition.

2. **3h16m of complete bot downtime, immediately after, with the position
   unprotected.** `dhanboy.service` was deliberately stopped (clean
   SIGTERM, `status=143`) at 14:03:34 IST - 5 seconds after the last SL-
   cancelled warning - and did not come back up until 17:19:29 IST. During
   this entire window, mid-MCX-session, nothing was watching this
   position's target or stop-loss at all.

3. **Real bug: silent reconciliation gap, confirmed across ~9 restarts
   spanning two days.** `Swing/trading_engine.py`'s
   `reconcile_broker_positions()` (via `get_open_mcx_positions()` in
   `Options/dhan_client.py`) is specifically designed to re-import a
   surviving MCX position like this one on every restart (added 12 Sep
   2026, confirmed working for other MCX trades before). At the 17:19:29
   restart, and every restart since (including all ~7 restarts today, 23
   Sep), reconciliation ran to completion with **zero exceptions logged**
   (confirmed: `grep -c "Could not reconcile broker positions"` across the
   whole window returns 0) but **never once surfaced COPPER** - no
   "Reconciled N existing Swing position(s)" line, no
   "Skipping Swing reconciliation for X - attributed to Y" line either
   (which DOES fire correctly for other unattributed positions, e.g. a
   real SBIN position from Options seen the same day) - meaning Dhan's own
   `get_positions()` REST response apparently never included this
   position at reconciliation time, even though the broker's own app UI
   still shows it as open. Root cause of that specific REST-vs-UI
   discrepancy is NOT resolved - would need a live broker positions dump
   to pin down (not done, since the user was actively closing the
   position by hand and a live debug session wasn't the priority at that
   moment).

## Why this is distinct from the same-day breakout-signal incident

[[2026-09-23-breakout-signal-single-snapshot-missed-signals-and-ws-walk-fix]]
(same day) is about ENTRY timing being late. This is a completely
different failure mode - a real position that entered fine, then became
fully invisible to every layer of the system (no SL, no monitor loop
polling for 3h16m, then no reconciliation on ~9 subsequent restarts) while
still real and open at the broker. Not the same bug, not the same fix.

## Status

User closed the position manually via the Dhan app on 23 Sep after this
investigation. No code fix has been deployed yet - see "recommended fixes"
below, deliberately not applied same-session per
[[feedback-live-trading-safety]] (no urgency once manually closed, and the
reconciliation-gap root cause itself isn't fully understood yet, which
makes a same-session code fix premature).

## Recommended fixes (not yet implemented)

1. Make `reconcile_broker_positions()` log an explicit line on EVERY
   restart even when it finds zero broker positions across all three
   segments (e.g. `"reconcile: 0 NSE_FNO / 0 NSE_EQ / 0 MCX_COMM positions
   from broker"`), so a silent gap like this is visible in logs instead of
   being indistinguishable from "genuinely nothing to reconcile." This is
   the highest-value fix - it would have made this gap loud instead of
   silent across all 9 restarts.
2. Latch/throttle the "SL order ended CANCELLED" warning so it logs once
   per state transition, not every ~6s indefinitely.
3. Ask Dhan support why a MARGIN-product (intraday) MCX option position
   survived a session/day boundary without RMS auto-square-off - that's
   unusual and worth understanding independent of our own code, since it's
   part of how this became a multi-day orphan in the first place.
4. Once the reconciliation-gap root cause is actually understood (needs a
   live broker positions API dump compared against what the app UI shows
   for a similar still-open MCX contract), fix the actual mismatch, not
   just the missing log line.
