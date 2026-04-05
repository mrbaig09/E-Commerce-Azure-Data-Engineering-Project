# E-Commerce Azure Data Engineering Project

An end-to-end production-grade data pipeline built on Microsoft Azure
using the Medallion Architecture (Bronze → Silver → Gold) with
incremental loading, Delta Lake, and automated orchestration.

---

## Architecture Overview

<img width="1360" height="1480" alt="ecommerce_medallion_architecture" src="https://github.com/user-attachments/assets/c1604b3c-386f-45a0-a05c-057092348791" />


---

## Azure Resources Used

<img width="1919" height="775" alt="image" src="https://github.com/user-attachments/assets/12ded8b2-8f85-4e69-a891-0425593a165d" />


---

## Tech Stack

| Azure Service | Purpose |
|---|---|
| Azure SQL Database | Source OLTP data store |
| Azure Data Factory | Pipeline orchestration + incremental load |
| Azure Data Lake Storage Gen2 | Bronze / Silver / Gold layered storage |
| Azure Databricks | PySpark data cleaning + transformation |
| Delta Lake | ACID transactions, time travel, MERGE upsert |
| Azure Key Vault | Centralized secrets management |

---

## Dataset

Indian e-commerce sales data with 15,000 orders:

| Column | Description |
|---|---|
| order_id | Unique order identifier |
| order_date | Date of order (DD-MM-YYYY) |
| customer_name | Customer full name |
| region | North / South / East / West |
| city | City of delivery |
| category | Product category |
| sub_category | Product sub-category |
| product_name | Full product name |
| quantity | Units ordered |
| unit_price | Price per unit |
| discount | Discount percentage |
| sales | Final sale amount |
| profit | Profit earned |
| payment_mode | UPI / Card / COD / Net Banking |
| payment_status | Paid / Pending |

---

## Medallion Architecture — Layer by Layer

### Bronze Layer (Raw)
- Stores raw data exactly as received from SQL source
- File format: Parquet (compressed, columnar)
- Partitioned by: year / month / day
- Never overwritten — append only
- Each ADF run creates a new dated folder

### Silver Layer (Cleaned)
- Duplicate records removed on order_id
- Null values filled with business defaults
- order_date parsed from VARCHAR to DateType
- Derived columns added:
  - order_year, order_month, order_quarter
  - net_sales (after discount)
  - profit_margin_pct
  - revenue_category (High / Medium / Low)
- Format: Delta Lake with MERGE upsert
- Partitioned by order_year / order_month

### Gold Layer (Aggregated)
- Three aggregated tables for reporting:
  1. monthly_sales_by_category — sales KPIs by region and category
  2. payment_analysis — payment mode breakdown by region
  3. top_products — best performing products by revenue
- Format: Delta Lake with incremental MERGE
- Ready for Power BI / reporting tools

---

## Pipeline Design

### ADF Pipeline — PL_Ecommerce_Incremental_Load

<img width="1425" height="495" alt="image" src="https://github.com/user-attachments/assets/d931a5a8-bf53-4e4b-8c63-f751fdb0666e" />


#### Activity 1 — Lookup: LKP_GetWatermark
- Reads last successful load date from watermark_table in SQL DB
- Returns single row with last_load_date column
- Used to filter only new records in next activity

#### Activity 2 — Copy Data: CPY_SQL_To_Bronze
- Source: Azure SQL Database (ecommerce_sales table)
- Query filters records WHERE order_date > last watermark date
- Adds 3 metadata columns: _ingestion_timestamp, _source_table, _load_type
- Sink: ADLS Gen2 Bronze container in Parquet format
- Partition path: bronze/yyyy/MM/dd/

#### Activity 3 — Stored Procedure: SP_UpdateWatermark
- Executes sp_update_watermark in SQL DB
- Updates watermark to current date only on successful copy
- Ensures no data loss if pipeline fails midway

#### Activity 4 — If Condition: IF_CheckRowsCopied
- Expression: @greater(activity('CPY_SQL_To_Bronze').output.rowsCopied, 0)
- True branch: proceeds to Databricks notebooks
- False branch: sends alert via Web Activity

#### Activity 5 — Notebook: NB_BronzeToSilver
- Triggers Databricks notebook nb_bronze_to_silver
- Reads today's Bronze parquet file only
- Cleans and transforms data
- Merges into Silver Delta table

#### Activity 6 — Notebook: NB_SilverToGold
- Triggers Databricks notebook nb_silver_to_gold
- Reads only new Silver rows since last Gold run
- Merges aggregations into 3 Gold Delta tables

---

## Incremental Load Strategy

### ADF — Watermark Pattern
