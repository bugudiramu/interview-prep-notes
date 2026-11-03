# Level 13 — Advanced PostgreSQL & Database Engineering

**Status:** Completed
**Focus:** Query optimization, advanced indexing, MVCC, locking, and scaling architecture

*Personal reference notes - Ramu (Fully Aligned with Roadmap)*

# Part 1 - Optimization Workflow & The Query Planner

The core loop for optimizing any slow query is the same every time: measure first, change one thing, measure again.

**Golden rule:** Never guess. Always confirm a fix worked by re-running `EXPLAIN ANALYZE` before and after.

### 1. EXPLAIN and EXPLAIN ANALYZE

Before touching anything, look at the actual execution plan. The **Query Planner** uses table statistics to calculate the most efficient way to fetch data.

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'ramu@example.com';
```

* **EXPLAIN:** Shows the planner's *estimated* execution plan and cost (arbitrary units, not time).
* **EXPLAIN ANALYZE:** Actually *executes* the query and returns the real timing (in milliseconds) alongside the estimates.
* **Sequential Scan (Seq Scan):** Reads the entire table from disk. Bad for large tables, but actually *faster* than an index scan for very tiny tables.
* **Index Scan:** Traverses the index to find row pointers, then fetches the actual row from the heap (disk).
* **Index Only Scan:** The index contains all requested columns; no heap fetch is needed (extremely fast).

### 2. Query Planner: How Postgres Executes Joins

When you write a `JOIN`, the Query Planner chooses one of three algorithms under the hood based on table sizes and indexes:
1. **Nested Loop Join:** Loops through one table row-by-row and looks up matching rows in the second table. Great if the second table is indexed on the join key.
2. **Hash Join:** Hashes the smaller table into memory, then scans the larger table against the hash table. Used for large, unsorted datasets.
3. **Merge Join:** Sorts both tables by the join key first, then zips them together. Highly efficient if both tables are already sorted via indexes.

### 3. WHERE Clause & Sargability

A predicate is "sargable" (Search ARGument ABLE) when the planner can use an index directly. Wrapping a column in a function usually breaks that.

**Functions on indexed columns**
This will NOT use a plain index on email:
```sql
SELECT * FROM users WHERE UPPER(email) = 'RAMU@EXAMPLE.COM';
```
Fix it with an expression index: `CREATE INDEX idx_users_email_upper ON users (UPPER(email));`

**Non-sargable predicates**
* **NOT IN (subquery):** If the subquery can return NULL, it forces a full scan. Prefer **NOT EXISTS**.
* **LIKE '%value':** A leading wildcard cannot use a standard B-Tree index.

### 4. Advanced Indexing Strategy

* **B-Tree Indexes:** The default index type in PostgreSQL. Structured as a balanced tree. Perfect for equality (`=`) and range queries (`<`, `>`, `BETWEEN`). Operates in $O(\log N)$ time.
* **Composite (Multi-column) Indexes:** Order matters! The most selective column goes first, following the **leftmost-prefix rule** (an index on `A, B, C` helps queries on `A`, `A, B`, or `A, B, C`).
* **Covering Indexes (`INCLUDE`):** Stores extra payload columns in the index leaves to guarantee an **Index Only Scan**.
  ```sql
  CREATE INDEX idx_orders_cust ON orders (customer_id) INCLUDE (status, amount);
  ```
* **Partial Indexes:** Saves massive amounts of memory by indexing only a subset of rows.
  ```sql
  CREATE INDEX idx_active_users ON users (email) WHERE deleted_at IS NULL;
  ```

### 5. Important Practical Question: Why is this query slow even though I have an index?

*(Crucial for Senior Interviews)*
If you add an index but `EXPLAIN ANALYZE` shows Postgres is still doing a Sequential Scan, check these 5 things:

1. **Low Selectivity:** Your query returns 60% of the table (e.g., `status = 'ACTIVE'`). The planner knows that bouncing between the index and the heap millions of times is slower than just scanning the whole table linearly.
2. **Data Type Mismatch:** Your indexed column is a `VARCHAR`, but you queried `WHERE id = 123` (integer). Postgres has to cast every row to an integer on the fly, breaking the index.
3. **Stale Statistics:** The planner thinks the table only has 10 rows because `ANALYZE` hasn't run recently. Run `ANALYZE table_name;` to update stats.
4. **Function Wrapping (Not Sargable):** You wrote `WHERE DATE(created_at) = '2026-01-01'` instead of `WHERE created_at >= '2026-01-01' AND created_at < '2026-01-02'`.
5. **Leading Wildcards:** You used `LIKE '%search'`.

# Part 2 - Database Engineering & Internals

### 1. ACID Properties

* **Atomicity:** All-or-nothing (Transactions).
* **Consistency:** Enforces database constraints and triggers.
* **Isolation:** Concurrent transactions don't interfere (Isolation Levels).
* **Durability:** Committed data is safe on disk via **WAL (Write-Ahead Logging)**.

### 2. MVCC (Multi-Version Concurrency Control)

PostgreSQL handles concurrency without forcing reads to wait for writes.
* When you `UPDATE` a row, Postgres doesn't overwrite it. It writes a **new version** of the row and marks the old one as invisible to new transactions.
* **VACUUM:** A background process that cleans up dead row versions to prevent table bloat.

### 3. Isolation Levels

PostgreSQL defaults to **Read Committed**.

| Level | Prevents Dirty Reads | Prevents Non-Repeatable Reads | Prevents Phantom Reads | Note | 
| ----- | ----- | ----- | ----- | ----- | 
| **Read Uncommitted** | Yes (in PG) | No | No | Behaves exactly like Read Committed in Postgres. | 
| **Read Committed** | Yes | No | No | **Default**. Queries see data committed before the *query* began. | 
| **Repeatable Read** | Yes | Yes | No | Queries see data committed before the *transaction* began. | 
| **Serializable** | Yes | Yes | Yes | Strictest. Emulates serial execution. Will throw serialization errors requiring app retries. | 

### 4. Locking Strategies & Deadlocks

**Pessimistic Locking (Database Level: Row Locks)**
Locks the actual row in the database.
* `SELECT ... FOR UPDATE`: Locks row for writing.
* `SELECT ... FOR SHARE`: Locks row for reading, preventing updates.
* **Deadlocks:** Tx1 locks Row A and waits for Row B; Tx2 locks Row B and waits for Row A. Postgres cancels one. *Fix: Always lock rows in a consistent order.*

**Optimistic Locking (Application Level)**
Does not lock rows. Uses a `version` column (e.g., Hibernate `@Version`).
* Executes: `UPDATE table SET val = X, version = 2 WHERE id = 1 AND version = 1`.
* If another Tx updated it first, 0 rows are affected. Throws an `OptimisticLockException`. Great for high-read, low-write contention.

# Part 3 - Scaling & Production Architecture

### 1. Connection Pooling (PgBouncer)

Opening a direct Postgres connection forks a heavy OS process. Microservices easily exhaust the `max_connections` limit.
* **PgBouncer** sits in front of Postgres, multiplexing thousands of lightweight application connections down to a small pool of physical database connections.

### 2. Read Replicas vs. Partitioning vs. Sharding

| Strategy | Problem it Solves | How it Works | 
| ----- | ----- | ----- | 
| **Partitioning** | Large table performance | Splits one logical table into physical pieces (by range/list). Queries use **partition pruning** to skip irrelevant partitions. | 
| **Read Replicas** | High read load | Duplicates the database asynchronously. Master handles writes; Replicas handle `SELECT` queries (Eventual Consistency). | 
| **Sharding** | Insurmountable write/storage load | Horizontal scaling across multiple physical servers based on a key (e.g., `tenant_id`). Highly complex app logic required. | 

### 3. Database Migrations

Schema changes must be versioned, repeatable, and automated (via **Flyway** or **Liquibase**).
* **Zero-Downtime Migrations:** Never run `ALTER TABLE ADD COLUMN default 'val'` on huge tables (locks the table). Add the column cleanly, deploy code to write to both, backfill data concurrently, then enforce constraints.