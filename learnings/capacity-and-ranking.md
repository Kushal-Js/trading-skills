# Capacity, ranking, and what actually gates a trade from happening

Applies to: DhanBoy's Options package. `MAX_LIVE_POSITIONS_CE`/`_PE` and
`TOP_N_STOCKS`. Live cap history: CE=4/PE=0/TOP_N=4 (27–28 Aug 2026, PE
fully off) → CE=3/PE=0 (30 Aug) → CE=2/PE=2 (31 Aug, PE re-enabled, CE
trimmed to keep combined exposure ~flat) → CE=1/PE=1 (1-per-side, some
point after) → CE=2/PE=2 (10 Sep AM) → CE=3/PE=3 (10 Sep midday) →
**CE=2/PE=2 (10 Sep 2026, reverted; `.env`-only)**. `TOP_N_STOCKS=4`.
Luxury's own `LUXURY_MAX_LIVE_POSITIONS_CE/_PE` tracked the same path and
landed at 2/2 on 10 Sep too. There is still NO combined CE+PE total cap in
either package — `_cap_for()` / `reserve_symbol()` gate each type
independently, so "caps at 2" means up to 4 concurrent (2 CE + 2 PE); a
true total ceiling would need a new `MAX_LIVE_POSITIONS_TOTAL` primitive.

Entry timing (gates NEW entries only; `RISK_THRESHOLD_CUTOFF_TIME` 11:30 is
separate — it only switches the max-loss/profit-protect thresholds, never
blocks entries):

- **Multi-window schedule** (`ENABLE_TRADING_WINDOWS` / `TRADING_WINDOWS`,
  `is_within_trading_windows()`) — added + deployed 10 Sep 2026 for
  **Options, Luxury AND Futures**, all at `09:15-11:00,14:00-15:28` (user
  request: "zone1 9:15 upto 11 AM, zone2 2 PM upto 3:28 PM ... trading only
  allowed within these 2 zones"). Windows are [start, end): start
  inclusive, end exclusive. So no new entries in the **11:00-14:00 gap** or
  after 15:28. `SQUARE_OFF_TIME` (15:15) still force-closes regardless, so
  15:15 is the real upper bound for anything opened in zone 2.
- When `ENABLE_TRADING_WINDOWS` is on it **supersedes** the older single
  cutoff (`ENABLE_TRADING_TIME_LIMIT` / `ALLOWED_TRADING_TIME`) —
  `is_past_allowed_trading_time()` short-circuits to False. Options' single
  cutoff was briefly turned ON at 11:00 earlier the same day; the windows
  feature replaced it. The single-cutoff env keys are left in place, inert,
  as the fallback if windows are ever turned off.

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
