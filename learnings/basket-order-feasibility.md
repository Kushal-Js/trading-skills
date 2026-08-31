# Dhan has no basket/multi-leg order API - margin is checked per-leg, in isolation

Applies to: any future combo strategy (e.g. a futures position + a
protective PE option on the same underlying). Investigated 31 Aug 2026 in
response to the user asking about basket-order feasibility for exactly
this futures+PE shape.

## Finding 1: no basket order endpoint exists

Enumerated every REST endpoint the `dhanhq` SDK (this bot's actual
dependency, via `Options/dhan_client.py`) implements:

```
/orders, /orders/{id}, /orders/external/{correlation_id}   - single order only
/super/orders, /super/orders/{id}                            - single INSTRUMENT, entry+target+SL legs bundled
/forever/orders                                               - single instrument, GTT-style standing order
/margincalculator, /positions, /positions/convert, /holdings, /fundlimit, ...
```

No `/basket`, `/multi-order`, or `/bulk` endpoint anywhere. `place_order()`
itself takes exactly one `security_id` per call - no list/array parameter
for multiple legs. Dhan's "Super Order" (the closest-sounding concept) is a
bracket order on ONE security (entry+target+SL), not a way to combine two
different instruments (e.g. a futures contract and an option) into one
order. Checked Tradehull too (the higher-level wrapper this bot actually
calls) - same result, nothing.

**Consequence**: a "futures + PE" combo can only ever be two independent
orders our own code sequences - never one atomic broker-side transaction.
One leg can fill while the other is rejected (RMS, margin, expiry, a
transient rate-limit fail - see `dhan-rate-limit-every-call-site.md`),
leaving a naked, unhedged position. Any all-or-nothing guarantee has to be
built at the application level (this bot already has the right building
blocks for that shape of problem - see `traderBoy/cross_strategy_registry.py`'s
claim/release pattern and `_process_one_entry`'s reserve->check->place->
release-on-failure pattern).

## Finding 2: margin_calculator is per-leg, with zero combo awareness

`margin_calculator(security_id, exchange_segment, transaction_type,
quantity, product_type, price, trigger_price)` - confirmed via its actual
signature - has no parameter for "given I already have/am about to place
this other position." It answers exactly one question: what would THIS
ONE order, by itself, cost in margin right now. There is no API to ask
"what would the combined margin be for a futures+PE combo on XYZ."

## Reasoning: what this means for placement order/timing

Since there's no basket-aware margin check, each leg's real-time RMS check
at the moment it's placed only knows about the account's *currently
settled* positions - not anything else in flight:

- **Futures leg placed and filled first, then PE leg placed**: by the time
  the PE leg is checked, the futures position is a real, settled fact.
  Exchange-mandated SPAN+Exposure margining (an NSE clearing-corp
  methodology, not Dhan-specific) is designed to recognize a genuine
  hedge - a long futures + protective put on the same underlying carries
  less net risk than either alone, and portfolio-level SPAN margin should
  reflect that. The PE leg's actual blocked margin would plausibly be
  lower than its standalone `margin_calculator` figure once the hedge is
  recognized.
- **Both legs placed concurrently** (the natural shape of this bot's
  existing `asyncio.gather`-based entry flow): the second order's RMS
  check likely still sees "no futures position yet," since fills are not
  instantaneous - confirmed live 31 Aug 2026 that a single order can sit
  `PENDING` at the broker for 8+ minutes before actually filling (see
  `traderBoy/NOTES.md` entry #60's lag audit). The second leg would then
  get margin-checked as if standalone, needing the FULL, non-hedged sum of
  both legs' margins available upfront - even if the eventual settled
  state needs meaningfully less once both are actually open.

**Practical rule for a future combo implementation**: budget for the worst
case (sum of both legs' standalone `margin_calculator` figures) as the
capital that must be free before attempting the combo, not the
theoretical hedged total - unless the code deliberately sequences
futures-first, confirms the fill, THEN places the PE leg, specifically to
try to capture the SPAN hedge discount on the second leg's margin check.

## What's confirmed vs. inferred here

**Confirmed from actual code/API surface**: no basket endpoint exists;
`margin_calculator` has no combo-awareness parameter; fills are not
instantaneous (a real 8m42s example exists).

**Inferred, not empirically verified for Dhan specifically**: that Dhan's
real-time portfolio margin engine actually grants the SPAN hedge discount
once both legs are genuinely open. This is standard exchange-level
mechanics that should apply, but confirming it for real would require
observing a real open combo position's margin utilization before/after
the second leg fills (via `/positions` + `/fundlimit`, not
`margin_calculator`) - deliberately not done here since it means placing
real trades, not a read-only query like the MTF eligibility check was.
