# A per-symbol pandas scan on the event loop froze the bot ~18 s after every restart

**Found:** 1 Oct 2026, while checking REST congestion after the Unified Momentum deploy (traderBoy `eca632f`).

**Symptom.** Right after "Application startup complete" the bot logged nothing for ~18.6 s, and the first HTTP
request in that window (safe_restart's `GET /unified-momentum/restart-report`, 10 s timeout) timed out - twice in
one evening. The bot looked healthy (`/health` had just answered) and came back on its own, so it was easy to miss.

**Cause.** The breakout dispatcher filters index/MCX symbols out of its restored PE watchlist with
`dhan_wrapper.is_mcx_commodity(s)` for ~150 symbols in a list comprehension inside an `async` function - i.e. ON
the event loop. `is_mcx_commodity` caches per symbol, but each FIRST lookup ran a pandas `str.startswith` over the
whole ~200k-row instrument master: 33 ms on a Mac, ~0.1 s on the 1-vCPU droplet. 150 first lookups = the whole bot
(price checks, exits, entries, HTTP) frozen ~18 s. Every restart repays it - including mid-session restarts, of
which 1 Oct had five.

**Fix.** Answer from a `frozenset` of MCX FUTCOM underlyings built in one pass per instrument DataFrame object
(~1 us per lookup). Proven identical to the old predicate on every name in two instrument masters (212,511 and
223,967 names, 0 mismatches) before deploying. Freeze after restart: 18.6 s -> 0.2 s.

**Rules of thumb.**
- A "cached" helper is only cheap on the second call - check what its first call costs and who calls it in bulk.
- Anything that scans the instrument master (or any big DataFrame) per item must not run on the event loop in a
  loop over symbols; precompute a set/dict once, or run the batch in the executor.
- A silent gap in the logs right after startup is a symptom worth timing - log silence + a timed-out first request
  = a blocked loop, not a quiet bot.
- Python's stdout to journald is block-buffered: library `print` lines (Tradehull's login/instrument messages) all
  carry the FLUSH time, not when they happened - use the logging module's own timestamps to time things.
