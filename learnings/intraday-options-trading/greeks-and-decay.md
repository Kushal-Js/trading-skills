# Option Greeks and time decay — what actually matters for an intraday ATM buyer

Source: cross-referenced across QuantInsti, Interactive Brokers, Schwab,
Option Alpha, and several Greeks-explainer sites. Researched 30 Aug 2026.
Written specifically for DhanBoy's actual position: buying ATM CE/PE
options intraday (minutes to at most a day or two holding), never
selling/writing options.

## Why ATM specifically decays fastest — and why we trade it anyway

ATM options carry the most extrinsic (time) value of any strike, because
they have the most genuine uncertainty about finishing ITM or OTM.
**Theta is highest at-the-money and decreases as spot moves away from the
strike** — an ATM option bleeds time value faster, per day, than an
equivalent ITM or OTM option on the same underlying. This sounds like an
argument against buying ATM — but it's paired with the reason ATM is
still the right choice for this strategy: **ATM is also where gamma peaks**
(delta near ±0.50, the point of maximum sensitivity to the underlying's
next move) and **where liquidity is best** (tightest bid-ask spreads,
highest open interest — see `liquidity-and-execution.md`). For a strategy
that's buying on a *fresh momentum signal* and expects to exit within
minutes to hours (never holding to expiry), the fast responsiveness
(gamma) and tight execution (liquidity) matter more than the theta bleed
over a holding period this short — theta decay over 20-60 minutes is a
real but small cost next to a genuine directional move; it dominates only
for someone holding for days near expiry, which this strategy explicitly
never does (see `SQUARE_OFF_TIME`/Friday-carve-out mechanics already
documented in `traderBoy/NOTES.md`).

## Theta and gamma both spike together near expiry — a genuine risk this bot already manages

**Near expiration, gamma and theta both spike for ATM options** — the same
proximity-to-expiry that makes an ATM option's price move fastest per
point of underlying movement (good for a momentum-following buyer) also
makes it bleed value fastest per elapsed minute (bad for anyone whose
signal takes a while to actually play out). This is exactly why
`Options/dhan_client.py`'s `get_atm_option` rolls forward to the next
listed expiry when the nearest one expires today (documented bug #28,
`traderBoy/NOTES.md`) — trading a same-day-expiry contract would mean
buying into the single most theta/gamma-extreme moment of the option's
entire life, for a strategy that has no edge specifically timed to that
extremity (it's a momentum-following strategy, not a 0DTE gamma-scalping
one).

## Vega — a smaller factor for this strategy's holding period, but not zero

Vega (sensitivity to implied volatility) matters most when a position is
held across an event that changes IV materially (earnings, a scheduled
announcement, a volatility regime shift) — over a typical 20-60 minute
intraday hold with no known event in the window, vega's contribution to
P&L is usually smaller than delta/gamma's. Worth being aware of as a
factor, not something this strategy currently screens for or needs to.

## Concrete implication for tuning this bot's own parameters

The lesson isn't "avoid ATM" or "avoid options near expiry" — it's:
**time-in-trade is the lever that controls how much theta/gamma-extremity
risk a position actually carries**, more than the strike choice itself
(which is already ATM by design, for good reason). This reinforces why
fast, reliable exit mechanics (Supertrend reversal, MAX_LOSS_HIT,
PROFIT_PROTECTION_HIT) matter more for this strategy's risk profile than,
say, switching to slightly-ITM strikes to reduce theta exposure would —
the actual exposure-control knob is holding time, and that's already
squarely what `learnings/exit-mechanics.md` is about.
