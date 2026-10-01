# Hedging Swing with Super Bollinger's hedge - 30-day backtest (1 Oct 2026)

**Question (user, 1 Oct 2026):** Swing paper made strong money 29 Sep - 1 Oct; would Super Bollinger's hedge
(-1,800 open loss + 1 ATR(14, 5-min) against the entry -> buy the opposite ATM option, stop 1,500, 30% trail from
+1,000, 15:15) reduce Swing's losses?

**On one day it looked great:** 1 Oct's 17 actual Swing trades +22,623 -> +34,223 with the hedge (3 hedges, two big
trend trails). Over 29 Sep - 1 Oct the same rule was -1,132 (29 Sep alone -9,782).

**Over 30 days it does not help** (traderBoy `0119e90`, research_swing_hedge_variants_30day.py; Swing simulated on
its live rules over 1 Sep - 1 Oct, 15-stock list, real option prices, modelled slippage):

| variant | hedge legs | hedges (stopped) | max drawdown |
|---|---|---|---|
| Swing alone (-109,536 modelled / -13,535 raw) | - | - | -156k |
| SB rule (1 ATR) | -4,052 | 130 (49) | -174k |
| 1.5 ATR | -4,090 | 104 (37) | -165k |
| reverse at Swing's losing Supertrend exit | -13,973 | 102 (34) | -179k |

**Learnings**
- A hedge that wins on a trending day loses on choppy ones; judge it on weeks, never on the day that prompted it.
- Swing's problem is its own edge after costs: ~96k of modelled slippage on 328 trades. Cheap, huge-lot options
  (MOTHERSON Rs 3.4 x 6,150) cost more in slippage (-25.5k) than they ever make; RBLBANK raw +20.3k -> +5.7k.
- Paper results that include instant same-side re-entries (fixed for paper too in 33916b1) overstate Swing.
- Universe = the 1 Oct list (picked partly on this window's data), so Swing's absolute number is flattered;
  the variant comparison is on identical trades.
