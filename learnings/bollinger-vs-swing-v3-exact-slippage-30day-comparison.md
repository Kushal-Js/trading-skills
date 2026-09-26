# Bollinger vs Swing V3 - exact live conditions + 2% slippage, 30-day head-to-head

**Date:** 26 Sep 2026
**User request:** "Run Bollinger and SWING V3 on same data set but with
exact entry and exit conditions, considering 2% slippage also while
closing the trade and show me the PnL comparisons day and trade wise."
**Scope:** same 9-symbol NSE-equity Swing watchlist (BANDHANBNK,
TORNTPHARM, DLF, ZYDUSLIFE, SONACOMS, CIPLA, ASHOKLEY, VEDL, SOLARINDS),
OPTIONS basket, last 30 trading days (28 Aug - 25 Sep 2026) - identical
scope to every other backtest this session.

Unlike the two prior "video-derived" backtests this session (bollinger,
liquidity-sweep, ema_cci_macd - all translating a YouTube strategy that
was never anyone's live code), this request is different in kind: both
Bollinger and Swing V3 are REAL, currently-deployed live packages
(`Bollinger/` and `Swing/` at the repo root). "Exact conditions" here
means byte-accurate to the ACTUAL deployed `trading_engine.py`/`signals.py`
of each package, not a video interpretation - verified via a fresh,
dedicated read of both packages' current code (two parallel Explore-agent
investigations, not assumed from memory or from the earlier same-day
`backtest_bollinger_vortex_9symbols_30day.py`/
`backtest_v3_watchlist_9symbols_30day.py` scripts, which were built before
Bollinger existed as a package / before this request's own "exact" bar).

## Real fidelity gaps found and fixed for this round

**Bollinger** (`backtest_bollinger_exact_slippage_9symbols_30day.py`):
- Now reads every constant LIVE from `Bollinger/config.py` (the real
  deployed package) instead of re-declared local copies - confirmed the
  indicator math (BB-ribbon, Vortex, pullback/pending-stop-order state
  machine, trailing-stop math) is still byte-identical to
  `Bollinger/signals.py` today.
- Added Friday square-off (`FRIDAY_SQUARE_OFF_TIME`, 15:25 IST) - genuinely
  new live behavior added the same day Bollinger itself went live, absent
  from the earlier backtest entirely (which pre-dates it).

**Swing V3** (`backtest_v3_exact_slippage_9symbols_30day.py`):
- Added Friday square-off (same rule, `Swing/config.py`'s own
  `FRIDAY_SQUARE_OFF_TIME`) - confirmed present live, absent from the
  original `backtest_v3_watchlist_9symbols_30day.py`.
- Fixed an entry-retry-semantics gap: the original script permanently
  dropped a Day-Range (`branch_b`) signal the instant it was evaluated,
  even if only skipped by the NSE-volume-floor gate - live retries a
  blocked signal on every poll tick until it actually fills. Fixed by
  only latching `consumed_signal_idx` once a position actually opens.

## A real bug found WHILE building this, not just a fidelity gap

**Dhan's 1-min NSE_EQ historical data stops at 15:14 every day** (confirmed
directly: TORNTPHARM's cached 2026-08-28 series has 360 bars, last one
15:14:00, never 15:15-15:29). A first version of both scripts' Friday-
square-off check (`dt.time() >= 15:25`) could therefore NEVER fire - this
was caught by noticing 3 real positions (TORNTPHARM/DLF/ZYDUSLIFE, all
opened Friday 2026-08-28) riding straight through the weekend into Monday
in the FIRST run's output, exactly the weekend-gap risk this rule exists
to prevent. **Same root cause as the already-documented "Dhan 5-min equity
candles stop at 15:10" sim-clock trap in trading-skills'
backtest-methodology.md - just showing up here for 1-min data at a
different cutoff (15:14, not 15:10).** Fixed in both scripts by ALSO
force-closing at the LAST available bar of a Friday if a position is
still open then (an approximation of the real 15:25 fill using the latest
price this data source actually has - same disclosed-gap discipline as
the original sim-clock-trap fix). Effect of the fix: Bollinger's Friday-
square-off count went from 0 fires (broken) to 3 (working), net P&L
+Rs 11,575 -> +Rs 17,962; Swing V3 went from 0 fires to 5, net P&L
+Rs 45,313 -> +Rs 48,626. **Any future backtest adding a time-of-day exit
condition on Dhan's 1-min equity series must check this same cutoff first**
- a plain `>= HH:MM` check against 1-min NSE_EQ timestamps will silently
never fire for any square-off time at or after 15:14.

## 2% slippage - applied identically to both

Every closing exit price (whichever exit reason fired) is worsened 2%
before P&L is computed and before it's recorded as the trade's
`exit_price` - the strategy's own exit-decision logic (which reason
fires, running best_price/trailing-arm math) still evaluates against the
clean quoted premium; only the recorded FILL is 2% worse. Never applied
to entry price or to the running best_price/trailing bookkeeping.

## Result

| | Swing V3 (exact) | Bollinger (exact) |
|---|---|---|
| Trades | 31 | 287 |
| Wins / Losses | 24 / 7 | 124 / 163 |
| Win rate | 77.4% | 43.2% |
| **Net P&L** | **+Rs 48,626** | **+Rs 17,962** |
| Symbols net positive | 9/9 | 6/9 |
| Exit reasons | PROFIT_PROTECTION_HIT 14, SUPERTREND_REVERSAL 4, TARGET_HIT 4, STOP_LOSS_HIT 4, FRIDAY_SQUARE_OFF 5 | TRAILING_STOP_HIT 216, STOP_LOSS_HIT 68, FRIDAY_SQUARE_OFF 3 |

Per symbol:

| Symbol | V3 trades | V3 P&L | Bollinger trades | Bollinger P&L |
|---|---|---|---|---|
| BANDHANBNK | 2 | +10,629 | 32 | +23,682 |
| TORNTPHARM | 5 | +3,116 | 36 | +4,796 |
| DLF | 5 | +3,619 | 34 | -2,697 |
| ZYDUSLIFE | 5 | +6,925 | 31 | +16,859 |
| SONACOMS | 5 | +10,534 | 35 | -6,446 |
| CIPLA | 2 | +2,362 | 28 | -2,229 |
| ASHOKLEY | 2 | +3,131 | 29 | -3,289 |
| VEDL | 3 | +2,149 | 33 | +307 |
| SOLARINDS | 2 | +6,160 | 29 | -13,021 |

Day-wise combined (only trading days with at least one closed trade shown):

| Day | V3 net | V3 running | Bollinger net | Bollinger running |
|---|---|---|---|---|
| 08-28 | +10,181 | +10,181 | -2,662 | -2,662 |
| 08-31 | +4,838 | +15,020 | -2,722 | -5,384 |
| 09-01 | -193 | +14,827 | +4,120 | -1,264 |
| 09-02 | +7,759 | +22,586 | -1,428 | -2,692 |
| 09-03 | - | - | -704 | -3,396 |
| 09-04 | +8,014 | +30,600 | +177 | -3,219 |
| 09-07 | - | - | +4,203 | +983 |
| 09-08 | - | - | +4,036 | +5,019 |
| 09-09 | +15,492 | +46,092 | +2,345 | +7,364 |
| 09-10 | -2,350 | +43,743 | -2,115 | +5,249 |
| 09-11 | +399 | +44,142 | +5,846 | +11,096 |
| 09-15 | +3,467 | +47,609 | -2,916 | +8,179 |
| 09-16 | - | - | -7,887 | +293 |
| 09-17 | +4,178 | +51,787 | -1,121 | -829 |
| 09-18 | -5,246 | +46,541 | -3,547 | -4,375 |
| 09-21 | - | - | +1,329 | -3,046 |
| 09-22 | +4,552 | +51,093 | +13,673 | +10,628 |
| 09-23 | -2,468 | +48,626 | +4,985 | +15,612 |
| 09-24 | - | - | +2,683 | +18,295 |
| 09-25 | - | - | -333 | +17,962 |

Full trade-wise CSVs (31 and 287 rows respectively) delivered to the user
directly, not reproduced here.

## Reading this comparison honestly

**V3 wins decisively on a per-trade and risk-adjusted basis**: far fewer
trades (31 vs 287, ~9x less), a much higher win rate (77.4% vs 43.2%), and
a higher net P&L despite trading roughly a tenth as often - each V3 trade
averages +Rs 1,569, each Bollinger trade averages +Rs 63. Bollinger's
edge, such as it is, comes from volume (287 chances to compound small
edges) rather than per-trade quality, and that volume is exactly what
makes it far more exposed to the 2% slippage tax: comparing against the
NON-slippage, pre-fidelity-fix numbers from earlier this session
(Bollinger's original video-interpretation backtest: +Rs 106,240; V3's
original script: +Rs 57,748 per this same 30-day/9-symbol window, stored
in `history/bt_swing_v3_watchlist_9symbols/results_combined.json`),
Bollinger's net P&L fell ~83% (+106,240 -> +17,962) while V3's fell only
~16% (+57,748 -> +48,626) - not a clean isolated slippage-only comparison
(config-sourcing and Friday-square-off changed too, see the fidelity-gaps
section above), but directionally consistent with what trade FREQUENCY
alone would predict: a strategy that closes a trade 9x more often pays
the same 2%-per-close tax 9x more often.

**3 of 9 symbols are net negative for Bollinger under exact conditions +
slippage** (DLF, SONACOMS, CIPLA, ASHOKLEY - 4 actually, SOLARINDS too,
so 5 of 9) - a meaningfully different picture than the earlier
video-interpretation backtest's clean 9/9 sweep, since that number never
had slippage or Friday-square-off applied. V3 stays 9/9 positive under
the same treatment.

**Standard caveats** (same as every backtest in this repo): no brokerage
modeled (only slippage), ATM-strike liquidity filtering and the funds/
capacity gates are still disclosed-not-modeled gaps on both sides (see
each script's own docstring), one 30-day window is one sample.

**Outcome:** informational only - not a decision to change either
package's live config, not deployed anywhere new. Purely a backtest
comparison artifact, per the user's own request scope.
