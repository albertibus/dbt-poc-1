# Job Postings Analytics with dbt and DuckDB

An analytics engineering proof of concept that transforms a Parquet dataset of job postings using **dbt** and **DuckDB**. The project provides insights into monthly posting trends, the most common locations, and the share of postings allowing remote work.

## Requirements

- Python 3.14 or later.
- [uv](https://docs.astral.sh/uv/getting-started/installation/) to manage the environment and dependencies.
- An internet connection to install dependencies and download the data.

## Getting started

Run the following commands from the repository root:

```bash
# Install dependencies in the project environment
uv sync

# Download the Parquet files into data/raw/
uv run python scripts/download_data.py

# Enter the dbt project
cd analytics

# Check the connection and build the models and their tests
uv run dbt debug --profiles-dir ../ci
uv run dbt build --profiles-dir ../ci
```

The script downloads approximately 85 MB: twelve monthly job posting files, from July 2025 through June 2026, and a skills file. Existing files are skipped. The data comes from [Luke Barousse's course dataset](https://github.com/lukebarousse/dbt_Analytics_Engineering_Course/releases/tag/dataset-v1).

The profile in `ci/profiles.yml` configures DuckDB and creates the database at `data/prod.duckdb`. Run dbt commands from `analytics/`: both the database path and the source file paths are relative to that directory.

## Models

```text
raw.job_postings (Parquet)
└── job_postings (view)
    ├── jobs_per_month (table)
    │   └── monthly_summary (table)
    ├── monthly_summary (table)
    └── top_locations (table)
```

| Model | Description |
| --- | --- |
| `job_postings` | Selects fields from the postings and excludes rows with errors. |
| `jobs_per_month` | Counts postings by month and search term. |
| `top_locations` | Shows the 10 locations with the most postings. |
| `monthly_summary` | Calculates monthly totals, remote postings, and their percentage. Only includes months with at least one remote posting. |

Models are materialized as tables by default; `job_postings` is a view that continues to read the original Parquet files. Keep the files in `data/raw/` and access the database from `analytics/` if you need to query that view.

The defined tests check that `job_postings.job_title` is not null and that `monthly_summary.month` is unique and not null. To run them separately:

```bash
# From analytics/
uv run dbt test --profiles-dir ../ci
```

## Continuous integration

The GitHub Actions workflow runs on every push to `main`, on Mondays, and manually. It installs dependencies, downloads the data, runs `dbt build`, and publishes `data/prod.duckdb` to the `warehouse` release. The release contains the database, not the Parquet files: the `job_postings` view requires those files to be available at the configured relative path.
