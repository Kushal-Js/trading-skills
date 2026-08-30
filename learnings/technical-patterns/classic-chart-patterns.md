# Classic technical setups — a broader reference alongside VCP

Source: cross-referenced across TrendSpider, TraderLion, tradingsim.com,
ChartMill, bapital.com. Researched 30 Aug 2026. Written as a companion to
`vcp.md` and `minervini-trend-template.md` — VCP is one specific,
well-defined setup; these are the other commonly-taught ones worth knowing
so a "technical setup" isn't assumed to mean only VCP.

## Cup and handle (bullish continuation)

Price rounds out a U-shaped base (the "cup") over an extended period, then
pulls back in a smaller, shallower consolidation on the right side (the
"handle") — often itself shaped like a small flag or pennant — before
breaking above the handle's high on volume meaningfully above average
(commonly cited: at least ~40% above the 20-day average volume). Stop
typically sits below the handle's low. Takes weeks to form properly on
daily/weekly charts — like VCP, this is a multi-week structural pattern,
not something to look for intraday.

## Flags and pennants (continuation patterns, either direction)

A sharp directional move (the "flagpole") followed by a brief, tight,
counter-trend consolidation (the "flag" — parallel channel, or "pennant" —
converging triangle) on declining volume, resolving with a breakout
continuing the original direction on rising volume. Meaningfully shorter-
duration than cup-and-handle or VCP — these can and do form on intraday
timeframes, which is the relevant one for DhanBoy's own holding period.
Conceptually this is close to what the bot's own Supertrend-based entry
trigger is already trying to catch (a pause in an established move,
followed by a continuation) — worth thinking about whether an explicit
flag/pennant detector adds anything beyond what Supertrend+ROC already
capture, or is redundant with it.

## Breakout / pullback-to-support (the general category underneath most of
the above)

Continuation patterns share a common logic: they appear *mid-trend*,
represent a pause rather than a reversal, and are higher-probability when
traded *in the direction of the already-established, dominant trend* —
not as a standalone reversal bet. This is the throughline connecting VCP,
cup-and-handle, and flags/pennants: all three are variations on "the trend
paused, now confirm it's resuming," just at different characteristic
durations (VCP/cup-and-handle: weeks; flags/pennants: days to intraday).

## Where this fits for DhanBoy

Given the bot's own intraday holding period, the most directly relevant
patterns from this list are **flags/pennants** (intraday-scale, same
timeframe as the strategy itself) — worth evaluating as a possible
additional or alternative signal to the existing Supertrend/ROC stack,
since it's measuring a related but not identical thing (a specific
geometric consolidation shape + volume behavior, vs. a trend-following
indicator crossing a threshold). **Cup-and-handle** functions the same way
VCP does for this bot's purposes — a slower, daily-chart context signal for
which underlyings are worth watching, not an intraday trigger itself.

No decision made yet on whether to actually build detection for any of
these beyond VCP — this file exists so "technical setup" knowledge isn't
narrowed to VCP alone when the F&O screener design (or any future one) gets
revisited.
