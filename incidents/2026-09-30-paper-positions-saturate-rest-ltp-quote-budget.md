# 2026-09-30: Paper positions used up the REST price budget real positions need

**What happened.** At the 30 Sep open, about 8 paper option positions were open (mostly Swing, plus Bollinger's). Their contracts are thin, so the WS tick for each often went quiet for more than 5 s (`LTP_STALE_AFTER_SECONDS`). Every such check fell back to a REST LTP call. Dhan's quote API allows about 1 request per second, and it was saturated:

- 09:15–09:40 IST: ~870 failed REST LTP calls (`Exception at calling ltp`, empty `remarks`, **not** DH-904).
- Most calls succeeded on the 2nd or 3rd retry, but each retry adds 1.5 s. A Swing paper entry took ~7 s from ATM lookup to fill.
- Real money was hit once Super Bollinger's first real trade opened (APLAPOLLO 27 OCT 2220 CE, 09:59). Three books held that same thin contract: Super Bollinger real, Bollinger paper and Swing paper. Four of Super Bollinger's price checks failed completely in 13 minutes (`[SuperBollinger] could not fetch LTP`).
- The broker SL-L (64.20) covered the max loss throughout. The risk was a late breakeven or hedge decision, and a 5-minute run of failures would trigger `LTP_STALE_FORCED_EXIT`.

**Fix.** traderBoy `da4942c`, deployed mid-session at 10:12 IST with the user's explicit approval. The APLAPOLLO CE was open and was re-adopted with its SL order, with no duplicate SL placed.

- Paper positions (Bollinger/Super Bollinger `order_id == "PAPER"`, Swing `product_type == "PAPER"`) now accept a WS tick up to `PAPER_LTP_MAX_AGE_SECONDS` (default 30 s) old before falling back to REST.
- Real positions are unchanged: the 5 s rule and the REST fallback still apply.
- Result over the same 2m15s window: REST LTP failures dropped from 102 to 12, and Super Bollinger's failed checks dropped to 0.
- Cost: paper exits on thin contracts become slightly less precise.

**Found in the same deploy.** The supervisor keeps each CE's entry spot in memory only. After the restart it would have re-seeded the entry spot from the current spot (~2216 instead of 2223.5). Now a re-adopted CE takes its entry spot from today's `POSITION_OPENED` trigger price. At 10:24 the hedge fired on a 10.7-point drop against an ATR of 6.33; measured from the restart spot, the drop was only ~3 points and the hedge would not have fired.

**Follow-up (1 Oct 2026 audit).** The fix did not end the failures. From 13:30 to 15:30 IST there were ~2,900 failed REST LTP calls, mostly on the REAL contracts. 3 real CEs + 2 real hedges, each read 2–3 times per 2 s cycle, exceed ~1/s on their own. 65 of the day's 75 complete price-check failures on real positions fell in that window. See [`learnings/price-path-cost-rest-budget-and-memory.md`](../learnings/price-path-cost-rest-budget-and-memory.md).

**Lesson.** Paper and real share every Dhan rate budget. Whenever a new paper engine polls prices, count its REST calls per second against Dhan's limits as if they were real. Real positions must never queue behind simulated ones.
