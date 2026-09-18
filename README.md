# valuation_lakehouse

A Databricks [Declarative Automation Bundle](https://docs.databricks.com/dev-tools/bundles/index.html)
for the valuation lakehouse. It contains a declarative ETL pipeline and a scheduled job, and it deploys to
Azure Databricks through the Databricks CLI.

> **Status: template scaffold.** The pipeline, job, shared library, and tests are still the samples from
> the `default-python` template. They read the NYC taxi sample dataset (`samples.nyctaxi.trips`), not
> valuation data. See [PROJECT_OVERVIEW.md](PROJECT_OVERVIEW.md) for the architecture, current state, and
> known gotchas.

## Repository layout

| Path | Contents |
|---|---|
| `databricks.yml` | Bundle root: variables and the `dev` / `prod` targets |
| `resources/` | One YAML file per deployed resource: the `valuation_lakehouse_etl` pipeline and `sample_job` |
| `src/valuation_lakehouse/` | Shared Python package used by jobs and pipelines |
| `src/valuation_lakehouse_etl/` | Pipeline source, with one dataset per file under `transformations/` |
| `src/sample_notebook.ipynb` | Notebook run by the job's first task |
| `tests/` | pytest suite, which runs on Databricks compute (see [Testing](#testing)) |
| `fixtures/` | Test data files for the `load_fixture` pytest fixture |
| `.github/workflows/deploy.yml` | CI/CD (see [CI/CD](#cicd)) |

## Prerequisites

- [Databricks CLI](https://docs.databricks.com/dev-tools/cli/install)
- [uv](https://docs.astral.sh/uv/getting-started/installation/)
- Python 3.12. If you don't have it, run `uv python install 3.12`.

For VS Code or Cursor, install the extensions recommended in `.vscode/extensions.json` (Databricks, Ruff, YAML).

## Setup

Install the dev tools (pytest, ruff, Databricks Connect, `databricks-dlt`, ipykernel) into `.venv`:

```bash
uv sync
```

Log in to the workspace. The workspace URL is the `workspace.host` value in `databricks.yml`.

```bash
databricks auth login --host <workspace-url> --profile <profile-name>
```

Pass `--profile <profile-name>` to every bundle command below. Otherwise the CLI uses whichever profile is
your default.

## Develop and deploy (dev target)

`dev` is the default target and runs in development mode, so a deploy:

- adds the prefix `[dev <your-username>]` to every deployed job and pipeline name
- pauses the job's daily trigger
- writes tables to catalog `valuation`, in a schema named after your username

```bash
# Check the bundle configuration
databricks bundle validate --profile <profile-name>

# Deploy your personal dev copy
databricks bundle deploy --profile <profile-name>

# Run a pipeline update
databricks bundle run valuation_lakehouse_etl --profile <profile-name>

# Refresh a single table
databricks bundle run valuation_lakehouse_etl --refresh sample_trips_valuation_lakehouse --profile <profile-name>

# Run the job: the notebook first, then a pipeline refresh
databricks bundle run sample_job --profile <profile-name>
```

To add a table to the pipeline, create a new file under `src/valuation_lakehouse_etl/transformations/`.
The pipeline picks up every file in that folder.

## Production (prod target)

The `prod` target writes to `valuation.prod`, deploys under `/Shared/.bundle/valuation_lakehouse/prod`,
and keeps the job's daily trigger active. CI deploys it on every merge to `main`. To deploy it by hand:

```bash
databricks bundle deploy --target prod --profile <profile-name>
```

## CI/CD

`.github/workflows/deploy.yml` runs on GitHub Actions:

| Trigger | Jobs |
|---|---|
| Pull request into `main` | **validate**: `uv sync`, `uv run ruff check .`, `databricks bundle validate -t dev` |
| Push to `main` | **validate**, then **deploy**: `databricks bundle deploy -t prod` |

CI signs in to Databricks as a service principal. The repository needs two secrets:
`DATABRICKS_CLIENT_ID` and `DATABRICKS_CLIENT_SECRET`.

## Lint and format

```bash
uv run ruff check .
uv run ruff format .
```

CI fails on `ruff check` errors, so run it before you push. The ruff config in `pyproject.toml` sets a line
length of 120 and treats the Databricks notebook globals (`spark`, `dbutils`, `display`, `sc`, `sqlContext`)
as builtins.

## Testing

```bash
DATABRICKS_CONFIG_PROFILE=<profile-name> uv run pytest
```

The tests do not run offline. `tests/conftest.py` opens a Databricks Connect session. If you haven't
configured a cluster (`DATABRICKS_CLUSTER_ID`) or serverless compute, it falls back to serverless compute,
which is billed.

## Dependencies

- **Pipeline runtime dependencies:** add them to the `environment` section of
  `resources/valuation_lakehouse_etl.pipeline.yml`, not to `dependencies` in `pyproject.toml`. Pipelines
  cache dependencies during development, so changes to `pyproject.toml` may not reach them.
- **Local dev tools:** `uv add --dev <package>`.
- **Version constraints:** don't edit the `[tool.uv]` block in `pyproject.toml` by hand.
  `databricks environments setup-local` generates it to match serverless environment version 5. To
  regenerate it, run:

  ```bash
  databricks environments setup-local --serverless-version 5
  ```
test
