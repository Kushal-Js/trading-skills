Status: DEPLOYED (partial/mixed) - two of the filters explored here are
live blocking gates today (volume floor; ADX/ER trend-strength re-entry
gate), one is live but shadow-mode-only forever so far (Efficiency Ratio
as a standalone gate), one was tested once and never carried forward
(Choppiness Index), and two were tested and explicitly rejected as
standalone blockers (RSI-extreme alone; ADX alone, mostly redundant with
the cooldown). See "What actually shipped" below for the current, real
state, and the Verification note at the bottom for what this write-up
could and couldn't independently re-check.

# Reversal / trend-strength filter arc (16-17 Sep 2026): 5 backtest rounds, from a bad morning to two live gates

This is the detailed record the existing single-line references in
[[luxury-signal-gated-live-simulation]] ("`LOSS_REENTRY_TREND_CHECK_
ENABLED`'s real ADX(>=20)/ER(>=0.3) check... genuinely gates the
1-prior-loss case") and `TRADING_JOURNAL.md`'s 17/18 Sep bundle rows were
pointing at without spelling out. Five backtest scripts, all in
`traderBoy` root, all read-only (only `dhan_wrapper.fetch_continuous_
intraday` / `_equity_security_id` calls, no order placement), ran in
sequence across roughly 13 hours on 16 Sep 2026, each one motivated by
what the previous round found. Two real, shipped commits followed the
next day.

## The trigger: 15-16 Sep real losses

Context transcribed directly in `backtest_reversal_filters_sep15_16.py`'s
own module docstring: a review of that morning's (16 Sep) live Options/
Futures/Luxury trades found a pattern worth prototyping filters against -
near-instant reversals right after entry (PAYTM Futures: entered 09:16:31
IST, MAX_LOSS_HIT 22 seconds later for -Rs4,603.75; YESBANK Options:
entered 09:16:16 IST, STOP_LOSS_HIT 61 seconds later for -Rs4,043.00 -
both durations independently verified against the script's own hardcoded
timestamps for this write-up) and Supertrend whipsaw in choppy conditions
(PETRONET Options: flat SUPERTREND_EXIT within ~33 seconds of entry,
re-entered, later stopped out for -Rs2,090.00). The user asked for 4
candidate filters to be prototyped and backtested against real trades
from both days.

Note on the docstring's own summary stat ("7 losses totaling -Rs19,837.55
against 4 wins totaling +Rs6,022.25, net -Rs13,815.30" for 16 Sep) versus
the script's actual hardcoded `RAW_TRADES` list: re-deriving the
16-Sep-only subset from `RAW_TRADES` as transcribed in the script gives 6
losses (-Rs18,157.55), 5 wins (+Rs8,118.55) and 1 flat trade (PETRONET,
Rs0.00) - a different split than the docstring's context line. This
write-up flags the discrepancy rather than silently reconciling it: the
"7L/4W" figure was very likely a quick manual count made in conversation
*before* the script was written (possibly over a slightly different or
broader trade set), while `RAW_TRADES` is the actual, more carefully
filtered dataset (`"reconciled": true` entries and Swing excluded) the
script went on to test against. Both 15+16 Sep combined in `RAW_TRADES`:
37 trades, net -Rs2,922.05 (10W/2 flat/13L on the 15th netting +Rs7,117.00,
5W/1 flat/6L on the 16th netting -Rs10,039.05).

## Round 1 - `backtest_reversal_filters_sep15_16.py` (16 Sep, 10:19)

**Method**: entry candle = the 5-min bar of the underlying stock at-or-
immediately-before the real entry time (matching what the live signal
itself reads), fetched via the same continuous multi-day series the live
bot's Supertrend/regime code uses. Four candidate filters prototyped:
- ADX(14) trend-strength gate: block if ADX < 20.
- RSI(14) exhaustion filter: block CE if RSI > 70, block PE if RSI < 30.
- Volume confirmation: block if entry-candle volume < 1.2x its 20-bar
  trailing average.
- Post-`SUPERTREND_EXIT` same-direction cooldown: block a same-symbol,
  same-direction re-entry within N minutes of a prior `SUPERTREND_EXIT`
  on that symbol (window swept at 1/3/5/10/15 minutes).

**Result** (per the commit that followed, `5291efa` - see Verification
note; this write-up could not re-run the script itself today): a volume
floor was the strongest individual filter, RSI-extreme alone was net
negative, and the two worst single-trade disasters (PAYTM, YESBANK) were
specifically found to have **abnormally HIGH volume, not thin volume** -
both had a genuine volume spike (98x and 24x the 20-bar average per the
follow-up round 2's own docstring) combined with an already-extreme RSI
reading, which is why a plain volume *floor* alone would have missed
them. That combination - RSI-extreme AND a volume spike >20x average -
is what round 2 goes on to formalize as a new 5th filter.

## Round 2 - `backtest_reversal_filters_15day.py` (16 Sep, 10:44)

Motivated directly by round 1's finding: plain "RSI extreme = block" had
a net negative effect because it also blocked genuine trend-continuation
winners that simply had a strong RSI reading from being in a real trend.
The combined condition (RSI extreme AND volume > 20x average) looked more
surgical on a hand-check of round 1's data - this round formalizes and
backtests it properly, at scale: every real Options/Futures/Luxury trade
in the trailing 15 calendar days (`history/2026-09-01_real_trades.log`
through `2026-09-16_real_trades.log`), loaded programmatically instead of
hand-transcribed - 139 trades versus round 1's 37.

Swing is explicitly excluded, and the docstring says why: every Swing
record in that window is either the retired v1 "basket hedge" design
(exit reasons that don't exist in the current v2 rewrite), a
`reconciled: true` restart-recovery record with no fresh entry signal to
test, or an MCX (COPPER) trade needing a separate futures data-fetch path
this script doesn't implement - nothing comparable to include.

Five filters tested: the original 4 from round 1, plus the new **RSI-
extreme + volume-spike "climax combo"** (>20x average). Per `5291efa`'s
commit message (the actual quoted net-effect numbers from this round's
run): volume floor alone **+Rs17,123.00** net across the 15-day sample
(vs +Rs7,131.50 in round 1's smaller 2-day sample - the two figures
`5291efa` cites together as "the single strongest filter" across both
rounds); the 10-minute cooldown had **zero forgone gains** in either
round while directly targeting the flat-exit/immediate-re-entry/stop-out
whipsaw pattern. Round 2's own script comment additionally summarizes
that ADX(14)<20 overlaps almost entirely with the cooldown filter's own
benefit - i.e. once the cooldown is in place, ADX doesn't add much more
on top of it as a standalone gate.

## Round 3 - `backtest_trend_strength_indicators.py` (16 Sep, 20:36)

Run after `19c551d` (volume floor promoted to a live gate that evening,
19:50 IST - see below) had already shipped. The open question this round
asks: ADX answers "trending vs. choppy," but so do two purpose-built
indicators constructed differently - does either add anything ADX
doesn't already capture?

- **Choppiness Index** (E.W. Dreiss, period 14): `100 * log10(sum(TR,n) /
  (highest_high(n) - lowest_low(n))) / log10(n)`, bounded 0-100. HIGH =
  lots of true-range covered for little net progress = choppy. Direction-
  agnostic, unlike ADX.
- **Kaufman Efficiency Ratio** (period 10): `|close[i]-close[i-n]| /
  sum(|close[j]-close[j-1]|)` over the window, bounded 0-1. Near 1 =
  every bar's move contributed to net direction (efficient/trending);
  near 0 = motion without progress (noisy).

Rather than trusting textbook default thresholds (61.8/38.2 for CI, 0.3
for ER - tuned on decades-old US equity/futures data), the script sweeps
a threshold range for each (CI: 55/58/61.8/65/70; ER: 0.2 through 0.9 in
0.05 steps) against the same 15-day, 139-trade Options/Futures/Luxury
pool from round 2, reporting empirical net P&L per threshold rather than
assuming the textbook number applies to this bot's 5-min NSE-options
entries.

**Result**: not independently re-run this session (see Verification
note), but round 4's own docstring quotes this round's headline number
directly: ER's standalone net effect at its **in-sample-best** threshold
was **+Rs23,504.30 at ER<0.45**, on a baseline of 143 trades (a slightly
larger evaluated count than round 2's 139, presumably reflecting more
symbols clearing the warmup-bar minimum for ER's shorter 10-period
lookback versus ADX's ~28-bar one). Round 4's docstring is explicit that
0.45 is "shown for reference only - NOT recommended as a real threshold"
because it was optimized on the same data used to score it (in-sample
overfitting) - the deployed threshold that followed is the more
conservative **ER < 0.3**. Choppiness Index does not reappear in any
later round or in any of the 5 downstream commits below; it was tested
once and, as far as this write-up can verify, implicitly dropped in
favor of ER without an explicit stated reason in any script or commit -
flagging this as an inference, not a confirmed decision.

## Round 4 - `backtest_er_vs_volume_floor.py` (16 Sep, 22:59)

Directly answers the open question round 3 left hanging: ER's standalone
net effect (+Rs23,504.30 at ER<0.45) and round 2's "recommended combo"
net effect (+Rs24,928.75, per this round's own docstring) were both drawn
from the *same* 15-day pool - unknown whether ER was catching the same
trades the existing filters already caught (redundant) or different ones
(additive). Method: compute both the volume ratio (identical formula to
the live gate) and ER for every trade, then compare (1) baseline, (2)
volume floor alone at the live threshold (1.2), (3) ER alone at 0.3
(conservative) and 0.45 (in-sample-best, reference only), (4) volume
floor OR ER combined at both ER thresholds, and (5) an explicit overlap
breakdown - of what ER blocks, how much does the volume floor already
catch too versus how much is unique to ER.

**Result**: the standalone-ER-at-0.3 number and the incremental-over-
volume-floor number both surface in the very next day's commit message
(`b43a873`, see below) rather than in this write-up's own re-run: ER
alone at the conservative 0.3 threshold was net **+Rs10,925.75** on 143
trades, with a real **+Rs5,561.25** incremental benefit on top of the
already-live volume floor - but "the evidence is thin and lumpy": 85% of
that incremental benefit came from a single day, and two other days had
ER cost money by blocking real winners. This is the finding that kept ER
shadow-mode-only rather than promoted to a live blocking gate (see
below).

## Round 5 - `backtest_er_daywise_with_swing_check.py` (16 Sep, 23:09)

Two purposes: (1) break round 4's one aggregate number into a day-by-day
table (baseline / volume-floor-net / ER-net / combined-net /
incremental per day) - exactly the granularity that surfaced the "85% of
the benefit is one day" finding `b43a873` cites; (2) rather than silently
repeating rounds 2-4's Swing exclusion, actually load every Swing record
in the 15-day window and classify *why* each one is or isn't usable
(`reconciled: true`; old v1 basket-hedge exit-reason taxonomy; or a
genuine fresh v2-era trade, reported separately if any exist and flagged
- not silently mishandled - if any turned out to be an MCX symbol needing
a contract-resolution path this script doesn't implement).

**Result**: not independently re-run this session. The per-day P&L
breakdown that would confirm the "one day drove 85% of ER's incremental
benefit" claim, and the outcome of the Swing inclusion check specifically
from this script's run, are not verified by this write-up - see
Verification note. (`416b3ff`, the next morning's commit, does add Swing
shadow-mode ER/ADX logging rather than a live Swing ER gate, consistent
with round 5's stated purpose of checking Swing inclusion before doing
anything further with it - but that specific causal link is this write-
up's inference from timing and content, not a commit message that cites
round 5 by name.)

## What actually shipped

Five commits followed, in this order, each traceable to a specific
round's finding by content (commit messages quote the round's own numbers
directly, not just by proximity in time):

| Commit | When | What | Traces to |
|---|---|---|---|
| `5291efa` | 16 Sep, 11:23 IST | Shadow-mode-only logging of ADX/RSI/volume-ratio/climax-combo/cooldown for every real Options/Futures/Luxury entry (`reversal_filters.py`, `history/<date>_reversal_filter_shadow.log`). Never blocks. | Rounds 1+2 (commit message quotes both rounds' volume-floor numbers, +Rs7,131.50 and +Rs17,123.00, and the PAYTM/YESBANK climax-combo finding by name) |
| `19c551d` | 16 Sep, 19:50 IST | Volume floor promoted from shadow-mode to a **live blocking gate**: `VOLUME_FLOOR_GATE_ENABLED`/`VOLUME_FLOOR_RATIO_MIN=1.2` for Options (explicit user request) and Swing/MCX symbols only. Fails open on auth/fetch failure. | Rounds 1+2 (same two cited numbers as `5291efa`, explicitly "the single strongest individual filter across two backtest rounds") |
| `b43a873` | 17 Sep, 00:23 IST | Kaufman Efficiency Ratio added as a **shadow-mode-only** logging field (never blocks), threshold deliberately 0.3 not the in-sample-best 0.45. | Rounds 3+4 (commit quotes the 143-trade, +Rs10,925.75-alone / +Rs5,561.25-incremental numbers and the "thin and lumpy... 85% from a single day" caveat verbatim) |
| `416b3ff` | 17 Sep, 08:10 IST | Futures gets the same live volume-floor gate Options already has (port, same 1.2 threshold). Swing gets ADX/RSI/volume-ratio/ER shadow-mode logging on every real entry (not yet a gate). | Volume floor: same evidence as `19c551d`, ported to a second package. Swing logging: likely motivated by round 5's Swing-inclusion check, not confirmed by commit-message citation - see round 5's own note above |
| `d758e6b` | 17 Sep, 21:30 IST (journal's own bundle groups the live deploy under "18 Sep") | Real incident (ATHERENERG 29 SEP 1540 PUT, Options, 17 Sep): SUPERTREND_EXIT loss then a same-day re-entry loss, because neither exit reason was in `LOSS_REPEAT_BLOCK_EXIT_REASONS` at the time. Two changes: broadened loss-repeat blocking to any real loss regardless of exit reason, and added `check_trend_strength` - a **live** re-entry gate requiring ADX>=20 **or** ER>=0.3 (either one, not both) before a symbol that already lost money today can re-enter. `LOSS_REENTRY_TREND_CHECK_ENABLED`, default on, across Options/Futures/Luxury. | Rounds 1-4 directly: reuses `ADX_MIN`/`ER_THRESHOLD` "rather than new, untested numbers" (commit message's own words), and the code comment at `reversal_filters.py:540-562` explicitly cites ATHERENERG's own logged entry-time ADX (13.24, "well below ADX_MIN... a genuinely choppy reading") as the incident the gate targets |

Deployed constants today, confirmed directly in `reversal_filters.py`
(lines 101-107) and each package's `config.py`: `ADX_MIN = 20.0`,
`RSI_OVERBOUGHT/OVERSOLD = 70.0/30.0`, `SPIKE_RATIO = 20.0`,
`COOLDOWN_MINUTES = 10`, `ER_THRESHOLD = 0.3`, `VOLUME_FLOOR_RATIO_MIN =
1.2` (Options/Futures/Swing-MCX, and Swing-NSE at the same default -
though Swing's NSE floor was independently lowered to 0.6x on 22 Sep for
unrelated reasons; see `TRADING_JOURNAL.md`'s 22 Sep entry, not part of
this arc).

**What never became a live blocking gate**: RSI-extreme alone (round
1/2 found it net negative - too blunt), the climax combo (RSI-extreme +
volume spike - stays a shadow-log field, never wired as its own block
even though it explains the arc's two worst trades), plain ADX as a
standalone per-entry gate (found to mostly overlap the cooldown filter's
benefit), the cooldown itself as a standalone gate (folded into shadow
logging only, not deployed as its own block), Choppiness Index (tested
once, never reappears), and Efficiency Ratio as a *general* per-entry
gate (only ever promoted as one half of the *re-entry-after-a-loss*
check in `d758e6b`, specifically because its evidence alone was judged
too thin and lumpy for a standalone gate per `b43a873`'s own words).

## Verification note (what this write-up could and couldn't check)

- **All 5 scripts were read in full** (docstrings, methodology,
  filter/threshold definitions, data-loading logic) - the round-by-round
  narrative and thresholds above come directly from that reading, not
  from guessing off filenames.
- **None of the 5 scripts could be executed this session.** Every one
  authenticates via `dhan_wrapper.authenticate()`, which as of `c61e82e`
  (17 Sep, after this arc) now refuses local `pin_totp` auth outright -
  see [[ws-candle-reconstruction-parity-results]]'s sibling incident,
  `incidents/2026-09-21-local-backtest-dhan-session-collision.md`: a
  local backtest re-authenticating with the live droplet bot's own
  credentials mints a new Dhan session token and silently kicks the live
  bot's session, which is exactly what happened once already. Bypassing
  that guard needs either a hand-off access token from the user or
  `ALLOW_LOCAL_PIN_TOTP=true` plus independent confirmation the live bot
  isn't currently running - neither was obtained for a documentation-only
  task, consistent with this repo's own live-trading-safety standing
  rule. This means every numeric result attributed to rounds 1-5 above
  comes from either (a) the hardcoded trade data / reversal-timing facts
  directly visible in the scripts' own source (independently re-derived
  and double-checked by this write-up, e.g. the PAYTM/YESBANK durations
  and the 15-day `RAW_TRADES` win/loss split above), or (b) numbers a
  *later* round's or commit's own docstring/message quotes from an
  earlier round's actual run output - not numbers this write-up computed
  or invented itself.
- **Separately, rounds 2-5 share a reproducibility bug worth flagging**:
  `backtest_reversal_filters_15day.py`'s `HISTORY_DIR` constant is
  hardcoded to a specific prior Claude Code session's scratchpad path
  (`/private/tmp/claude-501/.../60a0e686-.../scratchpad/history`), not
  `traderBoy/history` in the repo itself. That scratchpad directory still
  exists on this machine but is now empty (confirmed via `ls` this
  session) - even with valid auth, re-running rounds 2-5 today would load
  zero trades, not the real 139-143-trade sample the original run used.
  This looks like an unintentional hardcode (should very likely be
  `REPO_ROOT / "history"`, matching round 1's own relative `history/*.log`
  reads) rather than a deliberate scoping choice - worth a real fix if
  these scripts are ever rerun, but out of scope for this write-up
  (documentation-only, no `traderBoy` changes made).
- The commit-message-cited numbers this write-up leans on (`5291efa`,
  `19c551d`, `b43a873`, `d758e6b`) are treated as trustworthy because
  they were written by whoever actually ran the scripts with working
  auth, in the same session that produced the code - but they are still
  secondhand to this write-up, not independently reproduced here. If the
  scripts are ever fixed and rerun, treat any number above as worth
  reconfirming rather than assuming it will reproduce exactly (real
  intraday candle data for a given historical window can itself shift
  slightly between fetches near data-vendor boundaries, per this repo's
  own `learnings/backtest-methodology.md`).

## Caveats

- This whole arc is a single 15-16-day historical window (31 Aug - 16
  Sep, expanding through the rounds), same single-stretch-of-market
  caveat as every other backtest in this line of work - none of it
  generalizes to a different volatility regime without rechecking.
- All "net effect" numbers throughout are the standard avoided-losses-
  minus-forgone-gains framing this codebase's backtests use - a filter
  that "wins" net can still have blocked some real winning trades along
  the way; see each round's own blocked-trade tables (not reproduced here
  since this write-up couldn't re-run them) for the individual tradeoffs.
- The two gates that did ship (volume floor; ADX/ER re-entry) both fail
  OPEN on auth/fetch failure or insufficient data - a confirmed thin
  reading blocks, an unknown reading never does. This is a deliberate,
  repeated design choice across every gate in `reversal_filters.py`, not
  specific to this arc, but worth restating since it means these gates
  provide zero protection during exactly the conditions (feed problems,
  auth hiccups) where extra protection might matter most.
