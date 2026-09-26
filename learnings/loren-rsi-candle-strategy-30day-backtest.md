# "loren_rsi_candle" strategy - RSI-bullish/bearish state + candle-close confirmation, 30-day backtest

**Date:** 26 Sep 2026
**Source:** YouTube video "Copy My Insane Loren Setup Strategy (& Finally
Becoming Profitable)" (Upsurge Club, featuring Pankaj Sahu),
https://www.youtube.com/watch?v=PSNGdXCWmgg - mostly Hindi/Hinglish,
25:54 long. YouTube's own transcript panel got stuck loading indefinitely
every time it was opened this session, and the auto-translated Hindi
captions are a poor phonetic transliteration of the English trading
terms - not usable for exact numeric parameters. Rules were pulled
instead from the video's own on-screen ENGLISH whiteboard (captured via
seeking the player directly to the "Understanding the Setup with
Examples" chapter, ~9:10) plus the video's own description/chapter list.
**Scope:** backtest script only
(`traderBoy/backtest_loren_rsi_candle_swing_watchlist.py`), NOT wired
into Swing/trading_engine.py - same 9-symbol NSE-equity Swing watchlist
(BANDHANBNK, TORNTPHARM, DLF, ZYDUSLIFE, SONACOMS, CIPLA, ASHOKLEY, VEDL,
SOLARINDS), OPTIONS basket, last 30 trading days - identical scope to
this session's other three video-derived backtests ([[bollinger-vortex-strategy-30day-backtest]],
[[liquidity-sweep-prev-hour-strategy-30day-backtest]],
[[ema-cci-macd-strategy-30day-backtest]]).

## The rules, as pulled from the whiteboard + description

1. **Core signal, quoted directly off the whiteboard** (demonstrated on a
   BANKNIFTY chart with hand-drawn zigzag structure): "① RSI - Bullish ->
   plan CE/Long" and "② RSI - Bearish -> plan PE/Short" - RSI establishes
   directional BIAS first; a CE is only "planned" while RSI reads
   bullish, a PE only while RSI reads bearish.
2. **Confirmation before entry** (video description, quoted): "waiting
   for confirmation is one of the most important parts of the strategy
   instead of taking random trades," plus a dedicated chapter "The
   Problem with Trading Without Candlesticks" (19:45) - RSI direction
   alone is explicitly NOT the trigger; price/candlestick confirmation is
   required on top, specifically to "avoid getting trapped during
   sideways market conditions."
3. **Systematic stop-loss** (video description, quoted): "helps traders
   place stop losses in a systematic way" - a rule-based, not arbitrary,
   stop.
4. Presented as cross-asset (Bank Nifty, stocks, crypto, commodities all
   get dedicated chapters) - a generic direction+confirmation framework.

## Interpretation calls made (full detail in the script's own docstring)

- RSI period/levels never stated numerically (whiteboard says "Bullish"/
  "Bearish", not "RSI>70"/"<30" - overbought/oversold language is
  notably absent; a faint "30" annotation near the whiteboard sketch
  wasn't legible/confirmable at available resolution). Used RSI(14) with
  **the exact RSI_BULL_LEVEL=60/RSI_BEAR_LEVEL=40 pair this codebase's
  own `Swing/config.py` already defines for its live "Day Range
  Bull/Bear" v3 branch** (`DAY_RANGE_RSI_BULL_LEVEL`/`_BEAR_LEVEL`) -
  reusing a real, already-deployed "RSI enters a bullish/bearish
  directional STATE" convention already in this exact repo, rather than
  guessing at the illegible whiteboard number or defaulting to a generic
  70/30.
- "Confirmation"/candlestick dependency, made concrete: once RSI crosses
  into a bullish/bearish state (bar i), a "pending breakout" arms at that
  bar's own high (bullish) or low (bearish) - price must CLOSE beyond it
  on a later bar to confirm. Entry at the OPEN of the next bar. If RSI
  falls back through the neutral 50 midline before confirming, the
  pending setup is cancelled (the "sideways trap" the video describes).
- Systematic stop-loss: the confirmation candle's own low (long) / high
  (short), converted to a percentage of underlying price and applied to
  the option premium via `hard_stop_for` (floored at 1%), same convention
  as every other backtest here.
- No exit rule beyond the stop is ever described - ported this session's
  own "exit on the opposite signal" convention (a fresh RSI-bearish cross
  exits a LONG, a fresh RSI-bullish cross exits a SHORT), plus
  `MAX_LOSS_PROTECTION_RS` and 15:15 IST EOD square-off.
- One pending setup at a time; a new opposite RSI cross cancels the
  current pending setup and arms the other side instead.

## Result

| Symbol | Trades | Wins | Losses | Win rate | Net P&L |
|---|---|---|---|---|---|
| BANDHANBNK | 58 | 9 | 48 | 15.5% | +Rs 2,988 |
| TORNTPHARM | 51 | 9 | 42 | 17.6% | +Rs 1,306 |
| DLF | 49 | 4 | 45 | 8.2% | -Rs 13,870 |
| ZYDUSLIFE | 58 | 5 | 53 | 8.6% | -Rs 16,920 |
| SONACOMS | 61 | 4 | 57 | 6.6% | -Rs 20,396 |
| CIPLA | 55 | 3 | 52 | 5.5% | -Rs 7,182 |
| ASHOKLEY | 44 | 4 | 40 | 9.1% | +Rs 10,500 |
| VEDL | 48 | 4 | 44 | 8.3% | -Rs 2,530 |
| SOLARINDS | 43 | 4 | 39 | 9.3% | +Rs 55,555 |
| **COMBINED** | **467** | **46** | **420** | **9.9%** | **+Rs 9,451** |

Day-wise: net negative or flat on 14 of 20 active days; running total sat
at **-Rs 60,946 through 11 Sep**, then jumped to +Rs 16,056 in a single
day (15 Sep, +Rs 77,002) and drifted only modestly from there to the
final +Rs 9,451.

## The real finding: one trade explains the entire result, and it's a genuine crash, not a data bug

**SOLARINDS SHORT (PE) entered 2026-09-15 09:35, premium 519.8 -> exited
2026-09-15 15:15 (EOD_SQUARE_OFF) at 1,894.7 = +Rs 68,745 - 727% of the
strategy's entire 30-day combined net P&L.** Strip out this ONE trade and
the other 466 trades net to **-Rs 59,294**, not +Rs 9,451 - a completely
different picture (a clearly losing strategy on ordinary days).

**Verified this is a real market move, not a data-quality artifact**:
SOLARINDS' own 5-min candles that day show open 22,320 -> day low 19,065
-> close 19,240 (prior day's close 22,350) - a genuine **~13.9% single-
day crash**. For a ~Rs22,000 underlying, an ATM PE going from 519.8 to
1,894.7 (3.6x) across a move this size is consistent with real option
math (gamma acceleration as the strike goes from near-the-money to deep
ITM), not a stale-quote or bad-print artifact. The strategy's own RSI-
bearish-cross-then-candle-confirmation entry correctly caught the front
of this move at 09:35, well before the bulk of the decline, and correctly
held it to end-of-day rather than getting stopped out early.

**Same win-rate profile as a trend/breakout system**: 9.9% overall win
rate with net-positive P&L is the classic "lose small very often, win big
very rarely" shape (consistent with every other symbol's own per-symbol
numbers - DLF/ZYDUSLIFE/SONACOMS/CIPLA/VEDL are all net negative with
5-8% win rates, exactly what "hundreds of small stopped-out RSI-cross
attempts, no big catch yet" looks like). **The headline "+Rs 9,451, 9.9%
win rate" should NOT be read as this strategy being reliably profitable**
- it is one large, real, correctly-caught volatility event carrying a
structurally negative-expectancy setup (as measured over this specific
30-day window). A different 30-day window without a comparable crash
would very likely show the underlying -Rs ~59K-per-30-days character
this sample's other 466 trades already show.

**Other standard caveats** (same as every backtest in this repo): no
slippage/brokerage modeled, ATM-strike liquidity filtering is a disclosed
gap (see other scripts' own docstrings), one 30-day window is one sample
- doubly so here given how much of it rides on a single day's event.

**Outcome:** informational only - not deployed, not wired into Swing.
Purely a standalone backtest artifact. Given the extreme single-trade
concentration found here, this is the weakest of this session's four
video-derived strategies to consider for further iteration without first
re-testing over a materially longer window to see how often (if ever)
comparable catches recur relative to the steady drip of small losses.
