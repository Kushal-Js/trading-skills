# "Trinity" intraday strategy — HA bias + SSL Channel + Z-Score pullback (Animesh K, Upsurge Club)

**Source:** [Upsurge Club podcast ft. Animesh K, "This Trading Strategy
Handles Sideways Markets"](https://www.youtube.com/watch?v=d6ozaHLfSHE),
published 2026-09-23, ~37 min. Same Animesh K as
[[dual-ema-band-directional-strategy-animesh-k]], different setup this time
— sells a paid "Trinity Scalping & Intraday Trading" course covering the
extended version. This is a transcript-derived writeup of the free version
walked through on air — **not something we'd backtested before this
session.** Treat every rule below as a hypothesis, not a confirmed edge.

**Extraction caveat:** YouTube's transcript panel and `timedtext`/
`get_transcript` endpoints both returned empty/400 for this session (a
platform-side restriction, not specific to this video) — no full verbatim
transcript was available. This writeup and the backtest below are built
from the video's own on-screen slides (which spell out both indicator
formulas and thresholds verbatim, captured via screenshots at the relevant
timestamps) plus the description and chapter markers, not a line-by-line
transcript. Less complete than the dual-EMA-band writeup as a result —
noted per-section below where a claim is slide-sourced vs. inferred.

## Core mechanism (as described)

1. **Directional bias — previous day's DAILY Heikin-Ashi candle**, same
   convention as the dual-EMA-band video: HA close > HA open → bullish day
   (longs only); HA close < HA open → bearish day (shorts only). Per the
   video description directly (not just the slides).
2. **Trend filter — SSL Channel** ("Semaphore Signal Level Channel"), per
   its own on-screen slide, transcribed verbatim: *"A trend-following
   indicator that uses moving averages of Highs and Lows to determine
   trend direction... SSL Channel plots two moving averages: High MA =
   Moving Average of Highs, Low MA = Moving Average of Lows... When price
   closes above the channel (High MA > Low MA) → Bullish Trend... When
   price closes below the channel (High MA < Low MA) → Bearish Trend...
   When the lines cross → Potential Trend Change (Bullish/Bearish
   Crossover = Buy/Sell Signal)."* This is the well-known public "SSL
   Channel" indicator (Erwin Beckers' original TradingView script), not
   something custom. **The lookback period was never shown/stated in the
   segments reviewed** — the original public indicator's default is 10.
3. **Entry timing — Z-Score pullback**, per its own on-screen "QUICK
   GUIDE" slide, transcribed verbatim:
   - `Z > +2` → Unusually High (Potential Sell/Caution)
   - `+1 < Z ≤ +2` → Moderately High
   - `-1 ≤ Z ≤ +1` → Normal/Average Range
   - `-2 ≤ Z < -1` → Moderately Low
   - `Z < -2` → Unusually Low (Potential Buy/Caution)

   Standard formula shown on the same slide: `Z = (X - μ) / σ`. **Neither
   the price series used for X nor the rolling window for μ/σ was shown in
   the segments reviewed.** The "Understanding the Pullback System"
   chapter (16:47) title strongly implies Z-score extremes are used to
   time entries into pullbacks *within* the SSL-confirmed trend (buy the
   dip in an uptrend, sell the rally in a downtrend) rather than as a
   standalone mean-reversion signal — this is an inference from chapter
   sequencing, not a transcribed rule.
4. **Entry/Exit/Target** (chapter at 15:00) walks a live chart example
   narrating confluence — e.g. "the candle doesn't cut through its own
   level" as a confirmation cue — rather than stating a fixed numeric
   stop-loss % or target. **No fixed SL/target ratio was captured.**

## What's genuinely unclear (viewer comments on the video corroborate this)

- `@Kafir_Hindu`: "Z score and SSL not available in angelOne. Is it not
  the std indicator." — i.e. even viewers weren't sure if these are
  broker-standard indicators or the specific custom-tuned versions from
  the paid course.
- `@jaiprakashn9686`: calls the setup overcomplicated for beginners vs.
  Animesh's own earlier, simpler EMA(34) High/Low strategy (the
  dual-EMA-band one already in this repo).
- General pattern: this video, like the dual-EMA-band one, is an
  interview walkthrough, not a spec sheet. Exact numeric parameters
  (lookback periods, SL%, target ratio) are consistently the part left
  out of the free YouTube content and reserved for the paid course.

## Backtest result (30 trading days, 2026-08-14 to 2026-09-25)

Run via `traderBoy/backtest_trinity_ha_zscore_ssl_nifty_30day.py` (not
committed — ad-hoc script, same pattern as the repo's other untracked
`backtest_*.py` files). Tests a **mechanical interpretation** of the three
rules above, on the **NIFTY 50 index underlying** (not options), reported
in index points — options premium P&L, position sizing and the stock
sector/pick layer are NOT modeled. Auth used a hand-off access token
(`access_token` mode), never local `pin_totp`, per
[[feedback-live-trading-safety]] (this ran during live market hours).

**Assumptions made to fill the two gaps above** (flagged as assumptions,
not transcribed numbers):
- SSL Channel period = 10 (the public indicator's own default)
- Z-score window = 20 (a standard default for a rolling Z-score)
- Z-score pullback trigger = the video's own "moderate" boundary (±1):
  wait for Z to breach ±1 against the trade direction, then cross back —
  that crossback bar is the entry
- Exit = SSL trend flip (trailing exit on the trend indicator itself
  reversing) — no fixed SL/target modeled, since none was captured
- Daily bias must agree with SSL trend, or no trade that day

Both untested intraday timeframes swept, since the video never states one
(same convention as the dual-EMA-band backtest):

| Interval | Trades | Win rate | Avg win | Avg loss | Net (index pts) |
|---|---|---|---|---|---|
| 5-min  | 19 | 26.3% (5W/14L) | +22.3 | -21.0 | **-182.8** |
| 15-min | 5  | 40.0% (2W/3L)  | +139.6 | -51.4 | **+125.1** |

5-min is net negative and the dominant loss driver is the trend-flip exit
itself (`SSL_TREND_FLIPPED_UP`: 11 trades, -128.1 pts;
`SSL_TREND_FLIPPED_DOWN`: 6 trades, -104.2 pts) — i.e. most pullback
entries get stopped out again almost immediately by the same trend filter
that gated the entry, a whipsaw pattern. 15-min is net positive but on
only **5 trades total** over the month — far too small a sample to read
anything into; both winners were session-end square-offs (avg +139.6),
both losers were trend-flip exits.

**Caveats before reading anything into either number**: one 30-day window,
one instrument, no slippage/spread/brokerage, no real option premium/theta
(index points ≠ rupee P&L), and — the important one this time — **two of
the three core parameters (SSL period, Z-score window) are assumed
defaults, not numbers from the video.** Unlike the dual-EMA-band backtest
(where every parameter came directly from the transcript), this result
should be read as "does this general HA+trend-filter+pullback-timing
*shape* of strategy show any edge on NIFTY," not as a faithful
reproduction of what Animesh actually trades. If revisiting, the SSL
period and Z-score window are the two knobs most likely to change the
picture — worth a parameter sweep before drawing any conclusion, the same
way `INTERVALS_TO_TEST` was swept here. Full trade-by-trade JSON:
`/tmp/trinity_backtest_cache/results_trinity_nifty_30day.json` on the
machine the backtest was run from (not synced anywhere).

## Why this is here, not acted on

Per [[project-trading-skills-repo]] this repo captures external ideas
worth evaluating, it does not imply endorsement. The 5-min result here is
net negative and the 15-min result is a 5-trade sample — neither clears
any bar for consideration as a real strategy. If this gets revisited with
the parameter sweep noted above, log the result as a new file here (or
update this one) citing the actual numbers.
