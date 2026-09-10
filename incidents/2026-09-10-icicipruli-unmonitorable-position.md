# 2026-09-10 — ICICIPRULI: a position the bot could not price all day

## What happened

Options entered **ICICIPRULI 29 SEP 470 CE ×925 @ ₹10.30 at 09:17**. From
~09:38 onward Dhan returned **no LTP** for that contract on every poll
("No LTP returned for ICICIPRULI 29 SEP 470 CALL", repeated all day; a few
`ERROR | trading_engine | Could not fetch LTP` / `ValueError` too).

Consequence: `_exit_reason_for` never ran with a real price, so
**MAX_LOSS_HIT / TARGET_HIT / trailing-SL / PROFIT_PROTECTION could not be
evaluated for this position for ~6 hours**. Intraday the contract traded
down to **₹5.65** (11:01) — an unrealised **−₹4,300** at that moment, well
past the ₹1,000 after-cutoff MAX_LOSS cap — with no stop possible. It
recovered to ~₹10.75 by close purely by luck, ending ≈ flat (+₹185 M2M).

Then, because Options runs `ENABLE_SQUARE_OFF=false` (NRML carry-forward),
it was **held overnight** — still unmonitorable, exposed to the next day's
gap.

## Why the existing guards didn't catch it

- **Liquidity guard** (`LIQUIDITY_GUARD_ENABLED`) keys off *zero-volume
  bars* in the option's own 1-min candles via `refresh_liquidity_signal` /
  `get_cached_illiquid`. When the LTP feed returns *nothing at all* (not
  "zero volume" — no data), the guard has nothing to evaluate and never
  fires a `LIQUIDITY_GUARD_ZERO_VOLUME` exit.
- **The 1-min *historical* data for the contract existed the whole time**
  (a later read pulled 384 bars, including the ₹5.65 dip) — it's only the
  *live* LTP path (`get_option_ltp` / the WS cache) that was dead. So the
  contract wasn't un-tradable, just un-*priceable* through the path the
  monitor loop uses.

## The gap

There is no "**I have not been able to price this open position for N
minutes → force an exit (or at least alarm loudly)**" safety. A position
the monitor can't see is a position with zero downside protection, and on
NRML that silently rolls into an overnight naked long.

## Fix direction (not yet built)

- A staleness watchdog per open position: if `get_option_ltp` (WS + REST
  fallback) has failed continuously for > ~3-5 min, place a market exit
  using the last *historical* 1-min close as the mark, or at minimum raise
  a loud alert. Reuse the historical `intraday_minute_data` feed as the
  fallback-of-the-fallback for the exit decision, since it kept working.
- Consider refusing the *entry* in the first place when the ATM contract
  can't be priced at entry time (ICICIPRULI's LTP was already failing by
  09:38, ~20 min after entry).

## Immediate action

Square ICICIPRULI 470 CE off manually at the next open (it was ≈ flat, a
clean exit) rather than carry an unmonitorable position.

Related: [[capacity-and-ranking]] (this was a real *taken* trade, not a
capacity miss), and the SL-L-vs-bot note in [[exit-mechanics]] — an SL-L
order would NOT have helped here either, it needs the same dead LTP feed
to trigger.
