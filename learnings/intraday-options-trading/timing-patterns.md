# Intraday timing patterns — and where DhanBoy's own timing choices already line up (or don't)

Source: cross-referenced across Angel One, IIFL, Kotak Neo, and other
NSE-focused intraday-timing guides. Researched 30 Aug 2026.

## The general pattern (NSE, 9:15 AM – 3:30 PM IST)

- **09:15–09:30 (first 15 min)**: highest volatility, often the least
  reliable — reacting to overnight news and pre-market order flow, prone
  to unpredictable/false-signal swings.
- **09:30–10:30**: still elevated volatility, but more genuinely
  directional — opening-range-breakout patterns commonly form here.
- **10:00 onward specifically**: cited as where many intraday traders
  actually prefer to start, once the most chaotic opening volatility has
  settled.
- **10:30–12:00 (mid-morning)**: volatility settles further, moves can be
  slower/choppier.
- **Midday**: often the quietest stretch of the session.
- **14:30–15:30 (closing rush)**: activity picks back up from position
  adjustments, but the **last 30 minutes (15:00–15:30) are explicitly
  flagged as unreliable for starting a new intraday trade** — too little
  time left for a move to develop before the session ends.

## Where this independently validates a choice already made in this codebase

**K01's `DAILY_SCREEN_TIME=10:15`** (`K01/config.py`) was chosen for a
different, OI-buildup-specific reason at the time (session-level OI
comparisons are noisier in the opening minutes) — but it also happens to
land almost exactly on the general "volatility has settled enough to
trust the signal" boundary this research independently describes. Good
convergent validation that 10:15 wasn't an arbitrary choice, even though
it was reasoned to from a different angle originally.

## Where this is worth checking against the live bot's own entry cutoff

The live Options strategy's `ALLOWED_TRADING_TIME` cutoff mechanism
(`ENABLE_TRADING_TIME_LIMIT`, currently `false` per `traderBoy/.env`) is
currently disabled — new entries are allowed all day, right up to
`SQUARE_OFF_TIME`. This research doesn't argue for turning that back on
by itself (that was a deliberate, separately-reasoned decision paired with
the NRML/carry-forward change, per `traderBoy/NOTES.md`'s design-decision
entry) — but it does suggest that **if** entry-cutoff tuning is revisited,
the **last-30-minutes** window (15:00–15:30) is the specific stretch this
research flags as weakest for a *new* entry, more than the whole
afternoon broadly. Worth a note for whoever next reconsiders that
tradeoff, not a recommendation to change it now.

## What this does NOT establish

None of this is backtested against DhanBoy's own real trade data by
time-of-bucket — it's general market-structure research, not a finding
specific to this bot's actual historical performance. A genuinely useful
follow-up (not done here) would be: bucket the real trade history already
gathered this session (28 Aug's real trades, the various CSV backtests) by
entry hour and check whether win rate/P&L actually varies by time-of-day
the way this general research predicts, for *this specific strategy* on
*this specific instrument universe* (ATM stock options, not the
cash/futures market this research is more directly about).
