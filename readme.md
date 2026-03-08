# Retail Sales Data Engineering Pipeline
<img width="1722" height="467" alt="image" src="https://github.com/user-attachments/assets/2b286f51-8b28-48dd-8017-f20e151dab6f" />

<img width="1335" height="592" alt="image" src="https://github.com/user-attachments/assets/601225ee-728d-42fc-b43a-1f68b700d6a2" />

## Project Overview
This project implements an end-to-end **Retail Sales Data Pipeline** using the **Medallion Architecture (Bronze → Silver → Gold)**.  
The pipeline ingests raw retail sales data, performs data validation and transformation, and produces business-ready analytical tables for reporting.

The solution is orchestrated using **Azure Data Factory**, transformations are implemented using **Azure Databricks notebooks**, and data is stored in **Azure Data Lake Storage Gen2**.

---
# Technologies Used

- **Azure Data Factory** – Pipeline orchestration  
- **Azure Data Lake Storage Gen2** – Data lake storage  
- **Azure Databricks** – Data transformation and processing  
- **Azure SQL Database** – Staging and watermark tracking  
- **Delta Lake** – Gold layer storage  
- **PySpark** – Distributed data processing  
- **Git** – Version control  
- **CI/CD Pipelines** – Automated deployment  

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

# Gold Layer

The **Gold layer** contains business-ready analytical tables built using a **Star Schema**.

```
gold_layer
├── dim_customer
├── dim_products
├── dim_store
├── dim_date
└── fact_sales
```

Tables are stored using **Delta format**.

---

# Dimension Tables

## dim_customer

Source: Silver Layer

Transformations:

- Extract customer attributes
- Remove duplicates
- Compare with existing dimension table
- Generate surrogate keys
- Apply **SCD Type 1** updates

Sink:

```
gold/dim_customer
```

---

## dim_products

Source: Silver Layer

Transformations:

- Extract product attributes
- Remove duplicate records
- Generate surrogate keys
- Apply SCD Type 1 logic

Sink:

```
gold/dim_products
```

---

## dim_store

Source: Silver Layer

Transformations:

- Extract store attributes
- Identify new and existing stores
- Generate surrogate keys
- Apply SCD Type 1 merge logic

Sink:

```
gold/dim_store
```

---

## dim_date

Source:

Transaction date from Silver dataset

Transformations:

- Extract Year, Month, Quarter, Day
- Generate surrogate key
- Create calendar attributes

Sink:

```
gold/dim_date
```

---

# Fact Table

## fact_sales

Source: Silver Layer dataset

### Transformations

- Join with dimension tables:
  - dim_customer
  - dim_products
  - dim_store
  - dim_date
- Replace business keys with surrogate keys
- Generate analytical measures

### Example Measures

- Sales Amount
- Quantity
- Revenue

Sink:

```
gold/fact_sales
```

---

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



