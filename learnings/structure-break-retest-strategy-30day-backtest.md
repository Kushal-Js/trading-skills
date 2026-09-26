# "structure_break_retest" strategy - swing structure / CHOCH / inducement retest, backtest against the current Swing watchlist

**Date:** 26 Sep 2026
**Source:** YouTube podcast "Trade Big Moves on the 1-Minute Chart | Ft.
Mukesh Jha" (Upsurge Club), https://www.youtube.com/watch?v=b5OOqgoINdk,
~59 minutes, English. YouTube's transcript panel got stuck loading
indefinitely (same recurring issue as this session's other video-derived
backtests) - rules pulled from the video's own description + chapter list
+ direct chart-screenshot inspection at several `movie_player.seekTo()`
points inside the "Understanding the Setup on Charts" (16:40) and "Entry,
Exit & Stop-Loss" (28:30) chapters.
**Scope:** backtest script only
(`traderBoy/backtest_structure_break_retest_swing_watchlist.py`), NOT
wired into `Swing/trading_engine.py` - the **current live Swing
watchlist** as of 26 Sep 2026 ~16:00 IST (pulled fresh via `GET
/swing/watchlist`, not a hardcoded old list): 16 real NSE-equity names
(COPPER/NATURALGAS [MCX] and NIFTY/BANKNIFTY [index] excluded, same
convention as every other video-derived backtest this session -
different instrument resolution, out of scope here) - CANBK, VBL,
SONACOMS, ASHOKLEY, BANDHANBNK, TORNTPHARM, DLF, ZYDUSLIFE, MCX, BOSCHLTD,
APLAPOLLO, LICHSGFIN, RBLBANK, MOTHERSON, DIVISLAB, APOLLOHOSP. Note: "MCX"
here is the ticker for Multi Commodity Exchange of India Ltd, a real
listed NSE equity stock, not the MCX exchange segment. OPTIONS basket,
1-minute underlying/signal timeframe (the video's own explicit framing),
last ~20 available trading days (see caveat below).

## The setup, as pulled from the description + chart screenshots

Description (quoted): a 20-year trader "explains how he approaches
trading in sideways and reversal conditions... getting caught in false
moves, having stop-losses hit and struggling to identify where a reversal
can actually happen... breaks down concepts like swings, major and minor
structure, BOS, CHOCH and inducement... how he marks important levels,
identifies the area of interest and looks for confirmation on lower
timeframes... entries, exits and stop-losses, including the difference
between conservative and aggressive stop-loss placement and how
risk-reward can be planned around the setup."

Chart screenshots (NIFTY/BANKNIFTY 1-min, TradingView, captured at several
seek points) directly confirmed the visual pattern: a multi-leg swing
structure breaks at a marked horizontal level, price returns to retest
that exact level (a small wick piercing slightly beyond it, then closing
back on the break side - the inducement/liquidity sweep), and a strong
directional move follows immediately after the retest holds. This is a
standard "break of structure -> retest -> reversal continuation" setup.

## Interpretation calls (full detail in the script's own docstring)

1. **Swing-pivot size**: never stated numerically. Used a 5-bar-each-side
   fractal pivot on the 1-min chart (a swing needs ~10 minutes of
   confirmation either side) - matches the multi-leg-per-hour structures
   visible in the video's own chart replays.
2. **CHOCH, made concrete**: structure is BULLISH when the last two
   confirmed swing highs AND lows are both rising, BEARISH when both are
   falling. CHOCH-down = a close breaks the last confirmed swing low
   while structure is bullish (and the mirror for CHOCH-up) - the
   standard textbook definition, not the video's own exact words.
3. **Area of interest + inducement + lower-timeframe confirmation**, made
   concrete as one mechanism: the broken swing level IS the area of
   interest. After a CHOCH, a pending retest arms at that level; it
   confirms the first time a later bar's high/low pierces the level (the
   inducement wick) while that same bar's close rejects back to the CHOCH
   side. Entry at the next bar's open. Expires uncalled after 60 bars (1
   hour of 1-min bars) if price never returns.
4. **Stop-loss**: implemented the CONSERVATIVE variant the video names -
   beyond the retest bar's own wick extreme. The video's "aggressive"
   alternative (presumably tighter, just past the confirmation candle's
   body) was NOT implemented - a real, deliberately-skipped option, not
   silently dropped.
5. **Risk-reward**: no ratio given in the video. Used a fixed 2:1
   reward:risk on the underlying's own stop distance.
6. **Exit beyond target/stop**: a fresh confirmed opposite-direction
   signal closes the position (this repo's standard "opposite signal
   exit" convention), plus `MAX_LOSS_PROTECTION_RS` and 15:15 IST EOD
   square-off.

## Caveat: "last 30 trading days" resolved to 20 real days

`TEST_DAYS_BACK=30` was requested (this repo's own default, matching
every other backtest here), but Dhan's 1-minute-interval historical
endpoint only actually returned ~20 trading days of usable data even with
a 90-day lookback requested - the exact same effective retention limit
observed in this session's other 1-min-granularity backtest (the Loren
RSI-candle strategy also landed on "20 active days" for the same reason).
Not a bug in this script; a broker-side data-retention ceiling for 1-min
granularity, consistently reproduced across backtests. Actual window:
**28 Aug 2026 - 25 Sep 2026**.

## Results (16 symbols, 20 trading days, OPTIONS basket)

| Symbol | Trades | Wins | Losses | Win rate | Net P&L |
|---|---:|---:|---:|---:|---:|
| APLAPOLLO | 169 | 90 | 76 | 53.3% | +Rs 39,900 |
| SONACOMS | 150 | 77 | 73 | 51.3% | +Rs 36,505 |
| RBLBANK | 129 | 65 | 64 | 50.4% | +Rs 24,765 |
| BANDHANBNK | 137 | 56 | 80 | 40.9% | +Rs 22,032 |
| BOSCHLTD | 134 | 67 | 67 | 50.0% | +Rs 15,957 |
| DLF | 140 | 63 | 77 | 45.0% | +Rs 8,502 |
| MCX | 154 | 62 | 91 | 40.3% | +Rs 4,624 |
| ZYDUSLIFE | 147 | 65 | 81 | 44.2% | +Rs 2,700 |
| CANBK | 130 | 54 | 76 | 41.5% | +Rs 1,013 |
| APOLLOHOSP | 146 | 62 | 84 | 42.5% | +Rs 631 |
| ASHOKLEY | 147 | 58 | 89 | 39.5% | +Rs 700 |
| VBL | 137 | 53 | 84 | 38.7% | -Rs 829 |
| TORNTPHARM | 125 | 52 | 73 | 41.6% | -Rs 1,875 |
| DIVISLAB | 148 | 57 | 91 | 38.5% | -Rs 5,055 |
| MOTHERSON | 159 | 62 | 96 | 39.0% | -Rs 7,626 |
| LICHSGFIN | 135 | 53 | 74 | 39.3% | -Rs 26,100 |
| **COMBINED** | **2287** | **996** | **1276** | **43.6%** | **+Rs 115,845** |

11 of 16 symbols net positive; LICHSGFIN is the single biggest drag
(-Rs 26,100, the only loss exceeding Rs 10k). ~114 trades/day combined
across 16 symbols (~7 trades/symbol/day) - a genuinely high-frequency
setup at 1-min granularity, consistent with the "big moves happen often
enough on the 1-minute chart" framing of the video's own title.

Day-wise running total climbed steadily through the first half of the
window (+Rs 862 -> +Rs 137,030 by 15 Sep), then gave back roughly
Rs 21,000 of open profit across the back half (three separate down days:
-Rs 21,610 on 8 Sep, -Rs 10,983 on 21 Sep, -Rs 9,119 on 23 Sep) before
closing at +Rs 115,845 - a real, visible give-back pattern worth watching
if this were ever taken further, not a smooth equity curve.

Full day-wise table (20 days) and all 2287 individual trades were
delivered to the user directly (CSV) rather than reproduced here in full.

## Standard caveats (same as every backtest in this repo)

No slippage/brokerage modeled, ATM/near-ATM strike liquidity filtering is
a disclosed gap, one ~20-day window is one sample. The conservative-vs-
aggressive stop-loss choice (interpretation #4) directly trades off win
rate against average loss size - worth a follow-up comparison if this
strategy is ever considered for further iteration.

**Outcome:** informational only - not deployed, not wired into Swing.
Standalone backtest artifact, delivered per the user's explicit request
for day-wise and trade-wise P&L reports.

## Head-to-head vs. the live Bollinger strategy (27 Sep 2026 follow-up)

User asked whether this strategy beats the bot's own live Bollinger
strategy. Reran `backtest_bollinger_vortex_9symbols_30day.py` (unchanged,
existing script) against the EXACT SAME 16-symbol current watchlist and
the exact same `TEST_DAYS_BACK=30` request (which resolved to the same
28 Aug - 25 Sep, 20-trading-day window on this data) for a like-for-like
comparison. Each strategy kept its own NATIVE signal timeframe (Bollinger:
5-min, matching its real live config; structure_break_retest: 1-min,
matching the source video's own explicit framing) - a genuine, disclosed
difference between how each strategy actually operates, not an unfair
mismatch introduced for this comparison.

| Metric | structure_break_retest | Bollinger (live strategy) |
|---|---:|---:|
| Trades | 2,287 | 498 |
| Win rate | 43.6% | **56.9%** |
| Net P&L (20 days) | +Rs 115,845 | **+Rs 210,985** |
| Symbols net positive | 11 / 16 | **16 / 16** |
| Worst single day | -Rs 21,610 (8 Sep) | -Rs 157 (16 Sep, only red day) |
| Trades/day (combined) | ~114 | ~25 |

**Bollinger wins decisively on every dimension that matters** - roughly
82% more net P&L, 13 points higher win rate, every symbol profitable
(structure_break_retest lost money on 5 of 16), and a dramatically
smoother equity curve (Bollinger's day-wise P&L was negative on only ONE
of 20 days; structure_break_retest had three days losing over Rs 9,000
each). structure_break_retest also trades ~4.6x more often for less than
half the profit - a much worse risk-adjusted/effort-adjusted result.

Per-symbol Bollinger results (same 16 symbols, same window):

| Symbol | Trades | Win% | Net P&L |
|---|---:|---:|---:|
| ZYDUSLIFE | 31 | 58.1% | +Rs 28,935 |
| BANDHANBNK | 32 | 65.6% | +Rs 27,648 |
| MCX | 33 | 69.7% | +Rs 21,904 |
| BOSCHLTD | 27 | 63.0% | +Rs 16,000 |
| APLAPOLLO | 36 | 47.2% | +Rs 15,767 |
| LICHSGFIN | 34 | 45.5% | +Rs 12,600 |
| TORNTPHARM | 36 | 66.7% | +Rs 12,269 |
| RBLBANK | 25 | 36.0% | +Rs 12,541 |
| MOTHERSON | 29 | 62.1% | +Rs 10,763 |
| SONACOMS | 35 | 48.6% | +Rs 10,474 |
| DIVISLAB | 25 | 60.0% | +Rs 9,350 |
| ASHOKLEY | 29 | 72.4% | +Rs 8,000 |
| APOLLOHOSP | 30 | 53.3% | +Rs 7,712 |
| DLF | 34 | 52.9% | +Rs 7,220 |
| CANBK | 30 | 46.7% | +Rs 6,615 |
| VBL | 32 | 62.5% | +Rs 3,187 |
| **COMBINED** | **498** | **56.9%** | **+Rs 210,985** |

Note LICHSGFIN specifically: it was structure_break_retest's single
biggest loser (-Rs 26,100) but Bollinger's own trade on the exact same
symbol/window was solidly profitable (+Rs 12,600) - the same underlying
price action, two very different outcomes depending on which setup is
reading it, a useful illustration of why "does the strategy fit this
instrument's own behavior" matters as much as the strategy itself.

Full trade-wise CSV for this Bollinger comparison run delivered to the
user directly (498 trades), same as structure_break_retest's own export.

**Conclusion: no reason to prefer structure_break_retest over the
already-live Bollinger strategy based on this comparison** - it is not
being escalated for further development on the strength of this result.
