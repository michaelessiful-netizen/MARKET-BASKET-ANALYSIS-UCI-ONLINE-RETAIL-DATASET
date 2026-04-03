# MARKET-BASKET-ANALYSIS-UCI-ONLINE-RETAIL-DATASET
BigQuery (Google Cloud Console) | Standard SQL
-- Dataset: UCI Online Retail (541,909 transactions, Dec 2010 - Sep 2011)
-- Author: Rev. Mike Essiful
-- Description: Complete Association Rule Mining pipeline measuring
--              Support, Confidence, Lift, and Co-Occurrence for
--              product pair recommendations in a UK-based e-retailer.
 

-- STEP 1: CREATE RAW TABLE
-- Upload the UCI Online Retail .csv to BigQuery before running.
-- Schema is inferred on upload; confirm column types match below.
 
CREATE OR REPLACE TABLE `your_project.retail.raw_online_retail` (
  InvoiceNo   STRING,
  StockCode   STRING,
  Description STRING,
  Quantity    INT64,
  InvoiceDate STRING,
  UnitPrice   FLOAT64,
  CustomerID  FLOAT64,
  Country     STRING
);
 
 
-- STEP 2: DATA CLEANING & PREPARATION
-- Removes cancellations (InvoiceNo starting with 'C'),
-- returns (Quantity < 1), zero-price items, and
-- non-product stock codes (POST, AMAZON, BANK CHARGES, etc.)
 
CREATE OR REPLACE TABLE `your_project.retail.clean_transactions` AS
 
SELECT
  InvoiceNo,
  StockCode,
  TRIM(UPPER(Description))               AS Description,
  Quantity,
  PARSE_DATE('%m/%d/%Y', SPLIT(InvoiceDate, ' ')[OFFSET(0)]) AS InvoiceDate,
  UnitPrice,
  CAST(CustomerID AS INT64)              AS CustomerID,
  Country,
  ROUND(Quantity * UnitPrice, 2)         AS LineTotal
FROM `your_project.retail.raw_online_retail`
WHERE
  -- Remove cancellation invoices
  NOT STARTS_WITH(InvoiceNo, 'C')
  -- Remove returns and zero-quantity rows
  AND Quantity > 0
  -- Remove zero or negative unit prices
  AND UnitPrice > 0
  -- Remove known non-product stock codes
  AND StockCode NOT IN ('POST', 'D', 'M', 'BANK CHARGES', 'PADS', 'DOT', 'CRUK', 'AMAZONFEE', 'S', 'DCGSSBOY', 'DCGSSGIRL')
  -- Remove rows where stock code is purely alphabetic (service codes)
  AND REGEXP_CONTAINS(StockCode, r'^\d')
  -- Remove rows with missing descriptions
  AND Description IS NOT NULL
  AND TRIM(Description) != '';
 
 
-- STEP 3: EXPLORATORY DATA ANALYSIS (EDA)
-- Basic profile of the cleaned dataset.
 
-- 3a. Dataset overview
SELECT
  COUNT(*)                              AS total_line_items,
  COUNT(DISTINCT InvoiceNo)             AS total_orders,
  COUNT(DISTINCT CustomerID)            AS unique_customers,
  COUNT(DISTINCT StockCode)             AS unique_products,
  COUNT(DISTINCT Country)               AS unique_countries,
  MIN(InvoiceDate)                      AS earliest_date,
  MAX(InvoiceDate)                      AS latest_date,
  ROUND(SUM(LineTotal), 2)              AS total_revenue_usd
FROM `your_project.retail.clean_transactions`;
 
-- 3b. Revenue by country (top 20)
SELECT
  Country,
  COUNT(DISTINCT InvoiceNo)             AS total_orders,
  COUNT(DISTINCT CustomerID)            AS unique_customers,
  ROUND(AVG(Quantity), 2)               AS avg_basket_size,
  ROUND(AVG(LineTotal), 2)              AS avg_line_value,
  ROUND(SUM(LineTotal), 2)              AS total_revenue_usd
FROM `your_project.retail.clean_transactions`
GROUP BY Country
ORDER BY total_revenue_usd DESC
LIMIT 20;
 
-- 3c. Top products by revenue
SELECT
  StockCode,
  Description,
  COUNT(DISTINCT InvoiceNo)             AS num_orders,
  SUM(Quantity)                         AS total_units_sold,
  ROUND(SUM(LineTotal), 2)              AS total_revenue_usd,
  ROUND(AVG(UnitPrice), 2)              AS avg_unit_price,
  COUNT(DISTINCT CustomerID)            AS unique_customers,
  ROUND(AVG(LineTotal), 2)              AS avg_revenue_per_order,
  APPROX_TOP_COUNT(Country, 1)[OFFSET(0)].value AS primary_country
FROM `your_project.retail.clean_transactions`
GROUP BY StockCode, Description
ORDER BY total_revenue_usd DESC
LIMIT 100;
 
 
-- STEP 4: BASKET-LEVEL AGGREGATION
-- One row per invoice with total items and total value.
 
CREATE OR REPLACE TABLE `your_project.retail.basket_summary` AS
 
SELECT
  InvoiceNo,
  Country,
  InvoiceDate                                                   AS order_date,
  FORMAT_DATE('%A', InvoiceDate)                                AS day_of_week,
  EXTRACT(MONTH FROM InvoiceDate)                               AS order_month,
  COUNT(DISTINCT StockCode)                                     AS basket_size,
  ROUND(SUM(LineTotal), 2)                                      AS basket_value,
 
  -- Segment baskets by item count
  CASE
    WHEN COUNT(DISTINCT StockCode) = 1 THEN 'Single Item (1)'
    WHEN COUNT(DISTINCT StockCode) BETWEEN 2 AND 5 THEN 'Small (2-5)'
    ELSE 'Large (6+)'
  END AS segment
 
FROM `your_project.retail.clean_transactions`
GROUP BY InvoiceNo, Country, InvoiceDate;
 
 
-- STEP 5: ASSOCIATION RULE MINING
-- Computes Support, Confidence, and Lift for all product pairs
-- that co-occur in at least 30 orders.
 
-- 5a. Count total distinct orders (denominator for support)
--     Store as a scalar subquery reference inside the main query.
 
-- 5b. Build item-level order lists (one row per product per order)
CREATE OR REPLACE TABLE `your_project.retail.order_items` AS
 
SELECT DISTINCT
  InvoiceNo,
  StockCode,
  MAX(Description) AS Description     -- canonical name per stock code
FROM `your_project.retail.clean_transactions`
GROUP BY InvoiceNo, StockCode;
 
 
-- 5c. Self-join to generate all unique product pairs per order,
--     then aggregate association metrics across orders.
 
CREATE OR REPLACE TABLE `your_project.retail.association_rules` AS
 
WITH total_orders AS (
  -- Scalar: total number of distinct orders in cleaned dataset
  SELECT COUNT(DISTINCT InvoiceNo) AS n
  FROM `your_project.retail.clean_transactions`
),
 
order_counts AS (
  -- Individual product order frequencies
  SELECT
    StockCode,
    MAX(Description)              AS Description,
    COUNT(DISTINCT InvoiceNo)     AS orders
  FROM `your_project.retail.order_items`
  GROUP BY StockCode
),
 
pairs AS (
  -- Self-join: all ordered pairs (A < B alphabetically to avoid duplicates)
  SELECT
    a.InvoiceNo,
    a.StockCode                   AS product_a,
    a.Description                 AS desc_a,
    b.StockCode                   AS product_b,
    b.Description                 AS desc_b
  FROM `your_project.retail.order_items` a
  JOIN `your_project.retail.order_items` b
    ON  a.InvoiceNo  = b.InvoiceNo
    AND a.StockCode  < b.StockCode     -- eliminate self-pairs and reverse duplicates
),
 
pair_counts AS (
  -- Co-occurrence frequency per pair
  SELECT
    product_a,
    desc_a,
    product_b,
    desc_b,
    COUNT(DISTINCT InvoiceNo)     AS co_occurrences
  FROM pairs
  GROUP BY product_a, desc_a, product_b, desc_b
  -- Minimum support threshold: at least 30 co-occurring orders
  HAVING COUNT(DISTINCT InvoiceNo) >= 30
)
 
SELECT
  pc.product_a,
  pc.desc_a,
  pc.product_b,
  pc.desc_b,
  pc.co_occurrences,
 
  -- Support(A,B): share of all orders containing both A and B
  ROUND(pc.co_occurrences / tot.n, 4)                                       AS support_ab,
 
  -- Support(A): share of all orders containing A
  ROUND(oc_a.orders / tot.n, 4)                                             AS support_a,
 
  -- Support(B): share of all orders containing B
  ROUND(oc_b.orders / tot.n, 4)                                             AS support_b,
 
  -- Confidence(A → B): P(B | A)
  ROUND(pc.co_occurrences / oc_a.orders, 4)                                 AS confidence_a_to_b,
 
  -- Confidence(B → A): P(A | B)
  ROUND(pc.co_occurrences / oc_b.orders, 4)                                 AS confidence_b_to_a,
 
  -- Lift: how much more likely the pair co-occurs than by chance
  ROUND(
    (pc.co_occurrences / tot.n) /
    ((oc_a.orders / tot.n) * (oc_b.orders / tot.n)),
    4
  )                                                                           AS lift,
 
  oc_a.orders                                                                 AS orders_a,
  oc_b.orders                                                                 AS orders_b
 
FROM pair_counts    pc
CROSS JOIN total_orders tot
JOIN order_counts  oc_a ON oc_a.StockCode = pc.product_a
JOIN order_counts  oc_b ON oc_b.StockCode = pc.product_b
 
ORDER BY lift DESC;
 
 
-- STEP 6: TABLEAU EXPORT QUERIES
-- Run each SELECT and export results as CSV for Tableau Public.
 
-- 6a. Association Rules (tableau_01_association_rules.csv)
--     Top 500 pairs ordered by Lift descending
SELECT *
FROM `your_project.retail.association_rules`
ORDER BY lift DESC
LIMIT 500;
 
 
-- 6b. Basket Distribution (tableau_02_basket_distribution.csv)
--     Full basket-level summary for distribution and trend charts
SELECT
  InvoiceNo,
  Country,
  order_date,
  day_of_week,
  order_month,
  basket_size,
  basket_value,
  segment
FROM `your_project.retail.basket_summary`
ORDER BY order_date;
 
 
-- 6c. Product Revenue (tableau_03_product_revenue.csv)
--     Top 100 products by revenue for product-level analysis
SELECT
  StockCode,
  Description,
  COUNT(DISTINCT InvoiceNo)             AS num_orders,
  SUM(Quantity)                         AS total_units_sold,
  ROUND(SUM(LineTotal), 2)              AS total_revenue_usd,
  ROUND(AVG(UnitPrice), 2)              AS avg_unit_price,
  COUNT(DISTINCT CustomerID)            AS unique_customers,
  ROUND(AVG(LineTotal), 2)              AS avg_revenue_per_order,
  APPROX_TOP_COUNT(Country, 1)[OFFSET(0)].value AS primary_country
FROM `your_project.retail.clean_transactions`
GROUP BY StockCode, Description
ORDER BY total_revenue_usd DESC
LIMIT 100;
 
 
-- 6d. Country Patterns (tableau_04_country_patterns.csv)
--     Country-level aggregation for geographic visualization
SELECT
  Country,
  COUNT(DISTINCT InvoiceNo)             AS total_orders,
  COUNT(DISTINCT CustomerID)            AS unique_customers,
  ROUND(AVG(basket_size), 2)            AS avg_basket_size,
  ROUND(AVG(basket_value), 2)           AS avg_order_value_usd,
  ROUND(SUM(basket_value), 2)           AS total_revenue_usd
FROM `your_project.retail.basket_summary`
GROUP BY Country
ORDER BY total_revenue_usd DESC
LIMIT 20;
 
 
-- STEP 7: ANALYTICAL VALIDATION CHECKS
-- Run after Steps 4-6 to verify data integrity.
 
-- 7a. Confirm lift > 1 for all returned rules (no accidental negatives)
SELECT
  COUNT(*)                              AS total_rules,
  SUM(CASE WHEN lift > 1 THEN 1 ELSE 0 END) AS rules_lift_above_1,
  ROUND(MAX(lift), 4)                   AS max_lift,
  ROUND(MIN(lift), 4)                   AS min_lift,
  ROUND(AVG(lift), 4)                   AS avg_lift
FROM `your_project.retail.association_rules`;
 
-- 7b. Basket size distribution sanity check
SELECT
  segment,
  COUNT(*)                              AS order_count,
  ROUND(AVG(basket_size), 2)            AS avg_items,
  ROUND(AVG(basket_value), 2)           AS avg_value_usd
FROM `your_project.retail.basket_summary`
GROUP BY segment
ORDER BY avg_items;
 
-- 7c. Confirm confidence bounds [0,1]
SELECT
  ROUND(MIN(confidence_a_to_b), 4)      AS min_conf_atob,
  ROUND(MAX(confidence_a_to_b), 4)      AS max_conf_atob,
  ROUND(MIN(confidence_b_to_a), 4)      AS min_conf_btoa,
  ROUND(MAX(confidence_b_to_a), 4)      AS max_conf_btoa
FROM `your_project.retail.association_rules`;
 
-- 7d. Verify support_ab <= min(support_a, support_b) for all rules
SELECT
  COUNT(*) AS rules_violating_support_constraint
FROM `your_project.retail.association_rules`
WHERE support_ab > LEAST(support_a, support_b);
 
 
-- END OF SCRIPT
-- Replace `your_project.retail` with your actual BigQuery
-- project ID and dataset name before executing.
