# Polymarket Weather Early Entry Bot

Enters the top 3–4 temperature bucket markets on Polymarket two days before
settlement, when markets first open with flat ~0.20 YES pricing.

The edge: Polymarket initializes all 11 Atlanta TMAX buckets at roughly equal
prices (~0.20). Our model knows, two days out, which 3–4 buckets are the
likely winners. Entering those at 0.20 before the market corrects is the
entire strategy.

---

## How it works

**Day N − 2 (~1am ET after NWP models run):**

1. Generate the model signal — runs D+1 ensemble with today's observations
   and day-after-tomorrow NWP forecasts, ranks all buckets by probability.

2. Start the bot — polls for the Polymarket event to go live. As soon as
   each target bucket's market gets a live price, the bot enters that bucket
   immediately with a GTC limit order. Buckets can go live at different times;
   each is entered the moment it appears.

**Day N (settlement):** positions resolve at $1/share if the bucket is correct.

---

## Setup

Ensure these are set in `.env`:

```
PRIVATE_KEY=0x...        # Polygon wallet private key
PROFILE_ADDRESS=0x...    # Polymarket funder address (same wallet)
```

The Python scheduler must be running so NWP forecast data is fresh in the DB.

---

## Workflow

One command. The bot handles everything:

```bash
# Dry run (no orders placed):
bun scripts/runPolymarketWeatherEarlyEntry.ts --date 2026-06-05 --dry-run

# Live:
bun scripts/runPolymarketWeatherEarlyEntry.ts --date 2026-06-05
```

The bot runs in three phases:

**Phase 1 — Wait for event.** Polls Polymarket every 10 seconds until the
Atlanta TMAX event for the target date appears (any markets, no price needed).

**Phase 2 — Generate fresh signal.** The moment the event is detected, runs
`python/generate_polymarket_d2_signal.py` as a subprocess to get the latest
model predictions using the current NWP forecast data. This ensures the
signal is never stale.

**Phase 3 — Watch and enter.** For each of the top N model buckets, enters
immediately when that specific market gets a live price. Buckets that go live
minutes apart are each entered at their own moment — no batching.

Orders are GTC limits at `current_ask + 2¢` (capped at `--max-price`).
State is saved to `data/polymarket-early-bot/atl/state.json`; safe to restart.

To preview the signal without running the bot:

```bash
python python/generate_polymarket_d2_signal.py --date 2026-06-05
# prints: #1  86-87F    34.1%
#         #2  84-85F    28.7%
#         #3  88-89F    17.2%
```

### Options

| Flag | Default | Description |
|------|---------|-------------|
| `--date YYYY-MM-DD` | day+2 | Settlement date to target |
| `--top-n N` | 3 | Number of buckets to enter |
| `--budget USD` | 1.00 | USD per bucket |
| `--max-price 0-1` | 0.26 | Skip if YES already above this |
| `--location code` | atl | Location (currently only `atl`) |
| `--dry-run` | off | Log decisions, place no orders |

### Example session

```
bun scripts/runPolymarketWeatherEarlyEntry.ts --date 2026-06-05 --dry-run --top-n 3

══════════════════════════════════════════════════════════════════════
  Polymarket Weather Early Entry [ATL] — 2026-06-05  (DRY RUN)
══════════════════════════════════════════════════════════════════════
  Signal: obs_date=2026-06-03  model=kalshi  generated=2026-06-03T01:14:22

  Watching 3 buckets (entering each as soon as it goes live):
    #1  86-87F      34.1%
    #2  84-85F      28.7%
    #3  88-89F      17.2%

  Polling for markets (10s interval)...
  [  1/360]  Still waiting: 86-87F, 84-85F, 88-89F
  [  7/360]  Still waiting: 86-87F, 84-85F, 88-89F
  → BUY YES  84-85F  model=28.7%  mkt=0.200  limit=0.220  4.5 shares  ≈$0.99
    ✓ dry_run_order  status=dry_run
  [ 13/360]  Still waiting: 86-87F, 88-89F
  → BUY YES  86-87F  model=34.1%  mkt=0.205  limit=0.225  4.4 shares  ≈$0.99
    ✓ dry_run_order  status=dry_run
  → BUY YES  88-89F  model=17.2%  mkt=0.200  limit=0.220  4.5 shares  ≈$0.99

  Done. 3 entered, 0 skipped.
```

---

## File inventory

| File | Purpose |
|------|---------|
| `python/generate_polymarket_d2_signal.py` | Runs D+2 inference, writes signal JSON |
| `python/models/weather/inference.py` | Ensemble model (`--obs-lag 2` for D+2) |
| `src/bots/polymarketWeatherEarlyBot.ts` | Bot logic (poll + enter) |
| `scripts/runPolymarketWeatherEarlyEntry.ts` | CLI runner |
| `docs/data/score_d2_polymarket_atl.json` | Signal file (generated, git-ignored) |
| `data/polymarket-early-bot/atl/state.json` | Run state (generated, git-ignored) |

---

## Model notes

The signal generator runs the **D+1 kalshi ensemble model** with `obs_lag=2`
(today's observations instead of yesterday's). This is deliberate: there is
no separate D+2 model, and training one requires historical D+2 forecast
data with matching outcomes. At the D+2 horizon the NWP forecast signal
dominates; using slightly stale observations degrades calibration a few
percent but the bucket rank-ordering — which is all that matters for this
strategy — is reliable.

If a polymarket-specific model artifact exists under
`python/models/weather/artifacts/atl/polymarket/`, the generator
automatically prefers it.

### Bucket alignment

Polymarket Atlanta TMAX markets use **even-lo** 2°F buckets:
≤73°F, 74–75°F, 76–77°F, …, 90–91°F, ≥92°F.

The signal file includes both `odd` and `even` ranked lists. The bot reads
`even`. Verify alignment by comparing the market questions fetched at open
against the model bucket labels.

### Edge case: market uses odd-lo buckets

Some dates may use odd-lo alignment (75–76, 77–78, …). If all your entries
miss (no market match for any top bucket), switch to:

```bash
# In the bot config or a future --bucket-start flag, use signal.odd instead
```

This hasn't been observed in practice but is worth checking if entries fail.
