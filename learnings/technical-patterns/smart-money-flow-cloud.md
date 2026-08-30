# Smart Money Flow Cloud [BOSWaves] — mechanism breakdown

Source: Pine Script v6 indicator supplied directly by the user (30 Aug
2026), MPL-2.0 licensed, by TradingView author BOSWaves. This file
translates its logic into the same vocabulary as
`minervini-trend-template.md`/`vcp.md` so it can be reasoned about (and
potentially ported to Python) without re-reading Pine each time.

## What it actually is, in one sentence

An adaptive-band trend-following overlay — structurally the same *role* as
a Supertrend (persistent regime flips on a crossover, band width scales
with volatility) — but with two differences from the Supertrend K01/Options
already use: the band multiplier is **driven by a volume-weighted money-flow
strength reading** instead of being fixed, and it adds a distinct **"retest"
signal** for pullbacks within an established trend that Supertrend alone
doesn't surface.

## Component-by-component

**1. Trend baseline (`calcBasis`)** — an EMA or ALMA of length 34 (default),
computed *separately* on `open` and `close`, giving two lines
(`bs.bO`, `bs.bC`) whose gap forms the "cloud" fill. `bs.bC` (basis-of-close)
is the line everything else keys off (`bs.bMain`). This is just a smoothed
trend baseline — conceptually similar to what a moving-average-based trend
filter does, not something K01 currently has an equivalent of (K01's Stage 0
Trend Template uses 50/150/200-day SMAs for a *daily* structural gate, not
an intraday adaptive baseline).

**2. Money flow strength (`calcMF`)** — this is the indicator's distinctive
piece:
- Per-bar **Close Location Value**: `clv = ((close - low) - (high - close)) / (high - low)`
  — ranges -1 (closed at the low) to +1 (closed at the high). This is the
  same CLV formula behind the classic Chaikin Money Flow / Accumulation-
  Distribution family, not something novel to this script.
- `raw = clv * volume` — volume-weights each bar's directional close
  location.
- Over a rolling window (`mfLen`, default 24 bars): `mf = sum(raw) / sum(abs(raw))`
  — a volume-weighted, direction-signed ratio in [-1, +1]. This is
  functionally a Chaikin-Money-Flow-style oscillator, smoothed
  (`mfSmooth`, default 5-bar EMA) into `mfSm`.
- **Nonlinear boost**: `strength = clamp(|mfSm| ^ mfPower, 0, 1)` — raising
  to a power >1 (default 1.2) suppresses weak/noisy flow readings toward 0
  and lets strong readings approach 1 faster than linearly. This is the
  "boost" — a way to make the band-width response more binary (mostly tight
  or mostly wide, less time in an ambiguous middle state) than a raw linear
  mapping would give.

**3. Adaptive bands (`calcBands`)** — `mult = minMult + (maxMult - minMult) * strength`
  (defaults: minMult=0.9, maxMult=2.2), then `upper/lower = basis ± ATR(14) * mult`.
  **Key behavior**: bands WIDEN when money-flow strength is high (a genuine
  volume-backed move in progress) and TIGHTEN when flow is weak/noisy. This
  is the opposite of what it might sound like at first — a strong,
  volume-confirmed trend gets *more room* before the indicator will call a
  reversal (fewer whipsaws during a real move), while a weak/choppy period
  gets a *tighter* band that flips regime more readily (faster to bail out
  of a move with no real conviction behind it).

**4. Regime signal (`calcSignals`)** — `lastSignal` flips to bullish (+1) on
  `close` crossing over `upper`, flips to bearish (-1) on crossing under
  `lower`, and otherwise **persists** the prior regime (exactly like
  Supertrend's own "stays in the last state until proven wrong" logic — see
  `dhan_client._compute_supertrend`). The BUY/SELL labels
  (`switchUp`/`switchDown`) fire only on the regime *flip* itself, not every
  bar the regime holds.

**5. Retest signal (the piece K01 has no equivalent of)** — while regime is
  bullish (`lastSignal == 1`) AND the bar's low dips below the basis line
  (`bs.bC`) without the regime itself flipping, that's flagged as a
  "bullish retest" (✦ marker), with a configurable bar-count cooldown
  (`dotCooldown`, default 12) so it doesn't fire on every consecutive bar of
  a shallow chop against the baseline. Symmetric for bearish. **This models
  a pullback-to-trend-then-continuation setup** — a genuinely different
  entry opportunity than "wait for the next full regime flip," and worth
  noting as something K01's current momentum_signal (a one-shot crossover
  check) has no mechanism for at all.

**6. Trend Strength Gauge** — a `tanh`-compressed, EMA-smoothed measure of
  how far price sits inside its current band as a fraction of that band's
  own width, signed by regime direction. Purely a visual/UI feature (a
  table drawn on the chart) — not a separate tradable signal, and not
  something a headless/Python port would need to replicate at all.

## Direct comparison to K01's current Stage 3 momentum signal

`K01/paper_engine.py::momentum_signal()` currently requires **all four** of:
5-min RSI in a directional band, 5-min close vs. a *fixed*-multiplier
Supertrend(10, 3.0), a 1-min Supertrend crossover, and 5-min ROC sign
agreement. Every one of these is a fixed-parameter check; none scales with
how strong the underlying volume-driven move actually is. Smart Money Flow
Cloud's core idea — **band width (hence how "hard" it is to trigger a
regime flip) should scale with money-flow conviction, not stay fixed** — is
a structurally different idea from anything currently in K01's pipeline,
and is the piece worth evaluating for a Stage 3 revision (see
`../../designs/k01-smart-money-flow-revision.md`).

## Porting notes, if this is ever implemented in Python

Nothing here needs Pine-specific behavior — it's arithmetic over OHLCV
arrays, matching the shape of what `K01/paper_engine.py`'s existing
`_compute_rsi`/`_compute_atr`/`_compute_roc`/`_compute_supertrend`-style
pure functions already do. Needed building blocks not yet in that file:
a CLV/money-flow-ratio function, a power-based strength transform, and an
adaptive-multiplier ATR band (a straightforward generalization of the
existing fixed-multiplier `_compute_supertrend`, not a rewrite). The
Trend Strength Gauge (tanh/table) is chart-only and should NOT be ported —
there is no equivalent UI surface in a headless bot, and it isn't a trading
signal.
