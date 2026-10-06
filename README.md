# MLB Series Stats Pipeline

ELT pipeline ingesting MLB game data into BigQuery with dbt transformations and a Looker/Data Studio dashboard. Built to demonstrate data engineering fundamentals including idempotent incremental loads, dimensional modeling, and automated orchestration.

## Architecture

```
MLB Stats API
    ↓ Python extraction (daily via GitHub Actions)
Google Cloud Storage (raw JSON, Hive-partitioned)
    ↓ Python loading (BigQuery load jobs)
BigQuery: mlb_raw (partitioned tables)
    ↓ dbt Core transformations
BigQuery: mlb_staging (views) → mlb_marts (tables)
    ↓
Looker/Data Studio Dashboard
```

## Stack

| Layer | Tool |
|---|---|
| Orchestration | GitHub Actions |
| Extraction | Python, MLB Stats API |
| Raw Storage | Google Cloud Storage |
| Warehouse | BigQuery |
| Transformation | dbt Core + dbt-bigquery |
| Visualization | Looker/Data Studio |
| Dependency Management | uv |

## Data Model

### Raw Layer (`mlb_raw`)
Append-only tables partitioned by `game_date`. Loaded via BigQuery load jobs for DML compatibility.

| Table | Grain | Description |
|---|---|---|
| `raw_team_game_stats` | Team × Game | Team batting and pitching stats per game |
| `raw_player_batting_stats` | Player × Game | Individual batter stats per game with season snapshot |
| `raw_player_pitching_stats` | Player × Game | Individual pitcher stats per game with season snapshot |

### Staging Layer (`mlb_staging`)
Views over raw tables. Column renaming and type coercion.

### Intermediate Layer (`mlb_staging`)
| Model | Description |
|---|---|
| `int_game_series_map` | Assigns series_id surrogate key using MIN(game_date) window over home/away matchup. Handles rescheduled games via QUALIFY deduplication. |

### Mart Layer (`mlb_marts`)
Materialized tables. Series-level aggregations with rate stats recomputed from counting stats.

| Table | Grain | Key Metrics |
|---|---|---|
| `fct_batter_series_stats` | Batter × Series | AVG, OBP, SLG, OPS, HR, RBI |
| `fct_team_series_batting` | Team × Series | AVG, OBP, SLG, runs, hits |
| `fct_team_series_pitching` | Team × Series | ERA, WHIP, K/9, BB/9 |

## Design Decisions

**Idempotent loads**: Partition truncation before each load job ensures reruns produce identical results. Load jobs used over streaming inserts for DML compatibility.

**Series identification**: The MLB API does not provide a series primary key. Series are inferred using a 7-day window function partitioned by home/away team pair, with `MIN(game_date)` as the series start date.

**Rate stat recomputation**: AVG, OBP, SLG, ERA, WHIP are never averaged across games. All rate stats are recomputed from aggregated counting stats to ensure mathematical correctness.

**Player name canonicalization**: The MLB API returns inconsistent name encoding for some players. Player names are canonicalized in the mart layer by selecting the most frequent variant per `player_id`.

**Schema-on-write**: All BigQuery tables use explicit schemas defined at table creation time. Auto-detect is not used.

## Local Development

```bash
# Install dependencies
uv sync

# Set environment variables
cp .env.example .env
# Edit .env with your GCP credentials

# Run extraction pipeline
python3 -m extraction.pipeline

# Run dbt
cd mlb_dbt
dbt deps
dbt seed
dbt build
```

## Pipeline Schedule

Runs daily at 9AM ET via GitHub Actions cron. Processes previous day's games.

## dbt Documentation

```bash
cd mlb_dbt
dbt docs generate
dbt docs serve
```
