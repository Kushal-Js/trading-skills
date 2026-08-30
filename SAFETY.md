# Safety boundary

This repo makes the assistant (Claude) a better-informed analyst of DhanBoy's
trading decisions. It does **not** change the following, no matter how much
knowledge accumulates here:

- **No real order is ever placed without the user explicitly asking for it in
  that conversation.** Not "the knowledge base suggests this trade looks
  good" — an actual, current, human go-ahead, every time.
- **The live bot is never started or restarted without checking first** that
  no live positions are open (or, if they are, that the user understands what
  a restart does to them), and without the user's go-ahead for that specific
  restart.
- **No autonomous strategy changes go live** without the same deploy
  discipline used throughout traderBoy's history: change the config, test it
  offline against the real `.env`, confirm live positions are empty
  (twice — once before syncing, once immediately before restart), deploy,
  verify the *runtime* value on the droplet itself (not just the file).

## Why "self-learning" and "autonomous execution" are different goals

The stated long-term goal is a bot that makes better decisions and loses
less — that's a knowledge/analysis problem, and this repo helps with it
directly. It is a *different* goal from an assistant that places trades or
restarts the live service on its own judgment, and accumulating documentation
here is never treated as authorization to cross that line. If a future
session (this one included) is ever tempted to read "the docs say X is a good
setup, so let's just do it" — that's exactly the pattern to catch and stop
before it reaches a real order or a real restart.

"Everyday consistent earnings" is a stated goal, not a guarantee this repo
can produce. Markets and thin-liquidity option contracts (see
`learnings/exit-mechanics.md`) produce genuine, unpreventable losses on some
trades no matter how good the underlying knowledge is — the honest goal is
better-informed decisions and fewer *avoidable* mistakes, not zero losses.
