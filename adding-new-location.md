# Adding a New Weather Trading Location

This guide covers every step required to add a new Kalshi high-temperature market location, from database setup through live prediction and the poly-pages tab. Steps are ordered by dependency — complete them in sequence.

The example throughout uses **LAX** (Los Angeles, `KXHIGHLAX`, station `KLAX`).

---

## Parameters to collect before starting

| Parameter | ATL example | DC example | LAX example |
|-----------|-------------|------------|-------------|
| Location code (uppercase) | `ATL` | `DC` | `LAX` |
| ASOS station ID | `KATL` | `KDCA` | `KLAX` |
| Kalshi series ticker | `KXHIGHTATL` | `KXHIGHTDC` | `KXHIGHLAX` |
| Display name | `Atlanta, GA` | `Washington, DC` | `Los Angeles, CA` |
| Latitude | 33.6407 | 38.8521 | 33.9425 |
| Longitude | -84.4277 | -77.0377 | -118.4081 |
| DB `location_id` | 1 | 2 | 3 (next sequential) |

Confirm the Kalshi series ticker from the market URL, e.g. `kalshi.com/markets/kxhighlax/` → ticker is `KXHIGHLAX`.

---

## Phase 1 — Database

### 1.1 Create migration

Create `migrations/17_add_lax_location.sql` (increment the number for your new location):

```sql
INSERT INTO public.weather_location (code, station_id, kalshi_series, display_name)
VALUES ('LAX', 'KLAX', 'KXHIGHLAX', 'Los Angeles, CA');
```

### 1.2 Run the migration

```bash
psql "$DATABASE_URL" -f migrations/17_add_lax_location.sql
```

Verify:

```sql
SELECT id, code, station_id, kalshi_series FROM weather_location ORDER BY id;
-- should show ATL=1, DC=2, LAX=3
```

The `id` here is the `location_id` you will use in the TypeScript processor config.

---

## Phase 2 — Python configuration

### 2.1 `python/models/weather/db.py` — add to `LOCATION_CONFIGS`

[db.py:45](../python/models/weather/db.py#L45)

```python
LOCATION_CONFIGS: dict[str, dict] = {
    "atl": { ... },
    "dc":  { ... },
    "lax": {
        "station_id":    "KLAX",
        "kalshi_series": "KXHIGHLAX",
        "lat": 33.9425,
        "lon": -118.4081,
        "markets": {
            "kalshi": {
                "snapshot_quality": "klax-madis-hfm",
                "daily_quality":    "klax-madis-hfm-daily",
            },
            "polymarket": {
                "snapshot_quality": "klax-wunder-52m",
                "daily_quality":    "klax-wunder-52m-daily",
            },
        },
    },
}
```

### 2.2 `python/scheduler.py` — add to `_LOCATION_NAMES`

[scheduler.py:81](../python/scheduler.py#L81)

```python
_LOCATION_NAMES: dict[str, str] = {
    "atl": "Atlanta, GA",
    "dc":  "Washington, DC",
    "lax": "Los Angeles, CA",
}
```

> `LOCATIONS` itself is loaded from the DB at startup, so no change needed there once the migration runs.

### 2.3 `python/scheduler.py` — fix hardcoded MADIS sync loop

[scheduler.py:1006](../python/scheduler.py#L1006) — `run_madis_sync()` currently hardcodes KATL and a `dc` check. Replace with a loop:

```python
def run_madis_sync() -> None:
    for loc in LOCATIONS:
        lconf = get_location_config(loc)["markets"]["kalshi"]
        station = get_location_config(loc)["station_id"]
        _run_madis_sync_for(station, lconf["snapshot_quality"])

    try:
        today = datetime.now().strftime("%Y-%m-%d")
        n = derive_wunder52m_from_madis(today, today)
        if n:
            logger.info(f"[wunder-52m] {n} rows derived from MADIS")
    except Exception as e:
        logger.error(f"[wunder-52m] derivation failed: {e}")
```

### 2.4 `python/scheduler.py` — fix hardcoded daily rollup loop

[scheduler.py:1021](../python/scheduler.py#L1021) — `run_daily_rollup()` has the same ATL/DC pattern. Replace the per-location blocks with:

```python
for loc in LOCATIONS:
    lconf = get_location_config(loc)
    station = lconf["station_id"]
    kconf = lconf["markets"]["kalshi"]
    try:
        three_days_ago = (datetime.now() - timedelta(days=3)).strftime("%Y-%m-%d")
        run_iem_backfill(
            _date.fromisoformat(three_days_ago), _date.fromisoformat(today),
            delay_s=1.0, station=station
        )
        logger.info(f"[rollup] IEM ASOS precip backfill {station} OK")
    except Exception as e:
        logger.error(f"[rollup] IEM ASOS precip backfill {station} failed: {e}")
    try:
        ok = aggregate_madis_daily(
            today, station=station,
            data_quality=kconf["snapshot_quality"],
            daily_quality=kconf["daily_quality"],
        )
        logger.info(f"[rollup] aggregate_madis_daily {station} {today}: {'OK' if ok else 'no snapshots'}")
    except Exception as e:
        logger.error(f"[rollup] {station} aggregate failed: {e}")
```

---

## Phase 3 — TypeScript processor

### 3.1 `src/processors/weatherHistoricalBackfillProcessor.ts` — add to `LOCATION_CONFIGS`

[weatherHistoricalBackfillProcessor.ts:40](../src/processors/weatherHistoricalBackfillProcessor.ts#L40)

```typescript
const LOCATION_CONFIGS: Record<string, { location_id: number; lat: number; lon: number; stationId: string }> = {
  atl: { location_id: 1, lat: 33.6407, lon: -84.4277, stationId: "KATL" },
  dc:  { location_id: 2, lat: 38.8521, lon: -77.0377, stationId: "KDCA" },
  lax: { location_id: 3, lat: 33.9425, lon: -118.4081, stationId: "KLAX" },
};
```

The `location_id` must match the `id` from `weather_location` in the DB (step 1.2).

---

## Phase 4 — Frontend

### 4.1 `docs/index.html` — add nav link

[index.html:523](../docs/index.html#L523)

```html
<nav id="location-nav">
  <a href="?location=atl" class="loc-link">Atlanta</a>
  <a href="?location=dc" class="loc-link">Washington DC</a>
  <a href="?location=lax" class="loc-link">Los Angeles</a>
</nav>
```

### 4.2 `docs/index.html` — add label

[index.html:793](../docs/index.html#L793)

```javascript
const LOCATION_LABELS = { atl: "Atlanta", dc: "Washington DC", lax: "Los Angeles" };
```

> The data file paths (`data/intraday_even_hours_today_kalshi_${LOCATION}.json`, etc.) already use the `LOCATION` variable and require no changes.

---

## Phase 5 — Historical data backfill

Run these once to populate the DB with enough history to train models. The scheduler will take over for live data afterward.

### 5.1 Forecast backfill (Open-Meteo)

```bash
# D1 NWP forecast history (Open-Meteo previous runs, 2020–present)
bun src/processors/weatherHistoricalBackfillProcessor.ts open-meteo-previous-runs 2020-01-01 2026-06-24 1 lax
bun src/processors/weatherHistoricalBackfillProcessor.ts open-meteo-previous-runs 2020-01-01 2026-06-24 2 lax
bun src/processors/weatherHistoricalBackfillProcessor.ts open-meteo-previous-runs 2020-01-01 2026-06-24 3 lax

# Recent forecast (seed current forecasts)
bun src/processors/weatherHistoricalBackfillProcessor.ts open-meteo-recent-forecast 2026-06-01 2026-06-24 "" lax
```

### 5.2 Observation backfill (IEM ASOS)

```bash
# Intraday snapshots (MADIS-style HFM, ~5-min resolution since 2016)
bun src/processors/weatherHistoricalBackfillProcessor.ts observations-iem-daily 2020-01-01 2026-06-24 "" lax

# Or use the Python IEM backfiller directly (handles station param):
python python/models/weather/backfill_iem_asos.py \
    --start 2020-01-01 --end 2026-06-24 --station KLAX
```

### 5.3 Intraday snapshot backfill (Synoptic)

```bash
bun src/processors/weatherHistoricalBackfillProcessor.ts observations-synoptic-intraday 2023-01-01 2026-06-24 "" lax
bun src/processors/weatherHistoricalBackfillProcessor.ts observations-synoptic-daily 2023-01-01 2026-06-24 "" lax
```

---

## Phase 6 — Model training

Use a summer-only date range (`--months 5,6,7,8,9`) during active trading season. Match the `--end` date to the last day you have data for.

### 6.1 Daily ensemble

```bash
python python/models/weather/train.py \
    --start 2020-01-01 --end 2026-06-24 \
    --months 5,6,7,8,9 --location lax
```

### 6.2 Intraday model

```bash
# --out must point to artifacts/lax so train_meta can find it
python python/models/weather/train_intraday.py \
    --start 2023-01-01 --end 2026-06-24 \
    --months 5,6,7,8,9 --out python/models/weather/artifacts/lax
```

### 6.3 Meta model (daily + intraday fusion)

```bash
python python/models/weather/train_meta.py \
    --start 2020-01-01 --end 2026-06-24 \
    --months 5,6,7,8,9 --location lax
```

### 6.4 Max-timing model

```bash
python python/models/weather/train_max_timing.py \
    --start 2023-01-01 --end 2026-06-24 \
    --out python/models/weather/artifacts/lax
```

### 6.5 Give-up classifier (shared, re-train to include LAX data)

```bash
python python/models/weather/train_give_up_classifier.py \
    --start 2023-01-01 --end 2026-06-24
```

---

## Phase 7 — Smoke-test predictions

```bash
# Daily ensemble inference
python python/models/weather/inference.py --date 2026-06-24 --location lax

# Intraday inference
python python/models/weather/inference_intraday.py \
    --date 2026-06-24 --compare-d1 --location lax

# Edge report (extreme-bucket NO-bet strategy)
python python/models/weather/edge_report.py --date 2026-06-24 --location lax
```

Check that `docs/data/*_lax.json` files are populated after running inference.

---

## Phase 8 — Live scheduler

The scheduler reads `LOCATIONS` from the DB at startup. Once the migration is applied and the code changes above are deployed, restart the scheduler:

```bash
python python/scheduler.py
```

Verify in the logs:
- `[madis]` lines for `KLAX`
- `[rollup]` lines for `KLAX`
- Intraday prediction lines for `lax`
- JSON output files `docs/data/*_lax.json` being written each cycle

---

## Phase 9 — poly-pages tab

Once `docs/data/*_lax.json` files are present and the HTML changes from Phase 4 are committed, the poly-pages site will show a **Los Angeles** tab at `?location=lax`. The tab goes live automatically on the next GitHub Pages deploy.

Verify at: `https://<your-pages-url>/index.html?location=lax`

---

## Quick-reference checklist

### Code changes (required for every new location)
- [ ] Create migration `migrations/17_add_<loc>_location.sql` and run it
- [ ] `python/models/weather/db.py` — add entry to `LOCATION_CONFIGS`
- [ ] `python/scheduler.py` — add entry to `_LOCATION_NAMES`
- [ ] `python/scheduler.py` — refactor `run_madis_sync()` to loop (one-time fix, benefits all locations)
- [ ] `python/scheduler.py` — refactor `run_daily_rollup()` to loop (one-time fix)
- [ ] `src/processors/weatherHistoricalBackfillProcessor.ts` — add entry to `LOCATION_CONFIGS`
- [ ] `docs/index.html` — add nav link (line ~524) and label (line ~793)

### Data & model steps (per new location)
- [ ] Backfill NWP forecasts (Open-Meteo previous runs, D1/D2/D3)
- [ ] Backfill IEM ASOS observations from 2020-01-01
- [ ] Backfill Synoptic intraday snapshots from 2023-01-01
- [ ] Train daily ensemble (`train.py`)
- [ ] Train intraday model (`train_intraday.py`)
- [ ] Train meta model (`train_meta.py`)
- [ ] Train max-timing model (`train_max_timing.py`)
- [ ] Retrain give-up classifier (`train_give_up_classifier.py`)
- [ ] Smoke-test inference for today's date
- [ ] Restart scheduler and verify logs
- [ ] Confirm poly-pages tab loads at `?location=<code>`
