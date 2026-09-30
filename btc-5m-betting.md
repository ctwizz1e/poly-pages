# BTC 5-minute betting (Polymarket up/down fair-value bot)

Orientation for the BTC side of the repo: what exists, how to run it, and where the history
lives. For what has already been tested (and rejected), read the investigation log
[python/models/btc_5m/README.md](../python/models/btc_5m/README.md) before trying a new idea.

## Status (2026-09-29)

- **Dry-run only. Not approved for real money.** The live bot runs with `--dry-run`.
- **Gathering data only (user decision, 2026-09-29).** Don't change the live decision rule
  while dry-run data accumulates, even for backtest-validated filters like price ≥ 0.5. A rule
  change would split the live sample.
- **Nothing schedules it.** There is no cron job, scheduler job or process-manager entry. The
  bot and the settler are started by hand.
- **The old Strat Bot is gone.** `syncStratBot.ts`, `python/api/main.py` and the
  `stop-loss-bot` scripts in `package.json` point to files that no longer exist. The
  fair-value bot reuses Strat Bot's `strat_order` table, and tags its rows with
  `decision->>'bot' = 'btc5m_fairvalue'`.
- **Open issue:** the live trigger rate is still well above the backtest's rate (see "Open /
  unresolved" in the investigation log). Don't treat it as resolved because the live win rate
  looks fine at small n.

## What it bets on

Polymarket's BTC 5-minute up/down markets. Each market pays $1 if BTC ends the 5 minutes above
its starting price. The model treats a market as a digital option, with fair P(up) = N(d2):
the move so far is `ln(S_t / S_0)`, and volatility is trailing 30-minute realized volatility
from Coinbase ticks.

The live rule, as of 2026-09-28 (the log gives the reasoning behind each item):

1. The bot checks each market at T-240, 180, 120 and 60 seconds, polling every 5 seconds.
2. The market price is the last real trade for each token, from `data-api.polymarket.com/trades`.
   If either side's last trade is more than 90 seconds old, the bot skips that check.
3. A side is bet only if its edge (model probability minus market price) exceeds 0.10.
4. **Momentum only:** it bets only the side currently ahead, never the contrarian side.
5. **Magnitude filter:** it requires `|ln(S_t/S_0)| >= 0.0003 * sqrt(elapsed_secs / 60)`.
6. **One entry per market:** the first check that passes wins.
7. **Order:** a GTC buy at best ask + 1¢, sized from the live order book.
8. **Size:** $1 flat by default. `--bet-fraction-of-balance 0.01` sizes at 1% of the USDC
   balance, floored at $1 and capped at 5%.

A price ≥ 0.5 filter is validated in backtest but **not implemented in the bot yet**.

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
`strat_order` (the bot's orders and decisions).

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
