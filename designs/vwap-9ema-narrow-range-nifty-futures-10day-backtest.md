# Backtest: VWAP + 9 EMA "narrow range breakout" (ChartEase video) on NIFTY futures, 10 days

**Status: Backtested 2 Oct 2026. Lost money after costs in every variant tested. Not built, not wired into the bot.**

User request (2 Oct 2026): turn the YouTube video "Most Intraday Traders Enter at the Worst Time" (ChartEase, `youtube.com/watch?v=0T729ElXZ78`, Hindi) into a strategy and backtest it on NIFTY for the last 10 trading days with trade-wise P&L.

## Rules as stated in the video (from the Hindi transcript)

- 5-min chart only. Indicators: session VWAP (default settings) + EMA(9) on close.
- Bias: price above VWAP = buys only, below = sells only.
- Step 1, narrow range: VWAP and 9 EMA come very close together (the "key zone").
- Step 2, breakout: ONE strong candle (big body, small wick, his "hidden candle" idea) breaks both lines.
- Step 3, entry: buy above that candle's high (sell below its low), stop-loss at the other end of the candle, target 1:2, "then trail" (trail method not specified).
- "Expert hack" direction filter: the previous day's closing VWAP (15:25 5-min candle). Price, VWAP and EMA all above it = buys only; all below = sells only.

## How it was implemented (choices the video leaves open are marked *)

- **Instrument:** NIFTY Oct-2026 future (security 48704, lot 65). The NIFTY index has no volume, so VWAP needs futures volume. The Sep future (68407) expired 29 Sep and `/charts/intraday` returns nothing for it, so the Oct contract was used for all 10 days. Its volume was thin until ~24 Sep (0.6-1.2M/day vs 3-7M after the roll), but its VWAP is still a true VWAP of that contract.
- **Data:** 5-min + 1-min candles from Dhan `/v2/charts/intraday`, fetched read-only on the droplet with the bot's cached token (no login). Regular session 09:15-15:29 only; the 15:30-15:40 closing session is excluded. EMA is continuous across days; VWAP resets daily.
- **Window:** 18, 21, 22, 23, 24, 25, 28, 29, 30 Sep and 1 Oct 2026 (14 Sep was a holiday). Indicators warmed up from 8 Sep.
- \*Narrow range: |VWAP - EMA9| on the candle BEFORE the breakout <= 0.04% of price (~9 points, the bottom quarter of the window's gaps; the median gap was 21 points).
- \*Strong candle: body >= 60% of the candle's range; the body must straddle both lines (open beyond one side, close beyond the other) and the close must be on the right side of VWAP.
- \*Entry: stop order at the signal candle's high/low, valid for the next 3 five-minute candles, cancelled if the stop-loss side trades first. Filled from 1-min candles; a gap through the trigger fills at the 1-min open. Signals are only read after the 5-min candle closes (no forming-candle look-ahead).
- Exits on 1-min candles: stop-loss, 1:2 target (a limit order, no slippage), square-off at 15:20. If one 1-min candle touches both stop and target, the stop counts. On the entry minute no target is credited.
- \*Trail variant: when 1:2 is reached, the stop moves to +1R, then the trade exits on the first 5-min close beyond the 9 EMA.
- One position at a time; no new entries after 15:00.
- Costs per round trip, 1 lot: brokerage Rs 40, STT 0.05% on the sell side (raised from 0.02% on 1 Apr 2026), NSE transaction charge 0.00173%, SEBI fee, 0.002% stamp duty on the buy side, 18% GST on brokerage + exchange + SEBI. That is about Rs 880-910 per trade (~13.7 points). Plus 1 point of slippage per stop-order side.

## Results (1 lot, base settings)

11 trades, 3 winners (27%). Gross -109.5 points / -Rs 7,118. Costs Rs 9,798. **Net -Rs 16,916.** Three of the 10 days (22, 23, 28 Sep) had no signal. 30 Sep alone: three long stop-outs, -Rs 10,955.

| Variant | Trades | Wins | Gross Rs | Net Rs |
|---|---:|---:|---:|---:|
| Base (narrow <= 0.04%, body >= 60%, 3-candle entry window, fixed 1:2, filter on) | 11 | 3 | -7,118 | **-16,916** |
| 1:2 then trail (lock 1R, exit on 5-min close beyond 9 EMA) | 11 | 3 | +650 | -9,147 |
| Previous-day VWAP filter OFF | 14 | 2 | -14,300 | -26,816 |
| narrow <= 0.02% | 8 | 3 | -1,937 | -9,066 |
| narrow <= 0.06% / 0.08% / 0.10% (identical) | 12 | 3 | -8,593 | -19,296 |
| body >= 70% | 7 | 2 | -5,044 | -11,321 |
| entry window 1 candle | 10 | 3 | -3,750 | -12,666 |
| strong candle also > 20-candle average range | 9 | 2 | -7,826 | -15,847 |
| narrow <= 0.08% + trail | 12 | 3 | -826 | -11,528 |

## Takeaways

- **No variant made money after costs.** The best gross result (trail exit, +Rs 650) became -Rs 9,147 net.
- **Costs are the structural problem for this setup on 1 NIFTY futures lot.** Typical risk was 17-50 points, while the round trip costs ~14 points plus slippage. A 1:2 winner on a 17-point risk nets only about Rs 1,300.
- **The "expert hack" filter did help:** it cut net losses from -26.8k to -16.9k by removing counter-trend trades. The base entry, though, still stops out about 3 times in 4.
- **10 days and 11 trades is a tiny sample,** and late Sep 2026 was a falling, choppy market. Same caveat as every short backtest here. It is not proof the idea can never work, but nothing here supports deploying it.
- Data/tooling notes: an expired futures contract has no intraday data (same trap as `learnings/backtest-methodology.md`). The droplet token file is `date|token|expiry`, so take the middle field.

Code: scratch scripts `bt_vwap.py` + `vwap_common.py` (not committed to traderBoy; the rules above are complete enough to rebuild them).
