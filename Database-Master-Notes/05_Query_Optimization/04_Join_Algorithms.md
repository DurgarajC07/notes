# Join Algorithms - The Heart of Query Performance

## 1. Concept Explanation

**Join algorithms** are the methods databases use to combine rows from two tables based on a join condition. The choice of algorithm dramatically affects performance—sometimes by 1000× or more.

Think of it like merging two sorted lists: you could compare every item from list A with every item from list B (slow), or use smarter strategies that exploit sorting, hash tables, or indexes.

### The Three Core Algorithms

```
1. Nested Loop Join
   - Iterate outer table, lookup inner table for each row
   - Best for: Small outer table, indexed inner table
   - Complexity: O(N × log M) with index, O(N × M) without

2. Hash Join
   - Build hash table from smaller table, probe with larger table
   - Best for: Large tables, equi-joins (=), sufficient memory
   - Complexity: O(N + M) - Linear!

3. Merge Join
   - Sort both tables, merge like merge-sort
   - Best for: Already sorted data, range joins (<, >, BETWEEN)
   - Complexity: O(N log N + M log M) if sorting needed
```

### Visual Comparison

**Nested Loop Join:**
```
orders (1000 rows)          customers (index on id)
     ↓                              ↓
   For each order:           Index lookup (0.01ms each)
     order.customer_id  ───→  customer.id = ?
                              
Total: 1000 × 0.01ms = 10ms ✅
```

**Hash Join:**
```
customers (10K rows)        orders (100K rows)
     ↓                               ↓
Build hash table (50ms)     Probe hash table (100ms)
{id → row}                  For each order:
                              hash(customer_id) → customer row
                              
Total: 50ms + 100ms = 150ms ✅
```

**Merge Join:**
```
orders (sorted by user_id)  users (sorted by id)
     ↓                               ↓
Merge algorithm (like zipper):
  If orders.user_id < users.id → advance orders
  If orders.user_id > users.id → advance users
  If orders.user_id = users.id → output, advance both
  
Total: O(N + M) linear scan = 200ms ✅
```

---

## 2. Why It Matters

### Production Impact

**Wrong Join Algorithm:**
```sql
-- Query takes 45 seconds
SELECT o.order_id, c.customer_name
FROM orders o  -- 1M rows
JOIN customers c ON o.customer_id = c.id  -- 100K rows
WHERE o.created_at >= '2024-01-01';

-- Plan shows:
Nested Loop  (cost=0.43..5000000.00 rows=100000) (actual time=0.123..45678.901 rows=100000 loops=1)
  ->  Seq Scan on orders o  (cost=0.00..50000.00 rows=100000)
        Filter: (created_at >= '2024-01-01')
  ->  Index Scan on customers c  (cost=0.43..49.50 rows=1)
        Index Cond: (id = o.customer_id)

-- Problem: 100K index lookups (100K × 0.45s = 45s)
```

**Correct Join Algorithm:**
```sql
-- After fixing statistics: ANALYZE orders, customers;

-- Plan shows:
Hash Join  (cost=5000.00..15000.00 rows=100000) (actual time=50.123..850.456 rows=100000 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..8000.00 rows=100000)
        Filter: (created_at >= '2024-01-01')
  ->  Hash  (cost=2500.00..2500.00 rows=100000)
        ->  Seq Scan on customers c  (cost=0.00..2500.00 rows=100000)

-- Result: 0.85 seconds (53× FASTER!)
```

### Real Scenario: E-commerce Order History

**Scenario:** User viewing their order history page

```sql
-- Get last 10 orders with product details
SELECT o.order_id, o.total, p.product_name, p.price
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.user_id = 12345
ORDER BY o.created_at DESC
LIMIT 10;
```

**Bad Plan (Nested Loop on Large Table):**
```
Limit  (cost=0.56..500000.00 rows=10) (actual time=25678.901..25678.923 rows=10 loops=1)
  ->  Nested Loop  (cost=0.56..50000000.00 rows=1000)
        ->  Nested Loop  (cost=0.43..30000000.00 rows=5000)
              ->  Index Scan on orders o  (cost=0.43..800.00 rows=50)
                    Index Cond: (user_id = 12345)
                    Filter: (created_at DESC)
              ->  Seq Scan on order_items oi  (cost=0.00..600000.00 rows=100)  ← PROBLEM
                    Filter: (order_id = o.order_id)
        ->  Index Scan on products p  (cost=0.43..4.00 rows=1)
              Index Cond: (id = oi.product_id)

-- 50 orders × 600K scans per order = 30M row scans!
```

**Good Plan (Hash joins with proper indexes):**
```
Limit  (cost=100.00..150.00 rows=10) (actual time=5.123..5.234 rows=10 loops=1)
  ->  Sort  (cost=100.00..110.00 rows=50)
        Sort Key: o.created_at DESC
        ->  Hash Join  (cost=50.00..80.00 rows=50)
              Hash Cond: (oi.product_id = p.id)
              ->  Hash Join  (cost=20.00..40.00 rows=50)
                    Hash Cond: (oi.order_id = o.order_id)
                    ->  Index Scan using idx_order_items_order_id on order_items oi  (cost=0.43..15.00 rows=50)
                    ->  Hash  (cost=18.00..18.00 rows=50)
                          ->  Index Scan using idx_orders_user_id on orders o  (cost=0.43..18.00 rows=50)
                                Index Cond: (user_id = 12345)
              ->  Hash  (cost=20.00..20.00 rows=1000)
                    ->  Seq Scan on products p  (cost=0.00..20.00 rows=1000)

-- Result: 0.005 seconds (5,000× FASTER!)
```

---

## 3. Internal Working

### Nested Loop Join Internals

**Algorithm:**
```python
def nested_loop_join(outer_table, inner_table, join_condition):
    result = []
    for outer_row in outer_table:  # Outer loop
        for inner_row in inner_table:  # Inner loop (or index seek)
            if join_condition(outer_row, inner_row):
                result.append(merge(outer_row, inner_row))
    return result
```

**With Index (Index Nested Loop):**
```python
def index_nested_loop_join(outer_table, inner_table_index, join_key):
    result = []
    for outer_row in outer_table:
        # Single index lookup instead of full scan
        matching_rows = inner_table_index.seek(outer_row[join_key])
        for inner_row in matching_rows:
            result.append(merge(outer_row, inner_row))
    return result
    
# Complexity: O(N × log M) with B-Tree index
# Complexity: O(N × 1) with hash index
```

**Memory Usage:** Minimal (streaming rows)

**When Database Chooses This:**
- Outer table filtered to small result set (<1000 rows typically)
- Inner table has index on join key
- Join condition is indexed (equality, inequality)

**Example:**
```sql
-- Orders filtered to 10 rows
SELECT * FROM orders o  -- 10 rows after filter
JOIN customers c ON o.customer_id = c.id;  -- customers.id indexed

-- Nested Loop:
-- 1. Scan orders: 10 rows
-- 2. For each order: Index seek customers → 10 × 0.01ms = 0.1ms
-- Total: 0.1ms ✅
```

---

### Hash Join Internals

**Algorithm:**
```python
def hash_join(build_table, probe_table, join_key):
    # Phase 1: Build hash table (smaller table)
    hash_table = {}
    for row in build_table:
        key = row[join_key]
        if key not in hash_table:
            hash_table[key] = []
        hash_table[key].append(row)
    
    # Phase 2: Probe hash table (larger table)
    result = []
    for row in probe_table:
        key = row[join_key]
        if key in hash_table:
            for matching_row in hash_table[key]:
                result.append(merge(row, matching_row))
    
    return result
```

**Memory Usage:**
```
Memory needed = sizeof(build_table) + hash_table_overhead
                ≈ build_table_size × 1.5

Example:
- Build table: 100K rows × 200 bytes = 20MB
- Hash overhead: ~30%
- Total memory: 20MB × 1.5 = 30MB
```

**Hash Collision Handling:**
```
Hash buckets: 65536 (default PostgreSQL)
Hash function: murmurhash or cityhash (fast, good distribution)

Collision resolution: Chaining (linked list per bucket)
```

**When Database Chooses This:**
- Both tables large (>10K rows)
- Equi-join (=, IN)
- Sufficient memory for hash table (work_mem)
- No useful indexes

**Grace Hash Join (When Hash Table Doesn't Fit in Memory):**
```
1. Partition build table into N chunks (write to disk)
2. Partition probe table into same N chunks (write to disk)
3. For each partition pair:
     a. Load build partition into memory → build hash table
     b. Stream probe partition → probe hash table
     c. Output matches
4. Repeat for all partitions

Disk I/O overhead: 2 × (build_table + probe_table)
```

---

### Merge Join Internals

**Algorithm:**
```python
def merge_join(left_table_sorted, right_table_sorted, join_key):
    result = []
    left_idx = 0
    right_idx = 0
    
    while left_idx < len(left_table_sorted) and right_idx < len(right_table_sorted):
        left_row = left_table_sorted[left_idx]
        right_row = right_table_sorted[right_idx]
        
        if left_row[join_key] < right_row[join_key]:
            left_idx += 1  # Advance left
        elif left_row[join_key] > right_row[join_key]:
            right_idx += 1  # Advance right
        else:
            # Match found
            result.append(merge(left_row, right_row))
            # Handle duplicates (both sides may have multiple matching rows)
            temp_left = left_idx
            while temp_left < len(left_table_sorted) and left_table_sorted[temp_left][join_key] == left_row[join_key]:
                temp_right = right_idx
                while temp_right < len(right_table_sorted) and right_table_sorted[temp_right][join_key] == right_row[join_key]:
                    result.append(merge(left_table_sorted[temp_left], right_table_sorted[temp_right]))
                    temp_right += 1
                temp_left += 1
            left_idx = temp_left
            right_idx = temp_right
    
    return result
```

**When Database Chooses This:**
- Both tables already sorted on join key (or have suitable indexes)
- Non-equi joins (>, <, BETWEEN)
- Merge join avoids sorting cost

**Cost Breakdown:**
```
If tables presorted (indexes exist):
  Cost = O(N + M)  ← Linear scan, very fast
  
If tables need sorting:
  Cost = O(N log N + M log M + N + M)
       = Sort cost + Merge cost
  Often slower than hash join unless output needs sorting anyway
```

**Example:**
```sql
-- Both tables sorted by timestamp
SELECT * FROM events_2024 e1
JOIN events_2023 e2 ON e1.timestamp = e2.timestamp
WHERE e1.timestamp BETWEEN '2023-12-31' AND '2024-01-02';

-- Merge Join ideal:
-- 1. Index scan events_2024 (sorted by timestamp)
-- 2. Index scan events_2023 (sorted by timestamp)
-- 3. Merge → O(N + M) linear
```

---

## 4. Best Practices

### Practice 1: Ensure Smaller Table is Build Side (Hash Join)

**❌ Bad: Large Table as Build Side**
```sql
-- Optimizer builds hash table from orders (10M rows)
-- Probes with customers (1K rows)

Hash Join  (cost=500000.00..600000.00 rows=1000)
  Hash Cond: (c.id = o.customer_id)
  ->  Seq Scan on customers c  (cost=0.00..50.00 rows=1000)  ← Probe (1K rows)
  ->  Hash  (cost=200000.00..200000.00 rows=10000000)  ← Build (10M rows, 2GB!)
        ->  Seq Scan on orders o  (cost=0.00..200000.00 rows=10000000)

-- Hash table: 10M × 200 bytes = 2GB
-- Spills to disk → catastrophically slow
```

**✅ Good: Small Table as Build Side**
```sql
-- Swap join order (hint or rewrite)
SELECT /*+ LEADING(c o) */ * 
FROM customers c
JOIN orders o ON c.id = o.customer_id;

Hash Join  (cost=5000.00..25000.00 rows=1000)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..20000.00 rows=10000000)  ← Probe (10M rows)
  ->  Hash  (cost=50.00..50.00 rows=1000)  ← Build (1K rows, 200KB)
        ->  Seq Scan on customers c  (cost=0.00..50.00 rows=1000)

-- Hash table: 1K × 200 bytes = 200KB (fits in memory!)
```

### Practice 2: Add Index for Nested Loop Inner Table

**❌ Bad: No Index on Join Key**
```sql
-- Nested loop without index
Nested Loop  (cost=0.00..5000000000.00 rows=100000)
  ->  Seq Scan on orders o  (cost=0.00..50000.00 rows=100000)
  ->  Seq Scan on customers c  (cost=0.00..50000.00 rows=100000)  ← Full scan 100K times!
        Filter: (id = o.customer_id)

-- Total: 100K orders × 100K customers scanned = 10 billion comparisons
```

**✅ Good: Index on Inner Table**
```sql
CREATE INDEX idx_customers_id ON customers(id);  -- Primary key if not exists

Nested Loop  (cost=0.43..5000.00 rows=100000)
  ->  Seq Scan on orders o  (cost=0.00..2000.00 rows=100000)
  ->  Index Scan using idx_customers_id on customers c  (cost=0.43..0.03 rows=1)
        Index Cond: (id = o.customer_id)

-- Total: 100K orders × 0.03ms index seek = 3 seconds (vs 1000+ seconds before)
```

### Practice 3: Increase work_mem for Hash Joins

**❌ Bad: Insufficient Memory (Hash Spills to Disk)**
```sql
-- work_mem = 4MB (default)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id;

Hash Join  (cost=5000.00..25000.00 rows=1000000)
  Hash Cond: (o.customer_id = c.id)
  Buffers: shared hit=5000, temp read=120000 written=120000  ← DISK SPILL!
  ->  Seq Scan on orders o
  ->  Hash  (cost=2500.00..2500.00 rows=100000)
        Buckets: 131072  Batches: 8  ← 8 batches = wrote to disk 8 times
        ->  Seq Scan on customers c

-- Execution Time: 15000 ms
```

**✅ Good: Adequate Memory**
```sql
SET work_mem = '128MB';  -- Enough for hash table

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id;

Hash Join  (cost=5000.00..15000.00 rows=1000000)
  Hash Cond: (o.customer_id = c.id)
  Buffers: shared hit=5000  ← No disk spill
  ->  Seq Scan on orders o
  ->  Hash  (cost=2500.00..2500.00 rows=100000)
        Buckets: 131072  Batches: 1  ← Single batch (in-memory)
        ->  Seq Scan on customers c

-- Execution Time: 850 ms (18× FASTER!)
```

**Calculating Needed work_mem:**
```
work_mem needed = (rows × avg_row_size × 1.5) / (1024 × 1024)

Example:
- Hash table: 100K rows × 200 bytes × 1.5 (overhead) = 30MB
- Set work_mem = 64MB (leave headroom)
```

### Practice 4: Use Merge Join for Sorted Data

**❌ Bad: Hash Join on Already-Sorted Data**
```sql
-- Query joining on timestamp (both tables have indexes on timestamp)
SELECT * FROM events_2024 e1
JOIN events_2023 e2 ON e1.user_id = e2.user_id
WHERE e1.timestamp >= '2024-01-01';

-- Hash Join chosen:
Hash Join  (cost=50000.00..100000.00 rows=500000)
  Hash Cond: (e1.user_id = e2.user_id)
  ->  Index Scan on events_2024 e1 (sorted by timestamp)
  ->  Hash
        ->  Seq Scan on events_2023 e2  -- Ignores existing sort order

-- Wasted opportunity: e1 already sorted by index
```

**✅ Good: Merge Join Exploits Sort Order**
```sql
-- Hint to prefer merge join
SET enable_hashjoin = off;  -- Testing only

Merge Join  (cost=10000.00..50000.00 rows=500000)
  Merge Cond: (e1.user_id = e2.user_id)
  ->  Index Scan using idx_events_2024_user_id on events_2024 e1
  ->  Index Scan using idx_events_2023_user_id on events_2023 e2

-- No sorting needed, linear merge: 50% faster
```

---

## 5. Common Mistakes

### Mistake 1: Nested Loop on Large Cartesian Product

**Problem:**
```sql
-- Missing join condition!
SELECT * FROM orders o, customers c
WHERE o.created_at >= '2024-01-01'
  AND c.country = 'US';
  -- Missing: AND o.customer_id = c.id

-- Plan:
Nested Loop  (cost=0.00..5000000000000.00 rows=100000000000)  ← 100 BILLION rows!
  ->  Seq Scan on orders o  (cost=0.00..50000.00 rows=1000000)
  ->  Materialize  (cost=0.00..2500.00 rows=100000)
        ->  Seq Scan on customers c
              Filter: (country = 'US')

-- 1M orders × 100K customers = 100 billion combinations
-- Query runs for hours...
```

**Solution:**
```sql
-- Add missing join condition
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id  ← CRITICAL!
WHERE o.created_at >= '2024-01-01'
  AND c.country = 'US';

-- Now:
Hash Join  (cost=5000.00..15000.00 rows=50000)
-- Runtime: 0.8 seconds
```

### Mistake 2: Using Nested Loop with No Index

**Problem:**
```sql
-- Foreign key exists but no index on order_items.order_id
Nested Loop  (cost=0.00..5000000000.00 rows=100000)
  ->  Seq Scan on orders o  (cost=0.00..5000.00 rows=100)
  ->  Seq Scan on order_items oi  (cost=0.00..50000000.00 rows=1000)
        Filter: (order_id = o.order_id)  ← Sequential scan for EACH order!

-- 100 orders × 5M order_items scanned = 500M scans
```

**Solution:**
```sql
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Now:
Nested Loop  (cost=0.43..500.00 rows=100)
  ->  Seq Scan on orders o  (cost=0.00..50.00 rows=100)
  ->  Index Scan using idx_order_items_order_id on order_items oi  (cost=0.43..4.50 rows=10)
        Index Cond: (order_id = o.order_id)

-- 100 orders × 0.045ms index seek = 4.5ms (1,000,000× FASTER!)
```

### Mistake 3: Forcing Wrong Join Algorithm

**Problem:**
```sql
-- Developer thinks hash join is always best
SET enable_nestloop = off;
SET enable_mergejoin = off;

SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.order_id = 12345;  -- Single order

-- Now forced to use hash join:
Hash Join  (cost=5000.00..8000.00 rows=1)
  ->  Index Scan on orders o  (cost=0.43..8.45 rows=1)  ← 1 row
  ->  Hash  (cost=2500.00..2500.00 rows=100000)
        ->  Seq Scan on customers c  ← Builds hash table of 100K rows for 1 join!

-- Execution Time: 500ms
```

**Solution:**
```sql
RESET enable_nestloop;  -- Allow optimizer to choose

-- Now correctly chooses nested loop:
Nested Loop  (cost=0.86..16.90 rows=1)
  ->  Index Scan on orders o  (cost=0.43..8.45 rows=1)
        Index Cond: (order_id = 12345)
  ->  Index Scan on customers c  (cost=0.43..8.45 rows=1)
        Index Cond: (id = o.customer_id)

-- Execution Time: 0.05ms (10,000× FASTER!)
```

**Takeaway:** Only disable join algorithms temporarily for debugging. Let optimizer choose in production.

### Mistake 4: Not Monitoring Hash Join Batch Count

**Problem:**
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Output shows:
Hash  (cost=50000.00..50000.00 rows=2000000)
  Buckets: 262144  Batches: 16  ← RED FLAG: 16 batches!
  Memory Usage: 65536kB
  Disk Usage: 524288kB  ← Spilled to disk!
  
-- Each batch requires:
--   1. Write hash table partition to disk
--   2. Write probe table partition to disk
--   3. Read both back for join
-- Total I/O: 3 × (hash_table_size + probe_table_size) × batches
```

**Solution:**
```sql
-- Check current work_mem
SHOW work_mem;  -- 4MB

-- Calculate needed:
-- 2M rows × 256 bytes × 1.5 = 768MB

-- Increase (session-level for this query only)
SET LOCAL work_mem = '1GB';

-- Re-run:
Hash  (cost=50000.00..50000.00 rows=2000000)
  Buckets: 262144  Batches: 1  ← Fixed!
  Memory Usage: 786432kB

-- Execution Time: 3500ms → 450ms (8× faster)
```

---

## 6. Security Considerations

### 1. Resource Exhaustion via Cartesian Products

**Risk:**
```sql
-- Attacker crafts query missing join condition
SELECT * FROM table_a, table_b;  -- 1M × 1M = 1 trillion rows

-- Database attempts nested loop:
-- - Consumes 100% CPU for hours
-- - Fills disk with temp files (if spilling)
-- - Blocks other queries (resource contention)
```

**Mitigation:**
```sql
-- Set statement timeout
ALTER ROLE app_user SET statement_timeout = '30s';

-- Limit work_mem per query
ALTER ROLE app_user SET work_mem = '256MB';

-- Monitor for large nested loops:
SELECT pid, query, state, now() - query_start as runtime
FROM pg_stat_activity
WHERE query LIKE '%Nested Loop%'
  AND now() - query_start > interval '10 seconds';
```

---

## 7. Performance Optimization

### Optimization 1: Parallel Hash Join

**PostgreSQL 11+ Feature:**
```sql
-- Enable parallel hash join
SET max_parallel_workers_per_gather = 4;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Without parallel:
Hash Join  (cost=50000.00..100000.00 rows=5000000) (actual time=5678.901..9876.543 rows=5000000)

-- With parallel:
Gather  (cost=10000.00..50000.00 rows=5000000) (actual time=1234.567..2345.678 rows=5000000)
  Workers Planned: 4
  Workers Launched: 4
  ->  Parallel Hash Join  (cost=9000.00..40000.00 rows=1250000)
        Hash Cond: (o.customer_id = c.id)
        ->  Parallel Seq Scan on orders o  (cost=0.00..20000.00 rows=1250000)
        ->  Parallel Hash  (cost=5000.00..5000.00 rows=25000)
              Buckets: 65536  Batches: 1
              ->  Parallel Seq Scan on customers c  (cost=0.00..5000.00 rows=25000)

-- Execution Time: 2345ms → 4× speedup with 4 workers
```

### Optimization 2: Bloom Filters for Distributed Joins

**Concept:** Pre-filter probe table using bloom filter from build table (in distributed databases).

```
Database 1: orders shard
Database 2: customers shard

Without bloom filter:
  1. Hash customers → send hash table to orders node
  2. Orders node probes hash table
  3. Network transfer: ALL customer data

With bloom filter:
  1. Build bloom filter from customers (small, ~1MB)
  2. Send bloom filter to orders node
  3. Orders node filters ~90% of rows with bloom filter
  4. Send only matching 10% of orders back
  5. Final join on coordinator
  
Network reduction: 90% fewer bytes transferred
```

---

## 8. Examples

### Example 1: Optimizing 3-Table Join

**Scenario:** Dashboard showing orders with customer and product info

**Original Query (Slow: 25 seconds)**
```sql
SELECT o.order_id, c.customer_name, p.product_name, oi.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at >= '2024-01-01';
```

**Plan Analysis:**
```
Nested Loop  (cost=0.56..50000000.00 rows=500000) (actual time=0.123..25678.901 rows=500000)
  ->  Nested Loop  (cost=0.43..30000000.00 rows=500000)
        ->  Nested Loop  (cost=0.43..10000000.00 rows=100000)
              ->  Seq Scan on orders o  (cost=0.00..50000.00 rows=100000)
                    Filter: (created_at >= '2024-01-01')
              ->  Index Scan on customers c  (cost=0.43..100.00 rows=1)
                    Index Cond: (id = o.customer_id)
        ->  Seq Scan on order_items oi  (cost=0.00..300.00 rows=5)  ← NO INDEX!
              Filter: (order_id = o.order_id)
  ->  Index Scan on products p  (cost=0.43..4.00 rows=1)
        Index Cond: (id = oi.product_id)

-- Problem: Nested loop on order_items without index
-- 100K orders × 300ms per sequential scan = 30,000 seconds!
```

**Fix 1: Add Missing Index**
```sql
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
ANALYZE order_items;
```

**New Plan (Better, but still slow: 5 seconds)**
```
Hash Join  (cost=15000.00..50000.00 rows=500000) (actual time=1500.123..5000.456 rows=500000)
  Hash Cond: (oi.product_id = p.id)
  ->  Hash Join  (cost=8000.00..30000.00 rows=500000)
        Hash Cond: (oi.order_id = o.order_id)
        ->  Seq Scan on order_items oi  (cost=0.00..10000.00 rows=2000000)
        ->  Hash  (cost=6000.00..6000.00 rows=100000)
              ->  Hash Join  (cost=3000.00..6000.00 rows=100000)
                    Hash Cond: (o.customer_id = c.id)
                    ->  Seq Scan on orders o  (cost=0.00..2000.00 rows=100000)
                          Filter: (created_at >= '2024-01-01')
                    ->  Hash  (cost=1500.00..1500.00 rows=100000)
                          ->  Seq Scan on customers c
  ->  Hash  (cost=4000.00..4000.00 rows=200000)
        ->  Seq Scan on products p

-- Better! But still scans 2M order_items rows
```

**Fix 2: Covering Index to Eliminate Seq Scan**
```sql
CREATE INDEX idx_order_items_covering ON order_items(order_id, product_id, quantity);
```

**Final Plan (Fast: 0.5 seconds)**
```
Hash Join  (cost=15000.00..25000.00 rows=500000) (actual time=300.123..500.456 rows=500000)
  Hash Cond: (oi.product_id = p.id)
  ->  Hash Join  (cost=8000.00..15000.00 rows=500000)
        Hash Cond: (oi.order_id = o.order_id)
        ->  Index Only Scan on idx_order_items_covering oi  ← INDEX ONLY SCAN!
              (cost=0.43..5000.00 rows=2000000)
        ->  Hash  (cost=6000.00..6000.00 rows=100000)
              ->  Hash Join  (cost=3000.00..6000.00 rows=100000)
                    ->  Bitmap Heap Scan on orders o
                          Recheck Cond: (created_at >= '2024-01-01')
                          ->  Bitmap Index Scan on idx_orders_created
                    ->  Hash
                          ->  Seq Scan on customers c
  ->  Hash
        ->  Seq Scan on products p

-- Result: 25s → 0.5s (50× FASTER!)
```

---

## 9. Real-World Use Cases

### Use Case 1: Time-Series Data Merge Join

**Scenario:** Correlating temperature sensor data with equipment failures

```sql
-- Both tables have 10M rows, sorted by timestamp
SELECT s.sensor_id, s.temperature, e.event_type
FROM sensor_readings s
JOIN equipment_events e 
  ON s.equipment_id = e.equipment_id
  AND s.timestamp BETWEEN e.timestamp - INTERVAL '1 minute' AND e.timestamp;
```

**Why Merge Join is Perfect:**
- Both tables naturally sorted by timestamp (time-series insert order)
- Range condition (BETWEEN) instead of equality
- Merge join exploits sort order without additional sorting

**Plan:**
```
Merge Join (cost=1000.00..50000.00 rows=100000) (actual time=50.123..850.456 rows=95000)
  Merge Cond: (s.equipment_id = e.equipment_id AND s.timestamp BETWEEN ...)
  ->  Index Scan using idx_sensor_readings_ts on sensor_readings s
  ->  Index Scan using idx_equipment_events_ts on equipment_events e

-- Linear scan with early termination: 0.85 seconds ✅
```

---

### Use Case 2: Star Schema with Dimension Tables (Hash Joins)

**Scenario:** Data warehouse query joining fact table with multiple dimension tables

```sql
SELECT
    d.date,
    c.customer_segment,
    p.product_category,
    SUM(f.sales_amount) as total_sales
FROM fact_sales f
JOIN dim_date d ON f.date_

_key = d.date_key
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_product p ON f.product_key = p.product_key
WHERE d.year = 2024
GROUP BY d.date, c.customer_segment, p.product_category;
```

**Why Hash Joins Are Perfect:**
- Dimension tables small (1K-100K rows), fit in memory
- Fact table large (100M rows), streamed once
- Multiple equi-joins (=)

**Execution Strategy:**
```
1. Build hash tables for dimensions (parallel):
   - dim_date: 365 rows → 50KB hash table
   - dim_customer: 10K rows → 2MB hash table
   - dim_product: 5K rows → 1MB hash table
   
2. Stream fact_sales (100M rows), probe all 3 hash tables simultaneously
   - Each fact row: 3 hash lookups (~0.001ms each)
   - Total: 100M × 0.003ms = 300 seconds... but with parallel scan:
   
3. Parallel scan with 8 workers:
   - Each worker processes 12.5M rows
   - 300s / 8 = 37.5 seconds ✅
```

---

## 10. Interview Questions

### Q1: When would you choose Nested Loop over Hash Join?

**Senior Answer:**

```
Choose Nested Loop when:

1. **Small outer table with index on inner table**
   - Example: 10 orders joining 1M customers
   - Nested Loop: 10 × 0.01ms (index seek) = 0.1ms
   - Hash Join: Build hash table (500ms) + probe (50ms) = 550ms
   - Nested Loop 5500× faster!

2. **LIMIT query (early termination)**
   ```sql
   SELECT * FROM orders o
   JOIN customers c ON o.customer_id = c.id
   ORDER BY o.created_at DESC
   LIMIT 10;
   ```
   - Nested Loop stops after 10 matches
   - Hash Join must build entire hash table first (can't short-circuit)

3. **Index supports sort order**
   - Query needs ORDER BY on join key
   - Nested Loop with index scan returns pre-sorted rows
   - Hash Join requires additional sort step

4. **Low memory environment**
   - work_mem insufficient for hash table
   - Nested Loop uses minimal memory
   - Hash Join would spill to disk (very slow)

Never choose Nested Loop when:
- Both tables large (>10K rows each)
- No index on inner table (becomes O(N×M) cartesian product)
- Optimizer already chose hash join (trust it unless proven wrong)
```

### Q2: Explain how hash join handles hash collisions. What happens if there are many collisions?

**Staff Answer:**

```
Hash Join Collision Handling:

**Algorithm:**
```python
hash_table = {}  # Buckets (default 65536 in PostgreSQL)

# Build phase
for row in build_table:
    bucket = hash(row.join_key) % num_buckets
    if bucket not in hash_table:
        hash_table[bucket] = []
    hash_table[bucket].append(row)  # Chaining (linked list)

# Probe phase
for row in probe_table:
    bucket = hash(row.join_key) % num_buckets
    if bucket in hash_table:
        for candidate in hash_table[bucket]:  # Linear search within bucket
            if candidate.join_key == row.join_key:  # Exact match check
                output(merge(row, candidate))
```

**Collision Impact:**

1. **Good distribution (few collisions):**
   - Average bucket size: build_rows / num_buckets
   - Example: 1M rows / 65K buckets = 15 rows per bucket
   - Probe cost: O(1) hash + O(15) linear search ≈ O(1) ✅

2. **Poor distribution (many collisions):**
   - Skewed data: 90% of joins to same key (e.g., deleted_customer_id = -1)
   - One bucket has 900K rows, others have ~2 rows each
   - Probe cost: O(1) hash + O(900K) linear search = O(N) ❌
   - Effectively degrades to nested loop!

**Example:**
```sql
-- Data skew: 90% of orders have customer_id = -1 (deleted accounts)
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Plan shows:
Hash Join  (cost=5000.00..500000.00 rows=1000000) (actual time=50.123..45678.901 rows=1000000)
                                                                    ^^^^^^^^^^^^
                                                                    Much slower than estimated!
  Buffers: shared hit=100000, temp read=0 written=0
  
-- Analysis:
-- - Hash function distributes evenly across buckets
-- - But 900K orders hash to SAME customer_id (-1)  
-- - That bucket's linked list has 900K entries
-- - Each probe does linear search: 900K comparisons

-- Fix:
-- 1. Filter outliers:
WHERE o.customer_id > 0  -- Exclude deleted accounts

-- 2. Or handle separately:
(SELECT * FROM orders WHERE customer_id > 0 JOIN customers ...)
UNION ALL  
(SELECT * FROM orders WHERE customer_id = -1 JOIN deleted_customers ...)
```

**PostgreSQL Mitigations:**
- Dynamic bucket resizing (increase buckets if needed)
- Skew-aware hash functions (detect outliers, use different strategy)
- Hybrid hash join (switch to sort-merge for large buckets)
```

### Q3: Design the optimal join strategy for a 5-table query.

**Principal Answer:**

```
5-Table Join Optimization Framework:

**Given Query:**
```sql
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.id
JOIN categories cat ON p.category_id = cat.id
WHERE o.created_at >= '2024-01-01'
  AND c.country = 'US'
  AND cat.name = 'Electronics';
```

**Step 1: Analyze Table Sizes and Filters**
```sql
-- Check cardinalities:
SELECT 'orders' as tbl, COUNT(*) as rows FROM orders WHERE created_at >= '2024-01-01';
-- Result: 500K rows

SELECT 'customers' as tbl, COUNT(*) as rows FROM customers WHERE country = 'US';
-- Result: 200K rows

SELECT 'categories' as tbl, COUNT(*) as rows FROM categories WHERE name = 'Electronics';
-- Result: 1 row

-- Full table sizes:
-- orders: 10M | customers: 5M | order_items: 50M | products: 100K | categories: 100
```

**Step 2: Determine Join Order (Smallest Filtered First)**
```
Priority order:
1. categories (1 row) ← Start here
2. products (filtered by category: ~5K rows)
3. order_items (filtered by products: ~50K rows)
4. orders (filtered by date: 500K rows)
5. customers (filtered by country: 200K rows)

Join tree:
categories (1) 
  → products (5K)
     → order_items (50K)
        → orders (500K)
           → customers (200K)
```

**Step 3: Choose Join Algorithms**
```sql
-- Rewritten query with join strategy:

WITH electronics_products AS (
    -- Nested Loop (1 row → 5K rows)
    SELECT p.id, p.product_name
    FROM categories cat
    JOIN products p ON cat.id = p.category_id
    WHERE cat.name = 'Electronics'
),
electronics_orders AS (
    -- Hash Join (5K products, 50K order_items after filter)
    SELECT DISTINCT oi.order_id
    FROM electronics_products ep
    JOIN order_items oi ON ep.id = oi.product_id
),
recent_electronics_orders AS (
    -- Hash Join (50K filtered orders, 500K recent orders)
    SELECT o.order_id, o.customer_id, o.total
    FROM orders o
    WHERE o.created_at >= '2024-01-01'
      AND o.order_id IN (SELECT order_id FROM electronics_orders)
)
-- Final join
SELECT reo.order_id, c.customer_name, reo.total
FROM recent_electronics_orders reo
JOIN customers c ON reo.customer_id = c.id  -- Hash Join (50K orders, 200K US customers)
WHERE c.country = 'US';
```

**Step 4: Verify with EXPLAIN ANALYZE**
```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING)
[rewritten query];

-- Check:
-- 1. Are all CTEs using expected join algorithms?
-- 2. Row count estimates close to actuals?
-- 3. Any hash joins spilling to disk? (Batches: > 1)
-- 4. Total execution time acceptable?
```

**Step 5: Add Indexes as Needed**
```sql
-- From EXPLAIN ANALYZE, if seeing sequential scans:
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);
CREATE INDEX idx_orders_created ON orders(created_at) WHERE created_at >= '2024-01-01';
CREATE INDEX idx_customers_country ON customers(country) WHERE country = 'US';

-- Covering indexes for hot paths:
CREATE INDEX idx_order_items_covering ON order_items(product_id, order_id);
```

**Expected Performance:**
- Before optimization: 60+ seconds (cartesian products, wrong join order)
- After optimization: 1-2 seconds (selective filters applied early, right join algorithms)
```

---

## 11. Summary

### Join Algorithm Decision Matrix

| Scenario | Outer Table | Inner Table | Join Type | Best Algorithm | Why |
|----------|-------------|-------------|-----------|----------------|-----|
| Small × Large with index | <1K rows | Any size | Equi-join | **Nested Loop** | Index seek: O(N×log M) |
| Large × Large | >10K rows | >10K rows | Equi-join | **Hash Join** | O(N+M) linear, memory efficient |
| Sorted data | Any | Any | Any | **Merge Join** | Exploits existing sort order |
| LIMIT query | Any | Any | Any | **Nested Loop** | Early termination |
| Low memory | Any | Any | Equi-join | **Nested Loop** (indexed) | Minimal memory use |
| Range join | Any | Any | >, <, BETWEEN | **Merge Join** | Handles non-equality efficiently |

### Key Takeaways

1. **Nested Loop:**
   - Pros: Fast for small outer table, supports early termination, minimal memory
   - Cons: O(N×M) without index, catastrophic for large tables
   - When: Outer <1K rows, inner indexed, or LIMIT queries

2. **Hash Join:**
   - Pros: O(N+M) linear, handles large tables well
   - Cons: Requires memory, spills to disk if insufficient, no early termination
   - When: Large tables, equi-joins, sufficient memory

3. **Merge Join:**
   - Pros: O(N+M) for presorted data, handles range joins
   - Cons: Requires sorting if not presorted, O(N log N) sort cost
   - When: Data already sorted (indexes), range conditions, output needs sorting

### Optimization Checklist

- [ ] Verify indexes exist on join keys (for nested loop inner tables)
- [ ] Check hash join batch count (Batches: 1 = good, >1 = spilling to disk)
- [ ] Ensure smaller table is build side (hash joins)
- [ ] Monitor for cartesian products (missing join conditions)
- [ ] Tune work_mem for hash joins (avoid disk spills)
- [ ] Consider parallel joins for large datasets (PostgreSQL 11+)
- [ ] Rewrite 5+ table joins as CTEs for clearer execution plans
- [ ] Use EXPLAIN (ANALYZE, BUFFERS) to validate join strategy
- [ ] Watch for nested loops on large tables (should be hash/merge)
- [ ] Only disable join algorithms temporarily for debugging

**Remember:** The database optimizer is smart, but it's only as good as your statistics and indexes. Keep statistics fresh (ANALYZE), add appropriate indexes, and the optimizer will choose the right join algorithm 95% of the time.

---

**Next Reading:**
- `05_Index_Selection.md` - How optimizer chooses which index to use
- `06_Query_Hints.md` - When and how to override optimizer decisions
- `07_Statistics_Management.md` - Keeping cardinality estimates accurate
