# 52-week-high F&O screen + Swing vs Bollinger 30-day backtest (26 Sep 2026)

User request: find every NSE F&O stock at/near its 52-week high, backtest
the candidates against both Swing and Bollinger over the last 30 days,
then rebalance both live watchlists to the 15 best performers and drop
the 6 worst, out of the combined set of candidates + Bollinger's existing
9-equity watchlist.

## Screen methodology

`traderBoy/screen_fno_52week_high.py` (new, this session) - every NSE
OPTSTK underlying (210 stocks), ~370 calendar days of real daily OHLC via
Dhan's `historical_daily_data`, flagged if the latest close is within 1%
of the trailing-year high. Only **ZYDUSLIFE** (0.39% off, fresh 52w high
that day) cleared the 1% bar; the next 14 closest (up to ~5.5% off) were
carried forward anyway as the candidate pool, since a hard 1% cutoff left
too few names to backtest meaningfully.

**Real bug found and fixed mid-screen** (see `traderBoy` commit
`d24a823`): the screener's equity-instrument lookup (same pattern as
`Options/dhan_client.py`'s `_equity_security_id`) picked a colliding
NSE-listed debenture row over MOTHERSON's real equity share (see
`Options/dhan_client.py`'s own updated docstring for the full collision -
SEM_TRADING_SYMBOL="MOTHERSON" matches BOTH the real share, SEM_SERIES=
"EQ", and an unrelated bond, SEM_SERIES="D1"). Confirmed the DAILY-data
52-week-high numbers were unaffected (Dhan's `historical_daily_data`
happened to return identical series for both security_ids, verified
directly) - only the INTRADAY path used for live Supertrend/regime
signals was actually broken. Fixed by filtering `SEM_SERIES=="EQ"` in
both the screener and the shared `_equity_security_id`/
`_equity_instrument_meta` resolvers.

## 15-symbol candidate pool

ZYDUSLIFE, DIVISLAB, AUROPHARMA, APOLLOHOSP, SONACOMS, RBLBANK, BOSCHLTD,
APLAPOLLO, RADICO, MOTHERSON, MCX, NYKAA, TORNTPHARM, LICHSGFIN,
ADANIPORTS. (ZYDUSLIFE/SONACOMS/TORNTPHARM already overlapped Bollinger's
existing 9-equity watchlist.)

## Backtests run (30 days, options basket, real Dhan candle data)

- **Swing v2/"v3"** (`backtest_swing_v3_multi_symbol_30day.py`, fixed this
  session - see below): 61 trades, 50.8% win rate, **+Rs 55,158** net
  across all 15 symbols.
- **Bollinger** (`backtest_bollinger_vortex_9symbols_30day.py`, run twice:
  once against the 15 new candidates, once against the current 9-equity
  Bollinger watchlist for a like-for-like baseline): 440 trades, 54.4%
  win rate, **+Rs 175,126** across the 15 new candidates; 287 trades,
  57.5% win rate, **+Rs 106,240** across the existing 9-equity watchlist.

**Two stale checks fixed in the Swing backtest script** (`traderBoy`
commit `bc77c43`, previously uncommitted/untracked): it asserted against
`swing_config.MCX_SYMBOLS`, removed in the 25 Sep MCX live-reload
refactor (replaced with the live `dhan_wrapper.is_mcx_commodity()`
check, moved to run after `authenticate()` since it needs the instrument
master) - and against `ENTRY_STRATEGY_VERSION == "v2"` exactly, which
broke once the deployed `.env` value became the literal alias `"v3"`
(see `config-tuning-history.md`/commit `34c0103`). Neither reflects a
methodology gap - both are now fixed to match current live code.

**Caveat on comparing Swing vs Bollinger directly**: not apples-to-apples
on trade frequency. Bollinger is a pure trailing-stop system (no profit
target - see `bollinger-vortex-strategy-30day-backtest.md`) so it stays
in trend moves and fires ~7x more often than Swing's target+giveback-
protection exits (per-trade average Rs 904 for Swing vs Rs 398 for
Bollinger across the 15-symbol run). Bollinger's higher total P&L
partly just reflects more market exposure, not necessarily a cleaner
per-trade edge. Bollinger also has **zero live/paper track record** -
every number here is backtest-only.

## Combined 21-symbol ranking (Bollinger strategy, by net P&L)

| Rank | Symbol | Trades | Win Rate | Net P&L |
|---|---|---:|---:|---:|
| 1 | ZYDUSLIFE | 31 | 58.1% | +28,935 |
| 2 | BANDHANBNK | 32 | 65.6% | +27,648 |
| 3 | MCX | 33 | 69.7% | +21,904 |
| 4 | BOSCHLTD | 27 | 63.0% | +16,000 |
| 5 | APLAPOLLO | 36 | 47.2% | +15,767 |
| 6 | LICHSGFIN | 34 | 45.5% | +12,600 |
| 7 | RBLBANK | 25 | 36.0% | +12,541 |
| 8 | TORNTPHARM | 36 | 66.7% | +12,269 |
| 9 | MOTHERSON | 29 | 62.1% | +10,763 |
| 10 | SONACOMS | 35 | 48.6% | +10,474 |
| 11 | DIVISLAB | 25 | 60.0% | +9,350 |
| 12 | ASHOKLEY | 29 | 72.4% | +8,000 |
| 13 | APOLLOHOSP | 30 | 53.3% | +7,712 |
| 14 | DLF | 34 | 52.9% | +7,220 |
| 15 | ADANIPORTS | 33 | 54.5% | +6,056 |
| 16 | NYKAA | 18 | 38.9% | +5,937 |
| 17 | AUROPHARMA | 27 | 48.1% | +5,747 |
| 18 | VEDL | 33 | 63.6% | +5,635 |
| 19 | SOLARINDS | 29 | 41.4% | +3,467 |
| 20 | CIPLA | 28 | 46.4% | +2,593 |
| 21 | RADICO | 21 | 57.1% | -930 |

Top 15 (ranks 1-15) kept/added; bottom 6 (ranks 16-21) dropped where
present.

## Live watchlist rebalance (26 Sep 2026)

Checked the ACTUAL live droplet files via SSH first, not local copies -
local `data/watchlist` was completely stale (unrelated old symbol list)
and local `data/bollinger_watchlist` was missing CANBK/VBL (added
earlier the same day per the journal's own "CANBK and VBL added" row -
neither symbol was part of this screen/backtest and both were left
untouched here).

- **Removed from both** (bottom-6, present on live lists): VEDL,
  SOLARINDS, CIPLA.
- **Added to both** (top-15, missing from live lists): MCX, BOSCHLTD,
  APLAPOLLO, LICHSGFIN, RBLBANK, MOTHERSON, DIVISLAB, APOLLOHOSP.
- **Left untouched** (not part of this backtest): COPPER, NATURALGAS,
  NIFTY, BANKNIFTY, CANBK, VBL.
- NYKAA, AUROPHARMA, RADICO (also bottom-6) were never on either live
  list, so nothing to remove there.

Resulting Swing watchlist (20 symbols): COPPER, NATURALGAS, NIFTY,
BANKNIFTY, CANBK, VBL, SONACOMS, ASHOKLEY, BANDHANBNK, TORNTPHARM, DLF,
ZYDUSLIFE, MCX, BOSCHLTD, APLAPOLLO, LICHSGFIN, RBLBANK, MOTHERSON,
DIVISLAB, APOLLOHOSP.

Resulting Bollinger watchlist (18 symbols): same minus COPPER/NATURALGAS.

See `TRADING_JOURNAL.md`'s 26 Sep entries for the deploy/restart
timeline and post-restart validation.
