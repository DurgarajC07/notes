# Cost Estimation - The Optimizer's Calculator

## 1. Concept Explanation

**Cost estimation** is how the database optimizer assigns numeric "cost" values to different execution plans to choose the cheapest one. It's not measuring actual time—it's a relative scoring system based on I/O, CPU, and memory usage.

Think of it like a GPS estimating travel time: it uses

 historical data (traffic patterns = table statistics), known constants (speed limits = I/O costs), and heuristics to predict which route is fastest.

### Cost Components

```
Total Query Cost = Startup Cost + Run Cost

Startup Cost: Work before returning first row (sorts, hash table builds)
Run Cost: Work to return all rows (scans, joins, filters)
```

**PostgreSQL Cost Formula:**
```
Cost = (seq_page_cost × sequential_pages_read) +
       (random_page_cost × random_pages_read) +
       (cpu_tuple_cost × rows_processed) +
       (cpu_operator_cost × operations) +
       (cpu_index_tuple_cost × index_

_reads)
```

**Default PostgreSQL Costs:**
- `seq_page_cost` = 1.0 (baseline unit)
- `random_page_cost` = 4.0 (random I/O is 4× slower than sequential)
- `cpu_tuple_cost` = 0.01 (processing one row)
- `cpu_operator_cost` = 0.0025 (one comparison/operation)
- `cpu_index_tuple_cost` = 0.005 (processing one index entry)

### Example Calculation

**Sequential Scan on 10,000-page table:**
```
Cost = (seq_page_cost × pages) + (cpu_tuple_cost × rows)
     = (1.0 × 10,000) + (0.01 × 1,000,000)
     = 10,000 + 10,000
     = 20,000
```

**Index Scan on 100 rows:**
```
Cost = (random_page_cost × index_pages) + (cpu_index_tuple_cost × index_reads) +
       (random_page_cost × heap_pages) + (cpu_tuple_cost × rows)
     = (4.0 × 3) + (0.005 × 100) + (4.0 × 100) + (0.01 × 100)
     = 12 + 0.5 + 400 + 1
     = 413.5
```

**Decision:** Sequential scan (20,000) vs Index scan (413.5)
- For 100 rows → Index scan wins (413.5 < 20,000)
- For 500,000 rows → Sequential scan wins (random I/O too expensive)

---

## 2. Why It Matters

### Production Impact

**Without Understanding Cost Estimation:**
```
❌ "Why isn't my index being used?"
❌ "Optimizer chose the wrong plan"
❌ "Query slow only on production servers"
```

**With Cost Estimation Knowledge:**
```
✅ "Index not used because estimated cost (5000) > seq scan cost (3000)"
✅ "Adjust random_page_cost from 4.0 → 1.5 for SSD storage"
✅ "Cardinality estimate off (100 vs 100K rows) → ANALYZE table"
```

### Real Scenario: SSD vs HDD Configuration

**HDD Configuration (Default):**
```sql
-- random_page_cost = 4.0 (default for spinning disks)
EXPLAIN SELECT * FROM orders WHERE user_id = 12345;

-- Output:
Seq Scan on orders  (cost=0.00..20000.00 rows=100)
  Filter: (user_id = 12345)

-- Cost calculation:
-- Index Scan: 4.0 × 100 random pages = 400
-- Seq Scan: 1.0 × 10,000 sequential pages = 10,000
-- Optimizer chooses Seq Scan (lower cost)... but wrong for SSD!
```

**SSD Configuration (Optimized):**
```sql
-- Adjust for SSD (random I/O only 1.5× slower than sequential)
ALTER DATABASE mydb SET random_page_cost = 1.5;

EXPLAIN SELECT * FROM orders WHERE user_id = 12345;

-- Output:
Index Scan using idx_orders_user_id on orders  (cost=0.43..180.00 rows=100)
  Index Cond: (user_id = 12345)

-- Cost calculation:
-- Index Scan: 1.5 × 100 = 150
-- Seq Scan: 1.0 × 10,000 = 10,000
-- Now Index Scan chosen correctly!
```

**Result:** Query 50× faster after fixing cost parameters for hardware.

---

## 3. Internal Working

### How Optimizer Estimates Costs

```
1. Parse Query → AST
       ↓
2. Get Table Statistics
   - Row counts (pg_class.reltuples)
   - Column distributions (pg_stats)
   - Index availability
       ↓
3. Generate Candidate Plans
   - Scan methods (seq, index, bitmap)
   - Join orders (A→B→C vs B→A→C)
   - Join algorithms (nested loop, hash, merge)
       ↓
4. Estimate Cost for Each Plan
   - Use formulas with statistics
   - Apply selectivity estimates
       ↓
5. Choose Lowest Cost Plan
```

### Cardinality Estimation (Critical for Cost)

**Cardinality** = estimated number of rows returned by an operation.

**Selectivity** = fraction of rows passing a filter (0.0 to 1.0).

**Formula:**
```
Estimated Rows = Table Rows × Selectivity
```

**Example Selectivity Calculations:**

1. **Equality on unique column:**
   ```sql
   WHERE id = 12345  -- Primary key
   Selectivity = 1 / n_distinct = 1 / 1,000,000 = 0.000001
   Estimated rows = 1,000,000 × 0.000001 = 1 row
   ```

2. **Equality on non-unique column:**
   ```sql
   WHERE status = 'active'  -- status has 5 distinct values
   Selectivity = 1 / 5 = 0.2
   Estimated rows = 1,000,000 × 0.2 = 200,000 rows
   ```

3. **Range condition:**
   ```sql
   WHERE price BETWEEN 100 AND 200
   -- Uses histogram (if available) or assumes uniform distribution
   Selectivity = (200 - 100) / (max_price - min_price)
               = 100 / 1000 = 0.1
   Estimated rows = 1,000,000 × 0.1 = 100,000 rows
   ```

4. **Multiple AND conditions (independent):**
   ```sql
   WHERE status = 'active' AND country = 'US'
   Selectivity = sel(status) × sel(country)
               = 0.2 × 0.3 = 0.06
   Estimated rows = 1,000,000 × 0.06 = 60,000 rows
   ```

5. **OR conditions:**
   ```sql
   WHERE status = 'active' OR country = 'US'
   Selectivity = sel(status) + sel(country) - (sel(status) × sel(country))  -- Inclusion-exclusion
               = 0.2 + 0.3 - (0.2 × 0.3) = 0.44
   Estimated rows = 1,000,000 × 0.44 = 440,000 rows
   ```

### Histogram-Based Estimation

**Problem:** Uniform distribution assumption fails for skewed data.

**Example:**
```sql
-- Users table: 1M rows
-- Country distribution:
--   'US': 800,000 (80%)
--   'UK': 100,000 (10%)
--   'CA': 50,000 (5%)
--   Others: 50,000 (5%)

-- Without histogram (assumes uniform):
SELECT * FROM users WHERE country = 'US';
-- Estimated: 1,000,000 / n_distinct(country) = 1,000,000 / 50 = 20,000 rows ❌
-- Actual: 800,000 rows

-- With histogram:
-- Database stores:
--   'US' → 80% of rows
--   'UK' → 10% of rows
--   ...
-- Estimated: 1,000,000 × 0.8 = 800,000 rows ✅
```

**Creating Histograms:**
```sql
-- PostgreSQL (automatic with ANALYZE, adjustable)
ALTER TABLE users ALTER COLUMN country SET STATISTICS 1000;  -- Increase histogram buckets
ANALYZE users;

-- Oracle
EXEC DBMS_STATS.GATHER_TABLE_STATS('schema', 'users', method_opt=>'FOR ALL COLUMNS SIZE 254');

-- SQL Server
UPDATE STATISTICS users WITH FULLSCAN;
```

---

## 4. Best Practices

### Practice 1: Run ANALYZE After Bulk Changes

**❌ Bad: Stale Statistics**
```sql
-- Load 5M new rows
INSERT INTO orders SELECT * FROM staging_orders;  -- 5M rows

-- Run query immediately
SELECT * FROM orders WHERE status = 'pending';

-- Optimizer still thinks table has 100K rows (old statistics)
-- Chooses nested loop join → catastrophically slow
```

**✅ Good: Update Statistics**
```sql
INSERT INTO orders SELECT * FROM staging_orders;
ANALYZE orders;  -- Update statistics

-- Now optimizer knows table has 5.1M rows
-- Chooses hash join → fast
```

### Practice 2: Tune Cost Parameters for Your Hardware

**❌ Bad: Default Settings on Modern Hardware**
```sql
-- Defaults optimized for HDD (year 2000)
SHOW random_page_cost;  -- 4.0
SHOW effective_cache_size;  -- 4GB (way too low for 128GB RAM server)
```

**✅ Good: Tuned for SSD + Large RAM**
```sql
-- For SSD storage:
ALTER SYSTEM SET random_page_cost = 1.1;  -- SSD random I/O nearly as fast as sequential

-- For 128GB RAM server:
ALTER SYSTEM SET effective_cache_size = '96GB';  -- 75% of RAM

-- For NVMe:
ALTER SYSTEM SET random_page_cost = 1.05;  -- Even faster random access

-- Reload configuration
SELECT pg_reload_conf();

-- Re-run problematic queries
-- Optimizer now prefers indexes more aggressively
```

**Hardware-Specific Recommendations:**
| Storage Type | random_page_cost | seq_page_cost |
|--------------|------------------|---------------|
| HDD (spinning) | 4.0 (default) | 1.0 |
| SSD (SATA) | 1.1 - 1.5 | 1.0 |
| NVMe | 1.05 - 1.1 | 1.0 |
| Cloud (EBS) | 1.3 - 2.0 | 1.0 |
| RAM Disk | 1.0 | 1.0 |

### Practice 3: Identify and Fix Cardinality Mismatches

**How to Detect:**
```sql
EXPLAIN (ANALYZE, VERBOSE) [your query];

-- Look for large discrepancies:
Hash Join  (cost=5000.00..10000.00 rows=100 width=50) (actual rows=500000 loops=1)
                                    ^^^^^^^^^^^              ^^^^^^^^
                                    Estimated: 100          Actual: 500K
-- 5000× underestimate!
```

**Common Causes and Fixes:**

**1. Stale Statistics**
```sql
-- Check last analyze time
SELECT schemaname, tablename, last_analyze, n_live_tup
FROM pg_stat_user_tables
WHERE tablename = 'orders';

-- If last_analyze is old:
ANALYZE orders;
```

**2. Correlated Columns (Database Assumes Independence)**
```sql
-- Data: Orders from US have higher average total than other countries
-- Database assumes `country` and `total` are independent

SELECT * FROM orders
WHERE country = 'US' AND total > 1000;

-- Estimated: sel(country) × sel(total) = 0.3 × 0.1 = 0.03 → 30K rows
-- Actual: 0.25 → 250K rows (underestimate!)

-- Fix: Create extended statistics (PostgreSQL 10+)
CREATE STATISTICS orders_country_total_stats (dependencies)
ON country, total FROM orders;

ANALYZE orders;

-- Now optimizer knows correlation, estimates 240K rows ✅
```

**3. Outliers (Data Skew)**
```sql
-- Customer 12345 has 1M orders (bulk buyer)
-- Average customer has 10 orders

SELECT * FROM orders WHERE customer_id = 12345;

-- Estimated: avg = 10 rows
-- Actual: 1M rows (100,000× off!)

-- Fix: Use histograms OR handle outliers separately
CREATE INDEX idx_regular_customers ON orders(customer_id)
WHERE customer_id NOT IN (12345, 67890);  -- Partial index
```

### Practice 4: Use EXPLAIN (Not EXPLAIN ANALYZE) for Cost Comparison

**❌ Bad: Running Slow Query Just to See Plan**
```sql
-- This actually RUNS the query (takes 5 minutes)
EXPLAIN ANALYZE
SELECT * FROM huge_table;  -- 1 billion rows
```

**✅ Good: Estimate Only**
```sql
-- Shows estimated costs without executing
EXPLAIN
SELECT * FROM huge_table;

-- Fast! Database just plans, doesn't execute
```

---

## 5. Common Mistakes

### Mistake 1: Ignoring "rows" in EXPLAIN Output

**Problem:**
```sql
EXPLAIN SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at >= '2024-01-01';

-- Output:
Hash Join  (cost=5000.00..10000.00 rows=100 width=200)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..2000.00 rows=100)  ← Thinks 100 rows
        Filter: (created_at >= '2024-01-01')
  ->  Hash  (cost=2000.00..2000.00 rows=100000)
        ->  Seq Scan on customers c  (cost=0.00..2000.00 rows=100000)

-- Optimizer chose hash join thinking only 100 orders
-- Reality: 500,000 orders matching filter
-- Hash join spills to disk → catastrophically slow
```

**Solution:**
```sql
ANALYZE orders;  -- Update statistics

-- Now shows:
rows=480000  ← Accurate estimate
-- Optimizer chooses different join strategy
```

### Mistake 2: Assuming Cost = Execution Time

**Problem:**
```
Query A: cost=1000.00  actual time=50ms
Query B: cost=2000.00  actual time=30ms

"Query B has higher cost but runs faster???"
```

**Explanation:**
- Cost is **relative estimate** based on model (I/O, CPU)
- Actual time affected by:
  - Cache hits (warm vs cold)
  - Parallel workers
  - Server load
  - Network latency
- Cost useful for **comparing plans for same query**, not across queries

### Mistake 3: Over-Tuning cost_* Parameters

**Problem:**
```sql
-- Trying to "force" index usage
SET random_page_cost = 0.001;  -- Way too low

-- Now optimizer ALWAYS prefers indexes, even when seq scan is better
-- Result: Worse performance overall
```

**Solution:**
```sql
-- Use realistic values based on hardware
-- Measure with tools like fio, pgbench

-- Example tuning process:
-- 1. Run benchmark:
pgbench -c 10 -j 2 -T 60 mydb

-- 2. Measure random vs sequential I/O speed:
-- sequential_speed / random_speed = ratio
-- If ratio = 1.2, set random_page_cost = 1.2

-- 3. Test queries before/after:
EXPLAIN ANALYZE [query];  -- Before
ALTER SYSTEM SET random_page_cost = 1.2;
SELECT pg_reload_conf();
EXPLAIN ANALYZE [query];  -- After
-- Compare actual execution times
```

### Mistake 4: Not Accounting for Data Skew

**Problem:**
```sql
-- Status distribution:
--   'completed': 99%
--   'pending': 0.9%
--   'cancelled': 0.1%

-- Query for rare status:
SELECT * FROM orders WHERE status = 'cancelled';
-- Estimated (uniform): 1M / 3 = 333K rows
-- Actual: 1K rows
-- Optimizer chooses seq scan (wrong!)

-- Query for common status:
SELECT * FROM orders WHERE status = 'completed';
-- Estimated: 333K rows
-- Actual: 990K rows
-- Optimizer chooses index scan (wrong!)
```

**Solution:**
```sql
-- Increase statistics target for skewed columns
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;

-- Or create partial indexes:
CREATE INDEX idx_orders_cancelled ON orders(status) WHERE status = 'cancelled';
CREATE INDEX idx_orders_pending ON orders(status) WHERE status = 'pending';
-- No index for 'completed' (seq scan faster for 99% of rows)
```

---

## 6. Security Considerations

### 1. ANALYZE Can Block Writes

**Risk:**
```sql
-- ANALYZE takes ShareUpdateExclusiveLock
ANALYZE orders;  -- On 100GB table, takes 5 minutes

-- During this time:
-- - SELECT: ✅ Allowed
-- - INSERT: ⚠️ Blocked waiting for lock
```

**Mitigation:**
```sql
-- Use auto-vacuum instead (non-blocking)
ALTER TABLE orders SET (autovacuum_analyze_scale_factor = 0.05);  -- Analyze after 5% changes

-- Or run ANALYZE during maintenance window
-- Or use sampling:
ANALYZE orders (user_id, created_at);  -- Only analyze specific columns
```

### 2. Statistics Expose Data Distribution

**Risk:**
```sql
-- Attacker with read-only access to pg_stats:
SELECT tablename, attname, n_distinct, most_common_vals
FROM pg_stats
WHERE tablename = 'users' AND attname = 'email';

-- Output leaks:
-- most_common_vals: {admin@company.com, ceo@company.com, ...}
-- Reveals admin accounts!
```

**Mitigation:**
```sql
-- Restrict access to pg_stats
REVOKE SELECT ON pg_stats FROM public;
GRANT SELECT ON pg_stats TO dba_role;
```

---

## 7. Performance Optimization

### Optimization 1: Increase Statistics Target for Key Columns

**Problem:** Default statistics (100 buckets) insufficient for high-cardinality columns.

**Solution:**
```sql
-- Increase from default 100 to 10,000 buckets
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 10000;
ANALYZE orders;

-- Check improvement:
SELECT tablename, attname, n_distinct, null_frac
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'user_id';

-- Before: n_distinct = 10000 (capped at default)
-- After: n_distinct = 987654 (accurate)
```

### Optimization 2: Extended Statistics for Correlated Columns

**PostgreSQL 10+ Feature:**
```sql
-- Orders: country and payment_method are correlated
--   US orders: 90% credit card
--   International orders: 60% PayPal

CREATE STATISTICS orders_country_payment_stats (dependencies)
ON country, payment_method FROM orders;

ANALYZE orders;

-- Now queries with both filters get accurate estimates:
SELECT * FROM orders
WHERE country = 'US' AND payment_method = 'credit_card';

-- Before: Assumes independence: 0.3 × 0.7 = 0.21 → 210K rows
-- After: Knows correlation: 0.27 → 270K rows (closer to actual 280K)
```

### Optimization 3: Parallel ANALYZE for Large Tables

**PostgreSQL 14+:**
```sql
-- Enable parallel analyze
ALTER SYSTEM SET max_parallel_maintenance_workers = 4;
SELECT pg_reload_conf();

-- ANALYZE now uses 4 workers
ANALYZE orders;  -- 4× faster on large tables
```

---

## 8. Examples

### Example 1: Debugging Index Not Used

**Scenario:** Index exists but optimizer chooses seq scan

**Query:**
```sql
CREATE INDEX idx_orders_status ON orders(status);

EXPLAIN SELECT * FROM orders WHERE status = 'completed';

-- Output:
Seq Scan on orders  (cost=0.00..20000.00 rows=900000)
  Filter: (status = 'completed')

-- Index ignored! Why?
```

**Analysis:**
```sql
-- Check cost if index were used:
SET enable_seqscan = off;  -- Force index usage

EXPLAIN SELECT * FROM orders WHERE status = 'completed';

-- Output:
Index Scan using idx_orders_status on orders  (cost=0.43..450000.00 rows=900000)
  Index Cond: (status = 'completed')

-- Cost comparison:
-- Seq Scan: 20,000
-- Index Scan: 450,000
-- Index is 22× MORE expensive!
```

**Why Index is More Expensive:**
```
90% of rows match filter (900K / 1M)
Index scan does:
  1. Read index (random I/O): 4.0 × 10,000 index pages = 40,000
  2. Read heap (random I/O): 4.0 × 900,000 heap pages = 3,600,000
  Total: 3,640,000 (adjusted to 450,000 in plan)

Seq scan does:
  1. Read entire table sequentially: 1.0 × 10,000 pages = 10,000
  Total: 10,000

Seq scan wins by 364×!
```

**Solution:** This is correct behavior. For 90% of rows, seq scan IS faster.

**If you need fast index scans, filter more:**
```sql
SELECT * FROM orders WHERE status = 'pending';  -- Only 1% of rows

-- Now:
Index Scan using idx_orders_status on orders  (cost=0.43..500.00 rows=10000)
  Index Cond: (status = 'pending')

-- Index cost: 500 << Seq scan cost: 20,000
-- Index used ✅
```

---

### Example 2: Fixing Cardinality Mismatch

**Scenario:** Join creating 100× more rows than estimated

**Query:**
```sql
EXPLAIN ANALYZE
SELECT u.username, o.order_id, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'US';

-- Output:
Hash Join  (cost=5000.00..15000.00 rows=50000 width=50) (actual rows=5000000 loops=1)
                                    ^^^^^^^^^^^^^              ^^^^^^^^^^^
                                    Est: 50K                  Act: 5M (100× OFF!)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o  (cost=0.00..8000.00 rows=500000)  ← Accurate
  ->  Hash  (cost=4000.00..4000.00 rows=100000)  ← Accurate
        ->  Seq Scan on users u  (cost=0.00..4000.00 rows=100000)
              Filter: (country = 'US')

-- Problem: Assumes avg user has 0.5 orders
-- Reality: US users have 50 orders each (100× more!)
```

**Root Cause:** Correlation between `country='US'` and order count not known.

**Solution 1: Extended Statistics**
```sql
CREATE STATISTICS user_orders_stats (dependencies, ndistinct)
ON country FROM users,
   user_id FROM orders;

ANALYZE users, orders;

-- Rerun EXPLAIN ANALYZE
-- Now: rows=4800000 (much closer to actual 5M)
```

**Solution 2: Denormalize (Add order_count column)**
```sql
ALTER TABLE users ADD COLUMN order_count INTEGER DEFAULT 0;

UPDATE users u SET order_count = (
    SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id
);

CREATE INDEX idx_users_country_orders ON users(country, order_count);

-- Now query with accurate filter:
SELECT u.username, o.order_id, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'US' AND u.order_count > 0;

-- Optimizer knows exactly how many orders to expect
```

---

## 9. Real-World Use Cases

### Use Case 1: Multi-Tenant SaaS Cost Estimation

**Challenge:** Tenant 1 has 10M records, Tenant 2 has 100 records. Same query performs differently.

**Query:**
```sql
SELECT * FROM events
WHERE tenant_id = ?
ORDER BY created_at DESC
LIMIT 100;
```

**Tenant 1 (Large):**
```
Index Scan using idx_events_tenant_created on events  (cost=0.56..50000.00 rows=100)
  Index Cond: (tenant_id = 1)
  Filter: (created_at DESC)

-- Index scan on 10M rows, sort, limit
-- Actual time: 5 seconds (slow!)
```

**Tenant 2 (Small):**
```
Index Scan using idx_events_tenant_created on events  (cost=0.43..15.00 rows=100)
  Index Cond: (tenant_id = 2)

-- Index scan on 100 rows
-- Actual time: 0.01 seconds (fast!)
```

**Solution: Partition by Tenant Size**
```sql
-- Partition large tenants
CREATE TABLE events_tenant_1 PARTITION OF events
FOR VALUES IN (1);

-- Add additional index optimized for recent reads
CREATE INDEX idx_events_tenant_1_created_desc ON events_tenant_1 (created_at DESC);

-- Now query uses optimized index:
Index Scan using idx_events_tenant_1_created_desc on events_tenant_1  (cost=0.43..100.00 rows=100)

-- Actual time: 0.05 seconds (100× faster!)
```

---

## 10. Interview Questions

### Q1: Why might an index not be used even though it exists?

**Senior Answer:**

```
5 main reasons:

1. **Estimated Cost Too High (Most Common)**
   - Accessing >5-10% of table rows via index is more expensive than seq scan
   - Example: 90% of orders are 'completed' → seq scan cheaper
   - Fix: This is correct behavior. Filter more or accept seq scan.

2. **Stale Statistics**
   - Optimizer thinks table has 100K rows, actually has 10M
   - Fix: ANALYZE table;

3. **Wrong Hardware Cost Parameters**
   - random_page_cost=4.0 (HDD default) on SSD server
   - Fix: ALTER SYSTEM SET random_page_cost = 1.1;

4. **Function on Indexed Column**
   - WHERE LOWER(email) = 'x' disables index on email
   - Fix: Functional index OR rewrite query

5. **Data Type Mismatch**
   - WHERE user_id = '12345' (string) on INTEGER column
   - Implicit cast prevents index usage
   - Fix: WHERE user_id = 12345 (integer literal)

Debugging steps:
```sql
-- 1. Check if index would be cheaper:
SET enable_seqscan = off;
EXPLAIN [query];
SET enable_seqscan = on;

-- 2. Compare costs, identify why seq scan cheaper
-- 3. Fix root cause (statistics, cost params, or query rewrite)
```
```

### Q2: Explain how the optimizer estimates JOIN cardinality. What can go wrong?

**Staff Answer:**

```
JOIN cardinality estimation formula:

Base formula:
rows(A JOIN B) = rows(A) × rows(B) × selectivity(join_condition)

Example:
```sql
SELECT * FROM orders o JOIN users u ON o.user_id = u.id;

-- Estimation:
rows(orders) = 1,000,000
rows(users) = 100,000
selectivity(o.user_id = u.id) = 1 / n_distinct(user_id) = 1 / 100,000

Estimated rows = 1,000,000 × 100,000 × (1/100,000) = 1,000,000
```

What goes wrong:

1. **Correlated Join Keys**
   - Assumes independence between tables
   - Reality: Orders from 'US' users have higher avg order count
   - Fix: Extended statistics (PostgreSQL 10+)

2. **Multiple Join Conditions**
   - Assumes conditions are independent
   ```sql
   ON o.user_id = u.id AND o.country = u.country
   ```
   - Optimizer multiplies selectivities: sel(user_id) × sel(country)
   - Reality: country conditions are redundant (100% correlation)
   - Fix: Explicitly model correlation with statistics

3. **Foreign Key with Outliers**
   - Assumes uniform distribution of foreign key references
   - Reality: User 12345 has 1M orders, others have 10 avg
   - Estimation: 1M orders / 100K users = 10 per user (average)
   - Actual for user 12345: 1M (100,000× off!)
   - Fix: Partial indexes excluding outliers

4. **Cascade of Joins (Error Compounds)**
   ```sql
   A JOIN B JOIN C JOIN D
   ```
   - Error at each join multiplies
   - 2× error at each step → 2^3 = 8× error after 3 joins
   - Fix: Break into CTEs with updated statistics after each step

Verification:
```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT ...;

-- Look for:
-- rows=X (estimated) vs actual rows=Y
-- If X/Y > 10 or Y/X > 10 → investigate
```
```

### Q3: When would you adjust random_page_cost? How do you determine the right value?

**Principal Answer:**

```
Adjust random_page_cost when default (4.0) doesn't match your hardware's random vs sequential I/O performance.

**Default Assumption (random_page_cost = 4.0):**
- Based on HDDs (spinning disks, year 2000 hardware)
- Random seek: ~10ms
- Sequential read: ~2.5ms
- Ratio: 10/2.5 = 4.0

**Modern Hardware:**

| Storage | Sequential | Random | Ratio | Recommended random_page_cost |
|---------|-----------|--------|-------|------------------------------|
| HDD | 100 MB/s | 1 MB/s | 100× | 4.0 (default) |
| SSD (SATA) | 500 MB/s | 400 MB/s | 1.25× | 1.1-1.5 |
| NVMe | 3000 MB/s | 2800 MB/s | 1.07× | 1.05-1.1 |
| AWS EBS gp3 | 250 MB/s | 150 MB/s | 1.67× | 1.3-1.7 |
| RAM | Equal | Equal | 1× | 1.0 |

**How to Measure:**

Step 1: Benchmark your storage
```bash
# Sequential read speed
sudo fio --name=seqread --rw=read --bs=128k --size=4G --numjobs=1

# Random read speed
sudo fio --name=randread --rw=randread --bs=8k --size=4G --numjobs=4

# Calculate ratio:
# random_page_cost = seq_speed / random_speed
```

Step 2: Test with real queries
```sql
-- Record current performance
EXPLAIN (ANALYZE, BUFFERS) [typical query];
-- Note: execution_time_1, buffers_read_1

-- Adjust cost
ALTER SYSTEM SET random_page_cost = 1.2;
SELECT pg_reload_conf();

-- Re-run
EXPLAIN (ANALYZE, BUFFERS) [same query];
-- Note: execution_time_2, buffers_read_2

-- If execution_time_2 < execution_time_1 → Keep new value
-- If plans change but time similar → Monitor production before committing
```

Step 3: Monitor production impact
```sql
-- Track query performance over time
SELECT query,
       calls,
       mean_exec_time,
       stddev_exec_time
FROM pg_stat_statements
WHERE query LIKE '%orders%'
ORDER BY mean_exec_time DESC;
```

**Warning Cases:**
- Shared storage (SAN, NAS) → May need higher values (2.0-3.0)
- Network-attached databases → Include network latency
- Mixed workloads → Use average, or tune per-query with SET LOCAL

**Rollback Plan:**
```sql
-- If production breaks:
ALTER SYSTEM RESET random_page_cost;  -- Back to 4.0
SELECT pg_reload_conf();
```
```

---

## 11. Summary

### Key Concepts

1. **Cost is Relative, Not Absolute**
   - Cost units don't equal milliseconds
   - Use cost to compare plans for the same query
   - Actual execution time depends on cache, parallelism, load

2. **Cardinality Drives Cost**
   - Estimated rows (cardinality) is most important input
   - Row count mismatch → Wrong join algorithm, wrong scan type
   - Fix with ANALYZE, extended statistics, or denormalization

3. **Hardware Affects Cost Parameters**
   - Default random_page_cost=4.0 optimized for HDD
   - SSD: Use 1.1-1.5
   - NVMe: Use 1.05-1.1
   - Measure your hardware, don't guess

4. **Statistics Quality = Plan Quality**
   - Stale statistics → bad plans
   - Low statistics target → inaccurate histograms
   - Correlated columns → use extended statistics

### Cost Troubleshooting Flowchart

```
Query slow?
    ↓
Run EXPLAIN ANALYZE
    ↓
Compare estimated vs actual rows
    ├─ Close match (within 2×) → Cost model issue
    │   ├─ Check random_page_cost setting
    │   ├─ Check effective_cache_size
    │   └─ Consider hardware upgrade
    │
    └─ Large mismatch (>10×) → Cardinality issue
        ├─ Last ANALYZE old? → Run ANALYZE
        ├─ Still off? → Check data skew (outliers)
        ├─ Correlated columns? → Extended statistics
        └─ Complex query? → Simplify or add hints
```

### Production Checklist

- [ ] Run ANALYZE after bulk data changes (>5% of table)
- [ ] Tune cost parameters for your hardware (random_page_cost)
- [ ] Monitor cardinality estimates (pg_stat_statements)
- [ ] Increase statistics_target for high-cardinality columns
- [ ] Use extended statistics for correlated columns (PostgreSQL 10+)
- [ ] Set up auto-vacuum to keep statistics current
- [ ] Test cost parameter changes on staging first
- [ ] Document why you deviated from defaults
- [ ] Review query plans after major version upgrades (optimizer changes)

**Remember:** The optimizer is only as good as the statistics you give it. Garbage in, garbage out. Keep statistics fresh, and the optimizer will reward you with good plans.

---

**Next Reading:**
- `04_Join_Algorithms.md` - How nested loop, hash, and merge joins work
- `05_Index_Selection.md` - When the optimizer chooses which index
- `07_Statistics_Management.md` - Deep dive into ANALYZE and histograms
