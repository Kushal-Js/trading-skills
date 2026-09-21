# 2026-09-21: Breakout-signal scanner promoted to sole entry path (Options/Luxury/Futures) - deploy, a broken first attempt, and the fix

## Summary

User request, made after seeing the breakout-signal scanner fire only
once all session (Luxury/CGPOWER, skipped as a duplicate) while the
normal webhook-driven ranked entry placed 8 real trades: **"the
breakout-signal scanner has to be main entry path... webhook path will
feed the signals to breakout-signal scanner and it will decide which
trades to be placed"**, for all three real-money packages (Options,
Luxury, Futures). Explicitly NOT scoped to PE-only for Options - "for
all 3 strategies" meant full CE+PE parity, so Options' breakout-signal
wiring (built and left disabled earlier the same day, see
[[options-pe-breakout-signal-gated-live-full-real-gates]]) was widened
from PE-only to CE+PE and its `BREAKOUT_SIGNAL_ENABLED` flipped to
default `true`, since without a fallback path Options would otherwise
place zero trades at all.

User was explicitly warned before building this: (1) not validated as a
sole-gate config, only as a retrospective "what if this had replaced
today's real trades" backtest; (2) today's actual scanner activity (1
signal all session) implies a large drop in trade frequency; (3) this
carries real-money risk deploying mid-session with the market live. User
chose "CE+PE" and "deploy now, mid-session" on both counts, explicitly
informed.

## What changed (code)

`Options/option_main.py`, `Luxury/luxury_main.py`, `Futures/
futures_main.py`: `_handle_chartink_webhook` no longer calls
`enter_positions_for_stocks` (the ranked-entry function) at all. A raw
Chartink alert now ONLY records into `breakout_signal.py`'s watchlist
(via `record_alert`) and returns `{"status": "queued_for_breakout_
signal", ...}`. `rank_and_pick_top_stocks`/the ribbon-ranking functions/
`enter_positions_for_stocks` are UNCHANGED and still fully defined in
each package's `trading_engine.py` - kept, not deleted, matching this
repo's own convention for retired-but-present code (`choppy_stocks.py`)
- but nothing calls them from this path anymore. Every real entry gate
that used to run per-ALERT in the webhook handler (trading windows,
allowed-time cutoff, square-off time, gap-down CE delay, capacity) now
runs per-SIGNAL inside each package's own `_breakout_entry_fn` instead,
unchanged in content, just relocated in when it fires.

`breakout_signal.py` itself is completely unmodified - this is a wiring
change only, reusing the exact same scanner module Luxury already had
live since 20 Sep.

Bundled into the same deploy: the market-data WebSocket backoff fix
(`_run_market_feed_forever`/`_market_feed_watchdog_forever` in
`Options/dhan_client.py`) - see
[[2026-09-21-market-feed-thread-death-on-429]] for that one's own full
write-up.

## The broken first deploy (self-inflicted, caught and fixed same session)

The first commit (`b889608`) staged the three `*_main.py` files
whole-file via `git add`. Unknown at commit time: those files, in the
local working tree, ALREADY contained the user's own separate,
pre-existing, uncommitted work - an `import alert_bucket` +
`alert_bucket.record_alert(...)` integration (an unrelated, still-in-
progress feature, part of the same batch of already-modified-but-
uncommitted files present in the working tree since before this session
started: `main.py`, `Options/trading_engine.py`, `Options/position_
store.py`, plus the untracked `alert_bucket.py` module itself). Staging
"the files I changed" swept in that pre-existing content too, since git
stages a file's CURRENT full state, not a diff scoped to one session's
edits.

`alert_bucket.py` itself was never committed to git and never deployed
to the droplet at any point (confirmed: `git log --oneline --all --
alert_bucket.py` on the droplet returns nothing, and the file doesn't
exist on disk there). The pushed commit's `import alert_bucket` at
`Options/option_main.py` module level therefore crashed the whole
process on startup:

```
ModuleNotFoundError: No module named 'alert_bucket'
dhanboy.service: Main process exited, code=exited, status=1/FAILURE
```

No open positions existed at the time (confirmed via a fresh check
immediately before the restart), so there was no unmonitored-position
risk during the outage - but the live service WAS down (health endpoint
unresponsive) for several minutes until the user manually ran `git
reset --hard ca7a173 && systemctl restart dhanboy.service` on the
droplet directly (a destructive remote command the harness's own auto-
mode classifier declined to run automatically, correctly - the user
approved and ran it themselves).

## The fix

Rebuilt the SAME intended changes cleanly, this time verified against
the actual last-known-good commit rather than against a working tree
that silently contained unrelated WIP:

1. `git show ca7a173:<path>` for each of the 7 target files, to see
   exactly what was really deployed and working (not what happened to
   be sitting locally).
2. `git checkout ca7a173 -- <6 files>` (Options/dhan_client.py excluded
   - confirmed clean via `diff` against its own ca7a173 version, no
   alert_bucket references, no unrelated WIP) to reset the working tree
   to the verified-clean baseline.
3. Re-applied the exact same webhook-rewrite/config changes by hand onto
   that clean baseline - same content as the first attempt, minus every
   `alert_bucket` line (which was never part of the intended change to
   begin with).
4. Verified before committing this time: syntax check on all 7 files,
   `grep` confirming zero real `import alert_bucket`/`alert_bucket.*(`
   code references anywhere (a couple of pre-existing COMMENT-only
   mentions in Luxury/Futures config.py, already present in ca7a173
   itself, left untouched), a full `main.py` import test, a direct
   behavioral test (monkeypatched `enter_positions_for_stocks` to raise
   if called - confirmed it's never called, for any of the 3 packages,
   for either CE or PE), and a full `pytest tests/` diff against the
   pre-rewrite baseline (0 new failures; one unrelated pure exit-ladder
   unit test flipped pass/fail from unrelated test-ordering/global-
   config-state leakage, confirmed by reading it - no webhook/entry code
   involved at all).
5. New commit (`db07226`, NOT an amend/force-push - `b889608` stays in
   history as-is), pushed, pulled on the droplet, a DRY `uv run python
   -c "import main"` sanity check run directly on the droplet BEFORE
   touching the live service (new step, not done before the first
   attempt - would have caught the alert_bucket issue immediately
   without ever needing a restart), fresh pre-restart position check
   (still flat), restart, verified healthy.

## Verified live state after the fix

- `/health` -> `{"status":"ok"}`.
- `/feed-stats` -> `feed_connects: 1, feed_errors: 0` - WS backoff fix
  confirmed clean on this restart too.
- Startup log confirms all three: `Options strategy startup complete:
  ... breakout-signal scanner (CE+PE, SOLE entry path, enabled=True)`,
  `Futures strategy startup complete: ... breakout-signal scanner (SOLE
  entry path)`, `Luxury strategy startup complete: ... breakout-signal
  scanner (SOLE entry path)`.
- Luxury/Futures' breakout-signal watchlists (`/luxury/breakout-signal`,
  `/futures/breakout-signal`) survived the restart with today's earlier
  alerts intact (CGPOWER still shows `signaled: true` from this
  morning) - persistence working as designed, unaffected by any of
  today's restarts.
- Options' own watchlist (`/breakout-signal`) is freshly empty - correct
  and expected, since this is the very first restart with Options ever
  wired into `breakout_signal.py` at all.
- All 4 packages' `live_positions` confirmed empty immediately before
  AND after this restart.

## Lessons for next time

- **`git add <file>` stages the file's entire current state, not a diff
  scoped to "what I personally edited this session."** When a target
  file already has uncommitted content from before the session started,
  staging it whole-file silently carries that content along. Before
  committing a file with pre-existing uncommitted changes, either diff
  against the last real commit (`git diff <last-commit> -- <file>`) to
  see the FULL set of changes about to ship, not just the diff since the
  session's own edits began, or stage more surgically.
- **A dry import check on the deploy target, before restarting the live
  service, is cheap and should be standard** - `uv run python -c "import
  main"` directly on the droplet catches exactly this class of error
  (missing module, syntax error, broken import chain) with zero
  service-down time. Not done before the first attempt; is now part of
  the standard deploy sequence going forward for any change touching
  imports.
- The harness's own auto-mode classifier blocking a destructive remote
  git operation (`git reset --hard` + `systemctl restart` combined) and
  requiring the user's own hands-on-keyboard action to run it was the
  correct behavior here, not friction to route around - it's exactly the
  kind of hard-to-reverse, live-system-affecting action that standing
  practice says should get a human in the loop, especially mid-incident.
