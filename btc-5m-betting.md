# BTC 5-minute betting (Polymarket up/down fair-value bot)

Orientation for the BTC side of the repo: what exists, how to run it, and where the history
lives. For what has already been tested (and rejected), read the investigation log
[python/models/btc_5m/README.md](../python/models/btc_5m/README.md) before trying a new idea.

## Status (2026-10-01)

- **Dry-run only. Not approved for real money.** The live bot runs with `--dry-run`.
- **Gathering data only (user decision, 2026-09-29).** Don't change the live decision rule
  while dry-run data accumulates, even for a backtest-validated change. A rule change splits
  the live sample into "before" and "after" and makes neither as trustworthy. The price ≥ 0.5
  filter, tiered sizing, and the model-probability exit stop were all exceptions made
  deliberately (each shipped the same day it was validated) — not a reversal of this rule, but
  don't take that as license to change things casually either.
- **Nothing schedules it.** There is no cron job, scheduler job or process-manager entry. The
  bot and the settler are started by hand.
- **The old Strat Bot is gone.** `syncStratBot.ts`, `python/api/main.py` and the
  `stop-loss-bot` scripts in `package.json` point to files that no longer exist. The
  fair-value bot reuses Strat Bot's `strat_order` table, and tags its rows with
  `decision->>'bot' = 'btc5m_fairvalue'`.
- **Open issue, root cause found 2026-10-03:** the live trigger rate is well above the
  backtest's because `data-api /trades` is delayed ~2 minutes live (median ~105s), so the bot's
  "market price" was a stale print while the model is fresh. **Fixed 2026-10-03**: the signal
  now comes from the CLOB market websocket (~0.5s lag). **Confirmed 2026-10-05**: checkpoints
  are evenly populated again and trade prices match the book. The remaining trigger-rate gap
  (48% live vs 18% all-weeks backtest) is regime drift — the backtest's own last two weeks
  show 34-38% — not a defect. Treat all pre-2026-10-03 live decisions/P&L as a different
  (flawed-signal) sample. **Still open: whether the edge exists in the newest regime.** Only 26
  ws-signal dry-run trades exist so far (ROI -4.5%, inconclusive); live volume is ~13/day.
  Don't go live on current evidence.
- **New, unconfirmed against live data:** the model-probability exit stop (added 2026-10-01)
  hasn't fired in live/dry-run yet. Watch the investigation log's "Open/unresolved" section for
  how that plays out before trusting it the way the entry-side filters are now trusted.

## What it bets on

Polymarket's BTC 5-minute up/down markets. Each market pays $1 if BTC ends the 5 minutes above
its starting price. The model treats a market as a digital option, with fair P(up) = N(d2):
the move so far is `ln(S_t / S_0)`, and volatility is trailing 30-minute realized volatility
from Coinbase ticks.

The live rule, as of 2026-10-01 (the log gives the reasoning behind each item):

1. The bot checks each market at T-240, 180, 120 and 60 seconds, polling every 5 seconds.
2. The market price is the last real trade for each token, from Polymarket's CLOB market
   websocket (`polymarketMarketWsClient.ts`; since 2026-10-03 — `data-api /trades` is delayed
   ~2 minutes live and was the source before that). If either side's last trade is more than 90
   seconds old (or none has been seen since the market's subscription), the bot skips that check.
3. A side is bet only if its edge (model probability minus market price) exceeds 0.10.
4. **Momentum only:** it bets only the side currently ahead, never the contrarian side.
5. **Magnitude filter:** it requires `|ln(S_t/S_0)| >= 0.0003 * sqrt(elapsed_secs / 60)`.
6. **Price filter:** the side's own signal price must be ≥ 0.5 (`MIN_ENTRY_PRICE`).
7. **One entry per market:** the first checkpoint that passes all filters wins.
8. **Entry order:** a GTC buy at best ask + 1¢, sized from the live order book.
9. **Size:** $1 flat by default, or `--bet-fraction-of-balance 0.01` for 1% of the USDC
   balance (floored at $1, capped at 5%), then scaled 1.5x/1.0x/1.25x by a price tier
   (`priceTierWeight`: <0.65 / 0.65-0.85 / >0.85).
10. **Exit stop (added 2026-10-01):** every poll tick while a position is open, the bot
    re-evaluates the model's own probability for the held side. If it drops below
    `MODEL_PROB_STOP_THRESHOLD` (0.40), the bot exits immediately with a market SELL (FOK) —
    no near-resolution cutoff, it keeps trying right up to the market's end time. Otherwise the
    position rides to settlement as before.

## Running it

```bash
bun run btc5m-fairvalue-bot-dry        # always start here
bun run btc5m-fairvalue-bot            # real orders (not approved yet)
bun src/bots/btc5mFairValueBot.ts --dry-run --edge-threshold 0.07 --max-bets-per-day 10 --bet-fraction-of-balance 0.01
bun run btc5m-fairvalue-settle         # settles resolved strat_order rows; optional --run-key <key>
```

Research and backtests use the Python venv (`python/.venv`) and run from `python/models/btc_5m/`
(e.g. `python python/models/btc_5m/backtest.py`, `random_kfold_cv.py`). `data.py` caches
Postgres pulls in `python/models/btc_5m/cache/`.

## Where things live

| What | File |
|---|---|
| Live bot | `src/bots/btc5mFairValueBot.ts` |
| Settler (writes `final_value` once a market resolves) | `src/bots/btc5mFairValueSettler.ts` |
| Model, live TS port | `src/services/btcFairValueModel.ts` |
| Model, reference Python | `python/models/btc_5m/fair_value.py` |
| BTC price feed (live) | `src/clients/coinbaseWsClient.ts` (has a 30s no-message watchdog) |
| Polymarket trades / book / market lookup | `src/clients/polymarketClient.ts` (`getRecentTrades`, `getOrderBook`) |
| Wallet balance for sizing | `src/clients/collateralBalanceClient.ts` |
| Backfills | `src/processors/backfillBtc5mHistory.ts` (markets, outcomes), `backfillBtc5mRealTrades.ts` (real trade prints into `market_price_history`), `backfillCoinbaseBtcTicks.ts` (dense ticks into `btc_trade`) |
| Research scripts + investigation log | `python/models/btc_5m/` |

DB tables: `market` / `clob_market_token` / `market_outcome` (market type `btc_5m` = 5),
`market_price_history` (real trade prints since the rebuild), `btc_trade` (BTC ticks),
`strat_order` (the bot's orders and decisions — `exit_price`/`exit_reason`/`exited_at`
(migration 30) are set together, alongside `final_value`, on an early exit; `NULL` on a
normal held-to-settlement trade).

## Gotchas

- **Keep the two model copies in sync.** `btcFairValueModel.ts` is a hand port of
  `fair_value.py`. They were cross-checked once (2026-09-18), and no test catches drift.
- **Price source matters.** Gamma market snapshots are stale, and CLOB `/book`'s
  `lastTradePrice` is market-wide, not per token. Only `/trades` gives real per-token prints.
  `/book`'s `bestBid`/`bestAsk` are per token and fine for execution.
- **Validate with random 10-fold CV by market**, not a single chronological split. There are
  only about 14 weeks of data. Report results at both zero cost and 2¢ cost.
- **Never size with Kelly on the raw model probability.** The model is overconfident in the
  tail, and Kelly sizing hit a 97%+ drawdown in backtest.
- **Tick density drives the backtest.** Weeks backed by sparse Binance.us ticks produced a fake
  edge; those weeks were re-sourced from Coinbase. `api.binance.com` is geoblocked from this
  environment.
- **Open positions are tracked in memory only**, not in the DB. A bot restart while a position
  is open means that position rides to settlement unmonitored by the exit stop (the settler
  still settles it normally once the market resolves — it just never gets a chance to exit
  early). Short-lived in practice (positions live at most ~4 minutes), but worth knowing before
  assuming every position got a fair shot at the exit stop.
- **The exit is a market order (FOK), not a limit order**, on purpose — a stop that might not
  fill defeats the point. If the SELL fails (e.g. a FOK with no fillable liquidity), the bot
  logs an error and leaves the position open to retry next poll tick (5s later); it does not
  fall back to a limit order or widen anything.
