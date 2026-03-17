# Cashflow Data Lake Ingestion Pipeline (Microsoft Fabric + PySpark)

This project implements a scalable data ingestion pipeline to centralize daily cashflow and market data for a client-facing analytics team.

The solution leverages Microsoft Fabric and PySpark to ingest semi-structured Excel data into a data lake, while maintaining both current state and historical datasets for ongoing analysis.

## Business Problem

The analytics team received daily Excel-based data including:

- Cashflow events
- Current positions
- Historical positions
- Market data (e.g., coupon rates, currency rates)

Challenges:

- No centralized storage for daily data
- Manual file handling and version tracking
- No reliable way to maintain current vs historical views
- Data updated daily → required scalable ingestion

## Solution

Designed and implemented a data lake ingestion framework using Microsoft Fabric and PySpark that:
- Ingests daily Excel/CSV files into a centralized data lake
- Uses PySpark for scalable data processing
- Preserves raw data structure (schema-on-read)
- Maintains both:
   - Current datasets (latest state)
   - Historical datasets (append-only)
- Automates incremental updates using Python

## Architecture 
### Architecture Overview
        ┌──────────────────────────────┐
        │   Daily Excel / CSV Files    │
        │------------------------------│
        │ Cashflows                    │
        │ Positions (Current/Historical)│
        │ Currency Rates               │
        └──────────────┬───────────────┘
                       ↓
        ┌──────────────────────────────┐
        │   PySpark Ingestion Layer    │
        │------------------------------│
        │ • Read Excel / CSV           │
        │ • Schema alignment           │
        │ • ID standardization         │
        │ • Light cleaning             │
        └──────────────┬───────────────┘
                       ↓
        ┌──────────────────────────────┐
        │     Data Lake (OneLake)      │
        │------------------------------│
        │ Raw / Delta Tables           │
        └──────────────┬───────────────┘
                       ↓
        ┌──────────────────────────────┐
        │   Python Processing Layer    │
        │------------------------------│
        │ • Incremental updates        │
        │ • Current vs Historical logic│
        └──────────────┬───────────────┘
                       ↓
        ┌──────────────────────────────┐
        │   Analytics-Ready Tables     │
        │------------------------------│
        │ Current Positions            │
        │ Historical Cashflows         │
        └──────────────────────────────┘

### Data Flow
            Excel / CSV Sources
                   │
                   ▼
         ┌─────────────────────┐
         │ PySpark Ingestion   │
         │---------------------│
         │ Read files          │
         │ Standardize schema  │
         │ Generate IDs        │
         └─────────┬───────────┘
                   ▼
         ┌─────────────────────┐
         │ Raw Delta Tables    │
         └─────────┬───────────┘
                   ▼
         ┌─────────────────────┐
         │ Processing Layer    │
         │---------------------│
         │ Join datasets       │
         │ Deduplicate         │
         │ Apply business logic│
         └─────────┬───────────┘
                   ▼
         ┌─────────────────────┐
         │ Output Tables       │
         │---------------------│
         │ Current State       │
         │ Historical Records  │
         └─────────────────────┘

## Key Engineering Decisions

**Schema-on-read (Data Lake Design)**

Preserved original Excel structures to align with business needs while enabling scalable ingestion.

**Separation of Current vs Historical Data**

Designed pipelines to maintain both:
   - Latest “current” position view
   - Append-only historical datasets for auditability

**Incremental Processing**

Built Python logic to update datasets daily without full reloads.

**Lightweight Transformation Layer**

Applied minimal transformations during ingestion (e.g., ID standardization, deduplication) while preserving raw fidelity.

## Script Examples 
### Example Snippet 1
```
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# 1. Read daily source files into Spark DataFrames
df_positions = (
    spark.read
         .format("csv")
         .option("header", True)
         .option("inferSchema", True)
         .load("Files/project_folder/current_positions.csv")
)

df_rates = (
    spark.read
         .format("csv")
         .option("header", True)
         .option("inferSchema", True)
         .load("Files/project_folder/daily_rates.csv")
)

# 2. Standardize a reusable business key
df_positions = df_positions.withColumn(
    "position_key",
    F.concat_ws(" | ", F.col("instrument_name"), F.col("portfolio_name"))
)

# 3. Deduplicate daily market/rate data
rate_window = Window.partitionBy("instrument_name").orderBy(F.col("rate_value").desc())

df_rates_latest = (
    df_rates
    .withColumn("rn", F.row_number().over(rate_window))
    .filter(F.col("rn") == 1)
    .drop("rn")
)

# 4. Join related datasets into a processed analytical table
df_enriched = (
    df_positions.alias("p")
    .join(
        df_rates_latest.alias("r"),
        on="instrument_name",
        how="left"
    )
    .select(
        "position_key",
        "instrument_name",
        "portfolio_name",
        "as_of_date",
        "rate_type",
        "rate_value"
    )
)

# 5. Write processed result to a Delta table
(
    df_enriched.write
    .format("delta")
    .mode("overwrite")
    .saveAsTable("lakehouse.processed_positions")
)
```

### Example Snippet 2
```
from pyspark.sql import functions as F

# Daily incoming dataset
df_daily = (
    spark.read
         .format("csv")
         .option("header", True)
         .option("inferSchema", True)
         .load("Files/project_folder/daily_events.csv")
         .withColumn("load_date", F.current_date())
)

# Append to historical table
(
    df_daily.write
    .format("delta")
    .mode("append")
    .saveAsTable("lakehouse.historical_events")
)

# Refresh current-state table
df_current = (
    df_daily
    .groupBy("business_key")
    .agg(F.max("event_date").alias("latest_event_date"))
)

(
    df_current.write
    .format("delta")
    .mode("overwrite")
    .saveAsTable("lakehouse.current_events")
)
```
