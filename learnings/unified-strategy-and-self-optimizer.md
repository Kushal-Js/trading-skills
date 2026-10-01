# Unified strategy and the self-optimizer test (1 Oct 2026)

**Question:** combine Super Bollinger (pullback calls + hedge), SwingMomentum (efficiency-ratio momentum) and the rest
into one strategy on Rs 1.2 lakh; make it self-optimizing.

**Setup that keeps it honest:** walk-forward HYBRID weekly picks (no hindsight list), 3 Aug - 29 Sep (41 days), real
option prices, modelled slippage, one account with a cash limit; every config reported for August and September
separately; the self-optimizer's own setting fixed before looking at its result. traderBoy `9f2e00e`.

**What held up**
- Engine A (Super Bollinger pullback calls + hedge + S1/S2) is the profit engine. Premium >= Rs 10 alone added ~+42k.
- Engine B (SwingMomentum) only works on the PUT side as a complement to A's calls (+17.6k over both months); its calls
  lost, and on its own, intraday, on the unbiased universe it is breakeven (Aug -11.1k out-of-sample).
- A light market chop gate (no new entries while NIFTY's 2 h efficiency ratio < 0.10-0.12) improved profit and
  drawdown; stricter gates cost profit monotonically.
- Final (A 2 slots + B 2 slots PUTs + gate 0.10): +181.6k, max dd -13.5k, worst day -5.5k vs Super Bollinger as live
  +87.8k / -16.2k / -10.9k on the same account and lists.

**What did not**
- Self-optimizing the configuration from the last 10-20 days (weekly or daily re-pick): worse than fixed settings
  (a-priori variant +66.1k; median of 16 variants +102.4k vs fixed median +134.0k). Short windows are noise; the
  optimizer keeps switching to whatever just got lucky. Adapt through real-time filters (stock pick, chop gate,
  momentum state) instead of re-tuning parameters.
- Daily loss stop (6k), per-leg cap (45k), hedging the momentum engine.

**Risks:** two stocks (LAURUSLABS, BOSCHLTD) made 69% of the profit; rules were developed on the same two months;
61% of trades lose (profit from a few trend days).
