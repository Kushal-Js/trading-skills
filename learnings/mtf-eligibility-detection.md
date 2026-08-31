# Dhan's margin_calculator doesn't error on MTF-ineligible stocks - it silently returns 1X leverage

Applies to: any future MTF (Margin Trading Facility) equity-buying feature
for DhanBoy. Confirmed live, 31 Aug 2026, via a read-only (no order placed)
test call against `dhanhq`'s `/margincalculator` endpoint, reached through
Tradehull's already-authenticated session
(`dhan_wrapper.client.Dhan.margin_calculator(...)`).

## Context: why this needed testing at all

The user asked whether DhanBoy could also buy stocks via MTF, on top of its
existing options-only strategies. Before any design work, the open question
was: **how would the bot determine, per stock, whether MTF is actually
available for it?** Dhan's public scrip master CSV (`api-scrip-master.csv`,
the same file `traderBoy/choppy_stocks.py` already downloads for lot sizes)
has zero MTF-related columns - no static list to consult. The only concrete
lead was `dhanhq`'s `margin_calculator()` method (wraps Dhan's real
`/margincalculator` REST endpoint), which accepts a `product_type` including
`"MTF"` - untested until this session.

## The actual finding

Called `margin_calculator(security_id=..., exchange_segment="NSE_EQ",
transaction_type="BUY", quantity=1, product_type="MTF", price=<real LTP>)`
for 3 real stocks:

| Stock | NSE series | MTF response | Leverage |
|---|---|---|---|
| RELIANCE | EQ (normal) | `status: success`, `totalMargin: 280.94` (vs `1277.0` for CNC) | **4.55X** |
| EMAMIPAP | BE (trade-to-trade) | `status: success`, `totalMargin: 50.0` = IDENTICAL to its own CNC totalMargin | **1.00X** |
| KAYA | BE (trade-to-trade) | `status: success`, `totalMargin: 337.3` = IDENTICAL to its own CNC totalMargin | **1.00X** |

**`status` is `"success"` in every single case, including the two
MTF-ineligible BE-series stocks.** The API never returns an error, a
rejection, or any explicit "not eligible" signal for an MTF request on a
disqualified security. The only tell is that the computed `leverage` value
collapses to `"1.00X"` and the MTF response's `totalMargin` becomes
identical to the same stock's own CNC (full-cash, no-leverage) response -
i.e. Dhan silently priced the "MTF" request as if it were a plain cash
purchase, granting zero actual leverage, while still reporting overall
success.

A naive eligibility check like `if response["status"] == "success"` would
incorrectly treat **every** stock as MTF-eligible, BE-series included -
this is the trap. The correct check has to compare the *leverage value*
(or equivalently, compare MTF's `totalMargin` against a same-quantity CNC
call's `totalMargin` for the identical security) - eligible if
`leverage > "1X"` / MTF's margin requirement is genuinely lower than CNC's,
ineligible if they match.

## Side finding: `_equity_security_id`-style lookups can break on SME (SM-series) stocks

A fourth test stock, GOLDSTAR (NSE "SM"/SME-listed series), resolved to
`security_id="1"` via a naive `SEM_TRADING_SYMBOL` + `SEM_EXM_EXCH_ID=="NSE"`
+ `SEM_INSTRUMENT_NAME=="EQUITY"` filter (the same filter shape
`Options/dhan_client.py`'s `_equity_security_id()` uses) - clearly a bogus
value, not a real security ID. Both the CNC and MTF `margin_calculator`
calls for it failed identically with `DH-905 Input_Exception: Missing
required fields, bad values for parameters etc.` This is a data-quality/
lookup-collision issue on SME-series symbols specifically, not evidence
about MTF eligibility - worth fixing the lookup (likely needs to also
filter or prefer on `SEM_SERIES` when multiple rows match) before ever
relying on it for SME-listed stocks specifically.

## Practical recipe for a future MTF eligibility check

```python
def is_mtf_eligible(dhan_wrapper, security_id: str, price: float) -> bool:
    """Real-time check, not a static list - Dhan's own eligibility can
    change. See learnings/mtf-eligibility-detection.md for why `status`
    alone is not enough."""
    mtf = dhan_wrapper.client.Dhan.margin_calculator(
        security_id=security_id, exchange_segment="NSE_EQ",
        transaction_type="BUY", quantity=1, product_type="MTF", price=price,
    )
    if mtf.get("status") != "success":
        return False
    leverage = mtf.get("data", {}).get("leverage", "1X")
    return not leverage.upper().startswith("1.0") and not leverage.upper() == "1X"
```

Cheap (one REST call, no order placed), authoritative (asks Dhan directly
rather than guessing from `SEM_SERIES`/lot size/price), and correct against
all 3 confirmed real examples above. Retry-wrap it the same way every other
Dhan REST call site in `Options/dhan_client.py` is (`_retry`) before using
it on any real entry path - this endpoint is just as susceptible to Dhan's
documented transient rate-limit failures as every other one.
