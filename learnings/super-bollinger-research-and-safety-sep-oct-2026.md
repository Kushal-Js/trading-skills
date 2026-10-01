# Super Bollinger: what the 30 Sep - 1 Oct 2026 research and safety work found

Six findings from the work that turned Super Bollinger into the bot's only real strategy. They cover the
entry filter, stock selection, the August walk-forward, capital and slots, backtest realism, and the
ratcheted broker stop with its double-sell guard.

All the backtests ran on 3 Aug - 29 Sep 2026. Every rule was chosen in-sample on those two months, with no
out-of-sample data left. Read the rupee figures as rough sizes, not forecasts. Scripts and outputs are in
traderBoy (`research_*.py`, `research_results/2026-09-30_*.txt`).

Related: [`stock-selection-walkforward-hindsight-bias.md`](stock-selection-walkforward-hindsight-bias.md)
(the hindsight trap), [`super-bollinger-scale-in-out.md`](super-bollinger-scale-in-out.md) (S1/S2),
[`price-path-cost-rest-budget-and-memory.md`](price-path-cost-rest-budget-and-memory.md).

## 1. August walk-forward: selection has some value, and luck is large

The test (`research_selection_walkforward_aug.py`): CE-only base rules, a weekly walk-forward, modeled P&L.

| Rule | Total | August | September |
|---|---|---|---|
| HYBRID, top 15 (live rule) | +63.5k | +17.1k | +46.4k |
| Mean of 10 random lists | about -31k | about -12.5k | - |

- HYBRID beat 9 of the 10 random lists overall and 8 of 10 in August. One random draw did better (+74.4k).
- Old weekly ATH, momentum and a weak-stock list all lost money.
- HYBRID's first three August weeks were -22.4k, -14.1k and -3.8k; it lost to most random lists in those weeks.
- NIFTY fell in all nine weeks.

**Lesson:** a narrower list made August worse (top 8 +3.3k, top 5 -12.8k). The August fix was the entry
filter in section 2, not a different list.

## 2. Entry filter: the last closed 1-hour candle must be green

The search tried about 60 filters on the 193-trade August+September HYBRID call book: 1-hour structure, CHOP,
ADX, efficiency ratio, NIFTY regime, day context and time of entry. Scripts:
`research_super_bollinger_chop_filters.py` and `_chop_filter_resim.py`.

**Winner:** the stock's last closed 1-hour candle (09:15-anchored) is green at entry. Re-simulated with slot
limits:

| Book | With the filter | Without |
|---|---|---|
| Plain calls (163 trades) | +94.9k, drawdown -22.8k | +63.5k, drawdown -45.8k |
| Calls + hedge + S1 | +146.0k | +124.2k |

- The three losing August weeks became -9.6k, -0.5k and +6.8k.
- Of the 37 trades it dropped, 30 were losers.

**Robustness:**
- The drawdown cut holds for 30-, 45-, 60- and 90-minute candles.
- The extra profit holds only for 60- and 90-minute candles anchored at 09:15 or 09:45.
- So the drawdown cut is the real effect, and the extra profit is partly luck. About 60 filters were tried.

**What hurt:**
- every NIFTY or market filter;
- strict trend filters (1-hour Supertrend, higher-high/higher-low, above the 1-hour EMA20, ADX, efficiency ratio);
- "above the first 30-minute high" and "above the 20-day average".

Live since `83dc499` (30 Sep) as `entry_filter_1h=on`. A filter that only removes trades was safe to switch
on for real.

## 3. Stock selection: HYBRID's edge depends on the market regime

HYBRID = liquidity gate -> ATH top 40 -> best 20-session strategy fit -> top 15. It beat 98% of random lists
over July-September and was positive in each month. The split by regime:

| Weeks picked while NIFTY was... | HYBRID | Beats random | Fit -> next-week rank correlation |
|---|---|---|---|
| Below its 20-day average (7 weeks) | +28.2 | 100% | +0.18 |
| Above its 20-day average (7 weeks) | +6.0 | 54%, no better than random | -0.10 |

- The three weak August weeks were all "above" weeks.
- Switching to "resilient" stocks in those weeks was worse.
- The only rule that scored above HYBRID was a 40-session fit lookback (1 of 17 rules tried, not proven). It now
  runs as a recorded and scored **shadow list**, traded on paper only, and archived to git weekly.

**Lesson:** the list was not the problem; the market regime was. The lever in "NIFTY above" weeks is fewer
trades, not a different list (section 4).

## 4. Capital and slots: capital is the real limit

On 30 Sep the account had Rs 61k available. One call lot costs a median Rs 22.8k and up to Rs 55k.

Peak premium tied up at once:

| Setup | 4 slots | 2 slots |
|---|---|---|
| Calls + hedge | 2.06L | 1.14L |
| Calls + hedge + S1 + S2 | 3.08L | 1.54L |

Even at 4 slots, today's setup goes above Rs 61k on 25 of 41 days.

Cash-limited replay with a Rs 61k account (legs skipped when cash is short):

| Setup | 4 slots | 2 slots |
|---|---|---|
| Calls + hedge | +66.0k | +83.7k, drawdown -17k |
| + S1 + S2 | +90.6k | +115.8k |

At 4 slots, 44 calls and 13 hedges were skipped for lack of cash.

So the decisions were:
- **2 slots live**: at this balance, 2 slots beat 4 and leave cash for hedges.
- S1/S2 stay on paper: they need about 1L more at the peak, and their real-order version is not built.

Ideas backtested but parked as noise or one-episode results:
- **Fewer slots on "NIFTY above" days:** +106.8k vs +90.6k, drawdown -18.1k vs -23.5k. Only 11 days, all in one
  August stretch.
- **Stepped hedge trail (30/25/20%):** +7.0k vs +0.3k over 56 hedges.

Every call profit-trail variant lost money (-5k to -99k). Calls keep the breakeven stop only.

## 5. Backtest realism: discount the results by about 15-20%

Checks: `cc8f84d`, `c74d89f`, `research_results/2026-09-30_ws_vs_rest_candle_gap.txt`.

- The live trigger check sees WebSocket-built bars.
  - In 31% of bars the live bar's high is below the REST bar's high; in 5.4% by more than 0.05%.
  - Of 23 BULLISH trigger touches: 20 were seen in the same bar, 1 was late, 1 was missed and 1 had no live bar.
- 18 of the 23 entries the REST backtest takes were also taken live.
- Requiring the price to go past the trigger by a margin:

  | Margin | P&L |
  |---|---|
  | none (base) | +81.5k |
  | 0.02% | +70.9k |
  | 0.05% | +68.1k |
  | 0.10% | +34.0k |

- Randomly dropping 20% of entries gives a median of +66.0k.

**Lesson:** treat REST-bar backtests of touch-entry strategies as about 15-20% optimistic. A hedge or trail
modeled on 1-minute closes understates what a resting broker stop gets: on 56 hedges the 30% trail made
+297 on 1-minute closes and +1,640 when the exit fills on a touch.

## 6. Ratcheted broker stop and the double-sell trap

`7df5966`, `d49abe4`, `8e3dc91` and `4aeff3a`, all 1 Oct.

**The gap it closes:** each real position already had a broker stop-loss limit order at its loss cap. The
profit-side exits (the call's breakeven stop, the hedge's 30% trail) existed only in the bot. If the bot
restarted, froze or lost its price feed, nothing protected those gains.

**What the ratchet does:**
- It moves the existing broker stop up to the bot's own exit level.
  - Hedge PUT: the stop follows the trail once the trail is armed.
  - Call: the stop moves to entry once the call is +1,500.
- The trigger sits two ticks under the bot's level, so the bot still exits first and the broker stop is the backstop.
- A move needs at least Rs 250 of gain; at most one change every 5 seconds per order; never at or above the live
  price; only upwards.
- After 3 failed moves for an order it stops trying.
- State is saved to disk, so a restart keeps the moved levels.

**The trap it creates:** the bot's exit cancels the resting stop and then sells. Before the fix, a *failed*
cancel was logged and the sell went out anyway. If the stop was still live, there were now two sells for one
lot, and on a flat contract the second one opens a short. Before the ratchet, the stop sat at the loss cap, far
from where the bot exits, so this was nearly impossible. With the stop two ticks under the exit level, it
becomes a real race.

Since 1 Sep, 26 exits found an outstanding sell order. One cancel failed (RBLBANK, 29 Sep); the stop had
already filled, so the position was flat and nothing broke.

**The fix (`d49abe4`, and the same in Swing in `8e3dc91`):** after a failed cancel, read the order's status.

| Order status | What the bot does |
|---|---|
| Filled | Record the exit at the stop's fill price; send no sell. |
| Cancelled, rejected or expired | Sell as before. |
| Still open | Cancel once more. |
| Still open after that, or status unreadable | No exit order. Back off (10 s, 20 s, ...) and retry. The stop keeps protecting the position meanwhile. |

**Belt and braces (`4aeff3a`):**
- The supervisor sweep cancels any bot stop (tag `SL-`, SELL or BUY) resting on a contract the account holds
  none of. Holdings come from Dhan's raw position list across all segments.
- It cancels only after seeing the stop in two sweeps in a row. Manual orders, which have no tag, are never touched.
- If positions can't be read, it cancels nothing.
- It keeps running through the MCX evening session.

**Rule:** any change that moves a protective stop closer to the bot's own exit price must also make every
"cancel, then exit" path fail closed. Never send the exit while the old order's state is unknown.

## 1 Oct 2026 - 1-hour entry filter: candle alignment test (user request)

Question: should the 1-hour-green filter use a candle that runs across the overnight break instead of the
09:15-anchored candle (whose last candle of the day is a 15-minute 15:15-15:30 piece)? Same harness as the
30 Sep filter re-simulation (HYBRID weekly picks 3 Aug - 29 Sep, slot limit, CE only, modelled P&L, cache only,
in-sample). traderBoy `research_super_bollinger_1h_rolling_window.py`.

| last closed candle green | trades | plain | max dd | + hedge + S1 | before 10:15 |
|---|---|---|---|---|---|
| no filter | 193 | +63,495 | -45,752 | +124,150 | 36 / +6,189 |
| 09:15-anchored (live) | 163 | +94,945 | -22,773 | +145,997 | 24 / +19,873 |
| rolling last 60 min | 184 | +61,621 | -28,341 | +118,897 | 36 / +6,189 |
| grid at 09:45, overnight 15:00->09:45 | 175 | +53,374 | -50,346 | +122,930 | 34 / -2,666 |
| grid at 09:30, overnight 14:45->09:30 | 185 | +52,836 | -48,796 | +116,978 | 36 / +6,189 |

- A rolling hour ending at the trigger is nearly always green for an upside band breakout (the breakout itself
  made it green) - it repeats the signal instead of filtering it (13 skips vs 42).
- Shifting the hourly grid by 15-30 minutes turns the filter's gain into a loss vs no filter. The 09:15-anchored
  edge is therefore sensitive to candle alignment - treat it as fragile/possibly partly luck (in-sample), and
  watch its live skips (ENTRY_SKIPPED_1H_RED) against what those trades would have done.
- Live rule unchanged (user's decision pending at the time of writing).

## 1 Oct 2026 - Super Bollinger's hedge / scale-in on SWING's actual trades (user request)

traderBoy `research_swing_hedge_scale_actual_trades.py` (`082b566`, `b399c84`). 47 closed Swing NSE option
trades entered 29 Sep - 1 Oct (paper + 3 real SwingIndex; 28 Sep untestable - contracts expired 29 Sep, the
intraday API returns nothing; MCX left out). Real 1-min prices, modelled slippage, added legs closed 15:15.
Swing as traded +1,698.

| hedge trigger | hedges | stopped | hedge legs | + S1 |
|---|---|---|---|---|
| SB rule: -1,800 + 1 ATR (1-min) | 15 | 8 | -8,116 | +4,810 |
| + Supertrend against | 0 | - | 0 | - |
| Supertrend against only | 4 | 3 | -4,958 | -92 |
| 1 ATR on a CLOSED 5-min bar | 12 | 5 | -2,643 | +4,810 |
| 1.5 ATR | 15 | 6 | -4,408 | +6,336 |
| 2 ATR | 12 | 6 | -8,214 | +1,499 |
| reverse at Swing's losing Supertrend exit | 16 | 3 | +1,681 | - |

- Swing's losers bounce after the trigger far more than Super Bollinger's - the SB hedge rule loses on Swing.
- A Supertrend-confirmed hedge cannot fire during a Swing trade: Swing itself exits on the Supertrend flip
  (on the tick), before a closed 5-min bar shows it. The workable form is a stop-and-reverse leg at that exit.
- Confirming the ATR move on a CLOSED 5-min bar (not a 1-min touch) cut hedge losses by two thirds.
- S1's gain is mostly one trade (SONACOMS +4,837); the reverse leg's gain mostly one (NIFTY +3,676).
  Three days, 12-16 legs: no conclusion either way - needs the Aug-Sep replay or a separate paper variant.
