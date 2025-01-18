# Case Study #1 - Danny's Diner
## Table of Contents
- Introduction
- Context & Problem Statement
- Case study questions & solutions
  - Takeaways & Revised solutions
- Big takeaways

## Introduction
This is the first case study / challenge as part of 8 Week SQL Challenge in Serious SQL course provided by Data with Danny. More information together with ERD and dataset can be found [here](https://8weeksqlchallenge.com/case-study-1/).


## Context & Problem Statement
> Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favourite foods: sushi, curry and ramen. Danny’s Diner is in need of your assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.


> Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.


## Case study questions & solutions

### Joining all tables
As the tables are fairly small, I joined all three tables together to get the base table used to answer all questions.
<details>
  <summary>My solution</summary>

```sql
DROP TABLE IF EXISTS joined_table;
CREATE TEMP TABLE joined_table AS

WITH cte_joint AS(
SELECT
  sales.customer_id,
  sales.order_date,
  sales.product_id,
  menu.product_name,
  menu.price,
  members.join_date
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
  ON menu.product_id = sales.product_id
LEFT JOIN dannys_diner.members -- need to do a left join here as members table doesn't contain all customer_id in sales table
  ON members.customer_id = sales.customer_id
)

SELECT *
FROM cte_joint;

SELECT *
FROM joined_table;
```

</details>

### 1. What is the total amount each customer spent at the restaurant?
<details>
  <summary>My solution</summary>

```sql
SELECT
  customer_id,
  SUM(price) AS amount_spent
FROM joined_table
GROUP BY customer_id
ORDER BY customer_id;
```
</details>

| customer\_id | amount\_spent |
| ------------ | ------------- |
| A            | 76            |
| B            | 74            |
| C            | 36            |

### 2. How many days has each customer visited the restaurant?
<details>
  <summary>My solution</summary>

```sql
SELECT
  customer_id,
  COUNT(DISTINCT order_date) AS days_visited
FROM joined_table
GROUP BY customer_id
ORDER BY customer_id;
```
</details>

| customer\_id | days\_visited |
| ------------ | ------------- |
| A            | 4             |
| B            | 6             |
| C            | 2             |

### 3. What was the first item from the menu purchased by each customer?
<details>
  <summary>My solution</summary>

```sql
WITH cte_ranked AS(
SELECT
  customer_id,
  order_date,
  product_name,
  ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY order_date, product_id) AS _rank
FROM joined_table)

SELECT
  customer_id,
  product_name AS first_item_purchased
FROM cte_ranked
WHERE _rank = 1;
```
</details>

| customer\_id | first\_item\_purchased |
| ------------ | ---------------------- |
| A            | sushi                  |
| B            | curry                  |
| C            | ramen                  |


I assume that by the 'first item from the menu purchased' means we need to order by order_date and product_id
in the window functions, as in case two items were purchased on the same day, we'll pick the one appear in the menu first, thus, entering
the use of ROW_NUMBER(). Danny's solution used RANK() instead, meaning that the order the item appears in the menu doesn't matter.

#### Takeaways
- Use SELECT DISTINCT after RANK() to avoid duplicates


<details>
  <summary>Revised solution</summary>

```sql
WITH cte_ranked AS(
SELECT
  customer_id,
  order_date,
  product_name,
  RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS _rank
FROM joined_table)

SELECT DISTINCT -- need to add DISTINCT as otherwise get duplicate records when they're in the same rank.
  customer_id,
  product_name AS first_item_purchased
FROM cte_ranked
WHERE _rank = 1;
```
</details>

| customer\_id | first\_item\_purchased |
| ------------ | ---------------------- |
| A            | curry                  |
| A            | sushi                  |
| B            | curry                  |
| C            | ramen                  |




### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

<details>
  <summary>My solution</summary>

```sql
SELECT
  product_name,
  COUNT(*) AS times_purchased
FROM joined_table
GROUP BY product_name
ORDER BY times_purchased DESC
LIMIT 1;
```
</details>


| product\_name | times\_purchased |
| ------------- | ---------------- |
| ramen         | 8                |


### 5. Which item was the most popular for each customer?

<details>
  <summary>My solution</summary>

```sql
WITH cte_most_popular_items AS(
SELECT
  customer_id,
  product_name,
  COUNT(*) AS times_purchased,
  DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY COUNT(*) DESC) AS _rank_item
FROM joined_table
GROUP BY
  customer_id,
  product_name
ORDER BY
  customer_id,
  times_purchased DESC)

SELECT DISTINCT
  customer_id,
  product_name,
  times_purchased
FROM cte_most_popular_items
WHERE _rank_item = 1;
```
</details>

| customer\_id | product\_name | times\_purchased |
| ------------ | ------------- | ---------------- |
| A            | ramen         | 3                |
| B            | curry         | 2                |
| B            | ramen         | 2                |
| B            | sushi         | 2                |
| C            | ramen         | 3                |



### 6. Which item was purchased first by the customer after they became a member?
If join_date = order_date, can assume that the customer joined then made the order. Thus, we also count items purchased on the join_date


<details>
  <summary>My solution</summary>

```sql
WITH cte_after_join AS(
SELECT *,
  DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS ranked_item_after_join
FROM joined_table
WHERE order_date >= join_date
)

SELECT
  customer_id,
  order_date,
  product_name
FROM cte_after_join
WHERE ranked_item_after_join = 1;
```
</details>


| customer\_id | order\_date              | product\_name |
| ------------ | ------------------------ | ------------- |
| A            | 2021-01-07T00:00:00.000Z | curry         |
| B            | 2021-01-11T00:00:00.000Z | sushi         |

#### Takeaways
- Same as before, use SELECT DISTINCT after RANK(), DENSE_RANK() to avoid duplicates



<details>
  <summary>Revised solution</summary>

```sql
WITH cte_after_join AS(
SELECT *,
  DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS ranked_item_after_join
FROM joined_table
WHERE order_date >= join_date
)

SELECT DISTINCT
  customer_id,
  order_date,
  product_name
FROM cte_after_join
WHERE ranked_item_after_join = 1;
```
</details>




### 7. Which item was purchased just before the customer became a member?

<details>
  <summary>My solution</summary>

```sql
WITH cte_before_join AS(
SELECT *,
  DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS ranked_item_after_join
FROM joined_table
WHERE order_date < join_date
)

SELECT
  customer_id,
  order_date,
  product_name
FROM cte_before_join
WHERE ranked_item_after_join = 1;
```
</details>

| customer\_id | order\_date              | product\_name |
| ------------ | ------------------------ | ------------- |
| A            | 2021-01-01T00:00:00.000Z | sushi         |
| A            | 2021-01-01T00:00:00.000Z | curry         |
| B            | 2021-01-01T00:00:00.000Z | curry         |

#### Takeaways
This output is incorrect as the question asked me to select the item purchased on the date just before customers joined. I should have used  ORDER BY order_date DESC in the window function.

<details>
  <summary>Revised solution</summary>

```sql
WITH cte_before_join AS(
SELECT *,
  DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date DESC) AS ranked_item_after_join
FROM joined_table
WHERE order_date < join_date
)

SELECT DISTINCT
  customer_id,
  order_date,
  product_name
FROM cte_before_join
WHERE ranked_item_after_join = 1;
```
</details>

| customer\_id | order\_date              | product\_name |
| ------------ | ------------------------ | ------------- |
| A            | 2021-01-01T00:00:00.000Z | curry         |
| A            | 2021-01-01T00:00:00.000Z | sushi         |
| B            | 2021-01-04T00:00:00.000Z | sushi         |





### 8. What is the total items and amount spent for each member before they became a member?
<details>
  <summary>My solution</summary>

```sql
SELECT
  customer_id,
  COUNT(DISTINCT product_id) AS total_unique_items, -- assuming the question asked for count of unique items
  SUM(price) AS total_amount_spent
FROM joined_table
WHERE order_date < join_date
GROUP BY customer_id;
```
</details>

| customer\_id | total\_unique\_items | total\_amount\_spent |
| ------------ | -------------------- | -------------------- |
| A            | 2                    | 25                   |
| B            | 2                    | 40                   |

### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

<details>
  <summary>My solution</summary>

```sql
WITH cte_points AS(
SELECT
  customer_id,
  product_name,
  CASE
    WHEN product_name = 'sushi' THEN price * 10 * 2
    ELSE price * 10 END AS points
FROM joined_table)

SELECT
  customer_id,
  SUM(points) AS total_points
FROM cte_points
GROUP BY customer_id
ORDER BY customer_id;
```
</details>

| customer\_id | total\_points |
| ------------ | ------------- |
| A            | 860           |
| B            | 940           |
| C            | 360           |

#### Takeaways
- Only select column that I need eg. product_name in cte_points is redundant
- I can directly use SUM(CASE WHEN...) THEN a GROUP BY in one query, instead of using a CTE

<details>
  <summary>Revised solution</summary>

```sql
SELECT
  customer_id,
  SUM(CASE
    WHEN product_name = 'sushi' THEN price * 10 * 2
    ELSE price * 10 END) AS total_points
FROM joined_table
GROUP BY customer_id
ORDER BY customer_id;
```
</details>






### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?


<details>
  <summary>My solution</summary>

```sql
WITH cte_new_points AS(
SELECT
  customer_id,
  order_date,
  join_date,
  product_name,
  CASE
    WHEN order_date BETWEEN join_date AND (join_date + INTERVAL '6 DAYS')::DATE THEN price * 10 * 2
    ELSE price * 10 END AS points
FROM joined_table)

SELECT
  customer_id,
  SUM(points) AS new_points
FROM cte_new_points
WHERE join_date IS NOT NULL AND order_date < '2021-02-01'
GROUP BY customer_id
ORDER BY customer_id;
```
</details>

| customer\_id | new\_points |
| ------------ | ----------- |
| A            | 1270        |
| B            | 720         |

#### Takeaways
- Need to check the logic in CASE WHEN carefully. I forgot to include conditions for sushi e.g. sushi still has 2x points outside of first week as provided from Question 9.
- Use SUM(CASE WHEN) with GROUP BY to avoid the need for a CTE
- Use number * col_name in calculation e.g. '10 * price' instead of 'price * 10'  for readability

<details>
  <summary>Revised solution</summary>

```sql
SELECT
  customer_id,
  SUM(
    CASE
      WHEN product_name = 'sushi' THEN 2 * 10 * price
      WHEN order_date BETWEEN join_date AND (join_date + INTERVAL '6 DAYS')::DATE THEN 2 * 10 * price
      ELSE 10 * price
    END
  ) AS points
FROM joined_table
WHERE join_date IS NOT NULL AND order_date < '2021-02-01'::DATE
GROUP BY customer_id
ORDER BY customer_id;
```
</details>

| customer\_id | points |
| ------------ | ------ |
| A            | 1370   |
| B            | 820    |


### 11. Recreate the following table output using the available data

| customer\_id | order\_date | product\_name | price | member |
| ------------ | ----------- | ------------- | ----- | ------ |
| A            | 1/01/2021   | curry         | 15    | N      |
| A            | 1/01/2021   | sushi         | 10    | N      |
| A            | 7/01/2021   | curry         | 15    | Y      |
| A            | 10/01/2021  | ramen         | 12    | Y      |
| A            | 11/01/2021  | ramen         | 12    | Y      |
| A            | 11/01/2021  | ramen         | 12    | Y      |
| B            | 1/01/2021   | curry         | 15    | N      |
| B            | 2/01/2021   | curry         | 15    | N      |
| B            | 4/01/2021   | sushi         | 10    | N      |
| B            | 11/01/2021  | sushi         | 10    | Y      |
| B            | 16/01/2021  | ramen         | 12    | Y      |
| B            | 1/02/2021   | ramen         | 12    | Y      |
| C            | 1/01/2021   | ramen         | 12    | N      |
| C            | 1/01/2021   | ramen         | 12    | N      |
| C            | 7/01/2021   | ramen         | 12    | N      |

<details>
  <summary>My solution</summary>

```sql
DROP TABLE IF EXISTS all_joins;
CREATE TEMP TABLE all_joins AS

WITH cte_table_joint AS(
SELECT
  sales.customer_id,
  sales.order_date,
  sales.product_id,
  menu.product_name,
  menu.price,
  members.join_date
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
  ON menu.product_id = sales.product_id
LEFT JOIN dannys_diner.members
  ON members.customer_id = sales.customer_id
ORDER BY customer_id, order_date, product_name
),
final_output AS(
SELECT
  customer_id,
  order_date,
  product_name,
  price,
  CASE
    WHEN order_date >= join_date THEN 'Y'
    ELSE 'N' END AS member
FROM cte_table_joint)

SELECT * FROM final_output;
```
</details>




### 12. Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.

| customer\_id | order\_date | product\_name | price | member | ranking |
| ------------ | ----------- | ------------- | ----- | ------ | ------- |
| A            | 1/01/2021   | curry         | 15    | N      | null    |
| A            | 1/01/2021   | sushi         | 10    | N      | null    |
| A            | 7/01/2021   | curry         | 15    | Y      | 1       |
| A            | 10/01/2021  | ramen         | 12    | Y      | 2       |
| A            | 11/01/2021  | ramen         | 12    | Y      | 3       |
| A            | 11/01/2021  | ramen         | 12    | Y      | 3       |
| B            | 1/01/2021   | curry         | 15    | N      | null    |
| B            | 2/01/2021   | curry         | 15    | N      | null    |
| B            | 4/01/2021   | sushi         | 10    | N      | null    |
| B            | 11/01/2021  | sushi         | 10    | Y      | 1       |
| B            | 16/01/2021  | ramen         | 12    | Y      | 2       |
| B            | 1/02/2021   | ramen         | 12    | Y      | 3       |
| C            | 1/01/2021   | ramen         | 12    | N      | null    |
| C            | 1/01/2021   | ramen         | 12    | N      | null    |
| C            | 7/01/2021   | ramen         | 12    | N      | null    |

<details>
  <summary>My solution</summary>

```sql
WITH cte AS(
SELECT
  *,
  DENSE_RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date) AS all_ranking
FROM all_joins)

SELECT
  customer_id,
  order_date,
  product_name,
  price,
  member,
  CASE
    WHEN member = 'N' THEN NULL
    ELSE all_ranking END AS ranking
FROM cte;
```
</details>

#### Takeaways
- Redundant CTE i.e. Can use window function directly in the following query

<details>
  <summary>Revised solution</summary>

```sql
WITH joint_sales AS (
SELECT
  sales.customer_id,
  sales.order_date,
  menu.product_name,
  menu.price,
  CASE
    WHEN sales.order_date >= members.join_date::DATE THEN 'Y'
    ELSE 'N'
  END AS member
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
  ON sales.product_id = menu.product_id
LEFT JOIN dannys_diner.members
  ON sales.customer_id = members.customer_id
)
SELECT
  customer_id,
  order_date,
  product_name,
  price,
  member,
  CASE
    WHEN member = 'N' THEN NULL
    ELSE RANK() OVER (
      PARTITION BY customer_id, member
      ORDER BY order_date
      -- default is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )
  END AS ranking
FROM joint_sales
ORDER BY
  customer_id,
  order_date,
  product_name;
```
</details>

## Big Takeaways
- Use SELECT DISTINCT after RANK(), DENSE_RANK() when needed to avoid duplicates
- Only select the required columns
- Check the CASE WHEN logic carefully
- Read the questions carefully
- Check if a CTE is really needed e.g. I can directly use SUM(CASE WHEN...) THEN a GROUP BY in one query, instead of use a CTE for the CASE WHEN then use SUM() with GROUP BY in the following query
- Use number * col_name in calculation e.g. '10 * price' instead of 'price * 10' for readability

---
&copy; 2022 Yen Chi Nguyen
