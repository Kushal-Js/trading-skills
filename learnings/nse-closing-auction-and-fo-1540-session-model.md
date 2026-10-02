# NSE closing auction (cash) and 15:40 F&O close - the bot's session model is out of date

Found 2 Oct 2026 (load audit before Mon 5 Oct). Effective **Mon 3 Aug 2026** on NSE:

- **F&O-eligible stocks, cash market:** continuous trading ends **15:15**. A Closing Auction Session (CAS) runs
  after that and prints one closing trade around 15:28-15:30.
- **Equity derivatives (stock + index futures/options):** trade until **15:40** (was 15:30).
- **Indices (NIFTY/BANKNIFTY spot):** values freeze from 15:15 to ~15:28, then jump at the auction print.
  Option premiums keep moving with the futures until 15:40.

## Evidence (our own data, not only the news)

- Dhan REST 1-min history, cached (`history/bt_walkforward_long/underlying_1m`): every stock's last bar is
  **15:14** on every day from 3 Aug to 30 Sep. 31 Jul still ran to 15:29. NIFTY/BANKNIFTY keep bars to 15:29,
  but their 1-min ranges are **0.0 from 15:15 to 15:27**.
- Cached option 1-min candles (BANKNIFTY, APLAPOLLO): bars to **15:39** on 29 Sep, 30 Sep and 1 Oct.
- Live WS candles (`history/*_swing_candles_*`, `*_underlying_candles_*`), 23 Sep-1 Oct: the 15:15 and 15:20
  bars exist for only 4 of 30 (or 172) symbols (the 2 indices, flat, plus 2 MCX). The "15:25" bar is a single
  print with the whole auction volume (RELIANCE 1 Oct: 15,566,134).
- The parity report REST bars for 1 Oct end at 15:10 (72 bars, 09:15-15:10).

## What it breaks in the bot (traderBoy at 2232b57)

- `official_candles.py` / `Swing/candle_feed.py` assume stock sessions end at 15:30 (last 5-min bar 15:25,
  last 15-min bar 15:15). Dhan never publishes a stock bar after 15:10 (5-min) / 15:00 (15-min). Effects:
  - `ws_cover` says "download" for the whole first bar of every day: `_follows(prev-day 15:10 -> 09:15)` is
    false. The REST bases are refetched every 60 s from 09:15 to 09:20, plus one more round at 09:20.
  - From 15:15 to 15:30 every stock base waits for 15:15/15:20/15:25 bars that never come. That means one
    refetch per base per 60 s, which saturates the 2 calls/s budget. The executor also lags right after
    Unified Momentum's 15:15 square-off.
  - The auction print becomes a fake flat "15:25" WS bar. It is persisted, restored after a restart, and
    appended to the next morning's series until the base advances.
- `Options/config.MARKET_CLOSE_TIME=15:30` is used for NSE F&O. After 15:30, `place_market_order` tags a SELL
  as an **AMO** (it fills at the next open), and the orphan-stop sweeps stop. F&O is still open until 15:40.
  This only matters if a real position is still open after 15:30 (a square-off failure path).
- Backtests use the same REST history, so they already have the 15:15 cash close and the frozen index. This is
  a live-plumbing problem, not a strategy-parity one. One case: Scalper (BANKNIFTY, square-off 15:25) reads a
  frozen spot from 15:15 to 15:25, and its backtest had the same frozen bars.

## Rule

Bar math has to be session-aware per segment:
- NSE_EQ (F&O stocks): 09:15-15:15.
- IDX_I: 09:15-15:30 (frozen from 15:15).
- NSE_FNO orders and monitoring: until 15:40.
- MCX: unchanged.

Never hard-code 15:30 for "the NSE close".
