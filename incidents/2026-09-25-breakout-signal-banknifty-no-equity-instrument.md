# BANKNIFTY breakout-signal evaluation fails every cycle with "No NSE equity instrument found"

Found 25 Sep 2026, ~09:13 IST, during post-deploy monitoring of commit
`9933916` (unrelated perf fix - this is a separate, pre-existing issue
surfaced incidentally while watching logs, not caused by that deploy).

**Symptom**: `breakout_signal` logs, repeating each cycle:
```
ERROR | breakout_signal | BANKNIFTY: breakout-signal evaluation failed - will retry next cycle
ValueError: No NSE equity instrument found for BANKNIFTY
```
Traced to `Options/dhan_client.py:733`, `_equity_security_id()` - it only
matches rows where `SEM_EXM_EXCH_ID=="NSE"` and
`SEM_INSTRUMENT_NAME=="EQUITY"`. BANKNIFTY is an index, not an NSE equity
row, so this always raises for it - not a transient fetch failure, a
structural mismatch that will repeat every cycle indefinitely until fixed.

**Not an outage**: caught and logged ("will retry next cycle"), doesn't
crash the process, no real order placed or blocked as a result (nothing
else appears to depend on this specific call succeeding for BANKNIFTY).
Confirmed unrelated to `9933916` - different file section
(`_equity_security_id`, not the two ATM-fallback functions that commit
touched) and different code path (`breakout_signal.py`'s own universe
resolution, not `get_liquid_atm_option`).

**Likely cause, not yet confirmed**: BANKNIFTY was added to Swing's
watchlist this session (25 Sep, alongside NIFTY/SONACOMS/ASHOKLEY/
SOLARINDS/VEDL). `breakout_signal.py`'s own symbol universe appears to be
drawing from a shared source that now includes it, without excluding
index symbols the way Swing's own `INDEX_SYMBOLS`-aware code paths do -
`grep -rn BANKNIFTY breakout_signal.py` found no direct reference, so
whatever's feeding it BANKNIFTY does so dynamically (a shared universe/
watchlist source, not a hardcoded list) - not traced further yet.

**Not fixed here** - flagged during a live monitoring window with a
narrow purpose (watching one specific deploy), not the right moment to
make an unrelated live-code change. Next session: find what universe
`breakout_signal.py` reads that now contains BANKNIFTY, and either
exclude index symbols there the same way Swing's `is_index` checks do, or
confirm this is otherwise harmless and just noisy in the logs.
