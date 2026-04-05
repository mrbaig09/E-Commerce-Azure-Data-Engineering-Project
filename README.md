# 🛒 E-Commerce Azure Data Engineering Project

An end-to-end production-grade data pipeline built on Microsoft Azure 
using the Medallion Architecture (Bronze → Silver → Gold).

---

## 🏗️ Architecture

![Architecture](architecture/medallion_architecture.png)

---

## 🔧 Tech Stack

| Tool | Purpose |
|------|---------|
| Azure SQL Database | Source data store |
| Azure Data Factory | Orchestration & incremental load |
| Azure Data Lake Gen2 | Bronze / Silver / Gold storage |
| Azure Databricks | Data cleaning & transformation |
| Delta Lake | ACID transactions & time travel |
| Azure Key Vault | Secrets management |

---

## 📊 Dataset

Indian e-commerce sales data with 15,000 orders across:
- 5 regions, 20+ cities
- 12 product categories
- Multiple payment modes
- Date range: 2023 – 2025

---

## 🔄 Pipeline Flow
