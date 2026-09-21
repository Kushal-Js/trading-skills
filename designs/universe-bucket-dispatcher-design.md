Status: DEPLOYED LIVE 21 Sep 2026 (commit `fbd11f9`, restart 15:05 UTC /
20:35 IST), `UNIVERSE_DISPATCHER_ENABLED=true` for Luxury + Futures.
`universe_bucket` itself is currently EMPTY - no real Chartink screener is
pointed at its webhook yet ("after I integrate it" - user's own words,
not yet done as of this write-up), so this is live code with no real
trading effect until that webhook is wired up.

# universe_bucket + cross-package dispatcher + capacity backlog

## What this is, in one paragraph

A rolling 3-trading-day CE/PE symbol bucket (`universe_bucket.py`), fed by
a dedicated webhook so a NEW Chartink screener can become the breakout
scanner's watchlist source, plus a shared dispatcher (`breakout_signal.py`'s
own "dispatcher" section) that detects each bucket-sourced signal exactly
once and offers it to Luxury then Futures round-robin + sequential
fallback (never both racing for the same signal), plus a capacity backlog
that retries a signal both packages declined SPECIFICALLY for
`duplicate_or_capacity_full` once a slot frees up, gated by a real
momentum-reversal check on every retry.

## Why (the problem being solved)

User request, 21 Sep 2026, following the all-F&O-universe backtest
(`all-fno-universe-breakout-signal-15day-backtest.md`): "since there would
be just one source of signals now (breakout scanner), we also have to
design to pass these signals to LUXURY and FUTURES properly so that no
trade is left and is divided between 2 without dupes." Then, same day:
"once any slot from LUXURY or FUTURES gets free... [a signal] which
couldn't be filled earlier due to max capacity limit (stock still in
momentum) should be attempted."

The naive design (each package independently syncs the same shared
universe_bucket symbols into its OWN watchlist) has two real problems: (1)
both packages typically detect the identical signal within a scan cycle
of each other and both attempt entry, relying entirely on
`cross_strategy_registry`'s atomic claim to prevent a duplicate real
order - that part works, but (2) each package's own watchlist marks a
symbol "signaled" (never retried today) the instant it makes its ONE
entry attempt, INCLUDING when that attempt fails only because the other
package's claim briefly won a race it then abandons moments later
(rejected by its own gates, released) - the first package never gets a
second look, and the trade is genuinely lost to a race, not to either
package's own real risk controls.

## Design

### universe_bucket.py

- Separate CE/PE buckets (never shared state, matching every other CE/PE
  split in this codebase).
- One file per calendar day, `history/<date>_universe_bucket_<CE|PE>.json`
  - NEVER deleted (append-only audit trail, same convention as every
    other daily log in this repo).
- `active_symbols(option_type)` = union of the last 3 TRADING days
  (today + 2 prior), Saturday/Sunday skipped via calendar weekday check.
  **NSE market holidays are NOT excluded** - this codebase has no
  maintained forward-looking holiday calendar (checked before building,
  not fabricated) - a weekday holiday inside the lookback just
  contributes zero symbols (harmless, same as a real day the screener
  found nothing), meaning the window can occasionally cover fewer than 3
  true trading sessions during a holiday week.
- Two webhooks, standard Chartink payload shape: `POST /universe-bucket/
  webhook` (bullish -> CE), `/webhook-sell` (bearish -> PE). `GET
  /universe-bucket` for read-only observability.
- Verified (unit tests, isolated FastAPI router, no live app lifespan
  touched): rolling window correctly skips weekends
  (Mon-21st->[21,18,17], Tue-22nd->[22,21,18], confirming the 3rd-oldest
  day correctly drops off); CE/PE genuinely independent; malformed
  payload correctly rejected (422).

### breakout_signal.py dispatcher section

- `DISPATCHER_STRATEGY_NAME = "UniverseDispatcher"` reuses the EXISTING
  per-strategy watchlist/persistence machinery under a synthetic
  "strategy" name - no new persistence code needed.
- `universe_dispatcher_loop(targets)` - started ONCE for the whole
  process (main.py's own lifespan, not any one package's), given
  `[("Luxury", lcfg, luxury_entry_fn), ("Futures", fcfg, futures_entry_fn)]`.
  Registers both strategies in `_dispatcher_owned_strategies` so each
  package's own `_maybe_seed_universe` no-ops for them (prevents the
  double-processing race described above).
- `_dispatch_to_targets` - tries each target SEQUENTIALLY (never
  concurrently), starting position ROTATING per option_type (round-robin)
  so volume divides across both over time rather than always favoring
  whichever is listed first. Returns whether EVERY decline was
  specifically `duplicate_or_capacity_full` (vs some other reason like a
  gate/funds/delay) - only that specific case is backlog-worthy.
- **Capacity backlog**: a signal both targets decline purely on capacity
  is queued (FIFO). Drained FIRST every dispatcher tick (before new
  signals get their own scan budget - backlog entries have already been
  waiting). Each retry: (1) momentum re-check via
  `reversal_filters.check_underlying_move_confirms_exit` (the SAME real,
  already-backtested 0.10% confirmation threshold this codebase already
  trusts for exit confirmation, pointed the other direction - against the
  breakout rather than against an open position) - a reversed entry drops
  immediately, no point re-attempting into an invalidated breakout; (2) if
  still valid, re-dispatch through the same round-robin+fallback path -
  succeeds (removed), still purely capacity-blocked (stays queued), or
  declines differently now (dropped). Bounded by
  `BREAKOUT_CAPACITY_BACKLOG_MAX_AGE_MINUTES` (60 min default). In-memory
  only (resets on restart) - same accepted tradeoff as
  `cross_strategy_registry`/`PositionStore`'s own `reserved_symbols`.
- Verified (unit tests with fake entry functions, never real Dhan calls):
  round-robin rotation, sequential fallback (reject then accept), both-
  reject-logs-correctly, and the full backlog cycle (a reversed entry
  drops on cycle 1, a still-valid-but-capacity-blocked entry survives
  cycle 1 and succeeds on cycle 2 once a slot frees).

### underlying_candle_feed.py (built same day, separate concern)

WebSocket-based local 5-min candle reconstruction, an optional faster
alternative to REST-polling a wide watchlist - see the module's own
docstring. **Deliberately NOT enabled in this deploy**
(`BREAKOUT_USE_WS_CANDLES=false` everywhere) - the correctness parity
check found close/volume reconstruction reliable (213/213 matched bars
within 0.05%/1% respectively across RELIANCE/TCS/MAHABANK) but open-price
accuracy unverified under a genuine live tick stream (the REST-replay
test method samples once per minute, which can't validate true intra-
minute tick behavior) - flagged as needing a live, no-trading dry run
before enabling.

## Backtests run before deploying

1. **Simply Bull screener, 5 days** (`backtest_simply_bull_screener_5day.py`)
   - the user's own real screener export used as the curated per-day
   universe: 35 trades, +Rs47,257.35 vs real (same window, Luxury+Futures)
   121 trades, -Rs31,157.40. See
   [[simply-bull-screener-5day-backtest]] (journal entry, same date).
2. **Dispatcher vs racing isolation** - not separately backtested as a
   distinct scenario before deploy (time constraint) - the Simply Bull
   backtest's own dispatch logic already models round-robin+fallback
   without backlog.
3. **Capacity backlog isolation, Simply Bull scale**
   (`backtest_dispatcher_capacity_backlog_5day.py`) - ZERO measurable
   difference (31 trades, identical +Rs47,604.85, both with and without
   the backlog). Not a bug - verified via targeted unit tests that the
   mechanism works correctly in isolation; this dataset's signal volume
   (avg ~6/day) never actually exhausted BOTH packages' combined 6 CE
   slots at the same instant, so the backlog had nothing to do. See
   [[dispatcher-capacity-backlog-comparison]].
4. **210-stock/14-day scale** - queued, this is the dataset where
   capacity contention is real (890 `capacity_full` skips out of 2,184
   attempts, confirmed in the earlier racing simulation) - see
   [[dispatcher-capacity-backlog-comparison]] for the result once run.

## Deploy details

Commit `fbd11f9`. Files: `Options/Luxury/Futures/config.py` (new flags,
all default false/empty except `UNIVERSE_DISPATCHER_ENABLED` which was
explicitly set true in `.env` for this deploy), `Options/dhan_client.py`
(new equity Quote-mode subscription methods, kept in a SEPARATE lookup
dict from the existing option-LTP one - see the code's own comment citing
[[copper-mcx-security-id-collision-and-adoption]] for why that separation
matters), `breakout_signal.py`, `main.py` (dispatcher started in the
shared top-level lifespan, after every package's own lifespan has already
registered its entry function), 2 new files
(`universe_bucket.py`/`underlying_candle_feed.py`).

**Deliberately excluded from this deploy**: pre-existing, uncommitted
`alert_bucket.py`/loss-triggered-bucket-switch changes found sitting in
the working tree (not from this session) - references
`config.BUCKET_SWITCH_ENABLED`/`_LOSS_RS`/`_MIN_SCORE`, none of which are
defined anywhere, a real `AttributeError` waiting to fire. `main.py`'s
`import alert_bucket` + router-mount lines were surgically removed from
what got committed for this exact reason - left untouched, uncommitted,
in the local working tree for whoever finishes that separate feature.

Pre-restart: confirmed zero open positions on Options/Futures/Luxury/
Swing (twice - once before starting the commit/push sequence, once
immediately before the restart itself). Post-restart: clean startup log,
zero tracebacks from any new module, `/health` and `/universe-bucket`
both responding correctly, droplet RAM/swap healthy (521Mi available,
52Mi/1Gi swap used).

## What's still open

1. Point the user's new Chartink screener's webhook at `/universe-bucket/
   webhook` (or `-sell`) - until then this is live but inert (empty
   bucket, nothing to dispatch).
2. Run the 210-stock capacity-backlog comparison to get a real signal on
   whether the backlog earns its keep at meaningful scale.
3. A live, no-trading WS dry-run to validate `BREAKOUT_USE_WS_CANDLES`'s
   open-price accuracy before ever enabling it.
4. Someone should fix the missing `BUCKET_SWITCH_*` config keys (or
   decide not to pursue that feature) before that separate, unrelated
   work is ever deployed on its own - it will crash position monitoring
   otherwise.
