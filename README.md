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

<img width="1248" height="471" alt="image" src="https://github.com/user-attachments/assets/ecf295f7-e446-4d8b-a434-046771e72f9b" />




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

Run 1:  watermark = 01-01-2023
copies all 5000 rows
updates watermark = 04-04-2026
Run 2:  watermark = 04-04-2026
copies only 10 new rows
updates watermark = 05-04-2026
Run 3:  watermark = 05-04-2026
copies only next batch
updates watermark = 06-04-2026

### Databricks — Delta MERGE Pattern
New rows arrive in Silver
↓
IF order_id already exists → UPDATE the record
IF order_id is new        → INSERT the record
Historical data never rewritten ✅

### Gold — Control File Pattern
After each Gold run → saves last_run_ts to control file
Next run reads control file → filters Silver rows
WHERE _silver_load_ts > last_run_ts
Only new rows aggregated → merged into Gold tables

---

## ADF Pipeline — Successful Runs

### Run 1 — Full Initial Load (5000 rows)

<img width="1525" height="781" alt="image" src="https://github.com/user-attachments/assets/07393488-f40c-4d0b-81d1-caf92fc490f8" />



### Run 2 — Incremental Load (10 new rows)

<img width="1527" height="779" alt="image" src="https://github.com/user-attachments/assets/31a8ada7-8ce7-46ac-a06d-3e242ec6c6bb" />


---

## ADF Linked Services

![Linked Services](screenshots/05_linked_services.png)
<!--
    SCREENSHOT TO ADD:
    ADF Studio → Manage → Linked services
    Should show:
      - DataLakebronze (Azure Data Lake Storage Gen2)
      - SqlDatabaseSource (Azure SQL Database)
      - AzureDatabricksLinkedService (Azure Databricks)
    Save as: screenshots/05_linked_services.png
-->

---

## Storage — Bronze Container

<img width="1919" height="818" alt="image" src="https://github.com/user-attachments/assets/0157c1f8-e9ce-4c6b-b32f-ec2932a14893" />


---

## Storage — Silver Container

<img width="1919" height="790" alt="image" src="https://github.com/user-attachments/assets/93defc7c-3180-4d8b-a12d-ce95bbc0007e" />
<img width="1919" height="800" alt="image" src="https://github.com/user-attachments/assets/39179f16-02b6-4268-9368-5e87b437db57" />


---

## Storage — Gold Container

<img width="1919" height="788" alt="image" src="https://github.com/user-attachments/assets/0827bc54-d9d0-4d0e-8137-d8c12f394593" />


---

## Databricks — Notebooks Run

<img width="1919" height="750" alt="image" src="https://github.com/user-attachments/assets/4516e9c2-97bf-4d82-8f20-fd3981ef2183" />
<img width="1918" height="872" alt="image" src="https://github.com/user-attachments/assets/92fe5869-1562-4fb9-96de-618b19b613f2" />
<img width="1919" height="865" alt="image" src="https://github.com/user-attachments/assets/9b65afcc-fbb8-4d2e-b52f-75925e1d48c4" />


---

## Key Vault — Secrets Management

<img width="1919" height="823" alt="image" src="https://github.com/user-attachments/assets/8139bcd7-b200-48e0-93dc-69511351f0d6" />

---

## SQL Scripts

### Watermark Table
```sql
CREATE TABLE watermark_table (
    table_name     VARCHAR(100),
    last_load_date VARCHAR(20)
);

INSERT INTO watermark_table
VALUES ('ecommerce_sales', '01-01-2023');
```

### Stored Procedure
```sql
CREATE PROCEDURE sp_update_watermark
    @last_load_date VARCHAR(20),
    @table_name     VARCHAR(100)
AS
BEGIN
    UPDATE watermark_table
    SET last_load_date = @last_load_date
    WHERE table_name = @table_name;
END;
```

---

## Project Structure
ecommerce-azure-data-engineering/
│
├── README.md
│
├── architecture/
│   └── medallion_architecture.png
│
├── adf_pipelines/
│   └── PL_Ecommerce_Incremental_Load.json
│
├── databricks_notebooks/
│   ├── nb_bronze_to_silver.py
│   └── nb_silver_to_gold.py
│
├── sql_scripts/
│   ├── 01_create_tables.sql
│   ├── 02_create_watermark.sql
│   └── 03_stored_procedure.sql
│
├── sample_data/
│   └── ecommerce_sales_sample.csv
│
├── screenshots/
│   ├── 01_resource_group.png
│   ├── 02_adf_pipeline_design.png
│   ├── 03_adf_run1_success.png
│   ├── 04_adf_run2_incremental.png
│   ├── 05_linked_services.png
│   ├── 06_bronze_container.png
│   ├── 07_silver_container.png
│   ├── 08_gold_container.png
│   ├── 09_databricks_notebook_run.png
│   └── 10_key_vault_secrets.png
│
└── .gitignore

---

## How to Reproduce This Project

### Prerequisites
- Active Azure Subscription
- Azure SQL Database + SQL Server
- Azure Data Lake Storage Gen2
- Azure Data Factory
- Azure Databricks (Standard tier or above)
- Azure Key Vault

### Step 1 — SQL Setup
```sql
-- Run scripts in order
-- sql_scripts/01_create_tables.sql
-- sql_scripts/02_create_watermark.sql
-- sql_scripts/03_stored_procedure.sql
```

### Step 2 — Storage Setup
Create 3 containers in ADLS Gen2:
- bronze
- silver
- gold

### Step 3 — ADF Setup
- Import pipeline JSON from adf_pipelines/
- Create linked services for SQL and ADLS
- Update connection strings with your resource names
- Publish all

### Step 4 — Databricks Setup
- Upload notebooks from databricks_notebooks/
- Add storage account key to cluster Spark config:
- fs.azure.account.key.YOUR_STORAGE_ACCOUNT.dfs.core.windows.net YOUR_KEY

### Step 5 — Run Pipeline
- Trigger ADF pipeline manually for first run
- Pipeline runs automatically daily at 1 AM IST
- Databricks notebooks triggered automatically by ADF

---

## Pipeline Monitoring

All pipeline runs monitored from ADF Monitor tab:

- Green checkmark = activity succeeded
- Duration visible per activity
- Row counts visible in Copy activity output
- Failed runs send alert via Web Activity

---

## Author

**Your Name**
Azure Data Engineer — 4 Years Experience

- LinkedIn: your-linkedin-url
- Email: your-email
- Location: Pune, India

---

## License

This project is open source and available under the MIT License
