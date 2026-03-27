# Adapting Jaffle Shop for Impala #
This project takes the classic dbt 🥪 The Jaffle Shop 🦘 project and adapts it for dbt-core and a Cloudera Data Warehouse target, using Impala.

## Source Data ##
The original project has seed data, but we can use the sample S3 data instead to get a larger dataset.  To load the large dataset from S3 as discussed below, run the following on a node in the cluster.

### Pull the data from dbt's S3 to our S3 ###
As an alternative you could configure the cluster to use the external S3 bucket as a source directly.
```bash
#!/bin/bash

# 1. Customers
wget https://dbt-tutorial-public.s3-us-west-2.amazonaws.com/long_term_dataset/raw_customers.csv
hdfs dfs -mkdir -p s3a://go01-demo/tmp/dbt_data/raw_customers/
hdfs dfs -put -f raw_customers.csv s3a://go01-demo/tmp/dbt_data/raw_customers/

# 2. Orders
wget https://dbt-tutorial-public.s3-us-west-2.amazonaws.com/long_term_dataset/raw_orders.csv
hdfs dfs -mkdir -p s3a://go01-demo/tmp/dbt_data/raw_orders/
hdfs dfs -put -f raw_orders.csv s3a://go01-demo/tmp/dbt_data/raw_orders/

# 3. Order Items
wget https://dbt-tutorial-public.s3-us-west-2.amazonaws.com/long_term_dataset/raw_order_items.csv
hdfs dfs -mkdir -p s3a://go01-demo/tmp/dbt_data/raw_order_items/
hdfs dfs -put -f raw_order_items.csv s3a://go01-demo/tmp/dbt_data/raw_order_items/

# 4. Products
wget https://dbt-tutorial-public.s3-us-west-2.amazonaws.com/long_term_dataset/raw_products.csv
hdfs dfs -mkdir -p s3a://go01-demo/tmp/dbt_data/raw_products/
hdfs dfs -put -f raw_products.csv s3a://go01-demo/tmp/dbt_data/raw_products/

# 5. Supplies
wget https://dbt-tutorial-public.s3-us-west-2.amazonaws.com/long_term_dataset/raw_supplies.csv
hdfs dfs -mkdir -p s3a://go01-demo/tmp/dbt_data/raw_supplies/
hdfs dfs -put -f raw_supplies.csv s3a://go01-demo/tmp/dbt_data/raw_supplies/

# 6. Stores
wget https://dbt-tutorial-public.s3-us-west-2.amazonaws.com/long_term_dataset/raw_stores.csv
hdfs dfs -mkdir -p s3a://go01-demo/tmp/dbt_data/raw_stores/
hdfs dfs -put -f raw_stores.csv s3a://go01-demo/tmp/dbt_data/raw_stores/
```

### Load into Impala tables ###
The load the data in Impala (HUE), as external tables first then stored as Parquet for performance:
```sql
-- ==========================================
-- 1. CUSTOMERS
-- ==========================================
DROP TABLE IF EXISTS stg_raw_customers;
CREATE EXTERNAL TABLE stg_raw_customers (id STRING, name STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE
LOCATION 's3a://go01-demo/tmp/dbt_data/raw_customers/'
TBLPROPERTIES ('skip.header.line.count'='1');

DROP TABLE IF EXISTS raw_customers;
CREATE TABLE raw_customers (id STRING, name STRING) STORED AS PARQUET;

INSERT INTO raw_customers SELECT * FROM stg_raw_customers;

-- ==========================================
-- 2. ORDERS
-- ==========================================
DROP TABLE IF EXISTS stg_raw_orders;
CREATE EXTERNAL TABLE stg_raw_orders (
    id STRING, customer STRING, ordered_at STRING, store_id STRING, 
    subtotal INT, tax_paid INT, order_total INT
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE
LOCATION 's3a://go01-demo/tmp/dbt_data/raw_orders/'
TBLPROPERTIES ('skip.header.line.count'='1');

DROP TABLE IF EXISTS raw_orders;
CREATE TABLE raw_orders (
    id STRING, customer STRING, ordered_at TIMESTAMP, store_id STRING, 
    subtotal INT, tax_paid INT, order_total INT
) STORED AS PARQUET;

INSERT INTO raw_orders 
SELECT id, customer, CAST(ordered_at AS TIMESTAMP), store_id, subtotal, tax_paid, order_total 
FROM stg_raw_orders;

-- ==========================================
-- 3. ORDER ITEMS
-- ==========================================
DROP TABLE IF EXISTS stg_raw_order_items;
CREATE EXTERNAL TABLE stg_raw_order_items (id STRING, order_id STRING, sku STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE
LOCATION 's3a://go01-demo/tmp/dbt_data/raw_order_items/'
TBLPROPERTIES ('skip.header.line.count'='1');

DROP TABLE IF EXISTS raw_order_items;
CREATE TABLE raw_order_items (id STRING, order_id STRING, sku STRING) STORED AS PARQUET;

INSERT INTO raw_order_items SELECT * FROM stg_raw_order_items;

-- ==========================================
-- 4. PRODUCTS
-- ==========================================
DROP TABLE IF EXISTS stg_raw_products;
CREATE EXTERNAL TABLE stg_raw_products (
    sku STRING, name STRING, type STRING, price INT, description STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE
LOCATION 's3a://go01-demo/tmp/dbt_data/raw_products/'
TBLPROPERTIES ('skip.header.line.count'='1');

DROP TABLE IF EXISTS raw_products;
CREATE TABLE raw_products (
    sku STRING, name STRING, type STRING, price INT, description STRING
) STORED AS PARQUET;

INSERT INTO raw_products SELECT * FROM stg_raw_products;

-- ==========================================
-- 5. SUPPLIES
-- ==========================================
DROP TABLE IF EXISTS stg_raw_supplies;
CREATE EXTERNAL TABLE stg_raw_supplies (
    id STRING, name STRING, cost INT, perishable BOOLEAN, sku STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE
LOCATION 's3a://go01-demo/tmp/dbt_data/raw_supplies/'
TBLPROPERTIES ('skip.header.line.count'='1');

DROP TABLE IF EXISTS raw_supplies;
CREATE TABLE raw_supplies (
    id STRING, name STRING, cost INT, perishable BOOLEAN, sku STRING
) STORED AS PARQUET;

INSERT INTO raw_supplies SELECT * FROM stg_raw_supplies;

-- ==========================================
-- 6. STORES
-- ==========================================
DROP TABLE IF EXISTS stg_raw_stores;
CREATE EXTERNAL TABLE stg_raw_stores (
    id STRING, name STRING, opened_at STRING, tax_rate FLOAT
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE
LOCATION 's3a://go01-demo/tmp/dbt_data/raw_stores/'
TBLPROPERTIES ('skip.header.line.count'='1');

DROP TABLE IF EXISTS raw_stores;
CREATE TABLE raw_stores (
    id STRING, name STRING, opened_at TIMESTAMP, tax_rate FLOAT
) STORED AS PARQUET;

INSERT INTO raw_stores 
SELECT id, name, CAST(opened_at AS TIMESTAMP), tax_rate 
FROM stg_raw_stores;
```

### Test the imported data ###
Test the imported row counts:
```sql
SELECT 'raw_customers' AS table_name, COUNT(*) AS row_count FROM raw_customers
UNION ALL
SELECT 'raw_orders', COUNT(*) FROM raw_orders
UNION ALL
SELECT 'raw_order_items', COUNT(*) FROM raw_order_items
UNION ALL
SELECT 'raw_products', COUNT(*) FROM raw_products
UNION ALL
SELECT 'raw_supplies', COUNT(*) FROM raw_supplies
UNION ALL
SELECT 'raw_stores', COUNT(*) FROM raw_stores;

-- Compute stats for query optimization
COMPUTE STATS raw_customers;
COMPUTE STATS raw_orders;
COMPUTE STATS raw_order_items;
COMPUTE STATS raw_products;
COMPUTE STATS raw_supplies;
COMPUTE STATS raw_stores;
```

### Tidy up ###
Then clean up the intermediate tables:
```sql
DROP TABLE IF EXISTS stg_raw_products;
DROP TABLE IF EXISTS stg_raw_customers;
DROP TABLE IF EXISTS stg_raw_orders;
DROP TABLE IF EXISTS stg_raw_order_items;
DROP TABLE IF EXISTS stg_raw_supplies;
DROP TABLE IF EXISTS stg_raw_stores;
```

# 🥪 The Jaffle Shop 🦘 Project #

This is a sandbox project for exploring the basic functionality and latest features of dbt. It's based on a fictional restaurant called the Jaffle Shop that serves [jaffles](https://en.wikipedia.org/wiki/Pie_iron).

This README will guide you through setting up the project on dbt-core. Working through this example should give you a good sense of how dbt-core works and what's involved with setting up your own project.

## 🏗️ Platform setup

Create a logical database in your data warehouse for the Jaffle Shop project. We recommend using the name `jaffle_shop` for consistency with the project.

## 👷🏻‍♀️ Project setup

Once your development platform of choice and dependencies are set up, use the following steps to get the project ready for whatever you'd like to do with it.  Run a `dbt build` to build the project.

The following should now be done:

- Synthetic data loaded into your warehouse
- Development environment set up and ready to go
- The project built and tested

You're free to explore the Jaffle Shop from here, or if you want to learn more about [setting up Environment and Jobs](#%EF%B8%8F-setting-up-dbt-cloud-environments-and-jobs), [generating a larger dataset](#-working-with-a-larger-dataset), or [setting up pre-commit hooks](#-pre-commit-and-sqlfluff) to standardize formatting and linting workflows, carry on!
