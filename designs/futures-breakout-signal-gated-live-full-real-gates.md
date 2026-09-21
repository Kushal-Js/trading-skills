# Futures + breakout-signal gate, full real-gate replication (21 Sep 2026, user request)

Follow-up to [[futures-updated-risk-config-15day-backtest]]: the user
correctly flagged that the config-raise backtest's improvement (real
-Rs11,943.55 -> simulated -Rs714.75) looked too small to trust, since
that simulation was explicitly missing three real gates (liquidity
guard, gap-down CE delay, cross-strategy lock). This run adds all three
back AND replaces Futures' real alert-driven entries with the SAME
breakout-screener signal gate currently deployed live on Luxury
(clearance=0.3%, body=0.5%, relvol=1.2x - the sweep's #1 combo, see
[[luxury-breakout-detection-parameter-sweep]]), mirroring exactly how
[[luxury-signal-gated-live-simulation]] tested this hypothesis for
Luxury.

## What changed vs. the prior Futures backtests

New script: `backtest_futures_breakout_signal_gated_live.py`. Read
`Futures/trading_engine.py`'s real `_process_one_entry` and
`_exit_reason_for` directly (not assumed/copied from the older range-
breakout script) to get the exact gate order and logic:

| Gate | Prior Futures backtests | This run |
|---|---|---|
| Signal source | Real Chartink alerts (ranked batch) | Breakout-screener signal (per-symbol, same as Luxury's real `_breakout_entry_fn` - bypasses batch ranking entirely) |
| Liquidity guard (`get_liquid_atm_option`) | Not modeled | **Modeled** - zero-volume-bar streak + prior-session-volume check, with strike substitution |
| Gap-down/sharp-fall Nifty CE cool-off | Not modeled | **Modeled** - real Nifty50 historical 1-min series (security_id "13"), reconstructs `evaluate_nifty_open_condition`/`should_delay_ce_entry` exactly per alert day |
| Cross-strategy lock | Not modeled | **Modeled** - same interval-based proxy Luxury's own script uses (real trade history of Options/Luxury/Swing) |
| RSI-loss-reentry block | Approximated (older script) | Re-verified against `dhan_client.is_rsi_loss_reentry_blocked`'s real condition (`rsi > 88 OR rsi < prev_rsi`, only checked after a MAX_LOSS_HIT specifically) |
| Loss-repeat block counting | Assumed reason-restricted | **Corrected** - real code counts ANY monetary loss today, not just MAX_LOSS_HIT/STOP_LOSS_HIT (`LOSS_REPEAT_BLOCK_EXIT_REASONS` is vestigial/unused for the count itself, confirmed by reading the 18 Sep 2026 code comment) |
| EOD square-off | Not modeled | **Checked live**: `FUTURES_ENABLE_SQUARE_OFF=false` in the current `.env` (only `FUTURES_ENABLE_FRIDAY_SQUARE_OFF=true` applies) - Futures does NOT force-close daily at 15:15 as the code's own default would suggest; only Fridays. Modeled accordingly. |
| Risk config | Real trade history = pre-raise; simulation = post-raise (mixed) | Uses the current live config throughout (post-21-Sep raise: MAX_LOSS 5,500/3,100 CE, PP 5,000/2,500, giveback 8%) |

## Result

| | Trades | Wins | Win Rate | Total PnL |
|---|---:|---:|---:|---:|
| REAL Futures (actual, pre-raise config, real alerts) | 50 | 19 | 38.0% | -Rs11,943.55 |
| **Breakout-signal-gated, full real gates, new config** | 8 | 7 | **87.5%** | **+Rs13,359.10** |
| Delta | | | | **+Rs25,302.65** |

Raw breakout signals found across Futures-alerted symbol-days: **13**
(far fewer than Luxury's 72 over the same window - Futures receives
alerts for a smaller, different symbol set and only ever trades CE).
Of those 13, **5 were filtered out**: 4 by the volume-floor gate, 1 by
the Nifty gap-down cool-off. **8 entered, 7 won.**

### Why this result differs so much from the prior (incomplete) backtest

The earlier [[futures-updated-risk-config-15day-backtest]] used real
Chartink alerts (45 entered trades, near-breakeven -Rs714.75) - a much
larger, noisier trade set missing 3 real gates. This run uses a much
SMALLER, more selective signal source (13 raw signals vs however many
raw alerts fed those 45 trades) that already filters for a confirmed,
volume-backed breakout before an entry is even attempted - the same
effect the Luxury version of this same hypothesis showed (88.9% win
rate on 18 trades vs Luxury's real 36.0% on 125). Adding the liquidity
guard and gap-down gate on top removes a further 5 of the 13. Fewer,
higher-quality signals is the entire premise of the breakout gate, not
a modeling artifact - but see the caveats below on sample size.

### Trade-wise (8 trades)

| Day | Symbol | Signal Time | Entry | Exit | Exit Reason | Qty | PnL |
|---|---|---|---:|---:|---|---:|---:|
| 31 Aug | CAMS | 11:55 | 24.25 | 25.90 | PROFIT_PROTECTION_HIT | 825 | +1,359.60 |
| 01 Sep | HCLTECH | 09:20 | 30.80 | 36.96 | TARGET_HIT | 400 | +2,464.00 |
| 11 Sep | PAYTM | 09:50 | 49.95 | 59.94 | TARGET_HIT | 725 | +7,242.75 |
| 16 Sep | PAYTM | 09:15 | 84.15 | 76.56 | **MAX_LOSS_HIT** | 725 | **-5,500.00** |
| 16 Sep | YESBANK | 09:15 | 0.45 | 0.54 | TARGET_HIT | 31,100 | +2,799.00 |
| 17 Sep | TATASTEEL | 14:40 | 3.01 | 3.66 | TARGET_HIT | 2,750 | +1,787.50 |
| 18 Sep | DRREDDY | 09:15 | 11.05 | 13.26 | TARGET_HIT | 625 | +1,381.25 |
| 18 Sep | JUBLFOOD | 09:30 | 7.30 | 8.76 | TARGET_HIT | 1,250 | +1,825.00 |

The single loss (PAYTM, 16 Sep) landed at exactly -Rs5,500 - the new
MAX_LOSS_HIT cap, same precise price-level-conversion validation seen
in every other backtest in this line of work.

### Day-wise

| Day | Trades | W/L | PnL |
|---|---:|---|---:|
| 31 Aug | 1 | 1W/0L | +1,359.60 |
| 01 Sep | 1 | 1W/0L | +2,464.00 |
| 11 Sep | 1 | 1W/0L | +7,242.75 |
| 16 Sep | 2 | 1W/1L | -2,701.00 |
| 17 Sep | 1 | 1W/0L | +1,787.50 |
| 18 Sep | 2 | 2W/0L | +3,206.25 |
| **Total** | **8** | **7W/1L** | **+13,359.10** |

### Skip reasons (13 raw signals -> 8 entered)

| Reason | Count |
|---|---:|
| volume_floor_gate | 4 |
| nifty_gap_down_ce_delay | 1 |

No signals were blocked by the daily re-entry cap, RSI-loss-reentry
block, loss-repeat block, cross-strategy claim, capacity, or liquid-
contract resolution in this sample - the volume floor and gap-down gate
did all the filtering here.

## Caveats

- **8 trades is a very small sample** - smaller than any other backtest
  in this line of work. A single trade (PAYTM's -Rs5,500 loss) is 41%
  of the total by itself; the single largest winner (PAYTM +Rs7,242.75)
  is 54%. This result is far more sensitive to one or two trades
  turning out differently than the 45/50/59/125-trade samples elsewhere
  in this series.
- **Not a clean isolate of any one variable** - this run changes THREE
  things at once relative to the real baseline: (1) the entry signal
  source (breakout screener vs. raw Chartink alerts), (2) the risk
  config (post-raise vs. pre-raise), and (3) backtest completeness
  (adds liquidity/gap-down/cross-strategy gates the earlier Futures
  backtest didn't have). The delta is the combined effect of all three,
  not attributable to any single change.
- Funds check still not modeled (fails open, matching production's own
  fail-open behavior under a live infrastructure failure - not a new
  simplification).
- Cross-strategy lock is still the same interval-based proxy (real
  trade history of the OTHER strategies), not the real momentary
  `cross_strategy_registry` claim - see [[luxury-signal-gated-live-simulation]]'s
  own note on why the real one isn't reconstructable after the fact.
- This is a HYPOTHETICAL - Futures does not have the breakout-signal
  scanner deployed today (only Luxury does). Nothing here changes any
  live config; this is evaluation only, same as every sweep/hypothesis
  backtest in this series until the user explicitly asks to deploy it.
