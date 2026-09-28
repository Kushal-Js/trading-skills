# 2026-09-28: Swing entered NATURALGAS on a false BULLISH regime (WS-only EMA200)

**Trade.** Real Swing NATURALGAS 23 OCT 300 CALL, 16:55 → 17:51 IST, 18.35 → 18.20, −Rs 187.50. Exit was `SUPERTREND_REVERSAL_TICK`.

**How it got in.** v3 needs a 5-min Supertrend cross-up (real: the 16:50 candle closed at 301.2 over a line at 299.42) **and** `filter_bullish`: 15m Supertrend above OR regime bullish. The 15m Supertrend was below (300.5 vs 302.32), so the regime had to be bullish.

| Regime input | Value | Reading |
|---|---|---|
| 5m EMA200, full 45-day REST (correct) | 304.68 | vs 305.12 → **BEARISH** |
| 5m EMA200, what live used: 487 WS bars only | 305.41 | vs 305.12 → **BULLISH** |

**Root cause.** `Swing/signals._get_intraday_series` returned the WS-reconstructed series **on its own** once it had `min_bars` (200 for regime). The WS feed only holds bars since it was first subscribed, with gaps at every restart or feed drop: 23 Sep 142 bars, 24 Sep 170, 25 Sep 80, 28 Sep 95. The 15m side still came from REST, because the WS feed didn't have 200 fifteen-minute bars. The two EMAs were built on different histories, which broke the continuous-candles rule. Bollinger had the identical bug, fixed the same morning (`e32c91d`); Swing's copy was missed.

**Fix.** traderBoy `eab5ff0`, deployed 19:17 IST. The REST series is always the base:

- the forming trailing bar is trimmed;
- it is cached until a newer closed bar should exist, and refetched at most every 60s;
- the last good base is kept if a refetch fails.

WS bars only append newer bars. A replay of the 16:55 moment through the live `_fetch_regime_state_once` with the fix reads BEARISH (fast EMA 304.68), so the trade would have been blocked.

**Lesson.** Any "WS if it has N bars, else REST" shortcut is a hidden data-window change. Audit every such call site whenever one is found. Swing and Bollinger both had it.
