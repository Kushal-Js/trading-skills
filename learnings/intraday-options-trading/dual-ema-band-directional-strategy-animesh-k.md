# Dual-EMA-band directional intraday options strategy (Animesh K, Upsurge Club)

**Source:** [Upsurge Club podcast ft. Animesh K, "It's Impossible to Lose with
THIS Strategy!"](https://www.youtube.com/watch?v=o5i8WF0UIfE), published
2026-05-01, ~46 min. Animesh K describes ~17 years trading experience and
sells a paid "Dual Edge Trading" course covering the extended version of this
setup. This file is a transcript-derived writeup of the free version he
walks through on air — **not something we've backtested or verified against
our own data.** Treat every rule below as a hypothesis to test, not a
confirmed edge. Scope: intraday options buying (index or single-stock), not
selling, not swing.

## Core mechanism

Two charts, same instrument:

1. **Left chart — Daily timeframe, Heikin-Ashi candles.** Used only to set a
   single directional bias for the *entire next trading day*. Rule: if
   yesterday's daily HA candle closed green → only look for longs (call
   buying) today; if red → only look for shorts (put buying) today. No
   counter-trend trades regardless of what intraday price does. He frames
   this as removing the buy/sell decision entirely before the session
   starts — the only open question left is timing.
2. **Right chart — lower intraday timeframe, also Heikin-Ashi (exact minutes
   not stated on air).** This is the execution chart. Indicator: two EMAs,
   both length **34** (he calls out 34 as a Fibonacci number, no other
   justification given) — one EMA(34) plotted on **High**, one EMA(34) on
   **Low**. The two lines form a band.
   - Long bias day: wait for a candle to **close above** the upper band
     (EMA-34-High). That close is the entry trigger. Enter at/after that
     close, using the signal candle's high as a soft reference.
   - Short bias day: wait for a candle to **close below** the lower band
     (EMA-34-Low). Same logic, mirrored.
   - **No trade while price sits between the two bands** — that's read as a
     flat/consolidating session and explicitly skipped.
3. **Exit:** either (a) the signal candle's opposite extreme as a hard stop
   (e.g. long trade stops out if price closes back below the entry candle's
   low / re-enters the band), or (b) trail the position and only exit when
   price closes back through the *opposite* band edge, riding the move as
   long as it stays outside the band. He states explicitly he does not use a
   fixed rule between "book at 1:2 / 1:3 R" vs. "trail to band re-entry" —
   his advice is to pick one, commit to it, and test it over 100+ trades
   before switching, because flip-flopping between the two (e.g. deciding
   mid-trade that today you'll trail, after yesterday you booked 1:2) is
   itself named as a discipline failure.
   - One worked example in the video: entered short, took two small losses
     on failed short attempts the same multi-day down-trend, then the third
     short trade ran far enough to cover all prior losses plus a large net
     profit — used to argue that the first 1-2 stopped-out entries on a
     correct-bias day are "cost of doing business," not a reason to abandon
     the bias.

## Instrument selection layer (separate from the entry/exit mechanism)

- Prefers **stock options over index options** most of the time, reasoning:
  stock futures/options give a filtering exercise (find the day's momentum
  stock) that index trading doesn't, and premium-to-move ratio can be more
  favorable on a stock with a real news-driven move (his framing: a stock
  might carry a smaller lot/cheaper premium than Nifty but see 5%+ moves).
  Explicitly says index (Nifty/BankNifty) is "just another stock" to him,
  not a default.
- **Morning routine:** open business news TV at 9:00, mute or keep low
  volume, close it by 9:15. Purpose stated narrowly — get a *sector* hint
  (which sector is in the news today), not a specific stock pick or a
  trade signal. He argues color-coded stock tickers on financial TV
  primed his own bias when he tried to mute-and-watch, so he uses it only
  for sector rotation awareness, not stock selection.
- **Evening homework:** build a 4-5 stock watchlist the night before from
  whichever sector had news, using a broker scanner (he references Dhan's
  scanner) filtered for liquidity. Explicitly rejects trying to track the
  full stock universe — narrows to a handful and stays there.
- **Option strike/expiry mechanics stated on air:** use ATM options (avoids
  theta decay dominating on OTM, avoids paying for delta you don't need vs.
  ITM); avoid trading on expiry day itself; for stock options (monthly
  expiry only, no weekly), roll to next month's contract ~1-2 trading days
  before expiry (example given: 26th expiry → shift by the 24th/25th) citing
  liquidity drop-off in the expiring contract.

## Psychology/discipline claims (unverified, listed for completeness)

- Calls stop-loss "soft loss" (SL) deliberately — framed as a small,
  expected cost, not a failure, to reduce the tendency to widen or skip
  stops after a red trade.
- Warns against tracking mark-to-market (M2M) intraday — says watching M2M
  changes behavior mid-session the way watching a scoreboard changes an
  athlete's focus; the fix he gives is hiding the P&L ticker and watching
  chart + strategy only.
- Names three pillars: trader psychology, trading system, risk management —
  says a setup alone (system) fails without the other two.
- Recommends defining a profit **target in ROI %**, not an open-ended "as
  much as possible," and says most of the emotional volatility (overconfidence
  after a win, revenge-trading after a loss) traces back to not having that
  number fixed in advance.
- Distinguishes discretionary vs. systematic trading style and says traders
  with <10 years of chart experience (explicitly includes post-COVID retail
  traders) should default to systematic/rule-based, not discretionary reads
  of price action.
- Recommends paper-testing/backtesting any new strategy over a minimum of
  100 trades before going live with real capital.

## Open questions / things NOT specified on air

- Exact intraday timeframe for the execution (right) chart — never stated a
  number of minutes.
- Whether the 34-EMA-High/Low band is meant to also gate re-entries after a
  stop-out on the same bias-day, or just the first signal.
- Near the end of the video he references "20 34 EMA" as the filter he
  "created" — inconsistent with the "both EMAs are 34" instruction given
  earlier in the same video. Likely either a verbal slip, or a second EMA(20)
  used in the paid course's extended version that isn't detailed in the free
  video. Don't assume 20 belongs anywhere in this setup without further
  confirmation.
- No stated position-sizing rule beyond "define your own."

## Why this is here, not acted on

Per [[project-trading-skills-repo]] this repo captures external ideas worth
evaluating, it does not imply endorsement or that this has been checked
against our own price data. If this gets backtested against DhanBoy's
historical feeds, log the result as a new file here (or update this one)
citing the actual numbers — don't overwrite this summary with a verdict
without evidence attached.
