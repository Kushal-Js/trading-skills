# Design: revisiting K01's Stage 3 (stock selection) and order-placement approach — Smart Money Flow Cloud

**Status: proposal, not built.** Triggered by the user supplying the
"Smart Money Flow Cloud [BOSWaves]" Pine Script indicator (30 Aug 2026)
and asking to revisit K01's stock-selection and order-placement approach
in light of it. See `../learnings/technical-patterns/smart-money-flow-cloud.md`
for the full mechanism breakdown this proposal builds on — read that first,
this file assumes it.

This follows the same discipline `k01.md` itself was built under: design
and get explicit agreement on scope before touching `K01/paper_engine.py`
or `K01/config.py`. K01 is paper-only and currently disabled
(`K01_STRATEGY_ENABLED=false`), so nothing here carries real-money risk —
but it still shares Dhan REST/process infrastructure with the live
Options/Futures strategies (see `k01.md`'s shared-infrastructure-risk
finding), so a Stage 3 rewrite that changes call volume or timing still
needs the same rate-limit reasoning applied before re-enabling.

## What's actually being reconsidered

Two separate things, and they should be decided somewhat independently:

1. **Stock/entry selection (Stage 3)** — currently `momentum_signal()`
   requires ALL FOUR of: 5-min RSI in a directional band, 5-min close vs. a
   *fixed*-multiplier Supertrend(10,3), a 1-min Supertrend crossover, and
   5-min ROC sign agreement. None of these scale with how much real
   volume/conviction is behind the move.
2. **Order placement / entry timing** — currently single-shot: one entry
   per symbol, blocked from re-entry while a position is open
   (`_check_watchlist_for_entries`'s `if symbol in _open_positions: continue`),
   triggered only at the instant all four Stage 3 conditions first agree.

## Option A — Money-flow strength as an added filter (smallest change)

Keep Stage 3 exactly as-is, but require the Smart Money Flow Cloud's
`strength` reading (the volume-weighted CLV ratio, boosted by `mfPower`)
to clear a threshold (e.g. ≥0.4) at signal time, as a fifth AND-condition.

- **Pro**: minimal surface area — one new computed value, one new
  threshold config, no change to entry/exit timing logic at all.
- **Con**: doesn't actually use the adaptive-band regime-flip idea, which
  is arguably the more interesting part of the indicator — this is really
  just "add a volume-conviction filter," which could be done without
  porting the whole indicator (a plain Chaikin-Money-Flow style check would
  do the same job).

## Option B — Replace the dual-Supertrend regime check with the adaptive band (moderate change)

Swap `_compute_supertrend(h5, l5, c5, period=10, multiplier=3.0)` for a new
adaptive-band function whose multiplier scales with money-flow strength
(`minMult`→`maxMult` per the indicator's own formula) instead of being
fixed at 3.0. Keep RSI band and ROC sign as-is; the 1-min crossover check
would need to reference the new adaptive band too for consistency.

- **Pro**: this is the indicator's actual core idea, not just a bolt-on
  filter — bands widen during genuine volume-backed moves (fewer
  premature regime flips) and tighten during noise (faster exit from a
  low-conviction position). Directly addresses the "revisit the approach"
  framing, not just "add one more gate."
- **Con**: changes Stage 3's exit-relevant Supertrend parameters away from
  matching `Options/config.py`'s own exit-side Supertrend(10,3) — the exact
  mismatch `krishvi.md` flagged as a real problem when Krishvi's screener
  used period-7 instead of the bot's period-10. Needs a deliberate answer:
  is Stage 3's job "predict what the bot's OWN exit logic would call
  bullish" (argues for keeping period/mult matched to production) or
  "predict a genuinely different, hopefully better regime-detection
  signal" (argues for letting it diverge)? This wasn't an issue before
  because Stage 3 used the bot's own parameters verbatim; adopting an
  adaptive multiplier means it can no longer match by construction.

## Option C — Use "retest" as the actual entry trigger, not the flip (bigger change, touches order placement)

The retest signal (price pulling back to the trend baseline while regime
holds) is arguably a **better** entry than the raw flip: a flip crossover
happens after price has already cleared the ATR-scaled band — by
definition already a real move away from basis — whereas a retest is
priced close to the baseline, offering better risk/reward (tighter
stop-to-entry distance) and confirms the trend is intact rather than
brand new and unproven. This would mean:

- Entering on retest instead of (or in addition to) flip, which changes
  `_check_watchlist_for_entries`'s trigger condition itself.
- Potentially allowing a **second** entry into a symbol already
  in-position, if a retest fires while the original position is still
  open (a genuine scale-in) — this needs an explicit decision, since
  today's capacity model (`reserve_symbol`-equivalent's `if symbol in
  _open_positions: continue`) treats one open position per symbol as a
  hard block by design, and scale-ins interact with
  `MAX_LOSS_PER_TRADE_RS`/`highest_price`-based trailing logic in ways that
  need their own reasoning (which position's `highest_price` governs the
  combined size? does a second entry get its own independent target/SL, or
  average into the first?).

- **Pro**: potentially meaningfully better entries and a genuine second
  opportunity per name per day, not just a filter tweak.
- **Con**: real complexity increase in `PaperPosition`/entry bookkeeping,
  and — importantly — since K01 is a paper strategy meant to validate
  ideas cheaply, this is exactly the kind of change worth testing in
  isolation (paper) before ever considering it for the real Options
  strategy. Recommend building and testing Option A or B first, in a
  separate iteration, before taking on Option C's added complexity.

## Recommendation

Start with **Option B** (replace the fixed Supertrend multiplier with the
money-flow-adaptive one) as the core Stage 3 revision — it's the
indicator's actual idea, is a contained change (one new pure function next
to the existing `_compute_rsi`/`_compute_atr`/`_compute_roc`/
`_compute_supertrend` helpers), and is directly testable against the
existing CSV backtest methodology (`../learnings/backtest-methodology.md`)
before ever touching live-adjacent config. Treat Option C (retest-based
entries / scale-ins) as a distinct, later iteration once B has actual
paper-trading or backtest evidence behind it — not something to bundle into
the same change.

## Open questions needing the user's decision before implementation

1. **Which option (A/B/C, or a combination) to actually build first?**
   Recommendation above is B alone, C deferred.
2. **If B: does Stage 3's Supertrend deliberately diverging from the bot's
   own exit-side Supertrend(10,3) matter?** (See Option B's con above —
   this was a solved problem before; adopting an adaptive multiplier
   reopens it.)
3. **Timeframe**: keep the existing 5-min regime + 1-min crossover
   two-timeframe structure, or consolidate to a single timeframe now that
   the adaptive band is a different mechanism than the current fixed one?
4. **Money-flow lookback/smoothing/power parameters** — the indicator's own
   defaults (`mfLen=24`, `mfSmooth=5`, `mfPower=1.2`, `minMult=0.9`,
   `maxMult=2.2`) are TradingView chart-timeframe defaults (commonly daily
   or hourly charts) — they likely need re-tuning for K01's 5-min bars,
   the same way `SUPERTREND_5MIN_PERIOD`/`_MULTIPLIER` were deliberately
   set to match production rather than a scan's own untuned defaults.
5. **Validate via CSV backtest before any live-adjacent config change** —
   per the same discipline used for the DanDanaDan-2/Kaashvi-28 comparison
   (`../learnings/dandanadan-vs-kaashvi-3day-backtest.md`), any Stage 3
   revision should be measurable against real historical data before it
   ever runs against K01_STRATEGY_ENABLED=true, even in paper mode.

## Next step

Waiting on the user's answer to the open questions above before writing
any `K01/paper_engine.py` or `K01/config.py` changes.
