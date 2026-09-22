Status: BUILT, read-only analysis tool only - not wired into any live
strategy or backtest. Added 22 Sep 2026 (traderBoy, dhanBoy branch,
uncommitted at write time): `structure_break.py` +
`.claude/skills/structure-break/SKILL.md`.

# Multi-timeframe structure-break indicator

## What it is

A direct Python port of a Pine Script indicator the user pasted (BOSWaves'
"Smart Money Flow Cloud", MPL-2.0) - a money-flow-weighted adaptive ATR
band around a smoothed EMA/ALMA baseline. Its "structure break" is a
fully-closed candle's close crossing outside that band (the indicator's
own `switchUp`/`switchDown` Buy/Sell event) - **not** classic swing
higher-high/higher-low structure analysis. The name follows the user's
own framing when this was requested; confirmed with them up front (via
AskUserQuestion) that this band-cross was the intended "structure break"
definition, since the Pine script itself doesn't use that term.

Computes, per timeframe (5m/15m/1h/1d): regime (bullish/bearish), whether
a break happened on the last closed candle, bars since the last break, a
signed trend-strength percentage, and retest-dot events. Full math lives
in `compute_structure_break()`'s implementation - it's a straight
line-by-line port of the Pine indicator's `calcMF`/`calcBasis`/
`calcBands`/`calcSignals`/`f_trendStrengthSigned`.

## Design choices worth remembering

- **EMA/ATR convention**: reuses this repo's existing SMA-seeded EMA /
  Wilder-ATR style (`dhan_client.py`'s `_compute_ema`/
  `_compute_supertrend`), not Pine's own bar-0-seeded `ta.ema`. Same
  accepted tradeoff already documented for the real-money Supertrend/
  EMA-cross signals - values converge after the warm-up window, don't
  match TradingView bar-for-bar right at a freshly fetched window's start.
  Each timeframe fetches enough calendar-day history (15/30/90 days for
  5m/15m/1h, 400 for 1d) that the reported *last* bar sits well past that
  warm-up.
- **Data path**: same `Options.dhan_client.dhan_wrapper` calls every real
  signal in this repo uses (`fetch_continuous_intraday` for intraday,
  `historical_daily_data` for daily) - continuous multi-session series,
  per [[continuous-candles]], never fragmented at day boundaries. Equities
  only (`_equity_security_id` lookup) - no index/option support.
- **Per-timeframe isolation**: `analyze_symbol()` fetches each timeframe
  independently and catches exceptions per-timeframe, so one thin-history
  or rate-limited timeframe doesn't blank out the whole multi-TF report.

## Confirmed working (22 Sep 2026)

The pure computation (`compute_structure_break`) was smoke-tested against
synthetic OHLCV data - regime/band/strength/retest all compute without
exceptions, including edge cases (empty input, too-short input, ALMA
basis type). The live-fetch path (`fetch_timeframe`) was exercised against
`Options.dhan_client.dhan_wrapper` for real (RELIANCE, 5m) and correctly
hit the existing local-auth guard from
[[../incidents/2026-09-21-local-backtest-dhan-session-collision.md]]
(refused to `pin_totp` from a local process, telling the user to use a
handed-off access token instead) rather than crashing or risking kicking
the live droplet bot's session - confirms the error-handling path works,
not just the happy path. A full live multi-timeframe run against real
market data hasn't been done yet (needs `HANDOFF_DHAN_ACCESS_TOKEN` or
running it from a context where a session is already authenticated).

## What this is NOT

Not imported by Options/Futures/Luxury/Swing, places no orders, gates no
live decision. Per [[../SAFETY.md]], turning this into an actual
entry/exit signal is a separate, much larger task (config wiring, a real
backtest, explicit go-ahead) that hasn't been started.
