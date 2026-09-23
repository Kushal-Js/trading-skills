# 2026-09-23: BANDHANBNK CE stopped out in 70 seconds - dynamic-SL was over-sensitive, not the underlying "recovering"

## What triggered this

User noticed BANDHANBNK had recovered and asked why the bot still took a
loss on it. Pulled the real trade: Luxury, `BANDHANBNK 29 SEP 190 CALL`,
entered 09:22:14 IST @ Rs4.22, exited 09:23:24 IST @ Rs3.56 -
**70 seconds later** - `exit_reason=TRAILING_SL_HIT`, qty 3600, -Rs2,376.

The entry itself landed on a climactic spike: the shadow-mode diagnostic
log for this exact order shows `RSI=89.09` (deep exhaustion territory),
`volume=5.35x` average, `ADX=39.04`. The shadow filter even flags
`rsi_extreme_alone_blocks: true` for this trade - i.e. it *would* have
been blocked if that filter were live. It isn't (diagnostic-only, see
below for why it should stay that way).

## Finding 1: RSI-extreme-alone does NOT discriminate winners from losers

Joined 6 days of `reversal_filter_shadow.log` (16, 17, 18, 21, 22, 23
Sep) against `real_trades.log` by `order_id` - 102 matched trades across
Options/Futures/Luxury. Segmented by every shadow flag:

| Flag | True: win rate / avg PnL | False: win rate / avg PnL |
|---|---|---|
| `rsi_extreme_alone_blocks` | n=37, 30%, -Rs560 | n=65, 29%, -Rs334 |
| `adx_blocks` | n=24, 25%, -Rs428 | n=78, 31%, -Rs412 |
| `volume_blocks` | n=51, 29%, -Rs481 | n=51, 29%, -Rs350 |
| `er_blocks` | n=16, 25%, -Rs667 | n=82, 29%, -Rs370 |
| `recommended_combo_blocks` | n=52, 29%, -Rs498 | n=50, 30%, -Rs330 |

None of these separate winners from losers in this sample - if
anything, `rsi_extreme_alone_blocks=True` trades were slightly WORSE on
average than the unflagged ones, the opposite of what the single
BANDHANBNK anecdote suggested. **Conclusion: RSI-89-at-entry looked like
an obvious red flag in isolation, but it doesn't generalize** - same
class of result as the earlier ER-filter work
([[reversal-trend-strength-filter-arc]]) and the COPPER breakout-
strength filter ([[copper-breakout-strength-filter-rejected]]): a
single bad trade's own postmortem features rarely hold up as a blanket
rule once checked against the wider sample. Not promoted to a live gate
- stays shadow-only.

(Aside, not the focus of this doc: overall win rate across the full
joined sample was ~29-30% with total PnL solidly negative - Options
n=29/31%/-Rs24,593, Futures n=50/36%/-Rs15,895, Luxury n=84/36%/-Rs15,240,
Swing n=18/50%/-Rs8,362. Worth its own separate look at some point, not
investigated further here.)

## Finding 2: TRAILING_SL_HIT was 100% losses across the whole window - the mechanism never once protected profit

Every single `TRAILING_SL_HIT` exit in the 6-day window, any package,
any option type:

| Strategy | Symbol | Type | PnL |
|---|---|---|---|
| Options | MAZDOCK | PE | -Rs2,261 |
| Options | HCLTECH | PE | -Rs1,440 |
| Luxury | BLUESTARCO | CE | -Rs1,154 |
| Luxury | SWIGGY | CE | -Rs1,551 |
| Luxury | PGEL | PE | -Rs1,900 |
| Luxury | PATANJALI | CE | -Rs1,613 |
| Luxury | BANDHANBNK | CE | -Rs2,376 |

7/7 losses, -Rs12,295 total. The CE-side four (the ones this fix
targets) all lost **-15.6% to -17.3%** from entry to exit - essentially
the same magnitude as (BLUESTARCO/BANDHANBNK) or WORSE than
(SWIGGY/PATANJALI, likely real slippage on a fast market-order fill
through a thin book) Luxury's own plain `STOP_LOSS_PCT=0.16` hard stop
would have produced on its own. Not one of the seven captured any of the
gain that triggered the "trailing" label in the first place.

**Root cause**: `DYNAMIC_SL_STEP_PCT_CE=0.07` (the code default,
inherited by Options/Futures/Luxury alike - only Luxury's PE side had
ever been separately tuned, to 0.09, 1 Sep 2026) only requires a 7%
premium move before the stop starts tightening, and
`DYNAMIC_SL_INCREASE_PCT=0.01` only tightens it by 1% per step. On an
option premium - which can move several multiples faster than the
underlying in percentage terms, especially in the first minutes of the
session - a 7% move is well within ordinary noise, not evidence of a
real trend. BANDHANBNK's entire round trip (spike past the 7% trigger,
then reverse hard enough to blow through even the tightened floor) took
under 2 minutes. The mechanism was firing on volatility, not signal, and
even when it fired "correctly" the 1%-per-step tightening was too small
to meaningfully improve on the base stop anyway.

## Fix

`LUXURY_DYNAMIC_SL_STEP_PCT_CE` raised 0.07 -> 0.20 (`.env`, not yet
deployed - staged for the next scheduled restart, see
[[project-dhanboy-deployment]]'s checklist). Requires the premium to be
up MORE than the 16% base hard-stop's own width before dynamic
tightening can trigger at all - a standard trailing-stop principle
(only start protecting a gain once you're already ahead by more than
your worst-case risk), and directly targets the observed failure mode:
none of the 4 real CE losses in this sample would have been made worse
by waiting for a bigger move first, since the tightened floor never
beat the base stop in any of them anyway.

**Scope**: Luxury CE only, matching what was asked and where the
incident happened. Options showed the identical 100%-loses pattern
(MAZDOCK/HCLTECH, PE side) in the same window - not changed here,
flagged as a natural follow-up if it keeps showing up.

**Caveat, stated plainly**: `real_trades.log` doesn't persist
`highest_price`/peak-price per trade, so this couldn't be backtested
against the exact historical spikes that triggered each of these 7
exits - 0.20 is a principled default reasoned from the base stop's own
width, not a swept-and-optimized value from real peak data. If this
matters enough to get right precisely, the real fix is logging
`highest_price` (or the specific dynamic-SL step count) onto the closed
trade record going forward, so a proper step-size sweep becomes
possible. Revisit once a few weeks of real `TRAILING_SL_HIT` outcomes
have accumulated under 0.20.
