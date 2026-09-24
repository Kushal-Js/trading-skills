# v1 (ORB) and v2 (1h RSI/5min timing) adapted to NIFTY-50 INDEX OPTIONS (24 Sep 2026, user request)

**Status: exploratory backtesting only - nothing here is deployed live.**
Neither script touches any live package; both are standalone backtests in
`traderBoy` (repo root), paper/simulation only, no real orders placed.

See [[ashokley-futures-strategy-exploration-orb-v1]] (v1's original
FUTSTK design) and [[ashokley-futures-1hour-rsi-5min-timing-v2]] (v2's
original FUTSTK design + multi-symbol sweep) for the strategies this
adapts. This file covers ONLY what changed to make them trade NIFTY's
option chain instead of a stock's futures contract, and the new
constraint that comes with options specifically.

## What changed, both strategies

- **Signal source**: NIFTY's own SPOT INDEX candles (`security_id="13",
  exchange_segment="IDX_I", instrument_type="INDEX"` - same fetch this
  repo already uses for `fetch_nifty_1m_continuous` elsewhere, e.g.
  `backtest_all_fno_breakout_signal_15day.py:436-454`), not a futures
  contract's own candles. Opening range / RSI(14) computed off spot, no
  expiry constraint on this leg - it's not a derivative contract.
- **Instrument traded**: on a bullish signal, BUY the nearest-expiry ATM
  CE; on a bearish signal, BUY the nearest-expiry ATM PE. Never short
  options - same "always LONG for P&L purposes" convention this repo
  already uses for every stock-OPTIONS backtest path (e.g.
  `backtest_swing_v2_combined_generic.py`'s `run_options()`).
- **New shared module**: `nifty_options_bt_common.py` - first OPTIDX
  (index option) resolver in this repo; every prior
  `nearest_ce_for_strike_ref`/`nearest_pe_for_strike_ref` helper filtered
  `SEM_INSTRUMENT_NAME=="OPTSTK"` (stock options only). Filters the
  trading symbol on the exact `"NIFTY-"` prefix (a loose
  `startswith("NIFTY")` would also match unrelated instrument-master rows
  like `NIFTYFPI`).
- **Exit levels stay in SPOT terms** (opening-range stop/target for v1,
  RSI 70/30/time-cap for v2) - only the realized PnL is the option
  premium's own entry-vs-exit move, same convention as the existing
  stock-OPTIONS backtest path.

## NEW constraint specific to options (didn't apply to the FUTSTK versions)

Dhan's instrument master only lists CURRENTLY-LISTED contracts - an
expired weekly NIFTY option is delisted and its security_id/candles
become permanently unfetchable. Confirmed empirically this session: a
10-day v1 backtest run resolved cleanly for 8 of 10 days, but the two
OLDEST days (09/09, 09/10) came back with **no option price data at
entry time** and were skipped/logged rather than silently dropped or
treated as a bug:
```
2026-09-09 10:00:00+05:30: SKIPPED entry signal=SHORT - no option price data at entry time (likely expiry-ceiling gap)
2026-09-10 10:02:00+05:30: SKIPPED entry signal=SHORT - no option price data at entry time (likely expiry-ceiling gap)
```
**Practical ceiling found: the currently-listed near-week NIFTY contract
(2026-09-29 expiry) has usable historical data back through at least
2026-09-11** - a better window than this repo's own prior finding
("3 trading days", NOTES.md) suggested, but still bounded, and it WILL
shrink further back in time than that. Any future NIFTY-options backtest
should expect entries older than ~1.5-2 weeks to start failing, and
should treat SKIPPED entries as expected/normal, not a bug to chase.

## v1 (ORB) result - NIFTY options, 10-day window (2026-09-09 to 09-23)

Script: `backtest_nifty_options_orb_v1.py`, CLI args `test_days_back
opening_range_minutes rr_multiple` (default 5/15/2.0).

**8 trades (2 skipped at the expiry ceiling), 4 wins/4 losses (50.0%),
net +Rs11,713, avg +Rs1,464/trade.**

| Date | Side | Strike | Entry | Exit | Reason | PnL |
|---|---|---:|---|---|---|---:|
| 09/11 | LONG CE | 23300 | 10:08 @ 254.95 | 13:57 @ 322.00 | TARGET_HIT | +4,358 |
| 09/15 | SHORT PE | 23450 | 09:36 @ 196.00 | 15:29 @ 356.95 | TARGET_HIT | **+10,462** |
| 09/16 | SHORT PE | 23200 | 09:30 @ 207.00 | 15:29 @ 189.40 | EOD_SQUARE_OFF | -1,144 |
| 09/17 | LONG CE | 23250 | 09:31 @ 237.55 | 15:29 @ 235.80 | EOD_SQUARE_OFF | -114 |
| 09/18 | SHORT PE | 23300 | 09:34 @ 176.90 | 14:28 @ 128.30 | STOP_HIT | -3,159 |
| 09/21 | LONG CE | 23400 | 09:35 @ 156.95 | 15:29 @ 162.15 | EOD_SQUARE_OFF | +338 |
| 09/22 | SHORT PE | 23400 | 10:32 @ 111.95 | 15:29 @ 129.00 | EOD_SQUARE_OFF | +1,108 |
| 09/23 | LONG CE | 23400 | 10:45 @ 137.90 | 15:29 @ 135.80 | EOD_SQUARE_OFF | -136 |

Two trades (09/11, 09/15) are 127% of total profit combined - the rest
of the window is close to flat. Same single-day-dominance pattern as the
FUTSTK version of v1 (see [[ashokley-futures-strategy-exploration-orb-v1]]'s
own +26,250 outlier). This time TARGET_HIT actually fired twice (unlike
the FUTSTK version's 10-day sample where every exit was EOD) - options'
own leveraged premium moves reach the spot-implied target faster than the
underlying itself typically does, which makes intuitive sense (a CE/PE
premium moves several multiples of the spot % move near the money).

## v2 (1h RSI/5min timing) result - NIFTY options

Script: `backtest_nifty_options_1hour_rsi_v2.py`, CLI args
`test_days_back max_hold_minutes` (default 5/30).

**5-day window (2026-09-17 to 09-23): 9 trades, 6 wins/3 losses (66.7%),
net +Rs2,707, avg +Rs301/trade.**

**10-day window (2026-09-09 to 09-23): also 9 trades (same dates - no
signals existed before 09/17, none skipped), but net +Rs416, avg
+Rs46/trade - a MATERIALLY different number from the 5-day run despite
the "same" 9 trades.**

**Caveat, important and NOT specific to NIFTY:** the discrepancy is
because RSI(14) is Wilder-smoothed - its value at any given bar depends
on the ENTIRE path back to wherever the seed window started, not just
recent bars. Fetching a longer lookback (25d vs 20d, per this script's
`TEST_DAYS_BACK + 15` formula) shifts the RSI seed's starting point,
which shifts every downstream RSI value slightly, which shifts EXACTLY
which 5-min bar a crossover fires on near ambiguous levels - trade #2 in
the two runs is a real example: same day, but the 5-day run's version
entered at 13:15 (premium 162.0, exit +1,820) while the 10-day run's
version entered at 14:15 (premium 187.5, exit -471). This is an inherent
property of Wilder smoothing, not a bug in this script - but it DOES mean
this backtest's results are somewhat sensitive to the exact
`SIGNAL_LOOKBACK_DAYS` chosen, a caveat that likely also affects the
FUTSTK v2 results to some (smaller, since those used a consistent
lookback formula throughout) degree and hasn't been explicitly
quantified there.

## Open questions for the next session

- How far back can a NIFTY options backtest realistically go before the
  expiry ceiling bites in earnest - only tested up to 10 days so far.
- Quantify the RSI-lookback-sensitivity caveat above more rigorously
  (e.g. does it change WHICH bars cross, or just the RSI value's
  precision) - currently just observed empirically, not characterized.
- Both scripts default to buying options at whatever premium is quoted -
  no liquidity/spread guard, unlike the live Options package's real entry
  gates (LIQUID_CONTRACT_MIN_PRIOR_SESSION_VOLUME etc.) - a real backtest
  meant to inform a live decision would need those.
- Untested: BANKNIFTY or other index options, and whether v1/v2's
  FUTSTK-symbol-specific hold-time tuning (see v2's multi-symbol sweep)
  has an index-options analogue.
