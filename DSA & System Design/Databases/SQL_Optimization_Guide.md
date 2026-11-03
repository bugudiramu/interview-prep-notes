**SQL QUERY OPTIMIZATION**

A Working Guide + PostgreSQL Cheatsheet

_Personal reference notes - Ramu_

Focus: PostgreSQL

**Part 1 - Optimization Workflow**

The core loop for optimizing any slow query is the same every time: measure first, change one thing, measure again. Everything below fits into this loop.

**Golden rule:** Never guess. Always confirm a fix worked by re-running EXPLAIN ANALYZE before and after.

# **1\. EXPLAIN ANALYZE - Read the Plan First**

Before touching anything, look at the actual execution plan. This tells you what the query planner is really doing - not what you assume it's doing.

EXPLAIN ANALYZE

SELECT \* FROM users WHERE email = '<ramu@example.com>';

What to look for in the output:

- **Seq Scan** vs **Index Scan** vs **Index Only Scan** - a sequential scan on a large table is usually the first red flag.
- **Actual time** and **rows** vs the planner's **estimated rows** - a big gap between estimate and actual usually means stale statistics (run ANALYZE table_name;).
- **Cost** (arbitrary planner units, not milliseconds) - useful for comparing two plans of the same query, not for absolute timing.
- Nested loop / hash join / merge join choice - a nested loop over a huge unfiltered set is often a sign an index is missing.
- Use EXPLAIN (ANALYZE, BUFFERS) to also see how much was read from cache vs disk.

**Correction:** Cost units in EXPLAIN are not milliseconds - they're an internal planner estimate. Only EXPLAIN ANALYZE gives real elapsed time, because it actually executes the query.

# **2\. WHERE Clause & Sargability**

A predicate is "sargable" (Search ARGument ABLE) when the planner can use an index to satisfy it directly. Wrapping a column in a function usually breaks that.

## **Functions on indexed columns**

This will NOT use a plain index on email, even if one exists:

SELECT \* FROM users WHERE UPPER(email) = '<RAMU@EXAMPLE.COM>';

Fix it either by matching the function with an expression index, or by avoiding the function entirely:

\-- Option A: expression index that matches the function used in the query

CREATE INDEX idx_users_email_upper ON users (UPPER(email));

\-- Option B: normalize data at write time / use case-insensitive comparison

SELECT \* FROM users WHERE email = LOWER('<RAMU@EXAMPLE.COM>');

\-- or use citext extension for case-insensitive text columns

## **Non-sargable predicates**

These operators can defeat index usage depending on how they're used - check the plan every time rather than assuming:

- **NOT IN (subquery)** - the riskiest of the three. If the subquery can return NULL, NOT IN can silently return zero rows, and it typically forces a full scan. Prefer **NOT EXISTS** or a **LEFT JOIN ... WHERE right.id IS NULL** instead.
- **NOT EXISTS** - despite the "NOT", this one usually CAN use an index on the correlated column, and is generally the safer, faster choice over NOT IN.
- **LIKE '%value'** (leading wildcard) - cannot use a standard B-tree index, because the index is sorted left-to-right. LIKE 'value%' (trailing wildcard) CAN use a B-tree index. For arbitrary substring/full-text search, use a **trigram (pg_trgm)** or **GIN/full-text** index instead.

\-- Bad: leading wildcard, can't use a plain B-tree index

SELECT \* FROM users WHERE email LIKE '%gmail.com';

\-- OK: trailing wildcard, B-tree index usable

SELECT \* FROM users WHERE email LIKE 'ramu%';

\-- Better for arbitrary substring search: trigram index

CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX idx_users_email_trgm ON users USING GIN (email gin_trgm_ops);

**Correction from original notes:** NOT IN and LIKE are the ones that reliably hurt performance. NOT EXISTS is usually fine and is often the recommended replacement for NOT IN - it's not in the same "avoid" bucket.

# **3\. Select Only the Columns You Need**

Avoid \`SELECT \*\` in application code and reports.

- Fewer columns means less I/O and less data sent over the network.
- If every column in the query (in SELECT, WHERE, JOIN, ORDER BY) is covered by an index, Postgres can do an **Index Only Scan** and skip the heap entirely - fetching from the index alone.

\-- Less efficient

SELECT \* FROM users WHERE id = 42;

\-- Better: only what's needed, index-only scan possible if id is indexed

SELECT id, name, email FROM users WHERE id = 42;

# **4\. Indexing Strategy**

## **Composite (multi-column) indexes**

Order matters. Build the column order in the index to match how the query filters - the most selective / most commonly filtered-alone column usually goes first, following the \*\*leftmost-prefix rule\*\*: an index on (a, b, c) can serve queries filtering on (a), (a, b), or (a, b, c), but not on (b) or (c) alone.

\-- Query pattern:

SELECT \* FROM orders WHERE customer_id = 5 AND status = 'pending';

\-- Matching composite index (customer_id first, since it's the primary filter):

CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);

## **Don't over-index**

Every index speeds up reads but slows down every INSERT, UPDATE, and DELETE on that table, and takes disk space. Only add an index when a real, recurring query pattern needs it - verify with EXPLAIN, don't index speculatively.

- Check for unused indexes periodically via pg_stat_user_indexes and drop what nothing uses.
- Watch for duplicate/overlapping indexes covering the same leftmost columns.

# **5\. Re-check the Execution Plan**

After any change - new index, rewritten predicate, added partition - re-run EXPLAIN ANALYZE and compare actual timing, not just intuition. If a change doesn't measurably help, don't keep it just because it "should" help.

# **6\. Parameterized Queries / Prepared Statements**

Sending literal values inline means Postgres has to parse and plan the query fresh every time, even though the query shape is identical:

SELECT \* FROM users WHERE id = 1;

SELECT \* FROM users WHERE id = 2;

SELECT \* FROM users WHERE id = 3;

\-- Each one is treated as a distinct query by the planner

Using a bind parameter lets the same parsed/planned query be reused:

\-- Prepared statement

PREPARE get_user (int) AS

SELECT \* FROM users WHERE id = \$1;

EXECUTE get_user(1);

EXECUTE get_user(2);

**Nuance to remember:** Postgres actually plans the first 5 executions of a prepared statement individually (a "custom plan"), then may switch to a single reused "generic plan" if it decides that's cheap enough. For most simple, index-friendly queries this is invisible - but for queries where the best plan depends heavily on the parameter value, a generic plan can occasionally be worse. In practice this is rarely a problem, but it's worth knowing it's not a flat "always identical plan" guarantee.

# **7\. Window Functions**

Window functions compute a value across a set of related rows (a "window") without collapsing rows the way GROUP BY does - useful for running totals, rankings, and comparisons to other rows in the same query.

\-- Running total per customer, ordered by date

SELECT

customer_id,

order_date,

amount,

SUM(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_total

FROM orders;

\-- Rank orders per customer by amount

SELECT

customer_id, amount,

RANK() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rank

FROM orders;

- Common functions: ROW_NUMBER(), RANK(), DENSE_RANK(), LAG(), LEAD(), SUM()/AVG()/COUNT() OVER (...).
- Often replaces a self-join or correlated subquery - usually faster because the data is scanned once.

# **8\. Avoid Unnecessary Sorts / Dedup Work**

DISTINCT and UNION (without ALL) both require Postgres to sort or hash the result set to remove duplicates - real extra work.

- If a query needs DISTINCT to look correct, first ask why duplicates exist in the source data (a bad JOIN fan-out is the usual cause). Fixing the join is often better than masking it with DISTINCT on every query that touches it.
- **UNION** dedupes the combined result (sort/hash under the hood); **UNION ALL** just concatenates. Use UNION ALL whenever you know the two sides can't overlap or duplicates are fine.

\-- Does extra work to remove duplicate rows

SELECT city FROM customers

UNION

SELECT city FROM suppliers;

\-- Cheaper if duplicates are acceptable / known not to exist

SELECT city FROM customers

UNION ALL

SELECT city FROM suppliers;

- An ORDER BY that isn't backed by an index also forces a sort at query time - an index matching the ORDER BY columns can let Postgres read rows already in order.

# **9\. Partitioning & Materialized Views**

## **Table partitioning**

Splits one logical table into smaller physical pieces (by range, list, or hash on a key - e.g. date or region). If the query filters on the partition key, Postgres can use \*\*partition pruning\*\* to skip entire partitions instead of scanning the whole table.

CREATE TABLE events (

id bigint,

event_time timestamp NOT NULL,

payload jsonb

) PARTITION BY RANGE (event_time);

CREATE TABLE events_2026_01 PARTITION OF events

FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

\-- This query only touches the January partition, not the whole table:

SELECT \* FROM events WHERE event_time >= '2026-01-10' AND event_time < '2026-01-15';

## **Materialized views**

A materialized view runs a query once and physically stores the result, unlike a regular view which re-runs the underlying query on every access. It's a genuine mindset shift: the data is a snapshot and goes stale until explicitly refreshed.

CREATE MATERIALIZED VIEW monthly_revenue AS

SELECT date_trunc('month', order_date) AS month, SUM(amount) AS revenue

FROM orders

GROUP BY 1;

\-- Refresh when underlying data changes (blocks reads by default)

REFRESH MATERIALIZED VIEW monthly_revenue;

\-- Non-blocking refresh (requires a UNIQUE index on the view)

REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;

**Trade-off to remember:** Materialized views trade freshness for speed. Good for dashboards/reports that don't need to-the-second accuracy; not a substitute for a normal index on a frequently-updated table.

# **Quick Checklist - Before You Ship a Slow Query Fix**

1. Run EXPLAIN ANALYZE and identify the actual bottleneck (scan type, join type, estimate vs actual rows).
2. Check WHERE clause predicates for sargability - no bare functions on indexed columns, avoid NOT IN with nullable subqueries, watch leading-wildcard LIKE.
3. Select only needed columns - check if an index-only scan becomes possible.
4. Confirm indexes exist and match the leftmost-prefix of the query's filter columns.
5. Re-run EXPLAIN ANALYZE to confirm the change actually helped.
6. For repeated identical query shapes, use parameterized/prepared statements.
7. Remove unnecessary DISTINCT/UNION where the root cause (duplicate-producing join) can be fixed instead.
8. For very large tables, evaluate partitioning; for expensive repeated aggregates, evaluate a materialized view.

**Part 2 - PostgreSQL Cheatsheet**

Beginner-to-intermediate reference: core syntax and everyday commands, organized by topic.

# **Basic Querying**

SELECT column1, column2 FROM table_name;

SELECT \* FROM table_name WHERE condition;

SELECT DISTINCT column1 FROM table_name;

SELECT column1 AS alias_name FROM table_name;

SELECT \* FROM table_name ORDER BY column1 DESC;

SELECT \* FROM table_name LIMIT 10 OFFSET 20;

# **Filtering**

| **Clause**                       | **Example**                           | **Notes**                |
| -------------------------------- | ------------------------------------- | ------------------------ |
| \=, &lt;>, <, &gt;, &lt;=, &gt;= | WHERE age >= 18                       | Standard comparisons     |
| AND / OR / NOT                   | WHERE age >= 18 AND status = 'active' | Combine conditions       |
| BETWEEN                          | WHERE price BETWEEN 10 AND 50         | Inclusive range          |
| IN                               | WHERE status IN ('active','pending')  | Set membership           |
| LIKE / ILIKE                     | WHERE name ILIKE 'ram%'               | ILIKE = case-insensitive |
| IS NULL / IS NOT NULL            | WHERE deleted_at IS NULL              | Never use = NULL         |

# **Joins**

\-- Only matching rows on both sides

SELECT o.id, c.name FROM orders o

INNER JOIN customers c ON o.customer_id = c.id;

\-- All rows from the left, matched or NULL from the right

SELECT c.name, o.id FROM customers c

LEFT JOIN orders o ON o.customer_id = c.id;

\-- All rows from the right, matched or NULL from the left

SELECT c.name, o.id FROM customers c

RIGHT JOIN orders o ON o.customer_id = c.id;

\-- Every combination of rows (rarely intentional)

SELECT \* FROM a CROSS JOIN b;

# **Aggregation**

SELECT status, COUNT(\*) FROM orders GROUP BY status;

SELECT customer_id, SUM(amount) FROM orders

GROUP BY customer_id

HAVING SUM(amount) > 1000;

\-- Common aggregates: COUNT, SUM, AVG, MIN, MAX

**Remember:** WHERE filters rows before grouping; HAVING filters groups after aggregation. You can't use an aggregate like SUM() in WHERE.

# **Subqueries & CTEs**

\-- Subquery

SELECT \* FROM orders

WHERE customer_id IN (SELECT id FROM customers WHERE country = 'IN');

\-- Common Table Expression (CTE) - readable, reusable within the query

WITH high_value AS (

SELECT customer_id, SUM(amount) AS total

FROM orders

GROUP BY customer_id

HAVING SUM(amount) > 1000

)

SELECT c.name, h.total

FROM high_value h

JOIN customers c ON c.id = h.customer_id;

# **Data Modification (DML)**

INSERT INTO users (name, email) VALUES ('Ramu', '<ramu@example.com>');

UPDATE users SET status = 'inactive' WHERE last_login < now() - interval '90 days';

DELETE FROM sessions WHERE expires_at < now();

\-- Insert or update on conflict (upsert)

INSERT INTO users (id, email) VALUES (1, '<ramu@example.com>')

ON CONFLICT (id) DO UPDATE SET email = EXCLUDED.email;

# **Schema (DDL)**

CREATE TABLE users (

id SERIAL PRIMARY KEY,

name TEXT NOT NULL,

email TEXT UNIQUE NOT NULL,

created_at TIMESTAMP DEFAULT now()

);

ALTER TABLE users ADD COLUMN age INT;

ALTER TABLE users DROP COLUMN age;

ALTER TABLE users RENAME COLUMN name TO full_name;

DROP TABLE IF EXISTS temp_table;

# **Indexes**

CREATE INDEX idx_users_email ON users (email);

CREATE UNIQUE INDEX idx_users_email_unique ON users (email);

CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);

DROP INDEX idx_users_email;

\-- See all indexes on a table

\\d users -- psql shortcut

SELECT \* FROM pg_indexes WHERE tablename = 'users';

# **Transactions**

BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;

UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;

\-- ROLLBACK; instead of COMMIT to undo everything in the transaction

# **Useful psql Commands**

| **Command**        | **Meaning**                                    |
| ------------------ | ---------------------------------------------- |
| \`\\dt\`           | List tables                                    |
| \`\\d table_name\` | Describe table (columns, indexes, constraints) |
| \`\\di\`           | List indexes                                   |
| \`\\l\`            | List databases                                 |
| \`\\c dbname\`     | Connect to a database                          |
| \`\\x\`            | Toggle expanded (vertical) row output          |
| \`\\timing\`       | Toggle showing query execution time            |
| \`\\q\`            | Quit psql                                      |

# **Common Postgres Data Types**

| **Type**                    | **Use for**                                                                 |
| --------------------------- | --------------------------------------------------------------------------- |
| \`INT / BIGINT\`            | Whole numbers / large whole numbers                                         |
| \`NUMERIC(p,s)\`            | Exact decimals - money, precise math                                        |
| \`TEXT / VARCHAR(n)\`       | Strings - TEXT has no length limit                                          |
| \`BOOLEAN\`                 | true / false                                                                |
| \`TIMESTAMP / TIMESTAMPTZ\` | Date + time - prefer TIMESTAMPTZ for anything user-facing across time zones |
| \`DATE\`                    | Calendar date only                                                          |
| \`JSONB\`                   | Semi-structured data, indexable and queryable                               |
| \`UUID\`                    | Globally unique identifiers                                                 |
| \`ARRAY\`                   | e.g. INT\[\], TEXT\[\] - native array columns                               |

# **Handy Functions**

| **Function**             | **Example**                                             |
| ------------------------ | ------------------------------------------------------- |
| \`COALESCE\`             | COALESCE(nickname, name) - first non-null value         |
| \`NULLIF\`               | NULLIF(count, 0) - returns NULL if equal                |
| \`CAST / ::\`            | price::numeric or CAST(price AS numeric)                |
| \`now() / current_date\` | Current timestamp / date                                |
| \`date_trunc\`           | date_trunc('month', order_date)                         |
| \`EXTRACT\`              | EXTRACT(year FROM order_date)                           |
| \`STRING_AGG\`           | STRING_AGG(name, ', ') - concatenate rows into a string |
| \`ARRAY_AGG\`            | ARRAY_AGG(id) - collect rows into an array              |