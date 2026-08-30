# Incident: SAGILITY MAX_LOSS_HIT overshoot, 28 Aug 2026

## What happened

A CE ATM SAGILITY option position (entry 2.29 @ 10:07:12 IST, qty 12,000)
hit `MAX_LOSS_HIT` at 10:09:18 IST with exit price 2.14 — a realized loss of
**−Rs.1,800** against the live `MAX_LOSS_PER_TRADE_RS=1,200` cap. A
Rs.600 overshoot, ~50% over the intended cap.

This was the specific trade that triggered the whole investigation into
exit-mechanics timing later written up in `learnings/exit-mechanics.md` —
this file is the worked example that produced that general knowledge.

## Initial hypothesis (wrong, corrected during investigation)

First read: "5-second poll interval + fast price movement between polls."
This turned out to be an oversimplification of a mechanism that's actually
event-driven (`on_price_tick`), not purely poll-driven — see below.

## What actually happened, confirmed with real data

Pulled real 1-minute Dhan option candles for `SAGILITY 29 SEP 46 CALL`
(security_id 126173) around the trade window:

| Minute (UTC) | Open | High | Low | Close |
|---|---|---|---|---|
| 04:37 (entry) | 2.33 | 2.33 | 2.25 | 2.25 |
| 04:38 | 2.26 | 2.26 | 2.25 | 2.25 |
| 04:39 (exit) | **2.14** | 2.14 | 2.10 | 2.10 |

Cap threshold price = entry − (cap/qty) = 2.29 − (1200/12000) = **2.19**.
The price sat at 2.25–2.26 for two full minutes (correctly not triggering —
never crossed 2.19), then **gapped straight from 2.25 to 2.14 with zero
trades printing in between**. The bot's exit fired on the very first tick
past the gap (2.14, matching that minute's open — not its worse 2.10 low).

## Verification: replayed the exact tick sequence through the real code

Built `test_sagility_replay.py`, feeding the real 2.25 → 2.14 tick sequence
through the actual `on_price_tick` function (mocking only order placement),
with the later-deployed `LTP_STALE_AFTER_SECONDS` staleness fix both
disabled and enabled. **Identical result both ways**: exit at 2.14,
pnl = −1,800.00, matching the real trade exactly.

**Conclusion: this was a genuine liquidity gap, not a monitoring-latency
bug.** No trade ever printed at 2.19–2.24 that minute — the exit reacted to
the first real price it could possibly react to. Confirmed the same pattern
on two more of that day's SAGILITY `MAX_LOSS_HIT` trades (10:37 IST and
11:33 IST) via the same real-candle-data method; a third was ambiguous
(that candle's range did span the threshold, meaning a real intermediate
print may have existed, but tick-level data to prove it doesn't exist after
the fact).

## Why this matters beyond this one trade

- `on_price_tick` (event-driven) does not consult the LTP cache or any
  staleness setting at all — it acts on whatever price it's handed the
  instant it arrives. This means **responsiveness config changes
  (`MONITOR_INTERVAL_SECONDS`, `LTP_STALE_AFTER_SECONDS`) cannot fix a
  liquidity gap** — there was no earlier real price to react to.
- SAGILITY is representative of a class of risk: thin, low-premium
  (~Rs.2), high-quantity-per-lot (12,000 shares) option contracts where a
  single paisa of price movement between real trades is worth Rs.120 —
  large relative swings are structurally more likely to gap through a fixed
  rupee cap than a similarly-thinly-traded but higher-premium contract
  would.
- The rupee cap (`MAX_LOSS_PER_TRADE_RS`) is a **trigger threshold**, not a
  guaranteed fill price, for any option — this trade is the concrete proof.
  Lowering the cap value doesn't reduce gap risk either (a big enough gap
  blows through any cap value the same way); it only changes how far
  through the (now-lower) threshold price the position typically was before
  a real trade printed.
- What genuinely reduces this specific risk: excluding very thin,
  low-premium, high-quantity contracts like SAGILITY from the strategy
  entirely, or accepting occasional gap losses on such names as a structural
  cost of trading them. Config tuning inside the current exit-monitoring
  logic cannot solve it.

## What the investigation *did* correctly motivate

Even though this specific trade wasn't fixable, the investigation surfaced a
real, separate gap: `get_cached_option_ltp` had no staleness check at all —
if the WebSocket feed genuinely stalled (zero ticks, not just few real
trades) for an extended period, the poll loop's fallback would trust an
arbitrarily old cached price forever. That's a real bug, distinct from the
liquidity-gap mechanism above, and it was fixed via `LTP_STALE_AFTER_SECONDS`
(27 Aug 2026) — see `learnings/exit-mechanics.md` for the mechanism and a
synthetic stress test proving it actually bounds a feed-stall blind window
to ~5 seconds instead of indefinitely. The fix is real and worth keeping;
it just doesn't apply to *this* trade.
