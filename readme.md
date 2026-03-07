# **Retail Electronics Sales Data Engineering Pipeline**

<img width="1508" height="432" alt="image" src="https://github.com/user-attachments/assets/a8ac45c2-319d-4d3a-9b82-642b6d0517d8" />

Project Overview
This project demonstrates an end-to-end modern data engineering pipeline for processing Retail Electronics Sales data using the Medallion Architecture (Bronze → Silver → Gold).
The pipeline ingests raw sales data, performs incremental loading, applies data validation and transformation, and finally builds a star schema data model for analytical reporting.
The orchestration of the pipeline is managed using Azure Data Factory, while data transformation and processing are handled in Azure Databricks using PySpark.

# **Architecture Overview**

### The solution follows the Medallion Architecture pattern:

Source Files
     │
     ▼
Azure Data Lake Storage (Raw)
     │
     ▼
Azure SQL Database (Staging)
     │
     ▼
Bronze Layer (CSV)
     │
     ▼
Silver Layer (Parquet)
     │
     ▼
Gold Layer (Delta Tables)
     │
     ▼
Star Schema for Analytics

# **Technologies used:**

Azure Data Factory – Pipeline orchestration
Azure Data Lake Storage Gen2 – Data lake storage
Azure Databricks – Data transformation and processing
Azure SQL Database – Staging and watermark tracking
Delta Lake – Gold layer storage
PySpark – Data processing
Data Pipeline Workflow

# **The pipeline performs the following steps**

### 1. Source Data Ingestion
Retail Electronics Sales data is initially uploaded into Azure Data Lake Storage Gen2 as raw source files.
These files contain transactional sales data including:
Customer details
Product details
Store information
Transaction date
Sales amount

### 2. Incremental Load Detection
The pipeline begins with Lookup activities in Azure Data Factory:
Lookup: Last Load
Fetches the last processed timestamp from the watermark table stored in Azure SQL Database.
Lookup: Current Load
Determines the current maximum timestamp from the source data.
This allows the pipeline to process only new or updated records.

### 3. Incremental Data Load
A Copy Data activity performs the incremental data ingestion.
Source: ADLS Gen2 Raw Files
Destination: Azure SQL Database Staging Table
Only records between:
LastLoadTimestamp
and
CurrentLoadTimestamp
are copied.
This ensures efficient incremental data processing.

### 4. Watermark Update
After successful ingestion, a Stored Procedure activity updates the watermark table in Azure SQL Database.
This step ensures the pipeline remembers the last successful load timestamp for the next execution.

### 5. Bronze Layer Processing
The first Azure Databricks Notebook loads the raw staging data into the Bronze layer.
Bronze Layer Characteristics
Raw data ingestion
Minimal transformations
Stored in CSV format
Preserves original data structure
Bronze data acts as a raw historical data archive.

### 6. Silver Layer Processing
A second Databricks Notebook processes the Bronze data and loads it into the Silver layer.
Transformations Applied
Data validation
Null handling
Data type corrections
Standardization
Duplicate removal
Schema enforcement
The cleansed data is stored in:
Parquet format
Benefits of Parquet:
Columnar storage
Faster queries
Efficient compression

### 7. Gold Layer Data Modeling
The Gold layer represents business-ready data models.
In this stage, the pipeline builds a Star Schema consisting of:
Dimension Tables
dim_customers
dim_products
dim_store
dim_date
Fact Table
fact_sales

### 8. Slowly Changing Dimension (SCD Type 1)
Dimension tables use SCD Type 1 logic.
This means:
Existing records are overwritten
No historical tracking
Only the latest values are maintained
Example:
If a product price changes, the old value is replaced with the new value.

### 9. Gold Layer Storage
The final Gold tables are stored as Delta Tables in Azure Databricks.
#### Benefits of Delta Lake:
ACID transactions
Schema enforcement
Time travel
Faster analytics queries

# Pipeline Execution Flow

### The pipeline runs in the following sequence:

Lookup Last Load
      │
Lookup Current Load
      │
Copy Incremental Data
      │
Update Watermark Table
      │
Bronze Notebook
      │
Silver Notebook
      │
Dimension Notebooks
      │
Fact Sales Notebook

Dimension tables are built first, followed by the Fact Sales table.

# Star Schema Design

### The Gold layer implements the following data model:

              dim_customers
                     │
                     │
dim_products ─── fact_sales ─── dim_store
                     │
                     │
                 dim_date

This structure enables efficient analytical queries and reporting.

# Key Features of the Pipeline
Incremental data processing
Medallion architecture implementation
Data quality validation
Distributed data processing using Spark
Optimized storage using Parquet and Delta
Star schema for analytics
Automated orchestration using Azure Data Factory







