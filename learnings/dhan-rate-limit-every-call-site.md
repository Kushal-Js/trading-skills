# Dhan's rate limit bites every unpaced call site, not just obvious loops

Applies to: any script (production code or a one-off backtest) that calls
`dhan_wrapper`/`Options.dhan_client`/`Dhan.*` methods directly. Confirmed a
third time, 30-31 Aug 2026, in a new way worth its own file rather than just
another line under `traderBoy/NOTES.md` bug #5.

## The pattern, three times now

1. **K01's lookback-fix debugging** (30 Aug 2026): a rapid 3-symbol test
   loop with zero delay between calls returned empty `{'data': ''}` for all
   three - immediately fixed by adding a 1.5s delay. Root cause understood
   at the time as "a tight loop needs pacing."
2. **This session's K01 v2 universe backtest, Phase 1's anti-SAGILITY
   check** (31 Aug 2026): `dhan_wrapper.get_atm_option()` +
   `dhan_wrapper.get_option_ltp()` calls were NOT in an obviously-tight
   loop in the naive sense - they only ran once per Stage-0+1 survivor
   (~34 out of 210 candidates), each pair separated by however long the
   *daily-fetch* loop's own iteration took for other stocks in between.
   **Every single one still failed** with `"No LTP returned"` -
   including OBVIOUSLY liquid names (TITAN, SAIL, DIVISLAB - all three
   confirmed to fetch fine standalone with 1.5s pacing between calls,
   moments later). This was NOT the "tight loop" pattern from finding #1 -
   these two specific calls (`get_atm_option`, `get_option_ltp`) simply
   were never routed through any pacing mechanism at all, while the
   *other* calls earlier in the same script (`historical_daily_data`, via
   `bt_common.py`'s `retry_call`/`_paced_call`) were.

## The actual lesson

**"This script has pacing" is not a property of the script - it's a
property of each individual call site.** A backtest script can have
correct, proven pacing on its daily-fetch loop and StillGet completely
rate-limited because a *different* pair of calls later in the same run
(triggered once per survivor, not once per raw universe symbol) was never
wrapped the same way. Before trusting ANY new Dhan-calling code path
(production or backtest), check every distinct call site individually for
pacing - don't infer "the script paces its calls" from having seen pacing
on one loop within it.

## How this was caught, and how to catch it fast next time

**Diagnostic signal**: multiple candidates in a row all fail the exact
same way (`"No LTP returned"`, `{'data': ''}`, etc.) INCLUDING names known
to be highly liquid. A real "this specific option is illiquid" failure
would be sporadic/name-specific; a rate-limit collapse is systemic and
hits everything indiscriminately, liquid or not - that's the tell.

**Fast confirmation**: pull 2-3 of the "failed" names and re-run the exact
same calls standalone, with generous pacing (1-1.5s), immediately after.
If they now succeed with normal-looking data, it was rate-limiting, not a
real data/liquidity issue - confirmed this way in ~10 seconds before
committing to a full pipeline fix.

**Fix**: add an explicit `time.sleep()` at each previously-unpaced call
site - don't assume wrapping the *loop* fixes calls made outside the
originally-audited path.

## Consequence for K01 v2's backtest specifically

The first full-universe-scan run of `k01-smart-money-flow-revision.md`'s
Phase 1 produced "0/210 candidates passed" - which was NEVER a real
result about the market, purely this bug. Re-run after the fix produced
34/210 real candidates (matching a plausible, non-degenerate pass rate).
**Any zero-or-near-zero-candidate result from a new backtest script
deserves this exact diagnostic before being reported as a finding** -
"the strategy found nothing" and "the script silently rate-limited itself"
look identical from the summary numbers alone.
