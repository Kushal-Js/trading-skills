# MCX candle-feed persists duplicate bars at some restart boundaries (COPPER/NATURALGAS only)

Found 25 Sep 2026, ~10:30 IST, while verifying candle continuity across
the watchlist after an unrelated deploy (commit 903f69f). Confirmed
pre-existing - duplicate rows found dated 2026-09-23, two days before
either of today's deploys, so unrelated to both.

**Symptom**: `Swing/candle_feed.py`'s per-symbol 5-min bar series has
genuine duplicate `candle_start` timestamps for COPPER and NATURALGAS
only - confirmed 25/243 and similar for NATURALGAS via
`/swing/debug/candle-feed/candles/{symbol}`. Every NSE equity/index
symbol on the watchlist (NIFTY, BANKNIFTY, SONACOMS, ASHOKLEY, SOLARINDS,
VEDL) is clean - sorted, zero duplicates.

**Confirmed write-side, not a read/restore aggregation bug**: read the
raw persisted file directly (`history/2026-09-23_swing_candles_COPPER_
571298.log`) - 52 lines, 9 duplicate `candle_start` groups already IN
the file on disk. `_load_persisted_bars` is correctly loading exactly
what's there; `_persist_bar` is the one appending the same bar_start
more than once.

**Likely mechanism (not yet confirmed by direct log correlation, but
consistent with all evidence)**: MCX's day+evening session runs far
longer than NSE's (per Options/config.py's MCX_MARKET_OPEN_TIME/
_CLOSE_TIME), so a much larger fraction of this service's frequent
restarts (57 in 48h measured earlier this session) land WHILE COPPER/
NATURALGAS are actively trading, vs. NSE symbols where most restarts
land outside 9:15-15:30 IST. A restart's fresh process starts
`_SymbolState.current_bar_start = None`; if its first tick(s) after
restart line up such that it treats the bar spanning the restart moment
as newly-completing, it can re-emit a `_persist_bar` write for a
`candle_start` the PREVIOUS process instance already completed and
persisted just before dying - `append_jsonl`-style file writes are pure
appends, nothing on the write path checks "is this candle_start already
in today's file." NSE symbols get the exact same restart exposure in
principle, but hit it far less often simply because fewer restarts land
during their (shorter) trading hours.

**Impact - real, not cosmetic**: confirmed `Swing/signals.py`'s
`_structure_break_ws_candles` calls `candle_feed.get_candles_dict(...)` -
the SAME array - and COPPER's actual live entries route through
structure-break (`COPPER_STRUCTURE_BREAK_ENABLED`), which reads this
exact candle_feed data. A duplicated bar means a candle's OHLC gets
double-counted at whatever indicator reads the full bar series (a
flat/no-move duplicate like the 19:55 pair is harmless, but the 20:25
group shows three DIFFERENT partial-bar snapshots of the same candle -
1406.65/1406.65/1406.65 -> 1406.65/1406.65/1405.15/1405.8 ->
1405.8/1405.8/1405.8/1405.8 - i.e. the bar was persisted at three
different stages of its own formation, not identical copies). Any
structure-break computation spanning this window sees a distorted
sequence, not the real one.

**Not fixed here** - found while verifying something else, real scope
(likely: dedupe-on-persist, e.g. `_persist_bar` should overwrite/skip
when `candle_start` already exists in today's file rather than blind-
appending, or the loaded series should collapse same-timestamp bars to
the last-written one before use) needs its own session. `structure_break_
debug_snapshot`'s own fail_streak/computed_at fields looked healthy in
this same check (COPPER: combined=0, fail_streak=0, computed 34s before
checked) - the bug hasn't caused an outright failure so far, just silent
data distortion within the affected windows.
