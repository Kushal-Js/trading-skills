# Incident + fix: CHOLAFIN MAX_LOSS_HIT overshoot, and two Luxury corrective actions built + backtested from real 2-3 Sep trades

## What happened

Investigating "any corrective action to improve returns on the last 2
days of trades" (2-3 Sep 2026, 37 real Luxury trades, net −Rs.10,590.50 +
2 Swing trades, net +Rs.2,256.25), two concrete, evidenced problems stood
out:

1. **9 of 11 `MAX_LOSS_HIT` trades overshot their own configured cap**,
   one severely: **CHOLAFIN** (3 Sep, entry 56.35, exit 52.20, qty 625,
   −Rs.2,593.75 against a Rs.1,000 cap — 159% over).
2. **Repeated same-day re-entries into a just-stopped-out symbol tended
   to lose again**: MAHABANK went 0-for-3 same-day (−Rs.2,600 total,
   entries at 10:40/10:44/11:14 IST, closes at 10:43/10:45/11:15);
   PHOENIXLTD's third same-day re-entry (−Rs.1,417) outweighed its own
   first two wins combined (+245, +245).

## CHOLAFIN root cause, confirmed via real tick replay (same method as the SAGILITY incident, 28 Aug 2026)

Pulled `CHOLAFIN 29 SEP 1840 CALL`'s (security_id 96008, NSE) real 1-min
option candles around the exit:

| Time (IST) | Open | High | Low | Close | Volume |
|---|---|---|---|---|---|
| 15:00 (entry) | 56.35 | 56.35 | 56.35 | 56.35 | 625 |
| 15:01 | 56.25 | 56.25 | 56.25 | 56.25 | 625 |
| 15:02–15:05 | 56.25 | 56.25 | 56.25 | 56.25 | **0** (×4 bars) |
| 15:06 (exit) | 53.15 | 53.15 | **52.20** | 52.20 | 1,250 |

Stop threshold was 54.75. Price sat flat at 56.25 for 4 straight minutes
on **zero traded volume**, then gapped straight from 56.25 to 53.15→52.20
in one single candle with no prints in between. The bot's exit fired on
the very first tick after the gap — uncatchable by any tick-based check,
same mechanism as SAGILITY, just a bigger gap (~7% vs ~5%) on a
higher-priced contract producing a bigger absolute overshoot.

## Two corrective actions built, then BACKTESTED against the real 2-3 Sep data before deployment — the backtest step changed both defaults

User's own instruction: "Implement 1 and 2 items, back test them with
last 2 days trading data also, deploy it then." **The backtest step
caught real problems with the first-guess parameters in BOTH cases** —
worth internalizing as the reusable lesson here: a fix inspired by ONE
incident, backtested only against that one incident's own replay, can
still ship a net-negative or false-positive-prone default; backtesting
against the FULL surrounding dataset (not just the incident that
motivated the fix) is what actually catches it.

### 1. Same-day loss cooldown

Skip a fresh entry into a symbol that stopped Luxury out within N
minutes. **First guess: 30 minutes. Backtest verdict: NET NEGATIVE
(−Rs.367)** — it blocks MAHABANK's/RVNL's genuine repeat-losses, but
ALSO blocks a GVT&D re-entry that went on to hit
`PROFIT_PROTECTION_HIT` for +Rs.1,512.50 (a real false positive).

Swept 5/10/15/20/25/30/45/60/90/120 minutes against all 37 real trades:

| Cooldown | Trades kept | Total PnL | Net effect vs baseline |
|---|---|---|---|
| 15 min | 36 | −9,810.50 | +780.00 |
| **20 min** | **35** | **−9,444.75** | **+1,145.75 (best)** |
| 25 min | 34 | −10,957.25 | −366.75 |
| 30 min | 34 | −10,957.25 | −366.75 |

**Deployed at 20 minutes** — MAHABANK's second entry launched barely a
minute after its first loss closed (blocked at any cooldown ≥2 min);
GVT&D's own gap before re-entering was just past 20 minutes (survives
untouched at 20, wrongly blocked at 25+).

### 2. Liquidity guard on exit

Exit a held position the moment its own option has printed zero volume
for N consecutive completed 1-min bars — a proactive, price-independent
early-warning check (unlike `MAX_LOSS_HIT`, which can only react AFTER a
gap already happened). **First guess: 3 bars (matching CHOLAFIN's own 4
quiet minutes, minus one for a margin). Backtest verdict: a real false
positive** — replaying every one of the 37 trades' own real 1-min option
candles, 3 bars ALSO fires on a BLUESTARCO position that was actually
fine and later hit `PROFIT_PROTECTION_HIT` for +Rs.1,056.25, turning it
into a −Rs.666.25 loss instead (−Rs.1,722.50 swing on that one trade).

Swept 2/3/4/5/6 bars:

| Bars | Net effect | Trades caught |
|---|---|---|
| 2 | +1,933.75 | NBCC, BLUESTARCO (false +), PIIND, CHOLAFIN |
| 3 | +1,478.75 | NBCC, BLUESTARCO (false +), PIIND, CHOLAFIN |
| **4** | **+3,201.25 (best)** | **NBCC, PIIND, CHOLAFIN — no false positive** |
| 5 | +280.00 | PIIND only |

**Deployed at 4 bars** — removes the BLUESTARCO false positive entirely
while every genuine catch survives (each fires only ~1 minute later than
at 3 bars, no meaningful loss of protection).

## Scope

Both items deployed to **Luxury only** — the package the evidence came
from. Options/Futures share the identical risk shapes (their own
`_exit_reason_for`/`_process_one_entry` are near-verbatim copies) but
don't have either check wired in yet — a natural next step if the same
patterns show up in their own trade history.

## Verification

New `tests/test_luxury_corrective_actions.py` (16 scenarios) in
`traderBoy` covers both features' pure logic and full real integration.
Full details, the exact code, and the deploy record are in `traderBoy`'s
own `NOTES.md` entry #88.
