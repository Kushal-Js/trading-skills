# Stock selection: the hindsight trap, and a walk-forward test of selection rules (30 Sep 2026)

## The trap
Super Bollinger's backtest (+Rs 88,651 to +Rs 1,02,091, 31 Aug - 29 Sep) ran on the 15-stock watchlist
picked by ATH/composite on 27 Sep - using daily data that covers most of that same window. ATH ranks a
stock high BECAUSE it trended strongly in those weeks, which is exactly what the strategy profits from.
The mechanics were right (real option prices, honest entry timing, matched live paper trades); the
*stock list* was hindsight. **Any backtest on a watchlist chosen after the window started is optimistic.**

## Walk-forward test (research_stock_selection_walkforward.py)
Each Friday rank the 210 F&O stocks using only data up to that close, keep top 15, trade them the next
week with unchanged Super Bollinger rules (max 5 open, CE only, real option prices via Dhan rollingoption).
Decision dates 28 Aug, 4/11/18/25 Sep. This mirrors the live `dhanboy-weekly-watchlist-refresh.timer`
(Friday 00:00 IST, ATH top 15 -> both watchlists).

| Selection | Net (modeled) | H1 | H2 |
|---|---|---|---|
| Hindsight watchlist (27 Sep list) | +88,651 | +58,581 | +30,070 |
| STRATEGY_FIT (rules' own proxy PnL, prior 20 sessions, gated) | +23,827 | +9,519 | +14,308 |
| RANDOM #3 | +21,337 | -22,360 | +43,697 |
| RANDOM #2 | +3,881 | +23,576 | -19,694 |
| TREND_ATH_GATE | -27,836 | -23,955 | -3,881 |
| TREND_ATH (= live scheduler) | -31,003 | -19,823 | -11,180 |
| RANDOM #1 | -34,781 | -34,144 | -636 |
| MOMENTUM (20d ret / vol) | -49,481 | -16,856 | -32,625 |
| No selection (all 210) | -89,540 | -73,968 | -15,572 |

Takeaways: (1) ATH on its own is inside the random range for this strategy. (2) Luck is huge - three
random 15-stock draws span ~Rs 56k in one month, so one month cannot rank selection rules with
confidence. (3) STRATEGY_FIT is best and positive in both halves, but only at the top of the random range.
Next: 10 random draws for a real distribution, ATH-pool + FIT hybrid, daily refresh, and a longer window
(August) before trusting any rule.
