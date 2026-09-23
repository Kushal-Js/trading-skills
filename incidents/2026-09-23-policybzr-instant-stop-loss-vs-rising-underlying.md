# 2026-09-23: POLICYBZR (PB Fintech) CE stopped out in 37 seconds while the underlying was RISING

## What triggered this

User asked why "PB Fintech" wasn't traded this morning despite being
very bullish - it actually WAS traded (Dhan/NSE ticker `POLICYBZR`, not
"PBFINTECH"), just closed so fast it looked like nothing happened.

Real trade: Luxury, `POLICYBZR 29 SEP 1880 CALL`, entered 2026-09-23
13:30:08 IST @ Rs19.55 (qty 350) off a genuine confirmed breakout
(`range=1.50% body=0.58% relvol=2.07x`) - exited **37 seconds later**
at 13:30:47 IST @ Rs16.40, `reason=STOP_LOSS_HIT`, -Rs1,102.50 (-16.1%).
`sl=16.42` was computed correctly (20% of 19.55); the exit print (16.40)
landed right at that line.

## The suspicious part: the underlying never moved against the trade

Pulled POLICYBZR's real 5-min underlying candles for the exact window
via `/debug/underlying-feed/rest-candles/POLICYBZR`:

| time (IST) | open | high | low | close |
|---|---|---|---|---|
| 13:20 | 1859.0 | 1864.2 | 1855.8 | 1862.6 |
| 13:25 | 1862.2 | 1876.7 | 1861.5 | 1873.0 |
| 13:30 | 1872.5 | 1876.9 | 1868.0 | 1876.0 |

The stock climbed the entire time - never dropped, let alone enough to
explain a 16% CALL-premium collapse in well under a minute. A real
option-pricing move (delta/gamma/theta/vega, all of it) cannot produce
that shape while the underlying is confirmed rising toward the strike.

## Root cause (best available evidence, not 100% provable after the fact)

`get_option_ltp` (`Options/dhan_client.py:1709`) is a raw last-traded-
price read (`get_ltp_data`) with **no spread, volume, or consecutive-
tick sanity check** - the hard-stop poll trusts a single print
unconditionally. The most likely explanation: one thin/illiquid trade
printed well below fair value on the 1880 CE (a strike/expiry combo with
no reason to assume deep liquidity just because the underlying itself is
liquid), and the poll loop treated that single bad print as the real
market and fired `STOP_LOSS_HIT` immediately.

This is the same underlying risk class as the SAGILITY incident
(`learnings/exit-mechanics.md`, `learnings/intraday-options-trading/
liquidity-and-execution.md`) - "a moderately-priced option on a
moderately-liquid underlying can still have a specific strike where one
small trade prints far from fair value" - just manifesting on the STOP
side instead of the ENTRY-slippage side documented there. Not the same
mechanism as the BANDHANBNK trailing-SL case
(`2026-09-23-bandhanbnk-dynamic-sl-over-sensitivity.md`) - that one was
a real price move from a real peak; this one never had a real move in
either direction to justify the exit print.

Also present but likely NOT the primary cause: entry RSI=87.74 (extreme
overbought, shadow filter `recommended_combo_blocks=True`). Per the
BANDHANBNK postmortem's own broader-sample check, this flag alone
doesn't reliably separate winners from losers, so it's not being treated
as the explanation here - the underlying's own confirmed-rising candles
are the stronger, more direct evidence of a bad print rather than a real
reversal.

## Not fixed today - flagged only

No code change made. This touches live stop-loss/order-placement logic
directly, so it needs an explicit decision before altering it, not just
a plausible-cause writeup. Options worth considering if this recurs:
- Require the LTP to persist across 2 consecutive polls before treating
  a stop-loss breach as real (adds latency on a genuine fast move -
  tradeoff, not a free fix).
- Cross-check the option LTP move against the underlying's own
  concurrent move before trusting a single-tick stop trigger (expensive
  per-tick, and delta estimation itself is approximate).
- A volume/OI liquidity floor on the specific contract at entry time
  (same idea already flagged in `liquidity-and-execution.md`'s "Stage 1
  addition" note, not yet built).

Worth a second look if this same shape (large instant stop within
seconds of entry, contradicted by the underlying's own candles) recurs
on another symbol - one occurrence isn't enough to justify the latency
tradeoff of any of the above fixes.
