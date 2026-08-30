# Design: daily F&O stock screener skill

**Status: MVP built and deployed, 30 Aug 2026 — first live paper-trading
test starts 31 Aug 2026.** Implementation lives in `traderBoy`'s
`FnoScreener/` package (paper-trading only, mirrors `IndexScalping`/
`CopperOptions`'s existing pattern), not in this repo — see that repo's
`README.md`/`NOTES.md` entry #39 for the code-level details. This file
stays the design record; update it if a later test shows a threshold/
weight below needs to change, per the repo's own style guide, rather than
silently changing the number.

**MVP scope vs. this design:** Stages 0 (Trend Template only — the VCP
scored-bonus half is deferred) and 1 (liquidity floor) and 3 (intraday
momentum) are built. **Stage 2 (OI-buildup gating) is deferred** — it
needs a brand-new Dhan Option Chain integration that's never been used in
this codebase before, and wasn't made load-bearing on the very first live
test until it can be built and verified on its own. Until Stage 2 ships,
entries are momentum-only (Stage 3's four conditions must self-agree,
rather than also requiring OI-buildup agreement as designed below).

## Goal

Given the full NSE F&O stock universe (~208 stocks as of 2026), produce a
short, ranked, direction-split list of the day's best tradeable stocks —
each with a rationale, not just a score — as a repeatable, on-demand Claude
Code skill. Read-only analysis only; does not place orders or feed the live
bot automatically (see `SAFETY.md` — that would be a separate, later,
explicitly-decided step, not a side effect of building this).

## Data sources

| Need | Source | Status |
|---|---|---|
| F&O stock universe | `dhan_wrapper.instruments()` (Tradehull's cached scrip master, already authenticated) | **Already available** — no new integration |
| Daily/intraday OHLC + volume | `intraday_minute_data` / `historical_daily_data`, same calls `bt_common.py` already uses | **Already available** |
| Options OI, IV, Greeks per strike | [Dhan Option Chain API](https://dhanhq.co/docs/v2/option-chain/) — `POST https://api.dhan.co/v2/optionchain` | **New** — not used anywhere in the bot today |
| Futures OI + volume + depth | [Dhan Market Quote API](https://dhanhq.co/docs/v2/market-quote/) — `https://api.dhan.co/v2/marketfeed/quote` | **New** |
| Historical OI (yesterday's EOD OI, for the buildup comparison) | Dhan's historical-data endpoint's optional `oi` parameter (per Dhan's release notes) | **New**, confirm exact request shape while building |
| ~252 trading days of daily OHLC per stock (needed for Stage 0's 200-day MA + 52-week high/low) | Same `historical_daily_data` call, wider date range than `bt_common.py`'s typical short backtest windows | **Already available as an API call** — but note this is meaningfully more data per stock than anything fetched so far this session; factor into the rate-limit pacing plan below |

## Pipeline (now five stages — Stage 0 added 30 Aug 2026)

### Stage 0 — Daily-chart trend/base quality pre-filter (NEW)

Added after building out `learnings/technical-patterns/` (Minervini Trend
Template, VCP, classic continuation patterns). Runs **before** Stage 1,
on **daily** candles (not intraday) — a slower, structural filter answering
"is this underlying even worth watching today," ahead of the faster
liquidity/momentum stages below.

1. **Trend Template pass** (`learnings/technical-patterns/
   minervini-trend-template.md`) — all 8 criteria: price above 50/150/200-
   day MAs, 150-day above 200-day, 200-day rising ≥1 month, close ≥30%
   above 52-week low, close within 25% of 52-week high. Reject outright if
   any fail — same all-or-nothing logic as Stage 1's liquidity floor, for
   the same reason (institutional-grade trend structure isn't a
   partial-credit property).
2. **VCP presence, as a bonus signal not a hard gate** (`learnings/
   technical-patterns/vcp.md`) — among Trend-Template passers, flag ones
   currently showing a tightening contraction sequence (each contraction
   smaller and lower-volume than the last, final contraction depth
   preferably <10%) with an identifiable pivot price. This is a genuinely
   non-trivial detection algorithm (finding swing highs/lows, measuring
   sequential contraction depth/volume) — **not a hard gate at launch**,
   because it's real, careful code to get right and shouldn't block the
   whole screener's first working version. Ship Stage 0 with just the
   Trend Template gate first; add VCP detection as a scored bonus
   (+weight in Stage 4, or a "🎯 VCP setup" flag in the output) once it's
   been built and tested on its own, separately.

This explicitly does **not** replace Stage 3's intraday momentum stack
(5-min/1-min Supertrend, ROC) — Stage 0 decides *which underlyings* are
structurally worth watching; Stage 3 still decides *when intraday* to
actually flag one as a candidate. Conflating the two timeframes was the
exact mistake flagged in both technical-patterns files: VCP/Trend-Template
are weeks-long daily-chart signals, not intraday triggers.

### Stage 1 — Liquidity/volatility floor (reject before ranking)

A stock must clear **all three** of the following to be considered at all.
This runs first and rejects outright — no partial credit, no score
adjustment for failing it. Rationale: this directly encodes the SAGILITY
lesson (`learnings/exit-mechanics.md`) — a thin, low-premium, high-lot-size
contract can gap through any rupee-based risk cap, and good momentum/OI
scores don't offset that risk.

1. **ATR% floor: daily ATR(14) ÷ closing price ≥ 1.0%.** Below this, the
   stock isn't moving enough intraday to be worth the risk for a
   target/SL-based options trade. Set slightly under Krishvi's own
   `Daily % Change > 1` gate (an *already-realized* move) since ATR is an
   *average* — a stock can clear a 1% ATR floor and still occasionally
   deliver the >1% single-day move a scan like Krishvi's is looking for.
2. **Liquidity floor: 20-session average daily turnover (price × volume)
   ≥ Rs.50 crore.** Standard practical threshold for avoiding wide-bid-ask
   names; low cash-market turnover correlates with thin options too.
3. **Anti-SAGILITY band: reject if (ATM option premium < Rs.5) AND
   (lot_size ≥ 5,000).** This is the exact combination that produced the
   SAGILITY overshoot — very cheap premium means a single tick is a large
   rupee swing once multiplied by a big lot size. Either condition alone is
   fine (a Rs.3 premium with a 500-share lot is manageable; a Rs.50 premium
   with a 12,000-share lot is manageable); it's the combination that's
   dangerous.

### Stage 2 — OI buildup classification (directional bias signal)

Computed as a **session-level** read (today's current OI vs. yesterday's
EOD OI, today's current price vs. yesterday's close) — not a minute-level
read. Research consistently flags intraday OI swings as noisy
(scalpers/algos); a session-level comparison is the reliable version of
this signal.

| Price vs. yesterday's close | OI vs. yesterday's EOD OI | Classification | Read |
|---|---|---|---|
| ↑ | ↑ | Long buildup | Bullish — fresh longs |
| ↓ | ↑ | Short buildup | Bearish — fresh shorts |
| ↑ | ↓ | Short covering | Bullish, weaker conviction (exits, not new positions) |
| ↓ | ↓ | Long unwinding | Bearish, weaker conviction |

### Stage 3 — Momentum/trend confirmation

- **5-min RSI(14)**, used as a *sanity band* not a trend signal: bullish
  candidates must sit in **40–75** (excludes both "not really moving" and
  "already extended/exhaustion-risk" zones); bearish candidates in **25–60**
  (mirror image). This is deliberately different from Krishvi's own
  `RSI < 80` (a much looser ceiling) — for a screener meant to *lead* a
  trading decision rather than confirm one already in progress, tighter
  bands reduce the chance of catching a move that's already mostly played
  out.
- **5-min AND 1-min Supertrend(10, 3)** — deliberately the bot's own exit-
  side parameters (`Options/config.py`'s `SUPERTREND_PERIOD=10`,
  `SUPERTREND_MULTIPLIER=3.0`), **not** Krishvi's period-7. This closes the
  mismatch flagged in `learnings/screener-analysis/krishvi.md`: a stock
  this screener flags as "in a bullish 5-min trend" is bullish by the same
  measure the bot will later use to decide whether to exit a resulting
  position, not a faster/different one.
- **5-min ROC(9, Close)** — sign and magnitude both used (see scoring,
  below): sign must agree with the candidate's direction (positive for
  bullish, negative for bearish), magnitude feeds the composite score.

### Stage 4 — Gate, score, and split into two shortlists

**Gate:** a stock only proceeds if Stage 2's OI-buildup direction and
Stage 3's momentum direction **agree** (both bullish, or both bearish).
Chosen over a pure weighted sum deliberately: a weighted sum can let a
strong-momentum stock with contradicting or absent OI conviction (e.g.
price rallying on short covering while OI actually falls) sneak onto a
"bullish" list on momentum alone — exactly the "weaker conviction" case
Stage 2's own table flags. Requiring agreement produces a cleaner,
higher-conviction shortlist, at the cost of a shorter one on quiet days
(acceptable — a short honest list beats a padded uncertain one).

**Score** (0–1 per component, normalized against that day's full
qualifying universe, then weighted) for everything that passes the gate:

| Component | Weight | What it measures |
|---|---|---|
| ROC(9) magnitude | 30% | Momentum strength |
| OI % change magnitude (today vs. yesterday's EOD) | 30% | Conviction strength behind the buildup |
| Distance from 5-min Supertrend line (as % of price) | 20% | How decisively in-regime, not just barely on the right side |
| Today's volume vs. 20-session average | 20% | Relative-volume confirmation — is today unusual for this name |

Momentum and OI-conviction are weighted equally and highest (they're the
two signals Stage 4's gate already required to agree) — regime strength and
relative volume are secondary tiebreakers among already-qualified names.

**Output: two separate shortlists, top 10 each** — bullish (long-buildup or
short-covering + bullish momentum) and bearish (short-buildup or
long-unwinding + bearish momentum) — not one combined ranked list. A
bullish score and a bearish score aren't measuring the same trade, so
ranking them against each other doesn't mean anything; keeping them split
also mirrors how the bot's own capacity is split (`MAX_LIVE_POSITIONS_CE`/
`_PE`). Top 10 each: enough real optionality for a full trading day (more
than the bot's own 4-slot concurrent cap, since candidates can drop out by
the time you act) without being an overwhelming list to review each
morning. **Note the live bot currently has `MAX_LIVE_POSITIONS_PE=0` (PE
off)** — the bearish shortlist is still worth producing for awareness and
for whenever PE trading is re-enabled, but it's informational-only against
today's live config.

Per stock in each shortlist: symbol, composite score, ATR%, OI-buildup
classification + %OI change, RSI/Supertrend/ROC summary, one-line
rationale in plain language (not just the raw numbers).

**Timing:** run once per day, **~10:15–10:30 AM IST** (45–75 minutes after
open), not at market open itself. Rationale: OI buildup is a session-level
signal and opening-minutes OI is contaminated by overnight roll positions
settling and pre-market/opening-auction noise; giving the session 45+
minutes lets the OI and price reads actually mean something. Re-runnable
on demand for a refreshed read later in the day.

## Real engineering constraint: Dhan's rate limit

Confirmed and documented (`traderBoy/NOTES.md` bug #5): Dhan's market-data
REST calls have an undocumented rate limit, hit empirically in production.
Screening ~208 stocks means at minimum ~208 historical-data calls plus one
option-chain call per surviving candidate (after Stage 1's floor — likely a
much smaller number than 208 by that point, which helps). Needs deliberate
batching/pacing from the start — reuse the existing `_retry()` backoff
pattern in `Options/dhan_client.py` as the model, and add a small fixed
delay between calls rather than firing them as fast as possible. Time a
full run once built, before relying on it as a ~10:15 AM daily habit.

## Explicitly deferred, not part of this build

- **Feeding this screener's output into the live bot as an alert source**
  (replacing or supplementing Chartink webhooks) — a much bigger decision
  with real trading-strategy implications, not something to fall into as a
  side effect of building an analysis tool. If ever wanted, that's its own
  separate, explicit conversation.
- **Any order-placement action based on the shortlist** — this tool's job
  ends at "here's a ranked list for you to look at," per `SAFETY.md`.

## Next step

Build: a Python script (`skills/fno-daily-screener/screener.py`, same
shape/conventions as `traderBoy/bt_common.py`) implementing the four stages
above, plus a `SKILL.md` in the same folder wrapping it for on-demand
invocation. Update this file's `Status:` line once built.
