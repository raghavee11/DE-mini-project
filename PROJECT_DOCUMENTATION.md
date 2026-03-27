# E-Commerce Data Engineering Mini Project

## 1. Project Overview

**Project Name:** E-Commerce Data Engineering Mini Project  
**Purpose:** Build a scalable data pipeline to process e-commerce transaction data using medallion architecture (Bronze, Silver, Gold layers)  
**Technology Stack:** Databricks, Apache Spark, Delta Lake, Unity Catalog  
**Data Source:** AWS S3 bucket containing e-commerce CSV files  
**Target:** Business intelligence dashboards with KPI metrics

### Key Components
* **Bronze Layer**: Raw data ingestion from S3
* **Silver Layer**: Data cleansing, standardization, and transformation
* **Gold Layer**: Dimensional modeling and aggregated KPIs
* **Dashboard**: E-commerce KPI Dashboard for business insights

## 2. Architecture

### Medallion Architecture
```
S3 (Raw Data) → Bronze (Raw) → Silver (Cleansed) → Gold (Business-Ready) → Dashboard
```

**Catalog Structure:**
* Catalog: `ecommerce`
* Schemas: `bronze`, `silver`, `gold`
* Format: Delta Lake tables

## 3. Data Flow

### Stage 1: Bronze Layer (Data Ingestion)
* **Notebook**: `01_bronze/nb_bronze`
* **Input**: S3 bucket `s3a://de-ecommerce-miniproject/raw/`
* **Process**: 
  * Read metadata.csv to identify all source files
  * Load each CSV file dynamically
  * Write to Delta tables in `ecommerce.bronze` schema
* **Output**: 8 bronze tables (orders, customers, order_items, products, order_payments, order_reviews, sellers, geolocation)

### Stage 2: Silver Layer (Data Transformation)
* **Notebook**: `02_silver/nb_silver`
* **Input**: Bronze tables
* **Process**:
  * Column standardization (lowercase, underscore naming)
  * Data type conversions (dates, numerics)
  * Deduplication
  * Null handling
  * Data quality validations
  * Referential integrity checks
* **Output**: Cleansed tables in `ecommerce.silver` schema

### Stage 3: Gold Layer (Business Layer)
* **Notebook**: `03_gold/nb_gold`
* **Input**: Silver tables
* **Process**:
  * Dimensional modeling (star schema)
  * Fact table creation
  * KPI aggregations
* **Output**: Dimension/fact tables and KPI tables in `ecommerce.gold` schema

## 4. Layer Details

### Bronze Layer Tables
1. `ecommerce.bronze.orders`
2. `ecommerce.bronze.customers`
3. `ecommerce.bronze.order_items`
4. `ecommerce.bronze.products`
5. `ecommerce.bronze.order_payments`
6. `ecommerce.bronze.order_reviews`
7. `ecommerce.bronze.sellers`
8. `ecommerce.bronze.geolocation`

**Characteristics:**
* Schema inference disabled (all strings)
* No transformations applied
* Preserves source data exactly

### Silver Layer Transformations

#### Column Standardization
* Convert column names to lowercase
* Replace spaces with underscores
* Fix typos (e.g., `product_name_lenght` → `product_name_length`)

#### Data Type Conversions
* **Date fields**: Convert timestamp strings to date type
  * `order_purchase_timestamp`, `order_approved_at`, `order_delivered_carrier_date`, `order_delivered_customer_date`, `order_estimated_delivery_date`
  * `review_creation_date`, `review_answer_timestamp`
* **Numeric fields**: Cast to double
  * `price`, `freight_value`, `payment_value`
* **String fields**: Cast zip codes to string for consistency

#### Data Quality Rules
* **Deduplication**: Remove duplicate order_items and reviews
* **Null Handling**: Fill missing values with "unknown" for categorical fields
* **Validation Flags**:
  * `is_late_delivery`: Delivered after estimated date
  * `invalid_delivery`: Delivered before purchase date
  * `delivery_missing`: No delivery date recorded
  * `missing_category_flag`: Product category is unknown

#### Referential Integrity
* Inner joins ensure valid foreign keys:
  * Orders → Customers
  * Order_items → Products
  * Order_items → Sellers

#### Payment Aggregation
* Multiple payments per order aggregated to `total_payment`

### Gold Layer Structure

#### Dimension Tables
1. **dim_customer**: Customer master data
   * `customer_id`, `customer_unique_id`, `customer_city`, `customer_state`

2. **dim_product**: Product master data
   * `product_id`, `product_category_name`, `missing_category_flag`

3. **dim_seller**: Seller master data
   * `seller_id`, `seller_city`, `seller_state`

4. **dim_geography**: Location reference
   * `city`, `state`

5. **dim_date**: Date dimension
   * `date`, `year`, `month`

#### Fact Tables
1. **fact_orders**: Order header information
   * `order_id`, `customer_id`, `order_purchase_timestamp`, `order_status`, `is_late_delivery`

2. **fact_order_items**: Line-level order details
   * `order_id`, `product_id`, `seller_id`, `price`, `freight_value`

3. **fact_payments**: Payment transactions
   * `order_id`, `total_payment`

4. **fact_reviews**: Customer reviews
   * `order_id`, `review_score`

5. **fact_sales_summary**: Consolidated sales fact table
   * Joins all facts and dimensions for comprehensive analysis
   * Includes: customer, product, seller details, pricing, reviews, timestamps


## 6. KPI Metrics

### KPI Tables Created

1. **kpi_revenue_month**: Monthly revenue trends
   * Aggregation: `SUM(total_payment)` by `year`, `month`
   * Use case: Revenue tracking over time

2. **kpi_aov**: Average Order Value
   * Aggregation: `AVG(total_payment)`
   * Use case: Customer spending behavior

3. **kpi_top_products**: Top 10 products by revenue
   * Aggregation: `SUM(price)` by `product_id`, top 10
   * Use case: Product performance analysis

4. **kpi_top_sellers**: Top 10 sellers by revenue
   * Aggregation: `SUM(price)` by `seller_id`, top 10
   * Use case: Seller performance tracking

5. **kpi_orders_geo**: Orders by geography
   * Aggregation: `COUNT(*)` by `customer_city`, `customer_state`
   * Use case: Geographic distribution analysis

6. **kpi_customer_type**: New vs. returning customers
   * Logic: If monthly order count = 1 → "new", else → "returning"
   * Use case: Customer retention metrics

## 7. Dashboard

**Name:** E-commerce KPI Dashboard  
**Type:** Lakeview Dashboard  
**Data Source:** Gold layer KPI tables

**Visualizations:**
* Monthly revenue trends
* Average order value metrics
* Top products performance
* Top sellers ranking
* Geographic order distribution
* Customer segmentation (new vs. returning)

## 8. Technical Implementation

### Technologies Used
* **Databricks Workspace**: Collaborative data engineering environment
* **Apache Spark**: Distributed data processing
* **Delta Lake**: ACID transactions, time travel, schema evolution
* **Unity Catalog**: Data governance and access control
* **PySpark**: Python API for Spark transformations
* **AWS S3**: Source data storage

### Key Implementation Details

#### Dynamic Data Loading (Bronze)
```python
data = spark.read.format("csv").option("header", "true").load("s3a://de-ecommerce-miniproject/raw/metadata.csv")

for d in data.collect():
    temp = spark.read.format('csv').option("header","true").load(f"s3a://de-ecommerce-miniproject/raw/{d.file_name}.{d.extension}")
    temp.write.mode("overwrite").format("delta").saveAsTable(f"ecommerce.bronze.{d.table_name}")
```

#### Naming Standardization (Silver)
```python
def standardize_col_name(col_name):
    return col_name.strip().replace(' ', '_').lower()

def rename_columns(df):
    return df.toDF(*[standardize_col_name(c) for c in df.columns])
```

#### Data Quality Validation (Silver)
```python

orders = orders.withColumn(
    "is_late_delivery",
    col("order_delivered_customer_date") > col("order_estimated_delivery_date")
)

orders = orders.withColumn(
    "invalid_delivery",
    col("order_delivered_customer_date") < col("order_purchase_timestamp")
)
```

#### KPI Aggregation (Gold)
```python
kpi_revenue_month = order_level.groupBy("year","month") \
    .agg(sum("total_payment").alias("revenue"))

kpi_aov = order_level.agg(avg("total_payment").alias("avg_order_value"))
```

### Performance Optimizations
* **Delta Lake format**: Optimized Parquet with ACID properties
* **Partition strategy**: Partitioning by year/month for time-series queries
* **Schema enforcement**: Data quality at write time
* **Caching**: Intermediate DataFrames cached for reuse

## 10. Project Structure

```
DE-mini-project/
├── 01_bronze/
│   └── nb_bronze          # Bronze layer ingestion notebook
├── 02_silver/
│   └── nb_silver          # Silver layer transformation notebook
├── 03_gold/
│   └── nb_gold            # Gold layer modeling notebook
└── PROJECT_DOCUMENTATION.md  
```

