# Config-tuning history

Measured effects of specific Swing/Options config changes, backed by
backtests run against the actual live entry/exit logic (not a generic
strategy simulation) - see `backtest-methodology.md` for how these are
kept faithful to production.

## Swing Supertrend period/multiplier: 10/3.0 (current) vs 5/5.0 - SONACOMS 30-day

**Date:** 25 Sep 2026
**Symbol:** SONACOMS (NSE equity, OPTIONS basket, not an index symbol - v3
Day Range branch never fires for it either way)
**Question:** does tightening the Supertrend to period=5/multiplier=5.0
(from the live default period=10/multiplier=3.0) improve P&L?

**Method:** `traderBoy/backtest_sonacoms_supertrend_5_5_vs_current_30day.py`
- byte-for-byte port of `Swing/trading_engine.py`'s live v2/v3 entry filter
(5m Supertrend cross + 15m Supertrend/regime/gap-widening filter leg) and
exit ladder (MAX_LOSS_HIT -> TARGET_HIT -> PROFIT_PROTECTION_HIT ->
STOP_LOSS_HIT -> SUPERTREND_REVERSAL), replayed candle-by-candle over the
last 30 trading days (2026-08-14 to 2026-09-25) on continuous multi-day
candles. Both variants share the same regime/EMA/volume-floor computation
and the same option contract/price data - only the Supertrend
period/multiplier (applied to BOTH the 5-min entry/exit signal and the
15-min filter leg, since config.py only has one such value reused across
both timeframes) differs between runs.

**Result:**

| Variant | Trades | Wins | Losses | Win rate | Net P&L |
|---|---|---|---|---|---|
| CURRENT (period=10, mult=3.0) | 10 | 8 | 2 | 80.0% | **+Rs 17,946** |
| NEW (period=5, mult=5.0) | 8 | 3 | 5 | 37.5% | **-Rs 8,146** |

Delta: **-Rs 26,092** for the tighter (5/5.0) setting.

**Why it's worse, mechanically:** a shorter period (5 vs 10) makes the
Supertrend react to fewer bars, so it whipsaws inside the same trend far
more - 5 of the 8 NEW-variant trades exited via SUPERTREND_REVERSAL
(all losses) vs only 3 of 10 for CURRENT, and none of those 3 CURRENT
reversals were as costly. The larger multiplier (5.0 vs 3.0) does widen
the band, which should cut *some* noise, but not enough to offset the
much shorter lookback - net effect across this window was a clearly worse
entry/exit rhythm, not a wash.

**Scope of this finding:** one symbol (SONACOMS), one 30-day window,
OPTIONS basket. Not evidence either way for other watchlist symbols, MCX
symbols (different volume-floor gate), or the v3 Day Range branch (index
symbols only, never exercised here). Treat as a data point against
lowering the Supertrend period this aggressively for equity swing entries,
not a general verdict on all period/multiplier combinations.

**Outcome:** live `Swing/config.py` SUPERTREND_PERIOD/SUPERTREND_MULTIPLIER
left unchanged (10/3.0) - user asked for the comparison, not a deploy, and
the backtest itself argues against the change.

## Swing Supertrend entry-signal timeframe: 5-min (current) vs 1-min, period/mult unchanged - SONACOMS 30-day

**Date:** 25 Sep 2026 (same session, direct follow-up)
**Symbol:** SONACOMS, same window (2026-08-14 to 2026-09-25)
**Question:** instead of tuning period/multiplier, what if the
entry-trigger/exit-reversal Supertrend reads 1-min candles instead of
5-min (period=10, multiplier=3.0 unchanged both ways)?

**Config knob:** `Swing/config.py`'s `SUPERTREND_INTERVAL_MINUTES`
(default 5, `SWING_SUPERTREND_INTERVAL_MINUTES` env override). Confirmed
by reading `Swing/signals.py` (`get_supertrend_state(symbol)` with no
explicit interval reads this default) and `Swing/trading_engine.py`
(`_evaluate_entry_signal`): this knob **only** changes the entry-trigger/
exit-reversal Supertrend (and the Day Range branch's own Supertrend). It
does NOT touch the 15-min filter-leg Supertrend (`get_supertrend_state(
symbol, 15)` passes its interval explicitly) or the regime EMA200 5m/15m
reading - those stay exactly as before.

**IMPORTANT CORRECTION (see the PAYTM/VEDL/ASHOKLEY entry below):** this
SONACOMS run's Day Range branch B was hardcoded OFF, on the (WRONG, as of
this run) assumption that Day Range only fires for `config.INDEX_SYMBOLS`
(NIFTY/BANKNIFTY). `Swing/signals.py`'s own module comment says Day Range
was "promoted to the default for every Swing symbol" on 24 Sep 2026, and
`Swing/trading_engine.py`'s `_evaluate_entry_signal` confirms it: `day_
range = await signals.get_day_range_state(symbol)` runs unconditionally
for every non-COPPER symbol. So this SONACOMS result below is NOT a fully
byte-accurate port of production - branch B was never exercised here.
Given SONACOMS's regime/filter conditions this window, it's plausible but
not confirmed that including it wouldn't have changed the trade list.
Treat the number below as directionally informative, not exact.

**Method:** `traderBoy/backtest_sonacoms_supertrend_1min_vs_5min_30day.py`
- forked from the period/multiplier comparison script, reusing its cached
equity/option data. Same filter/gate/exit-ladder logic; only the
Supertrend series' source candles (5-min vs 1-min) and the event loop's
signal-candle-dedup/entry-candle bookkeeping (now 1-min-resolution for the
NEW variant) changed.

**Result:**

| Variant | Trades | Wins | Losses | Win rate | Net P&L |
|---|---|---|---|---|---|
| CURRENT (5-min ST) | 10 | 8 | 2 | 80.0% | **+Rs 17,946** |
| NEW (1-min ST) | 60 (59 closed, 1 still open) | 26 | 30 | 44.1% | **+Rs 40,670** |

Delta: **+Rs 22,724** for the 1-min entry-signal timeframe, despite a much
lower win rate (44.1% vs 80.0%) - avg win Rs +2,879 vs avg loss only
Rs -1,139, a favorable ~2.5:1 win/loss size ratio that more than offsets
firing 6x more often.

**Real-world caveat (not modeled here, same as every backtest in this
repo per `backtest-methodology.md`):** 60 trades in 30 days means ~2/day -
6x the order flow of the current 5-min setting. This backtest uses candle
close prices with no slippage/brokerage/bid-ask-spread modeling; at that
frequency those costs compound meaningfully and are NOT reflected in the
+Rs 40,670 figure. The 5-min setting's smaller trade count is far more
forgiving of unmodeled execution friction. Also worth flagging: this
volume of same-day re-entries on one symbol would interact with the
dup-order guard and MCX/NSE volume-floor gate far more than the current
setting ever does in practice - worth a live-shadow/paper-trade check
before treating this as deployable, not just a clean backtest number.

**Scope:** same single-symbol, single-window caveats as the entry above,
PLUS the Day Range omission noted above - **superseded in direction by the
PAYTM/VEDL/ASHOKLEY result below, which found v4 net NEGATIVE using the
methodology that correctly includes Day Range.** SONACOMS alone is not
representative - see that entry for the full picture.

**Outcome:** not deployed - user asked for the comparison only.

**Update 25 Sep 2026 (same session) - named "v4" in code, NOT deployed:**
this feature set is now `Swing/config.py`'s `ENTRY_STRATEGY_VERSION="v4"`
(commit pending in `traderBoy`) - same v2/v3 combined entry filter and exit
ladder, only difference is `SUPERTREND_INTERVAL_MINUTES` forced to 1. Live
`.env` is still `v2`; `v4` is valid but unselected.

**Real blocker found while wiring this in, not yet fixed:** `Swing/
candle_feed.py`'s live WebSocket candle feed only ever buckets raw ticks
into `BASE_INTERVAL_MINUTES=5` bars and its `_resample` helper can only
build COARSER multiples of that base (`interval_minutes % 5 != 0` raises
`ValueError`) - it cannot produce 1-min bars at all. `get_supertrend_
state`'s broad `except Exception` swallows that error and fails open to
the last cached/`None` state, so v4 would get **no real live signal**
whenever the WS feed is fresh (i.e. during ordinary market hours) - not a
crash, just silent inaction. The 30-day backtest above is unaffected (it
reads real 1-min candles straight from Dhan's REST history, never through
`candle_feed.py`), but the number it produced is NOT yet achievable live.
Fixing this for real means changing `candle_feed.py`'s base bucket from
5-min to 1-min (`_resample` already generalizes to serve 5/15-min from a
1-min base once ticks are bucketed that finely) - a materially larger
change since EVERY Swing signal (regime EMA, both Supertrends, Day Range)
derives from that same feed, not just v4's. Flagged to the user; not done
without explicit go-ahead given the blast radius.

## v3 vs v4 (1-min Supertrend/Day Range), Day-Range-correct methodology - PAYTM, VEDL, ASHOKLEY, 30-day

**Date:** 25 Sep 2026 (same session, direct follow-up - this is the
methodologically-correct multi-symbol re-run the SONACOMS entry above was
missing)
**Symbols:** PAYTM, VEDL, ASHOKLEY - all plain NSE equity F&O, OPTIONS
basket, not MCX/index
**Window:** last 30 trading days (2026-08-14 to 2026-09-25, per-symbol
exact range varies slightly with each one's own trading calendar)

**Method:** `traderBoy/backtest_v3_vs_v4_paytm_vedl_ashokley_30day.py` -
forked from `backtest_swing_v3_multi_symbol_30day.py` (the SOLARINDS/VEDL
script that already correctly includes Day Range branch B for ordinary
equities), extended to run BOTH v3 (5-min Supertrend + Day Range, i.e.
today's live default) and v4 (1-min Supertrend + Day Range) per symbol.
Confirmed via `Swing/signals.py`'s `_fetch_day_range_state_once` that Day
Range's own Supertrend/RSI/open-close comparison all read `config.
SUPERTREND_INTERVAL_MINUTES` too - so v4 changes BOTH branch A (Supertrend
cross) and branch B (Day Range) simultaneously, not just branch A. The
15-min filter leg and 5m/15m regime EMA are unaffected in both variants
(unchanged from every prior entry in this file).

**Result - day-wise, per symbol:**

PAYTM:
| Date | v3 (trades/W/L/net/running) | v4 (trades/W/L/net/running) |
|---|---|---|
| 08-28 | -- | 2/0/2/-2,900/-2,900 |
| 08-31 | 1/1/0/+2,030/+2,030 | -- |
| 09-02 | -- | 1/0/1/-689/-3,589 |
| 09-03 | 1/1/0/+435/+2,465 | -- |
| 09-04 | -- | 1/0/1/-3,988/-7,576 |
| 09-08 | 1/1/0/+3,299/+5,764 | -- |
| 09-25 | 1/1/0/+5,474/+11,238 | 1/0/1/-2,827/-10,404 |
| **TOTAL** | **4 trades, 4W/0L, +Rs 11,238** | **5 trades, 0W/5L, -Rs 10,404** |

VEDL:
| Date | v3 | v4 |
|---|---|---|
| 08-28 | 1/1/0/+2,760/+2,760 | 1/1/0/+920/+920 |
| 09-02 | 1/1/0/+920/+3,680 | 1/1/0/+230/+1,150 |
| 09-23 | 1/0/1/-978/+2,702 | 1/0/1/-345/+805 |
| **TOTAL** | **3 trades, 2W/1L, +Rs 2,702** | **3 trades, 2W/1L, +Rs 805** |

ASHOKLEY:
| Date | v3 | v4 |
|---|---|---|
| 08-28 | -- | 3/1/2/+250/+250 |
| 08-31 | 1/1/0/+3,100/+3,100 | -- |
| 09-01 | -- | 1/1/0/+2,150/+2,400 |
| 09-02 | 1/1/0/+1,200/+4,300 | -- |
| **TOTAL** | **2 trades, 2W/0L, +Rs 4,300** | **4 trades, 2W/2L, +Rs 2,400** |

**Combined:**

| Variant | Trades | Wins | Losses | Net P&L |
|---|---|---|---|---|
| v3 (5-min ST/Day Range) | 9 | 8 | 1 | **+Rs 18,240** |
| v4 (1-min ST/Day Range) | 12 | 4 | 8 | **-Rs 7,199** |

Delta: **-Rs 25,439** - v4 loses money where v3 profits handsomely, the
OPPOSITE conclusion from the SONACOMS-only result above. PAYTM is the
worst case: v4 went 0-for-5.

**Why this contradicts the SONACOMS result:** two real, compounding
differences from the earlier (incomplete) SONACOMS run: (1) this run
correctly includes Day Range branch B, which SONACOMS's did not, and
Day Range's own signal quality is itself timeframe-sensitive - a 1-min
RSI(14)/Supertrend read on Day Range is a much noisier read than a 5-min
one, plausibly degrading branch B entries specifically; (2) different
symbols behave differently under a tighter Supertrend - SONACOMS happened
to have a favorable win/loss SIZE ratio at 1-min resolution that
outweighed its lower hit rate, but that is not a universal property of
tightening the entry timeframe, as PAYTM/VEDL/ASHOKLEY show plainly here.

**Conclusion so far across all v4 backtests in this file:** v4 (1-min
Supertrend/Day Range) is NOT a broadly better setting. It won on SONACOMS
alone (missing Day Range) and lost clearly across PAYTM/VEDL/ASHOKLEY
(Day Range included, methodologically correct). The SONACOMS result
should NOT be generalized - if anything, this set of runs is evidence
AGAINST switching the live watchlist to v4, not for it.

**Outcome:** not deployed. Live `.env` remains `SWING_ENTRY_STRATEGY_
VERSION=v2`. v4 also still has the live-feed gap noted above (candle_
feed.py can't serve 1-min bars), so it isn't currently runnable live
regardless of these backtest numbers.
