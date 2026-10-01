# Parabolic SAR: earlier, but no edge for Unified Momentum or the Scalper (1 Oct 2026)

**Question (user):** can the Parabolic SAR give EARLIER entries and EARLIER reversal exits for Unified Momentum
(pullback calls + momentum puts, 5-min signals) and the BANKNIFTY Scalper (1-min signals)?
Sources read: Investopedia's introduction (trend dots, flips, trailing stop, whipsaws in ranges, filter with a
long-term MA / ADX) and QuantInsti's formula + code (TA-Lib `SAR(high, low, 0.02, 0.2)`, signal = close vs SAR).

**Implementation check.** Wilder's PSAR (SAR(next) = SAR + AF x (EP - SAR), AF 0.02 +0.02 per new extreme up to
0.20, SAR never inside the two prior bars, flip when the low/high crosses, new SAR = old extreme). Cross-checked
against a line-by-line port of TA-Lib's TA_SAR: 0 trend mismatches and 0 mismatches with QuantInsti's close-vs-SAR
signal on BANKNIFTY 1-min (16,338 bars) and three stocks' 5-min series; SAR values identical after ~11 start-up bars.
Two settings fixed before looking at results: 0.02/0.02/0.20 (std) and 0.01/0.01/0.10 (slow).

**It IS earlier.** PSAR was already on the signal's side at 89% of the Scalper's Supertrend entry signals (median
4 min earlier) and at 88% of engine B's bearish 5-min signals (median 20 min earlier). Engine A's pullback entries
mostly come with PSAR already bullish (90%).

**But earlier is not better** (traderBoy `research_psar_um_scalper.py`, results
`research_results/2026-10-01_psar_um_scalper.txt`, cache only):
- Scalper, live settings, 1 lot, 1 Sep - 1 Oct, after Dhan charges + 1 pt slippage per fill: baseline -5.8k. Every
  PSAR variant is worse in BOTH settings - filter -10.6k / -13.7k, + PSAR exit -16.2k / -15.0k, PSAR replaces the
  Supertrend exit -15.9k / -18.9k, extra PSAR-flip entries -25.9k / -18.7k, PSAR-only entries -45.1k / -31.9k,
  full PSAR system -52.6k / -38.5k, PSAR exit only while losing -20.1k / -14.6k.
- Unified Momentum, deployed config, 3 Aug - 29 Sep, baseline +146.0k: engine A PSAR exits +42.1k / +46.8k (std)
  and +127.3k / +136.7k (slow) - an accelerating stop cuts the big trend-day winners (top 5 days = 84% of profit);
  A loss-cut exit +86.3k (std) / +143.5k (slow); B loss-cut exit +145.3k / +139.6k. Engine B variants flip sign between the two settings (PSAR-flip entries -13.1k std vs
  +8.7k..+16.7k slow on ~21 trades; exits +2.8k/+4.5k std vs -4.4k/-1.6k slow) - noise, not an edge.

**Robustness tally:** of 26 UM variants, NONE beats the baseline in both months with both settings; the best-looking
(slow-PSAR momentum-put entries, +14k..+17k) loses with the std setting.

**Lessons**
1. An indicator that reacts earlier mostly reacts to noise on 1-min/5-min intraday data; with option spreads and
   charges each extra whipsaw costs real money. "Earlier" has to be paid for.
2. Strategies that earn from riding a few big trend days are hurt by any accelerating trailing stop (PSAR's AF
   tightens the stop exactly when the trend is strongest).
3. Robustness test: a variant only counts if BOTH parameter settings beat the baseline in BOTH halves. A sign flip
   between AF 0.02 and 0.01 means the "edge" is noise.
4. Hindsight trap: "PSAR turned against 74 of 79 trades that later hit the Supertrend exit, +18k if exited there"
   conditions on the outcome. Applied forward ("exit on a PSAR flip only while losing") it lost MORE (-20.1k vs
   -5.8k), because the same flips also cut trades that later recovered.

**Side finding:** the live Scalper rules themselves backtest at -5.8k after costs over 1 Sep - 1 Oct on 1 lot
(raw +9.9k, 8/22 winning days, max drawdown -22.5k) - flagged to the user.
