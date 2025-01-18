# Case Study #4 - Data Bank
## Table of Contents
- Introduction
- Context & Problem Statement
- Exploratory Data Analysis
- Case study questions & solutions
  - Takeaways & Revised solutions
- Takeaways

## Introduction
This is the 4th case study / challenge as part of 8 Week SQL Challenge in Serious SQL course provided by Data with Danny. More information together with ERD and dataset can be found [here](https://8weeksqlchallenge.com/case-study-4/).


## Context & Problem Statement
> There is a new innovation in the financial industry called Neo-Banks: new aged digital only banks without physical branches.
Danny thought that there should be some sort of intersection between these new age banks, cryptocurrency and the data world…so he decides to launch a new initiative - Data Bank!
Data Bank runs just like any other digital bank - but it isn’t only for banking activities, they also have the world’s most secure distributed data storage platform!
Customers are allocated cloud data storage limits which are directly linked to how much money they have in their accounts. There are a few interesting caveats that go with this business model, and this is where the Data Bank team need your help!
The management team at Data Bank want to increase their total customer base - but also need some help tracking just how much data storage their customers will need.
This case study is all about calculating metrics, growth and helping the business analyse their data in a smart way to better forecast and plan for their future developments!

## Exploratory Data Analysis

### A. Customer Nodes Exploration
#### 1. Inspect row counts - `COUNT(*)`

```sql
SELECT
  COUNT(*) AS row_count
FROM
  data_bank.customer_nodes;
```

There are a total of 3,500 rows in the customer_nodes table.

#### 2. Count of unique values in a column - `COUNT(DISTINCT col_name)`

```sql
SELECT
  COUNT(DISTINCT node_id) AS num_unique_node,
  COUNT(DISTINCT customer_id) AS num_unique_customer
FROM
  data_bank.customer_nodes;
```
There are 5 unique nodes and 500 unique customers in the system.

#### 3. Frequency of values in a single column - `COUNT(*)` and `GROUP BY`
```sql
SELECT
  node_id,
  COUNT(*) AS row_count
FROM
  data_bank.customer_nodes
GROUP BY
  node_id;
```
This answer the question: What is the frequency of values in the `node_id`?
| node_id | row_count |
| ------- | --------- |
| 1       | 728       |
| 3       | 699       |
| 5       | 707       |
| 2       | 662       |
| 4       | 704       |

#### 4. Percentage of unique values in a column - `COUNT(*)` & `100 * COUNT(*) / SUM(COUNT(*)) OVER ()` for percentage

```sql
SELECT
  node_id,
  COUNT(*) AS frequency,
  ROUND(COUNT(*)::NUMERIC / SUM(COUNT(*)) OVER() * 100, 2) AS percentage
FROM
  data_bank.customer_nodes
GROUP BY
  node_id
ORDER BY percentage DESC;
```
This answer the question: What is the percentage of each unique `node_id` in the table?
| node_id | frequency | percentage |
| ------- | --------- | ---------- |
| 1       | 728       | 20.80      |
| 5       | 707       | 20.20      |
| 4       | 704       | 20.11      |
| 3       | 699       | 19.97      |
| 2       | 662       | 18.91      |

#### 5. Count of multiple columns combination(s)

```sql
SELECT
  customer_id,
  node_id,
  COUNT(*) AS row_count
FROM
  data_bank.customer_nodes
GROUP BY
  customer_id,
  node_id
ORDER BY row_count DESC
LIMIT 5;
```

This answers the following questions:
- What is the max number of times you can observe the same combination of `customer_id` and `node_id` in this table?
- What are the 5 most frequent `customer_id` and `node_id` combinations in the `customer_nodes` table?

| customer_id | node_id | row_count |
| ----------- | ------- | --------- |
| 26          | 3       | 6         |
| 422         | 4       | 5         |
| 389         | 2       | 5         |
| 102         | 4       | 5         |
| 397         | 3       | 5         |

#### 6. Find duplicate records across all columns - Select all columns and use `GROUP BY` with all columns
```sql
SELECT
  customer_id,
  region_id,
  node_id,
  start_date,
  end_date,
  COUNT(*) AS frequency
FROM
  data_bank.customer_nodes
GROUP BY
  customer_id,
  region_id,
  node_id,
  start_date,
  end_date
ORDER BY frequency DESC
LIMIT 5;
```
This query returns each unique record and the frequency. If frequency > 1, it means there are duplicates.
| customer_id | region_id | node_id | start_date | end_date   |
| ----------- | --------- | ------- | ---------- | ---------- |
| 76          | 2         | 3       | 2020-03-30 | 2020-04-11 |
| 194         | 4         | 5       | 2020-02-26 | 2020-02-27 |
| 39          | 5         | 4       | 2020-04-09 | 2020-04-23 |
| 435         | 3         | 1       | 2020-01-01 | 2020-01-09 |
| 343         | 2         | 5       | 2020-01-01 | 2020-01-22 |

There is no duplicate records in the `customer_nodes` table.

### B. Customer Transactions Exploration

#### 1. Summary statistics - Mean, median, mode, min, max, min - max, stddev, var
Let's look at the key summary statistics for `txn_amount` column in the  `customer_transactions` table.
```sql
SELECT
  'txn_amount' AS measure,
  ROUND(MIN(txn_amount), 2) AS min_value,
  ROUND(MAX(txn_amount), 2) AS max_value,
  ROUND(AVG(txn_amount), 2) AS mean_value,
  ROUND(
  -- this function actually returns a float which is incompatible with ROUND!
  -- we use this cast function to convert the output type to NUMERIC
    CAST(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY txn_amount) AS NUMERIC), 2)
    AS median_value,
  ROUND(
    MODE() WITHIN GROUP (ORDER BY txn_amount), 2) AS mode_value,
  ROUND(
    STDDEV(txn_amount), 2) AS standard_deviation,
  ROUND(
    VARIANCE(txn_amount), 2) AS variance
FROM data_bank.customer_transactions;
```
| measure    | min_value | max_value | mean_value | median_value | mode_value | standard_deviation | variance |
| ---------- | --------- | --------- | ---------- | ------------ | ---------- | ------------------ | -------- |
| txn_amount | 0.00      | 1000.00   | 504.21     | 503.00       | 483.00     | 288.12             | 83014.52 |

- The mean, median and mode values are very close to each other.
- 68% of values lie in the range of 504.01 +- 288.12 (+- 1*stddev)
- 95% of values lie in the range of mean 504.01 +- 2\*288.12 (+- 2*stddev)
- 99.7% of values lie in the range of mean 504.01 +- 3\*288.12 (+- 3*stddev)

#### 2. Another method to calculate the mode - Use `COUNT(*)` and `GROUP BY` -> helpful in case of multiple modes!
```sql
SELECT
  txn_amount,
  COUNT(*) AS frequency
FROM data_bank.customer_transactions
GROUP BY txn_amount
ORDER BY frequency DESC
LIMIT 5;
```

#### 3. Cumulative distributions - `NTILE(100) OVER (ORDER BY col_name)`
- `NTILE(100)` window function order the `txn_amount` values from smallest to largest, then break them into 100 buckets.
- For each bucket, we then calculate what is the minimum and maximum values, and how many records there are.

```sql
WITH percentile_values AS (
  SELECT
    txn_amount,
    NTILE(100) OVER (ORDER BY txn_amount) AS percentile
  FROM data_bank.customer_transactions
)

SELECT
  percentile,
  MIN(txn_amount) AS floor_value,
  MAX(txn_amount) AS ceiling_value,
  COUNT(*) AS frequency
FROM percentile_values
GROUP BY percentile
ORDER BY percentile;
```
Results:
- 5% of txn_amount are under $51.
- It appears that there are 59 values in each of the first five buckets.

| percentile | floor_value | ceiling_value | frequency |
| ---------- | ----------- | ------------- | --------- |
| 1          | 0           | 11            | 59        |
| 2          | 12          | 20            | 59        |
| 3          | 20          | 31            | 59        |
| 4          | 31          | 40            | 59        |
| 5          | 40          | 51            | 59        |

- 99% of values are under $990.
- With the last 5 buckets, each of them has 58 values.

| percentile | floor_value | ceiling_value | frequency |
| ---------- | ----------- | ------------- | --------- |
| 96  | 951 | 963  | 58 |
| 97  | 963 | 972  | 58 |
| 98  | 972 | 981  | 58 |
| 99  | 981 | 990  | 58 |
| 100 | 990 | 1000 | 58 |

#### 4. Investigate outliers at end range using percentile - `percentile = 100` & `percentile = 1` using `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`
##### 4.1 Large outlier checking
- Look at `WHERE percentile = 100`
```sql
WITH percentile_values AS (
  SELECT
    txn_amount,
    NTILE(100) OVER (ORDER BY txn_amount) AS percentile
  FROM data_bank.customer_transactions
)

SELECT
  txn_amount,
  ROW_NUMBER() OVER(ORDER BY txn_amount DESC) AS row_number_order,
  RANK() OVER(ORDER BY txn_amount DESC) AS rank_order,
  DENSE_RANK() OVER(ORDER BY txn_amount DESC) AS dense_rank_order
FROM percentile_values
WHERE percentile = 100
ORDER BY txn_amount DESC;
```
| txn_amount | row_number_order | rank_order | dense_rank_order |
| ---------- | ---------------- | ---------- | ---------------- |
| 1000       | 1                | 1          | 1                |
| 1000       | 2                | 1          | 1                |
| 999        | 3                | 3          | 2                |
| 999        | 4                | 3          | 2                |
| 999        | 5                | 3          | 2                |
| 999        | 6                | 3          | 2                |
| 999        | 7                | 3          | 2                |
| 999        | 8                | 3          | 2                |

- In other datasets, we might see outliers that could devastatingly skew mean results.

##### 4.2 Small outlier checking
- Look at `WHERE percentile = 1`
```sql
WITH percentile_values AS (
  SELECT
    txn_amount,
    NTILE(100) OVER (ORDER BY txn_amount) AS percentile
  FROM data_bank.customer_transactions
)

SELECT
  txn_amount,
  ROW_NUMBER() OVER(ORDER BY txn_amount DESC) AS row_number_order,
  RANK() OVER(ORDER BY txn_amount DESC) AS rank_order,
  DENSE_RANK() OVER(ORDER BY txn_amount DESC) AS dense_rank_order
FROM percentile_values
WHERE percentile = 1
ORDER BY txn_amount DESC;
```

#### 5. Remove outliers if needed by creating a temporary (temp) table
- Let's assume we want to remove some outliers, we will create a temp table e.g. values that are greater than / equal to 900 or less than / equal to 10.

```sql
DROP TABLE IF EXISTS clean_transactions;

CREATE TEMP TABLE clean_transactions AS (
  SELECT *
  FROM data_bank.customer_transactions
  WHERE txn_amount < 900 AND txn_amount > 10  
);
```
- Then we recalculate summary statistics on this new temp table

```sql
SELECT
  'txn_amount' AS measure,
  ROUND(MIN(txn_amount), 2) AS min_value,
  ROUND(MAX(txn_amount), 2) AS max_value,
  ROUND(AVG(txn_amount), 2) AS mean_value,
  ROUND(
  -- this function actually returns a float which is incompatible with ROUND!
  -- we use this cast function to convert the output type to NUMERIC
    CAST(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY txn_amount) AS NUMERIC), 2)
    AS median_value,
  ROUND(
    MODE() WITHIN GROUP (ORDER BY txn_amount), 2) AS mode_value,
  ROUND(
    STDDEV(txn_amount), 2) AS standard_deviation,
  ROUND(
    VARIANCE(txn_amount), 2) AS variance
FROM clean_transactions;
```

- Then we recalculate the new cumulative distribution on the new data. The result will be more relevant and obvious compared to the result above in case of outlier(s).
```sql
WITH percentile_values AS (
  SELECT
    txn_amount,
    NTILE(100) OVER (ORDER BY txn_amount) AS percentile
  FROM clean_transactions
)

SELECT
  percentile,
  MIN(txn_amount) AS floor_value,
  MAX(txn_amount) AS ceiling_value,
  COUNT(*) AS frequency
FROM percentile_values
GROUP BY percentile
ORDER BY percentile;
```

8. Frequency distributions (Histogram) - `WIDTH_BUCKET()`
- Build a histogram that shows how the values are distributed.

```sql
SELECT
  WIDTH_BUCKET(txn_amount, 0, 1000, 10) AS bucket,
  ROUND(AVG(txn_amount), 2) AS average_txn_amount,
  COUNT(*) AS frequency
FROM data_bank.customer_transactions
GROUP BY bucket
ORDER BY bucket;
```
![image](Txn_amount histogram.png)

## Case study questions & solutions
To be continued

A. Customer Nodes Exploration
1. How many unique nodes are there on the Data Bank system?
2. What is the number of nodes per region?
3. How many customers are allocated to each region?
4. How many days on average are customers reallocated to a different node?
5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

## Takeaways


---
&copy; 2025 Yen Chi Nguyen
