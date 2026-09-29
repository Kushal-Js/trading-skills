# COPPER Rs 1k scalp (SB + Swing v3 + Bollinger, TP/SL Rs 1k, momentum re-entry)

Status: backtested 29 Sep 2026, NOT built. Script: traderBoy
`backtest_copper_hft_1k_scalp_10day.py` (data: `copper_hft_fetch.py`,
cache `history/bt_copper_hft_1k/`).

## Setup
- Window 16-29 Sep 2026 (10 MCX days). Signals on the futures contract live
  actually used (SEP to 25 Sep, OCT from 28 Sep), Dhan's own 5/15/60-min bars.
- A = structure-break 5m/15m/1h agreement + 5m Supertrend (live COPPER entry).
  B = A's regime + extra triggers from Swing v3 (ST cross / Day Range) or a
  Bollinger pending-stop fire. C = SB + ST5 + v3 EMA200 regime + BB state all
  agree.
- Exits: TP +Rs1k (resting limit), SL -Rs1k, SB clean reversal, 23:25 flat.
  Re-entry after a TP while SB + ST5 + BB state still on that side.
- OPT = real 1-min bars of ATM COPPER 23 OCT CE/PE (qty 2500); 0.5%-floor
  slippage on market fills; real MCX charges. FUT = 1 lot futures.

## Result (net Rs, 10 days)
| | A SB only | B SB+v3/BB (the request) | C confluence |
|---|---|---|---|
| OPT 1k/1k + re-entry | -464 (20 tr, 60%) | **-8,564** (31, 52%) | -12,491 (28, 46%) |
| FUT 1k/1k + re-entry | -24,691 | -78,829 | -1,08,541 |

## Why it loses - friction, not signal
- ATM copper option (~Rs 25-30 premium): a TP nets ~+Rs 850, an SL nets ~-Rs 1,500
  (entry slippage ~Rs 350 + SL slippage ~Rs 350 + charges ~Rs 160, SL gaps
  through on fast 1-min bars). Breakeven win rate ~64%.
- Futures: charges alone ~Rs 650/round trip (CTT 0.01% on ~Rs 35L notional) +
  2 ticks slippage - a Rs 1k target (0.40/kg, about one 1-min bar's range) is
  pure noise. Never scalp COPPER futures for Rs 1k.
- Adding v3/BB triggers adds trades, not edge: they fire mostly in the same
  chop the SB filter already sits in (28-29 Sep: 19 of B's 31 trades).
- Immediate next-minute re-entry after a TP chases the spike: B's 13
  re-entries summed ~-Rs 4.5k.

## Levers that helped (OPT, 1k/1k)
- Limit (passive) entry instead of market: A +2,764, B -1,596.
- Re-entry only at the next 5m close (not next minute): B -5,537.
- Stop for the day after 2 SLs: B -1,619, C -3,697.
- All three combined: A +3,876 (68% win), B +2,690, C +39.
- Wider targets beat Rs 1k: A 5k/2k +13,775, B 5k/2k +6,292 (in-sample, 15-18 trades).

## Caveats
- Only ~7 days had tradable OCT option prints (SEP options expired, not
  replayable); 20-35 trades per variant - every number above is within noise.
- Limit-entry fills assumed at the bar open (optimistic; misses/adverse
  selection not modelled).
