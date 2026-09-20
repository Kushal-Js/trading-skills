Status: CODE COMPLETE, UNCOMMITTED in traderBoy (dhanBoy branch) - flag
`BUCKET_SWITCH_ENABLED` defaults true in Options/Futures/Luxury but nothing
has been pushed or deployed to the droplet yet. Backtested 20 Sep 2026,
net result modestly POSITIVE but Options alone is negative and the sample
is thin (90 switch events across 14 days) - awaiting the user's decision
on whether/how to deploy.

# Loss-triggered alert-bucket switch

## Design (user request 19 Sep 2026)

Two shared, restart-persistent buckets (CE/PE) record every alert that
hits Options/Futures/Luxury today, whether or not it was traded. A
background ranker keeps every bucket symbol scored with the MA-ribbon
"burst possibility" score ([[ribbon-switch-shadow]]'s `ribbon_score.py`).
When a real open position's unrealized loss reaches
`config.BUCKET_SWITCH_LOSS_RS` (600, configurable per package - raised
from an initial 500 to match the user's explicit spec), the owning
package's poll loop calls `alert_bucket.maybe_switch()`: it picks the
best-ranked, not-currently-held bucket symbol of the same option type
scoring >= `BUCKET_SWITCH_MIN_SCORE` (50), **enters it first** through the
package's own real entry path (every existing guard still applies -
capacity is allowed to exceed the cap by exactly one for this call), and
only then exits the losing position (`exit_reason="BUCKET_SWITCH"`).
Enter-first is deliberate: if no candidate gets through, nothing changes -
the loser just keeps running under its normal exit ladder, so a failed
replacement can never leave the book flat for nothing.

Implementation: `traderBoy/alert_bucket.py` (buckets + ranker + switch
logic), wired into `Options|Futures|Luxury/{config,trading_engine,
position_store,*_main}.py` and `main.py`. Persistence: `history/<date>_
alert_bucket_{CE,PE}.json`, rewritten atomically, reloaded at startup -
survives a mid-day restart. Not wired into Swing.

This is a DIFFERENT, more aggressive mechanism than
[[ribbon-switch-shadow]]'s `decide_switch()` (which only reallocates an
already-full capacity slot for a *materially* better-scoring candidate,
with a 15-point margin and 15-minute hold grace) - this one fires purely
off a fixed rupee loss on the held position, regardless of capacity
headroom or the loser's own current trend health.

## Backtest methodology (14 days: 31 Aug - 18 Sep 2026, all days with
webhook_alerts.log data available; "last 15 days" was requested but only
14 exist)

Script: `traderBoy/backtest_bucket_switch.py`. For each real losing trade
(Options/Futures/Luxury; Swing excluded):

1. Reconstruct that day's CE/PE buckets from `webhook_alerts.log`. The raw
   log never records which endpoint (buy/sell) received an alert, so
   option_type was recovered via a `(strategy, scan_name)` -> option_type
   table built empirically by matching each alert to a same-symbol real
   position opened within 90 seconds. **100% coverage, zero disagreement**:
   every one of the 2,519 Options/Futures/Luxury alerts in this window
   mapped to exactly one option_type across 511 matched alerts.
2. Walk the losing trade's own real historical 1-min option-premium path
   (Dhan `intraday_minute_data` - the contracts' expiry, 29 SEP 2026, is
   still active so every trade in this window resolves against the
   CURRENT instrument master) to find the first minute unrealized loss
   actually reached Rs 600. A trade that never crosses is unaffected
   either way and excluded from the delta (19 of 109 crossing events also
   ended up here because no eligible candidate existed at the crossing
   moment - correctly treated as "no switch, baseline unchanged").
3. At that minute, score every eligible bucket candidate (same
   option_type, alerted earlier, not held by ANY of the 3 strategies per
   `real_trades.log`'s own opened_at/closed_at intervals) with the real
   `ribbon_score` functions on a genuine 7-calendar-day 5-min series
   ending at that minute (matches `alert_bucket._score_symbol_sync`'s own
   lookback exactly).
4. If a candidate scores >= 50: resolve its real ATM contract at that
   moment's spot price, enter at the next 1-min candle's open, and run a
   SIMPLIFIED target/hard-stop/EOD-only exit ladder forward on its own
   real historical price path.

### A real bug caught before trusting the first result

The first pass sized the replacement leg using the LOSING position's own
share-quantity. Every option contract has a different real F&O lot size,
and reusing the loser's share-count silently mis-sized the replacement -
e.g. CHOLAFIN (lot 625) -> FORCEMOT (lot 25) is a 25x mismatch. This
single bug dominated the first result: total delta swung from **-Rs
148,969** (looked strongly negative) to **+Rs 17,060** (modest positive)
after fixing the replacement leg to resize to `lot_size *
config.QUANTITY_LOTS` exactly like `Options/trading_engine.py`'s own
`enter_positions_for_stocks` does. The worst individual deltas in the
buggy run were all +-Rs 15k-62k on exactly these lot-mismatched pairs;
after the fix the worst individual delta anywhere is ~Rs 9.3k, in line
with the rest of the dataset. **Lesson for any future backtest touching a
replacement/switch mechanic in this codebase: always resize a NEW
instrument to its OWN real lot, never reuse another instrument's
share-count** - the same class of mistake [[opening-burst-slot-and-sl-
target-sensitivity]]'s own two bugs warns about (silently wrong until
cross-checked against a sanity bound).

## Results (corrected)

| Strategy | Crossing events | Switched | Baseline PnL | Switch-scenario PnL | Delta |
|---|---|---|---|---|---|
| Options | 48 | 35 | -69,391.25 | -83,789.75 | **-14,398.50** |
| Luxury | 46 | 42 | -63,926.75 | -38,976.95 | **+24,949.80** |
| Futures | 15 | 13 | -24,339.05 | -17,830.05 | **+6,509.00** |
| **Total** | **109** | **90** | **-157,657.05** | **-140,596.75** | **+17,060.30** |

Of the 90 actual switches, only **43 (47.8%) improved on baseline** - the
net positive comes from win/loss magnitude asymmetry (best switch +9,332,
worst -4,900), not from a real edge in picking winners. 174 total losing
trades existed in the window; only 109 (63%) ever reached the Rs 600
threshold at all, and 19 of those had no eligible bucket candidate at the
crossing moment (left unaffected, matching the enter-first design).

**Options is the one strategy where this backtest says switching would
have made things WORSE, not better** - opposite sign from Luxury/Futures.
With only 35 switch events for Options specifically, this could be sample
noise rather than a real strategy-specific effect; not enough evidence
either way to explain it causally yet.

## Caveats (read before deciding whether to deploy)

- The replacement leg's exit ladder is target/hard-stop/EOD ONLY - no
  dynamic trailing SL, Supertrend/EMA-cross exit, or liquidity guard, all
  of which exist in real production and would likely cut both winners and
  losers earlier/cleaner than a raw stop. Net effect on the reported
  numbers is unknown (could cut either side of the distribution).
- Candidate selection takes the single best-scoring eligible candidate
  only. Production tries up to 3 in score order and can fail an entry
  gate (RSI/trend, volume floor, liquidity, funds, cross-strategy claim) -
  none of those gates are replicated here, so real-world switches may
  sometimes fall through to a worse candidate or fail entirely where this
  backtest assumes success.
- `BUCKET_SWITCH_MAX_PER_DAY`/retry/candidate-block brakes are not
  simulated - every qualifying crossing is treated as an independent
  opportunity, which could overstate switch volume on a day with several
  losing trades relative to the real per-day cap.
- PE-side scoring (`score_ribbon_breakdown`) carries the same
  unproven-in-production caveat `ribbon_score.py` itself states.
- 14 days / 90 switch events is a thin sample for a feature this
  consequential - directional at best, same bar this repo already holds
  other backtests to before recommending a flip to live.

## What still needs a decision before going live

Per [[project-dhanboy-deployment]]/live-trading-safety practice: this is
uncommitted, undeployed code. The user needs to decide, given a modest net
positive with Options going the other way: deploy all 3 as-is, deploy only
Luxury+Futures and hold Options, gather more days of data first, or not
deploy. Full per-event detail (all 109 events) is in
`traderBoy/history/bucket_switch_backtest_events.json` (git-ignored, kept
local) for anyone who wants to dig into individual switches.
