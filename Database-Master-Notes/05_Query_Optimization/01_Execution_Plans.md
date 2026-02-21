# Execution Plans - Reading the Database's Mind

## 1. Concept Explanation

An **execution plan** (also called query plan or explain plan) is the database optimizer's blueprint for executing a SQL query. It's a tree structure showing:

- **Which algorithms** the database will use (sequential scan, index scan, hash join, etc.)
- **In what order** operations will execute
- **Estimated costs** (I/O, CPU, memory)
- **Row counts** at each step

Think of it like a GPS route: the optimizer evaluates multiple paths and chooses the "cheapest" one based on statistics, indexes, and cost models.

### Key Components

```
Execution Plan Tree:
┌─────────────────────────────────────┐
│ Sort (cost=1000, rows=100)          │  ← Root node (final operation)
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│ Hash Join (cost=800, rows=1000)     │  ← Join operation
└────────┬───────────────┬────────────┘
         │               │
    ┌────▼────┐     ┌───▼─────┐
    │ Seq Scan│     │Index Scan│         ← Leaf nodes (table access)
    │ orders  │     │ customers│
    └─────────┘     └──────────┘
```

### Execution Plan Types

1. **EXPLAIN** - Estimated plan (no execution)
2. **EXPLAIN ANALYZE** - Actual plan (executes query, shows real metrics)
3. **EXPLAIN VERBOSE** - Detailed output with column lists
4. **EXPLAIN (FORMAT JSON)** - Machine-readable format

---

## 2. Why It Matters

### Production Impact

**Without Understanding Plans:**
```
❌ "This query is slow"
❌ "Add more indexes"
❌ "Increase server RAM"
```

**With Execution Plan Analysis:**
```
✅ "Sequential scan on 10M rows → Add index on (user_id, created_at)"
✅ "Nested loop join causing O(N²) → Force hash join"
✅ "Sort operation spilling to disk → Increase work_mem"
```

### Real-World Scenario

**E-commerce Dashboard Query**
```sql
SELECT o.order_id, u.username, SUM(oi.price)
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.created_at >= '2024-01-01'
GROUP BY o.order_id, u.username
ORDER BY SUM(oi.price) DESC
LIMIT 10;
```

**Bad Plan:** 45 seconds, 100% CPU
```
Sort  (cost=150000.00..150025.00 rows=10000)
  ->  HashAggregate  (cost=140000.00..145000.00 rows=10000)
        ->  Nested Loop  (cost=0.00..130000.00 rows=1000000)
              ->  Seq Scan on orders  (cost=0.00..50000.00 rows=100000)
                    Filter: (created_at >= '2024-01-01'::date)
              ->  Index Scan on users  (cost=0.43..0.80 rows=1)
```

**Good Plan:** 0.8 seconds, 20% CPU
```
Limit  (cost=5000.00..5000.25 rows=10)
  ->  Sort  (cost=5000.00..5050.00 rows=10000)
        ->  HashAggregate  (cost=4500.00..4800.00 rows=10000)
              ->  Hash Join  (cost=1000.00..4000.00 rows=100000)
                    Hash Cond: (o.user_id = u.id)
                    ->  Bitmap Heap Scan on orders  (cost=100.00..2000.00 rows=100000)
                          Recheck Cond: (created_at >= '2024-01-01'::date)
                          ->  Bitmap Index Scan on idx_orders_created  (cost=0.00..100.00)
                    ->  Hash  (cost=500.00..500.00 rows=50000)
                          ->  Seq Scan on users  (cost=0.00..500.00 rows=50000)
```

**Difference:** Index on `orders.created_at` changed Seq Scan → Bitmap Index Scan.

---

## 3. Internal Working

### How the Optimizer Works

```
Query Text
    ↓
┌────────────────────┐
│ Parser             │ Parse SQL → Abstract Syntax Tree
└────────┬───────────┘
         ↓
┌────────────────────┐
│ Rewriter           │ Apply views, rules, CTEs
└────────┬───────────┘
         ↓
┌────────────────────┐
│ Planner/Optimizer  │ Generate candidate plans
└────────┬───────────┘
         ↓
┌────────────────────┐
│ Cost Estimation    │ Calculate cost for each plan
└────────┬───────────┘
         ↓
┌────────────────────┐
│ Best Plan Selected │ Lowest cost wins
└────────┬───────────┘
         ↓
┌────────────────────┐
│ Executor           │ Run the plan, return results
└────────────────────┘
```

### Cost Model

**PostgreSQL Cost Formula:**
```
Total Cost = (seq_page_cost × pages_read) +
             (random_page_cost × random_reads) +
             (cpu_tuple_cost × rows_processed) +
             (cpu_operator_cost × operations)
```

**Default Values:**
- `seq_page_cost` = 1.0 (sequential I/O baseline)
- `random_page_cost` = 4.0 (random I/O 4× slower than sequential)
- `cpu_tuple_cost` = 0.01 (processing one row)
- `cpu_operator_cost` = 0.0025 (one operation like comparison)

**Example Calculation:**
```
Sequential Scan on 1M rows table (10,000 pages):
Cost = (1.0 × 10,000) + (0.01 × 1,000,000) = 20,000

Index Scan on 100 rows (100 random page reads):
Cost = (4.0 × 100) + (0.01 × 100) = 401
```

### Scan Types (Efficiency Comparison)

**1. Sequential Scan (Fastest for Large Result Sets)**
```sql
-- Reads entire table sequentially
Seq Scan on orders  (cost=0.00..20000.00 rows=1000000)
  Filter: (status = 'pending')

-- When Used: No index, or >5-10% of rows needed
-- I/O Pattern: Predictable, uses OS readahead
-- Cost: Low per row, high total for large tables
```

**2. Index Scan (Fast for Small Result Sets)**
```sql
-- Uses B-Tree index, accesses heap for each row
Index Scan using idx_order_id on orders  (cost=0.43..8.45 rows=1)
  Index Cond: (order_id = 12345)

-- When Used: <1% of rows, exact matches
-- I/O Pattern: Random access (expensive)
-- Cost: Low total for small results
```

**3. Bitmap Index Scan (Hybrid Approach)**
```sql
-- Step 1: Scan index, build bitmap of matching pages
Bitmap Index Scan on idx_orders_status  (cost=0.00..500.00)
  Index Cond: (status = 'shipped')

-- Step 2: Sort bitmap by page number, scan heap sequentially
Bitmap Heap Scan on orders  (cost=500.00..5000.00 rows=50000)
  Recheck Cond: (status = 'shipped')

-- When Used: 1-10% of rows, combines multiple indexes
-- I/O Pattern: Sequential heap access (efficient)
-- Cost: Middle ground between Seq and Index Scan
```

**4. Index Only Scan (Fastest - No Heap Access)**
```sql
-- All columns in WHERE/SELECT are in index
Index Only Scan using idx_orders_user_created on orders
  (cost=0.43..1000.00 rows=5000)
  Index Cond: (user_id = 123 AND created_at >= '2024-01-01')

-- When Used: Covering index, table visibility map up-to-date
-- I/O Pattern: Index only, no heap reads
-- Cost: Lowest for indexed columns
```

### Join Algorithms

**1. Nested Loop Join (Small Tables, Indexed)**
```
For each row in outer table:
    For each matching row in inner table (via index):
        Output row

Cost: O(N × log M) with index, O(N × M) without
Best for: Outer table small (<1000 rows), inner table indexed
```

```sql
Nested Loop  (cost=0.43..1500.00 rows=100)
  ->  Seq Scan on orders  (cost=0.00..50.00 rows=100)
        Filter: (status = 'pending')
  ->  Index Scan using idx_customer_id on customers  (cost=0.43..14.50 rows=1)
        Index Cond: (id = orders.customer_id)
```

**2. Hash Join (Large Tables, Equality Joins)**
```
1. Build hash table from smaller table (build phase)
2. Scan larger table, probe hash table (probe phase)

Cost: O(N + M) - Linear!
Best for: Large tables, equi-joins (=), sufficient memory
```

```sql
Hash Join  (cost=5000.00..25000.00 rows=100000)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..15000.00 rows=1000000)
  ->  Hash  (cost=2500.00..2500.00 rows=100000)
        ->  Seq Scan on customers c  (cost=0.00..2500.00 rows=100000)
```

**3. Merge Join (Sorted Inputs)**
```
1. Sort both tables by join key
2. Merge sorted streams (like merge sort)

Cost: O(N log N + M log M) if sorting needed, O(N + M) if pre-sorted
Best for: Already sorted data, non-equality joins (<, >)
```

```sql
Merge Join  (cost=10000.00..18000.00 rows=50000)
  Merge Cond: (o.order_date = s.sale_date)
  ->  Sort  (cost=5000.00..5250.00 rows=100000)
        Sort Key: o.order_date
        ->  Seq Scan on orders o
  ->  Sort  (cost=4500.00..4700.00 rows=80000)
        Sort Key: s.sale_date
        ->  Seq Scan on sales s
```

---

## 4. Best Practices

### 1. Always EXPLAIN ANALYZE for Troubleshooting

**❌ Bad: Guessing**
```sql
-- Query is slow, add random indexes
CREATE INDEX idx_random ON orders(random_column);
```

**✅ Good: Measure First**
```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending';

-- Output shows:
-- Seq Scan on orders  (cost=0.00..20000.00 rows=100000) (actual time=0.123..456.789 rows=98234 loops=1)
--   Filter: (status = 'pending'::text)
--   Rows Removed by Filter: 901766
-- Planning Time: 0.234 ms
-- Execution Time: 467.123 ms

-- Analysis: Scanning 1M rows, filtering out 90% → Need index on status
CREATE INDEX idx_orders_status ON orders(status);

-- Verify improvement:
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending';

-- New output:
-- Bitmap Heap Scan on orders  (cost=500.00..5000.00 rows=100000) (actual time=5.234..25.678 rows=98234 loops=1)
--   Recheck Cond: (status = 'pending'::text)
--   ->  Bitmap Index Scan on idx_orders_status  (cost=0.00..475.00 rows=100000)
-- Execution Time: 28.456 ms  ← 16× FASTER
```

### 2. Read Execution Plans Bottom-Up (Tree Leaves First)

```sql
EXPLAIN ANALYZE
SELECT u.username, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at >= '2024-01-01'
GROUP BY u.username;
```

**Output (read from bottom to top):**
```
HashAggregate  (cost=8000.00..8500.00 rows=5000)                    ← 5. Final grouping
  Group Key: u.username
  ->  Hash Left Join  (cost=3000.00..7500.00 rows=100000)           ← 4. Join users + orders
        Hash Cond: (u.id = o.user_id)
        ->  Bitmap Heap Scan on users u (cost=100.00..800.00 rows=10000)  ← 1. Filter users (START HERE)
              Recheck Cond: (created_at >= '2024-01-01'::date)
              ->  Bitmap Index Scan on idx_users_created (cost=0.00..98.50)  ← 1a. Use index
                    Index Cond: (created_at >= '2024-01-01'::date)
        ->  Hash  (cost=2000.00..2000.00 rows=100000)                ← 2. Build hash table
              ->  Seq Scan on orders o (cost=0.00..2000.00 rows=100000)    ← 3. Scan orders
```

**Reading Order:**
1. Bitmap Index Scan → filters users (10k rows)
2. Seq Scan → reads all orders (100k rows)
3. Hash → builds hash table from orders
4. Hash Left Join → joins filtered users with orders
5. HashAggregate → groups by username

### 3. Compare Estimated vs Actual Rows

**❌ Bad: Ignore Row Count Mismatches**
```
Hash Join  (cost=1000.00..5000.00 rows=100 width=16) (actual rows=500000 loops=1)
                                   ^^^^^^^^              ^^^^^^^^^^^^
                                   Estimated: 100       Actual: 500,000 ← HUGE MISMATCH!
```

**Why This Matters:**
- Optimizer chose Hash Join thinking only 100 rows
- With 500k rows, Hash Join spills to disk (slow)
- Should have chosen different join method

**✅ Good: Investigate Statistics**
```sql
-- Check when table was last analyzed
SELECT schemaname, tablename, last_analyze, last_autoanalyze
FROM pg_stat_user_tables
WHERE tablename = 'orders';

-- Result:
-- last_analyze: NULL
-- last_autoanalyze: 2023-06-01 (8 months ago!)

-- Solution: Update statistics
ANALYZE orders;

-- Re-run EXPLAIN ANALYZE
-- Now: (cost=1000.00..8000.00 rows=480000 width=16) (actual rows=500000 loops=1)
--                              ^^^^^^^^^^^              ^^^^^^^^^^^
-- Much closer! Optimizer now chooses better plan.
```

### 4. Watch for Operations That Spill to Disk

**❌ Bad: Ignoring Disk Spills**
```
Sort  (cost=50000.00..55000.00 rows=2000000 width=100) (actual time=12345.678..15678.901 rows=2000000 loops=1)
  Sort Key: created_at
  Sort Method: external merge  Disk: 512000kB  ← SPILLED TO DISK (SLOW!)
  ->  Seq Scan on orders
```

**✅ Good: Increase Memory or Optimize**
```sql
-- Option 1: Increase work_mem for this session
SET work_mem = '1GB';

-- Re-run query
Sort  (cost=50000.00..55000.00 rows=2000000) (actual time=1234.567..1456.789 rows=2000000 loops=1)
  Sort Key: created_at
  Sort Method: quicksort  Memory: 256000kB  ← IN-MEMORY (FAST!)

-- Option 2: Use index to avoid sort
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Now no sort needed:
Index Scan using idx_orders_created_at on orders  (cost=0.43..80000.00 rows=2000000)
```

### 5. Beware of Function Calls on Indexed Columns

**❌ Bad: Function Prevents Index Usage**
```sql
-- Index on created_at exists, but NOT used
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE DATE(created_at) = '2024-01-15';

-- Output:
Seq Scan on orders  (cost=0.00..25000.00 rows=5000)
  Filter: (date(created_at) = '2024-01-15'::date)  ← Function call disables index
```

**✅ Good: Use Index-Friendly Predicates**
```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE created_at >= '2024-01-15'
  AND created_at < '2024-01-16';

-- Output:
Index Scan using idx_orders_created_at on orders  (cost=0.43..800.00 rows=5000)
  Index Cond: (created_at >= '2024-01-15' AND created_at < '2024-01-16')  ← Index used!
```

---

## 5. Common Mistakes

### Mistake 1: Not Analyzing Outliers

**Problem:**
```sql
-- Query fast 99% of the time, slow for specific customers
SELECT * FROM orders WHERE customer_id = 12345;

EXPLAIN ANALYZE shows:
Index Scan using idx_customer_id on orders  (cost=0.43..8.45 rows=1) (actual rows=1000000 loops=1)
                                                            ^^^^^^^^              ^^^^^^^^^^^
                                                            Expected: 1          Actual: 1M
```

**Why:** Customer 12345 is a bulk buyer (1M orders), but statistics show average customer has 10 orders.

**Solution:**
```sql
-- Create partial index excluding outliers
CREATE INDEX idx_regular_customers ON orders(customer_id)
WHERE customer_id NOT IN (12345, 67890);  -- Known bulk buyers

-- Handle outliers separately
-- Query for regular customers uses partial index
-- Query for bulk customers uses different strategy (batch processing)
```

### Mistake 2: Over-Relying on EXPLAIN (Without ANALYZE)

**Problem:**
```sql
EXPLAIN SELECT * FROM orders WHERE status = 'pending';

-- Shows:
Seq Scan on orders  (cost=0.00..20000.00 rows=100)  ← Estimate
```

**Reality:**
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';

-- Shows:
Seq Scan on orders  (cost=0.00..20000.00 rows=100) (actual rows=900000 loops=1)
                                            ^^^^^^              ^^^^^^^^
```

**Why:** EXPLAIN shows estimates, EXPLAIN ANALYZE shows reality. Always use ANALYZE for production debugging.

### Mistake 3: Ignoring Parallel Query Plans

**Problem:**
```sql
-- Slow query on modern multi-core server
EXPLAIN ANALYZE
SELECT COUNT(*) FROM orders WHERE status = 'completed';

-- Output:
Aggregate  (cost=25000.00..25000.01 rows=1) (actual time=5678.901..5678.902 rows=1 loops=1)
  ->  Seq Scan on orders  (cost=0.00..20000.00 rows=2000000) (actual time=0.123..4567.890 rows=2000000 loops=1)
        Filter: (status = 'completed')

-- Using only 1 CPU core!
```

**Solution:**
```sql
-- Enable parallel query
SET max_parallel_workers_per_gather = 4;

EXPLAIN ANALYZE
SELECT COUNT(*) FROM orders WHERE status = 'completed';

-- Output:
Finalize Aggregate  (cost=8000.00..8000.01 rows=1) (actual time=1234.567..1234.568 rows=1 loops=1)
  ->  Gather  (cost=7000.00..8000.00 rows=4) (actual time=1234.560..1234.565 rows=5 loops=1)
        Workers Planned: 4
        Workers Launched: 4
        ->  Partial Aggregate  (cost=6000.00..6000.01 rows=1) (actual time=1230.123..1230.124 rows=1 loops=5)
              ->  Parallel Seq Scan on orders  (cost=0.00..5000.00 rows=500000) (actual time=0.050..890.123 rows=400000 loops=5)
                    Filter: (status = 'completed')

-- 4.6× faster with 4 workers
```

### Mistake 4: Missing Expensive Subqueries

**Problem:**
```sql
EXPLAIN ANALYZE
SELECT o.order_id, o.total,
       (SELECT COUNT(*) FROM order_items WHERE order_id = o.order_id) as item_count
FROM orders o
WHERE o.created_at >= '2024-01-01';

-- Output:
Seq Scan on orders o  (cost=0.00..500000000.00 rows=100000) (actual time=0.123..45678.901 rows=100000 loops=1)
  Filter: (created_at >= '2024-01-01')
  SubPlan 1
    ->  Aggregate  (cost=4500.00..4500.01 rows=1) (actual time=0.456..0.457 rows=1 loops=100000)  ← LOOPS 100K TIMES!
          ->  Seq Scan on order_items  (cost=0.00..4000.00 rows=200) (actual time=0.120..0.400 rows=5 loops=100000)
                Filter: (order_id = o.order_id)

-- Total time: 45 seconds
```

**Solution:**
```sql
-- Use JOIN instead of correlated subquery
EXPLAIN ANALYZE
SELECT o.order_id, o.total, COUNT(oi.id) as item_count
FROM orders o
LEFT JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.created_at >= '2024-01-01'
GROUP BY o.order_id, o.total;

-- Output:
HashAggregate  (cost=8000.00..8500.00 rows=100000) (actual time=800.123..850.456 rows=100000 loops=1)
  Group Key: o.order_id
  ->  Hash Left Join  (cost=3000.00..7000.00 rows=500000) (actual time=50.123..650.789 rows=500000 loops=1)
        Hash Cond: (o.order_id = oi.order_id)
        ->  Seq Scan on orders o  (cost=0.00..2000.00 rows=100000)
              Filter: (created_at >= '2024-01-01')
        ->  Hash  (cost=2000.00..2000.00 rows=500000)
              ->  Seq Scan on order_items oi  (cost=0.00..2000.00 rows=500000)

-- Total time: 0.85 seconds (53× FASTER!)
```

---

## 6. Security Considerations

### 1. Execution Plans Can Leak Data

**Risk:**
```sql
-- Attacker crafts query to extract filter values
EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM users WHERE email = 'admin@company.com' AND password = 'leaked_hash';

-- Output shows:
Filter: ((email = 'admin@company.com'::text) AND (password = 'leaked_hash'::text))
                 ^^^^^^^^^^^^^^^^^^^^^^^^^^             ^^^^^^^^^^^^^^^
```

**Mitigation:**
```sql
-- Restrict EXPLAIN to authorized users
REVOKE EXECUTE ON FUNCTION pg_read_file FROM public;

-- Sanitize plan output in application logs
-- Never log EXPLAIN VERBOSE in production
```

### 2. Timing Attacks via EXPLAIN ANALYZE

**Risk:**
```sql
-- Attacker infers data existence via execution time
EXPLAIN ANALYZE
SELECT * FROM credit_cards WHERE number = '4111111111111111';
-- Execution Time: 0.001 ms (not found)

EXPLAIN ANALYZE
SELECT * FROM credit_cards WHERE number = '4532123456789012';
-- Execution Time: 5.234 ms (found, processed row)
```

**Mitigation:**
```sql
-- Use constant-time operations for sensitive queries
-- Add artificial delays
-- Rate-limit EXPLAIN requests
```

---

## 7. Performance Optimization

### Optimization Workflow

```
1. Identify slow query (pg_stat_statements, slow query log)
2. Run EXPLAIN ANALYZE on production-like data
3. Identify bottleneck (scan type, join algorithm, sort)
4. Apply fix (index, rewrite, configuration)
5. Verify improvement with EXPLAIN ANALYZE
6. Monitor in production
```

### Optimization Strategies by Bottleneck

**Bottleneck: Sequential Scan on Large Table**
```sql
-- Before:
Seq Scan on orders  (cost=0.00..50000.00 rows=100) (actual time=5678.901..5679.123 rows=100 loops=1)
  Filter: (user_id = 12345)
  Rows Removed by Filter: 2000000

-- Fix: Add index
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- After:
Index Scan using idx_orders_user_id on orders  (cost=0.43..50.00 rows=100) (actual time=0.123..0.456 rows=100 loops=1)
  Index Cond: (user_id = 12345)
```

**Bottleneck: Nested Loop Join (Cartesian Product)**
```sql
-- Before:
Nested Loop  (cost=0.00..500000000.00 rows=100000000) (actual time=0.123..456789.012 rows=100000000 loops=1)
  ->  Seq Scan on table_a  (cost=0.00..500.00 rows=10000)
  ->  Seq Scan on table_b  (cost=0.00..500.00 rows=10000)

-- Fix: Add missing join condition or force hash join
SET enable_nestloop = off;

-- After:
Hash Join  (cost=1000.00..5000.00 rows=10000) (actual time=50.123..250.456 rows=10000 loops=1)
  Hash Cond: (table_a.id = table_b.a_id)
  ->  Seq Scan on table_a  (cost=0.00..500.00 rows=10000)
  ->  Hash  (cost=500.00..500.00 rows=10000)
        ->  Seq Scan on table_b  (cost=0.00..500.00 rows=10000)
```

**Bottleneck: Sort Spilling to Disk**
```sql
-- Before:
Sort  (cost=50000.00..55000.00 rows=2000000) (actual time=12345.678..15678.901 rows=2000000 loops=1)
  Sort Key: created_at
  Sort Method: external merge  Disk: 512000kB
  ->  Seq Scan on orders

-- Fix: Increase work_mem or use index
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- After:
Index Scan using idx_orders_created_at on orders  (cost=0.43..80000.00 rows=2000000) (actual time=0.123..1234.567 rows=2000000 loops=1)
```

### Reading Execution Time Breakdown

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING)
SELECT * FROM orders WHERE user_id = 123;
```

**Output:**
```
Index Scan using idx_orders_user_id on orders  (cost=0.43..50.00 rows=100 width=200) (actual time=0.123..5.678 rows=100 loops=1)
  Index Cond: (user_id = 123)
  Buffers: shared hit=25 read=10 dirtied=0 written=0
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Planning Time: 0.234 ms
Execution Time: 6.789 ms
```

**Analysis:**
- **Buffers: shared hit=25** → 25 pages found in shared_buffers (cache) ✅
- **Buffers: read=10** → 10 pages read from disk (slow) ⚠️
- **Execution Time: 6.789 ms** → includes I/O time
- **Improvement:** If read=10 happens frequently, increase shared_buffers

---

## 8. Examples

### Example 1: Fixing Slow Dashboard Query

**Scenario:** E-commerce admin dashboard showing today's sales

**Initial Query (Slow: 45 seconds)**
```sql
SELECT
    p.name,
    COUNT(DISTINCT o.order_id) as order_count,
    SUM(oi.quantity * oi.price) as revenue
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id
WHERE o.created_at >= CURRENT_DATE
GROUP BY p.id, p.name
ORDER BY revenue DESC
LIMIT 20;
```

**EXPLAIN ANALYZE Output:**
```
Limit  (cost=500000.00..500000.05 rows=20) (actual time=45678.901..45678.923 rows=20 loops=1)
  ->  Sort  (cost=500000.00..510000.00 rows=100000) (actual time=45678.890..45678.910 rows=20 loops=1)
        Sort Key: (sum((oi.quantity * oi.price))) DESC
        Sort Method: top-N heapsort  Memory: 25kB
        ->  HashAggregate  (cost=450000.00..480000.00 rows=100000) (actual time=45000.123..45500.456 rows=50000 loops=1)
              Group Key: p.id, p.name
              ->  Hash Left Join  (cost=200000.00..400000.00 rows=5000000) (actual time=1000.123..42000.789 rows=5000000 loops=1)
                    Hash Cond: (oi.order_id = o.order_id)
                    ->  Seq Scan on order_items oi  (cost=0.00..100000.00 rows=5000000) (actual time=0.050..15000.123 rows=5000000 loops=1)
                    ->  Hash  (cost=150000.00..150000.00 rows=1000000) (actual time=900.456..900.456 rows=1000000 loops=1)
                          Buckets: 1048576  Batches: 2  Memory Usage: 50000kB
                          ->  Seq Scan on orders o  (cost=0.00..150000.00 rows=1000000) (actual time=0.123..700.456 rows=1000000 loops=1)
                                Filter: (created_at >= CURRENT_DATE)
                                Rows Removed by Filter: 9000000  ← PROBLEM: Scanning 10M rows, filtering 9M
              ->  Hash  (cost=20000.00..20000.00 rows=100000) (actual time=50.123..50.123 rows=100000 loops=1)
                    ->  Seq Scan on products p  (cost=0.00..20000.00 rows=100000)

Planning Time: 5.678 ms
Execution Time: 45690.234 ms
```

**Problems Identified:**
1. **Seq Scan on orders filtering 9M rows** → Need index on `orders.created_at`
2. **Seq Scan on order_items (5M rows)** → Need index on `order_items.order_id`
3. **Hash Join spilling to disk (Batches: 2)** → Need more memory

**Fixes Applied:**
```sql
-- Fix 1: Index on orders.created_at
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Fix 2: Index on order_items (order_id, product_id) for joins
CREATE INDEX idx_order_items_order_product ON order_items(order_id, product_id);

-- Fix 3: Increase work_mem for this query
SET work_mem = '256MB';

-- Re-run query
```

**Optimized EXPLAIN ANALYZE:**
```
Limit  (cost=8000.00..8000.05 rows=20) (actual time=850.123..850.145 rows=20 loops=1)
  ->  Sort  (cost=8000.00..8500.00 rows=50000) (actual time=850.110..850.130 rows=20 loops=1)
        Sort Key: (sum((oi.quantity * oi.price))) DESC
        Sort Method: top-N heapsort  Memory: 25kB
        ->  HashAggregate  (cost=7000.00..7500.00 rows=50000) (actual time=800.123..820.456 rows=50000 loops=1)
              Group Key: p.id, p.name
              ->  Hash Left Join  (cost=3000.00..6500.00 rows=100000) (actual time=50.123..750.789 rows=100000 loops=1)
                    Hash Cond: (oi.order_id = o.order_id)
                    ->  Index Scan using idx_order_items_order_product on order_items oi  (cost=0.43..2000.00 rows=100000)
                    ->  Hash  (cost=2500.00..2500.00 rows=50000) (actual time=45.678..45.678 rows=50000 loops=1)
                          Buckets: 65536  Batches: 1  Memory Usage: 5000kB  ← No disk spill!
                          ->  Bitmap Heap Scan on orders o  (cost=500.00..2500.00 rows=50000) (actual time=5.123..35.456 rows=50000 loops=1)
                                Recheck Cond: (created_at >= CURRENT_DATE)
                                Heap Blocks: exact=5000
                                ->  Bitmap Index Scan on idx_orders_created_at  (cost=0.00..487.50 rows=50000)  ← Index used!
                                      Index Cond: (created_at >= CURRENT_DATE)
              ->  Hash  (cost=20000.00..20000.00 rows=100000)
                    ->  Seq Scan on products p  (cost=0.00..20000.00 rows=100000)

Execution Time: 860.789 ms  ← 53× FASTER (45s → 0.86s)
```

**Results:**
- **Before:** 45 seconds, full table scans
- **After:** 0.86 seconds, index scans
- **Improvement:** 53× faster

---

### Example 2: Debugging Incorrect Join Order

**Scenario:** Customer report joining 3 tables (users, orders, payments)

**Query:**
```sql
SELECT u.username, o.order_id, p.amount
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN payments p ON o.id = p.order_id
WHERE u.country = 'US'
  AND o.created_at >= '2024-01-01'
  AND p.status = 'completed';
```

**Bad Plan (Nested Loop Hell):**
```
Nested Loop  (cost=0.43..5000000.00 rows=100) (actual time=0.123..456789.012 rows=50000 loops=1)
  ->  Nested Loop  (cost=0.43..3000000.00 rows=1000) (actual time=0.100..345678.901 rows=100000 loops=1)
        ->  Seq Scan on users u  (cost=0.00..50000.00 rows=100000) (actual time=0.050..500.123 rows=1000000 loops=1)
              Filter: (country = 'US')
              Rows Removed by Filter: 9000000  ← Scanning 10M users
        ->  Index Scan using idx_orders_user_id on orders o  (cost=0.43..29.50 rows=10) (actual time=0.345..0.350 rows=100 loops=1000000)
              Index Cond: (user_id = u.id)
              Filter: (created_at >= '2024-01-01')
              Rows Removed by Filter: 90  ← Checking date filter AFTER join
  ->  Index Scan using idx_payments_order_id on payments p  (cost=0.43..2.00 rows=1) (actual time=0.010..0.011 rows=1 loops=100000)
        Index Cond: (order_id = o.id)
        Filter: (status = 'completed')
```

**Problems:**
1. Scanning 10M users, filtering after
2. Nested loops on large intermediate result
3. Date filter applied too late

**Solution: Add Indexes and Let Optimizer Choose Better Plan**
```sql
-- Add covering indexes
CREATE INDEX idx_users_country ON users(country);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);
CREATE INDEX idx_payments_order_status ON payments(order_id, status);

-- Update statistics
ANALYZE users, orders, payments;
```

**Good Plan (Hash Joins):**
```
Hash Join  (cost=8000.00..15000.00 rows=50000) (actual time=100.123..850.456 rows=50000 loops=1)
  Hash Cond: (o.user_id = u.id)
  ->  Hash Join  (cost=4000.00..10000.00 rows=100000) (actual time=50.123..700.789 rows=100000 loops=1)
        Hash Cond: (p.order_id = o.id)
        ->  Bitmap Heap Scan on payments p  (cost=500.00..2000.00 rows=100000) (actual time=5.123..150.456 rows=100000 loops=1)
              Recheck Cond: (status = 'completed')
              ->  Bitmap Index Scan on idx_payments_order_status  (cost=0.00..475.00 rows=100000)
                    Index Cond: (status = 'completed')
        ->  Hash  (cost=3000.00..3000.00 rows=150000) (actual time=40.123..40.123 rows=150000 loops=1)
              Buckets: 262144  Batches: 1  Memory Usage: 12000kB
              ->  Bitmap Heap Scan on orders o  (cost=1000.00..3000.00 rows=150000) (actual time=10.123..30.456 rows=150000 loops=1)
                    Recheck Cond: (created_at >= '2024-01-01')
                    ->  Bitmap Index Scan on idx_orders_user_created  (cost=0.00..962.50 rows=150000)
                          Index Cond: (created_at >= '2024-01-01')
  ->  Hash  (cost=3000.00..3000.00 rows=200000) (actual time=45.678..45.678 rows=200000 loops=1)
        Buckets: 262144  Batches: 1  Memory Usage: 15000kB
        ->  Bitmap Heap Scan on users u  (cost=500.00..3000.00 rows=200000) (actual time=5.123..35.456 rows=200000 loops=1)
              Recheck Cond: (country = 'US')
              ->  Bitmap Index Scan on idx_users_country  (cost=0.00..450.00 rows=200000)
                    Index Cond: (country = 'US')

Execution Time: 860.789 ms  ← 530× FASTER (456s → 0.86s)
```

**Key Improvements:**
1. Bitmap scans filter early (reduce intermediate results)
2. Hash joins instead of nested loops
3. All filters use indexes

---

## 9. Real-World Use Cases

### Use Case 1: SaaS Multi-Tenant Query Optimization

**Scenario:** SaaS platform with 50,000 tenants, 100M records

**Query:**
```sql
SELECT * FROM events
WHERE tenant_id = 12345
  AND event_type = 'page_view'
  AND created_at >= NOW() - INTERVAL '30 days'
ORDER BY created_at DESC
LIMIT 100;
```

**Challenge:** Query plan varies drastically by tenant size
- Small tenant (100 events) → Index Scan (0.01s)
- Large tenant (10M events) → Sequential Scan (45s)

**Solution: Partitioning + Partial Indexes**
```sql
-- Partition by tenant_id ranges
CREATE TABLE events_tenant_0_1000 PARTITION OF events
FOR VALUES FROM (0) TO (1000);

CREATE TABLE events_tenant_1000_2000 PARTITION OF events
FOR VALUES FROM (1000) TO (2000);

-- Create indexes on each partition
CREATE INDEX ON events_tenant_0_1000 (tenant_id, created_at);
CREATE INDEX ON events_tenant_1000_2000 (tenant_id, created_at);

-- EXPLAIN ANALYZE now shows:
Limit  (cost=0.43..100.00 rows=100) (actual time=0.123..5.678 rows=100 loops=1)
  ->  Index Scan using events_tenant_0_1000_tenant_id_created_at_idx on events_tenant_0_1000
        Index Cond: (tenant_id = 12345 AND created_at >= NOW() - INTERVAL '30 days')
        Filter: (event_type = 'page_view')

-- Partition pruning eliminates scanning other partitions
-- Query time: 0.005s for all tenant sizes
```

### Use Case 2: Analytics Query with Columnar Storage

**Scenario:** Data warehouse running aggregate queries on 1B rows

**Query:**
```sql
SELECT
    DATE_TRUNC('day', created_at) as day,
    COUNT(*) as event_count,
    COUNT(DISTINCT user_id) as unique_users
FROM events
WHERE created_at >= '2024-01-01'
GROUP BY DATE_TRUNC('day', created_at)
ORDER BY day;
```

**Row-Based Storage (Slow):**
```
EXPLAIN ANALYZE shows:
HashAggregate  (cost=500000.00..550000.00 rows=365) (actual time=45678.901..45680.123 rows=365 loops=1)
  Group Key: date_trunc('day', created_at)
  ->  Seq Scan on events  (cost=0.00..450000.00 rows=100000000) (actual time=0.123..42000.789 rows=100000000 loops=1)
        Filter: (created_at >= '2024-01-01')
        Rows Removed by Filter: 900000000

Execution Time: 45690.456 ms  ← 45 seconds
```

**Columnar Storage with cstore_fdw (Fast):**
```sql
-- Convert to columnar storage
CREATE FOREIGN TABLE events_columnar (
    created_at TIMESTAMP,
    user_id INTEGER,
    event_type VARCHAR
) SERVER cstore_server
OPTIONS (filename '/data/events.cstore', compression 'pglz');

-- Same query on columnar table:
EXPLAIN ANALYZE shows:
HashAggregate  (cost=50000.00..50365.00 rows=365) (actual time=2345.678..2346.789 rows=365 loops=1)
  Group Key: date_trunc('day', created_at)
  ->  Foreign Scan on events_columnar  (cost=0.00..40000.00 rows=100000000) (actual time=50.123..2100.456 rows=100000000 loops=1)
        Filter: (created_at >= '2024-01-01')
        CStore File: /data/events.cstore
        CStore Projections: created_at, user_id  ← Only reads 2 columns!
        Rows Removed by Filter: 0  ← Predicate pushdown at storage level

Execution Time: 2350.123 ms  ← 19× FASTER (45s → 2.3s)
```

**Why Columnar is Faster:**
- Only reads needed columns (created_at, user_id), not entire rows
- Better compression (10× smaller on disk)
- Predicate pushdown to storage layer

---

## 10. Interview Questions

### Q1: How would you debug a query that's slow only sometimes?

**Senior Level Answer:**

```
1. Capture execution plans during slow periods:
   - Use auto_explain extension in PostgreSQL
   - Log slow queries with EXPLAIN ANALYZE

2. Compare plans (fast vs slow):
   - Different join algorithms?
   - Different scan types?
   - Row count mismatches?

3. Common causes:
   a) Parameter Sniffing (SQL Server)
      - Plan cached for different parameter values
      - Solution: OPTION (RECOMPILE) or plan guides

   b) Stale Statistics
      - Estimate 100 rows, actually 1M rows
      - Solution: ANALYZE tables regularly

   c) Lock Contention
      - Query waiting for locks, not CPU-bound
      - Solution: Check pg_locks, reduce transaction duration

   d) Memory Pressure
      - Hash joins spilling to disk during peak times
      - Solution: Increase work_mem or use indexes

4. Reproduce issue:
   - Run EXPLAIN ANALYZE with production-like data
   - Simulate concurrent load (pgbench, JMeter)

Example diagnostic query:
```sql
-- PostgreSQL: Find queries with varying execution times
SELECT query,
       calls,
       mean_exec_time,
       stddev_exec_time,
       min_exec_time,
       max_exec_time
FROM pg_stat_statements
WHERE stddev_exec_time > mean_exec_time  -- High variance
ORDER BY stddev_exec_time DESC
LIMIT 20;
```
```

---

### Q2: Explain the difference between Seq Scan, Index Scan, and Bitmap Scan. When does the optimizer choose each?

**Staff Level Answer:**

```
1. Sequential Scan:
   - Reads entire table sequentially (all pages in order)
   - When: >5-10% of rows needed, no suitable index, table smaller than threshold
   - Cost: seq_page_cost × pages (default 1.0 per page)
   - Example: Full table count, filtering on non-indexed column

2. Index Scan:
   - Uses B-Tree index, accesses heap for each row
   - When: <1% of rows, exact lookups, high selectivity
   - Cost: random_page_cost × pages (default 4.0 per page) + index cost
   - Example: WHERE id = 12345 (primary key lookup)

3. Bitmap Index Scan + Bitmap Heap Scan:
   - Two-phase: Build bitmap of matching pages → Scan heap sequentially
   - When: 1-10% of rows, multiple index conditions (AND/OR)
   - Cost: Hybrid (index random reads + sequential heap reads)
   - Example: WHERE status = 'active' AND created_at >= '2024-01-01'

Decision factors:
- Row count: Seq < Bitmap < Index (for increasing selectivity)
- Memory: Bitmap needs memory for bitmap storage
- I/O pattern: Seq (sequential) < Bitmap (mostly sequential) < Index (random)

Example cost comparison for 1M row table:
- Seq Scan (100% of rows): 10,000 pages × 1.0 = 10,000
- Bitmap Scan (5% of rows): 500 pages × 1.0 + index overhead = 800
- Index Scan (0.01% of rows): 100 rows × 4.0 random reads = 400
```

---

### Q3: What does "actual time=0.123..456.789" mean in EXPLAIN ANALYZE output?

**Mid-Level Answer:**

```
Format: actual time=START..END

- START (0.123 ms): Time to return FIRST row
- END (456.789 ms): Time to return ALL rows

Key insights:

1. Large difference (START << END):
   - Iterative processing (e.g., large scan)
   - Example: Seq Scan on 1M rows
     actual time=0.050..450.123
     (finds first row quickly, processes rest slowly)

2. Small difference (START ≈ END):
   - All-or-nothing operation (e.g., sort)
   - Example: Sort
     actual time=500.123..500.456
     (must finish entire sort before returning first row)

3. For LIMIT queries:
   - START time matters most (early termination)
   - Example:
     Limit (rows=10)
       -> Index Scan actual time=0.050..0.123
     Only processes until 10 rows found

4. Multiplication by loops:
   - Total time = (END - START) × loops
   - Example:
     Nested Loop (loops=1000)
       -> Index Scan actual time=0.100..0.200 (loops=1000)
     Real time: 0.200 ms × 1000 = 200 ms total

Production debugging:
- If START is high → Slow to find first row (index scan startup cost)
- If END - START is high → Slow to process all rows (large dataset)
```

---

### Q4: How do you force the optimizer to use a specific join algorithm?

**Senior Level Answer:**

```
PostgreSQL:
SET enable_nestloop = off;     -- Disable nested loop join
SET enable_hashjoin = off;     -- Disable hash join
SET enable_mergejoin = off;    -- Disable merge join

MySQL:
SELECT /*+ NO_BNL(table_a, table_b) */ ...   -- Disable block nested loop
SELECT /*+ JOIN_ORDER(t1, t2, t3) */ ...     -- Force specific join order

SQL Server:
SELECT * FROM table_a
INNER HASH JOIN table_b ON ...   -- Force hash join
OPTION (LOOP JOIN)               -- Force nested loop for entire query

Oracle:
SELECT /*+ USE_HASH(a b) */ ...  -- Force hash join between a and b
SELECT /*+ USE_NL(a b) */ ...    -- Force nested loop

When to force join algorithm:
1. Optimizer lacks statistics (stale ANALYZE)
2. Complex queries with 10+ tables
3. Known data skew (outliers)
4. Testing performance differences

Best practice:
- Forcing should be temporary (investigate root cause)
- Update statistics first: ANALYZE table;
- Document why hint is needed
- Remove hints after optimizer fix

Example diagnosis:
```sql
-- Check why optimizer chose nested loop
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at >= '2024-01-01';

-- Output shows:
-- Nested Loop (cost=0.43..500000.00 rows=100) (actual rows=1000000)
--                                    ^^^^^^^^              ^^^^^^^^^^
-- Cardinality mismatch! Optimizer thinks 100 rows, actually 1M

-- Solution: Update statistics
ANALYZE orders;

-- Re-run EXPLAIN
-- Now shows: Hash Join (cost=5000.00..10000.00 rows=980000) (actual rows=1000000)
-- Optimizer fixed itself after accurate statistics
```
```

---

### Q5: Walk me through optimizing this slow query.

**Given Query (Running 30 seconds):**
```sql
SELECT
    u.username,
    COUNT(DISTINCT o.order_id) as order_count,
    AVG(o.total) as avg_order_value,
    STRING_AGG(DISTINCT p.name, ', ') as products_purchased
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
LEFT JOIN products p ON oi.product_id = p.id
WHERE u.created_at >= '2023-01-01'
  AND o.status = 'completed'
GROUP BY u.id, u.username
HAVING COUNT(DISTINCT o.order_id) >= 5
ORDER BY avg_order_value DESC
LIMIT 100;
```

**Principal Level Answer:**

```
Step 1: Analyze Current Plan
---------------------------------
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
[query];

Issues found:
1. Seq Scan on users (10M rows) → Need index on created_at
2. Nested Loop joins → Should be hash joins
3. STRING_AGG on products → Expensive aggregation
4. LEFT JOIN but filtering on o.status → Should be INNER JOIN
5. HAVING filter applied too late

Step 2: Schema Analysis
---------------------------------
-- Check existing indexes
SELECT schemaname, tablename, indexname, indexdef
FROM pg_indexes
WHERE tablename IN ('users', 'orders', 'order_items', 'products');

-- Check table sizes
SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
FROM pg_tables
WHERE tablename IN ('users', 'orders', 'order_items', 'products');

Results:
- users: 10M rows, 5GB, index on id only
- orders: 50M rows, 20GB, index on id, user_id
- order_items: 200M rows, 80GB, index on order_id, product_id
- products: 100K rows, 50MB, index on id

Step 3: Query Rewrite
---------------------------------
-- Fix 1: Change LEFT JOIN to INNER JOIN (filtering on o.status makes it inner)
-- Fix 2: Pre-filter orders before joining
-- Fix 3: Eliminate expensive STRING_AGG if possible (or materialize separately)

Rewritten query:
```sql
WITH qualified_users AS (
    -- Pre-filter users
    SELECT id, username
    FROM users
    WHERE created_at >= '2023-01-01'
),
user_orders AS (
    -- Pre-filter orders and aggregate
    SELECT
        o.user_id,
        COUNT(DISTINCT o.order_id) as order_count,
        AVG(o.total) as avg_order_value
    FROM orders o
    WHERE o.status = 'completed'
      AND o.user_id IN (SELECT id FROM qualified_users)  -- Semi-join optimization
    GROUP BY o.user_id
    HAVING COUNT(DISTINCT o.order_id) >= 5  -- Filter early
)
SELECT
    qu.username,
    uo.order_count,
    uo.avg_order_value,
    -- Materialize products separately or remove if not critical
    '(computed separately)' as products_purchased
FROM qualified_users qu
INNER JOIN user_orders uo ON qu.id = uo.user_id
ORDER BY uo.avg_order_value DESC
LIMIT 100;
```

Step 4: Add Indexes
---------------------------------
-- Index for users filter
CREATE INDEX idx_users_created_at ON users(created_at) WHERE created_at >= '2023-01-01';

-- Covering index for orders aggregation
CREATE INDEX idx_orders_user_status_total ON orders(user_id, status) INCLUDE (total, order_id);

-- Verify index usage
EXPLAIN ANALYZE [rewritten_query];

Step 5: Configuration Tuning
---------------------------------
-- Increase memory for hash aggregations
SET work_mem = '256MB';

-- Enable parallel aggregation
SET max_parallel_workers_per_gather = 4;

Step 6: Measure Results
---------------------------------
Before:
- Execution Time: 30,000 ms
- Buffers: shared read=500000 (4GB read from disk)
- Sequential scans on all tables

After:
- Execution Time: 450 ms (67× faster)
- Buffers: shared hit=5000 read=100 (mostly cached)
- Index scans + hash joins

Step 7: Monitor Production
---------------------------------
-- Track query performance over time
SELECT query_id, mean_exec_time, stddev_exec_time, calls
FROM pg_stat_statements
WHERE query LIKE '%qualified_users%'
ORDER BY mean_exec_time DESC;

-- Set up alerting for regression
-- Alert if execution time > 1 second
```

**Key Takeaways:**
1. Always measure first (EXPLAIN ANALYZE)
2. Rewrite query logic before adding indexes
3. Use CTEs to structure complex queries
4. Add indexes based on actual plan bottlenecks
5. Monitor after deployment (production behavior ≠ local testing)
```

---

## 11. Summary

### Key Concepts

1. **Execution Plans are Essential**
   - EXPLAIN ANALYZE shows actual vs estimated performance
   - Bottom-up reading (leaf nodes first)
   - Watch for row count mismatches

2. **Scan Types Matter**
   - Sequential: Large result sets (>5-10% of table)
   - Index: Small result sets (<1% of table)
   - Bitmap: Middle ground (1-10% of table)
   - Index Only: Best when covering index exists

3. **Join Algorithms Have Trade-offs**
   - Nested Loop: Small outer, indexed inner
   - Hash Join: Large tables, equi-joins, sufficient memory
   - Merge Join: Pre-sorted inputs, non-equality joins

4. **Statistics Drive Decisions**
   - Stale statistics → bad plans
   - Run ANALYZE after bulk changes
   - Check actual vs estimated rows

5. **Optimization is Iterative**
   - Measure → Identify bottleneck → Fix → Verify
   - One change at a time
   - Always compare before/after

### Decision Matrix

| Situation | Likely Cause | Solution |
|-----------|--------------|----------|
| Seq Scan on filtered column | Missing index | CREATE INDEX |
| Nested Loop with large tables | Lack of statistics | ANALYZE tables |
| Sort spilling to disk | Insufficient memory | Increase work_mem or use index |
| High estimated vs actual rows | Stale statistics | ANALYZE, consider histograms |
| Cartesian product (huge rows) | Missing join condition | Add WHERE clause |
| Slow only sometimes | Parameter sniffing | RECOMPILE or plan guides |

### Production Checklist

- [ ] Use EXPLAIN ANALYZE for all slow queries
- [ ] Compare estimated vs actual row counts
- [ ] Check for sequential scans on large tables
- [ ] Verify indexes are actually used
- [ ] Monitor for disk spills (sorts, hash joins)
- [ ] Update statistics regularly (ANALYZE)
- [ ] Review join algorithms (nested loop on large tables = bad)
- [ ] Enable auto_explain for production debugging
- [ ] Set up pg_stat_statements monitoring
- [ ] Alert on execution time regressions

**Remember:** The execution plan is the database's language for explaining its decisions. Learn to read it fluently, and you'll solve performance problems 10× faster.

---

**Next Steps:**
- Practice reading plans with `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)`
- Enable pg_stat_statements in production
- Study `02_Query_Rewriting.md` for rewrite patterns
- Review `04_Join_Algorithms.md` for join internals
