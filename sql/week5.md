---
aliases:
  - DE SQL Week 5
  - SQL Window Functions Week 5
tags:
  - data-engineering
  - sql
  - window-functions
  - deduplication
  - event-analytics
  - interview-preparation
  - week-5
status: active
difficulty: intermediate
study_time: 6 days
created: 2026-08-16
previous: '[[Data-Engineer-SQL-Week-4-Obsidian]]'
---

# Data Engineer SQL — Week 5 Detailed Notes

> [!info] Week 5 goal
> Calculate rankings, trends, changes, running metrics, and latest-record logic without collapsing detail rows. Use deterministic window specifications suitable for Data Engineering pipelines.

## Obsidian navigation

- [[#Day 1 — Window-function foundations|Day 1 — Foundations]]
- [[#Day 2 — ROW_NUMBER, RANK, DENSE_RANK, and NTILE|Day 2 — Ranking]]
- [[#Day 3 — LAG, LEAD, and change detection|Day 3 — LAG and LEAD]]
- [[#Day 4 — Running totals and moving calculations|Day 4 — Running and moving metrics]]
- [[#Day 5 — Window frames and value functions|Day 5 — Frames and value functions]]
- [[#Day 6 — Deduplication and top-N patterns|Day 6 — Deduplication and top-N]]
- [[#Week 5 mini-project — Event sequencing and sessionization|Mini-project]]
- [[#Week 5 interview questions|Interview questions]]
- [[#Week 5 final practice set|Final practice]]
- [[#Week 5 one-page cheat sheet|Cheat sheet]]

## Progress tracker

- [ ] Day 1 completed
- [ ] Day 2 completed
- [ ] Day 3 completed
- [ ] Day 4 completed
- [ ] Day 5 completed
- [ ] Day 6 completed
- [ ] Event analytics mini-project completed
- [ ] Week 5 completion test passed

> [!tip] Determinism rule
> If a window order can tie, add a stable tiebreaker such as an ingestion sequence or unique record ID. Otherwise a rerun can select a different row.

**Level:** Intermediate  
**Duration:** 6 study days plus 1 revision/rest day  
**Daily time:** 75–100 minutes  
**Primary dialect:** PostgreSQL-style SQL  
**Prerequisite:** [[Data-Engineer-SQL-Week-4-Obsidian|Week 4 — Subqueries and CTEs]]

## Week 5 learning outcomes

By the end of this week, you should be able to:

- Explain how window functions differ from `GROUP BY`.
- Use `OVER`, `PARTITION BY`, window `ORDER BY`, and frames.
- Choose among `ROW_NUMBER`, `RANK`, and `DENSE_RANK`.
- Use `LAG` and `LEAD` for previous/next-row analysis.
- Detect attribute changes between sequential records.
- Calculate running totals and moving averages.
- Explain `ROWS` versus `RANGE` and peer-row behavior.
- Use `FIRST_VALUE`, `LAST_VALUE`, and `NTH_VALUE` safely.
- Keep the latest record per business key deterministically.
- Return top-N records per group.
- Sequence user events and create inactivity-based sessions.

---

# Practice setup

Continue using all tables from Weeks 1–4.

## Create duplicated customer staging data

```sql
CREATE TABLE customer_staging (
    ingestion_id       BIGINT PRIMARY KEY,
    customer_id        INTEGER NOT NULL,
    customer_name      VARCHAR(100),
    city               VARCHAR(100),
    email              VARCHAR(200),
    source_updated_at  TIMESTAMP NOT NULL,
    ingestion_time     TIMESTAMP NOT NULL
);

INSERT INTO customer_staging VALUES
(1, 1, 'Aarav Sharma', 'Pune',      'aarav@example.com',     '2026-07-01 09:00:00', '2026-07-01 09:05:00'),
(2, 1, 'Aarav Sharma', 'Mumbai',    'aarav@example.com',     '2026-08-01 10:00:00', '2026-08-01 10:05:00'),
(3, 1, 'Aarav Sharma', 'Mumbai',    'aarav.new@example.com', '2026-08-01 10:00:00', '2026-08-01 10:06:00'),
(4, 2, 'Diya Patel',   'Mumbai',    'diya@example.com',      '2026-07-15 08:00:00', '2026-07-15 08:03:00'),
(5, 2, 'Diya Patel',   'Pune',      'diya@example.com',      '2026-08-05 11:00:00', '2026-08-05 11:02:00'),
(6, 3, 'Kabir Singh',  'Pune',      NULL,                    '2026-07-20 14:00:00', '2026-07-20 14:04:00'),
(7, 3, 'Kabir Singh',  'Pune',      'kabir@example.com',     '2026-07-22 09:00:00', '2026-07-22 09:02:00'),
(8, 4, 'Meera Iyer',   'Bengaluru', 'meera@example.com',     '2026-08-03 12:00:00', '2026-08-03 12:01:00');
```

Important details:

- A business key can have several source versions.
- Customer `1` has two rows with the same source timestamp.
- `ingestion_time` and `ingestion_id` provide deterministic tie resolution.
- The newest row is not necessarily the one physically returned last.

## Create user events

```sql
CREATE TABLE user_events (
    event_id          BIGINT PRIMARY KEY,
    user_id           INTEGER NOT NULL,
    event_type        VARCHAR(50) NOT NULL,
    event_time        TIMESTAMP NOT NULL,
    ingestion_time    TIMESTAMP NOT NULL
);

INSERT INTO user_events VALUES
(10001, 1, 'login',        '2026-08-01 09:00:00', '2026-08-01 09:00:05'),
(10002, 1, 'view_product', '2026-08-01 09:05:00', '2026-08-01 09:05:03'),
(10003, 1, 'add_to_cart',  '2026-08-01 09:12:00', '2026-08-01 09:12:04'),
(10004, 1, 'purchase',     '2026-08-01 09:20:00', '2026-08-01 09:20:02'),
(10005, 1, 'login',        '2026-08-01 11:15:00', '2026-08-01 11:15:02'),
(10006, 1, 'view_product', '2026-08-01 11:18:00', '2026-08-01 11:18:03'),
(10007, 2, 'login',        '2026-08-01 10:00:00', '2026-08-01 10:00:02'),
(10008, 2, 'view_product', '2026-08-01 10:08:00', '2026-08-01 10:08:03'),
(10009, 2, 'add_to_cart',  '2026-08-01 10:50:00', '2026-08-01 10:50:01'),
(10010, 2, 'purchase',     '2026-08-01 11:00:00', '2026-08-01 11:00:02'),
(10011, 3, 'login',        '2026-08-02 08:30:00', '2026-08-02 08:30:03'),
(10012, 3, 'view_product', '2026-08-02 08:35:00', '2026-08-02 08:35:02'),
(10013, 3, 'view_product', '2026-08-02 08:35:00', '2026-08-02 08:35:04'),
(10014, 3, 'logout',       '2026-08-02 09:00:00', '2026-08-02 09:00:02');
```

Events `10012` and `10013` share an event timestamp. `event_id` provides deterministic event ordering.

> [!warning] Setup safety
> Run these setup statements once. Use a separate practice schema or skip creation if the tables already exist.

---

# Day 1 — Window-function foundations

## 1.1 GROUP BY versus a window function

`GROUP BY` collapses several input rows into one row per group.

```sql
SELECT customer_id,
       SUM(order_total) AS customer_total
FROM orders
WHERE customer_id IS NOT NULL
GROUP BY customer_id;
```

Output grain: one row per customer.

A window function calculates across related rows while preserving every detail row.

```sql
SELECT order_id,
       customer_id,
       order_total,
       SUM(order_total) OVER (
           PARTITION BY customer_id
       ) AS customer_total
FROM orders
WHERE customer_id IS NOT NULL;
```

Output grain remains one row per order.

## 1.2 OVER clause

Every window function uses `OVER`.

```sql
function_name(expression) OVER (
    PARTITION BY grouping_columns
    ORDER BY sequencing_columns
    ROWS BETWEEN frame_start AND frame_end
)
```

Not every function needs every part.

## 1.3 Window without PARTITION BY

The entire result set is one partition:

```sql
SELECT order_id,
       order_total,
       SUM(order_total) OVER () AS overall_total
FROM orders;
```

Each detail row receives the same overall total.

## 1.4 PARTITION BY

`PARTITION BY` divides rows into independent calculation groups without collapsing them.

```sql
SELECT order_id,
       customer_id,
       order_total,
       COUNT(*) OVER (
           PARTITION BY customer_id
       ) AS customer_order_count
FROM orders
WHERE customer_id IS NOT NULL;
```

Every order for the same customer receives the same customer order count.

## 1.5 Window ORDER BY

Window ordering controls sequence inside the calculation. It does not guarantee final result order.

```sql
SELECT order_id,
       customer_id,
       order_date,
       ROW_NUMBER() OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
       ) AS customer_order_sequence
FROM orders
WHERE customer_id IS NOT NULL
ORDER BY customer_id, customer_order_sequence;
```

The final `ORDER BY` controls output display.

## 1.6 Logical processing position

Window functions are logically evaluated after:

- `FROM` and joins
- `WHERE`
- `GROUP BY`
- `HAVING`

They are evaluated before the final `ORDER BY` and `LIMIT`.

Therefore, window results cannot normally be filtered in `WHERE` of the same query.

Invalid:

```sql
SELECT o.*,
       ROW_NUMBER() OVER (
           PARTITION BY customer_id
           ORDER BY order_date DESC
       ) AS rn
FROM orders AS o
WHERE rn = 1;
```

Use a CTE or derived table:

```sql
WITH ranked AS (
    SELECT o.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY order_date DESC, order_id DESC
           ) AS rn
    FROM orders AS o
)
SELECT *
FROM ranked
WHERE rn = 1;
```

Some engines support `QUALIFY` for direct window-result filtering. PostgreSQL does not. Databricks SQL supports `QUALIFY`; verify current platform syntax.

## 1.7 Aggregate after GROUP BY, then window

Daily revenue and share of total:

```sql
WITH daily AS (
    SELECT order_date,
           SUM(order_total) AS daily_value
    FROM orders
    GROUP BY order_date
)
SELECT order_date,
       daily_value,
       SUM(daily_value) OVER () AS overall_value,
       100.0 * daily_value / NULLIF(SUM(daily_value) OVER (), 0)
           AS percentage_of_total
FROM daily
ORDER BY order_date;
```

The CTE makes the daily grain explicit before the window calculation.

## 1.8 Multiple windows

```sql
SELECT order_id,
       customer_id,
       order_total,
       SUM(order_total) OVER () AS overall_total,
       SUM(order_total) OVER (PARTITION BY customer_id) AS customer_total,
       AVG(order_total) OVER (PARTITION BY customer_id) AS customer_average
FROM orders
WHERE customer_id IS NOT NULL;
```

## 1.9 Named window specification

PostgreSQL-style:

```sql
SELECT order_id,
       customer_id,
       order_date,
       ROW_NUMBER() OVER customer_window AS sequence_number,
       LAG(order_date) OVER customer_window AS previous_order_date
FROM orders
WINDOW customer_window AS (
    PARTITION BY customer_id
    ORDER BY order_date, order_id
);
```

Support and syntax vary by engine. Named windows reduce repetition when several functions use the same specification.

## 1.10 Window-function categories

| Category | Examples | Purpose |
|---|---|---|
| Ranking | `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` | Order and segment rows |
| Offset | `LAG`, `LEAD` | Previous or next values |
| Aggregate window | `SUM`, `AVG`, `COUNT`, `MIN`, `MAX` | Running or partition metrics |
| Value | `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE` | Value at a position in a frame |
| Distribution | `PERCENT_RANK`, `CUME_DIST` | Relative position |

## Day 1 common mistakes

- Confusing `PARTITION BY` with `GROUP BY`.
- Expecting window ordering to sort final output.
- Filtering a window alias in `WHERE`.
- Omitting a tiebreaker from window ordering.
- Applying a window before aligning input grain.
- Assuming every SQL engine supports `QUALIFY` or named windows.
- Forgetting that NULL values remain detail rows.

## Day 1 exercises

1. Display every order with overall order count.
2. Display every order with overall known order total.
3. Display each order with customer order count.
4. Display each order with customer total and average.
5. Display each product with category average price.
6. Display each employee with department average salary.
7. Calculate each order amount as a percentage of customer total.
8. Calculate daily completed revenue, then overall completed revenue.
9. Assign an order sequence per customer.
10. Use a CTE to filter the latest order per customer.
11. Explain why a window does not change output grain.
12. Explain why window `ORDER BY` and final `ORDER BY` are different.

## Day 1 solutions

```sql
-- 1
SELECT order_id,
       COUNT(*) OVER () AS overall_order_count
FROM orders;

-- 2
SELECT order_id,
       order_total,
       SUM(order_total) OVER () AS overall_known_total
FROM orders;

-- 3
SELECT order_id,
       customer_id,
       COUNT(*) OVER (PARTITION BY customer_id) AS customer_order_count
FROM orders
WHERE customer_id IS NOT NULL;

-- 4
SELECT order_id,
       customer_id,
       order_total,
       SUM(order_total) OVER (PARTITION BY customer_id) AS customer_total,
       AVG(order_total) OVER (PARTITION BY customer_id) AS customer_average
FROM orders
WHERE customer_id IS NOT NULL;

-- 5
SELECT product_id,
       product_name,
       category,
       unit_price,
       AVG(unit_price) OVER (PARTITION BY category) AS category_average
FROM products;

-- 6
SELECT employee_id,
       employee_name,
       department,
       salary,
       AVG(salary) OVER (PARTITION BY department) AS department_average
FROM employees;

-- 7
SELECT order_id,
       customer_id,
       order_total,
       100.0 * order_total
           / NULLIF(SUM(order_total) OVER (PARTITION BY customer_id), 0)
           AS customer_total_percentage
FROM orders
WHERE customer_id IS NOT NULL;

-- 8
WITH daily AS (
    SELECT order_date,
           SUM(order_total) AS completed_revenue
    FROM orders
    WHERE order_status = 'COMPLETED'
    GROUP BY order_date
)
SELECT order_date,
       completed_revenue,
       SUM(completed_revenue) OVER () AS overall_completed_revenue
FROM daily;

-- 9
SELECT order_id,
       customer_id,
       order_date,
       ROW_NUMBER() OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
       ) AS order_sequence
FROM orders
WHERE customer_id IS NOT NULL;

-- 10
WITH ranked AS (
    SELECT o.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY order_date DESC, order_id DESC
           ) AS rn
    FROM orders AS o
    WHERE customer_id IS NOT NULL
)
SELECT *
FROM ranked
WHERE rn = 1;
```

---

# Day 2 — ROW_NUMBER, RANK, DENSE_RANK, and NTILE

## 2.1 ROW_NUMBER

`ROW_NUMBER` assigns a unique sequence within each partition.

```sql
SELECT product_id,
       product_name,
       category,
       unit_price,
       ROW_NUMBER() OVER (
           PARTITION BY category
           ORDER BY unit_price DESC, product_id
       ) AS price_row_number
FROM products;
```

Even tied prices receive different row numbers. The tiebreaker determines which row comes first.

## 2.2 RANK

`RANK` assigns the same rank to ties and leaves gaps afterward.

Values `100, 100, 90` receive ranks `1, 1, 3`.

```sql
SELECT product_id,
       category,
       unit_price,
       RANK() OVER (
           PARTITION BY category
           ORDER BY unit_price DESC
       ) AS price_rank
FROM products;
```

Do not add a unique tiebreaker to the ranking order if equal business values are supposed to tie. A tiebreaker would remove the tie.

## 2.3 DENSE_RANK

`DENSE_RANK` assigns the same rank to ties without gaps.

Values `100, 100, 90` receive ranks `1, 1, 2`.

```sql
SELECT product_id,
       category,
       unit_price,
       DENSE_RANK() OVER (
           PARTITION BY category
           ORDER BY unit_price DESC
       ) AS dense_price_rank
FROM products;
```

## 2.4 Ranking comparison

```sql
SELECT order_id,
       order_total,
       ROW_NUMBER() OVER (ORDER BY order_total DESC, order_id) AS row_number_value,
       RANK() OVER (ORDER BY order_total DESC) AS rank_value,
       DENSE_RANK() OVER (ORDER BY order_total DESC) AS dense_rank_value
FROM orders
WHERE order_total IS NOT NULL;
```

| Requirement | Function |
|---|---|
| Exactly one deterministic row per position | `ROW_NUMBER` |
| Competition ranking with gaps | `RANK` |
| Rank distinct values without gaps | `DENSE_RANK` |

## 2.5 Top N per group

Top two products per category:

```sql
WITH ranked_products AS (
    SELECT p.*,
           ROW_NUMBER() OVER (
               PARTITION BY category
               ORDER BY unit_price DESC, product_id
           ) AS rn
    FROM products AS p
)
SELECT *
FROM ranked_products
WHERE rn <= 2
ORDER BY category, rn;
```

This returns exactly two rows per category when at least two rows exist.

## 2.6 Top N including ties

Use `RANK` or `DENSE_RANK` depending on definition:

```sql
WITH ranked_products AS (
    SELECT p.*,
           DENSE_RANK() OVER (
               PARTITION BY category
               ORDER BY unit_price DESC
           ) AS price_rank
    FROM products AS p
)
SELECT *
FROM ranked_products
WHERE price_rank <= 2;
```

More than two rows can be returned when tied values exist.

## 2.7 Nth highest distinct value

Second-highest distinct order total:

```sql
WITH ranked_amounts AS (
    SELECT order_total,
           DENSE_RANK() OVER (
               ORDER BY order_total DESC
           ) AS amount_rank
    FROM orders
    WHERE order_total IS NOT NULL
)
SELECT MAX(order_total) AS second_highest_total
FROM ranked_amounts
WHERE amount_rank = 2;
```

## 2.8 NTILE

`NTILE(n)` distributes ordered rows into approximately equal buckets.

```sql
SELECT customer_id,
       customer_name,
       credit_limit,
       NTILE(4) OVER (
           ORDER BY credit_limit DESC NULLS LAST, customer_id
       ) AS credit_quartile
FROM customers;
```

Bucket sizes can differ by one. Tied values can fall into different buckets because `NTILE` distributes rows, not distinct values.

## 2.9 PERCENT_RANK and CUME_DIST

`PERCENT_RANK` estimates relative rank from 0 to 1. `CUME_DIST` gives the proportion of rows less than or equal to the current ordered value, accounting for direction.

```sql
SELECT employee_id,
       department,
       salary,
       PERCENT_RANK() OVER (
           PARTITION BY department
           ORDER BY salary
       ) AS salary_percent_rank,
       CUME_DIST() OVER (
           PARTITION BY department
           ORDER BY salary
       ) AS salary_cumulative_distribution
FROM employees;
```

Exact interpretation at ties matters; test with small data.

## 2.10 Ranking NULL values

NULL ordering differs across engines. Use explicit `NULLS FIRST` or `NULLS LAST` where supported, or a `CASE` sort expression.

For ranking monetary amounts, you may choose to exclude NULL values entirely.

## Day 2 common mistakes

- Using `ROW_NUMBER` when ties must share rank.
- Using `RANK` when exactly N rows are required.
- Adding a unique tiebreaker to `RANK` and accidentally eliminating ties.
- Omitting a tiebreaker from `ROW_NUMBER`.
- Assuming `NTILE` keeps tied values together.
- Ignoring NULL ordering.
- Filtering a rank in the same query `WHERE` clause.

## Day 2 exercises

1. Number orders globally from newest to oldest.
2. Number orders per customer from newest to oldest.
3. Rank products by price globally.
4. Rank products by price within category.
5. Compare all three ranking functions on order totals.
6. Return the latest order per customer.
7. Return the first order per customer.
8. Return the top two products per category exactly.
9. Return top two distinct prices per category including ties.
10. Return the second-highest distinct order total.
11. Divide employees into salary quartiles.
12. Explain why `RANK <= 3` can return more than three rows.

## Day 2 solutions

```sql
-- 1
SELECT order_id,
       order_date,
       ROW_NUMBER() OVER (
           ORDER BY order_date DESC, order_id DESC
       ) AS newest_sequence
FROM orders;

-- 2
SELECT order_id,
       customer_id,
       order_date,
       ROW_NUMBER() OVER (
           PARTITION BY customer_id
           ORDER BY order_date DESC, order_id DESC
       ) AS customer_recency
FROM orders
WHERE customer_id IS NOT NULL;

-- 3
SELECT product_id,
       product_name,
       unit_price,
       RANK() OVER (ORDER BY unit_price DESC) AS global_price_rank
FROM products;

-- 4
SELECT product_id,
       product_name,
       category,
       unit_price,
       RANK() OVER (
           PARTITION BY category
           ORDER BY unit_price DESC
       ) AS category_price_rank
FROM products;

-- 5
SELECT order_id,
       order_total,
       ROW_NUMBER() OVER (ORDER BY order_total DESC, order_id) AS rn,
       RANK() OVER (ORDER BY order_total DESC) AS rnk,
       DENSE_RANK() OVER (ORDER BY order_total DESC) AS dense_rnk
FROM orders
WHERE order_total IS NOT NULL;

-- 6
WITH ranked AS (
    SELECT o.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY order_date DESC, order_id DESC
           ) AS rn
    FROM orders AS o
    WHERE customer_id IS NOT NULL
)
SELECT * FROM ranked WHERE rn = 1;

-- 7
WITH ranked AS (
    SELECT o.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY order_date, order_id
           ) AS rn
    FROM orders AS o
    WHERE customer_id IS NOT NULL
)
SELECT * FROM ranked WHERE rn = 1;

-- 8
WITH ranked AS (
    SELECT p.*,
           ROW_NUMBER() OVER (
               PARTITION BY category
               ORDER BY unit_price DESC, product_id
           ) AS rn
    FROM products AS p
)
SELECT * FROM ranked WHERE rn <= 2;

-- 9
WITH ranked AS (
    SELECT p.*,
           DENSE_RANK() OVER (
               PARTITION BY category
               ORDER BY unit_price DESC
           ) AS price_rank
    FROM products AS p
)
SELECT * FROM ranked WHERE price_rank <= 2;

-- 10
WITH ranked AS (
    SELECT order_total,
           DENSE_RANK() OVER (ORDER BY order_total DESC) AS amount_rank
    FROM orders
    WHERE order_total IS NOT NULL
)
SELECT MAX(order_total) AS second_highest_total
FROM ranked
WHERE amount_rank = 2;

-- 11
SELECT employee_id,
       employee_name,
       salary,
       NTILE(4) OVER (
           ORDER BY salary DESC, employee_id
       ) AS salary_quartile
FROM employees;
```

---

# Day 3 — LAG, LEAD, and change detection

## 3.1 LAG

`LAG` returns a value from an earlier row in the window order.

```sql
SELECT order_id,
       customer_id,
       order_date,
       order_total,
       LAG(order_total) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
       ) AS previous_order_total
FROM orders
WHERE customer_id IS NOT NULL;
```

The first row in each partition has no previous row and returns NULL by default.

## 3.2 LEAD

`LEAD` returns a value from a later row.

```sql
SELECT order_id,
       customer_id,
       order_date,
       LEAD(order_date) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
       ) AS next_order_date
FROM orders
WHERE customer_id IS NOT NULL;
```

## 3.3 Offset and default arguments

General form:

```sql
LAG(expression, offset, default_value) OVER (...)
```

Two rows back with a default zero:

```sql
LAG(order_total, 2, 0) OVER (
    PARTITION BY customer_id
    ORDER BY order_date, order_id
)
```

Be careful: a default zero and a genuine previous NULL have different meanings.

## 3.4 Difference from previous value

```sql
WITH sequenced AS (
    SELECT order_id,
           customer_id,
           order_date,
           order_total,
           LAG(order_total) OVER (
               PARTITION BY customer_id
               ORDER BY order_date, order_id
           ) AS previous_order_total
    FROM orders
    WHERE customer_id IS NOT NULL
)
SELECT *,
       order_total - previous_order_total AS amount_change
FROM sequenced;
```

## 3.5 Time between events

PostgreSQL timestamp subtraction returns an interval:

```sql
WITH sequenced AS (
    SELECT event_id,
           user_id,
           event_type,
           event_time,
           LAG(event_time) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS previous_event_time
    FROM user_events
)
SELECT *,
       event_time - previous_event_time AS time_since_previous_event
FROM sequenced
ORDER BY user_id, event_time, event_id;
```

Date/time-difference syntax varies by engine.

## 3.6 Detect attribute changes

```sql
WITH sequenced AS (
    SELECT s.*,
           LAG(city) OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at, ingestion_time, ingestion_id
           ) AS previous_city
    FROM customer_staging AS s
)
SELECT *,
       CASE
           WHEN previous_city IS DISTINCT FROM city THEN 1
           ELSE 0
       END AS city_change_flag
FROM sequenced;
```

For the first record, `previous_city` is NULL. If the first row should not count as a change, also check row number:

```sql
WITH sequenced AS (
    SELECT s.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at, ingestion_time, ingestion_id
           ) AS sequence_number,
           LAG(city) OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at, ingestion_time, ingestion_id
           ) AS previous_city
    FROM customer_staging AS s
)
SELECT *,
       CASE
           WHEN sequence_number = 1 THEN 0
           WHEN previous_city IS DISTINCT FROM city THEN 1
           ELSE 0
       END AS city_change_flag
FROM sequenced;
```

## 3.7 Month-over-month growth

```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date) AS month_start,
           SUM(CASE
                   WHEN order_status = 'COMPLETED' THEN order_total
                   ELSE 0
               END) AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
),
with_previous AS (
    SELECT month_start,
           revenue,
           LAG(revenue) OVER (ORDER BY month_start) AS previous_revenue
    FROM monthly
)
SELECT month_start,
       revenue,
       previous_revenue,
       revenue - previous_revenue AS absolute_change,
       100.0 * (revenue - previous_revenue)
           / NULLIF(previous_revenue, 0) AS growth_percentage
FROM with_previous;
```

## 3.8 Consecutive event pattern

Find rows whose next event is a purchase:

```sql
WITH sequenced AS (
    SELECT e.*,
           LEAD(event_type) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS next_event_type
    FROM user_events AS e
)
SELECT *
FROM sequenced
WHERE next_event_type = 'purchase';
```

This checks immediate sequence only, not whether a purchase occurs eventually.

## 3.9 Gap detection

```sql
WITH sequenced AS (
    SELECT e.*,
           LAG(event_time) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS previous_event_time
    FROM user_events AS e
)
SELECT *,
       CASE
           WHEN previous_event_time IS NULL THEN 1
           WHEN event_time - previous_event_time > INTERVAL '30 minutes' THEN 1
           ELSE 0
       END AS new_session_flag
FROM sequenced;
```

This flag becomes the basis for sessionization.

## Day 3 common mistakes

- Omitting partitioning and comparing different users.
- Omitting a tiebreaker for equal timestamps.
- Treating the first-row NULL as a real previous value.
- Confusing a missing previous row with a previous NULL value.
- Calculating percentage change without protecting zero denominator.
- Assuming `LEAD` finds any future matching event rather than the immediate next row.
- Using engine-specific interval syntax without checking the platform.

## Day 3 exercises

1. Return previous order total per customer.
2. Return next order date per customer.
3. Calculate change from previous order total.
4. Calculate days between customer orders.
5. Return previous product price within category order.
6. Detect customer city changes in staging.
7. Detect email changes in staging.
8. Return previous event type per user.
9. Calculate time between user events.
10. Flag event gaps above 30 minutes.
11. Find events immediately followed by purchase.
12. Explain why event ID is needed when timestamps tie.

## Day 3 solutions

```sql
-- 1
SELECT order_id,
       customer_id,
       order_total,
       LAG(order_total) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
       ) AS previous_order_total
FROM orders
WHERE customer_id IS NOT NULL;

-- 2
SELECT order_id,
       customer_id,
       order_date,
       LEAD(order_date) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
       ) AS next_order_date
FROM orders
WHERE customer_id IS NOT NULL;

-- 3
WITH x AS (
    SELECT order_id,
           customer_id,
           order_total,
           LAG(order_total) OVER (
               PARTITION BY customer_id
               ORDER BY order_date, order_id
           ) AS previous_order_total
    FROM orders
    WHERE customer_id IS NOT NULL
)
SELECT *, order_total - previous_order_total AS amount_change
FROM x;

-- 4: PostgreSQL date subtraction
WITH x AS (
    SELECT order_id,
           customer_id,
           order_date,
           LAG(order_date) OVER (
               PARTITION BY customer_id
               ORDER BY order_date, order_id
           ) AS previous_order_date
    FROM orders
    WHERE customer_id IS NOT NULL
)
SELECT *, order_date - previous_order_date AS days_since_previous_order
FROM x;

-- 5
SELECT product_id,
       category,
       unit_price,
       LAG(unit_price) OVER (
           PARTITION BY category
           ORDER BY unit_price, product_id
       ) AS previous_category_price
FROM products;

-- 6 and 7
WITH x AS (
    SELECT s.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at, ingestion_time, ingestion_id
           ) AS sequence_number,
           LAG(city) OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at, ingestion_time, ingestion_id
           ) AS previous_city,
           LAG(email) OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at, ingestion_time, ingestion_id
           ) AS previous_email
    FROM customer_staging AS s
)
SELECT *,
       CASE
           WHEN sequence_number > 1 AND city IS DISTINCT FROM previous_city THEN 1
           ELSE 0
       END AS city_change_flag,
       CASE
           WHEN sequence_number > 1 AND email IS DISTINCT FROM previous_email THEN 1
           ELSE 0
       END AS email_change_flag
FROM x;

-- 8, 9, and 10
WITH x AS (
    SELECT e.*,
           LAG(event_type) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS previous_event_type,
           LAG(event_time) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS previous_event_time
    FROM user_events AS e
)
SELECT *,
       event_time - previous_event_time AS time_since_previous,
       CASE
           WHEN previous_event_time IS NULL THEN 1
           WHEN event_time - previous_event_time > INTERVAL '30 minutes' THEN 1
           ELSE 0
       END AS new_session_flag
FROM x;

-- 11
WITH x AS (
    SELECT e.*,
           LEAD(event_type) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS next_event_type
    FROM user_events AS e
)
SELECT * FROM x WHERE next_event_type = 'purchase';
```

---

# Day 4 — Running totals and moving calculations

## 4.1 Running total

```sql
WITH daily AS (
    SELECT order_date,
           SUM(CASE
                   WHEN order_status = 'COMPLETED' THEN order_total
                   ELSE 0
               END) AS daily_revenue
    FROM orders
    GROUP BY order_date
)
SELECT order_date,
       daily_revenue,
       SUM(daily_revenue) OVER (
           ORDER BY order_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_revenue
FROM daily
ORDER BY order_date;
```

The explicit frame means from the first ordered row through the current row.

## 4.2 Running total per partition

```sql
SELECT order_id,
       customer_id,
       order_date,
       order_total,
       SUM(order_total) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS customer_running_total
FROM orders
WHERE customer_id IS NOT NULL;
```

Each customer running total restarts at the partition boundary.

## 4.3 Moving average by rows

Seven-row moving average of daily revenue:

```sql
WITH daily AS (
    SELECT order_date,
           SUM(CASE
                   WHEN order_status = 'COMPLETED' THEN order_total
                   ELSE 0
               END) AS daily_revenue
    FROM orders
    GROUP BY order_date
)
SELECT order_date,
       daily_revenue,
       AVG(daily_revenue) OVER (
           ORDER BY order_date
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ) AS seven_row_moving_average
FROM daily;
```

This is seven available rows, not necessarily seven calendar days. If dates are missing, create a calendar table and fill absent dates before applying the window.

## 4.4 Moving sum

```sql
SUM(daily_revenue) OVER (
    ORDER BY order_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
) AS seven_row_moving_sum
```

## 4.5 Centered moving average

```sql
AVG(daily_revenue) OVER (
    ORDER BY order_date
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
) AS centered_three_row_average
```

At the beginning and end, fewer than three rows may be available.

## 4.6 Cumulative count

```sql
SELECT event_id,
       user_id,
       event_time,
       COUNT(*) OVER (
           PARTITION BY user_id
           ORDER BY event_time, event_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS user_event_sequence_count
FROM user_events;
```

This resembles `ROW_NUMBER`, but aggregate windows allow conditions.

## 4.7 Conditional running count

```sql
SELECT event_id,
       user_id,
       event_type,
       event_time,
       SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) OVER (
           PARTITION BY user_id
           ORDER BY event_time, event_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS purchases_so_far
FROM user_events;
```

## 4.8 Remaining total

```sql
SUM(daily_revenue) OVER (
    ORDER BY order_date
    ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
) AS remaining_revenue
```

## 4.9 Running percentage of total

```sql
WITH daily AS (
    SELECT order_date,
           SUM(order_total) AS daily_value
    FROM orders
    GROUP BY order_date
),
windowed AS (
    SELECT order_date,
           daily_value,
           SUM(daily_value) OVER (
               ORDER BY order_date
               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
           ) AS running_value,
           SUM(daily_value) OVER () AS overall_value
    FROM daily
)
SELECT *,
       100.0 * running_value / NULLIF(overall_value, 0)
           AS cumulative_percentage
FROM windowed;
```

## 4.10 Performance considerations

Window functions may require repartitioning and sorting large datasets.

Consider:

- Filter unnecessary rows early.
- Select only needed columns.
- Align distribution/partitioning with window keys when platform design allows.
- Avoid many different window specifications in one query.
- Pre-aggregate when detail is not required.
- Inspect spill, shuffle, sorting, and skew.

## Day 4 common mistakes

- Omitting an explicit frame for running calculations.
- Confusing seven rows with seven calendar days.
- Applying a moving window before daily aggregation.
- Forgetting the partition and mixing customers.
- Using a centered window when only past data is allowed.
- Ignoring partial windows at boundaries.
- Running large windows over unfiltered detail unnecessarily.

## Day 4 exercises

1. Calculate global running order total.
2. Calculate running order total per customer.
3. Calculate cumulative order count per customer.
4. Calculate daily completed revenue.
5. Add running completed revenue.
6. Add a three-row moving average.
7. Add a three-row moving sum.
8. Calculate remaining revenue from current date onward.
9. Calculate cumulative percentage of total revenue.
10. Calculate purchases-so-far per user.
11. Explain why a row-based moving average can skip calendar dates.
12. Describe two window-performance risks.

## Day 4 solutions

```sql
-- 1
SELECT order_id,
       order_date,
       order_total,
       SUM(order_total) OVER (
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total
FROM orders;

-- 2 and 3
SELECT order_id,
       customer_id,
       order_date,
       order_total,
       SUM(order_total) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS customer_running_total,
       COUNT(*) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS customer_cumulative_order_count
FROM orders
WHERE customer_id IS NOT NULL;

-- 4 through 9
WITH daily AS (
    SELECT order_date,
           SUM(CASE
                   WHEN order_status = 'COMPLETED' THEN order_total
                   ELSE 0
               END) AS daily_revenue
    FROM orders
    GROUP BY order_date
),
windowed AS (
    SELECT order_date,
           daily_revenue,
           SUM(daily_revenue) OVER (
               ORDER BY order_date
               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
           ) AS running_revenue,
           AVG(daily_revenue) OVER (
               ORDER BY order_date
               ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
           ) AS three_row_moving_average,
           SUM(daily_revenue) OVER (
               ORDER BY order_date
               ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
           ) AS three_row_moving_sum,
           SUM(daily_revenue) OVER (
               ORDER BY order_date
               ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
           ) AS remaining_revenue,
           SUM(daily_revenue) OVER () AS overall_revenue
    FROM daily
)
SELECT *,
       100.0 * running_revenue / NULLIF(overall_revenue, 0)
           AS cumulative_percentage
FROM windowed
ORDER BY order_date;

-- 10
SELECT event_id,
       user_id,
       event_type,
       event_time,
       SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) OVER (
           PARTITION BY user_id
           ORDER BY event_time, event_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS purchases_so_far
FROM user_events;
```

---

# Day 5 — Window frames and value functions

## 5.1 What is a frame?

For an ordered window, the frame defines which rows around the current row participate in the calculation.

Common boundaries:

- `UNBOUNDED PRECEDING`: first row in partition
- `n PRECEDING`: n rows before current row
- `CURRENT ROW`
- `n FOLLOWING`: n rows after current row
- `UNBOUNDED FOLLOWING`: last row in partition

## 5.2 ROWS frame

`ROWS` counts physical ordered rows.

```sql
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
```

This includes at most three rows: two prior rows plus current row.

## 5.3 RANGE frame

`RANGE` groups peer rows sharing the same ordering value and may define value-based ranges depending on engine support.

Important default behavior: with window `ORDER BY`, many engines default to a range-like frame ending at the current row and including peers. This can make running totals jump across tied values.

Use an explicit `ROWS` frame for row-by-row running calculations.

## 5.4 Peer rows example

Events `10012` and `10013` have the same `event_time`.

```sql
SELECT event_id,
       user_id,
       event_time,
       COUNT(*) OVER (
           PARTITION BY user_id
           ORDER BY event_time
           RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS range_count,
       COUNT(*) OVER (
           PARTITION BY user_id
           ORDER BY event_time, event_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS row_count
FROM user_events;
```

The `RANGE` result can include all equal-time peers together, while the deterministic `ROWS` result advances one event at a time.

## 5.5 FIRST_VALUE

```sql
SELECT order_id,
       customer_id,
       order_date,
       FIRST_VALUE(order_date) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS first_order_date
FROM orders
WHERE customer_id IS NOT NULL;
```

## 5.6 LAST_VALUE default-frame trap

This can return the current row value rather than the partition last value:

```sql
LAST_VALUE(order_date) OVER (
    PARTITION BY customer_id
    ORDER BY order_date, order_id
)
```

The default frame often ends at the current row.

Correct whole-partition frame:

```sql
LAST_VALUE(order_date) OVER (
    PARTITION BY customer_id
    ORDER BY order_date, order_id
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

## 5.7 FIRST_VALUE and reversed order

Latest value can be obtained with `FIRST_VALUE` and descending order:

```sql
FIRST_VALUE(order_date) OVER (
    PARTITION BY customer_id
    ORDER BY order_date DESC, order_id DESC
)
```

This often avoids the `LAST_VALUE` frame surprise, but explicit intent remains important.

## 5.8 NTH_VALUE

Second order date in a customer partition:

```sql
NTH_VALUE(order_date, 2) OVER (
    PARTITION BY customer_id
    ORDER BY order_date, order_id
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
) AS second_order_date
```

If fewer than two rows exist, the result is NULL.

## 5.9 MIN and MAX versus FIRST and LAST

- `MIN(order_date)` gives the smallest date, independent of another column.
- `FIRST_VALUE(status) ORDER BY order_date` gives the status associated with the first ordered row.

Use value functions when you need an attribute from a positioned row.

## 5.10 Exclusion and GROUPS frames

The SQL standard and some engines support `GROUPS` frames and frame exclusions. Availability varies. These are advanced features; master `ROWS` and understand peer behavior first.

## Day 5 common mistakes

- Relying on the default frame.
- Assuming `RANGE` and `ROWS` are identical.
- Using `LAST_VALUE` without an unbounded-following frame.
- Omitting a tiebreaker from value-function ordering.
- Using `MIN` when an attribute from the earliest row is required.
- Forgetting that partial frames have fewer rows at boundaries.
- Assuming advanced frame syntax is portable.

## Day 5 exercises

1. Calculate running count with explicit `ROWS`.
2. Compare `ROWS` and `RANGE` on tied event times.
3. Display first order date on every customer order.
4. Display latest order date on every customer order.
5. Display first order status per customer.
6. Display latest order status per customer.
7. Display second order date per customer.
8. Calculate centered three-row average daily revenue.
9. Calculate total partition revenue on every customer row.
10. Explain the `LAST_VALUE` default-frame trap.
11. Explain peers in a `RANGE` frame.
12. Explain when `FIRST_VALUE` is better than `MIN`.

## Day 5 solutions

```sql
-- 1
SELECT event_id,
       user_id,
       event_time,
       COUNT(*) OVER (
           PARTITION BY user_id
           ORDER BY event_time, event_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_event_count
FROM user_events;

-- 2
SELECT event_id,
       user_id,
       event_time,
       COUNT(*) OVER (
           PARTITION BY user_id
           ORDER BY event_time
           RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS range_count,
       COUNT(*) OVER (
           PARTITION BY user_id
           ORDER BY event_time, event_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS rows_count
FROM user_events;

-- 3 through 7
SELECT order_id,
       customer_id,
       order_date,
       order_status,
       FIRST_VALUE(order_date) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS first_order_date,
       LAST_VALUE(order_date) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS latest_order_date,
       FIRST_VALUE(order_status) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS first_order_status,
       LAST_VALUE(order_status) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS latest_order_status,
       NTH_VALUE(order_date, 2) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS second_order_date
FROM orders
WHERE customer_id IS NOT NULL;

-- 8
WITH daily AS (
    SELECT order_date,
           SUM(CASE WHEN order_status = 'COMPLETED' THEN order_total ELSE 0 END)
               AS daily_revenue
    FROM orders
    GROUP BY order_date
)
SELECT order_date,
       daily_revenue,
       AVG(daily_revenue) OVER (
           ORDER BY order_date
           ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
       ) AS centered_average
FROM daily;

-- 9
SELECT order_id,
       customer_id,
       order_total,
       SUM(order_total) OVER (
           PARTITION BY customer_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS customer_total
FROM orders
WHERE customer_id IS NOT NULL;
```

---

# Day 6 — Deduplication and top-N patterns

## 6.1 Why deduplication needs a rule

Duplicate business keys are not necessarily identical records. A correct rule states:

- Partition key: which rows represent the same entity?
- Ordering: which version is preferred?
- Tiebreaker: how are equal primary ordering values resolved?
- Delete behavior: should tombstones win?
- Quality behavior: should invalid latest rows be accepted or rejected?

## 6.2 Latest row per business key

```sql
WITH ranked AS (
    SELECT s.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at DESC,
                        ingestion_time DESC,
                        ingestion_id DESC
           ) AS rn
    FROM customer_staging AS s
)
SELECT *
FROM ranked
WHERE rn = 1;
```

Customer `1` has tied source timestamps. The later ingestion time and unique ingestion ID make selection deterministic.

## 6.3 Why MAX timestamp join can fail

```sql
WITH latest_timestamp AS (
    SELECT customer_id,
           MAX(source_updated_at) AS latest_updated_at
    FROM customer_staging
    GROUP BY customer_id
)
SELECT s.*
FROM customer_staging AS s
JOIN latest_timestamp AS l
  ON s.customer_id = l.customer_id
 AND s.source_updated_at = l.latest_updated_at;
```

Customer `1` returns two rows because both share the maximum source timestamp.

`ROW_NUMBER` with tiebreakers returns exactly one.

## 6.4 Preserve duplicate diagnostics

```sql
SELECT customer_id,
       COUNT(*) AS version_count,
       MIN(source_updated_at) AS first_source_update,
       MAX(source_updated_at) AS latest_source_update
FROM customer_staging
GROUP BY customer_id
HAVING COUNT(*) > 1;
```

Do not discard duplicates without metrics and lineage. Track how many rows were received, kept, and rejected.

## 6.5 Exact duplicate versus business-key duplicate

- Exact duplicate: every relevant column is identical.
- Business-key duplicate: same business key, different attributes or timestamps.

`SELECT DISTINCT` can remove exact duplicate rows but cannot choose the correct business-key version.

## 6.6 Top N per group exactly

```sql
WITH ranked AS (
    SELECT p.*,
           ROW_NUMBER() OVER (
               PARTITION BY category
               ORDER BY unit_price DESC, product_id
           ) AS rn
    FROM products AS p
)
SELECT *
FROM ranked
WHERE rn <= 3;
```

## 6.7 Top N distinct values including ties

```sql
WITH ranked AS (
    SELECT p.*,
           DENSE_RANK() OVER (
               PARTITION BY category
               ORDER BY unit_price DESC
           ) AS price_rank
    FROM products AS p
)
SELECT *
FROM ranked
WHERE price_rank <= 3;
```

## 6.8 First and last event per user

```sql
WITH ranked AS (
    SELECT e.*,
           ROW_NUMBER() OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS first_rn,
           ROW_NUMBER() OVER (
               PARTITION BY user_id
               ORDER BY event_time DESC, event_id DESC
           ) AS last_rn
    FROM user_events AS e
)
SELECT *
FROM ranked
WHERE first_rn = 1
   OR last_rn = 1;
```

## 6.9 QUALIFY alternative

Platforms supporting `QUALIFY` can write:

```sql
SELECT s.*
FROM customer_staging AS s
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY customer_id
    ORDER BY source_updated_at DESC, ingestion_time DESC, ingestion_id DESC
) = 1;
```

PostgreSQL requires a CTE or derived table. Keep the portable pattern for interviews unless a dialect is specified.

## 6.10 Delete duplicate rows cautiously

First materialize or inspect the exact keep/reject classification. Use a transaction and stable row identifier. Never delete using business key alone unless all versions are intended targets.

Read-only classification:

```sql
WITH ranked AS (
    SELECT s.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at DESC,
                        ingestion_time DESC,
                        ingestion_id DESC
           ) AS rn
    FROM customer_staging AS s
)
SELECT *,
       CASE WHEN rn = 1 THEN 'KEEP' ELSE 'REJECT_DUPLICATE' END AS action
FROM ranked;
```

## 6.11 Deduplication validation

After deduplication:

- Output rows should equal distinct business keys.
- No business key should appear more than once.
- Every output row should exist in source lineage.
- Rejected count should equal input count minus output count.
- The chosen row should match ordering and tiebreaker rules.
- Reprocessing the same batch should select the same rows.

## Day 6 common mistakes

- Using only timestamp ordering when ties are possible.
- Using `RANK` for one-row deduplication.
- Using `DISTINCT` to select latest business-key records.
- Filtering invalid records before deciding whether latest-valid or latest-any is required.
- Deleting before reviewing the keep/reject set.
- Losing source lineage and rejection counts.
- Using top-N with `RANK` when exactly N rows are required.

## Day 6 exercises

1. Rank staging versions newest first per customer.
2. Keep one latest row per customer deterministically.
3. Compare result with max-timestamp join.
4. Count received versions per customer.
5. Classify staging rows as keep or reject.
6. Validate output row count against distinct customer count.
7. Return top two products per category exactly.
8. Return top two distinct prices per category including ties.
9. Return latest order per customer.
10. Return first and latest event per user.
11. Explain why `RANK = 1` may return several rows.
12. Write a deduplication validation checklist.

## Day 6 solutions

```sql
-- 1 and 2
WITH ranked AS (
    SELECT s.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at DESC,
                        ingestion_time DESC,
                        ingestion_id DESC
           ) AS rn
    FROM customer_staging AS s
)
SELECT * FROM ranked WHERE rn = 1;

-- 3: demonstrates tied latest timestamps
WITH latest_timestamp AS (
    SELECT customer_id,
           MAX(source_updated_at) AS latest_updated_at
    FROM customer_staging
    GROUP BY customer_id
)
SELECT s.*
FROM customer_staging AS s
JOIN latest_timestamp AS l
  ON s.customer_id = l.customer_id
 AND s.source_updated_at = l.latest_updated_at;

-- 4
SELECT customer_id, COUNT(*) AS version_count
FROM customer_staging
GROUP BY customer_id;

-- 5
WITH ranked AS (
    SELECT s.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at DESC,
                        ingestion_time DESC,
                        ingestion_id DESC
           ) AS rn
    FROM customer_staging AS s
)
SELECT *,
       CASE WHEN rn = 1 THEN 'KEEP' ELSE 'REJECT_DUPLICATE' END AS action
FROM ranked;

-- 6
WITH deduplicated AS (
    SELECT s.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at DESC,
                        ingestion_time DESC,
                        ingestion_id DESC
           ) AS rn
    FROM customer_staging AS s
)
SELECT
    (SELECT COUNT(*) FROM deduplicated WHERE rn = 1) AS output_rows,
    (SELECT COUNT(DISTINCT customer_id) FROM customer_staging) AS distinct_keys;

-- 7
WITH ranked AS (
    SELECT p.*,
           ROW_NUMBER() OVER (
               PARTITION BY category
               ORDER BY unit_price DESC, product_id
           ) AS rn
    FROM products AS p
)
SELECT * FROM ranked WHERE rn <= 2;

-- 8
WITH ranked AS (
    SELECT p.*,
           DENSE_RANK() OVER (
               PARTITION BY category
               ORDER BY unit_price DESC
           ) AS price_rank
    FROM products AS p
)
SELECT * FROM ranked WHERE price_rank <= 2;

-- 9
WITH ranked AS (
    SELECT o.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY order_date DESC, order_id DESC
           ) AS rn
    FROM orders AS o
    WHERE customer_id IS NOT NULL
)
SELECT * FROM ranked WHERE rn = 1;

-- 10
WITH ranked AS (
    SELECT e.*,
           ROW_NUMBER() OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS first_rn,
           ROW_NUMBER() OVER (
               PARTITION BY user_id
               ORDER BY event_time DESC, event_id DESC
           ) AS last_rn
    FROM user_events AS e
)
SELECT *
FROM ranked
WHERE first_rn = 1 OR last_rn = 1;
```

---

# Week 5 mini-project — Event sequencing and sessionization

## Requirement

Create event-level output containing:

1. Event ID and user ID
2. Event type and time
3. Event sequence number per user
4. Previous event type and time
5. Time since previous event
6. New-session flag when inactivity is over 30 minutes
7. Session number per user
8. Event sequence inside each session
9. First and last event time for the session
10. Session duration

Then create one row per session with event count, purchase count, start, end, and duration.

## Event-level solution

```sql
WITH sequenced AS (
    -- Grain: one row per event
    SELECT e.*,
           ROW_NUMBER() OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS user_event_sequence,
           LAG(event_type) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS previous_event_type,
           LAG(event_time) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS previous_event_time
    FROM user_events AS e
),
session_flags AS (
    -- Grain: one row per event
    SELECT *,
           event_time - previous_event_time AS time_since_previous_event,
           CASE
               WHEN previous_event_time IS NULL THEN 1
               WHEN event_time - previous_event_time > INTERVAL '30 minutes' THEN 1
               ELSE 0
           END AS new_session_flag
    FROM sequenced
),
session_numbers AS (
    -- Grain: one row per event
    SELECT *,
           SUM(new_session_flag) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
           ) AS session_number
    FROM session_flags
),
session_enriched AS (
    -- Grain: one row per event
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY user_id, session_number
               ORDER BY event_time, event_id
           ) AS event_sequence_in_session,
           MIN(event_time) OVER (
               PARTITION BY user_id, session_number
           ) AS session_start_time,
           MAX(event_time) OVER (
               PARTITION BY user_id, session_number
           ) AS session_end_time
    FROM session_numbers
)
SELECT event_id,
       user_id,
       event_type,
       event_time,
       user_event_sequence,
       previous_event_type,
       previous_event_time,
       time_since_previous_event,
       new_session_flag,
       session_number,
       event_sequence_in_session,
       session_start_time,
       session_end_time,
       session_end_time - session_start_time AS session_duration
FROM session_enriched
ORDER BY user_id, event_time, event_id;
```

## Session-level solution

```sql
WITH sequenced AS (
    SELECT e.*,
           LAG(event_time) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS previous_event_time
    FROM user_events AS e
),
flags AS (
    SELECT *,
           CASE
               WHEN previous_event_time IS NULL THEN 1
               WHEN event_time - previous_event_time > INTERVAL '30 minutes' THEN 1
               ELSE 0
           END AS new_session_flag
    FROM sequenced
),
numbered AS (
    SELECT *,
           SUM(new_session_flag) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
           ) AS session_number
    FROM flags
)
SELECT user_id,
       session_number,
       MIN(event_time) AS session_start_time,
       MAX(event_time) AS session_end_time,
       MAX(event_time) - MIN(event_time) AS session_duration,
       COUNT(*) AS event_count,
       SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchase_count
FROM numbered
GROUP BY user_id, session_number
ORDER BY user_id, session_number;
```

## Mini-project validation

- Every event must appear exactly once.
- Every user must start with session number 1.
- Session number should never decrease.
- Every new-session flag should increment the session number by one.
- Sum session event counts must equal source event count.
- Session start must be less than or equal to session end.
- Event order must be deterministic for tied timestamps.

> [!note] Dialect note
> PostgreSQL interval syntax is used. In Databricks SQL or another platform, adapt timestamp-difference and interval expressions while preserving the logic.

---

# Week 5 interview questions

## Foundations

### 1. What is a window function?

A function that calculates across related rows while preserving the detail-row grain.

### 2. GROUP BY versus window function?

`GROUP BY` collapses rows into groups. A window function returns a value for each input row.

### 3. What does OVER do?

It defines the row set and ordering used by a window function.

### 4. What does PARTITION BY do?

It divides rows into independent calculation partitions without collapsing them.

### 5. Window ORDER BY versus final ORDER BY?

Window ordering controls calculation sequence. Final ordering controls result presentation.

### 6. Can a window function be used in WHERE?

Normally not in the same query block because windows are evaluated after `WHERE`. Use a CTE, derived table, or `QUALIFY` where supported.

### 7. Does a window function change grain?

No, unless a later query filters or aggregates the windowed result.

## Ranking

### 8. ROW_NUMBER versus RANK?

`ROW_NUMBER` assigns unique sequence numbers. `RANK` gives ties the same rank and leaves gaps.

### 9. RANK versus DENSE_RANK?

Both preserve ties; `RANK` leaves gaps and `DENSE_RANK` does not.

### 10. Which function is used for exactly one latest row per key?

`ROW_NUMBER` with deterministic descending ordering and a stable tiebreaker.

### 11. How do you return top three rows per category?

Assign `ROW_NUMBER` partitioned by category, ordered by the metric descending and a tiebreaker, then filter row number at most three.

### 12. How do you include ties in top N?

Use `RANK` or `DENSE_RANK` according to the ranking definition.

### 13. Why can RANK less than or equal to three return more than three rows?

Several rows can share qualifying ranks.

### 14. What does NTILE do?

It distributes ordered rows into a requested number of approximately equal buckets.

### 15. Does NTILE keep equal values in the same bucket?

Not necessarily. It distributes rows, so ties may be split.

## Offset and frames

### 16. What does LAG do?

It returns a value from a previous row in the window order.

### 17. What does LEAD do?

It returns a value from a following row in the window order.

### 18. Common LAG use cases?

Change detection, time gaps, period-over-period comparisons, and event sequencing.

### 19. What is a window frame?

The subset of ordered partition rows used for the current row calculation.

### 20. What does unbounded preceding to current row mean?

From the first row in the partition through the current row, commonly used for running totals.

### 21. ROWS versus RANGE?

`ROWS` counts physical ordered rows. `RANGE` considers ordering values and peers, with capabilities varying by engine.

### 22. Why specify ROWS explicitly for running totals?

It avoids default-frame and peer-row surprises when ordering values tie.

### 23. Why can LAST_VALUE return the current value?

The default frame often ends at the current row. Use an unbounded-following frame for the partition last value.

### 24. Seven-row moving average versus seven-day moving average?

A row window uses seven available records. A calendar-day metric needs a complete date series or an engine-supported time range.

## Deduplication

### 25. How do you deduplicate to the latest record?

Use `ROW_NUMBER` partitioned by business key and ordered by update timestamp and deterministic ingestion tiebreakers descending; keep row 1.

### 26. Why is MAX timestamp plus join insufficient?

Several records can share the maximum timestamp and all match back.

### 27. Why not use RANK for one-row deduplication?

Tied ordering values receive the same rank, so rank 1 can return several rows.

### 28. Why not use DISTINCT?

It removes identical result rows but does not choose the correct version among different rows sharing a business key.

### 29. What makes deduplication deterministic?

Ordering columns that uniquely resolve every tie, normally ending with a stable unique ingestion or source record ID.

### 30. What should be logged during deduplication?

Input count, distinct keys, kept count, rejected count, duplicate groups, chosen rule, and lineage identifiers.

## Data Engineer scenarios

### 31. Running total jumps for tied timestamps. Why?

A default or `RANGE` frame includes peer rows together. Use deterministic ordering and explicit `ROWS` frame.

### 32. Latest-row result changes between runs. Why?

The ordering has unresolved ties or uses unstable physical order.

### 33. Moving average ignores dates with no records. Why?

The input lacks those date rows. Join to a calendar table and fill zero or NULL according to the metric definition.

### 34. A window query is slow on distributed data. What do you inspect?

Partition size, skew, shuffles, sorting, spills, filters, selected columns, and whether pre-aggregation can reduce data.

### 35. How do you sessionize events?

Use `LAG` to calculate inactivity, flag new sessions, and cumulative-sum the flag within each user.

### 36. How do you detect a customer city change?

Order versions deterministically, compare city with `LAG(city)`, and handle the first row separately using row number.

### 37. Can several window functions share one specification?

Yes. Repeat the same partition/order or use a named window where the engine supports it.

### 38. When should you pre-aggregate before a window?

When the metric should operate at daily, monthly, customer, or another coarser grain rather than raw detail.

### 39. QUALIFY versus CTE filtering?

`QUALIFY` filters window results in supported engines. A CTE or derived table is more portable.

### 40. What is the most important window-function interview habit?

State partition, ordering, frame, tie behavior, and output grain before writing the function.

---

# Week 5 final practice set

## Foundations and ranking

1. Display every order with overall total.
2. Display every order with customer total.
3. Display each product with category average.
4. Number orders per customer chronologically.
5. Number orders per customer newest first.
6. Rank order totals globally.
7. Dense-rank product prices by category.
8. Return latest order per customer.
9. Return first order per customer.
10. Return top three products per category exactly.
11. Return top three distinct prices per category with ties.
12. Return second-highest distinct order total.

## LAG and LEAD

13. Return previous order date per customer.
14. Return next order total per customer.
15. Calculate amount change per customer.
16. Calculate days between orders.
17. Detect customer city changes.
18. Detect customer email changes.
19. Return previous event type.
20. Calculate event inactivity interval.
21. Find events immediately before a purchase.
22. Flag new sessions after 30 minutes.

## Aggregate windows and frames

23. Calculate global running order total.
24. Calculate customer running total.
25. Calculate daily completed revenue.
26. Add cumulative revenue.
27. Add seven-row moving average.
28. Add seven-row moving sum.
29. Add remaining revenue.
30. Add cumulative percentage.
31. Display first and latest order date per customer.
32. Display first and latest order status per customer.
33. Display second order date per customer.
34. Compare `ROWS` and `RANGE` on tied events.

## Deduplication and Data Engineering

35. Count staging versions per customer.
36. Keep latest staging row per customer.
37. Classify rows as keep or duplicate reject.
38. Validate deduplicated row count.
39. Show max-timestamp tie behavior.
40. Return first and last event per user.
41. Build event sequence numbers.
42. Build session numbers.
43. Build session-level metrics.
44. Calculate month-over-month revenue growth.
45. Rank customers by completed revenue.
46. Segment customers into revenue quartiles.
47. Find customers above their customer-average order total.
48. Create a daily anomaly flag when revenue exceeds its trailing average significantly.
49. Explain how to fill missing dates before a true seven-day window.
50. Write a production deduplication validation checklist.

## Selected final solutions

```sql
-- 12
WITH ranked AS (
    SELECT order_total,
           DENSE_RANK() OVER (ORDER BY order_total DESC) AS value_rank
    FROM orders
    WHERE order_total IS NOT NULL
)
SELECT MAX(order_total) AS second_highest_total
FROM ranked
WHERE value_rank = 2;

-- 21
WITH sequenced AS (
    SELECT e.*,
           LEAD(event_type) OVER (
               PARTITION BY user_id
               ORDER BY event_time, event_id
           ) AS next_event_type
    FROM user_events AS e
)
SELECT *
FROM sequenced
WHERE next_event_type = 'purchase';

-- 23 and 24
SELECT order_id,
       customer_id,
       order_date,
       order_total,
       SUM(order_total) OVER (
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS global_running_total,
       SUM(order_total) OVER (
           PARTITION BY customer_id
           ORDER BY order_date, order_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS customer_running_total
FROM orders;

-- 26 through 30
WITH daily AS (
    SELECT order_date,
           SUM(CASE WHEN order_status = 'COMPLETED' THEN order_total ELSE 0 END)
               AS daily_revenue
    FROM orders
    GROUP BY order_date
),
windowed AS (
    SELECT order_date,
           daily_revenue,
           SUM(daily_revenue) OVER (
               ORDER BY order_date
               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
           ) AS cumulative_revenue,
           AVG(daily_revenue) OVER (
               ORDER BY order_date
               ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
           ) AS seven_row_average,
           SUM(daily_revenue) OVER (
               ORDER BY order_date
               ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
           ) AS seven_row_sum,
           SUM(daily_revenue) OVER (
               ORDER BY order_date
               ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
           ) AS remaining_revenue,
           SUM(daily_revenue) OVER () AS overall_revenue
    FROM daily
)
SELECT *,
       100.0 * cumulative_revenue / NULLIF(overall_revenue, 0)
           AS cumulative_percentage
FROM windowed;

-- 36 and 37
WITH ranked AS (
    SELECT s.*,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY source_updated_at DESC,
                        ingestion_time DESC,
                        ingestion_id DESC
           ) AS rn
    FROM customer_staging AS s
)
SELECT *,
       CASE WHEN rn = 1 THEN 'KEEP' ELSE 'REJECT_DUPLICATE' END AS action
FROM ranked;

-- 44
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date) AS month_start,
           SUM(CASE WHEN order_status = 'COMPLETED' THEN order_total ELSE 0 END)
               AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
),
with_previous AS (
    SELECT month_start,
           revenue,
           LAG(revenue) OVER (ORDER BY month_start) AS previous_revenue
    FROM monthly
)
SELECT *,
       100.0 * (revenue - previous_revenue)
           / NULLIF(previous_revenue, 0) AS growth_percentage
FROM with_previous;

-- 45 and 46
WITH customer_revenue AS (
    SELECT customer_id,
           SUM(CASE WHEN order_status = 'COMPLETED' THEN order_total ELSE 0 END)
               AS completed_revenue
    FROM orders
    WHERE customer_id IS NOT NULL
    GROUP BY customer_id
)
SELECT customer_id,
       completed_revenue,
       DENSE_RANK() OVER (ORDER BY completed_revenue DESC) AS revenue_rank,
       NTILE(4) OVER (ORDER BY completed_revenue DESC, customer_id) AS revenue_quartile
FROM customer_revenue;

-- 47
WITH enriched AS (
    SELECT o.*,
           AVG(order_total) OVER (
               PARTITION BY customer_id
           ) AS customer_average
    FROM orders AS o
    WHERE customer_id IS NOT NULL
)
SELECT *
FROM enriched
WHERE order_total > customer_average;

-- 48: simple trailing-average anomaly rule
WITH daily AS (
    SELECT order_date,
           SUM(CASE WHEN order_status = 'COMPLETED' THEN order_total ELSE 0 END)
               AS daily_revenue
    FROM orders
    GROUP BY order_date
),
scored AS (
    SELECT order_date,
           daily_revenue,
           AVG(daily_revenue) OVER (
               ORDER BY order_date
               ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING
           ) AS prior_seven_row_average
    FROM daily
)
SELECT *,
       CASE
           WHEN prior_seven_row_average IS NOT NULL
            AND daily_revenue > 2 * prior_seven_row_average
           THEN 'ANOMALY'
           ELSE 'NORMAL'
       END AS anomaly_status
FROM scored;
```

---

# Week 5 one-page cheat sheet

## Basic window

```sql
function(expression) OVER (
    PARTITION BY group_key
    ORDER BY sequence_column, unique_tiebreaker
    ROWS BETWEEN frame_start AND frame_end
)
```

## Latest row per key

```sql
WITH ranked AS (
    SELECT s.*,
           ROW_NUMBER() OVER (
               PARTITION BY business_key
               ORDER BY updated_at DESC,
                        ingestion_time DESC,
                        unique_ingestion_id DESC
           ) AS rn
    FROM staging AS s
)
SELECT *
FROM ranked
WHERE rn = 1;
```

## Top N per group

```sql
WITH ranked AS (
    SELECT t.*,
           ROW_NUMBER() OVER (
               PARTITION BY group_key
               ORDER BY metric DESC, unique_id
           ) AS rn
    FROM source AS t
)
SELECT * FROM ranked WHERE rn <= 3;
```

## Previous and next values

```sql
LAG(value) OVER (
    PARTITION BY business_key
    ORDER BY event_time, unique_id
)

LEAD(value) OVER (
    PARTITION BY business_key
    ORDER BY event_time, unique_id
)
```

## Running total

```sql
SUM(amount) OVER (
    PARTITION BY business_key
    ORDER BY business_date, unique_id
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

## Moving average

```sql
AVG(daily_metric) OVER (
    ORDER BY business_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
)
```

## Whole-partition first and last values

```sql
FIRST_VALUE(value) OVER (
    PARTITION BY business_key
    ORDER BY event_time, unique_id
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)

LAST_VALUE(value) OVER (
    PARTITION BY business_key
    ORDER BY event_time, unique_id
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

## Ranking choice

| Requirement | Function |
|---|---|
| One deterministic row per position | `ROW_NUMBER` |
| Ties share rank with gaps | `RANK` |
| Ties share rank without gaps | `DENSE_RANK` |
| Approximately equal row buckets | `NTILE` |

## Window validation checklist

- [ ] Input grain declared
- [ ] Partition key correct
- [ ] Ordering columns correct
- [ ] Stable tiebreaker included where needed
- [ ] Frame written explicitly
- [ ] NULL ordering decided
- [ ] Tie behavior tested
- [ ] First/last partition rows tested
- [ ] Missing dates considered
- [ ] Output grain validated

## Week 5 completion test

You have completed Week 5 when you can:

- Explain windows versus grouped aggregation.
- Choose the correct ranking function.
- Write deterministic latest-row and top-N queries.
- Detect changes and time gaps with `LAG`.
- Calculate running and moving metrics using explicit frames.
- Explain `ROWS`, `RANGE`, and peer rows.
- Use `LAST_VALUE` without the default-frame bug.
- Validate deduplication counts and keys.
- Sessionize events with a gap rule.

## Next week preview

Week 6 covers database engineering foundations:

- DDL and DML
- Data types and constraints
- Transactions and isolation concepts
- Indexes and query plans
- Partitioning and pruning
- Query optimization checklist
- Reliable schema design for Data Engineering
