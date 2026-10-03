# Unified Momentum end-to-end stress test (3 Oct 2026)

**Question:** before Unified Momentum's first real trading day (Mon 5 Oct 2026), does the whole bot stay up and
stay fast when candles form under heavy load and when Dhan misbehaves?

**Method.** The real bot (traderBoy `c3d0864`, every strategy) ran under uvicorn in an isolated copy on a Mac, with
the droplet's config and data. Dhan was replaced at the client layer by a fake broker: REST, order book with
market / limit / stop-limit fills, positions, funds, margin, both WebSockets, and Dhan's rate limits
(history > 5/s and quotes > 1/s return DH-904). All outbound network was blocked. The clock was set to Monday and
ran at real speed. History was the real cached 1-minute data; Monday was one simulated path per stock.

| Run | Sim time | Load |
|---|---|---|
| Baseline | 09:03–09:40 | Production load |
| Stress | 10:58–11:33 | 45 stocks, 5× ticks; REST slow / outage, DH-904 storm, WebSocket drops, slow and rejected orders, flash moves |
| Afternoon | 13:52–14:32 | Entry cutoffs |
| Close | 15:07–15:27 | Restart while holding a call and a put, then the 15:15 square-off |

The droplet's vCPU is 4.5× slower than the Mac on the same Python benchmark. CPU figures below are scaled by that.

## What held

- **Stability:** no crash, no dead thread and no memory growth in any run.
- **WebSocket:** reconnected 2.0 s after each drop.
- **Broker stops:** placed 0.05–1.1 s after each fill. Stop fills were noticed in 0.3–0.6 s.
- **Restart and close:** a restart adopted an open call and an open put. The 15:15 square-off placed its SELL 3.3 s after 15:15.
- **Rules:** cutoffs and slot limits held. No orphan or duplicate stops.
- **Rate limit:** the bot never broke Dhan's limit by itself.

## What was learned

1. **The 09:05 warm-up reloads what was used on the previous trading day.** After a holiday, Unified Momentum stocks that
   Swing does not trade lacked engine B's 15-min Supertrend base. Engine B's first 09:20 pass then downloaded them one
   by one, about 1 s each.
   - Check the bases file after every holiday or watchlist change.
2. **Engine B fetches cold history inline, one stock at a time.** Measured blind time:
   - 209 s after a restart (45 stocks)
   - 147 s with 2–4 s REST
   - 52–85 s after WebSocket drops

   Tick-driven exits and the broker stop were not affected.
3. **Engine A's trigger tick reaches the BUY after 5–6 Dhan calls made one after another.** Trigger → order took 1.2 s
   at baseline and 4–8 s under stress. Resolving the contract at the bar close would take most of this off the trigger path.
4. **A funds-refused put was retried every ~6 s for the whole candle.** One stock was retried 43 times. Each retry
   repeated the Dhan calls.
5. **With ₹1.08 lakh, a hedge put could not be funded while one call and the Scalper's put were open.** There was
   ₹34.5k free against a ₹42.8k put. The 1.2-lakh backtest assumes the hedge is always placed.
6. **A restart mid-session costs about a minute:**
   - pending orders 25–59 s late at the first bar
   - order queue wait 2–2.5 s
7. **CPU on the droplet:**
   - Production-like load: about 24% of the vCPU, consistent with 13–21% measured live.
   - 3× stocks with 5× tick rate: about 75%, with peaks above 100%.
8. **A latent `KeyError: 'close'` in the late-entry shadow check (UM and Super Bollinger).** `forming_bar()` exposes
   `last`, not `close`. It has never fired live.

Report: https://claude.ai/artifact/YVh7vwDKVuQPAKWnT65Xuo. Harness: traderBoy `.claude/test-wip/um_stress_harness/`.

**Fixed and deployed 3 Oct 2026 21:42 IST (traderBoy `8b790bc`):**
- Items 1–4 and 8 are fixed:
  - The warm-up now loads every series Unified Momentum declares for every listed stock. After a restart it runs at once, Unified Momentum stocks only.
  - Engine B warms cold stocks in the background.
  - The contract is resolved before the trigger.
  - A funds-refused put waits 60 s before the next try.
  - The `forming["last"]` field name is fixed.
- Measured in re-runs:
  - Engine B's 09:20 pass: 7.8 s → 0.3 s.
  - Trigger → order under stress: 3.9 s → 0.27 s.
  - Engine B's worst pass after a restart: 209 s → 46 s.
- Item 5 (capital) is the user's call; funds are being added 4 Oct.
- Item 6 (the restart minute) remains bound by Dhan's pacing.
