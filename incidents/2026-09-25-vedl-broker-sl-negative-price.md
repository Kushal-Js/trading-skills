# VEDL broker-side stop-loss order rejected: DH-905 Invalid Price (negative trigger/limit)

Found 25 Sep 2026, ~09:35 IST, while investigating a real live entry
(`VEDL 29 SEP 265 CALL`, LONG CE, qty/pnl_multiplier 1150, fill 2.95,
opened 09:30:05 IST). Unrelated to the same-morning deploy of `9933916`
(different module entirely - this is `Swing/position_store.py`'s broker
stop-loss math, not the ATM-resolution fallback that commit touched).

## Root cause (confirmed by direct calculation, not guessed)

`broker_stop_trigger_and_limit()` (`Swing/position_store.py:214`):
```python
per_unit_cap = cap_rs / quantity
gap = cap_rs * gap_multiple / quantity
trigger = fill_price - per_unit_cap   # LONG side
limit = trigger - gap
```
Live values: `MAX_LOSS_PROTECTION_RS=4500` (code default, no
`SWING_MAX_LOSS_PROTECTION_RS` override in `.env`),
`BROKER_STOP_LOSS_LIMIT_GAP_MULTIPLE=0.05` (also code default - `.env`
only has the bare/Futures/Luxury-prefixed keys, no `SWING_`-prefixed one).

For VEDL: `per_unit_cap = 4500/1150 = 3.91`, `fill_price = 2.95`.
`trigger = 2.95 - 3.91 = -0.96`, `limit = -0.96 - 0.196 = -1.16`. **Both
negative** - not a valid price on any exchange, so Dhan's real response
was exactly correct: `DH-905 Input_Exception: Invalid Price`. This was
deterministic, not a transient/flaky rejection - the same inputs will
produce the same negative prices every time.

## Generalizes beyond this one trade

Triggers whenever `MAX_LOSS_PROTECTION_RS / pnl_multiplier > entry_price`
- i.e. any position where the fixed rupee-loss cap divided by lot size
exceeds the option's own premium. Real risk on any cheap/far-OTM contract
with a large lot size, any symbol, not VEDL-specific. No existing test
(`tests/test_swing_broker_stop_loss.py`) covers this negative-price case,
and the function has no floor/clamp on the computed trigger/limit.

## Why it wasn't worse

The call site (`Swing/trading_engine.py:687-706`) wraps this in
`except Exception: logger.exception(...); proceeding without it, the
existing poll/tick-driven MAX_LOSS_HIT check still protects this
position exactly as before` - graceful degradation, not a crash, and the
position remains genuinely protected by the tick-driven check (same
mechanism [[exit-mechanics]] already documents as reliable). No extra
real-money risk was taken beyond "no resting broker-side order for this
one position" - the internal check still runs every tick.

## Not fixed here

Flagged during an unrelated deploy-watch session, not the right moment
for a live code change. Two reasonable fixes for later: (1) clamp
trigger/limit to a small positive floor (e.g. never below one tick) when
the raw calculation goes non-positive, or (2) detect the non-positive
case before ever calling the broker and skip straight to "proceeding
without it" with the same log line, avoiding the doomed API call
entirely. Either preserves the existing fail-open behavior - this is a
"stop wasting an API call and a confusing DH-905 in the log" fix, not a
correctness fix to real trading logic (the position was never actually
unprotected).
