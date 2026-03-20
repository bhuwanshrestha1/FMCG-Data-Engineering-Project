# FMCG Data Engineering Project: Unified Medallion Pipeline

## Project Overview
This project simulates a real-world data engineering challenge where **Atlon**, a leading sports equipment manufacturer with mature ERP systems, acquires **Sports Bar**, a startup relying on unstructured data spread across spreadsheets, cloud drives, and various APIs. 

The goal was to build a reliable, scalable data layer using **Databricks** to unify analytics, resolve "data chaos," and support enterprise supply chain forecasting.

## Technical Stack
*   **Platform:** Databricks (Free/Community Edition)
*   **Storage:** AWS S3 (Object Store for landing child company data, bucket: `sportsbar-kasis11`)
*   **Languages:** PySpark (Python), Spark SQL
*   **Architecture:** Medallion Architecture (Bronze, Silver, Gold)
*   **Data Model:** Star Schema
*   **Orchestration:** Databricks Jobs
*   **Analytics:** Databricks Dashboards (Atlon BI 360) and Genie AI Assistant

## Architecture Strategy: Medallion Architecture
The pipeline ensures data quality from ingestion to consumption by executing through three refined layers:

1. **Bronze (Raw Layer)**
   - Ingests raw CSV data from AWS S3 (`sportsbar-kasis11`) for the child company, and handles one-time data imports for the parent company. 
   - Includes important metadata such as ingestion timestamps and source file names for auditing and lineage tracking.

2. **Silver (Cleaned Layer)**
   - Performs critical PySpark data transformations to create a "Single Source of Truth":
     - **Deduplication:** Removes duplicate records based on primary keys.
     - **Data Cleaning:** Trims whitespace, standardizes spelling errors (e.g., "protein"), and fixes typos in city names (e.g., standardizing "Bangalore" vs "Bangaluru").
     - **Feature Engineering:** Splits product variants, extracts years from dates, and generates surrogate keys using **SHA hashing** to create unique, unified product codes across the disparate Atlon and Sports Bar systems.

3. **Gold (Curated Layer)**
   - Aggregates the cleaned Silver data into a **Star Schema** optimized for business-level analysis.
   - Contains dimensions (`dim_customers`, `dim_products`, `dim_gross_price`, `dim_date`) and a fact table (`fact_orders`).
   - Utilizes Delta Lake **Upsert (MERGE INTO)** operations to ensure parent and child company data are seamlessly consolidated. 

## Implementation Details

### 1. Data Processing Strategy (Fact Orders)
*   **Historical Backfill (`1_full_load_fact.ipynb`):** Processed five months of historical order data using batch processing to establish the initial data lake fact table.
*   **Incremental Daily Load (`2_incremental_load_fact.ipynb`):** Implemented a robust S3 folder staging strategy strictly for the daily influx of new orders. New order data arriving in the S3 `landing/` folder is processed, automatically moved to a `processed/` folder within the S3 bucket to prevent double-ingestion, and then merged/upserted into the Gold `fact_orders` layer.

### 2. Orchestration
Automated the end-to-end pipeline using **Databricks Jobs**. Tasks are strictly sequenced to ensure dimension tables (Customers, Products, Prices) are processed and updated *before* the Fact table (Orders) is calculated. The job runs on a daily schedule (e.g., 11:00 PM) to provide fully updated insights for the next business morning.

### 3. Analytics & Insights
*   **Denormalized View (`denormalise_table_query_fmcg.txt`):** A comprehensive flat view (`vw_fact_orders_enriched`) joins the `fact_orders` table with all Dimensions to drastically optimize BI dashboard performance.
*   **Atlon BI 360 Dashboard:** A final Databricks dashboard visualizing key metrics such as **Total Revenue**, **Sold Quantity**, **Top Products**, and **Revenue Share by Channel**.
*   **Genie AI:** Deployed an AI-driven Databricks assistant allowing business stakeholders to query the underlying Star Schema using natural language (e.g., *"What are the top 5 products by revenue?"*).

---

## Repository Structure

```text
├── 0_data/                                  # Raw Source Data Examples
│   ├── 1_parent_company/                    # Atlon ERP exports
│   └── 2_child_company/                     # Sports Bar unstructured data
│
├── 1_codes/                                 # Databricks Notebooks (PySpark & SQL)
│   ├── 1_setup/
│   │   ├── dim_date_table_creation.ipynb    # Populates static dim_date
│   │   ├── setup_catalog.ipynb              # Catalog & schema initialization
│   │   └── utilities.ipynb                  # Helper/UDFs (e.g., SHA hashing)
│   │
│   ├── 2_dimension_data_processing/
│   │   ├── 1_customers_data_processing.ipynb # Deduplication & City standardizations
│   │   ├── 2_products_data_processing.ipynb  # Variant splitting & Hashing
│   │   └── 3_pricing_data_processing.ipynb
│   │
│   └── 3_fact_data_processing/
│       ├── 1_full_load_fact.ipynb           # Historical 5-month backfill
│       └── 2_incremental_load_fact.ipynb    # Daily Delta MERGE INTO (Upserts)
│
└── 2_dashboarding/                          # BI, Reporting & Genie AI Layer
    ├── denormalise_table_query_fmcg.txt     # SQL Script for Gold analytical view
    └── fmcg_dashboard.pdf                   # Exported Atlon BI 360 Dashboard
```

## Key Results
*   **Reliability:** Established a trusted, unified data layer successfully bridging two different enterprise architectures.
*   **Scalability:** The architecture seamlessly supports incremental updates, S3 staging, and long-term data growth.
*   **Efficiency:** Automated the "landing-to-gold" workflow via scheduled Databricks Jobs, eliminating manual data cleaning overhead for the analytics team.

## How to Run
1. **Landing Zone:** Upload raw `.csv` files into the S3 bucket `sportsbar-kasis11`.
2. **Databricks Setup:** Execute notebooks in `1_codes/1_setup/` to configure the workspace.
3. **ETL Execution:** Run the dimension processing notebooks, followed by the fact processing notebooks (full load first, then schedule incremental ones).
4. **Dashboard:** Execute the denormalized view SQL script to power the Atlon BI 360 Dashboard and Genie AI features.
