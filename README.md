1. Data Ingestion (Bronze Layer)

Raw data was ingested from Azure Blob Storage using a Fivetran connector.

A connection was established using a secure connection string
Data was sourced from 6 CSV files
Each file was mapped to a corresponding table in the azure_blob_storage schema
All tables were stored under the deminiproject catalog
Key Points:
Automated ingestion using Fivetran
Schema mapping for structured storage
Raw data preserved without transformation



2. Data Transformation (Silver Layer)

The raw data was cleaned and standardized to make it suitable for downstream analytics.

6 separate Databricks notebooks were created (one per table)
Each notebook performs transformations independently
Transformations Applied:
Standardized date formats to YYYY-MM-DD
Converted column names to snake_case
Removed metadata columns added by Fivetran
Handled missing/null values
Cleaned customer names by removing special characters
Output:
Cleaned and transformed tables stored in the Silver layer
Data is structured, consistent, and ready for modeling


3. Data Modeling & Analytics (Gold Layer)

The Gold layer focuses on business-level insights through fact/dimension modeling and KPI generation.

Fact & Dimension Tables

Two notebooks were created:

Sales Notebook
Generates fact_sales
Creates relevant dimension tables
Inventory Notebook
Generates fact_inventory
Creates supporting dimension tables
KPI Calculation
A dedicated KPI notebook computes key business metrics
These KPIs support decision-making and performance analysis
Data Cubes
A final notebook (datacube) creates two data cubes:
Based on fact_sales
Based on fact_inventory
Enables multi-dimensional analysis


4. Orchestration

Pipeline orchestration is handled using Databricks Jobs.

Workflow Design:
Silver Layer Execution
All 6 transformation notebooks run in parallel
Since they operate on independent datasets
Gold Layer (Fact Table Creation)
sales and inventory notebooks:
Run in parallel
Triggered only after Silver layer completion
Final Steps
kpi notebook runs after fact tables are ready
datacube notebook runs sequentially after KPI

Important Notes:

    - In the silver layer in transform_customer notebook, there were two records with same data but the contact being different. Since the joined dates were in dates of same day and no timestamp had to delete one randomly.
    - All the dates are formatted to YYYY-MM-DD format.
    - Some of the record in transform_sales had INR and $ symbol but since no default currency was provided and other records had no symbol, the default currency was taken as INR and the symbols were removed.
    - For the record with $ symbol, after checking it's other sales and product info, it was concluded that the symbol is a misprint and the currency is actually in INR with $ symbol being a mistake.
    - In KPI many assumptions were made which are mentioned in their respective code.
    
