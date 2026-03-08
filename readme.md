# Retail Sales Data Engineering Pipeline
<img width="1722" height="467" alt="image" src="https://github.com/user-attachments/assets/2b286f51-8b28-48dd-8017-f20e151dab6f" />

<img width="1335" height="592" alt="image" src="https://github.com/user-attachments/assets/601225ee-728d-42fc-b43a-1f68b700d6a2" />
# Project Highlights

This project demonstrates an **end-to-end Azure Data Engineering pipeline** built using modern cloud data architecture patterns.

Key highlights of this project include:

- Implementation of **Medallion Architecture (Bronze → Silver → Gold)** for scalable data processing
- **Incremental data ingestion** using watermark-based loading
- **Azure Data Factory pipelines** for orchestration and automation
- **Azure Databricks notebooks** for large-scale data transformations using PySpark
- Implementation of **data quality validation rules**
- Handling of invalid records using a **quarantine data zone**
- Creation of **Star Schema analytical model** in the Gold layer
- Implementation of **Slowly Changing Dimension (SCD Type 1)** logic
- **Delta Lake tables** for optimized analytical storage
- **Pipeline monitoring and audit logging** using SQL audit tables
- **Failure notification system** using Azure Logic Apps email alerts
- **CI/CD implementation** using GitHub integration with Azure Data Factory
- **Scheduled pipeline execution** using ADF triggers (every 12 hours)

This project simulates a **real-world retail data engineering workflow** and demonstrates best practices used in modern cloud data platforms.

---

# Skills Demonstrated

- Azure Data Factory
- Azure Databricks
- PySpark
- Azure Data Lake Storage Gen2
- Delta Lake
- Data Modeling (Star Schema)
- Incremental Data Processing
- Data Quality Validation
- Data Pipeline Monitoring
- CI/CD for Data Pipelines
- Cloud Data Engineering Architecture
---

## Project Overview
This project implements an end-to-end **Retail Sales Data Pipeline** using the **Medallion Architecture (Bronze → Silver → Gold)**.  
The pipeline ingests raw retail sales data, performs data validation and transformation, and produces business-ready analytical tables for reporting.

The solution is orchestrated using **Azure Data Factory**, transformations are implemented using **Azure Databricks notebooks**, and data is stored in **Azure Data Lake Storage Gen2**.
  
---
# Architecture

The project follows the **Medallion Architecture** pattern:

Bronze → Raw Data  
Silver → Cleaned and Validated Data  
Gold → Business Ready Analytical Tables

---

## Pipeline Flow

The Azure Data Factory pipeline performs the following steps:

1. Retrieve the **last processed watermark** from the watermark table.
2. Identify the **current load timestamp**.
3. Calculate the **incremental record count** using a Script activity.
4. If **incremental records exist**:
   - Load incremental data into the Bronze layer.
   - Update the watermark table.
   - Execute the Databricks transformation pipeline (Silver → Gold).
   - Log pipeline execution status in the audit table.
5. If **no incremental records exist**:
   - Log `NoData` status in the audit table.
6. If pipeline fails:
   - Trigger **Web Activity → Azure Logic App → Email Alert**

The pipeline runs automatically every **12 hours using an ADF Trigger**.

---

# Bronze Layer

The **Bronze layer** stores raw data ingested from the source system.

## Characteristics

- Minimal transformations
- Raw historical storage
- Stored in CSV format
- Preserves original data structure

## Source

Azure Data Lake Storage Gen2 raw files.

## Sink

Bronze container in Data Lake.

Example path: /bronze/retail_sales/


---

# Silver Layer

The **Silver layer** performs data cleansing and transformation.

### Source

Bronze Layer → Raw CSV files

### Data Quality Checks

| Validation Rule | Description |
|---|---|
| Unit_Price > 0 | Ensures product price is valid |
| Units_Sold > 0 | Ensures quantity sold is valid |
| Discount_Percent between 0 and 100 | Ensures discount values are realistic |

Duplicate records are removed using **Order_ID**.

---

### Invalid Data Handling

Records failing validation rules are written to a **quarantine location**.

```
/silver/quarantine/
```

This allows investigation of data issues without affecting analytics workloads.

---

### Data Transformations

After validation, the following transformations are applied:

| Column | Description |
|---|---|
| Gross_Revenue | Unit_Price × Units_Sold |
| Discount_Amount | Gross_Revenue × Discount_Percent |
| Total_Revenue | Gross_Revenue − Discount_Amount |

Numeric columns are standardized by rounding to two decimal places.

---

### Output

Cleaned data is written to the Silver layer in **Parquet format**.

```
/silver/clean/
```

---

# Data Transformations – Gold Layer

All transformations done using ADF Data Flows and Databricks Notebooks (Spark-backed)

## dim_customer
- Source: Silver Layer (dim_passenger)
- Derived Column: Extract customer attributes (name, age, gender_flag, country)
- Filter: Remove duplicate records based on natural key
- Surrogate Key Generation: Using incremental logic with parameter-driven approach
  - Parameter: Incremental_Flag (0 = full load, 1 = incremental load)
  - If Flag = 0: Start surrogate keys from 1
  - If Flag = 1: Get max existing key from target table and add monotonically_increasing_id()
  - New column: dim_customer_key = max_value + monotonically_increasing_id()
- SCD Type 1: Implemented using Delta merge operation
  - Merge condition: Match on business key
  - When matched: Update existing records with latest attributes
  - When not matched: Insert new records with generated surrogate keys
- Sink: Delta Lake → gold/dim_customer/

## dim_products
- Source: Silver Layer
- Select: Extract product attributes (product_id, product_name, category, price)
- Aggregate: Remove duplicate product records
- Surrogate Key Generation: Using same incremental logic
  - Full load (Flag=0): Keys start from 1
  - Incremental load (Flag=1): max_value + monotonically_increasing_id()
- SCD Type 1: Delta merge operation for upserts
- Sink: Delta Lake → gold/dim_products/

## dim_store
- Source: Silver Layer
- Select: Extract store attributes (store_id, store_name, location, region)
- Surrogate Key Generation: Parameter-based incremental logic
  - New stores get keys = max existing key + monotonically_increasing_id()
- SCD Type 1: Delta merge for updates
- Sink: Delta Lake → gold/dim_store/

## dim_date
- Source: Transaction dates from Silver Layer
- Derived Column: Extract Year, Month, Quarter, Day, Week number
- Surrogate Key Generation: Generate integer surrogate key (YYYYMMDD format)
- Derived Column: Create additional calendar attributes
- Alter Row: Insert only (static dimension, no SCD needed)
- Sink: Delta Lake → gold/dim_date/

## fact_sales
- Source: Silver Layer dataset
- Join: Replace business keys with surrogate keys by joining with all dimension tables
- Derived Column: Calculate measures (Sales Amount, Quantity, Revenue)
- Select: Final fact table columns with surrogate keys and measures
- Alter Row: Insert only (new transactions)
- Sink: Delta Lake → gold/fact_sales/

---

## 🔑 Surrogate Key Generation Strategy

All dimension tables follow a consistent approach for surrogate key generation using parameter-driven incremental logic:

# Slowly Changing Dimension (SCD Type 1)

Dimension tables use **SCD Type 1** logic.

Characteristics:

- Existing records are overwritten
- No historical tracking
- Only the latest value is stored

Example:

If a product price changes, the old value is replaced with the new value.

---
# Pipeline Monitoring and Alerts

To ensure reliability, the pipeline includes **automated monitoring and alerting**.

## Failure Alert Mechanism

If any activity fails:

1. Azure Data Factory triggers a **Web Activity**
2. The Web Activity sends a request to an **Azure Logic App**
3. Logic App sends an **email notification**


---

# Audit Logging

Pipeline executions are logged in the table
pipeline_audit_log

Captured information includes:

- Pipeline Name
- Pipeline Run ID
- Activity Name
- Source System
- Table Processed
- Start Time
- End Time
- Execution Status
- Processed Row Count
- Watermark Value
- Error Message

---
# Scheduling

The pipeline is triggered automatically using an **Azure Data Factory Trigger** that runs every **12 hours**.

---

# CI/CD Implementation

The project uses **GitHub integration with Azure Data Factory**.

Workflow

1. Development happens in **feature branches**
2. Code is merged into **main branch**
3. Azure Data Factory publishes pipelines
4. CI/CD deploys updated pipelines

---

# Technologies Used

- Azure Data Factory – Pipeline orchestration
- Azure Data Lake Storage Gen2 – Data lake storage
- Azure Databricks – Data transformation
- Azure SQL Database – Watermark tracking and audit logs
- Delta Lake – Gold layer storage
- PySpark – Distributed data processing
- Azure Logic Apps – Email alert notifications
- GitHub – Version control
- CI/CD Pipelines – Automated deployment

---

# Key Data Engineering Concepts Demonstrated

This project demonstrates several important data engineering concepts:

- Medallion Architecture (Bronze / Silver / Gold)
- Incremental Data Processing
- Watermark-Based Data Loading
- Data Quality Validation
- Star Schema Data Modeling
- Slowly Changing Dimensions (SCD Type 1)
- Distributed Data Processing with PySpark
- Pipeline Monitoring and Alerting
- CI/CD for Data Pipelines
- Cloud Data Engineering using Azure Services

---

# How to Run This Project

Follow the steps below to deploy and execute the project.

## 1. Setup Azure Resources

Create the following Azure services:

- Azure Data Factory
- Azure Data Lake Storage Gen2
- Azure Databricks Workspace
- Azure SQL Database
- Azure Logic App (for email alerts)

---

## 2. Configure Data Lake Structure

Create the following containers and folders.
# How to Run This Project

Follow the steps below to deploy and execute the project.

## 1. Setup Azure Resources

Create the following Azure services:

- Azure Data Factory
- Azure Data Lake Storage Gen2
- Azure Databricks Workspace
- Azure SQL Database
- Azure Logic App (for email alerts)

---

## 2. Configure Data Lake Structure

Create the following containers and folders.
# How to Run This Project

Follow the steps below to deploy and execute the project.

## 1. Setup Azure Resources

Create the following Azure services:

- Azure Data Factory
- Azure Data Lake Storage Gen2
- Azure Databricks Workspace
- Azure SQL Database
- Azure Logic App (for email alerts)

---

## 2. Configure Data Lake Structure

Create the following containers and folders

```
datalake
│
├── bronze
│ └── retail_sales
│
├── silver
│ ├── clean
│ └── quarantine
│
└── gold
├── dim_customer
├── dim_products
├── dim_store
├── dim_date
└── fact_sales
```

---

## 3. Upload Source Dataset

Upload the retail dataset into the Bronze layer.

Example location:
/bronze/retail_sales/retail_sales_timestamp.csv

---

## 4. Deploy Azure Data Factory Pipelines

Deploy the following pipelines:
pl_retail_sales_ingestion
pl_retail_sales_silver_to_gold

These pipelines handle:

- Incremental ingestion
- Watermark tracking
- Data transformation
- Monitoring and logging

---

## 5. Configure Databricks Notebooks

Create notebooks for:
silver_transformation
dim_customer
dim_products
dim_store
dim_date
fact_sales

These notebooks perform Silver and Gold layer transformations.

---

## 6. Configure Pipeline Trigger

Create an **Azure Data Factory Trigger** to schedule the ingestion pipeline.

Schedule: Every 12 Hours

---

## 7. Configure Failure Alerts

Set up **Azure Logic App** for pipeline monitoring.

Workflow:
```
ADF Web Activity
│
▼
Azure Logic App
│
▼
Send Email Alert
```

This ensures the data engineering team receives notifications if pipeline execution fails.

---

# Pipeline Audit Logging

To track pipeline execution and monitor operational health, this project implements an **audit logging framework**.

Each pipeline run writes execution metadata into the **pipeline_audit_log** table.

This helps with:

- Pipeline monitoring
- Failure investigation
- Performance tracking
- Operational reporting

---

# Audit Table Schema

The audit table stores execution metadata for every pipeline run.





