# UFC Events — Medallion Architecture

A Bronze → Silver → Gold data pipeline built in **Databricks** that ingests the [UFC Complete Dataset (all events 1996–2024)](https://www.kaggle.com/datasets/maksbasher/ufc-complete-dataset-all-events-1996-2024) from Kaggle, cleans and standardizes it, and produces an analytics-ready star schema.

## Architecture Overview

```
Kaggle Dataset (CSV)
        │
        ▼
┌─────────────────────────────────┐
│  BRONZE (workspace.ufc_bronze)   │  Raw ingest with metadata
│  fighter_stats                   │  (_source_file, _ingestion_timestamp)
│  medium_dataset                   │
│  large_dataset                   │
│  completed_events_small           │
│  upcoming_events_small           │
└──────────────┬──────────────────┘
               │ deduplicate · trim · parse dates · split location
               ▼
┌─────────────────────────────────┐
│  SILVER (ufc_silver)             │  Cleaned & standardized
│  events                          │  (date parsing, city/state/country split)
│  fights                          │  (winner resolution, finish categories)
│  fighters                        │  (BMI, win_rate, weight_class)
│  fights_medium                   │  (fight duration in seconds)
└──────────────┬──────────────────┘
               │ enrich · build star schema
               ▼
┌─────────────────────────────────┐
│  GOLD (ufc_gold)                 │  Analytics-ready star schema
│  fact_fights                     │  (fights + event details join)
│  dim_fighter                     │  (fighter stats with surrogate key)
│  dim_event                       │  (event details with surrogate key)
│  dim_time                        │  (date dimension: year, quarter, weekday)
└─────────────────────────────────┘
```

## Data Source

| File | Description | Rows (approx.) |
| --- | --- | --- |
| `fighter_stats.csv` | Fighter physical stats and career records | ~4,000 |
| `large_dataset.csv` | Detailed fight-by-fight data (95 columns) | ~7,000 |
| `medium_dataset.csv` | Alternative fight stats with event info | ~7,000 |
| `completed_events_small.csv` | Completed UFC event listings | ~700 |
| `upcoming_events_small.csv` | Upcoming/scheduled UFC events | ~50 |

## Pipeline Stages

### Bronze Layer

* Loads raw CSV files from Kaggle via `kagglehub` (driver-local download)
* Converts to Spark DataFrames using pandas
* Adds `_source_file` and `_ingestion_timestamp` metadata columns
* Writes as Delta tables with `overwrite` mode for idempotency

### Silver Layer

* **events** — Parsed dates (`yyyy-MM-dd`), split `location` into `city` / `state` / `country`, added `event_year`, `event_month`, `event_quarter`
* **fights** — Resolved `winner` ("Red"/"Blue") to fighter name, categorized `method` into `finish_category` (KO/TKO, Submission, Decision, DQ)
* **fighters** — Added calculated columns: `bmi`, `win_rate`, `weight_class` (Flyweight → Heavyweight based on UFC weight limits)
* **fights_medium** — Parsed dates, split location, calculated `fight_duration_seconds` from `time` field

### Gold Layer

Star schema for analytics:

* **fact_fights** — Enriched fight records joined with event details (date, location, city, country)
* **dim_fighter** — Fighter dimension with surrogate key (`fighter_key`)
* **dim_event** — Event dimension with surrogate key (`event_key`)
* **dim_time** — Date dimension with `date_key`, year, month, quarter, day-of-week, weekend flag

## Getting Started

### Prerequisites

* Databricks workspace (AWS, Azure, or GCP)
* Python 3 with `kagglehub` installed
* Spark compute (Serverless or interactive cluster)

## Repository Structure

The pipeline is split into one notebook per medallion layer:

```
01_bronze_ingestion.ipynb   # Kaggle download -> raw Delta tables (workspace.ufc_bronze)
02_silver_layer.ipynb      # Bronze -> cleaned & standardized tables (ufc_silver)
03_gold_layer.ipynb        # Silver -> star schema (ufc_gold) + validation summary
```

### Running the Pipeline

1. Clone this repo into a Databricks Git folder (or import the three notebooks into your workspace)
2. Run them in order: **01_bronze_ingestion** → **02_silver_layer** → **03_gold_layer** — each layer depends on the previous layer's tables
3. The pipeline creates three schemas: `workspace.ufc_bronze`, `ufc_silver`, `ufc_gold`

> **Note:** The bronze notebook's `pip install kagglehub` and `kagglehub.dataset_download` cells require internet access on the driver node.

## Key Transformations

| Transformation | Input | Output |
| --- | --- | --- |
| Date parsing | `"March 15, 2024"` / `"3/15/2024"` | `date` (DATE type) |
| Location split | `"Las Vegas, NV, USA"` | `city`, `state`, `country` |
| Winner resolution | `winner = "Red"` | `winner_name = "Jon Jones"` |
| Finish categorization | `method = "KO/TKO Punches"` | `finish_category = "KO/TKO"` |
| BMI calculation | `height_cm`, `weight_kg` | `bmi = weight / (height/100)²` |
| Win rate | `wins`, `losses` | `win_rate = wins / (wins + losses)` |
| Weight class | `weight_kg = 93.4` | `weight_class = "Heavyweight"` |
| Fight duration | `time = "4:30"` | `fight_duration_seconds = 270` |

## Technologies

* **Databricks** — Compute and Delta Lake storage
* **PySpark** — Distributed data processing
* **Delta Lake** — ACID transactions for all tables
* **kagglehub** — Dataset download from Kaggle
* **pandas** — Local CSV reading before Spark conversion

## License

This project uses the [UFC Complete Dataset](https://www.kaggle.com/datasets/maksbasher/ufc-complete-dataset-all-events-1996-2024) from Kaggle. Please check the dataset's license terms for usage restrictions.