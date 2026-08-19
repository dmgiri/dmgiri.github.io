---
aliases:
  - DE SQL Week 1
  - SQL Beginner Week 1
tags:
  - data-engineering
  - sql
  - sql-fundamentals
  - interview-preparation
  - week-1
status: active
difficulty: beginner
study_time: 6 days
created: 2026-08-16
---

# Data Engineer SQL — Week 1 Detailed Notes

> [!info] How to use this note
> Complete one day at a time. Run every SQL example, solve the exercises before opening the solutions, and mark finished sections with the checkboxes below.

## Obsidian navigation

- [[#Day 1 — SQL and SELECT fundamentals|Day 1 — SELECT fundamentals]]
- [[#Day 2 — WHERE and Boolean conditions|Day 2 — Filtering]]
- [[#Day 3 — IN, BETWEEN, LIKE, and NULL|Day 3 — Patterns and NULL]]
- [[#Day 4 — ORDER BY, LIMIT, and DISTINCT|Day 4 — Sorting and DISTINCT]]
- [[#Day 5 — String, numeric, date, NULL, and conversion functions|Day 5 — Functions]]
- [[#Day 6 — Revision, Data Engineer application, and assessment|Day 6 — Revision and assessment]]
- [[#Week 1 one-page cheat sheet|Week 1 cheat sheet]]

## Progress tracker

- [x] Day 1 completed
- [x] Day 2 completed
- [ ] Day 3 completed
- [ ] Day 4 completed
- [ ] Day 5 completed
- [ ] Day 6 completed
- [ ] Week 1 completion test passed

> [!tip] Obsidian tip
> Use Reading view for clean notes, Source view to edit SQL, and search for the tag `#sql-fundamentals` to find this note later.


**Level:** Beginner  
**Duration:** 6 study days plus 1 revision/rest day  
**Daily time:** 60–90 minutes  
**Primary dialect:** PostgreSQL-style SQL  

These notes build the foundation needed for Data Engineering SQL. The examples use a retail dataset and introduce habits that matter in production: selecting only required columns, using deterministic ordering, handling NULL correctly, writing readable filters, and checking data quality.

## Week 1 learning outcomes

By the end of the week, you should be able to:

- Explain databases, schemas, tables, rows, columns, keys, and data types.
- Retrieve all or selected columns with `SELECT`.
- Create aliases and calculated columns.
- Filter records with `WHERE` and comparison operators.
- Combine conditions using `AND`, `OR`, and `NOT`.
- Use `IN`, `BETWEEN`, `LIKE`, and `IS NULL` correctly.
- Sort results with `ORDER BY`.
- Remove duplicate result rows with `DISTINCT`.
- Limit output and write deterministic top-N queries.
- Use common string, numeric, date, NULL, and conversion functions.
- Solve a small source-data inspection task independently.

---

# Practice environment

## The retail dataset

Run the following statements in PostgreSQL or adapt the data types to your SQL platform.

```sql
CREATE TABLE customers (
    customer_id      INTEGER PRIMARY KEY,
    customer_name    VARCHAR(100) NOT NULL,
    email            VARCHAR(200),
    city             VARCHAR(100),
    country          VARCHAR(100) NOT NULL,
    signup_date      DATE NOT NULL,
    is_active        BOOLEAN NOT NULL,
    credit_limit     DECIMAL(12,2),
    updated_at       TIMESTAMP NOT NULL
);

CREATE TABLE products (
    product_id       INTEGER PRIMARY KEY,
    product_name     VARCHAR(150) NOT NULL,
    category         VARCHAR(100) NOT NULL,
    unit_price       DECIMAL(12,2) NOT NULL,
    stock_quantity   INTEGER,
    created_at       TIMESTAMP NOT NULL
);

CREATE TABLE orders (
    order_id         INTEGER PRIMARY KEY,
    customer_id      INTEGER,
    order_date       DATE NOT NULL,
    order_status     VARCHAR(30) NOT NULL,
    order_total      DECIMAL(14,2),
    sales_channel    VARCHAR(30),
    updated_at       TIMESTAMP NOT NULL
);
```

```sql
INSERT INTO customers VALUES
(1, 'Aarav Sharma',  'aarav@example.com',  'Pune',      'India', '2025-12-15', TRUE,  50000.00, '2026-08-01 10:00:00'),
(2, 'Diya Patel',    'diya@example.com',   'Mumbai',    'India', '2026-01-10', TRUE,  75000.00, '2026-08-02 11:30:00'),
(3, 'Kabir Singh',   NULL,                 'Pune',      'India', '2026-02-05', FALSE, 25000.00, '2026-07-20 09:15:00'),
(4, 'Meera Iyer',    'meera@example.com',  'Bengaluru', 'India', '2026-02-15', TRUE,  NULL,     '2026-08-03 12:00:00'),
(5, 'Rohan Das',     'rohan@example.com',  NULL,        'India', '2026-03-01', TRUE,  40000.00, '2026-08-04 14:45:00'),
(6, 'Sara Khan',     'sara@example.com',   'Pune',      'India', '2026-03-12', TRUE,  90000.00, '2026-08-05 08:20:00'),
(7, 'Vikram Rao',    'vikram@example.com', 'Hyderabad', 'India', '2026-04-01', FALSE, 30000.00, '2026-08-06 18:10:00'),
(8, 'Anaya Gupta',   'anaya@example.com',  'Pune',      'India', '2026-04-20', TRUE,  60000.00, '2026-08-07 16:30:00');

INSERT INTO products VALUES
(101, 'Laptop Pro',         'Electronics', 85000.00,  8,  '2025-11-01 09:00:00'),
(102, 'Wireless Mouse',     'Electronics',  1200.00, 50,  '2025-11-02 09:00:00'),
(103, 'Office Chair',       'Furniture',   15000.00, 12,  '2025-11-03 09:00:00'),
(104, 'Standing Desk',      'Furniture',   28000.00, NULL,'2025-11-04 09:00:00'),
(105, 'SQL Fundamentals',   'Books',         799.00, 35,  '2025-11-05 09:00:00'),
(106, 'Data Engineering',   'Books',        1299.00, 20,  '2025-11-06 09:00:00'),
(107, 'USB-C Hub',          'Electronics',  3500.00,  0,  '2025-11-07 09:00:00'),
(108, 'Notebook Pack',      'Stationery',    450.00, 75,  '2025-11-08 09:00:00');

INSERT INTO orders VALUES
(1001, 1, '2026-07-01', 'COMPLETED',  2400.00, 'WEB',    '2026-07-01 11:00:00'),
(1002, 2, '2026-07-02', 'SHIPPED',   85000.00, 'APP',    '2026-07-03 08:00:00'),
(1003, 1, '2026-07-04', 'CANCELLED',  3500.00, 'WEB',    '2026-07-04 13:00:00'),
(1004, 3, '2026-07-05', 'COMPLETED', 15000.00, 'STORE',  '2026-07-05 17:00:00'),
(1005, 4, '2026-07-07', 'PENDING',       NULL, 'APP',    '2026-07-07 10:00:00'),
(1006, 6, '2026-07-10', 'COMPLETED', 28000.00, 'WEB',    '2026-07-10 20:00:00'),
(1007, 8, '2026-07-12', 'SHIPPED',    1299.00, 'APP',    '2026-07-13 07:30:00'),
(1008, 6, '2026-07-15', 'COMPLETED',  4700.00, 'STORE',  '2026-07-15 15:00:00'),
(1009, 7, '2026-07-18', 'CANCELLED',  1200.00, 'WEB',    '2026-07-18 09:45:00'),
(1010, 2, '2026-07-20', 'COMPLETED', 15000.00, 'APP',    '2026-07-20 19:15:00'),
(1011, NULL, '2026-07-21', 'COMPLETED', 799.00, 'STORE', '2026-07-21 12:00:00'),
(1012, 8, '2026-07-25', 'PENDING',    3500.00, NULL,     '2026-07-25 16:00:00');
```

## Before querying the data

Ask these five questions whenever you receive a new table:

1. What does one row represent?
2. Which column or combination of columns uniquely identifies a row?
3. Which columns can contain NULL?
4. What are the expected values and data types?
5. Which timestamps represent the business event and ingestion/update time?

These questions prevent many downstream mistakes.

---

# Day 1 — SQL and SELECT fundamentals

## 1.1 Database concepts

### Database

A database is an organized collection of data managed by a database system.

### Schema

A schema is a named logical container for tables, views, functions, and other objects. A fully qualified table name may look like:

```sql
SELECT *
FROM sales.orders;
```

Here, `sales` is the schema and `orders` is the table.

### Table

A table stores related data in rows and columns.

- In `customers`, one row represents one customer.
- In `orders`, one row represents one order.
- This row meaning is called the **grain**.

Always identify grain before joining or aggregating tables.

### Row and column

- A row is one record or instance.
- A column is one attribute of that record.
- A data type defines which values a column can store.

### Common data types

| Category | Examples | Used for |
|---|---|---|
| Integer | `INTEGER`, `BIGINT` | IDs, counts, quantities |
| Decimal | `DECIMAL(12,2)` | Money and exact numeric values |
| Approximate numeric | `FLOAT`, `DOUBLE` | Scientific or approximate calculations |
| Text | `VARCHAR`, `CHAR`, `STRING` | Names, statuses, descriptions |
| Date/time | `DATE`, `TIMESTAMP` | Business dates and event times |
| Boolean | `BOOLEAN` | True/false flags |

For money, prefer an exact decimal type instead of floating point.

## 1.2 Basic SELECT syntax

`SELECT` reads data from a table.

```sql
SELECT column_1, column_2
FROM table_name;
```

Return all columns:

```sql
SELECT *
FROM customers;
```

Return only required columns:

```sql
SELECT customer_id, customer_name, city
FROM customers;
```

### Why avoid SELECT star in production?

`SELECT *` is useful during exploration, but production transformations should normally name required columns because:

- Unneeded columns increase data reading and movement.
- A new source column can unexpectedly change the output schema.
- Column intent is less clear.
- Duplicate column names become confusing after joins.
- Sensitive columns may be exposed accidentally.

## 1.3 Column aliases

Aliases change output headings; they do not rename the source column.

```sql
SELECT customer_name AS name,
       signup_date AS joined_on
FROM customers;
```

`AS` is often optional, but including it makes column aliases easy to recognize.

Table aliases reduce repetition:

```sql
SELECT c.customer_id,
       c.customer_name,
       c.city
FROM customers AS c;
```

## 1.4 Literals and calculated columns

A literal is a fixed value written in the query.

```sql
SELECT customer_id,
       customer_name,
       'CRM' AS source_system
FROM customers;
```

Calculated columns are expressions evaluated for each row.

```sql
SELECT product_id,
       product_name,
       unit_price,
       unit_price * 1.18 AS price_with_tax
FROM products;
```

Operator precedence applies. Use parentheses to make business logic obvious:

```sql
SELECT product_name,
       unit_price,
       unit_price + (unit_price * 0.18) AS price_with_tax
FROM products;
```

## 1.5 Semicolons, formatting, and comments

A semicolon ends a SQL statement.

Single-line comment:

```sql
-- Return the customer dimension columns required by the report
SELECT customer_id, customer_name
FROM customers;
```

Multi-line comment:

```sql
/*
Daily source inspection query.
Owner: Data Engineering team.
*/
SELECT order_id, order_date
FROM orders;
```

Readable format:

```sql
SELECT
    customer_id,
    customer_name,
    signup_date
FROM customers;
```

## Day 1 common mistakes

- Misspelling a table or column.
- Forgetting commas between selected columns.
- Placing a comma after the final selected column.
- Using single quotes around column names. Single quotes create text literals.
- Assuming output row order without `ORDER BY`.
- Using `SELECT *` in a permanent pipeline.

## Day 1 exercises

1. Return every column from `products`.
2. Return `product_id`, `product_name`, and `unit_price`.
3. Display `product_name` as `name` and `unit_price` as `price`.
4. Add a literal column called `source_system` with the value `ERP`.
5. Calculate a 10 percent discounted price.
6. Calculate inventory value as `unit_price * stock_quantity`.
7. Return customer ID, name, and signup date using table alias `c`.
8. Explain the grain of each practice table.

## Day 1 solutions

```sql
-- 1
SELECT * FROM products;

-- 2
SELECT product_id, product_name, unit_price
FROM products;

-- 3
SELECT product_name AS name,
       unit_price AS price
FROM products;

-- 4
SELECT product_id,
       product_name,
       'ERP' AS source_system
FROM products;

-- 5
SELECT product_id,
       product_name,
       unit_price,
       unit_price * 0.90 AS discounted_price
FROM products;

-- 6
SELECT product_id,
       product_name,
       unit_price * stock_quantity AS inventory_value
FROM products;

-- 7
SELECT c.customer_id,
       c.customer_name,
       c.signup_date
FROM customers AS c;
```

Grain answers:

- `customers`: one customer per row.
- `products`: one product per row.
- `orders`: one order per row.

---

# Day 2 — WHERE and Boolean conditions

## 2.1 Purpose of WHERE

`WHERE` keeps only rows whose condition evaluates to true.

```sql
SELECT order_id, order_status, order_total
FROM orders
WHERE order_status = 'COMPLETED';
```

SQL conceptually evaluates `FROM` before `WHERE`, then constructs the `SELECT` output.

## 2.2 Comparison operators

| Operator | Meaning |
|---|---|
| `=` | Equal |
| `<>` or `!=` | Not equal |
| `>` | Greater than |
| `<` | Less than |
| `>=` | Greater than or equal |
| `<=` | Less than or equal |

Examples:

```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price >= 5000;
```

```sql
SELECT customer_name, signup_date
FROM customers
WHERE signup_date < '2026-03-01';
```

Use ISO date literals in the form `YYYY-MM-DD` when supported.

## 2.3 AND

All conditions connected by `AND` must be true.

```sql
SELECT order_id, order_status, order_total
FROM orders
WHERE order_status = 'COMPLETED'
  AND order_total >= 10000;
```

## 2.4 OR

At least one condition connected by `OR` must be true.

```sql
SELECT customer_id, customer_name, city
FROM customers
WHERE city = 'Pune'
   OR city = 'Mumbai';
```

## 2.5 NOT

`NOT` reverses a Boolean condition.

```sql
SELECT order_id, order_status
FROM orders
WHERE NOT order_status = 'CANCELLED';
```

A clearer equivalent is often:

```sql
WHERE order_status <> 'CANCELLED'
```

## 2.6 Operator precedence

`AND` is evaluated before `OR`. Parentheses make the intended business rule explicit.

Potentially wrong:

```sql
WHERE country = 'India'
  AND city = 'Pune'
   OR city = 'Mumbai'
```

This means:

```text
(country is India AND city is Pune) OR city is Mumbai
```

Safer version:

```sql
WHERE country = 'India'
  AND (city = 'Pune' OR city = 'Mumbai')
```

## 2.7 Filtering Boolean values

PostgreSQL-style:

```sql
SELECT customer_id, customer_name
FROM customers
WHERE is_active = TRUE;
```

Some systems use `1` and `0` or another representation. Check the actual data type.

## 2.8 Case sensitivity

String comparison behavior depends on the database, collation, and function used. A value stored as `completed` may not equal `COMPLETED`.

Portable normalization pattern:

```sql
WHERE UPPER(order_status) = 'COMPLETED'
```

Be careful: applying a function to a filtered column can affect index use or data skipping. Standardize important status values during ingestion when possible.

## Day 2 common mistakes

- Using `==` instead of `=`.
- Mixing `AND` and `OR` without parentheses.
- Comparing numbers as text.
- Using double quotes or no quotes incorrectly for text literals.
- Assuming text comparisons are case-insensitive.
- Filtering a timestamp with only an end date and unintentionally excluding part of that date.

## Day 2 exercises

1. Find products costing more than 10,000.
2. Find products with stock below 10.
3. Find completed orders worth at least 10,000.
4. Find customers who signed up before 1 March 2026.
5. Find active Pune customers.
6. Find orders from the web channel that are completed or shipped.
7. Find products outside the Books category.
8. Explain how the result could change if parentheses are removed from exercise 6.

## Day 2 solutions

```sql
-- 1
SELECT product_id, product_name, unit_price
FROM products
WHERE unit_price > 10000;

-- 2
SELECT product_id, product_name, stock_quantity
FROM products
WHERE stock_quantity < 10;

-- 3
SELECT order_id, order_total
FROM orders
WHERE order_status = 'COMPLETED'
  AND order_total >= 10000;

-- 4
SELECT customer_id, customer_name, signup_date
FROM customers
WHERE signup_date < '2026-03-01';

-- 5
SELECT customer_id, customer_name
FROM customers
WHERE is_active = TRUE
  AND city = 'Pune';

-- 6
SELECT order_id, order_status, sales_channel
FROM orders
WHERE sales_channel = 'WEB'
  AND (order_status = 'COMPLETED' OR order_status = 'SHIPPED');

-- 7
SELECT product_id, product_name, category
FROM products
WHERE category <> 'Books';
```

---

# Day 3 — IN, BETWEEN, LIKE, and NULL

## 3.1 IN

`IN` checks whether a value matches any item in a list.

```sql
SELECT order_id, order_status
FROM orders
WHERE order_status IN ('COMPLETED', 'SHIPPED');
```

Equivalent but less concise:

```sql
WHERE order_status = 'COMPLETED'
   OR order_status = 'SHIPPED'
```

`NOT IN` excludes listed values:

```sql
WHERE order_status NOT IN ('CANCELLED', 'PENDING')
```

Advanced warning: `NOT IN` can behave unexpectedly if a subquery returns NULL. Later, prefer `NOT EXISTS` for anti-join logic.

## 3.2 BETWEEN

`BETWEEN` includes both boundaries.

```sql
SELECT order_id, order_date
FROM orders
WHERE order_date BETWEEN '2026-07-05' AND '2026-07-15';
```

Equivalent:

```sql
WHERE order_date >= '2026-07-05'
  AND order_date <= '2026-07-15'
```

For timestamps, a half-open interval is normally safer:

```sql
WHERE updated_at >= '2026-07-01 00:00:00'
  AND updated_at <  '2026-08-01 00:00:00'
```

This includes all of July without depending on fractional-second precision.

## 3.3 LIKE

`LIKE` searches text using wildcards.

| Wildcard | Meaning |
|---|---|
| `%` | Zero or more characters |
| `_` | Exactly one character |

Names starting with `S`:

```sql
SELECT customer_name
FROM customers
WHERE customer_name LIKE 'S%';
```

Names containing `ar`:

```sql
WHERE customer_name LIKE '%ar%'
```

Exactly five-character text:

```sql
WHERE some_column LIKE '_____'
```

PostgreSQL supports `ILIKE` for case-insensitive pattern matching. For portability:

```sql
WHERE LOWER(customer_name) LIKE '%ara%'
```

Patterns beginning with `%` can require broad scanning because the starting characters are unknown.

## 3.4 NULL

NULL means missing or unknown. It is not:

- zero,
- an empty string,
- the text `NULL`, or
- automatically equal to another NULL.

Correct NULL filter:

```sql
SELECT customer_id, customer_name
FROM customers
WHERE email IS NULL;
```

Incorrect:

```sql
WHERE email = NULL
```

Find known values:

```sql
WHERE email IS NOT NULL
```

## 3.5 Three-valued logic

A SQL condition can evaluate to:

- true,
- false,
- unknown.

`WHERE` keeps only true rows. Comparisons involving NULL usually become unknown, so they are not returned.

Suppose `stock_quantity` is NULL:

```sql
WHERE stock_quantity <> 0
```

The NULL row is not returned because NULL is unknown, not nonzero. If the requirement includes unknown inventory, write:

```sql
WHERE stock_quantity <> 0
   OR stock_quantity IS NULL
```

## 3.6 COALESCE introduction

`COALESCE` returns the first non-NULL expression.

```sql
SELECT customer_name,
       COALESCE(city, 'Unknown') AS city
FROM customers;
```

For calculations:

```sql
SELECT product_name,
       unit_price * COALESCE(stock_quantity, 0) AS known_inventory_value
FROM products;
```

Do not replace NULL blindly. Unknown stock and zero stock have different business meanings.

## Day 3 common mistakes

- Assuming `BETWEEN` excludes boundaries.
- Using `BETWEEN` carelessly with timestamps.
- Using `=` or `<>` with NULL.
- Assuming `NOT IN` returns NULL-valued rows.
- Treating missing data as zero without confirming business meaning.
- Forgetting that `_` is a one-character wildcard.

## Day 3 exercises

1. Find orders in `COMPLETED`, `SHIPPED`, or `PENDING` status.
2. Find orders placed from 5 July through 15 July 2026.
3. Find products priced between 1,000 and 20,000 inclusive.
4. Find customer names beginning with `A`.
5. Find product names containing `Data`.
6. Find customers without an email.
7. Find customers without a city.
8. Find products with unknown stock.
9. Display customer city as `Unknown` when missing.
10. Explain why `stock_quantity <> 0` does not return unknown stock.

## Day 3 solutions

```sql
-- 1
SELECT *
FROM orders
WHERE order_status IN ('COMPLETED', 'SHIPPED', 'PENDING');

-- 2
SELECT *
FROM orders
WHERE order_date BETWEEN '2026-07-05' AND '2026-07-15';

-- 3
SELECT product_name, unit_price
FROM products
WHERE unit_price BETWEEN 1000 AND 20000;

-- 4
SELECT customer_name
FROM customers
WHERE customer_name LIKE 'A%';

-- 5
SELECT product_name
FROM products
WHERE product_name LIKE '%Data%';

-- 6
SELECT customer_id, customer_name
FROM customers
WHERE email IS NULL;

-- 7
SELECT customer_id, customer_name
FROM customers
WHERE city IS NULL;

-- 8
SELECT product_id, product_name
FROM products
WHERE stock_quantity IS NULL;

-- 9
SELECT customer_id,
       customer_name,
       COALESCE(city, 'Unknown') AS city
FROM customers;
```

---

# Day 4 — ORDER BY, LIMIT, and DISTINCT

## 4.1 SQL does not guarantee default order

Without `ORDER BY`, result order is unspecified. A database may return rows differently after data changes, optimization, scaling, or execution on multiple workers.

## 4.2 ORDER BY ascending

Ascending order is the default.

```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price ASC;
```

Equivalent:

```sql
ORDER BY unit_price
```

## 4.3 ORDER BY descending

```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC;
```

## 4.4 Multiple sort columns

The second sort column resolves ties in the first.

```sql
SELECT category, product_name, unit_price
FROM products
ORDER BY category ASC,
         unit_price DESC,
         product_id ASC;
```

This sorts by category, then most expensive product, then product ID.

## 4.5 Sorting by expressions and aliases

```sql
SELECT product_name,
       unit_price * COALESCE(stock_quantity, 0) AS inventory_value
FROM products
ORDER BY inventory_value DESC;
```

Most engines allow a `SELECT` alias in `ORDER BY` because ordering is logically processed after projection.

## 4.6 NULL ordering

Default placement of NULL values can differ across SQL engines. PostgreSQL allows explicit control:

```sql
ORDER BY stock_quantity ASC NULLS LAST
```

For portability, use a `CASE` expression or check engine behavior.

## 4.7 LIMIT

Return only a specified number of rows:

```sql
SELECT order_id, order_total
FROM orders
ORDER BY order_total DESC
LIMIT 5;
```

`LIMIT` syntax is not universal. Standards-oriented or other engines may use `FETCH FIRST`, `TOP`, or different syntax.

### Deterministic top N

If multiple rows share the same amount, add a unique tiebreaker:

```sql
SELECT order_id, order_total
FROM orders
WHERE order_total IS NOT NULL
ORDER BY order_total DESC,
         order_id ASC
LIMIT 5;
```

Without a tiebreaker, which tied rows appear can vary.

## 4.8 DISTINCT

Return unique status values:

```sql
SELECT DISTINCT order_status
FROM orders;
```

With multiple selected columns, distinctness applies to the complete combination:

```sql
SELECT DISTINCT order_status, sales_channel
FROM orders;
```

This returns unique status-channel pairs, not independently unique values from each column.

### DISTINCT is not a duplicate-data repair strategy

Do not add `DISTINCT` simply because a join unexpectedly produces duplicates. First determine:

- the grain of both tables,
- whether keys are unique,
- whether the join condition is complete, and
- which record should be kept.

`DISTINCT` may hide a data-quality or join-grain bug.

## 4.9 Logical query order for Week 1

Conceptually:

1. `FROM`
2. `WHERE`
3. `SELECT`
4. `DISTINCT`
5. `ORDER BY`
6. `LIMIT`

Example:

```sql
SELECT DISTINCT sales_channel
FROM orders
WHERE order_status = 'COMPLETED'
ORDER BY sales_channel
LIMIT 3;
```

## Day 4 common mistakes

- Expecting consistent order without `ORDER BY`.
- Applying `LIMIT` without ordering when the requirement is top or latest.
- Forgetting a deterministic tiebreaker.
- Thinking `DISTINCT` applies to only the first selected column.
- Using `DISTINCT` to conceal an incorrect join.
- Assuming every engine sorts NULL the same way.

## Day 4 exercises

1. Sort customers by newest signup date first.
2. Sort products by category ascending and price descending.
3. Return the three most expensive products.
4. Return the four earliest orders.
5. List unique order statuses alphabetically.
6. List unique status-channel combinations.
7. Return the two most recently updated active customers.
8. Write a deterministic query for the five lowest-priced products.

## Day 4 solutions

```sql
-- 1
SELECT customer_id, customer_name, signup_date
FROM customers
ORDER BY signup_date DESC, customer_id ASC;

-- 2
SELECT product_id, product_name, category, unit_price
FROM products
ORDER BY category ASC, unit_price DESC, product_id ASC;

-- 3
SELECT product_id, product_name, unit_price
FROM products
ORDER BY unit_price DESC, product_id ASC
LIMIT 3;

-- 4
SELECT order_id, order_date
FROM orders
ORDER BY order_date ASC, order_id ASC
LIMIT 4;

-- 5
SELECT DISTINCT order_status
FROM orders
ORDER BY order_status;

-- 6
SELECT DISTINCT order_status, sales_channel
FROM orders
ORDER BY order_status, sales_channel;

-- 7
SELECT customer_id, customer_name, updated_at
FROM customers
WHERE is_active = TRUE
ORDER BY updated_at DESC, customer_id DESC
LIMIT 2;

-- 8
SELECT product_id, product_name, unit_price
FROM products
ORDER BY unit_price ASC, product_id ASC
LIMIT 5;
```

---

# Day 5 — String, numeric, date, NULL, and conversion functions

Function names and exact behavior can differ across SQL platforms. Understand the purpose, then verify syntax for the engine you use.

## 5.1 String functions

### UPPER and LOWER

```sql
SELECT customer_name,
       UPPER(customer_name) AS upper_name,
       LOWER(customer_name) AS lower_name
FROM customers;
```

Useful for normalization and comparisons, although data should ideally be standardized before frequent querying.

### TRIM

Remove leading and trailing spaces:

```sql
SELECT TRIM(customer_name) AS clean_name
FROM customers;
```

Related functions include `LTRIM` and `RTRIM`.

### LENGTH

```sql
SELECT product_name,
       LENGTH(product_name) AS name_length
FROM products;
```

### CONCAT

```sql
SELECT customer_id,
       CONCAT(customer_name, ' - ', country) AS customer_label
FROM customers;
```

PostgreSQL also supports the `||` concatenation operator. NULL behavior can differ by function or engine, so test it.

### SUBSTRING

```sql
SELECT product_name,
       SUBSTRING(product_name FROM 1 FOR 5) AS first_five_characters
FROM products;
```

Some platforms use `SUBSTR` or comma-based arguments.

### REPLACE

```sql
SELECT email,
       REPLACE(email, '@example.com', '@company.com') AS migrated_email
FROM customers
WHERE email IS NOT NULL;
```

## 5.2 Numeric functions

### ROUND

```sql
SELECT product_name,
       unit_price,
       ROUND(unit_price * 1.18, 2) AS tax_inclusive_price
FROM products;
```

### ABS

Returns the absolute value:

```sql
SELECT ABS(-250) AS absolute_value;
```

### CEIL and FLOOR

```sql
SELECT CEIL(12.2) AS rounded_up,
       FLOOR(12.8) AS rounded_down;
```

## 5.3 Date and time functions

### Current values

```sql
SELECT CURRENT_DATE AS run_date,
       CURRENT_TIMESTAMP AS run_timestamp;
```

In production pipelines, be aware of the session time zone.

### Extract a date component

```sql
SELECT order_id,
       order_date,
       EXTRACT(MONTH FROM order_date) AS order_month
FROM orders;
```

### Truncate to a time period

PostgreSQL-style:

```sql
SELECT order_date,
       DATE_TRUNC('month', order_date) AS month_start
FROM orders;
```

Databricks SQL also provides date-truncation functions, but accepted units and result types should be checked for that engine.

### Date arithmetic

PostgreSQL-style:

```sql
SELECT order_id,
       order_date,
       CURRENT_DATE - order_date AS age_in_days
FROM orders;
```

Engines differ significantly in date-add and date-difference syntax.

## 5.4 COALESCE

Return the first non-NULL value:

```sql
SELECT customer_name,
       COALESCE(city, 'Unknown') AS city,
       COALESCE(credit_limit, 0) AS displayed_credit_limit
FROM customers;
```

Again, replacing unknown with zero is only correct if that is the agreed business rule.

## 5.5 NULLIF

`NULLIF(a, b)` returns NULL if `a = b`; otherwise it returns `a`.

This can protect a division from a zero denominator:

```sql
SELECT 100.0 / NULLIF(0, 0) AS safe_result;
```

The result is NULL rather than a division-by-zero error.

## 5.6 CAST

Convert a value to another data type:

```sql
SELECT CAST(order_id AS VARCHAR) AS order_id_text
FROM orders;
```

PostgreSQL also supports shorthand such as `order_id::VARCHAR`, but `CAST` is more portable.

Conversions can fail if the source value is invalid. Some platforms provide safe or try-cast functions that return NULL instead of failing.

### Do not cast join and filter columns unnecessarily

This pattern may be expensive:

```sql
WHERE CAST(order_date AS VARCHAR) LIKE '2026-07%'
```

Prefer a range that preserves the column type:

```sql
WHERE order_date >= '2026-07-01'
  AND order_date <  '2026-08-01'
```

This is clearer and more likely to allow pruning or efficient access.

## 5.7 Function composition

Functions can be nested:

```sql
SELECT customer_id,
       UPPER(TRIM(customer_name)) AS normalized_name,
       LOWER(TRIM(email)) AS normalized_email
FROM customers;
```

Each function should support a clear business or data-quality rule.

## Day 5 common mistakes

- Assuming all databases use identical date functions.
- Ignoring time zones for timestamps.
- Converting dates to text before filtering.
- Applying functions to large filter/join columns unnecessarily.
- Confusing rounding with formatting.
- Replacing NULL without confirming the business meaning.
- Assuming a cast will always succeed.

## Day 5 exercises

1. Display every customer name in uppercase.
2. Display customer emails in lowercase, excluding NULL emails.
3. Return each product name and its character length.
4. Create a customer label combining name, city, and country.
5. Calculate product price after 18 percent tax, rounded to two decimals.
6. Extract the year and month from each order date.
7. Return orders placed in July 2026 using a date range.
8. Display missing sales channels as `UNKNOWN`.
9. Cast order ID to text.
10. Explain why a date range is usually better than casting a date to text for filtering.

## Day 5 solutions

```sql
-- 1
SELECT customer_id,
       UPPER(customer_name) AS customer_name_upper
FROM customers;

-- 2
SELECT customer_id,
       LOWER(email) AS normalized_email
FROM customers
WHERE email IS NOT NULL;

-- 3
SELECT product_name,
       LENGTH(product_name) AS character_count
FROM products;

-- 4
SELECT customer_id,
       CONCAT(
           customer_name,
           ' - ',
           COALESCE(city, 'Unknown'),
           ', ',
           country
       ) AS customer_label
FROM customers;

-- 5
SELECT product_id,
       product_name,
       ROUND(unit_price * 1.18, 2) AS tax_inclusive_price
FROM products;

-- 6
SELECT order_id,
       EXTRACT(YEAR FROM order_date) AS order_year,
       EXTRACT(MONTH FROM order_date) AS order_month
FROM orders;

-- 7
SELECT *
FROM orders
WHERE order_date >= '2026-07-01'
  AND order_date <  '2026-08-01';

-- 8
SELECT order_id,
       COALESCE(sales_channel, 'UNKNOWN') AS sales_channel
FROM orders;

-- 9
SELECT CAST(order_id AS VARCHAR) AS order_id_text
FROM orders;
```

---

# Day 6 — Revision, Data Engineer application, and assessment

## 6.1 Week 1 syntax map

```sql
SELECT DISTINCT
       column_1,
       column_2 AS alias_name,
       expression AS calculated_column
FROM table_name
WHERE condition_1
  AND (condition_2 OR condition_3)
ORDER BY column_1 DESC,
         column_2 ASC
LIMIT 10;
```

## 6.2 Data Engineer source inspection checklist

When you receive a new source table, write queries to inspect:

1. A small sample with deterministic ordering.
2. Expected columns and data types.
3. Candidate key values.
4. NULLs in important fields.
5. Distinct status/category values.
6. Minimum and maximum business dates.
7. Unexpected future dates.
8. Invalid numeric ranges.
9. Leading/trailing spaces and inconsistent casing.
10. Source update or ingestion timestamp range.

Week 2 will add aggregation so you can calculate counts for these checks.

## 6.3 Mini-project — Inspect the orders source

Write queries that answer the following requirements.

1. Return a five-row deterministic sample ordered by order ID.
2. Return only the columns needed for a daily order report.
3. List every distinct status.
4. List every distinct sales channel, including NULL.
5. Find orders with missing customer IDs.
6. Find orders with missing totals.
7. Find orders with missing sales channels.
8. Find orders whose totals are negative.
9. Find orders updated during July 2026 using a half-open timestamp interval.
10. Return the three highest-value completed orders.
11. Create a display status normalized to uppercase.
12. Create an order reference containing a prefix and the order ID.

## Mini-project solution

```sql
-- 1. Deterministic sample
SELECT *
FROM orders
ORDER BY order_id
LIMIT 5;

-- 2. Daily report columns
SELECT order_id,
       customer_id,
       order_date,
       order_status,
       order_total
FROM orders;

-- 3. Status domain
SELECT DISTINCT order_status
FROM orders
ORDER BY order_status;

-- 4. Channel domain including NULL
SELECT DISTINCT sales_channel
FROM orders
ORDER BY sales_channel;

-- 5. Missing customer key
SELECT *
FROM orders
WHERE customer_id IS NULL;

-- 6. Missing amount
SELECT *
FROM orders
WHERE order_total IS NULL;

-- 7. Missing channel
SELECT *
FROM orders
WHERE sales_channel IS NULL;

-- 8. Invalid negative amount
SELECT *
FROM orders
WHERE order_total < 0;

-- 9. July updates
SELECT *
FROM orders
WHERE updated_at >= '2026-07-01 00:00:00'
  AND updated_at <  '2026-08-01 00:00:00';

-- 10. Highest completed totals
SELECT order_id, customer_id, order_total
FROM orders
WHERE order_status = 'COMPLETED'
  AND order_total IS NOT NULL
ORDER BY order_total DESC, order_id ASC
LIMIT 3;

-- 11. Normalized display status
SELECT order_id,
       UPPER(TRIM(order_status)) AS normalized_status
FROM orders;

-- 12. Order reference
SELECT order_id,
       CONCAT('ORD-', CAST(order_id AS VARCHAR)) AS order_reference
FROM orders;
```

## 6.4 Week 1 interview questions and answers

### 1. What is SQL?

SQL is a declarative language used to define, read, change, and control data in relational and SQL-compatible systems. Declarative means you describe the result; the engine chooses an execution plan.

### 2. What is the difference between a table, row, and column?

A table stores related records. A row is one record at the table grain. A column is one attribute of the record.

### 3. What is table grain?

Grain defines exactly what one row represents. Examples are one customer, one order, or one order item.

### 4. What is the difference between SELECT star and selected columns?

`SELECT *` returns all columns. Explicit selection returns only required columns, making output stable, clear, and often more efficient.

### 5. What is an alias?

An alias is a temporary name used in query output or to reference a table. It does not rename the underlying database object.

### 6. What is the difference between WHERE and ORDER BY?

`WHERE` filters rows. `ORDER BY` sorts the rows remaining in the result.

### 7. What is the difference between AND and OR?

`AND` requires all connected conditions to be true. `OR` requires at least one to be true.

### 8. Why use parentheses in filters?

Parentheses make intended grouping explicit and prevent operator precedence from changing the business rule.

### 9. Is BETWEEN inclusive?

Yes, both lower and upper boundaries are included.

### 10. Why prefer a half-open range for timestamps?

`timestamp >= start AND timestamp < next_boundary` includes the full period without depending on time precision or constructing an end-of-day value.

### 11. What do percent and underscore mean in LIKE?

Percent matches zero or more characters; underscore matches exactly one character.

### 12. What is NULL?

NULL represents missing or unknown data. It is not zero or an empty string.

### 13. Why does column equals NULL fail?

Equality with NULL evaluates to unknown. Use `IS NULL` or `IS NOT NULL`.

### 14. What does DISTINCT do?

It removes duplicate combinations of every expression in the selected result.

### 15. Does SQL guarantee row order?

Only when the query specifies `ORDER BY`.

### 16. Why add a tiebreaker to a top-N query?

A unique tiebreaker makes the chosen rows deterministic when sort values tie.

### 17. What does COALESCE do?

It returns the first non-NULL expression from its argument list.

### 18. What does CAST do?

It converts a value from one data type to another when the conversion is valid.

### 19. Why can a function in a WHERE clause be expensive?

Applying a function to every row can prevent an engine from using an index, partition pruning, or other data-skipping optimization.

### 20. Why should a Data Engineer care about NULL and data types?

Incorrect assumptions about missing values and types cause rejected records, inaccurate calculations, failed joins, and inconsistent pipeline results.

## 6.5 Final practice set

Solve without looking at previous examples.

### Basic

1. Return customer ID and name for all customers.
2. Return active customers only.
3. Find products in the Electronics category.
4. Find orders worth less than 5,000.
5. Find customers from Pune or Mumbai.
6. Find customers not from Pune.
7. Find products with zero stock.
8. Find products with unknown stock.
9. Find orders placed on or after 10 July 2026.
10. Find completed web orders.

### Pattern and range

11. Find customers whose names begin with `M`.
12. Find product names containing `SQL`.
13. Find products priced from 500 to 5,000 inclusive.
14. Find orders placed from 10 July through 20 July inclusive.
15. Find orders whose statuses are completed or shipped.

### Sorting and limiting

16. Return the newest three customers by signup date.
17. Return the cheapest four products.
18. Sort orders by date descending and order ID descending.
19. Return distinct customer cities.
20. Return distinct status-channel combinations.

### Functions and data quality

21. Normalize customer emails to lowercase.
22. Replace missing city with `Unknown` for display.
23. Calculate product price after a 15 percent discount.
24. Return order IDs as references such as `ORD-1001`.
25. Find all missing important fields in orders: customer, total, or channel.
26. Find inactive Pune customers.
27. Find active customers with credit limits above 50,000.
28. Return the longest product names first.
29. Return the three latest updates from the customer table.
30. Write a query that safely returns all July order updates.

## 6.6 Final practice answer key

```sql
-- 1
SELECT customer_id, customer_name FROM customers;

-- 2
SELECT * FROM customers WHERE is_active = TRUE;

-- 3
SELECT * FROM products WHERE category = 'Electronics';

-- 4
SELECT * FROM orders WHERE order_total < 5000;

-- 5
SELECT * FROM customers WHERE city IN ('Pune', 'Mumbai');

-- 6
SELECT *
FROM customers
WHERE city <> 'Pune'
   OR city IS NULL;

-- 7
SELECT * FROM products WHERE stock_quantity = 0;

-- 8
SELECT * FROM products WHERE stock_quantity IS NULL;

-- 9
SELECT * FROM orders WHERE order_date >= '2026-07-10';

-- 10
SELECT *
FROM orders
WHERE order_status = 'COMPLETED'
  AND sales_channel = 'WEB';

-- 11
SELECT * FROM customers WHERE customer_name LIKE 'M%';

-- 12
SELECT * FROM products WHERE product_name LIKE '%SQL%';

-- 13
SELECT * FROM products WHERE unit_price BETWEEN 500 AND 5000;

-- 14
SELECT *
FROM orders
WHERE order_date BETWEEN '2026-07-10' AND '2026-07-20';

-- 15
SELECT *
FROM orders
WHERE order_status IN ('COMPLETED', 'SHIPPED');

-- 16
SELECT customer_id, customer_name, signup_date
FROM customers
ORDER BY signup_date DESC, customer_id DESC
LIMIT 3;

-- 17
SELECT product_id, product_name, unit_price
FROM products
ORDER BY unit_price ASC, product_id ASC
LIMIT 4;

-- 18
SELECT *
FROM orders
ORDER BY order_date DESC, order_id DESC;

-- 19
SELECT DISTINCT city
FROM customers
ORDER BY city;

-- 20
SELECT DISTINCT order_status, sales_channel
FROM orders
ORDER BY order_status, sales_channel;

-- 21
SELECT customer_id, LOWER(TRIM(email)) AS normalized_email
FROM customers;

-- 22
SELECT customer_id, COALESCE(city, 'Unknown') AS city
FROM customers;

-- 23
SELECT product_id,
       product_name,
       ROUND(unit_price * 0.85, 2) AS discounted_price
FROM products;

-- 24
SELECT order_id,
       CONCAT('ORD-', CAST(order_id AS VARCHAR)) AS order_reference
FROM orders;

-- 25
SELECT *
FROM orders
WHERE customer_id IS NULL
   OR order_total IS NULL
   OR sales_channel IS NULL;

-- 26
SELECT *
FROM customers
WHERE is_active = FALSE
  AND city = 'Pune';

-- 27
SELECT *
FROM customers
WHERE is_active = TRUE
  AND credit_limit > 50000;

-- 28
SELECT product_id, product_name
FROM products
ORDER BY LENGTH(product_name) DESC, product_id ASC;

-- 29
SELECT customer_id, customer_name, updated_at
FROM customers
ORDER BY updated_at DESC, customer_id DESC
LIMIT 3;

-- 30
SELECT *
FROM orders
WHERE updated_at >= '2026-07-01 00:00:00'
  AND updated_at <  '2026-08-01 00:00:00';
```

---

# Week 1 one-page cheat sheet

```sql
-- Select columns
SELECT column_1, column_2
FROM table_name;

-- Alias and calculation
SELECT column_1 AS new_name,
       amount * quantity AS calculated_value
FROM table_name;

-- Filter
SELECT *
FROM table_name
WHERE amount >= 1000
  AND status IN ('COMPLETED', 'SHIPPED');

-- Range
WHERE business_date >= '2026-07-01'
  AND business_date <  '2026-08-01'

-- Pattern
WHERE customer_name LIKE 'A%'

-- NULL
WHERE important_column IS NULL

-- Sort and limit
ORDER BY metric DESC, unique_id ASC
LIMIT 10;

-- Unique combinations
SELECT DISTINCT column_1, column_2
FROM table_name;

-- Basic cleaning
SELECT UPPER(TRIM(status)) AS normalized_status,
       LOWER(TRIM(email)) AS normalized_email,
       COALESCE(city, 'Unknown') AS city
FROM table_name;

-- Type conversion
SELECT CAST(identifier AS VARCHAR)
FROM table_name;
```

## Week 1 completion test

You have completed Week 1 when you can do all of the following without copying:

- Select only required columns.
- Add aliases and calculated expressions.
- Write filters containing both `AND` and `OR` safely.
- Explain and use NULL correctly.
- Write an inclusive range and a half-open timestamp range.
- Use `LIKE`, `IN`, and `BETWEEN`.
- Return a deterministic top-N result.
- Explain why `DISTINCT` is not a universal duplicate fix.
- Normalize text with string functions.
- Describe the grain of `customers`, `products`, and `orders`.

## Next week preview

Week 2 introduces:

- `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX`
- `GROUP BY`
- `WHERE` versus `HAVING`
- `CASE WHEN`
- deeper NULL handling
- daily metrics and data-quality reports
