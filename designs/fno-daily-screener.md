# Design: daily F&O stock screener skill

**Status: proposal, not built.** Written 30 Aug 2026 after web research +
applying lessons already in `learnings/`. This is the plan to refine with
the user before writing any code. Sections marked "OPEN QUESTION" are
genuinely undecided and need a decision, not an assumption.

## Goal

Given the full NSE F&O stock universe (~208 stocks as of 2026, per NSE's
periodic F&O-segment additions), produce a short, ranked list of the
day's best tradeable stocks — with a directional bias and a one-line
rationale each — as a repeatable, on-demand Claude Code skill. Read-only
analysis only; does not place orders or feed the live bot automatically
(see `SAFETY.md` — that would be a separate, later, explicitly-decided
step, not a side effect of building this).

## Data sources

| Need | Source | Status |
|---|---|---|
| F&O stock universe | `dhan_wrapper.instruments()` (Tradehull's cached scrip master, already authenticated) | **Already available** — no new integration |
| Daily/intraday OHLC + volume | `intraday_minute_data` / `historical_daily_data`, same calls `bt_common.py` already uses | **Already available** |
| Options OI, IV, Greeks per strike | [Dhan Option Chain API](https://dhanhq.co/docs/v2/option-chain/) — `POST https://api.dhan.co/v2/optionchain` | **New** — not used anywhere in the bot today |
| Futures OI + volume + depth | [Dhan Market Quote API](https://dhanhq.co/docs/v2/market-quote/) — `https://api.dhan.co/v2/marketfeed/quote` | **New** |
| Historical OI (for buildup classification, not just today's snapshot) | Dhan's historical-data endpoint now supports an optional `oi` parameter (per Dhan's own release notes) | **New**, needs confirming exact request shape once we start building |

## Proposed pipeline (four stages, in order)

### Stage 1 — Liquidity/volatility floor (reject before ranking)

Filter out anything too thin to trade safely *before* it can rank well on
other criteria. This directly encodes the SAGILITY lesson in
`learnings/exit-mechanics.md`: a thin, low-premium, high-lot-size contract
can gap through any rupee-based risk cap, and no amount of good momentum
scoring makes that risk go away.

Proposed floor (OPEN QUESTION — exact thresholds need agreement):
- ATR% (ATR ÷ price) above some minimum — e.g. reject anything under ~0.5–1%
  average daily range, so the stock actually moves enough intraday to be
  worth the risk.
- Minimum average daily volume / turnover, to avoid wide-bid-ask names.
- Option premium sanity band — avoid the SAGILITY pattern (very low premium
  × very high lot size, where a 1-paisa tick is a large rupee swing).

### Stage 2 — OI buildup classification (directional bias signal)

Standard four-way read, price vs. OI change over the session:

| Price | OI | Classification | Read |
|---|---|---|---|
| ↑ | ↑ | Long buildup | Bullish — fresh longs |
| ↓ | ↑ | Short buildup | Bearish — fresh shorts |
| ↑ | ↓ | Short covering | Bullish, but weaker conviction (exits, not new positions) |
| ↓ | ↓ | Long unwinding | Bearish, weaker conviction |

This is a genuinely different signal from price action alone — it
distinguishes "the market is expressing a fresh view" from "positions are
just being closed out." Worth noting from research: intraday OI swings can
be noisy (scalpers/algos), so this reads better as a *session-level*
(vs. minute-level) signal — don't over-trust a 5-minute OI blip.

### Stage 3 — Momentum/trend confirmation

Reuse the exact machinery already built and validated this session:
5-min RSI(14), 5-min/1-min Supertrend regime + crossover, 5-min ROC(9).
This is deliberately the same shape as Krishvi's own Chartink formula
(`learnings/screener-analysis/krishvi.md`) — a self-built screener can
either mirror that logic or knowingly diverge from it (e.g. using
Supertrend(10,3) to match the bot's own exit-side parameters instead of
Krishvi's period-7, closing the mismatch documented in that file).

### Stage 4 — Score, rank, shortlist

OPEN QUESTION: exact scoring formula/weights across stages 2–3. Options to
decide between:
- A simple weighted sum (normalize each signal 0–1, weight and add).
- A gating approach (Stage 2 must agree directionally with Stage 3 — only
  rank stocks where OI buildup and momentum point the same way; disagreement
  = excluded, not just down-weighted).
- Separate bullish and bearish shortlists (top N long-buildup+bullish-
  momentum stocks, top N short-buildup+bearish-momentum stocks) rather than
  one combined ranked list.

Output: a short list (OPEN QUESTION: how many — 10? 15?) with, per stock:
symbol, direction, ATR%, OI-buildup classification, momentum-signal
summary, one-line rationale.

## Real engineering constraint: Dhan's rate limit

Confirmed and documented (`traderBoy/NOTES.md` bug #5): Dhan's market-data
REST calls have an undocumented rate limit, hit empirically in production.
Screening ~208 stocks daily means ~208+ REST calls at minimum (more once
option-chain calls are added per stock). This needs deliberate batching/
pacing from the start, not a naive per-stock loop — likely a small delay
between calls plus retry-with-backoff (the existing `_retry()` pattern in
`Options/dhan_client.py` is the right model to reuse). Worth timing a full
run once built to know how long a "morning screen" actually takes before
relying on it pre-market.

## Explicitly deferred, not part of this build

- **Feeding this screener's output into the live bot as an alert source**
  (replacing or supplementing Chartink webhooks) — a much bigger decision
  with real trading-strategy implications, not something to fall into as a
  side effect of building an analysis tool. If ever wanted, that's its own
  separate, explicit conversation.
- **Any order-placement action based on the shortlist** — this tool's job
  ends at "here's a ranked list for you to look at," per `SAFETY.md`.

## Next step

Resolve the OPEN QUESTIONs above (floor thresholds, scoring approach,
shortlist size) with the user, then build: a Python script (same shape as
`bt_common.py`) plus a `SKILL.md` wrapping it for on-demand invocation.
