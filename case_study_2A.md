# Case Study #2 - Pizza Runner
## Table of Contents
- Introduction
- Context & Problem Statement
- Case study questions & solutions
  - Takeaways & Revised solutions
- Big takeaways

## Introduction
This is the 2nd case study / challenge as part of 8 Week SQL Challenge in Serious SQL course provided by Data with Danny. More information together with ERD and dataset can be found [here](https://8weeksqlchallenge.com/case-study-2/).


## Context & Problem Statement
> Danny launched Pizza Runner as he believed  “80s Retro Styling and Pizza Is The Future!” Danny started by recruiting “runners” to deliver fresh pizza from Pizza Runner Headquarters (otherwise known as Danny’s house) and also maxed out his credit card to pay freelance developers to build a mobile app to accept orders from customers.

## Case study questions & solutions

#### A. Pizza Metrics

| table\_name      | column\_name | data\_type                  |
| ---------------- | ------------ | --------------------------- |
| customer\_orders | order\_id    | integer                     |
| customer\_orders | customer\_id | integer                     |
| customer\_orders | pizza\_id    | integer                     |
| customer\_orders | exclusions   | character varying           |
| customer\_orders | extras       | character varying           |
| customer\_orders | order\_time  | timestamp without time zone |



1. How many pizzas were ordered?

```sql
SELECT COUNT(*) FROM pizza_runner.customer_orders;
```

2. How many unique customer orders were made?


```sql
SELECT COUNT(DISTINCT order_id) FROM pizza_runner.customer_orders;
```

3. How many successful orders were delivered by each runner?

- First, we check count distribution of unique values in cancellation column
```sql
SELECT
  cancellation,
  COUNT(*) AS row_count
FROM pizza_runner.runner_orders
GROUP BY cancellation;
```
| cancellation            | row\_count |
| ----------------------- | ---------- |
|                         | 3          |
| null                    | 3          |
|                         | 2          |
| Customer Cancellation   | 1          |
| Restaurant Cancellation | 1          |



- We assume that if the cancellation order is different to 'Restaurant Cancellation' or 'Customer Cancellation', the order was successful.



```sql

WITH cte_successful AS (
SELECT *
FROM pizza_runner.runner_orders
WHERE cancellation NOT IN ('Customer Cancellation', 'Restaurant Cancellation') OR cancellation IS NULL

)
SELECT
  runner_id,
  COUNT(*) AS successful_order_count
FROM cte_successful
GROUP BY runner_id
ORDER BY runner_id;


```


##### Takeaways
- Should have used COUNT(DISTINCT order_id instead of COUNT(*) to avoid counting same order_id
- No need the CTE but can use the WHERE filter in the main query directly

4. How many of each type of pizza was delivered?



```sql

WITH cte_pizza_delivered AS(
SELECT
  customer_orders.order_id,
  customer_orders.pizza_id,
  runner_orders.cancellation
FROM pizza_runner.customer_orders
LEFT JOIN pizza_runner.runner_orders
ON customer_orders.order_id = runner_orders.order_id
WHERE cancellation NOT IN ('Customer Cancellation', 'Restaurant Cancellation') OR cancellation IS NULL)

SELECT
  pizza_name,
  COUNT(*) AS pizza_delivered_count
FROM cte_pizza_delivered
INNER JOIN pizza_runner.pizza_names
  ON pizza_names.pizza_id =  cte_pizza_delivered.pizza_id
GROUP BY pizza_name
ORDER BY pizza_delivered_count DESC;


```

OR

```sql

SELECT
  t2.pizza_name,
  COUNT(pizza_name) AS pizza_delivered_count
FROM pizza_runner.customer_orders AS t1
INNER JOIN pizza_runner.pizza_names AS t2
  ON t1.pizza_id = t2.pizza_id
WHERE EXISTS (
  SELECT 1 FROM pizza_runner.runner_orders AS t3
  WHERE t1.order_id = t3.order_id
  AND (
    t3.cancellation IS NULL
    OR t3.cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation')
  )
)
GROUP BY t2.pizza_name
ORDER BY t2.pizza_name;
```

##### Takeaways
- LEFT SEMI JOIN is preferred for efficiency ie. The SELECT statement will only look at columns in resulting inner join of customer_orders & pizza_names tables, without selecting all other columns from runner_order table if a left join or inner join is used to join with runner_order.
- LEFT SEMI JOIN is used to avoid taking in duplicate records in the target tables.

5. How many Vegetarian and Meatlovers were ordered by each customer?

```sql
WITH cte_pizza_flag AS(
SELECT
  customer_orders.customer_id,
  customer_orders.pizza_id,
  pizza_names.pizza_name,
  CASE WHEN pizza_names.pizza_name = 'Meatlovers' THEN 1
    ELSE 0 END AS meatlovers,
  CASE WHEN pizza_names.pizza_name = 'Vegetarian' THEN 1
    ELSE 0 END AS vegetarian
FROM pizza_runner.customer_orders
LEFT JOIN pizza_runner.pizza_names
ON customer_orders.pizza_id = pizza_names.pizza_id
ORDER BY customer_id)

SELECT
  customer_id,
  SUM(meatlovers) AS meatlovers_count,
  SUM(vegetarian) AS vegetarian_count
FROM cte_pizza_flag
GROUP BY customer_id
ORDER BY customer_id;
```

OR

```sql
SELECT
  customer_id,
  SUM(CASE WHEN pizza_id = 1 THEN 1 ELSE 0 END) AS meat_lovers,
  SUM(CASE WHEN pizza_id = 2 THEN 2 ELSE 0 END) AS vegetarian
FROM pizza_runner.customer_orders
GROUP BY customer_id
ORDER BY customer_id;
```

##### Takeaways
- Use a SUM(CASE WHEN) directly to maximise efficiency
- Comments can be added, keep to 80 character only so readers don't have to scroll horizontally.


6. What was the maximum number of pizzas delivered in a single order?

- Because we need to find the number of pizzas delivered, we need to join the customer_orders and runner_orders tables to get the successful deliveries.

```sql
WITH cte_pizza_delivered AS(

SELECT
  order_id,
  COUNT(pizza_id) AS pizza_count,
  RANK() OVER(ORDER BY COUNT(*) DESC) AS count_rank
FROM pizza_runner.customer_orders
WHERE EXISTS(
  SELECT 1
  FROM pizza_runner.runner_orders
  WHERE runner_orders.order_id = customer_orders.order_id
  AND (cancellation IS NULL OR cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation'))
  )
GROUP BY order_id)

SELECT
  pizza_count
FROM cte_pizza_delivered
WHERE count_rank = 1;
```




7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

```sql

-- Clean extras and exclusions columns
WITH clean_customer_orders AS(
SELECT
  order_id,
  customer_id,
  pizza_id,
  CASE WHEN exclusions IN ('null', '') THEN NULL ELSE exclusions END AS exclusions,
  CASE WHEN extras IN ('null', '') THEN NULL ELSE extras END AS extras,
  order_time
FROM pizza_runner.customer_orders
)

-- If there is an extra or exclusion, there is a change
SELECT
  customer_id,
  SUM(
    CASE
      WHEN exclusions IS NULL AND extras IS NULL THEN 1
      ELSE 0
    END
  ) AS no_changes,
  SUM(
    CASE
      WHEN exclusions IS NOT NULL OR extras IS NOT NULL THEN 1
      ELSE 0
    END
  ) AS at_least_1_change
FROM clean_customer_orders
WHERE EXISTS(
  SELECT 1
  FROM pizza_runner.runner_orders
  WHERE runner_orders.order_id = clean_customer_orders.order_id
  AND (
    cancellation IS NULL
    OR cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation'))
  )
GROUP BY customer_id
ORDER BY customer_id;
```

8. How many pizzas were delivered that had both exclusions and extras?

```sql
WITH clean_customer_orders AS(
SELECT
  order_id,
  customer_id,
  pizza_id,
  CASE WHEN exclusions IN ('null', '') THEN NULL ELSE exclusions END AS exclusions,
  CASE WHEN extras IN ('null', '') THEN NULL ELSE extras END AS extras,
  order_time
FROM pizza_runner.customer_orders
)

SELECT
  SUM(
    CASE
      WHEN exclusions IS NOT NULL AND extras IS NOT NULL THEN 1
      ELSE 0
    END
  ) AS count_both_exclusions_and_extras
FROM clean_customer_orders
WHERE EXISTS(
  SELECT 1
  FROM pizza_runner.runner_orders
  WHERE runner_orders.order_id = clean_customer_orders.order_id
  AND (
    cancellation IS NULL
    OR cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation'))
  )
;

```

9. What was the total volume of pizzas ordered for each hour of the day?

```sql
SELECT
  DATE_PART('hour', order_time) AS hour_of_day,
  COUNT(pizza_id) AS pizza_count
FROM pizza_runner.customer_orders
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

- As the question might mean to ask about the hour of each day ordered, we can run a query including order_date

```sql
WITH cte_order_time AS(
SELECT
  pizza_id,
  CAST(TO_CHAR(order_time, 'YYYY-MM-DD') as date) as "order_date",
  DATE_PART('hour', order_time) AS hour
FROM pizza_runner.customer_orders)

SELECT
  order_date,
  hour,
  COUNT(*) AS order_count
FROM cte_order_time
GROUP BY
  order_date,
  hour
ORDER BY
  order_date,
  hour;

```

10. What was the volume of orders for each day of the week?

```sql
SELECT
  TO_CHAR(order_time, 'Day') AS day_of_week,
  COUNT(order_id) AS order_count
FROM pizza_runner.customer_orders
GROUP BY
  day_of_week,
  DATE_PART('dow', order_time)  
ORDER BY DATE_PART('dow', order_time); -- this has to be added in the group by above, or used in an aggregate function,  otherwise it can't be used in this order by

```
##### Takeaways
- To use columns in order by clause, they have to appear in the GROUP BY clause or be used in an aggregate function, though they don't necessarily appear in the SELECT clause

---
&copy; 2022 Yen Chi Nguyen
