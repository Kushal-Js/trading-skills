# 2026-09-28 — COPPER 23 OCT 1400 PUT exited "early" on PROFIT_PROTECTION_HIT

**Strategy:** Swing (MCX carve-out, real money). **Result:** +Rs 1,900.

## Timeline (IST)

| Time | Event |
|---|---|
| 10:01:28 | Entry: BUY 1 lot (2,500 multiplier) @ 22.19 on structure-break BEARISH. Broker SL-L at 20.39. Target 26.63, hard SL 17.75. |
| 10:01–12:05 | REST `No LTP returned` for this contract about 40 times. No WS ticks were cached, so the contract barely traded. |
| 11:12:50 | Bot restarted by another session's deploy. Position reconciled with `best_price` reset to 22.19. |
| 12:05:44 | `PROFIT_PROTECTION_HIT`. The bot cancelled the resting SL-L and sold at market. Filled @ 22.95. |

COPPER futures traded 1404.3–1409.0 all session (about 0.3%). The structure-break signal was still −1 (bearish) at 12:15, so the trade's thesis had not reversed. This was not a trend exit.

## Why it fired

MCX-options profit protection is `SWING_PROFIT_PROTECTION_RS_MCX=4000` with `..._GIVEBACK_PCT_MCX=0.02`:

- It arms once the peak premium is at least 22.19 + 4000/2500 = **23.79**.
- After that it exits on any LTP ≤ 0.98 × peak, which is about **0.48 premium points** (Rs 1,190) below the peak.

With the underlying flat, a print of 23.79 or higher after 11:12 was almost certainly a spread or odd-lot print in a thin contract, not a real move. The next print back near the mid tripped the 2% floor. The market sell then filled 0.3–0.4 below the floor. So the rule armed at Rs 4,000 of paper profit but banked Rs 1,900.

**Confirmed later the same day** from the contract's real 1-min candles (user-supplied read-only token):

- 11:55: 1 lot traded, taking the premium from 22.43 to 23.18.
- 11:56–11:57: 3 more lots traded, reaching **23.80**. That is a profit of Rs 4,025, so the rule armed.
- 11:58–12:03: zero volume.
- 12:05: the bar was O 23.5, L 22.95, C 23.0 on 14 lots. The low printed below the 23.32 giveback floor, so the rule fired.

The restart did NOT matter here. The pre-restart peak was 23.76 at 11:01, just under the 23.79 arm level.

## Pattern, not a one-off

Every Swing COPPER `PROFIT_PROTECTION_HIT` since 15 Sep realised only +0.26 to +1.46 premium points. The arm threshold is 1.6 points. Each exit gives back most of the gain that armed it, which is consistent with thin-liquidity prints plus market-order slippage.

## What this is NOT

This was **not** a close-based vs tick-based timing problem. `PROFIT_PROTECTION_HIT` is already evaluated on every tick (WS `on_price_tick`, plus the 5s poll) in all four packages. Making signal exits (Supertrend, EMA-cross, structure-break) tick-based would add more intrabar whipsaw exits, not fewer. The 24 Sep day already had 5 losing `STRUCTURE_BREAK_SQUARE_OFF` churns on close-based exits.

## Relation to prior findings

`learnings/exit-mechanics.md` found that a blanket give-back buffer is net-negative on NSE Options and Luxury, because option premiums mean-revert. That population was liquid NSE stock options. COPPER's issue is different: in a thin contract, the "peak" itself is untrustworthy. Candidate fixes need a COPPER-specific backtest on real option 1-min data before deployment:

- arm only on the bid, or on N consecutive prints;
- measure giveback as a % of peak profit instead of peak price;
- exit with a limit order instead of a market order.

## Outcome (same day)

Backtests are in `learnings/exit-mechanics.md` → "MCX option profit-protection: fix the execution, not the rule". The PP rule itself was the right one for COPPER; the market-order exit was the leak. Shipped in traderBoy `2479813`:

- **`SWING_MCX_PP_LIMIT_EXIT_ENABLED`:** a PP exit on an MCX option goes out as a sell limit at the trigger price, falling back to market after 180s or on a hard-stop / max-loss breach.

The same investigation found a real Swing bug. A bought PE is `instrument_side == "LONG"`, and `_evaluate_exit_signal` read the reversal direction from that alone. So every OPTIONS PE since 15 Sep exited on a *bearish* crossover and ignored the bullish one. Fixed in the same commit.
