# mind-food-move

Data engineering pipeline (DuckDB → dbt → Airflow) exploring how diet quality
relates to mental health and physical activity.

## Goal

Test (not "prove") the association between diet, mental health and sport, at two
levels of granularity:

- **Individual** — NHANES: 24h dietary recall → Healthy Eating Index, PHQ-9,
  accelerometer. Cycles 2005-06 and 2011-14.
- **Country-year** — FAOSTAT (food availability), WHO GHO (depression),
  Olympic medals, World Bank (GDP, population) as controls.

## Stack

Everything runs in containers, orchestrated by a single `docker-compose.yml`.
Nothing is installed on the host except Docker. `uv` is the package manager
inside the images (never `pip` directly).

- `ingestion` — Python 3.12, writes Parquet to `data/raw/`
- `dbt` — dbt-duckdb, run as a one-shot container
- `airflow` — webserver, scheduler, and its own metadata Postgres
- `streamlit` — dashboard, reads `data/gold/*.parquet`

`data/` is a bind mount shared by the services, so the DuckDB file and the
Parquet exports live on the host and survive container rebuilds.

## Scope

- Current stage: DuckDB only. Gold tables are exported to `data/gold/*.parquet`
  and Streamlit reads those, so nothing holds a lock on the warehouse file.
- Postgres as a serving layer and deployment to a Hetzner VM come later. Don't
  add them until I say so.
- Out of scope: AWS, Athena, S3. This project targets a self-hosted VM.

## Commands

Always run tools through Compose, never on the host.

- Start everything: `docker compose up -d`
- Full dbt build: `docker compose run --rm dbt build`
- Single model and its dependencies: `docker compose run --rm dbt build --select +model_name`
- dbt docs: `docker compose run --rm --service-ports dbt docs serve --host 0.0.0.0`
- Dashboard: `http://localhost:8501` (the `streamlit` service)
- Python tests: `docker compose run --rm ingestion pytest`
- Rebuild an image after changing dependencies: `docker compose build <service>`

## dbt conventions

- Layers: `stg_<source>__<entity>` → `int_<description>` → `dim_*` / `fct_*`
- Staging: renaming, casting and cleaning only. No joins, no business logic
- Lowercase SQL, import-style CTEs at the top of each file
- Every model has a `.yml` with a description and `unique` + `not_null` tests on its key
- Static lookups (PHQ-9 thresholds, country code mapping) → seeds in `dbt/seeds/`

## Data rules

- Never commit data: `data/` is in `.gitignore`
- Only one process may write to the DuckDB file at a time. Ingestion, dbt and
  the export step run sequentially; Streamlit opens an in-memory connection and
  reads the Parquet exports, never the warehouse file
- NHANES: analyses must use survey weights (e.g. `WTDRD1`), not simple averages
- Country codes: everything resolves to ISO3 via the `country_mapping` seed
- In results, always say correlation or association, never causation
- Cite the source and licence of every dataset in the README
- Keep SQL portable across DuckDB and Postgres: wrap engine-specific functions
  (date handling, string functions) in dbt macros rather than inlining them

## Working with me

- Write code, comments and docs in English
- Always behave like if i'm learning dbt and Airflow: briefly explain non-obvious choices
- Ask for help when you need it
- Go one step at a time, don't generate the whole project at once
- Ask before adding new dependencies
- For larger tasks, show me the plan before implementing