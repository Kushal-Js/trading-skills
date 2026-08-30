# trading-skills

A knowledge base of trading learnings for **DhanBoy** — the live multi-strategy
options/futures bot in [traderBoy](https://github.com/Kushal-Js/traderBoy)
(private repo, `dhanBoy` branch). This repo is *not* the bot's code — it's
where we write down what we've actually learned about how the strategy, the
market, and the tooling behave, so future decisions (config tuning, screener
design, incident response) start from accumulated evidence instead of
re-deriving everything from scratch each time.

## What goes here

- **`learnings/`** — durable findings about *how things actually behave*:
  exit mechanics, screener logic, backtest methodology and its limits,
  capacity/ranking effects, config-tuning history. Each file should be
  something a future session (human or Claude) can read cold and act on
  correctly, without re-running the original investigation.
- **`incidents/`** — case studies of specific real trades/days worth
  remembering in detail (a surprising loss, a mechanism that wasn't obvious,
  a screener/bot mismatch). One file per incident, dated.
- **`SAFETY.md`** — the non-negotiable operating boundary. Read this first.

## What this repo is for

Sharper analysis, faster investigation, better-informed config decisions,
fewer repeated mistakes. The goal is a bot (and an assistant) that makes
*better-reasoned* decisions over time — not a bot that acts with less
oversight over time. See `SAFETY.md` for why those are different goals.

## Style

- Write for someone who wasn't there. State the claim, the evidence, and the
  scope it applies to (which strategy, which config values, which market
  conditions) — a finding that was true under one config isn't automatically
  true after the config changes.
- Prefer "here's what we verified and how" over "here's what we believe."
  Cite the concrete evidence (a log line, a backtest number, a screener
  clause) wherever possible.
- Update a file rather than leaving a stale one when a later finding
  supersedes it — note what changed and why, don't just delete the history.
