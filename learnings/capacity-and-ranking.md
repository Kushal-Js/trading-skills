# Capacity, ranking, and what actually gates a trade from happening

Applies to: DhanBoy's Options package. `MAX_LIVE_POSITIONS_CE`/`_PE` and
`TOP_N_STOCKS`. Live cap history: CE=4/PE=0/TOP_N=4 (27–28 Aug 2026, PE
fully off) → CE=3/PE=0 (30 Aug) → CE=2/PE=2 (31 Aug, PE re-enabled, CE
trimmed to keep combined exposure ~flat) → CE=1/PE=1 (1-per-side, some
point after) → **CE=2/PE=2 (10 Sep 2026, user request "update Option max
concurrent trade quota to 2" — `.env`-only change on the droplet, code
default was already 2; up to 4 concurrent, 2 per side)**. `TOP_N_STOCKS=4`.

## Three separate things can block an alert from becoming a trade — don't conflate them

1. **Capacity cap** (`MAX_LIVE_POSITIONS_CE`/`_PE`) — a hard reject, checked
   both early (webhook-level, before ranking even runs — `remaining_
   capacity()`) and again at entry time (`PositionStore.reserve_symbol`).
2. **Dedup** — a symbol already open/in-flight is skipped for *that symbol
   specifically*; it doesn't block other symbols in the same alert.
3. **Ranking** (`TOP_N_STOCKS` + `SELECT_BOTTOM_N_STOCKS`) — only matters
   when an alert lists *more* candidates than `TOP_N_STOCKS`. With fewer
   candidates than the slice size, top-N and bottom-N select the identical
   set — ranking direction is a non-factor for small alerts.

**How to tell which one blocked a specific missed trade:** check the actual
webhook log line. "No CE capacity left (N live/in-flight already) - ignoring
alert" = capacity, rejected before ranking ever ran. "skipped - already
open/in-flight, or no capacity" for one specific symbol while a *different*
symbol from the same alert enters successfully = dedup on that symbol, not a
capacity problem. Confirmed this distinction concretely investigating
DELHIVERY (27 Aug 2026): of 9 alerts mentioning it, exactly 1 was a real
capacity block (whole-alert rejection), 5 were dedup skips (proven by a
different stock from the same alert entering right after), and ranking was
never the cause (every alert listing it had ≤3 stocks, under the then-live
`TOP_N_STOCKS=3`).

## Raising the capacity cap does increase both winners and losers, roughly proportionally

Backtest comparison, same CSV (`02 Kaashvi.csv`), same 4 trading days:

| Config | Trades | Win rate | Realized P&L | MAX_LOSS_HIT count |
|---|---|---|---|---|
| CE=2, TOP_N=3 (pre-27 Aug) | 142 | 61.3% | +88,766.40 | 30 (21.1% of trades) |
| CE=4, TOP_N=4 (27 Aug on) | 209 | 56.5% | +104,135.20 | 49 (23.4% of trades) |

More capacity let through 47% more trades. MAX_LOSS_HIT count grew
proportionally (+63%), not because any single trade got more likely to lose
(21.1% → 23.4% is a small shift) — it's a volume effect, not a quality
effect. Net P&L still improved because PROFIT_PROTECTION_HIT gains scaled up
even more (+54K on the extra volume vs. +39K more in MAX_LOSS_HIT drag). This
generalizes: **raising position-count caps trades win-rate percentage
against absolute profit** — expect a slightly lower win rate at higher
capacity, and judge the change on total P&L and total drawdown risk, not win
rate alone.

## Real margin/funds can be the actual binding constraint even when the capacity cap says there's room

Found investigating 28 Aug 2026's real trade log: from mid-morning onward,
**65 BUY orders were rejected by Dhan's RMS that day** — 43 for
"insufficient funds" (shortfall grew from ~Rs.200 to over Rs.35,000 across
the session), 22 for a stock genuinely banned from F&O that day (SAIL,
unrelated to funds). The `MAX_LIVE_POSITIONS_CE=4` cap was *allowing* up to
4 concurrent positions, but the account frequently couldn't actually fund a
3rd or 4th position — several specific alerts the bot wanted to take
(COFORGE, TECHM, GLENMARK, DELHIVERY re-tries) were blocked by real margin,
not by config.

**Lesson: a capacity-cap config change should be evaluated against the
account's actual available margin, not just backtest P&L** — a cap that
looks good in a backtest (which assumes unlimited funds) can be silently
throttled by real broker rejections in live trading, and those rejections
don't show up in a backtest at all. Check current available margin
(`portfolio_agent_tool` action `funds`, or the droplet's own Dhan session)
before raising a capacity cap meaningfully.

## Backtest vs. live webhook delivery are independent — don't assume a CSV's timestamps match reality

A Chartink CSV export's alert timestamps for a symbol do **not** necessarily
correspond to when (or whether) the live webhook actually delivered a
matching alert that day. Confirmed by cross-referencing DELHIVERY's and
ZYDUSLIFE's real webhook-log timestamps against the CSV's timestamps for the
same symbol/day — they never matched. A backtest against a CSV tells you
what the bot's strategy *logic* would have done given those exact alerts; it
does not tell you whether those exact alerts were ever actually delivered
live. Treat backtest P&L as "what this logic does against this alert
pattern," not as "what actually happened live."
