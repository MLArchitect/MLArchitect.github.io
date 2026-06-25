---
layout: page
title: ""
subtitle: "Real-time retail intelligence: turning raw transactions into actionable insights"
cover-img: 
thumbnail-img: 
share-img: 
tags: [Databricks, Delta Live Tables, PySpark, Unity Catalog, Streaming, Azure, Data Engineering]
---

<h4>Databricks Apparel Streaming Pipeline</h4>

<p align='justify'>
Retail businesses lose revenue when they can't answer basic questions in time: <em>Which store is underperforming today? Which products are running out? Which customers haven't returned?</em> The data exists — but it sits in raw transaction files, unprocessed and unusable.
</p>

<p align='justify'>
This project solves that problem by building a real-time streaming pipeline that continuously ingests sales, customer, product, and store data and delivers clean, analytics-ready outputs within minutes of each transaction. Business teams get always-fresh dashboards. Data quality issues are caught automatically before they reach reporting. Customer and product history is preserved across changes — so no analytical question is left unanswerable.
</p>

***

## Pipeline Architecture

<p align="center">
<img src="https://raw.githubusercontent.com/MLArchitect/databricks-apparel-streaming/main/DLT.png" width="900" alt="DLT Pipeline Graph">
</p>

The DLT pipeline graph above shows all tables completing successfully (green checkmarks). Each stream flows independently through Bronze → Silver → Gold before joining into the denormalized sales facts table.

All four schemas live inside the **`apparel_store`** Unity Catalog:

| Schema | Purpose |
|---|---|
| `00_landing` | Raw streaming files in a Unity Catalog Volume |
| `01_bronze` | Raw Delta streaming tables with audit metadata |
| `02_silver` | Cleaned, typed, and historically tracked tables |
| `03_gold` | Business-ready aggregations and materialized views |

***

## Step 1 — Landing Zone: Synthetic Data Generation

<p align='justify'>
A synthetic data generator simulates a real apparel retail operation, continuously producing records for sales transactions, customers, products, and stores. Files land in a <strong>Unity Catalog Volume</strong> under <code>apparel_store.00_landing.streaming</code>, organized by entity:
</p>

```
/Volumes/apparel_store/00_landing/streaming/
├── sales/
├── customers/
├── items/
└── stores/
```

This approach lets the pipeline run and be tested without needing a real production data source — the generator can be started and stopped at any time to observe how each layer reacts to incoming data in real time.

***

## Step 2 — Bronze Layer: Streaming Ingestion with DLT

<p align='justify'>
The Bronze layer ingests raw files from the landing zone as <strong>streaming Delta tables</strong> using Databricks Auto Loader. Each table adds two audit columns — <code>ingest_timestamp</code> and <code>source_file_path</code> — to support lineage and traceability. All columns are cast to their correct types at this stage.
</p>

Critically, each Bronze table enforces a **hard data quality rule** using DLT expectations: if a primary key column (`transaction_id`, `customer_id`, `product_id`, `store_id`) is NULL, the pipeline fails immediately rather than passing invalid records downstream.

```python
@dlt.table(name=BRONZE_SALES, comment="Raw sales data")
@dlt.expect_or_fail("valid_transaction_id", "transaction_id is not null")
def bronze_sales():
    return (
        spark.readStream.format("delta").load(RAW_SALES_PATH)
        .withColumn("ingest_timestamp", F.lit(datetime.now(timezone.utc)))
        .withColumn("source_file_path", F.col("_metadata.file_path"))
        .withColumn("transaction_id", F.col("transaction_id").cast("int"))
        .withColumn("event_time",      F.col("event_time").cast("timestamp"))
        .withColumn("unit_price",      F.col("unit_price").cast("double"))
        .withColumn("total_amount",    F.col("total_amount").cast("double"))
        # ... additional columns
    )
```

Four Bronze tables are produced:
- `01_bronze.bronze_sales`
- `01_bronze.bronze_customers`
- `01_bronze.bronze_products`
- `01_bronze.bronze_stores`

**Why `expect_or_fail` instead of `expect_or_drop`?**  
A missing primary key means the record cannot be linked to any other entity. Dropping it silently would create invisible gaps in analytics. Failing the pipeline forces the issue to be investigated and fixed at the source.

***

## Step 3 — Silver Layer: Cleaning, SCD Type 2, and Data Quality

<p align='justify'>
The Silver layer transforms raw Bronze streams into clean, typed, business-meaningful tables. It is split into four files (02A through 02D), one per entity, each responsible for its own cleaning logic and quality rules.
</p>

### Streaming Views and Cleaned Streams

Each entity first passes through a **cleaning view** that applies business rules — for example, standardizing email formats, filtering out test records, and validating value ranges. These views act as a staging step before writing to persistent Silver tables.

### SCD Type 2 for Customer and Product Dimensions

<p align='justify'>
Customer and product records change over time — a customer updates their address, a product changes its price. Rather than overwriting the old record, <strong>SCD Type 2</strong> preserves the full history:
</p>

- When a record changes → the existing row is **closed** with an `end_date`
- A new row is **inserted** with the updated values and a new `start_date`
- The current active record always has `end_date = NULL`

This allows the pipeline to answer historical questions: *"What was this customer's address when they placed this order in March?"*

### Silver Tables Produced

| Table | Description |
|---|---|
| `silver_customers` | Full SCD Type 2 customer history |
| `silver_customers_current` | Current active customer records only |
| `silver_products` | Full SCD Type 2 product history |
| `silver_products_current` | Current active product records only |
| `silver_sales_transactions` | Cleaned and validated sales records |
| `silver_returns_transactions` | Separated returns from sales stream |
| `silver_stores` | Cleaned store dimension |
| `silver_stores_current` | Current active stores only |

***

## Step 4 — Gold Layer: Aggregations and Materialized Views

<p align='justify'>
The Gold layer is the analytics-facing output of the pipeline. It joins Silver tables and produces four business-ready outputs as <strong>materialized views</strong> in DLT — automatically refreshed whenever upstream Silver data changes.
</p>

| Table | Business Question Answered |
|---|---|
| `denormalized_sales_facts` | Single joined fact table across all dimensions |
| `gold_daily_sales_by_store` | How much revenue did each store generate per day? |
| `gold_customer_lifetime_value` | What is each customer's total spend and order history? |
| `gold_product_performance` | Which products sell most, and at what margins? |

***

## Unity Catalog and Governance

<p align='justify'>
All tables, schemas, and volumes are registered in <strong>Databricks Unity Catalog</strong> under the <code>apparel_store</code> catalog. This provides:
</p>

- **Centralized access control** — permissions managed at catalog, schema, or table level
- **Data lineage** — Unity Catalog automatically tracks how data flows between tables
- **Audit logs** — every read and write is logged for compliance

***

## Key Engineering Decisions

| Decision | Why |
|---|---|
| `expect_or_fail` on primary keys | Missing PKs break all downstream joins — fail fast rather than silently corrupt data |
| SCD Type 2 for customers and products | Retail analytics often need historical snapshots — overwriting loses that ability |
| Separate `_current` views | Most queries only need current state; dedicated views avoid filtering on every query |
| Materialized views in Gold | Automatically stay in sync with Silver without manual refresh jobs |
| Unity Catalog for all assets | Single governance layer for access control, lineage, and audit across all schemas |
| `variables.py` for all names | One place to change catalog/schema names — no hardcoded strings scattered across notebooks |

***

## Tech Stack

**Azure Databricks · Delta Live Tables · PySpark · Auto Loader · Delta Lake · Unity Catalog · Azure Data Lake Storage Gen2**

***

<p align='justify'>
<a href="https://github.com/MLArchitect/databricks-apparel-streaming" target="_blank">View the code on GitHub →</a>
</p>
