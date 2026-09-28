# 2026-09-28: manual exit not detected (NATURALGAS 23 OCT 300 CALL)

**What happened.** Swing opened a real NATURALGAS 23 OCT 300 CALL at 15:36 IST at 18.60, with a broker SL-L attached. At 16:33 the user closed it by hand in the Dhan app:

1. A market sell was REJECTED, because the resting SL-L held the quantity.
2. The user cancelled the SL-L.
3. The user sold at market at 17.40, for −Rs 1,500.

The bot logged only "stop-loss order ... ended as CANCELLED without firing" and **kept tracking the position**. Its next exit signal would have sent a real SELL for a contract no longer held, i.e. a naked short. Found by checking the broker directly (`/v2/positions`, `/v2/orders`). Cleared by a restart: reconciliation only rebuilds positions the broker holds.

**Root cause.** Nothing compared the bot's position with the broker's quantity before selling. The existing flat check ran only after an exit failure (`exit_failure_count >= 1`) or after the bot itself cancelled a stale order.

**Fix.** traderBoy `68301d1`, deployed 16:40 IST, in all four packages:

- New `broker_flat_check.confirmed_flat()`: flat only on two zero net-quantity reads 2s apart. The positions API can lag a fill; Bollinger has exited 28s after entry. Any read error → unknown, and the old behaviour is kept.
- When the SL-L ends without firing and the broker is confirmed flat → `MANUAL_EXIT_DETECTED`, no order sent.
- Before the first exit order, the same check → `MANUAL_EXIT_DETECTED` instead of selling.

**Gap left.** A manual exit is recorded at the cached LTP, not the real manual fill price, so the bot's trade log P&L for such a trade is approximate. The broker's own P&L is the truth. Here it was −Rs 1,500.
