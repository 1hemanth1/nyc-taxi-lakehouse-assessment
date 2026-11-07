# nyc-taxi-lakehouse-assessment

End-to-end cloud data pipeline built on Azure Data Factory, ADLS Gen2, and Azure Databricks (Unity Catalog) to ingest, validate, and aggregate NYC TLC Yellow Taxi trip data (`2024-10` through `2025-01`).


## Repository Structure
```text
nyc-taxi-lakehouse-assessment/
├── notebooks/
│   ├── 01_medallion_pipeline.ipynb
├── screenshots/
│   ├── 01_adf_pipeline_runs.png             # Initial, incremental, and re-run monitoring evidence
│   ├── 02_adls_raw_storage.png              # Hive-partitioned raw zone layout
│   ├── 03_reconciliation_and_idempotency.png# Row counts per layer per month
│   ├── 04_quarantine_dq_rules.png           # Quarantine rule failure breakdown
│   ├── 05_gold_business_metrics.png         # Gold daily borough revenue output
│   └── 06_unity_catalog_lineage.png         # Unity Catalog lineage graph
└── README.md

Architecture & Data Flow
Ingestion (ADF -> ADLS Gen2): Parameterized pipeline (pl_ingest_tlc_data) loads monthly Parquet files and lookup CSVs into Hive partitions (raw/tlc/yellow/year=YYYY/month=MM/) using Managed Identity.

Bronze Layer: Ingests raw files via COPY INTO with schema evolution (mergeSchema = true) and timestampNtz support.

Silver & Quarantine Layers: Cleans data, performs broadcast joins against taxi_zone_lookup, and routes records based on 5 data quality rules. Clean data goes to Silver; failing data goes to Quarantine.

Gold Layer: Produces three aggregated business reporting tables (daily_revenue_by_borough, hourly_demand_patterns, top_od_zone_pairs).

Design Decisions, Trade-Offs & Assumptions
Managed Identity over Keys: Used system-assigned Managed Identity with RBAC (Storage Blob Data Contributor) between ADF and ADLS Gen2 to eliminate hardcoded storage account keys.

Delta Lake Partition Overwrites: Utilized .option("replaceWhere", "_year_month = 'YYYY-MM'") for Silver and Quarantine writes instead of full table overwrites, minimizing compute cost and ensuring deterministic partition-level idempotency.

Broadcast Joins: Applied broadcast hints (F.broadcast) to the taxi_zone_lookup reference table to eliminate shuffling overhead across millions of trip records.

Assumptions: Assumed single-region deployment (Canada Central / East US) on Azure Free Tier with spending limit enabled; assumed TLC schema additions (cbd_congestion_fee in Jan 2025) are safely handled via Delta schema evolution.

Known Limitations & Production Enhancements
Current Limitation: Ingestion runs sequentially via ADF copy activities rather than true parallel event-driven streaming.

Production Changes:

Replace scheduled ADF triggers with Event Grid triggers firing upon blob creation in ADLS Gen2.

Implement automated CI/CD deployment via GitHub Actions and ARM/Bicep templates.

Introduce Azure Monitor alerts and Databricks job run notifications for failed pipeline executions.

Approximate Cost Incurred
Total Cost: ~$0.00 (Operated entirely within the Azure Free Trial limits and Databricks Community / Free Tier capacity during development and testing).

AI Usage Statement
AI coding assistants were used to scaffold boilerplate PySpark transformation loops and format documentation. All architecture patterns, partition override logic, data quality rules, and pipeline configurations were reviewed, tested, and validated by the author.



