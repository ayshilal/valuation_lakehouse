# Project Overview — valuation_lakehouse

> **Living document.** This file describes what the project *is* and what state it is *currently in*.
> It is maintained by AI agents and contributors as work progresses — see
> [Maintaining this file](#maintaining-this-file) at the bottom.
>
> `README.md` covers setup and commands. This file covers architecture, state, and gotchas.

**Last updated:** 2026-09-17

---

## Current state

**Scaffold only — no valuation logic has been written yet.**

The project was generated from the Databricks `default-python` bundle template and the sample
code is still in place. Every transformation, notebook, and test targets the NYC taxi sample
dataset (`samples.nyctaxi.trips`), not any valuation data. The name is aspirational; the frame
is empty.

What *has* been customized away from the template default:

- Azure workspace host wired into both targets
- `catalog` variable set to `valuation`
- Tags applied via presets (`project: valuation`, `environment`, `owner: hilal`)
- Prod target given an explicit `root_path` and a `CAN_MANAGE` permission grant

Git history is a single commit (`d62eb0a Initial commit: Databricks asset bundle`) plus
uncommitted local tweaks.

---

## What this is

A **Databricks Asset Bundle (DAB)** — a declarative, version-controlled definition of Databricks
resources (jobs, pipelines) that deploys via the Databricks CLI. The bundle root is
`databricks.yml`; resource definitions live in `resources/` and are pulled in by `include:`.

---

## Repository layout

| Path | Role |
|---|---|
| `databricks.yml` | Bundle root — name, UUID, variables, dev/prod targets |
| `resources/` | Resource definitions, one YAML per resource |
| `resources/sample_job.job.yml` | The scheduled job |
| `resources/valuation_lakehouse_etl.pipeline.yml` | The declarative ETL pipeline |
| `src/valuation_lakehouse/` | Shared Python library, importable by jobs and pipelines |
| `src/valuation_lakehouse_etl/` | Pipeline source tree (transformations + explorations) |
| `src/sample_notebook.ipynb` | Notebook executed as the job's first task |
| `tests/` | pytest suite — runs against **live** Databricks compute |
| `fixtures/` | Test fixture data (currently empty, `.gitkeep` only) |

---

## Deployable resources

### Pipeline: `valuation_lakehouse_etl`

Defined in `resources/valuation_lakehouse_etl.pipeline.yml`. Serverless, rooted at
`src/valuation_lakehouse_etl`, and it picks up **every** file under `transformations/` via a glob:

```yaml
libraries:
  - glob:
      include: ../src/valuation_lakehouse_etl/transformations/**
```

Dependencies are installed by pointing an editable install at the deployed workspace path
(`--editable ${workspace.file_path}`), which pulls in whatever `pyproject.toml` declares.

**Current transformations** (both still template samples, and they chain):

1. `sample_trips_valuation_lakehouse` — reads `samples.nyctaxi.trips` into a table
2. `sample_zones_valuation_lakehouse` — reads table 1 back, groups by `pickup_zip`, sums `fare_amount`

Convention, per `src/valuation_lakehouse_etl/README.md`: **one dataset per file** under
`transformations/`.

### Job: `sample_job`

Defined in `resources/sample_job.job.yml`. Runs on a periodic trigger (every 1 day) with two
tasks in sequence:

1. `notebook_task` → runs `src/sample_notebook.ipynb`
2. `refresh_pipeline` → depends on task 1, refreshes the pipeline above

It passes `catalog` and `schema` through as job parameters, which the notebook reads via
`dbutils.widgets.get(...)`. Email-on-failure notifications are present but commented out.

---

## Targets and variables

Two variables, `catalog` and `schema`, declared in `databricks.yml` and bound per target:

| | dev (default) | prod |
|---|---|---|
| mode | `development` | `production` |
| catalog | `valuation` | `valuation` |
| schema | `${workspace.current_user.short_name}` | `prod` |
| host | same Azure workspace | same Azure workspace |
| schedule | **paused** (dev mode) | active |

Dev gives each developer a personal schema in the shared `valuation` catalog, so deployments
don't collide. `mode: development` also pauses the job's daily trigger automatically.

Deploy with `databricks bundle deploy --target dev|prod`. Always pass `--profile <name>` and let
the user choose the profile.

---

## Local development

- Dependencies are managed with **uv**: `uv sync --dev`
- Python is pinned to `>=3.12,<3.13`
- Runtime dependencies in `pyproject.toml` are intentionally **empty** — the file warns that
  pipeline dependencies are cached during development and must instead go in the `environment`
  section of the pipeline YAML
- Lint: `ruff`, line length 120, with Spark globals (`spark`, `dbutils`, `display`, `sc`,
  `sqlContext`) declared as builtins so they don't flag as undefined

### Testing

`uv run pytest`. **These tests are not offline.** `tests/conftest.py` builds a real
`DatabricksSession` via Databricks Connect, and if no compute is configured it silently falls
back to serverless and bills for it:

> ☁️ no compute specified, falling back to serverless compute

A `load_fixture` fixture is available for loading JSON/CSV from `fixtures/`, but nothing uses it
yet — the only test (`tests/sample_taxis_test.py`) counts rows in the live taxi sample table.

---

## Known issues and gotchas

| Issue | Where | Notes |
|---|---|---|
| Hardcoded schema | `src/valuation_lakehouse_etl/explorations/sample_exploration.ipynb` | Queries `valuation.ahyalciner_us.sample_trips_...` — one developer's dev schema baked in. Breaks for any other user or against prod. |
| Tests hit live compute | `tests/conftest.py`, `tests/sample_taxis_test.py` | `pytest` spins up serverless compute and incurs cost. Not runnable offline or in plain CI without credentials. |
| f-string with no placeholders | `sample_zones_valuation_lakehouse.py` | `spark.read.table(f"sample_trips_...")` — harmless, but ruff flags it |
| Unused imports | both transformation files | `col` / `sum` imported but partly unused |
| Prod root_path is a user home dir | `databricks.yml` | Prod artifacts deploy under an individual's `/Workspace/Users/...` path. Fine solo; revisit before this is a team or production-critical bundle. |
| Empty `fixtures/` | `fixtures/` | Fixture loading is wired up but unused — the path to make tests runnable offline |

---

## Maintaining this file

**Agents and contributors: update this file as part of the same change that makes it stale.**

Update it when you:

- add, remove, or rename a transformation, job, pipeline, or other bundle resource
- change targets, variables, catalogs, schemas, or deployment configuration
- add or drop a dependency, or change the Python/tooling setup
- fix something listed under [Known issues and gotchas](#known-issues-and-gotchas) — remove the row
- discover a new gotcha worth warning the next person about — add a row
- replace template sample code with real logic (this is the big one: the
  [Current state](#current-state) section stops being true the moment that happens)

Bump **Last updated** when you edit. Keep it factual and current — describe what the repo *is*
now, not a history of what it was. Do not let it drift into a changelog.
