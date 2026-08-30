# Design: K01 v2 — retest-based entries (Option C), enhanced with existing trading-skills findings

**Status: proposal, not built — explicitly design/skill-building phase
only.** Per the user's own framing (30 Aug 2026): "We won't deploy anything
for paper trading, first we will work on our skill set, have some perfect
strategy created." No `K01/paper_engine.py` or `config.py` changes until
this design is backtested and explicitly approved for a build pass.

**Supersedes the "pick one of A/B/C" framing of this file's first version**
(git history) — the user chose **Option C** (retest-based entries), with
instruction to layer in the rest of this repo's existing findings on top,
not use it in isolation. This version is the actual concrete design, not a
menu of options.

## Read this first: the realistic target is not "no loss"

The user's stated goal was "very clear signals... and almost no loss."
**No options strategy achieves that, and it's worth saying plainly rather
than quietly inheriting an impossible bar.** Two reasons, both already on
record in this repo:

1. **Genuine liquidity gaps are unpreventable, not a monitoring bug.**
   `learnings/exit-mechanics.md`'s SAGILITY case: entry 2.29, cap 2.19, the
   real market printed 2.25 → 2.14 with *zero trades in between* — no
   possible exit logic could have done better than react to the first
   post-gap tick. This can happen to a perfectly-designed strategy on a
   thin contract; it's a liquidity property of the instrument, not a flaw
   in the signal.
2. **Even the cleanest real backtest so far still lost sometimes.**
   DanDanaDan-2's 3-day backtest (`learnings/dandanadan-vs-kaashvi-3day-
   backtest.md`) — the best result measured in this codebase — had a 77.4%
   win rate, not 100%: 2 of 31 trades hit MAX_LOSS_HIT.

**The actual, checkable target this design aims at**: beat that existing
77.4%-win-rate / 2-MAX_LOSS_HIT-in-31 benchmark on a real backtest, via
tighter entries and stricter liquidity gating — not eliminate losses
outright. Treat "win rate" and "MAX_LOSS_HIT frequency" on a real CSV/Dhan
backtest as the metrics that settle whether this design actually improved
anything, the same way `learnings/backtest-methodology.md`'s standard
workflow already does for every other screener change in this repo.

## The core mechanic (Option C, concretized)

Replace Stage 3's current "enter the instant all four conditions first
agree" (a flip-chasing entry, by construction already priced away from the
baseline) with a **retest-based entry**: wait for an established regime,
then enter on the pullback-to-baseline, not the initial flip. Per
`learnings/technical-patterns/smart-money-flow-cloud.md`'s mechanism
breakdown, this means:

1. **Regime establishment** — an adaptive-multiplier band (money-flow
   strength scales the ATR multiplier, replacing the current fixed
   Supertrend(10,3) — this folds in what was "Option B" in the prior
   version of this doc; B and C aren't actually separable, since C's retest
   concept only exists relative to B's adaptive band) flips bullish/bearish.
   **Don't enter on the flip itself** — record it and wait.
2. **Retest confirmation** — price pulls back to touch/cross the trend
   baseline (`bs.bC`) without the regime itself flipping. This is the entry
   trigger. Rationale: entry price is close to the baseline, so the natural
   stop (a regime flip) is a *small* move away — meaningfully better
   risk/reward geometry than entering right after a flip, where price is
   already displaced from baseline by construction.
3. **Money-flow strength gate at the retest bar** (folds in "Option A") —
   require `strength ≥ 0.4` (tune via backtest) at the retest bar itself,
   not just at the original flip. Rationale: a retest on collapsing volume
   is a warning sign the move is running out of conviction, not a clean
   pullback-and-continue setup — this is exactly the distinction the
   indicator's own nonlinear boost is trying to surface.
4. **Existing Stage 0/Stage 1 gates unchanged** — Trend Template (daily
   structural quality) and the liquidity/anti-SAGILITY floor stay exactly
   as they are. These are the two checks doing the actual "avoid a
   catastrophic loss" work (a thin, structurally-weak name is dangerous
   regardless of how good the intraday entry signal is) — nothing about
   Option C changes that reasoning, so don't touch them.

## Scale-ins — the order-placement half of Option C

Allow **one** additional entry into an already-open symbol (cap: 2 total
entries per symbol, per side) if:
- A second retest fires while the regime is still intact for that symbol,
  AND
- The existing position is **currently favorable** (mark price ≥ its own
  entry price) — this is the load-bearing safety rule. **Never scale into
  a position that's currently underwater** — that's averaging down, the
  single most common way a "smart" pyramiding rule turns into a much
  bigger loser than a flat single entry would have been. Only pyramid into
  strength.
- Capacity allows it (`MAX_CONCURRENT_CE`/`_PE` counts each entry
  separately, not each symbol — two entries on one symbol consume two of
  the four/whatever slots).

**Each entry is tracked as its own fully independent sub-position** — own
entry price, own target/SL/highest_price/trailing state — rather than
averaging into one combined position. Simplest to reason about, and avoids
inventing new averaging-math that the existing `_exit_reason_for`/dynamic-
SL logic was never designed around.

**Shared regime-exit rule**: if the adaptive-band regime flips *against*
direction for a symbol, close **every** open sub-position for that symbol
immediately, regardless of each one's individual target/SL/PROFIT_
PROTECTION state. A regime flip means the premise every entry on that
symbol was made under (this trend is intact) is now false — waiting for
each sub-position's own price-based exit to catch up independently is
strictly worse than reacting to the regime signal directly, since the
regime flip is available as a first-class signal already.

## Cross-referencing the rest of this repo, per the user's instruction

- **`learnings/intraday-options-trading/greeks-and-decay.md`**: retest
  entries sit closer to the baseline than flip entries, meaning a smaller
  expected move is needed to reach target — shorter expected holding time
  reinforces why ATM (not ITM/OTM) stays correct: ATM's gamma/liquidity
  advantage matters more than ever when the intended hold is short, and
  theta bleed over a short hold stays small regardless.
- **`learnings/intraday-options-trading/liquidity-and-execution.md`**: this
  file already flagged that the premium/lot-size anti-SAGILITY proxy
  doesn't catch every thin-liquidity case (a specific strike can be thin
  even at a moderate premium/lot-size). **Given scale-ins increase total
  exposure per name, this gap matters more under this design than it did
  before** — recommend pulling a real OI/volume floor on the specific ATM
  contract (K01's own deferred Stage 2, needs the Dhan Option Chain API)
  forward in priority, ahead of building scale-ins, rather than after.
- **`learnings/intraday-options-trading/timing-patterns.md`**: the
  last-30-minutes window is flagged as weakest for a *new* entry. Recommend
  applying an entry-time cutoff specifically to retest/scale-in entries
  (e.g. no new entries after 15:00 IST) even though K01's current
  `ENABLE_TRADING_TIME_LIMIT`-equivalent isn't on for the live bot — a
  scale-in fired at 15:10 has little runway before `SQUARE_OFF_TIME`
  (15:15) to reach target, which works directly against the "fewer, better
  losers" goal.
- **`learnings/exit-mechanics.md`**: the shared regime-exit rule above is
  a direct application of this file's own finding that a signal-driven exit
  (Supertrend flip) can and should act independently of price-threshold
  exits, rather than waiting for MAX_LOSS_HIT/target to catch up.
- **`learnings/screener-analysis/dandanadan-2.md` /
  `kaashvi-28.md`**: both confirm (now with real backtest numbers) that
  *fewer, better-confirmed* entry conditions beat *more, weaker* ones on
  win rate and MAX_LOSS_HIT frequency. Retest-plus-strength-gate is exactly
  the same philosophy — trading entry frequency for entry quality — applied
  to K01 instead of to which Chartink screener to trust.

## What this design deliberately does NOT change

- Stage 0 (Trend Template) and Stage 1 (liquidity/anti-SAGILITY floor) —
  unchanged, still hard gates before a symbol is even watchlisted.
- `PAPER_TRADING_ONLY` — stays true; this whole design is for paper
  validation, per the user's own framing this session.
- Target/stop-loss/dynamic-SL/PROFIT_PROTECTION mechanics themselves — this
  design changes *when* an entry happens and *how many* can exist per
  symbol, not what closes a position once open.

## Before any code gets written

1. **Backtest first.** Per `learnings/backtest-methodology.md`'s standard
   workflow: implement the adaptive-band + retest + strength-gate logic as
   pure functions (mirroring `_compute_rsi`/`_compute_atr`/`_compute_roc`),
   then replay against real historical 5-min/1-min Dhan data for a
   multi-day window, comparing win rate / MAX_LOSS_HIT frequency /
   P&L-per-trade against the existing DanDanaDan-2 benchmark (77.4%/2/31,
   ₹1,063 per trade) before this ever touches `K01/paper_engine.py` for
   real.
2. **Tune the money-flow parameters for K01's actual bar sizes** — the
   indicator's own defaults (`mfLen=24`, `mfSmooth=5`, `mfPower=1.2`,
   `minMult=0.9`, `maxMult=2.2`) were not chosen for 5-min bars
   specifically; re-tune the same way `SUPERTREND_5MIN_PERIOD`/
   `_MULTIPLIER` were deliberately matched to production rather than
   inherited from a Chartink scan's own untuned defaults.
3. **Decide the OI/volume-floor question above** (pull Stage 2 forward or
   not) before scale-ins specifically, since that's the piece most directly
   addressing exposure risk once more than one entry per symbol is possible.

## Next step

Waiting on: which piece to prototype and backtest first (recommend the
adaptive-band + retest + strength-gate as a single-entry-per-symbol
backtest first — validate the entry-quality improvement in isolation
before adding scale-ins' extra complexity on top).
