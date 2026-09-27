# "Dual RSI 50-10" system (Bharat Jhunjhunwala) - RSI(50) trend filter + RSI(10) 60/40 entry, 30-day backtest

**Date:** 27 Sep 2026
**Source:** YouTube video "You're Using RSI WRONG - The Dual RSI 50-10
System That Changes Everything" (Bharat Jhunjhunwala),
https://www.youtube.com/watch?v=EaAgHNcurAM - **no captions/transcript
available at all** (player showed "Subtitles/closed captions unavailable",
transcript panel opened empty). Rules pulled from the video's own written
description instead (quoted in the script's docstring), not from a
transcript.
**Scope:** backtest script only
(`traderBoy/backtest_dual_rsi_50_10_bharat_jhunjhunwala_30day.py`), NOT
wired into any live package. Cash-equity underlying only (no options
premium modeled). 6-symbol watchlist = the real NSE names the video's
description explicitly cites as chart examples: Tata Motors, Kotak
Mahindra Bank, CG Power, Zee Entertainment, Navin Fluorine, Tata
Communications. Bitcoin/Ethereum (also named) excluded - no crypto data
path in this Dhan-based repo. Last 30 trading days, both 5-min and 15-min
timeframes swept.

**Real-world data note:** Tata Motors demerged in 2025 into separate
Commercial/Passenger-Vehicle listings - the original TATAMOTORS ticker no
longer trades. Substituted TMPV (Tata Motors Passenger Vehicles,
security_id 3456 in Dhan's instrument master), the closest currently-
tradeable successor.

## The rules, as stated in the video's description (quoted)

- "The 50-Period RSI Trend Identifier - RSI above 50 means uptrend, below
  50 means downtrend."
- "The 10-Period RSI Entry System - using a shorter lookback to time EXACT
  pullback entries"
- "RSI crossing 60 and 40 on the 10-period as precise buy and sell
  triggers"
- "The 50-10 RSI Combo - how combining both lookback periods creates a
  complete system for trend identification AND entry timing in one
  indicator" - i.e. both RSI(50) and RSI(10) are computed on the SAME
  chart/timeframe (unlike the dual-EMA-band video's separate daily-bias +
  intraday-entry split - see [[dual-ema-band-directional-strategy-animesh-k]]).
- "Works for both LONG and SHORT setups with equal precision."

## Interpretation calls made (full detail in the script's own docstring)

- **Entry:** LONG when RSI(50)>50 AND RSI(10) closes-crosses up through 60.
  SHORT when RSI(50)<50 AND RSI(10) closes-crosses down through 40.
- **Exit (assumption - not stated in the description):** the OPPOSITE
  60/40 trigger - LONG exits on RSI(10) crossing back down through 40,
  SHORT exits on RSI(10) crossing back up through 60. Chosen as the most
  literal reading of "60 and 40 ... as buy AND sell triggers" without
  inventing an unstated third rule (no fixed R-target or trend-flip exit
  is ever mentioned).
- **Holding period (assumption):** modeled INTRADAY - square-off at 15:15
  IST (this repo's own live `SQUARE_OFF_TIME` convention), no new entries
  after 15:00 IST. The description's hashtags mix #SwingTrading and
  #IntradayTrading and never settle it; picked intraday because the
  video's own top YouTube comment describes testing it on "Nifty ke 2 aur
  5 minute" charts (explicitly intraday), and a 10-period RSI on 5/15-min
  bars reacts on a ~1hr timescale, naturally intraday rather than
  multi-day swing.
- **Timeframe (unstated - swept, not guessed):** both 5-min and 15-min.
- **Position sizing (assumption):** Rs 100,000 notional per trade,
  qty = floor(capital / entry price), independent per symbol (no shared
  portfolio capital constraint).

## Result

### 5-min

| Symbol | Trades | Win rate | Net P&L |
|---|---|---|---|
| TMPV | 58 | 41.4% | +Rs 3,778 |
| KOTAKBANK | 75 | 33.3% | -Rs 1,053 |
| CGPOWER | 64 | 42.2% | +Rs 2,285 |
| ZEEL | 59 | 35.6% | +Rs 1,639 |
| NAVINFLUOR | 64 | 43.8% | +Rs 31 |
| TATACOMM | 63 | 42.9% | -Rs 883 |
| **COMBINED** | **383** | **39.7%** | **+Rs 5,798** |

### 15-min

| Symbol | Trades | Win rate | Net P&L |
|---|---|---|---|
| TMPV | 27 | 55.6% | +Rs 513 |
| KOTAKBANK | 32 | 40.6% | -Rs 1,988 |
| CGPOWER | 30 | 53.3% | +Rs 561 |
| ZEEL | 32 | 43.8% | +Rs 3,359 |
| NAVINFLUOR | 31 | 48.4% | -Rs 1,014 |
| TATACOMM | 37 | 40.5% | -Rs 10,217 |
| **COMBINED** | **189** | **46.6%** | **-Rs 8,785** |

Both intervals: essentially flat-to-slightly-negative over 30 days on
~1 lakh notional per trade - not a strategy edge, within noise.

## The real finding: the strategy's own stated exit rule is the losing side; the UNMODELED time-based square-off carries all the profit

Broken down by exit reason (5-min, combined across symbols):

| Exit reason | Trades | Net P&L |
|---|---|---|
| RSI10_CROSS_UP_60 (SHORT's own stated exit) | 139 | -Rs 25,110 |
| RSI10_CROSS_DOWN_40 (LONG's own stated exit) | 129 | -Rs 42,080 |
| SESSION_END_SQUARE_OFF (day rollover, not the video's rule) | 57 | +Rs 31,481 |
| SQUARE_OFF_TIME (15:15 IST, not the video's rule) | 57 | +Rs 40,803 |

Same shape at 15-min: the two RSI(10)-cross exits net -Rs 60,903 combined;
the two time-based square-offs net +Rs 51,463 combined. **In both
timeframes, every trade that actually hit the video's own stated 60/40
exit trigger was a net loser as a group, and 100% of the net profit came
from trades that were still open and got force-closed by an unrelated
housekeeping rule (EOD square-off) instead.** This strongly suggests the
60/40-cross exit cuts winners short mid-trend (the same band that
generated the entry re-triggers again almost immediately once price keeps
moving, closing a position that was still working) - the entry signal
looks directionally reasonable, but the exit signal as literally read from
the description is actively harmful, not neutral.

**Other standard caveats** (same as every backtest in this repo): no
slippage/brokerage modeled, one 30-day window is one sample, exit rule is
an explicit interpretation (no transcript was available to confirm the
video's actual stated exit, if any).

**Outcome:** informational only - not deployed, not wired into any live
package. If revisited, the natural next test is dropping the RSI(10)
cross as an exit entirely and letting the time-based square-off (or a
trend-flip-on-RSI50 exit) run the position instead, given how cleanly the
data splits here.
