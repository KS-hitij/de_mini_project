# DE Mini Project Documentation

## Project Overview

This is a multi-layer data engineering project following the **Medallion Architecture** (Bronze → Silver → Gold → Data Cubes) for retail analytics.

* **Domain**: Retail analytics covering sales, inventory, customers, stores, products, and returns
* **Source**: Azure Blob Storage via Fivetran
* **Tech Stack**: Databricks, PySpark, Delta Lake
* **Catalog**: `de_mini_project`
* **Cloud Provider**: AWS

## Architecture & Data Pipeline

### Medallion Architecture Layers

#### 1. Bronze Layer (`azure_blob_storage` schema)

**Purpose**: Raw data ingestion from Azure Blob Storage via Fivetran

**Tables** (6 tables):
* `customer` - Raw customer data
* `inventory` - Raw inventory records
* `product` - Raw product information
* `sales` - Raw sales transactions
* `stores` - Raw store details
* `transaction` - Raw return transactions

**Characteristics**:
* Contains raw column names with special characters (e.g., `!!Cust_Ref_ID!!`, `[Contact_Info]`, `%SKU_ID%`)
* Includes Fivetran metadata columns: `_file`, `_line`, `_modified`, `_fivetran_synced`
* Data quality issues present (inconsistent date formats, NULL values, duplicates)

#### 2. Silver Layer (`silver` schema)

**Purpose**: Data cleansing and standardization

**Tables** (6 tables):
* `customer` - Cleansed customer data
* `inventory` - Cleansed inventory records
* `product` - Cleansed product information
* `sales` - Cleansed sales transactions
* `stores` - Cleansed store details
* `return` - Cleansed return transactions (renamed from `transaction`)

**Transformations**:
* **Column name normalization**: Remove special characters, standardize naming
* **Data type conversions**: Standardize date formats (YYYY-MM-DD)
* **Data cleansing**:
  * Replace "NULL" strings with actual nulls
  * Remove quotes and underscores from names
  * Standardize phone number formats
  * Handle duplicate records
  * Remove Fivetran metadata columns
* **Basic validations**: Handle "Unknown Customer" cases

**Silver Layer Notebooks**:
* `transform_customer` - Standardizes customer data, handles multiple date formats
* `transform_inventory` - Cleans inventory records
* `transform_product` - Normalizes product information
* `transform_sales` - Processes sales transactions
* `transform_stores` - Standardizes store data
* `transform_returns` - Processes return transactions

#### 3. Gold Layer (`gold` schema)

**Purpose**: Business-ready star schema for analytics

**Fact Tables** (2 tables):

1. **`fact_sales`** (20 rows)
   * **Grain**: One row per transaction
   * **Columns**:
     * `transaction_id` (PK)
     * `sku_id` (FK to dim_inventory)
     * `customer_id` (FK to dim_customer)
     * `store_id` (FK to dim_stores)
     * `qty_sold` (numeric measure)
     * `unit_price` (numeric measure)
     * `date` (transaction date)
     * `return_id` (FK to dim_returns, nullable)
     * `return_date` (nullable)
   * **Join Logic**: Left join with returns on `transaction_id` to preserve all transactions

2. **`fact_inventory`** (8 rows)
   * **Grain**: One row per SKU per store
   * **Columns**:
     * `sku_id` (FK to dim_inventory)
     * `store_id` (FK to dim_stores)
     * `stock_on_hand` (numeric measure)
     * `base_cost` (numeric measure)
     * `marked_price` (numeric measure)

**Dimension Tables** (4 tables):

1. **`dim_customer`** (6 rows)
   * `customer_id` (PK)
   * `full_name`
   * `contact_info`
   * `joined_date`
   * `gender_code`

2. **`dim_stores`** (3 stores)
   * `store_id` (PK)
   * `location` (store name)
   * `city` (region format: CITY_AREA)
   * `manager_contact`
   * **Stores**:
     * S_01: TechNova - Downtown (Chennai South)
     * S_02: TechNova - Mall Road (Bengaluru North)
     * S_03: TechNova - Airport (Mumbai West)

3. **`dim_inventory`**
   * `sku_id` (PK)
   * `store_id`
   * `last_audit_date`
   * `category` (Electronics, Lifestyle, Accessories)

4. **`dim_returns`** (10 return reasons)
   * `return_id` (PK)
   * `reason` (Damaged, Defective, Wrong Item, Not Happy, Changed Mind, High Temp Issue)

**Gold Layer Notebooks**:

* **`sales`** - Creates `fact_sales`, `dim_customer`, `dim_stores`, `dim_returns`
  * Joins sales with returns (left join on `transaction_id`)
  * Separates fact metrics from dimensional attributes
  * Preserves all transactions even without returns

* **`inventory`** - Creates `fact_inventory` and `dim_inventory`
  * Joins inventory with product data
  * Splits into fact (metrics) and dimension (attributes) tables

#### 4. Data Cube Layer (`datacube` schema)

**Purpose**: Pre-aggregated denormalized views optimized for BI reporting

**Tables** (2 datacubes):

1. **`sales`** datacube - Denormalized view joining:
   * `fact_sales`
   * `dim_customer`
   * `dim_stores`
   * `dim_inventory`
   * `dim_returns`
   * **Join Type**: Left joins to preserve all transactions
   * **Use Case**: Sales analytics, customer analysis, return analysis

2. **`inventory`** datacube - Denormalized view joining:
   * `fact_inventory`
   * `dim_inventory`
   * `dim_stores`
   * **Use Case**: Inventory analytics, stock management, store performance

**Datacube Notebook**:
* `datacube` - Creates both denormalized cubes from gold layer tables

## Data Cube Strategy

### Answer: **2 Data Cubes**

Given **2 fact tables** (`fact_sales` and `fact_inventory`), the project correctly implements **2 separate data cubes**:

1. **Sales Cube**: Combines sales transactions with customer, store, product, and return dimensions
2. **Inventory Cube**: Combines inventory levels with product categories and store information

### Why This Approach is Optimal:

* **Distinct Business Processes**: Each fact table represents a different business process:
  * Sales cube: Transaction-level analysis (who bought what, when, where)
  * Inventory cube: Stock-level analysis (what's in stock, where, at what cost)

* **Different Granularity**:
  * Sales: One row per transaction
  * Inventory: One row per SKU per store

* **Query Performance**: Separate cubes are optimized for their specific use cases

* **Clear Separation**: Sales analytics vs inventory analytics have distinct requirements

* **Scalability**: Adding more fact tables (e.g., fact_promotions, fact_shipments) would each get their own cube

### Alternative Approaches (Not Used):

* **Single Combined Cube**: Would be problematic due to different grain levels and would create a Cartesian product
* **Additional Aggregate Cubes**: Could add pre-aggregated versions (e.g., daily/monthly sales) if query performance requires it

## Key Performance Indicators (KPIs)

**KPI Notebook**: `kpi` - Calculates 10 business metrics

### 1. Total Revenue
* **Value**: 9,850
* **Logic**: Excludes returned purchases (`return_id IS NULL`)
* **Formula**: `SUM(qty_sold * unit_price)` where `return_id IS NULL`

### 2. Profit per Product

Profit calculated as difference between marked price and base cost (not sold price):

| SKU | Base Cost | Marked Price | Profit | Category |
| --- | --- | --- | --- | --- |
| P_8810 | 40,000 | 60,000 | 20,000 | Electronics |
| P_1022 | 1,200 | 2,500 | 1,300 | Electronics |
| P_9921 | 600 | 1,500 | 900 | Electronics |
| P_1100 | 150 | 800 | 650 | Accessories |
| P_7721 | 100 | 450 | 350 | Lifestyle |
| P_4451 | 50 | 200 | 150 | Lifestyle |

### 3. Top Revenue Category
* **Category**: Electronics
* **Revenue**: 6,350 (excludes returns)
* **Logic**: Group by category, sum revenue from non-returned transactions

### 4. Average Items per Transaction
* **Value**: 2.1 items
* **Formula**: `AVG(qty_sold)`

### 5. Return Rate
* **Value**: 50%
* **Details**: 10 out of 20 transactions were returned
* **Formula**: `(COUNT(return_id) / COUNT(*)) * 100`

### 6. Out of Stock Items
* **Count**: 2 products
* **Items**:
  * P_1022 at S_02 (Electronics)
  * P_1100 at S_01 (Accessories)
* **Logic**: `stock_on_hand = 0`

### 7. Store Performance

Ranked by total revenue (including returned purchases):

| Rank | Store ID | Store Name | City | Revenue |
| --- | --- | --- | --- | --- |
| 1 | S_01 | TechNova - Downtown | Chennai South | 119,450 |
| 2 | S_02 | TechNova - Mall Road | Bengaluru North | 58,251 |
| 3 | S_03 | TechNova - Airport | Mumbai West | 5,900 |

### 8. Discount Impact

* **Purpose**: Tracks revenue loss from selling below marked price
* **Formula**: `(marked_price - unit_price) * qty_sold`
* **Use Case**: Identify discount strategies and their financial impact

### 9. Repeat Customers

Customers with multiple orders (excludes "Unknown Customer"):

| Customer ID | Full Name | Order Count |
| --- | --- | --- |
| C-551 | ARUN KUMAR | 7 |
| C-882 | Priya Sharma | 3 |
| C-102 | vikram singh | 3 |
| C-404 | Anjali Nair | 2 |
| C-999 | Rahul .K | 2 |

### 10. Slow Moving Inventory

* **Timeframe**: Last 60 days (since no products sold in last 30 days)
* **Item**: P_7721 (Lifestyle category) at S_03
* **Details**: Base cost: 100, Marked price: 450, Last audit: 2025-01-01

## Dashboards & Visualization

* **Dashboard**: "New Dashboard 2026-03-27"
* **Data Source**: `de_mini_project.datacube.sales`
* **Purpose**: BI reporting on sales metrics

## Project Statistics

* **Total Tables**: 20 tables
  * Bronze layer: 6 tables
  * Silver layer: 6 tables
  * Gold layer: 6 tables (2 facts + 4 dimensions)
  * Datacube layer: 2 cubes

* **Notebooks**: 11 transformation and analytics notebooks
  * 6 silver layer transformation notebooks
  * 2 gold layer modeling notebooks
  * 1 datacube notebook
  * 1 KPI analytics notebook

* **Data Volume**: Small sample dataset for demonstration
  * 20 sales transactions
  * 8 inventory records
  * 6 customers
  * 3 stores
  * 10 returns (50% return rate)

* **Date Range**: January 2026 (sales data from Jan 15-31, 2026)

## Technical Notes

* **Table Format**: All tables use Delta format for ACID transactions
* **Write Mode**: Overwrite mode used for all table writes (full refresh)
* **Data Quality Handling**:
  * Silver layer handles "Unknown Customer" cases
  * Null contact information preserved
  * Duplicate customer records handled by keeping first record
* **Join Strategy**: Gold layer uses left joins to preserve all transactions
* **KPI Logic**: Revenue calculations exclude returned purchases
* **Date Standardization**: Multiple date formats normalized in silver layer:
  * DD-MM-YYYY
  * YYYY.MM.DD
  * MMM DD, YYYY
  * YYYY-MM-DD

## Data Lineage

```
Azure Blob Storage (via Fivetran)
    ↓
BRONZE LAYER (azure_blob_storage schema)
    ├── customer, inventory, product, sales, stores, transaction
    ↓
SILVER LAYER (silver schema) [Data Cleansing]
    ├── customer, inventory, product, sales, stores, return
    ↓
GOLD LAYER (gold schema) [Star Schema Modeling]
    ├── FACT TABLES: fact_sales, fact_inventory
    └── DIMENSION TABLES: dim_customer, dim_stores, dim_inventory, dim_returns
    ↓
DATACUBE LAYER (datacube schema) [Denormalization]
    ├── sales (denormalized sales datacube)
    └── inventory (denormalized inventory datacube)
    ↓
BI DASHBOARDS & ANALYTICS
    └── KPI calculations and visualizations
```

## Future Enhancements

### Data Pipeline
* **Incremental Loading**: Replace full overwrites with incremental (merge/upsert) patterns
* **Change Data Capture (CDC)**: Implement CDC from source systems
* **Data Quality Framework**: Add comprehensive validation rules and data quality checks
* **Error Handling**: Implement quarantine tables for bad records
* **Schema Evolution**: Handle schema changes gracefully

### Data Modeling
* **SCD Type 2**: Implement Slowly Changing Dimensions for historical tracking
  * Track customer address changes
  * Track product price history
  * Track store manager changes
* **Additional Fact Tables**:
  * `fact_promotions` - Track promotional campaigns
  * `fact_shipments` - Track delivery performance
* **Junk Dimensions**: Consolidate flags and indicators
* **Bridge Tables**: Handle many-to-many relationships

### Analytics & KPIs
* **Time-Series Analysis**: Add date dimensions for trend analysis
* **Customer Segmentation**: RFM analysis (Recency, Frequency, Monetary)
* **Cohort Analysis**: Track customer behavior over time
* **Predictive Analytics**: Forecast demand and identify churn risk
* **Inventory Optimization**: Reorder point calculations, safety stock levels

### Automation & Orchestration
* **Scheduled Jobs**: Automate pipeline execution with Databricks workflows
* **Monitoring & Alerting**: Set up alerts for pipeline failures and data quality issues
* **Data Lineage Tracking**: Implement automated lineage documentation
* **Performance Optimization**: Partition tables, optimize file sizes (Z-ORDER, OPTIMIZE)

### Security & Governance
* **Access Control**: Implement fine-grained access control (Unity Catalog)
* **Data Masking**: Mask PII fields (customer contact info)
* **Audit Logging**: Track data access and modifications
* **Data Retention Policies**: Implement archival strategy

### Scalability
* **Performance Tuning**: Add table partitioning by date
* **Caching Strategy**: Materialize frequently accessed aggregations
* **Streaming**: Consider streaming ingestion for real-time analytics

---

**Documentation Created**: 2026-03-27  
**Project Catalog**: `de_mini_project`  
**Architecture**: Medallion (Bronze → Silver → Gold → Datacube)  
**Data Cubes**: 2 (Sales Cube + Inventory Cube)
