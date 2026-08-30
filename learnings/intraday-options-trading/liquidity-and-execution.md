# Liquidity and execution — concrete thresholds behind the SAGILITY lesson

Source: cross-referenced across Options Hawk, SteadyOptions, Options
Animal, and other options-execution explainers. Researched 30 Aug 2026.
This file gives the general, sourced thresholds behind what
`learnings/exit-mechanics.md` and `learnings/paper-trading-shared-
infrastructure-risk.md` already found empirically from real trades
(SAGILITY) — read that file first for the concrete incident this connects
to.

## Concrete liquidity thresholds worth having in mind

- **Open interest**: look for at least a few hundred contracts as a bare
  minimum; a commonly-cited stronger bar is **>1,000 contracts OI and
  >100 contracts daily volume** for genuinely comfortable execution.
- **Strike proximity**: ATM strikes are consistently the most liquid on
  any given underlying — deep ITM/OTM strikes see materially lower volume
  and wider spreads. This is one more reason (alongside gamma/theta
  reasoning in `greeks-and-decay.md`) the ATM choice is right, not
  incidental.
- **Expiry proximity**: near-term (nearest monthly) contracts are
  typically more liquid than longer-dated ones — consistent with
  `get_atm_option`'s own preference for the nearest listed expiry (rolling
  forward only to avoid the same-day-expiry RMS-rejection problem, bug
  #28, not as a general liquidity optimization, but the two happen to
  point the same direction).

## Spread and slippage patterns — directly relevant to entry/exit timing

**Spreads are typically widest at market open and close** — mid-session
tends to have tighter spreads. This is a genuinely useful cross-check
against `timing-patterns.md`'s finding on volatility-by-time-of-day: the
first 15 minutes and the last 30 minutes are both higher-volatility *and*
wider-spread windows — a double reason entries in that window carry more
execution-cost risk than the same signal firing mid-session, independent
of the momentum signal's own quality.

**Market orders vs. limit orders**: a market order fills at the full
quoted spread and risks slippage in a fast market; a limit order near the
midpoint controls the entry price but risks not filling at all in a fast
move. DhanBoy's own order placement (`Options/dhan_client.py`) uses market
orders for both entry and exit — a deliberate tradeoff favoring fill
certainty over price optimization, appropriate for a strategy whose edge
depends on acting on a fresh signal quickly (a limit order that doesn't
fill defeats the purpose of a momentum-following entry).

## How this connects to what's already been found empirically

The SAGILITY incident (`learnings/exit-mechanics.md`) is exactly what
"wide spread / thin liquidity" looks like in practice for an option that
still technically has *some* volume: not a total absence of trades, but
few enough that the price can gap several ticks between prints, and a
market order's fill price can land meaningfully worse than the last quote
before it. K01's "anti-SAGILITY band" (`K01/config.py`:
`ANTI_SAGILITY_MAX_PREMIUM_RS`/`ANTI_SAGILITY_MIN_LOT_SIZE`) is a direct,
if narrow, attempt to pre-filter for this — rejecting the specific
combination of very-low premium and very-high lot size. **What this
research adds**: a more general, broker-side liquidity floor (OI/volume
thresholds on the option contract itself, not just a premium/lot-size
proxy) would catch a wider range of thin-liquidity risk than the current
proxy does, since a moderately-priced option on a small-lot contract could
still be thin if the specific strike itself has low OI — not something
`ANTI_SAGILITY_*`'s premium/lot-size check would catch. Worth considering
as a Stage 1 addition once the Option Chain API (already flagged as
needed for Stage 2's OI-buildup gating) is integrated — the OI data would
serve both purposes.
