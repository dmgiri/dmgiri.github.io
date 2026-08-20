---
aliases:
  - DE SQL Week 2
  - SQL Aggregations Week 2
tags:
  - data-engineering
  - sql
  - sql-aggregations
  - data-quality
  - interview-preparation
  - week-2
status: active
difficulty: beginner
study_time: 6 days
created: 2026-08-16
previous: '[[Data-Engineer-SQL-Week-1-Obsidian]]'
---

# Data Engineer SQL — Week 2 Detailed Notes

> [!info] Week 2 goal
> Convert detailed rows into trustworthy metrics using aggregate functions, grouping, conditional logic, and correct NULL handling. Complete the exercises before opening the solutions.

## Obsidian navigation

- [[#Day 1 — Aggregate functions|Day 1 — Aggregate functions]]
- [[#Day 2 — GROUP BY|Day 2 — GROUP BY]]
- [[#Day 3 — WHERE versus HAVING|Day 3 — WHERE versus HAVING]]
- [[#Day 4 — CASE WHEN and conditional aggregation|Day 4 — CASE WHEN]]
- [[#Day 5 — NULL handling and safe calculations|Day 5 — NULL handling]]
- [[#Day 6 — Daily metrics and data-quality reporting|Day 6 — Data-quality reporting]]
- [[#Week 2 interview questions|Interview questions]]
- [[#Week 2 final practice set|Final practice set]]
- [[#Week 2 one-page cheat sheet|Cheat sheet]]

## Progress tracker

- [ ] Day 1 completed
- [ ] Day 2 completed
- [ ] Day 3 completed
- [ ] Day 4 completed
- [ ] Day 5 completed
- [ ] Day 6 completed
- [ ] Mini-project completed
- [ ] Week 2 completion test passed

> [!tip] Obsidian workflow
> Add your own answer below each exercise using a collapsible heading or callout. Link difficult concepts to a separate note such as `[[SQL NULL Logic]]` or `[[Conditional Aggregation]]`.

**Level:** Beginner  
**Duration:** 6 study days plus 1 revision/rest day  
**Daily time:** 60–90 minutes  
**Primary dialect:** PostgreSQL-style SQL  
**Prerequisite:** [[Data-Engineer-SQL-Week-1-Obsidian|Week 1 — SQL fundamentals]]

## Week 2 learning outcomes

By the end of this week, you should be able to:

- Use `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX` correctly.
- Explain how aggregates handle NULL values.
- Group data by one or more dimensions.
- State the output grain of an aggregated query.
- Explain the difference between `WHERE` and `HAVING`.
- Build business rules with `CASE WHEN`.
- Create several metrics in one scan using conditional aggregation.
- Prevent divide-by-zero with `NULLIF`.
- Use `COALESCE` without changing business meaning accidentally.
- Build daily sales and source-quality reports.
- Detect missing, duplicate, invalid, and unexpected data using SQL metrics.

---

# Practice data

Use the `customers`, `products`, and `orders` tables created in Week 1.

## Optional Week 2 extension data

The following extra rows make grouping exercises more interesting. Insert them only once.

```sql
INSERT INTO orders VALUES
(1013, 1, '2026-08-01', 'COMPLETED',  5000.00, 'WEB',   '2026-08-01 11:00:00'),
(1014, 2, '2026-08-01', 'CANCELLED',  2200.00, 'APP',   '2026-08-01 12:00:00'),
(1015, 4, '2026-08-02', 'COMPLETED', 11000.00, 'APP',   '2026-08-02 14:00:00'),
(1016, 6, '2026-08-02', 'COMPLETED',  7500.00, 'WEB',   '2026-08-02 15:00:00'),
(1017, 8, '2026-08-03', 'PENDING',       NULL, 'STORE', '2026-08-03 09:00:00'),
(1018, 3, '2026-08-03', 'SHIPPED',    4200.00, 'APP',   '2026-08-03 10:30:00'),
(1019, 1, '2026-08-04', 'COMPLETED',  1800.00, 'WEB',   '2026-08-04 16:00:00'),
(1020, NULL, '2026-08-04', 'COMPLETED', 950.00, NULL,  '2026-08-04 17:00:00');
```

> [!warning] Rerun safety
> Because `order_id` is a primary key, running the insert twice will fail. Production loading processes should be idempotent, meaning a retry does not create duplicate results.

## Table grains

| Table | Grain | Candidate key |
|---|---|---|
| `customers` | One row per customer | `customer_id` |
| `products` | One row per product | `product_id` |
| `orders` | One row per order | `order_id` |

When an aggregate query groups orders by date, its output grain becomes one row per order date. Grain changes when grouping changes.

---

# Day 1 — Aggregate functions

## 1.1 What is aggregation?

Aggregation summarizes multiple input rows into one result value or one result per group.

Examples:

- Number of orders
- Total completed revenue
- Average order value
- Earliest order date
- Latest source update timestamp

Without `GROUP BY`, an aggregate query normally returns one result row for all filtered input rows.

## 1.2 COUNT star

`COUNT(*)` counts rows, including rows containing NULL values.

```sql
SELECT COUNT(*) AS order_count
FROM orders;
```

Data Engineer use cases:

- Source row count
- Target row count
- Batch count
- Count reconciliation

## 1.3 COUNT column

`COUNT(column)` counts only rows where that column is not NULL.

```sql
SELECT COUNT(order_total) AS orders_with_total
FROM orders;
```

Compare row count with populated values:

```sql
SELECT COUNT(*) AS total_rows,
       COUNT(order_total) AS populated_totals,
       COUNT(*) - COUNT(order_total) AS missing_totals
FROM orders;
```

This is a useful data-quality pattern.

## 1.4 COUNT DISTINCT

Count unique non-NULL values:

```sql
SELECT COUNT(DISTINCT customer_id) AS ordering_customers
FROM orders;
```

Important points:

- NULL is not counted by `COUNT(DISTINCT column)`.
- Exact syntax for multiple distinct columns varies by SQL engine.
- On very large distributed data, exact distinct counts can be expensive.
- Some platforms provide approximate distinct-count functions for faster estimates.

## 1.5 SUM

`SUM` adds non-NULL numeric values.

```sql
SELECT SUM(order_total) AS total_order_value
FROM orders;
```

Completed revenue:

```sql
SELECT SUM(order_total) AS completed_revenue
FROM orders
WHERE order_status = 'COMPLETED';
```

Do not assume every stored order amount should be counted as revenue. Define which statuses represent recognized revenue.

## 1.6 AVG

`AVG` calculates the average of non-NULL values.

```sql
SELECT AVG(order_total) AS average_order_value
FROM orders;
```

Conceptually:

```text
SUM(non-NULL amounts) / COUNT(non-NULL amounts)
```

It does not automatically treat NULL as zero.

Compare the meanings:

```sql
SELECT AVG(order_total) AS average_known_total,
       AVG(COALESCE(order_total, 0)) AS average_treating_missing_as_zero
FROM orders;
```

These are different business metrics. The second is only valid if a missing total truly means zero.

## 1.7 MIN and MAX

```sql
SELECT MIN(order_date) AS earliest_order_date,
       MAX(order_date) AS latest_order_date,
       MIN(updated_at) AS earliest_update,
       MAX(updated_at) AS latest_update
FROM orders;
```

Data Engineering uses:

- Validate date ranges.
- Confirm watermark coverage.
- Detect unexpected future dates.
- Check earliest and latest source arrival.

`MIN` and `MAX` can also work with text, but the result follows the database collation and ordering rules.

## 1.8 Multiple aggregates in one query

```sql
SELECT COUNT(*) AS order_count,
       COUNT(DISTINCT customer_id) AS distinct_customers,
       SUM(order_total) AS total_value,
       AVG(order_total) AS average_value,
       MIN(order_total) AS minimum_value,
       MAX(order_total) AS maximum_value
FROM orders;
```

The database can often compute these metrics during one scan.

## 1.9 Aggregates over no matching rows

If no rows match:

- `COUNT` returns `0`.
- `SUM`, `AVG`, `MIN`, and `MAX` normally return NULL.

```sql
SELECT COUNT(*) AS row_count,
       SUM(order_total) AS total_value
FROM orders
WHERE order_status = 'UNKNOWN_STATUS';
```

Use `COALESCE(SUM(...), 0)` only when zero is the correct reporting value.

## 1.10 Aggregate data types and precision

The return type depends on the input type and SQL engine. Important considerations:

- Summing many integers can exceed a small integer type.
- Financial calculations need exact decimal precision.
- Averages may return a higher-precision decimal.
- Rounding should normally happen at the final reporting boundary, not after every intermediate operation.

## Day 1 common mistakes

- Expecting `COUNT(column)` to include NULL.
- Treating NULL order totals as zero without a business rule.
- Counting revenue from cancelled orders.
- Using `COUNT(DISTINCT ...)` without considering cost.
- Rounding intermediate values too early.
- Assuming aggregates always return zero when no rows match.

## Day 1 exercises

1. Count all customers.
2. Count customers with a known email.
3. Calculate the number of missing customer emails.
4. Count distinct cities, excluding NULL automatically.
5. Count all orders.
6. Count distinct customers represented in orders.
7. Calculate total completed order value.
8. Calculate average shipped order value.
9. Return the cheapest and most expensive product.
10. Return the earliest signup date and latest customer update timestamp.
11. Return order count, total, average, minimum, and maximum total in one query.
12. Explain why `AVG(COALESCE(order_total, 0))` can be misleading.

## Day 1 solutions

```sql
-- 1
SELECT COUNT(*) AS customer_count
FROM customers;

-- 2
SELECT COUNT(email) AS customers_with_email
FROM customers;

-- 3
SELECT COUNT(*) - COUNT(email) AS missing_email_count
FROM customers;

-- 4
SELECT COUNT(DISTINCT city) AS distinct_city_count
FROM customers;

-- 5
SELECT COUNT(*) AS order_count
FROM orders;

-- 6
SELECT COUNT(DISTINCT customer_id) AS ordering_customers
FROM orders;

-- 7
SELECT SUM(order_total) AS completed_order_value
FROM orders
WHERE order_status = 'COMPLETED';

-- 8
SELECT AVG(order_total) AS average_shipped_value
FROM orders
WHERE order_status = 'SHIPPED';

-- 9
SELECT MIN(unit_price) AS cheapest_price,
       MAX(unit_price) AS highest_price
FROM products;

-- 10
SELECT MIN(signup_date) AS earliest_signup,
       MAX(updated_at) AS latest_update
FROM customers;

-- 11
SELECT COUNT(*) AS order_count,
       SUM(order_total) AS total_value,
       AVG(order_total) AS average_value,
       MIN(order_total) AS minimum_value,
       MAX(order_total) AS maximum_value
FROM orders;
```

---

# Day 2 — GROUP BY

## 2.1 Why GROUP BY is needed

Without grouping, an aggregate describes all filtered rows. `GROUP BY` divides input rows into groups and calculates aggregates for each group.

Orders by status:

```sql
SELECT order_status,
       COUNT(*) AS order_count
FROM orders
GROUP BY order_status;
```

Output grain: one row per `order_status`.

## 2.2 Grouping by one column

Revenue by sales channel:

```sql
SELECT sales_channel,
       SUM(order_total) AS total_order_value
FROM orders
GROUP BY sales_channel;
```

Rows with NULL channel form their own group.

## 2.3 Grouping by multiple columns

```sql
SELECT order_status,
       sales_channel,
       COUNT(*) AS order_count,
       SUM(order_total) AS total_value
FROM orders
GROUP BY order_status, sales_channel
ORDER BY order_status, sales_channel;
```

Output grain: one row per unique status-channel combination.

Adding a grouping column creates a more detailed grain and usually increases the number of output rows.

## 2.4 Selection rule

In a grouped query, each selected expression should normally be:

- included in `GROUP BY`, or
- calculated using an aggregate function.

Valid:

```sql
SELECT order_status,
       COUNT(*) AS order_count
FROM orders
GROUP BY order_status;
```

Invalid in standards-oriented SQL:

```sql
SELECT order_status,
       order_date,
       COUNT(*)
FROM orders
GROUP BY order_status;
```

There are many order dates in each status group, so the engine cannot choose one meaningful date.

Some systems allow non-grouped columns under special settings, but the result can be ambiguous. Do not rely on that behavior.

## 2.5 Grouping by date

Daily metrics:

```sql
SELECT order_date,
       COUNT(*) AS order_count,
       SUM(order_total) AS total_value
FROM orders
GROUP BY order_date
ORDER BY order_date;
```

Monthly metrics in PostgreSQL:

```sql
SELECT DATE_TRUNC('month', order_date) AS month_start,
       COUNT(*) AS order_count,
       SUM(order_total) AS total_value
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month_start;
```

Function syntax differs by platform. In Databricks SQL, verify the current date-truncation function and accepted unit.

## 2.6 Grouping by an expression

Price band summary:

```sql
SELECT CASE
           WHEN unit_price < 1000 THEN 'Low'
           WHEN unit_price < 10000 THEN 'Medium'
           ELSE 'High'
       END AS price_band,
       COUNT(*) AS product_count
FROM products
GROUP BY CASE
             WHEN unit_price < 1000 THEN 'Low'
             WHEN unit_price < 10000 THEN 'Medium'
             ELSE 'High'
         END;
```

Some engines allow the alias `price_band` in `GROUP BY`; others do not. Repeating the expression or using a subquery is more portable.

## 2.7 Grouping NULL values

All NULL values for a grouping expression form one group.

```sql
SELECT COALESCE(sales_channel, 'UNKNOWN') AS channel,
       COUNT(*) AS order_count
FROM orders
GROUP BY COALESCE(sales_channel, 'UNKNOWN');
```

This changes display but does not update source values.

## 2.8 GROUP BY versus DISTINCT

These queries return the same set of status values:

```sql
SELECT DISTINCT order_status
FROM orders;
```

```sql
SELECT order_status
FROM orders
GROUP BY order_status;
```

Use `DISTINCT` when you only need unique combinations. Use `GROUP BY` when calculating aggregates per group.

## 2.9 Grouping by column position

Some engines allow:

```sql
SELECT order_status, COUNT(*)
FROM orders
GROUP BY 1;
```

Although concise, positional grouping is harder to maintain because reordering selected columns can change behavior. Prefer explicit expressions in production SQL.

## 2.10 Grain-first thinking

Before writing an aggregate query, complete this sentence:

```text
The result contains one row per ________.
```

Examples:

- one row per status,
- one row per order date,
- one row per month and sales channel.

If you cannot state the output grain, the query is not ready.

## Day 2 common mistakes

- Selecting a non-aggregated column not present in `GROUP BY`.
- Grouping at the wrong grain.
- Adding unnecessary columns to `GROUP BY` and fragmenting metrics.
- Forgetting that NULL becomes a group.
- Using a date function that differs across SQL engines.
- Assuming `GROUP BY` sorts results. Use `ORDER BY` explicitly.

## Day 2 exercises

1. Count customers by city.
2. Count active and inactive customers.
3. Calculate average credit limit by city.
4. Count products by category.
5. Calculate minimum, maximum, and average price by category.
6. Count orders by status.
7. Calculate order count and total value by sales channel.
8. Calculate completed revenue by order date.
9. Calculate order count by date and status.
10. Group customer signups by month.
11. Display missing city as `Unknown` and count customers.
12. State the output grain of exercises 7, 8, and 9.

## Day 2 solutions

```sql
-- 1
SELECT city, COUNT(*) AS customer_count
FROM customers
GROUP BY city
ORDER BY city;

-- 2
SELECT is_active, COUNT(*) AS customer_count
FROM customers
GROUP BY is_active;

-- 3
SELECT city, AVG(credit_limit) AS average_credit_limit
FROM customers
GROUP BY city;

-- 4
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category
ORDER BY category;

-- 5
SELECT category,
       MIN(unit_price) AS minimum_price,
       MAX(unit_price) AS maximum_price,
       AVG(unit_price) AS average_price
FROM products
GROUP BY category;

-- 6
SELECT order_status, COUNT(*) AS order_count
FROM orders
GROUP BY order_status
ORDER BY order_status;

-- 7
SELECT sales_channel,
       COUNT(*) AS order_count,
       SUM(order_total) AS total_value
FROM orders
GROUP BY sales_channel
ORDER BY sales_channel;

-- 8
SELECT order_date,
       SUM(order_total) AS completed_revenue
FROM orders
WHERE order_status = 'COMPLETED'
GROUP BY order_date
ORDER BY order_date;

-- 9
SELECT order_date,
       order_status,
       COUNT(*) AS order_count
FROM orders
GROUP BY order_date, order_status
ORDER BY order_date, order_status;

-- 10
SELECT DATE_TRUNC('month', signup_date) AS signup_month,
       COUNT(*) AS signup_count
FROM customers
GROUP BY DATE_TRUNC('month', signup_date)
ORDER BY signup_month;

-- 11
SELECT COALESCE(city, 'Unknown') AS city,
       COUNT(*) AS customer_count
FROM customers
GROUP BY COALESCE(city, 'Unknown')
ORDER BY city;
```

Grain answers:

- Exercise 7: one row per sales channel.
- Exercise 8: one row per order date containing completed orders.
- Exercise 9: one row per order date and order status combination.

---

# Day 3 — WHERE versus HAVING

## 3.1 Core difference

- `WHERE` filters detail rows before grouping.
- `HAVING` filters groups after aggregation.

```sql
SELECT order_status,
       COUNT(*) AS order_count
FROM orders
WHERE order_date >= '2026-07-01'
GROUP BY order_status
HAVING COUNT(*) >= 2;
```

Process conceptually:

1. Read `orders`.
2. Keep orders from 1 July onward.
3. Group remaining rows by status.
4. Count each group.
5. Keep groups whose count is at least two.
6. Produce selected columns.

## 3.2 Logical query processing order

For the topics learned so far:

1. `FROM`
2. `WHERE`
3. `GROUP BY`
4. `HAVING`
5. `SELECT`
6. `DISTINCT`
7. `ORDER BY`
8. `LIMIT`

This explains why an aggregate cannot normally appear in `WHERE`:

```sql
-- Invalid
SELECT order_status, COUNT(*)
FROM orders
WHERE COUNT(*) > 2
GROUP BY order_status;
```

The groups and counts do not exist when `WHERE` is evaluated.

Correct:

```sql
SELECT order_status, COUNT(*) AS order_count
FROM orders
GROUP BY order_status
HAVING COUNT(*) > 2;
```

## 3.3 WHERE filters source rows

Revenue by channel for completed orders only:

```sql
SELECT sales_channel,
       SUM(order_total) AS completed_revenue
FROM orders
WHERE order_status = 'COMPLETED'
GROUP BY sales_channel;
```

Cancelled and pending rows never enter the groups.

## 3.4 HAVING filters aggregate results

Channels with completed revenue above 10,000:

```sql
SELECT sales_channel,
       SUM(order_total) AS completed_revenue
FROM orders
WHERE order_status = 'COMPLETED'
GROUP BY sales_channel
HAVING SUM(order_total) > 10000;
```

## 3.5 Use both together

Customers with at least two completed orders during July:

```sql
SELECT customer_id,
       COUNT(*) AS completed_order_count,
       SUM(order_total) AS completed_revenue
FROM orders
WHERE order_status = 'COMPLETED'
  AND order_date >= '2026-07-01'
  AND order_date <  '2026-08-01'
GROUP BY customer_id
HAVING COUNT(*) >= 2;
```

## 3.6 Row filters belong in WHERE

Some engines allow non-aggregate conditions in `HAVING`, but row-level conditions should generally use `WHERE`:

```sql
-- Preferred
WHERE order_status = 'COMPLETED'
```

Benefits:

- Clear intent
- Less input data before aggregation
- Better chance of partition pruning or other optimization
- More portable behavior

## 3.7 HAVING with multiple conditions

```sql
SELECT customer_id,
       COUNT(*) AS order_count,
       SUM(order_total) AS total_value
FROM orders
WHERE customer_id IS NOT NULL
GROUP BY customer_id
HAVING COUNT(*) >= 2
   AND SUM(order_total) >= 5000;
```

## 3.8 Alias availability in HAVING

Some engines allow a selected alias in `HAVING`; others require the full aggregate expression.

Portable approach:

```sql
HAVING SUM(order_total) >= 5000
```

Instead of relying on:

```sql
HAVING total_value >= 5000
```

## Day 3 common mistakes

- Using an aggregate in `WHERE`.
- Using `HAVING` for a simple row filter.
- Filtering status after grouping when only one status should contribute to revenue.
- Forgetting the difference between rows and groups.
- Relying on a `SELECT` alias in `HAVING` across engines.
- Using `<= end_timestamp` instead of a safe half-open range.

## Day 3 exercises

1. Return cities containing at least two customers.
2. Return categories with an average product price above 5,000.
3. Return statuses containing more than two orders.
4. Return channels with total known order value above 10,000.
5. Return customers with at least two orders.
6. Return customers with at least two completed orders.
7. Return dates whose completed revenue exceeds 10,000.
8. Return July statuses with at least two orders.
9. Return product categories containing at least two in-stock products.
10. Explain which conditions belong in `WHERE` and which belong in `HAVING` for exercise 9.

## Day 3 solutions

```sql
-- 1
SELECT city, COUNT(*) AS customer_count
FROM customers
GROUP BY city
HAVING COUNT(*) >= 2;

-- 2
SELECT category, AVG(unit_price) AS average_price
FROM products
GROUP BY category
HAVING AVG(unit_price) > 5000;

-- 3
SELECT order_status, COUNT(*) AS order_count
FROM orders
GROUP BY order_status
HAVING COUNT(*) > 2;

-- 4
SELECT sales_channel, SUM(order_total) AS total_value
FROM orders
GROUP BY sales_channel
HAVING SUM(order_total) > 10000;

-- 5
SELECT customer_id, COUNT(*) AS order_count
FROM orders
WHERE customer_id IS NOT NULL
GROUP BY customer_id
HAVING COUNT(*) >= 2;

-- 6
SELECT customer_id, COUNT(*) AS completed_order_count
FROM orders
WHERE customer_id IS NOT NULL
  AND order_status = 'COMPLETED'
GROUP BY customer_id
HAVING COUNT(*) >= 2;

-- 7
SELECT order_date, SUM(order_total) AS completed_revenue
FROM orders
WHERE order_status = 'COMPLETED'
GROUP BY order_date
HAVING SUM(order_total) > 10000;

-- 8
SELECT order_status, COUNT(*) AS order_count
FROM orders
WHERE order_date >= '2026-07-01'
  AND order_date <  '2026-08-01'
GROUP BY order_status
HAVING COUNT(*) >= 2;

-- 9
SELECT category, COUNT(*) AS in_stock_product_count
FROM products
WHERE stock_quantity > 0
GROUP BY category
HAVING COUNT(*) >= 2;
```

Exercise 9 uses `WHERE stock_quantity > 0` because it filters product rows before grouping. `HAVING COUNT(*) >= 2` filters category groups after counting.

---

# Day 4 — CASE WHEN and conditional aggregation

## 4.1 What is CASE?

`CASE` returns a value based on conditions. It can be used in `SELECT`, `ORDER BY`, `GROUP BY`, and aggregate expressions.

Searched `CASE` syntax:

```sql
CASE
    WHEN condition_1 THEN result_1
    WHEN condition_2 THEN result_2
    ELSE default_result
END
```

## 4.2 Create categories

```sql
SELECT order_id,
       order_total,
       CASE
           WHEN order_total IS NULL THEN 'Unknown'
           WHEN order_total < 5000 THEN 'Small'
           WHEN order_total < 20000 THEN 'Medium'
           ELSE 'Large'
       END AS order_size
FROM orders;
```

Conditions are checked from top to bottom. The first matching branch wins.

## 4.3 Condition order matters

Incorrect:

```sql
CASE
    WHEN order_total >= 5000 THEN 'Medium'
    WHEN order_total >= 20000 THEN 'Large'
    ELSE 'Small'
END
```

An amount of 30,000 matches the first condition and is labeled `Medium`. Test the most specific or highest threshold first:

```sql
CASE
    WHEN order_total >= 20000 THEN 'Large'
    WHEN order_total >= 5000 THEN 'Medium'
    ELSE 'Small'
END
```

## 4.4 Simple CASE

Use simple `CASE` for equality against one expression:

```sql
SELECT order_id,
       CASE order_status
           WHEN 'COMPLETED' THEN 'Successful'
           WHEN 'SHIPPED' THEN 'In progress'
           WHEN 'PENDING' THEN 'In progress'
           WHEN 'CANCELLED' THEN 'Unsuccessful'
           ELSE 'Unknown'
       END AS status_group
FROM orders;
```

Searched `CASE` is more flexible because it supports ranges and multiple conditions.

## 4.5 Always consider ELSE

If no branch matches and `ELSE` is absent, `CASE` returns NULL.

An explicit `ELSE 'Unknown'` can expose new or invalid source values. Avoid forcing an unexpected status into an existing category.

## 4.6 Conditional counting

Count several statuses in one row:

```sql
SELECT COUNT(*) AS total_orders,
       SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_orders,
       SUM(CASE WHEN order_status = 'SHIPPED' THEN 1 ELSE 0 END) AS shipped_orders,
       SUM(CASE WHEN order_status = 'PENDING' THEN 1 ELSE 0 END) AS pending_orders,
       SUM(CASE WHEN order_status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_orders
FROM orders;
```

This pattern is called **conditional aggregation**.

## 4.7 Conditional sums

```sql
SELECT order_date,
       SUM(CASE
               WHEN order_status = 'COMPLETED'
               THEN order_total
               ELSE 0
           END) AS completed_revenue,
       SUM(CASE
               WHEN order_status = 'CANCELLED'
               THEN order_total
               ELSE 0
           END) AS cancelled_value
FROM orders
GROUP BY order_date
ORDER BY order_date;
```

## 4.8 Conditional distinct counts

```sql
SELECT COUNT(DISTINCT CASE
                          WHEN order_status = 'COMPLETED'
                          THEN customer_id
                      END) AS completed_customers
FROM orders;
```

Rows not matching the condition produce NULL, and `COUNT(DISTINCT ...)` ignores NULL.

## 4.9 Conditional average

```sql
SELECT AVG(CASE
               WHEN order_status = 'COMPLETED'
               THEN order_total
           END) AS average_completed_value
FROM orders;
```

No `ELSE 0` is used because non-completed rows should not enter the average denominator.

This distinction is important:

```sql
AVG(CASE WHEN condition THEN value END)
```

calculates the average of matching values, while:

```sql
AVG(CASE WHEN condition THEN value ELSE 0 END)
```

includes every row and inserts zero for non-matches.

## 4.10 Data-quality flags

```sql
SELECT order_id,
       CASE
           WHEN customer_id IS NULL THEN 'MISSING_CUSTOMER'
           WHEN order_total IS NULL THEN 'MISSING_TOTAL'
           WHEN order_total < 0 THEN 'NEGATIVE_TOTAL'
           WHEN sales_channel IS NULL THEN 'MISSING_CHANNEL'
           ELSE 'VALID'
       END AS quality_status
FROM orders;
```

This returns only the first detected issue. If a row can have multiple issues, create separate flags or an array/list depending on platform capability.

## 4.11 PostgreSQL FILTER alternative

PostgreSQL supports aggregate filtering:

```sql
SELECT COUNT(*) FILTER (WHERE order_status = 'COMPLETED') AS completed_orders
FROM orders;
```

`SUM(CASE WHEN ...)` is more portable and important for interviews. Check whether the target engine supports `FILTER` or provides functions such as `COUNT_IF`.

## Day 4 common mistakes

- Placing thresholds in the wrong order.
- Omitting `ELSE` without considering unmatched values.
- Using `ELSE 0` inside an average when non-matching rows should be excluded.
- Returning mixed incompatible data types from branches.
- Counting nullable expressions without understanding NULL behavior.
- Using one quality label when several issues must be reported.

## Day 4 exercises

1. Label products as `Budget`, `Standard`, or `Premium` using appropriate price thresholds.
2. Label customers as `Active` or `Inactive`.
3. Map order statuses into `Successful`, `In progress`, and `Unsuccessful`.
4. Count active and inactive customers in one row.
5. Count each order status in one row.
6. Calculate completed and shipped values in separate columns.
7. Calculate daily web, app, and store order counts.
8. Count orders with missing customer ID, total, or channel.
9. Count distinct customers who completed an order.
10. Calculate average completed order value without adding zeros for other statuses.
11. Create separate quality flags for missing customer, total, and channel.
12. Explain why condition order matters in a range-based `CASE`.

## Day 4 solutions

```sql
-- 1
SELECT product_id,
       product_name,
       unit_price,
       CASE
           WHEN unit_price < 1000 THEN 'Budget'
           WHEN unit_price < 10000 THEN 'Standard'
           ELSE 'Premium'
       END AS price_band
FROM products;

-- 2
SELECT customer_id,
       customer_name,
       CASE WHEN is_active = TRUE THEN 'Active' ELSE 'Inactive' END AS activity_status
FROM customers;

-- 3
SELECT order_id,
       CASE
           WHEN order_status = 'COMPLETED' THEN 'Successful'
           WHEN order_status IN ('SHIPPED', 'PENDING') THEN 'In progress'
           WHEN order_status = 'CANCELLED' THEN 'Unsuccessful'
           ELSE 'Unknown'
       END AS status_group
FROM orders;

-- 4
SELECT SUM(CASE WHEN is_active = TRUE THEN 1 ELSE 0 END) AS active_customers,
       SUM(CASE WHEN is_active = FALSE THEN 1 ELSE 0 END) AS inactive_customers
FROM customers;

-- 5
SELECT SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed,
       SUM(CASE WHEN order_status = 'SHIPPED' THEN 1 ELSE 0 END) AS shipped,
       SUM(CASE WHEN order_status = 'PENDING' THEN 1 ELSE 0 END) AS pending,
       SUM(CASE WHEN order_status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled
FROM orders;

-- 6
SELECT SUM(CASE WHEN order_status = 'COMPLETED' THEN order_total ELSE 0 END) AS completed_value,
       SUM(CASE WHEN order_status = 'SHIPPED' THEN order_total ELSE 0 END) AS shipped_value
FROM orders;

-- 7
SELECT order_date,
       SUM(CASE WHEN sales_channel = 'WEB' THEN 1 ELSE 0 END) AS web_orders,
       SUM(CASE WHEN sales_channel = 'APP' THEN 1 ELSE 0 END) AS app_orders,
       SUM(CASE WHEN sales_channel = 'STORE' THEN 1 ELSE 0 END) AS store_orders
FROM orders
GROUP BY order_date
ORDER BY order_date;

-- 8
SELECT SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS missing_customer,
       SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END) AS missing_total,
       SUM(CASE WHEN sales_channel IS NULL THEN 1 ELSE 0 END) AS missing_channel
FROM orders;

-- 9
SELECT COUNT(DISTINCT CASE
                          WHEN order_status = 'COMPLETED' THEN customer_id
                      END) AS completed_customers
FROM orders;

-- 10
SELECT AVG(CASE
               WHEN order_status = 'COMPLETED' THEN order_total
           END) AS average_completed_value
FROM orders;

-- 11
SELECT order_id,
       CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END AS missing_customer_flag,
       CASE WHEN order_total IS NULL THEN 1 ELSE 0 END AS missing_total_flag,
       CASE WHEN sales_channel IS NULL THEN 1 ELSE 0 END AS missing_channel_flag
FROM orders;
```

---

# Day 5 — NULL handling and safe calculations

## 5.1 NULL recap

NULL represents missing or unknown information. It does not equal zero, an empty string, or another NULL.

```sql
WHERE order_total IS NULL
```

not:

```sql
WHERE order_total = NULL
```

## 5.2 Three-valued logic

SQL predicates can be true, false, or unknown. `WHERE` keeps only true rows.

| Expression when `order_total` is NULL | Result |
|---|---|
| `order_total = 0` | Unknown |
| `order_total <> 0` | Unknown |
| `order_total > 100` | Unknown |
| `order_total IS NULL` | True |
| `order_total IS NOT NULL` | False |

This query excludes both zero and NULL values:

```sql
SELECT *
FROM orders
WHERE order_total <> 0;
```

To include missing totals explicitly:

```sql
WHERE order_total <> 0
   OR order_total IS NULL
```

## 5.3 COALESCE

`COALESCE` returns the first non-NULL expression.

```sql
SELECT customer_name,
       COALESCE(city, 'Unknown') AS displayed_city
FROM customers;
```

Fallback chain:

```sql
SELECT order_id,
       COALESCE(sales_channel, 'UNASSIGNED') AS channel
FROM orders;
```

### Presentation versus calculation

It is often safe to display missing text as `Unknown`. Replacing missing numeric data with zero can change metrics.

```sql
SELECT AVG(order_total) AS average_known_total,
       AVG(COALESCE(order_total, 0)) AS average_after_zero_fill
FROM orders;
```

Document the business rule whenever you fill missing values.

## 5.4 NULLIF

`NULLIF(a, b)` returns NULL when `a = b`; otherwise it returns `a`.

Safe division:

```sql
SELECT completed_orders * 100.0 / NULLIF(total_orders, 0) AS completion_rate
FROM some_daily_summary;
```

If `total_orders` is zero, the denominator becomes NULL and the result becomes NULL rather than raising a division-by-zero error.

## 5.5 Safe percentage in one query

```sql
SELECT COUNT(*) AS total_orders,
       SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_orders,
       100.0 * SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END)
           / NULLIF(COUNT(*), 0) AS completion_percentage
FROM orders;
```

Multiplying by `100.0` helps avoid integer division in engines that divide integers as integers.

## 5.6 NULL in arithmetic

Most arithmetic involving NULL returns NULL.

```sql
SELECT 100 + NULL;  -- NULL
```

For inventory value:

```sql
SELECT product_name,
       unit_price * stock_quantity AS inventory_value
FROM products;
```

Unknown stock produces unknown inventory value. This is usually honest. Converting it to zero could incorrectly imply there is no stock.

## 5.7 NULL in aggregates

```sql
SELECT COUNT(*) AS rows,
       COUNT(order_total) AS known_totals,
       SUM(order_total) AS sum_known_totals,
       AVG(order_total) AS average_known_totals
FROM orders;
```

Most aggregates ignore NULL values. `COUNT(*)` counts rows.

## 5.8 Count missing values

Three equivalent patterns for one nullable column:

```sql
SELECT COUNT(*) - COUNT(order_total) AS missing_total_count
FROM orders;
```

```sql
SELECT SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END) AS missing_total_count
FROM orders;
```

PostgreSQL-specific aggregate filter:

```sql
SELECT COUNT(*) FILTER (WHERE order_total IS NULL) AS missing_total_count
FROM orders;
```

The `CASE` form is highly portable and supports more complex rules.

## 5.9 NOT IN and NULL warning

When a `NOT IN` list or subquery contains NULL, comparisons can become unknown and produce unexpected results.

```sql
-- Dangerous if the list/subquery can contain NULL
WHERE customer_id NOT IN (...)
```

For table-to-table anti-matching, later use `NOT EXISTS` or a left anti join. If using `NOT IN`, remove NULLs explicitly and understand the semantics.

## 5.10 NULL-safe equality

Some engines provide NULL-safe comparison operators. PostgreSQL supports:

```sql
a IS NOT DISTINCT FROM b
```

This treats two NULL values as equal for comparison. Other platforms may use `<=>` or a named function. Verify the target engine.

## 5.11 Missing, invalid, and zero are different

For `order_total`:

- NULL: total is missing or unknown.
- `0`: known total is zero.
- Negative: possibly a refund or invalid value depending on the model.
- Positive: ordinary amount.

Do not combine these states until the business definition is clear.

## Day 5 common mistakes

- Replacing all numeric NULLs with zero.
- Forgetting that arithmetic with NULL returns NULL.
- Dividing integer values and losing decimals.
- Dividing by zero.
- Using `NOT IN` when NULL may appear.
- Assuming NULL values are equal.
- Treating missing, invalid, and zero as the same state.

## Day 5 exercises

1. Count missing customer emails.
2. Count missing customer cities.
3. Count missing order totals.
4. Display missing sales channel as `UNASSIGNED`.
5. Return both average known order total and average after treating missing as zero.
6. Calculate the percentage of orders with a known total.
7. Calculate completion percentage safely.
8. Calculate the percentage of customers with an email.
9. Classify order total as `Missing`, `Zero`, `Negative`, or `Positive`.
10. Return products whose stock is either zero or unknown.
11. Calculate known inventory value without pretending unknown stock is zero.
12. Explain why average known total and zero-filled average differ.

## Day 5 solutions

```sql
-- 1
SELECT SUM(CASE WHEN email IS NULL THEN 1 ELSE 0 END) AS missing_email_count
FROM customers;

-- 2
SELECT SUM(CASE WHEN city IS NULL THEN 1 ELSE 0 END) AS missing_city_count
FROM customers;

-- 3
SELECT SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END) AS missing_total_count
FROM orders;

-- 4
SELECT order_id,
       COALESCE(sales_channel, 'UNASSIGNED') AS sales_channel
FROM orders;

-- 5
SELECT AVG(order_total) AS average_known_total,
       AVG(COALESCE(order_total, 0)) AS average_zero_filled_total
FROM orders;

-- 6
SELECT 100.0 * COUNT(order_total) / NULLIF(COUNT(*), 0) AS known_total_percentage
FROM orders;

-- 7
SELECT 100.0 * SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END)
           / NULLIF(COUNT(*), 0) AS completion_percentage
FROM orders;

-- 8
SELECT 100.0 * COUNT(email) / NULLIF(COUNT(*), 0) AS email_percentage
FROM customers;

-- 9
SELECT order_id,
       order_total,
       CASE
           WHEN order_total IS NULL THEN 'Missing'
           WHEN order_total = 0 THEN 'Zero'
           WHEN order_total < 0 THEN 'Negative'
           ELSE 'Positive'
       END AS total_quality
FROM orders;

-- 10
SELECT *
FROM products
WHERE stock_quantity = 0
   OR stock_quantity IS NULL;

-- 11
SELECT product_id,
       product_name,
       CASE
           WHEN stock_quantity IS NULL THEN NULL
           ELSE unit_price * stock_quantity
       END AS inventory_value
FROM products;
```

---

# Day 6 — Daily metrics and data-quality reporting

## 6.1 What makes a useful metric query?

A reliable metric query has:

1. A clear business definition.
2. A declared output grain.
3. Correct status and date filters.
4. Explicit NULL behavior.
5. Deterministic dimensions.
6. Reconciliation checks.
7. Comments for non-obvious rules.

## 6.2 Daily order metrics

Output grain: one row per order date.

```sql
SELECT order_date,
       COUNT(*) AS total_orders,
       COUNT(DISTINCT customer_id) AS distinct_customers,
       SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_orders,
       SUM(CASE WHEN order_status = 'SHIPPED' THEN 1 ELSE 0 END) AS shipped_orders,
       SUM(CASE WHEN order_status = 'PENDING' THEN 1 ELSE 0 END) AS pending_orders,
       SUM(CASE WHEN order_status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_orders,
       SUM(CASE
               WHEN order_status = 'COMPLETED' THEN order_total
               ELSE 0
           END) AS completed_revenue,
       AVG(CASE
               WHEN order_status = 'COMPLETED' THEN order_total
           END) AS average_completed_order_value,
       MIN(updated_at) AS first_update_time,
       MAX(updated_at) AS last_update_time
FROM orders
GROUP BY order_date
ORDER BY order_date;
```

## 6.3 Daily percentage metrics

```sql
SELECT order_date,
       COUNT(*) AS total_orders,
       100.0 * SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END)
           / NULLIF(COUNT(*), 0) AS completion_percentage,
       100.0 * SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END)
           / NULLIF(COUNT(*), 0) AS missing_total_percentage
FROM orders
GROUP BY order_date
ORDER BY order_date;
```

## 6.4 Source data-quality summary

One-row summary for the complete source:

```sql
SELECT COUNT(*) AS row_count,
       COUNT(DISTINCT order_id) AS distinct_order_ids,
       COUNT(*) - COUNT(DISTINCT order_id) AS duplicate_key_excess_rows,
       SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS null_order_id_count,
       SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS null_customer_id_count,
       SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END) AS null_total_count,
       SUM(CASE WHEN sales_channel IS NULL THEN 1 ELSE 0 END) AS null_channel_count,
       SUM(CASE WHEN order_total < 0 THEN 1 ELSE 0 END) AS negative_total_count,
       SUM(CASE
               WHEN order_status NOT IN ('COMPLETED', 'SHIPPED', 'PENDING', 'CANCELLED')
               THEN 1 ELSE 0
           END) AS invalid_status_count,
       MIN(order_date) AS minimum_order_date,
       MAX(order_date) AS maximum_order_date,
       MIN(updated_at) AS minimum_update_timestamp,
       MAX(updated_at) AS maximum_update_timestamp
FROM orders;
```

Because `order_id` is a primary key in the practice table, duplicates and NULL keys cannot be inserted. In a raw landing table, these constraints may not exist; that is where the checks become essential.

## 6.5 Daily data-quality report

Output grain: one row per order date.

```sql
SELECT order_date,
       COUNT(*) AS row_count,
       COUNT(DISTINCT order_id) AS distinct_order_ids,
       COUNT(*) - COUNT(DISTINCT order_id) AS duplicate_key_excess_rows,
       SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS missing_customer_count,
       SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END) AS missing_total_count,
       SUM(CASE WHEN sales_channel IS NULL THEN 1 ELSE 0 END) AS missing_channel_count,
       SUM(CASE WHEN order_total < 0 THEN 1 ELSE 0 END) AS negative_total_count,
       MIN(updated_at) AS first_update,
       MAX(updated_at) AS last_update
FROM orders
GROUP BY order_date
ORDER BY order_date;
```

## 6.6 Domain validation

Inspect actual status values:

```sql
SELECT order_status,
       COUNT(*) AS occurrences
FROM orders
GROUP BY order_status
ORDER BY occurrences DESC, order_status;
```

Find unexpected values:

```sql
SELECT order_status,
       COUNT(*) AS occurrences
FROM orders
WHERE order_status NOT IN ('COMPLETED', 'SHIPPED', 'PENDING', 'CANCELLED')
   OR order_status IS NULL
GROUP BY order_status;
```

## 6.7 Freshness check

```sql
SELECT MAX(updated_at) AS latest_source_update,
       CURRENT_TIMESTAMP AS checked_at,
       CURRENT_TIMESTAMP - MAX(updated_at) AS source_age
FROM orders;
```

Date-difference syntax varies by engine. A production alert should compare source age against an agreed service-level threshold.

## 6.8 Reconciliation report

If you have `source_orders` and `target_orders`, compare at the same grain:

```sql
SELECT order_date,
       COUNT(*) AS source_count,
       SUM(order_total) AS source_total
FROM source_orders
GROUP BY order_date;
```

```sql
SELECT order_date,
       COUNT(*) AS target_count,
       SUM(order_total) AS target_total
FROM target_orders
GROUP BY order_date;
```

Week 3 joins will combine these results into a single difference report.

## 6.9 Metric validation questions

Before publishing daily completed revenue, ask:

- Does completed revenue include tax?
- Are refunds represented as negative orders or another table?
- Which time zone defines the business date?
- Should late-arriving orders update a prior date?
- Can one order appear multiple times in the source?
- Is order total NULL before payment completion?
- Which statuses count as revenue?
- Should test or internal orders be excluded?

SQL syntax can be correct while the metric definition is wrong.

## Day 6 mini-project — Daily operations dashboard

Create one query at daily grain containing:

1. Order date
2. Total order count
3. Distinct known customer count
4. Completed order count
5. Cancelled order count
6. Completion percentage
7. Completed revenue
8. Average completed order value
9. Web order count
10. App order count
11. Store order count
12. Missing customer count
13. Missing total count
14. Missing channel count
15. Latest update timestamp

Then filter the report to dates having either:

- more than one missing important value, or
- completed revenue above 10,000.

## Mini-project solution

```sql
SELECT order_date,
       COUNT(*) AS total_orders,
       COUNT(DISTINCT customer_id) AS distinct_known_customers,
       SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_orders,
       SUM(CASE WHEN order_status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_orders,
       100.0 * SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END)
           / NULLIF(COUNT(*), 0) AS completion_percentage,
       SUM(CASE
               WHEN order_status = 'COMPLETED' THEN order_total
               ELSE 0
           END) AS completed_revenue,
       AVG(CASE
               WHEN order_status = 'COMPLETED' THEN order_total
           END) AS average_completed_order_value,
       SUM(CASE WHEN sales_channel = 'WEB' THEN 1 ELSE 0 END) AS web_orders,
       SUM(CASE WHEN sales_channel = 'APP' THEN 1 ELSE 0 END) AS app_orders,
       SUM(CASE WHEN sales_channel = 'STORE' THEN 1 ELSE 0 END) AS store_orders,
       SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS missing_customer_count,
       SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END) AS missing_total_count,
       SUM(CASE WHEN sales_channel IS NULL THEN 1 ELSE 0 END) AS missing_channel_count,
       MAX(updated_at) AS latest_update_timestamp
FROM orders
GROUP BY order_date
HAVING
       SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END)
     + SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END)
     + SUM(CASE WHEN sales_channel IS NULL THEN 1 ELSE 0 END) > 1
    OR SUM(CASE
               WHEN order_status = 'COMPLETED' THEN order_total
               ELSE 0
           END) > 10000
ORDER BY order_date;
```

> [!note] Quality-count meaning
> The HAVING expression counts missing fields, not necessarily distinct bad rows. One row missing three fields contributes three issues. If the requirement is bad-row count, use one `CASE` with `OR` conditions.

Bad-row count pattern:

```sql
SUM(
    CASE
        WHEN customer_id IS NULL
          OR order_total IS NULL
          OR sales_channel IS NULL
        THEN 1
        ELSE 0
    END
) AS invalid_row_count
```

---

# Week 2 interview questions

## Fundamentals

### 1. What is an aggregate function?

An aggregate function summarizes multiple input rows into one result value per query or per group.

### 2. What is the difference between COUNT star and COUNT column?

`COUNT(*)` counts rows. `COUNT(column)` counts non-NULL values in that column.

### 3. Does COUNT DISTINCT count NULL?

For a single expression, NULL is normally excluded.

### 4. How do SUM and AVG handle NULL?

They ignore NULL input values. `AVG` divides the sum of non-NULL values by their non-NULL count.

### 5. What do aggregates return when no rows match?

`COUNT` returns zero. `SUM`, `AVG`, `MIN`, and `MAX` generally return NULL.

### 6. What is GROUP BY?

It divides rows into groups based on one or more expressions and calculates aggregates for each group.

### 7. What is output grain?

It defines what one result row represents. In a query grouped by date and channel, one row represents one date-channel combination.

### 8. Why must selected non-aggregated columns appear in GROUP BY?

Each output group can contain several source values. Without grouping or aggregation, the engine has no single meaningful value to return.

### 9. Does GROUP BY sort output?

No. Use `ORDER BY` when order is required.

### 10. How are NULL values treated in GROUP BY?

NULL values for the grouping expression form one group.

## WHERE, HAVING, and CASE

### 11. WHERE versus HAVING?

`WHERE` filters input rows before grouping. `HAVING` filters groups after aggregation.

### 12. Can an aggregate function be used in WHERE?

Normally no, because grouping and aggregates are logically evaluated after `WHERE`. Use `HAVING` or an outer query.

### 13. Why place row conditions in WHERE instead of HAVING?

It expresses intent correctly and can reduce data before aggregation, improving optimization opportunities.

### 14. What is CASE WHEN?

It is a conditional expression that returns a value from the first matching branch.

### 15. Why does CASE condition order matter?

The first matching branch wins. A broad condition placed before a specific one can make the specific branch unreachable.

### 16. What happens if CASE has no matching branch and no ELSE?

It returns NULL.

### 17. What is conditional aggregation?

It places conditional expressions inside aggregates to calculate several filtered metrics in one grouped query.

### 18. How do you count completed orders?

Use `SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END)` or an engine-supported conditional-count function.

### 19. Why avoid ELSE zero in a conditional average?

It adds non-matching rows as zeros to the denominator, producing a different metric from the average of matching values only.

## NULL and quality

### 20. What is COALESCE?

It returns the first non-NULL expression in its argument list.

### 21. What is NULLIF?

It returns NULL when two expressions are equal; otherwise it returns the first expression. It is often used to protect division by zero.

### 22. Why can filling NULL with zero be dangerous?

Zero is a known value while NULL is unknown. Replacing NULL can change sums, averages, flags, and business meaning.

### 23. How do you calculate a percentage safely?

Use decimal arithmetic and divide by `NULLIF(denominator, 0)`.

### 24. How do you count missing values?

Use `COUNT(*) - COUNT(column)` or `SUM(CASE WHEN column IS NULL THEN 1 ELSE 0 END)`.

### 25. Why is NOT IN risky with NULL?

If the comparison set contains NULL, the result can become unknown and exclude rows unexpectedly. `NOT EXISTS` is usually safer for anti-matching.

## Data Engineer scenarios

### 26. Source count and target count match. Is the load validated?

No. Counts can match while keys or values differ. Also compare distinct keys, NULLs, duplicates, totals, date ranges, domains, and selected row-level values.

### 27. Revenue increased after a pipeline change. What should you check?

Check status filters, duplicates, grain, join multiplication, late data, currency, refunds, NULL handling, and whether metric definitions changed.

### 28. A daily report omits dates with zero orders. Why?

Grouping the orders table only produces dates that exist in the data. A calendar/date dimension and a left join are needed to display zero-order dates; joins are covered in Week 3.

### 29. COUNT DISTINCT is slow on a huge dataset. What options exist?

Consider approximate distinct functions, pre-aggregation, partition pruning, incremental summaries, or a redesigned metric—depending on the required accuracy and engine.

### 30. What should every metric query document?

Its grain, filters, status rules, NULL treatment, date/time zone, calculation formula, expected freshness, and reconciliation method.

---

# Week 2 final practice set

Solve these without opening the answer key.

## Aggregate fundamentals

1. Count all products.
2. Count products with known stock.
3. Count products with unknown stock.
4. Calculate total known inventory units.
5. Calculate average product price.
6. Return minimum and maximum product creation timestamps.
7. Count distinct product categories.
8. Calculate total customer credit limit using non-NULL values.
9. Return earliest and latest order dates.
10. Count distinct known customers appearing in orders.

## Grouping

11. Count products by category.
12. Calculate total stock by category.
13. Calculate average price by category.
14. Count customers by city and activity status.
15. Count orders by status.
16. Count orders by channel.
17. Calculate completed revenue by date.
18. Calculate order count by month.
19. Calculate average order total by status.
20. Return status-channel groups with their counts.

## HAVING

21. Return product categories with at least two products.
22. Return categories whose average price exceeds 5,000.
23. Return statuses containing at least three orders.
24. Return customers with at least two known-total orders.
25. Return dates with total order value above 10,000.
26. Return channels with at least two completed orders.
27. Return cities with at least two active customers.
28. Return months with completed revenue above 20,000.

## CASE and NULL

29. Classify credit limits as `Missing`, `Low`, `Medium`, or `High`.
30. Count products with zero, positive, and unknown stock in one row.
31. Count completed, non-completed, and unknown-status orders.
32. Calculate web and app values in separate columns.
33. Count orders with any missing important field.
34. Calculate percentage of customers with a city.
35. Calculate cancellation percentage safely.
36. Calculate average completed order total correctly.
37. Display missing channels as `UNASSIGNED` and group counts.
38. Calculate daily missing-total percentage.

## Data Engineering reports

39. Build a one-row customer quality report.
40. Build a one-row product quality report.
41. Build a daily order quality report.
42. Build a monthly order-status summary.
43. Return days with invalid orders.
44. Return source date and update timestamp boundaries.
45. Create a status-domain frequency report.
46. Create a channel-domain frequency report including missing values.
47. Build daily completed revenue and completed customer count.
48. Build a daily operational report with counts, values, and quality metrics.

## Week 2 final answer key

```sql
-- 1
SELECT COUNT(*) AS product_count FROM products;

-- 2
SELECT COUNT(stock_quantity) AS products_with_known_stock FROM products;

-- 3
SELECT COUNT(*) - COUNT(stock_quantity) AS unknown_stock_count FROM products;

-- 4
SELECT SUM(stock_quantity) AS total_known_units FROM products;

-- 5
SELECT AVG(unit_price) AS average_price FROM products;

-- 6
SELECT MIN(created_at) AS earliest_creation,
       MAX(created_at) AS latest_creation
FROM products;

-- 7
SELECT COUNT(DISTINCT category) AS category_count FROM products;

-- 8
SELECT SUM(credit_limit) AS total_known_credit_limit FROM customers;

-- 9
SELECT MIN(order_date) AS earliest_order,
       MAX(order_date) AS latest_order
FROM orders;

-- 10
SELECT COUNT(DISTINCT customer_id) AS ordering_customers FROM orders;

-- 11
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category;

-- 12
SELECT category, SUM(stock_quantity) AS total_known_stock
FROM products
GROUP BY category;

-- 13
SELECT category, AVG(unit_price) AS average_price
FROM products
GROUP BY category;

-- 14
SELECT city, is_active, COUNT(*) AS customer_count
FROM customers
GROUP BY city, is_active;

-- 15
SELECT order_status, COUNT(*) AS order_count
FROM orders
GROUP BY order_status;

-- 16
SELECT sales_channel, COUNT(*) AS order_count
FROM orders
GROUP BY sales_channel;

-- 17
SELECT order_date, SUM(order_total) AS completed_revenue
FROM orders
WHERE order_status = 'COMPLETED'
GROUP BY order_date;

-- 18
SELECT DATE_TRUNC('month', order_date) AS order_month,
       COUNT(*) AS order_count
FROM orders
GROUP BY DATE_TRUNC('month', order_date);

-- 19
SELECT order_status, AVG(order_total) AS average_total
FROM orders
GROUP BY order_status;

-- 20
SELECT order_status, sales_channel, COUNT(*) AS order_count
FROM orders
GROUP BY order_status, sales_channel;

-- 21
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category
HAVING COUNT(*) >= 2;

-- 22
SELECT category, AVG(unit_price) AS average_price
FROM products
GROUP BY category
HAVING AVG(unit_price) > 5000;

-- 23
SELECT order_status, COUNT(*) AS order_count
FROM orders
GROUP BY order_status
HAVING COUNT(*) >= 3;

-- 24
SELECT customer_id, COUNT(order_total) AS known_total_orders
FROM orders
WHERE customer_id IS NOT NULL
GROUP BY customer_id
HAVING COUNT(order_total) >= 2;

-- 25
SELECT order_date, SUM(order_total) AS total_value
FROM orders
GROUP BY order_date
HAVING SUM(order_total) > 10000;

-- 26
SELECT sales_channel, COUNT(*) AS completed_order_count
FROM orders
WHERE order_status = 'COMPLETED'
GROUP BY sales_channel
HAVING COUNT(*) >= 2;

-- 27
SELECT city, COUNT(*) AS active_customer_count
FROM customers
WHERE is_active = TRUE
GROUP BY city
HAVING COUNT(*) >= 2;

-- 28
SELECT DATE_TRUNC('month', order_date) AS order_month,
       SUM(order_total) AS completed_revenue
FROM orders
WHERE order_status = 'COMPLETED'
GROUP BY DATE_TRUNC('month', order_date)
HAVING SUM(order_total) > 20000;

-- 29
SELECT customer_id,
       CASE
           WHEN credit_limit IS NULL THEN 'Missing'
           WHEN credit_limit < 30000 THEN 'Low'
           WHEN credit_limit < 70000 THEN 'Medium'
           ELSE 'High'
       END AS credit_band
FROM customers;

-- 30
SELECT SUM(CASE WHEN stock_quantity = 0 THEN 1 ELSE 0 END) AS zero_stock,
       SUM(CASE WHEN stock_quantity > 0 THEN 1 ELSE 0 END) AS positive_stock,
       SUM(CASE WHEN stock_quantity IS NULL THEN 1 ELSE 0 END) AS unknown_stock
FROM products;

-- 31
SELECT SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed,
       SUM(CASE WHEN order_status <> 'COMPLETED' THEN 1 ELSE 0 END) AS non_completed,
       SUM(CASE WHEN order_status IS NULL THEN 1 ELSE 0 END) AS unknown_status
FROM orders;

-- 32
SELECT SUM(CASE WHEN sales_channel = 'WEB' THEN order_total ELSE 0 END) AS web_value,
       SUM(CASE WHEN sales_channel = 'APP' THEN order_total ELSE 0 END) AS app_value
FROM orders;

-- 33
SELECT SUM(
           CASE
               WHEN customer_id IS NULL
                 OR order_total IS NULL
                 OR sales_channel IS NULL
               THEN 1 ELSE 0
           END
       ) AS invalid_order_count
FROM orders;

-- 34
SELECT 100.0 * COUNT(city) / NULLIF(COUNT(*), 0) AS known_city_percentage
FROM customers;

-- 35
SELECT 100.0 * SUM(CASE WHEN order_status = 'CANCELLED' THEN 1 ELSE 0 END)
           / NULLIF(COUNT(*), 0) AS cancellation_percentage
FROM orders;

-- 36
SELECT AVG(CASE WHEN order_status = 'COMPLETED' THEN order_total END)
           AS average_completed_total
FROM orders;

-- 37
SELECT COALESCE(sales_channel, 'UNASSIGNED') AS channel,
       COUNT(*) AS order_count
FROM orders
GROUP BY COALESCE(sales_channel, 'UNASSIGNED');

-- 38
SELECT order_date,
       100.0 * SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END)
           / NULLIF(COUNT(*), 0) AS missing_total_percentage
FROM orders
GROUP BY order_date;

-- 39
SELECT COUNT(*) AS row_count,
       COUNT(DISTINCT customer_id) AS distinct_keys,
       SUM(CASE WHEN customer_name IS NULL THEN 1 ELSE 0 END) AS missing_name,
       SUM(CASE WHEN email IS NULL THEN 1 ELSE 0 END) AS missing_email,
       SUM(CASE WHEN city IS NULL THEN 1 ELSE 0 END) AS missing_city,
       SUM(CASE WHEN credit_limit < 0 THEN 1 ELSE 0 END) AS negative_credit_limit,
       MIN(signup_date) AS earliest_signup,
       MAX(signup_date) AS latest_signup,
       MAX(updated_at) AS latest_update
FROM customers;

-- 40
SELECT COUNT(*) AS row_count,
       COUNT(DISTINCT product_id) AS distinct_keys,
       SUM(CASE WHEN product_name IS NULL THEN 1 ELSE 0 END) AS missing_name,
       SUM(CASE WHEN category IS NULL THEN 1 ELSE 0 END) AS missing_category,
       SUM(CASE WHEN unit_price IS NULL THEN 1 ELSE 0 END) AS missing_price,
       SUM(CASE WHEN unit_price < 0 THEN 1 ELSE 0 END) AS negative_price,
       SUM(CASE WHEN stock_quantity < 0 THEN 1 ELSE 0 END) AS negative_stock,
       SUM(CASE WHEN stock_quantity IS NULL THEN 1 ELSE 0 END) AS unknown_stock
FROM products;

-- 41
SELECT order_date,
       COUNT(*) AS row_count,
       COUNT(DISTINCT order_id) AS distinct_keys,
       SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS missing_customer,
       SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END) AS missing_total,
       SUM(CASE WHEN sales_channel IS NULL THEN 1 ELSE 0 END) AS missing_channel,
       SUM(CASE WHEN order_total < 0 THEN 1 ELSE 0 END) AS negative_total
FROM orders
GROUP BY order_date;

-- 42
SELECT DATE_TRUNC('month', order_date) AS order_month,
       order_status,
       COUNT(*) AS order_count,
       SUM(order_total) AS total_value
FROM orders
GROUP BY DATE_TRUNC('month', order_date), order_status;

-- 43
SELECT order_date,
       SUM(
           CASE
               WHEN customer_id IS NULL
                 OR order_total IS NULL
                 OR sales_channel IS NULL
               THEN 1 ELSE 0
           END
       ) AS invalid_order_count
FROM orders
GROUP BY order_date
HAVING SUM(
           CASE
               WHEN customer_id IS NULL
                 OR order_total IS NULL
                 OR sales_channel IS NULL
               THEN 1 ELSE 0
           END
       ) > 0;

-- 44
SELECT MIN(order_date) AS earliest_order_date,
       MAX(order_date) AS latest_order_date,
       MIN(updated_at) AS earliest_update,
       MAX(updated_at) AS latest_update
FROM orders;

-- 45
SELECT order_status, COUNT(*) AS occurrences
FROM orders
GROUP BY order_status
ORDER BY occurrences DESC, order_status;

-- 46
SELECT COALESCE(sales_channel, 'UNASSIGNED') AS channel,
       COUNT(*) AS occurrences
FROM orders
GROUP BY COALESCE(sales_channel, 'UNASSIGNED')
ORDER BY occurrences DESC, channel;

-- 47
SELECT order_date,
       SUM(CASE WHEN order_status = 'COMPLETED' THEN order_total ELSE 0 END)
           AS completed_revenue,
       COUNT(DISTINCT CASE
                          WHEN order_status = 'COMPLETED' THEN customer_id
                      END) AS completed_customers
FROM orders
GROUP BY order_date;

-- 48
SELECT order_date,
       COUNT(*) AS total_orders,
       COUNT(DISTINCT customer_id) AS distinct_customers,
       SUM(CASE WHEN order_status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_orders,
       SUM(CASE WHEN order_status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_orders,
       SUM(CASE WHEN order_status = 'COMPLETED' THEN order_total ELSE 0 END)
           AS completed_revenue,
       AVG(CASE WHEN order_status = 'COMPLETED' THEN order_total END)
           AS average_completed_value,
       SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS missing_customer,
       SUM(CASE WHEN order_total IS NULL THEN 1 ELSE 0 END) AS missing_total,
       SUM(CASE WHEN sales_channel IS NULL THEN 1 ELSE 0 END) AS missing_channel,
       MAX(updated_at) AS latest_update
FROM orders
GROUP BY order_date
ORDER BY order_date;
```

---

# Week 2 one-page cheat sheet

## Aggregate functions

```sql
SELECT COUNT(*) AS row_count,
       COUNT(nullable_column) AS populated_count,
       COUNT(DISTINCT business_key) AS distinct_keys,
       SUM(amount) AS total_amount,
       AVG(amount) AS average_amount,
       MIN(event_date) AS earliest_date,
       MAX(event_date) AS latest_date
FROM table_name;
```

## Grouped metrics

```sql
SELECT dimension_1,
       dimension_2,
       COUNT(*) AS row_count,
       SUM(amount) AS total_amount
FROM table_name
WHERE row_condition
GROUP BY dimension_1, dimension_2
HAVING SUM(amount) > 1000
ORDER BY dimension_1, dimension_2;
```

## Conditional aggregation

```sql
SELECT business_date,
       COUNT(*) AS total_rows,
       SUM(CASE WHEN status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_rows,
       SUM(CASE WHEN status = 'COMPLETED' THEN amount ELSE 0 END) AS completed_value,
       AVG(CASE WHEN status = 'COMPLETED' THEN amount END) AS average_completed_value
FROM table_name
GROUP BY business_date;
```

## Missing-value metrics

```sql
SELECT COUNT(*) - COUNT(nullable_column) AS missing_count,
       SUM(CASE WHEN nullable_column IS NULL THEN 1 ELSE 0 END) AS missing_count_again,
       100.0 * COUNT(nullable_column) / NULLIF(COUNT(*), 0) AS populated_percentage
FROM table_name;
```

## Safe percentage

```sql
100.0 * numerator / NULLIF(denominator, 0)
```

## Logical processing order

```text
FROM
WHERE
GROUP BY
HAVING
SELECT
DISTINCT
ORDER BY
LIMIT
```

## Week 2 completion test

You have completed Week 2 when you can:

- Explain `COUNT(*)` versus `COUNT(column)`.
- Predict how every aggregate handles NULL.
- State the grain of a grouped result.
- Use `WHERE` for row filters and `HAVING` for group filters.
- Write a `CASE` with correctly ordered conditions.
- Calculate counts, sums, and averages conditionally.
- Calculate a percentage without integer division or divide-by-zero.
- Build a daily metric report.
- Build a daily data-quality report.
- Explain why correct SQL syntax does not guarantee a correct business metric.

## Next week preview

Week 3 covers:

- Primary and foreign keys
- `INNER JOIN`
- `LEFT JOIN`
- `RIGHT JOIN` and `FULL OUTER JOIN`
- Self joins and cross joins
- Join multiplication and grain
- `UNION`, `UNION ALL`, `INTERSECT`, and `EXCEPT`
- Source-to-target reconciliation using joins
