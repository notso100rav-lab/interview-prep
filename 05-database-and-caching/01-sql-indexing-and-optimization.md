# Chapter 1: SQL Indexing & Optimization

> **Estimated study time:** 2 days | **Priority:** 🔴 High

---

## Key Concepts

- B-tree vs Hash indexes
- Clustered vs non-clustered indexes
- Composite index and the leftmost prefix rule
- Covering index / index-only scan
- Partial indexes
- Reading `EXPLAIN ANALYZE` output
- Index selectivity
- When NOT to index
- Query rewrites
- Partitioning basics

---

## Questions & Answers

---

### Q1 — 🟢 What is a B-tree index and why is it the default in PostgreSQL?

<details><summary>Click to reveal answer</summary>

A **B-tree (Balanced Tree)** index stores keys in sorted order in a self-balancing tree structure. Every leaf is at the same depth, so lookups are O(log n).

**Why it's the default:**
- Supports `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `LIKE 'prefix%'`
- Works with `ORDER BY` and `GROUP BY`
- Handles NULLs (PostgreSQL B-tree stores NULLs; they're always considered not equal)

```sql
-- Default B-tree index
CREATE INDEX idx_users_email ON users(email);

-- PostgreSQL uses it for equality and range
SELECT * FROM users WHERE email = 'alice@example.com';
SELECT * FROM users WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
```

</details>

---

### Q2 — 🟢 What is a Hash index? When would you use one over a B-tree?

<details><summary>Click to reveal answer</summary>

A **Hash index** maps each key to a bucket via a hash function. Lookups are O(1) for equality but do NOT support range queries.

```sql
CREATE INDEX idx_sessions_token ON sessions USING HASH (token);
-- Fast for:
SELECT * FROM sessions WHERE token = 'abc123xyz';
-- Cannot use for:
SELECT * FROM sessions WHERE token > 'abc'; -- hash index not used
```

**Use Hash when:**
- Column is used only in `=` comparisons
- Values are very long strings (hash is smaller than the full key in B-tree)

**In practice:** PostgreSQL's B-tree is usually fast enough for equality too, so Hash indexes are rarely chosen explicitly.

</details>

---

### Q3 — 🟢 What is the difference between a clustered and non-clustered index in PostgreSQL?

<details><summary>Click to reveal answer</summary>

| | Clustered | Non-clustered (Heap) |
|---|---|---|
| Data storage | Table rows stored in index order | Heap stores rows; index has pointers (TID) |
| PostgreSQL default | ❌ PostgreSQL heaps are unordered | ✅ All PostgreSQL indexes are non-clustered by default |
| `CLUSTER` command | Physically reorders table once | Must re-run `CLUSTER` to maintain order |
| InnoDB (MySQL) | Primary key IS the clustered index | Secondary indexes point to PK value |

```sql
-- PostgreSQL: reorder table on disk by index (one-time, not maintained)
CLUSTER users USING idx_users_created_at;

-- After inserts, order is lost again — must re-CLUSTER periodically
```

**Key point:** PostgreSQL does NOT maintain a clustered index automatically. Use `BRIN` indexes for naturally ordered data (timestamps on append-only tables).

</details>

---

### Q4 — 🟡 Explain the leftmost prefix rule for composite indexes.

<details><summary>Click to reveal answer</summary>

A composite index on `(a, b, c)` can be used by queries that filter on:
- `a` alone ✅
- `a, b` ✅
- `a, b, c` ✅
- `b` alone ❌ (skips leftmost column)
- `b, c` alone ❌
- `a, c` — partially ✅ (uses `a`, but `c` filter requires heap re-check)

```sql
CREATE INDEX idx_orders_composite ON orders(user_id, status, created_at);

-- Uses index fully:
SELECT * FROM orders WHERE user_id = 1 AND status = 'PAID' AND created_at > '2024-01-01';

-- Uses index on user_id only (status skipped for range on created_at):
SELECT * FROM orders WHERE user_id = 1 AND created_at > '2024-01-01';

-- Does NOT use index:
SELECT * FROM orders WHERE status = 'PAID';
```

**Design tip:** Put the highest-selectivity equality columns first, then range columns last.

</details>

---

### Q5 — 🟡 What is a covering index and how does it enable an index-only scan?

<details><summary>Click to reveal answer</summary>

A **covering index** includes all columns needed by a query, so PostgreSQL can answer the query entirely from the index without touching the heap (table rows).

```sql
-- Query needs: user_id (filter) + email, name (SELECT)
CREATE INDEX idx_users_covering ON users(user_id) INCLUDE (email, name);

-- Now this query is an index-only scan:
SELECT email, name FROM users WHERE user_id = 42;
```

```
EXPLAIN ANALYZE SELECT email, name FROM users WHERE user_id = 42;
-- Output:
Index Only Scan using idx_users_covering on users
  Index Cond: (user_id = 42)
  Heap Fetches: 0    <-- no heap access!
```

**Note:** PostgreSQL still checks the visibility map to confirm rows are visible (MVCC). `Heap Fetches: 0` means the visibility map confirmed all pages are all-visible.

</details>

---

### Q6 — 🟡 What is a partial index and when is it useful?

<details><summary>Click to reveal answer</summary>

A **partial index** includes only rows that satisfy a `WHERE` condition. It's smaller, faster to scan, and reduces write overhead.

```sql
-- Only index active users (99% of queries filter on is_active = true)
CREATE INDEX idx_users_active_email ON users(email) WHERE is_active = true;

-- This query uses the partial index:
SELECT * FROM users WHERE email = 'alice@example.com' AND is_active = true;

-- Index on unprocessed orders only
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'PENDING';
```

**Benefits:**
- Much smaller index (e.g., 1% of rows if `status='PENDING'` is rare)
- Faster writes (inactive rows don't update the index)
- Lower memory footprint for `pg_buffercache`

</details>

---

### Q7 — 🟡 How do you read `EXPLAIN ANALYZE` output? What do Seq Scan, Index Scan, and Index Only Scan mean?

<details><summary>Click to reveal answer</summary>

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 5 AND status = 'PAID';
```

**Sample output:**
```
Index Scan using idx_orders_user_status on orders  (cost=0.42..8.45 rows=3 width=120) (actual time=0.025..0.031 rows=3 loops=1)
  Index Cond: ((user_id = 5) AND (status = 'PAID'))
Buffers: shared hit=4
Planning Time: 0.2 ms
Execution Time: 0.1 ms
```

| Term | Meaning |
|---|---|
| `cost=0.42..8.45` | Estimated startup cost .. total cost (planner units) |
| `rows=3` | Estimated row count |
| `actual time=0.025..0.031` | Real start ms .. end ms |
| `loops=1` | How many times this node executed |
| `Seq Scan` | Full table scan — no index used |
| `Index Scan` | Uses index, then fetches heap rows for columns |
| `Index Only Scan` | Uses index only — no heap fetch (covering index) |
| `Bitmap Heap Scan` | Index finds row locations, then batch-fetches heap pages |

**Red flags:** `Seq Scan` on large table, `rows=1000` vs `actual rows=50000` (bad statistics → run `ANALYZE`).

</details>

---

### Q8 — 🟢 What is index selectivity and why does it matter?

<details><summary>Click to reveal answer</summary>

**Selectivity** = fraction of rows returned by a predicate.
- High selectivity (close to 0) → few rows returned → index is useful
- Low selectivity (close to 1) → many rows returned → seq scan may be faster

```sql
-- HIGH selectivity: email is unique → 1 row out of 1M → great for index
SELECT * FROM users WHERE email = 'alice@example.com';

-- LOW selectivity: gender has 2 values → 500K out of 1M → index often skipped
SELECT * FROM users WHERE gender = 'F';
```

PostgreSQL uses statistics (`pg_stats`, `n_distinct`, `correlation`) to decide.

```sql
-- Check selectivity stats
SELECT attname, n_distinct, most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'users' AND attname = 'gender';
```

</details>

---

### Q9 — 🟢 When should you NOT create an index?

<details><summary>Click to reveal answer</summary>

**Avoid indexes when:**

1. **Low cardinality** — boolean, gender, status with 2-3 values: seq scan + parallel query beats index
2. **Small tables** — PostgreSQL's seq scan on <1000 rows is faster than index overhead
3. **Write-heavy tables** — every `INSERT`/`UPDATE`/`DELETE` must update all indexes; too many indexes slow writes
4. **Rarely queried columns** — indexes consume disk and memory (shared buffers) even if unused
5. **Columns always used with a function** — `WHERE LOWER(email) = 'x'` doesn't use `idx(email)`

```sql
-- Check unused indexes (after running for a while):
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

</details>

---

### Q10 — 🟡 How do you fix `WHERE LOWER(email) = ?` not using an index?

<details><summary>Click to reveal answer</summary>

A function on an indexed column prevents index use. The fix is a **functional index**.

```sql
-- Bad: index on email is NOT used
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- Fix 1: Create a functional index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- Now PostgreSQL uses the index:
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com'; -- ✅

-- Fix 2: Use citext extension for case-insensitive text type
CREATE EXTENSION IF NOT EXISTS citext;
ALTER TABLE users ALTER COLUMN email TYPE citext;
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'Alice@Example.COM'; -- ✅ case-insensitive
```

</details>

---

### Q11 — 🟡 How does `LIMIT` with `ORDER BY` benefit from an index?

<details><summary>Click to reveal answer</summary>

When the `ORDER BY` column has an index, PostgreSQL can scan the index in order and stop after `LIMIT` rows — no sort needed.

```sql
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- PostgreSQL scans index in DESC order, stops after 10 rows
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- Plan: Index Scan Backward using idx_orders_created_at (no Sort node!)

-- Without index:
-- Plan: Seq Scan → Sort → Limit (much slower on large tables)
```

**For pagination:**
```sql
-- Offset-based (slow for deep pages — must scan all prior rows)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10 OFFSET 10000;

-- Keyset/cursor pagination (fast — uses index)
SELECT * FROM orders
WHERE created_at < '2024-06-01T12:00:00'
ORDER BY created_at DESC LIMIT 10;
```

</details>

---

### Q12 — 🟢 Why should you avoid `SELECT *` in production queries?

<details><summary>Click to reveal answer</summary>

1. **Prevents index-only scans** — extra columns force heap fetches
2. **Wastes network bandwidth** — sends unused columns to application
3. **Breaks application code** on schema changes (new columns added)
4. **Pulls LOB/TOAST columns** unnecessarily (text blobs, jsonb large fields)

```sql
-- Bad
SELECT * FROM products WHERE category_id = 5;

-- Good: only fetch what you need
SELECT id, name, price FROM products WHERE category_id = 5;

-- In JPA/Hibernate: use projections
public interface ProductSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();
}

List<ProductSummary> findByCategoryId(Long categoryId);
```

</details>

---

### Q13 — 🟡 What is table partitioning and when should you use it?

<details><summary>Click to reveal answer</summary>

**Partitioning** splits a large table into smaller physical sub-tables (partitions) while presenting a single logical table.

**Types in PostgreSQL:**
- **Range partitioning** — by date range (most common for time-series)
- **List partitioning** — by discrete values (e.g., country)
- **Hash partitioning** — by hash of a column

```sql
-- Range partitioning by month
CREATE TABLE events (
    id BIGSERIAL,
    user_id BIGINT,
    event_type VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- PostgreSQL uses partition pruning — only scans relevant partition
SELECT * FROM events WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01';
```

**Use partitioning when:** table > 100M rows, queries always filter on partition key, need fast `DROP` old data (drop partition vs DELETE).

</details>

---

### Q14 — 🟡 What is a BRIN index and when is it better than B-tree?

<details><summary>Click to reveal answer</summary>

**BRIN (Block Range INdex)** stores min/max values per range of disk pages. Tiny size (few KB vs GB for B-tree) but only works when data is naturally ordered on disk.

```sql
-- Perfect for: append-only tables where created_at increases over time
CREATE INDEX idx_events_brin ON events USING BRIN (created_at);

-- 128 pages per range by default; tune with pages_per_range
CREATE INDEX idx_events_brin ON events USING BRIN (created_at) WITH (pages_per_range = 64);
```

**When to prefer BRIN over B-tree:**
- Very large append-only tables (logs, events, IoT data)
- Column correlates with physical insert order (timestamps, auto-increment IDs)
- Storage is a concern (BRIN is 1000x smaller than B-tree for same table)

**Downside:** Much less precise than B-tree; PostgreSQL must check entire block ranges.

</details>

---

### Q15 — 🟡 How do you detect slow queries in PostgreSQL?

<details><summary>Click to reveal answer</summary>

```sql
-- 1. Enable pg_stat_statements extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 2. Find top 10 slowest queries
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- 3. Log slow queries in postgresql.conf
-- log_min_duration_statement = 200   -- log queries taking > 200ms

-- 4. Check currently running queries
SELECT pid, now() - query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- 5. Kill a runaway query
SELECT pg_cancel_backend(pid);   -- graceful
SELECT pg_terminate_backend(pid); -- forceful
```

</details>

---

### Q16 — 🟡 What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`?

<details><summary>Click to reveal answer</summary>

| | `EXPLAIN` | `EXPLAIN ANALYZE` |
|---|---|---|
| Executes query? | ❌ No | ✅ Yes |
| Shows estimated rows | ✅ | ✅ |
| Shows actual rows | ❌ | ✅ |
| Shows actual timing | ❌ | ✅ |
| Safe for SELECT | ✅ | ✅ |
| Safe for INSERT/UPDATE | ✅ | ⚠️ Actually modifies data — wrap in transaction and ROLLBACK |

```sql
-- Safe way to EXPLAIN ANALYZE a write:
BEGIN;
EXPLAIN ANALYZE UPDATE orders SET status = 'PAID' WHERE id = 1;
ROLLBACK;

-- Full output with buffers and format:
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM orders WHERE user_id = 1;
```

</details>

---

### Q17 — 🟢 What does `VACUUM` and `ANALYZE` do in PostgreSQL?

<details><summary>Click to reveal answer</summary>

**VACUUM:**
- Reclaims storage from dead tuples left by MVCC (UPDATE/DELETE creates dead rows)
- `VACUUM` — marks dead space as reusable
- `VACUUM FULL` — rewrites entire table (exclusive lock, compacts disk)
- `autovacuum` runs automatically in background

**ANALYZE:**
- Updates planner statistics (`pg_statistic`)
- Without up-to-date stats, planner makes bad cardinality estimates → bad plans

```sql
-- Manual vacuum + analyze
VACUUM ANALYZE orders;

-- Check dead tuples
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

**Autovacuum tuning for high-write tables:**
```sql
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- trigger at 1% dead tuples (default 20%)
    autovacuum_analyze_scale_factor = 0.005
);
```

</details>

---

### Q18 — 🟡 How do you handle index bloat?

<details><summary>Click to reveal answer</summary>

Index bloat occurs when many DELETE/UPDATE operations leave dead index entries that aren't reclaimed by regular `VACUUM` (VACUUM reclaims heap but not always index pages efficiently).

```sql
-- Check index bloat (pgstattuple extension)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT * FROM pgstattuple('idx_orders_user_id');
-- dead_tuple_percent > 20% → consider REINDEX

-- Rebuild index without locking table (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_orders_user_id;

-- Or drop and recreate concurrently
DROP INDEX CONCURRENTLY idx_orders_user_id;
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

</details>

---

### Q19 — 🟡 What is the difference between `EXISTS` and `IN` for subqueries?

<details><summary>Click to reveal answer</summary>

```sql
-- IN: evaluates subquery fully, builds a list, checks membership
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE country = 'US');

-- EXISTS: short-circuits as soon as one match is found
SELECT * FROM orders o WHERE EXISTS (
    SELECT 1 FROM users u WHERE u.id = o.user_id AND u.country = 'US'
);
```

**Performance:**
- `EXISTS` is generally faster when subquery returns many rows (stops at first match)
- `IN` with a small, fixed list is fine: `WHERE status IN ('PAID', 'SHIPPED')`
- PostgreSQL's optimizer often rewrites `IN` to a semi-join anyway — check `EXPLAIN`
- `NOT EXISTS` is usually faster than `NOT IN` (NULL handling: `NOT IN` returns no rows if subquery contains NULL)

</details>

---

### Q20 — 🟡 Explain the difference between `INNER JOIN`, `LEFT JOIN`, and a correlated subquery in terms of performance.

<details><summary>Click to reveal answer</summary>

```sql
-- INNER JOIN: efficient, planner can choose hash join / merge join / nested loop
SELECT o.id, u.name
FROM orders o
INNER JOIN users u ON u.id = o.user_id
WHERE u.country = 'US';

-- LEFT JOIN: returns all orders, NULL for users columns if no match
SELECT o.id, u.name
FROM orders o
LEFT JOIN users u ON u.id = o.user_id;

-- Correlated subquery: executes once per outer row — O(n) queries!
SELECT o.id, (SELECT name FROM users WHERE id = o.user_id) AS user_name
FROM orders o;
-- ⚠️ This is the N+1 problem in SQL form
```

**Rule:** Always prefer `JOIN` over correlated subqueries for bulk retrieval. Correlated subqueries are only acceptable for `EXISTS`/`NOT EXISTS` checks where short-circuiting helps.

</details>

---

### Q21 — 🔴 How would you optimize a query that joins 4 large tables?

<details><summary>Click to reveal answer</summary>

```sql
-- Original slow query
SELECT o.id, u.name, p.title, c.name AS category
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
JOIN categories c ON c.id = p.category_id
WHERE o.created_at > '2024-01-01'
  AND u.country = 'US'
  AND c.name = 'Electronics';
```

**Optimization steps:**

1. **Ensure all join columns are indexed:**
```sql
CREATE INDEX ON orders(user_id);
CREATE INDEX ON order_items(order_id);
CREATE INDEX ON order_items(product_id);
CREATE INDEX ON products(category_id);
```

2. **Filter early — push predicates to CTEs or subqueries:**
```sql
WITH us_users AS (SELECT id, name FROM users WHERE country = 'US'),
     electronics AS (SELECT id FROM categories WHERE name = 'Electronics'),
     elec_products AS (SELECT p.id, p.title FROM products p JOIN electronics e ON e.id = p.category_id)
SELECT o.id, u.name, p.title
FROM orders o
JOIN us_users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
JOIN elec_products p ON p.id = oi.product_id
WHERE o.created_at > '2024-01-01';
```

3. **Use `SET enable_hashjoin = on`** and check planner choice with `EXPLAIN`
4. **Partition `orders` by `created_at`** if the table is very large
5. **Consider materialized view** if this is a recurring report query

</details>

---

### Q22 — 🟡 What is a GIN index and when do you use it?

<details><summary>Click to reveal answer</summary>

**GIN (Generalized Inverted Index)** is designed for composite values where elements need to be searched individually — arrays, JSONB, full-text search.

```sql
-- Array containment search
CREATE INDEX idx_products_tags_gin ON products USING GIN (tags);
SELECT * FROM products WHERE tags @> ARRAY['java', 'backend'];

-- JSONB search
CREATE INDEX idx_events_data_gin ON events USING GIN (data);
SELECT * FROM events WHERE data @> '{"type": "click"}';

-- Full-text search
CREATE INDEX idx_articles_fts ON articles USING GIN (to_tsvector('english', body));
SELECT * FROM articles WHERE to_tsvector('english', body) @@ to_tsquery('java & indexing');
```

**GIN vs GiST:**
- GIN: faster reads, slower writes, larger index (exact matches)
- GiST: faster writes, slower reads (lossy — may have false positives, rechecks heap)

</details>

---

### Q23 — 🟡 How do you use window functions for analytics queries?

<details><summary>Click to reveal answer</summary>

```sql
-- Running total of revenue per user
SELECT
    user_id,
    created_at,
    amount,
    SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM orders;

-- Rank users by total spend
SELECT
    user_id,
    total_spend,
    RANK() OVER (ORDER BY total_spend DESC) AS spend_rank,
    DENSE_RANK() OVER (ORDER BY total_spend DESC) AS dense_rank
FROM (
    SELECT user_id, SUM(amount) AS total_spend FROM orders GROUP BY user_id
) t;

-- Lag/Lead for day-over-day comparison
SELECT
    date_trunc('day', created_at) AS day,
    COUNT(*) AS orders_today,
    LAG(COUNT(*)) OVER (ORDER BY date_trunc('day', created_at)) AS orders_yesterday
FROM orders
GROUP BY 1;
```

Window functions do NOT reduce rows like `GROUP BY` — each row retains its identity.

</details>

---

### Q24 — 🔴 What is connection pooling and how does it affect query performance?

<details><summary>Click to reveal answer</summary>

PostgreSQL spawns a process per connection (~5-10MB RAM each). Without pooling, opening a new connection per request adds 10-50ms overhead and exhausts resources.

**PgBouncer** sits between application and PostgreSQL:
- **Session mode** — client gets one PG connection for its session lifetime
- **Transaction mode** — connection returned to pool after each transaction (most efficient)
- **Statement mode** — returned after each statement (breaks multi-statement transactions)

```yaml
# application.yml with HikariCP
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

**HikariCP sizing formula:**
```
pool_size = (core_count * 2) + effective_spindle_count
```
For 4-core machine with SSD (spindle_count=1): pool_size = 4*2+1 = **9**

Too large a pool causes CPU context switching and DB lock contention.

</details>

---

### Q25 — 🔴 How would you approach optimizing a `GROUP BY` query that is doing a sequential scan?

<details><summary>Click to reveal answer</summary>

```sql
-- Slow: seq scan on 10M row orders table
SELECT status, COUNT(*) FROM orders GROUP BY status;
```

**Step 1:** Check with `EXPLAIN ANALYZE` — is it a seq scan because of low selectivity?

**Step 2:** For low-cardinality GROUP BY (status has 5 values), a partial index per status + `UNION ALL` can beat a seq scan:

```sql
SELECT 'PAID' AS status, COUNT(*) FROM orders WHERE status = 'PAID'
UNION ALL
SELECT 'PENDING', COUNT(*) FROM orders WHERE status = 'PENDING'
UNION ALL
SELECT 'CANCELLED', COUNT(*) FROM orders WHERE status = 'CANCELLED';
```

**Step 3:** For reporting on large tables, use a **materialized view** refreshed periodically:

```sql
CREATE MATERIALIZED VIEW order_status_summary AS
SELECT status, COUNT(*) AS total, SUM(amount) AS revenue
FROM orders
GROUP BY status;

CREATE UNIQUE INDEX ON order_status_summary(status);

-- Refresh without locking reads:
REFRESH MATERIALIZED VIEW CONCURRENTLY order_status_summary;
```

**Step 4:** Consider **TimescaleDB** or **ClickHouse** if analytical queries dominate your workload.

</details>

---

### Q26 — 🟡 What is `work_mem` and how does it affect sort and hash operations?

<details><summary>Click to reveal answer</summary>

`work_mem` is the memory allocated **per sort/hash operation per query**. If the data fits in `work_mem`, PostgreSQL does an in-memory sort; otherwise it spills to disk (temp files — much slower).

```sql
-- Check current setting
SHOW work_mem; -- default 4MB

-- Increase for a specific session (for heavy analytics)
SET work_mem = '256MB';
EXPLAIN ANALYZE SELECT * FROM orders ORDER BY amount DESC;
-- Before: Sort Method: external merge  Disk: 45000kB
-- After:  Sort Method: quicksort  Memory: 180kB

-- Per-user setting (DBA-level)
ALTER ROLE analytics_user SET work_mem = '128MB';
```

**Caution:** If 100 connections each run a query with 3 sort nodes, memory usage = 100 × 3 × work_mem. Set conservatively globally; override per session for heavy queries.

</details>

---

## Summary Table

| Concept | Key Rule |
|---|---|
| B-tree | Default; supports `=`, `<`, `>`, `BETWEEN`, `LIKE 'x%'` |
| Hash | Only `=`; rarely chosen explicitly |
| Composite index | Leftmost prefix rule; equality columns first |
| Covering index | Include all SELECT columns to enable index-only scan |
| Partial index | Add `WHERE` clause to index subset of rows |
| Seq Scan | Not always bad — fast for small tables or low-selectivity |
| Functional index | Required when using functions on indexed columns |
| BRIN | For naturally ordered large append-only tables |
| GIN | For JSONB, arrays, full-text search |
| work_mem | Controls in-memory sort/hash; increase per session for analytics |
