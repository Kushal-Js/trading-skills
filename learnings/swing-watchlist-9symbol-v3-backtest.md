# Swing v3, 9-symbol NSE-equity watchlist backtest - 30 days

**Date:** 26 Sep 2026
**Symbols:** BANDHANBNK, TORNTPHARM, DLF, ZYDUSLIFE (4 added to the
watchlist this same session), SONACOMS, CIPLA, ASHOKLEY, VEDL, SOLARINDS
(the 5 pre-existing NSE-equity watchlist symbols) - COPPER/NATURALGAS
(MCX) and NIFTY/BANKNIFTY (index) excluded, covered separately elsewhere
in this repo.
**Window:** last 30 trading days, per-symbol (roughly 2026-08-14 to
2026-09-25/26, exact range varies slightly with each symbol's own
calendar)
**Live config at time of run:** `SWING_ENTRY_STRATEGY_VERSION=v3` (the
droplet's actual live setting as of this session - see `TRADING_JOURNAL.
md`'s 26 Sep entry), `TARGET_PCT_NON_COPPER=0.35`, `MAX_LOSS_PROTECTION_
RS=4500`, `PROFIT_PROTECTION_RS_OPTIONS=3000`/`GIVEBACK_PCT_OPTIONS=0.02`,
`NSE_VOLUME_FLOOR_RATIO_MIN=0.6`, `HARD_STOP_LOSS_PCT=0.20`.

**Method:** `traderBoy/backtest_v3_watchlist_9symbols_30day.py` - single-
variant fork of `backtest_v3_vs_v4_paytm_vedl_ashokley_30day.py` (v4
dropped, only v3's 5-min Supertrend + 15-min filter leg + Day Range
Bull/Bear, byte-accurate to `Swing/trading_engine.py`'s live branch).

**Result - day-wise, combined:**

| Date | Trades | Wins | Losses | Net P&L | Running |
|---|---|---|---|---|---|
| 08-28 | 7 | 5 | 2 | +2,777 to +2,850 range, net +5,736 | +5,736 |
| 08-31 | 2 | 2 | 0 | +6,162 | +11,899 |
| 09-01 | 2 | 1 | 1 | +844 | +12,742 |
| 09-02 | 3 | 3 | 0 | +9,102 | +21,845 |
| 09-04 | 2 | 2 | 0 | +9,035 | +30,880 |
| 09-09 | 4 | 4 | 0 | +17,217 | +48,097 |
| 09-10 | 2 | 1 | 1 | -1,806 | +46,290 |
| 09-11 | 1 | 1 | 0 | +5,451 | +51,742 |
| 09-15 | 1 | 1 | 0 | +3,750 | +55,492 |
| 09-17 | 1 | 1 | 0 | +4,732 | +60,224 |
| 09-18 | 2 | 0 | 2 | -5,062 | +55,162 |
| 09-22 | 2 | 2 | 0 | +4,907 | +60,069 |
| 09-23 | 2 | 0 | 2 | -2,321 | +57,748 |

**Totals: 31 trades, 23 wins/8 losses (74.2% win rate), net +Rs 57,748.**

**Per-symbol:**

| Symbol | Trades | Wins | Losses | Win rate | Net P&L |
|---|---|---|---|---|---|
| BANDHANBNK (new) | 2 | 2 | 0 | 100.0% | **+Rs 11,628** |
| TORNTPHARM (new) | 5 | 2 | 3 | 40.0% | +Rs 1,912 |
| DLF (new) | 5 | 3 | 2 | 60.0% | +Rs 2,707 |
| ZYDUSLIFE (new) | 5 | 4 | 1 | 80.0% | +Rs 6,750 |
| SONACOMS | 5 | 4 | 1 | 80.0% | +Rs 17,518 (best performer) |
| CIPLA | 2 | 2 | 0 | 100.0% | +Rs 2,720 |
| ASHOKLEY | 2 | 2 | 0 | 100.0% | +Rs 4,300 |
| VEDL | 3 | 2 | 1 | 66.7% | +Rs 2,702 |
| SOLARINDS | 2 | 2 | 0 | 100.0% | +Rs 7,510 |

**Reading this:** all 4 newly-added symbols would have been net positive
over this window under live v3 logic, though TORNTPHARM's 40% win rate
(2W/3L) is the weakest of the 9 and worth watching once real trades
accumulate - its wins were larger than its losses (+2,694/+3,750 vs
-256/-2,931/-1,344) so it stayed net positive, but with a much thinner
margin than BANDHANBNK/ZYDUSLIFE. SONACOMS remains the standout performer
of the whole watchlist. 08-28 and 09-09 were the two best combined days
(multiple symbols firing together); 09-18 and 09-23 were the only two
net-negative days, both driven by 2 simultaneous losses rather than one
outlier.

**Caveats, same as every backtest in this file's methodology:** no
slippage/brokerage modeled; each symbol's option-contract availability is
capped by Dhan's currently-listed-only instrument master (a handful of
entries near the start of the window were skipped as
`expired_contract_unavailable`, logged per-symbol in the run's own
output, not counted here); one 30-day window is one sample, not a
robustness guarantee.

**Outcome:** informational - no config change made. Watchlist stays as
added (13 symbols total, see `TRADING_JOURNAL.md`'s 26 Sep entry).
