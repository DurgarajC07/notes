# Composite Indexes - Multi-Column Index Design

## 1. Concept Explanation

**Composite indexes** (also called **multi-column indexes** or **concatenated indexes**) index two or more columns together, creating a sorted structure that enables efficient queries filtering on combinations of those columns. Column order is critical: the index is sorted first by column 1, then by column 2 within each column 1 value, and so on.

Think of a composite index like a phone book:
- **Single-column index:** Sorted by last name only (can't efficiently find "all Smiths in Boston")
- **Composite index:** Sorted by (state, city, last_name) → Can efficiently find "all Smiths in Boston, MA"

### Structure Example

```
Table: employees (1M rows)
Columns: department, hire_date, salary

Composite index: idx_dept_date ON (department, hire_date)

Physical structure (simplified):
['Engineering', '2024-01-15', RID: page 1, offset 5]
['Engineering', '2024-01-20', RID: page 1, offset 8]
['Engineering', '2024-02-01', RID: page 3, offset 2]
['Sales', '2024-01-10', RID: page 2, offset 1]
['Sales', '2024-01-25', RID: page 4, offset 7]
['Sales', '2024-02-15', RID: page 5, offset 3]

Sorted by department first, then hire_date within each department.

Query A: WHERE department = 'Engineering'
  → Navigate to 'Engineering' in B-Tree → Scan all 'Engineering' entries ✅

Query B: WHERE department = 'Engineering' AND hire_date > '2024-02-01'
  → Navigate to ('Engineering', '2024-02-01') → Scan forward ✅

Query C: WHERE hire_date > '2024-02-01'
  → No efficient access! Must scan all departments, checking hire_date ❌
  → hire_date not first column → Index not sorted by hire_date globally
```

**Key Properties:**
- **Leftmost prefix rule:** Index usable for queries filtering on (col1), (col1, col2), (col1, col2, col3), but NOT (col2), (col3), (col2, col3)
- **Sort order:** Sorted by col1, then col2, then col3 (nested sorting)
- **Selectivity:** Most selective column should usually be first (more on this below)
- **Storage:** Single index covering multiple columns (vs separate indexes per column)

---

## 2. Why It Matters

### Production Impact: Multi-Column Query Performance

**Scenario: User search by city + age range**

```sql
-- Table: users (100M rows)
SELECT user_id, name, email
FROM users
WHERE city = 'San Francisco'
  AND age BETWEEN 25 AND 35;

-- Distribution:
-- city='San Francisco': 2M users (2% of data)
-- age BETWEEN 25 AND 35: 20M users (20% of data)
-- Intersection (city + age): 400K users (0.4% of data)
```

**Option 1: No Index (Seq Scan)**
```sql
-- Plan: Seq Scan on users
-- Scan: 100M rows, filter city + age
-- I/O: 100M rows × 8KB/1000 rows = 800K pages
-- Time: 60 seconds ❌
```

**Option 2: Separate Indexes**
```sql
CREATE INDEX idx_city ON users(city);
CREATE INDEX idx_age ON users(age);

-- Plan: Bitmap Index Scan (if supported)
-- 1. Scan idx_city → 2M matching rows (bitmap A)
-- 2. Scan idx_age → 20M matching rows (bitmap B)
-- 3. Bitmap AND: A ∩ B = 400K rows
-- 4. Fetch 400K heap tuples (random I/O!)

-- Cost:
-- - Index scans: 2M + 20M lookups = 22M operations
-- - Heap fetches: 400K random page reads
-- Time: 15 seconds (still slow) ⚠️

-- Problem: Heap fetches are random I/O (not sequential)
```

**Option 3: Composite Index**
```sql
CREATE INDEX idx_city_age ON users(city, age);

-- Plan: Index Scan using idx_city_age
-- 1. Navigate to ('San Francisco', 25) in B-Tree
-- 2. Scan forward until ('San Francisco', 36)
-- 3. Fetch 400K heap tuples (but in sort order → Likely clustered!)

-- Cost:
-- - Index navigation: log(100M) ≈ 27 comparisons
-- - Index scan: 400K entries (sequential read within index)
-- - Heap fetches: 400K rows (if clustered, ~sequential)

-- Time: 0.5 seconds (120× FASTER than seq scan!) ✅

-- Savings:
-- - I/O: 800K pages → 400K pages (50% reduction)
-- - Random reads: 0 → Mostly sequential (if data clustered)
-- - CPU: 100M row filters → 400K row fetches
```

### Real Numbers: E-Commerce Product Search

**Scenario: 10M products, search by (category, price_range)**

| Approach | Index Setup | Query Time | Index Size | Coverage |
|----------|-------------|------------|------------|----------|
| No index | - | 8 seconds | 0MB | - |
| Single idx_category | (category) | 2 seconds | 200MB | Only category queries |
| Separate indexes | (category), (price) | 1.5 seconds | 400MB | Bitmap AND overhead |
| Composite index | (category, price) | 0.05 seconds | 250MB | Both columns |
| Covering composite | (category, price) INCLUDE (name, image_url) | 0.02 seconds | 500MB | Index-only scan |

**Winner: Composite index (category, price)**
- 40× faster than separate indexes (1.5s → 0.05s)
- Smaller than separate indexes (250MB vs 400MB)
- Covers both columns efficiently

---

## 3. Internal Working

### Leftmost Prefix Rule

The **leftmost prefix rule** determines which queries can use a composite index.

**Index: idx_abc ON (col_a, col_b, col_c)**

| Query WHERE Clause | Index Usable? | Reason |
|-------------------|---------------|--------|
| `col_a = 5` | ✅ Yes | Uses leftmost prefix (col_a) |
| `col_a = 5 AND col_b = 10` | ✅ Yes | Uses (col_a, col_b) prefix |
| `col_a = 5 AND col_b = 10 AND col_c = 15` | ✅ Yes | Uses full index (col_a, col_b, col_c) |
| `col_b = 10` | ❌ No | Missing col_a (not leftmost) |
| `col_c = 15` | ❌ No | Missing col_a, col_b |
| `col_b = 10 AND col_c = 15` | ❌ No | Missing col_a |
| `col_a = 5 AND col_c = 15` | ⚠️ Partial | Uses col_a, scans + filters col_c |

**Partial Usage Explained:**

```sql
-- Index: idx_abc ON (col_a, col_b, col_c)
-- Query: WHERE col_a = 5 AND col_c = 15

-- Plan:
Index Scan using idx_abc
  Index Cond: (col_a = 5)  ← Uses index for col_a
  Filter: (col_c = 15)     ← Scans col_a=5 entries, filtering col_c

-- Process:
-- 1. Navigate to col_a = 5 in B-Tree (efficient)
-- 2. Scan all (5, *, *) entries where col_a = 5
-- 3. For each entry, check if col_c = 15 (post-filter, not index seek)

-- Performance:
-- If col_a = 5 matches 1000 rows, scans all 1000, filters to 10 matching col_c = 15
-- Better than seq scan (1M rows → 1000 rows), but not as efficient as full index usage
```

### Sort Order and Nested Sorting

**Index: idx_dept_date_salary ON (department, hire_date, salary)**

```
Index entries (sorted):

department    hire_date    salary    RID
---------------------------------------------
Engineering   2024-01-10   80000     (1,1)
Engineering   2024-01-10   85000     (1,2)  ← Same dept & date, sorted by salary
Engineering   2024-01-15   90000     (2,3)
Engineering   2024-02-01   75000     (3,4)
Sales         2024-01-05   60000     (4,5)
Sales         2024-01-20   65000     (5,6)
Sales         2024-02-10   70000     (6,7)

Sorting:
1. Primary sort: department (Engineering < Sales alphabetically)
2. Secondary sort: Within each department, sorted by hire_date
3. Tertiary sort: Within each (department, hire_date), sorted by salary
```

**Query Optimization with Sort:**

```sql
-- Query: Top 10 highest salaries in Engineering, hired after 2024-01-15
SELECT * FROM employees
WHERE department = 'Engineering'
  AND hire_date > '2024-01-15'
ORDER BY salary DESC
LIMIT 10;

-- Plan with composite index:
Index Scan Backward using idx_dept_date_salary  ← Scan in reverse!
  Index Cond: (department = 'Engineering' AND hire_date > '2024-01-15')
  Limit: 10

-- Process:
-- 1. Navigate to ('Engineering', '2024-01-15', +∞) in B-Tree
-- 2. Scan backward (descending salary within matching date range)
-- 3. Stop after 10 rows

-- Cost: log(N) + 10 rows
-- No explicit sort phase needed! ✅ (Index provides sort order)
```

### Column Order: Selectivity vs Query Pattern

**Selectivity Rule: Most selective column first (usually)**

```
Table: orders (10M rows)
Columns:
  - status: 5 values (pending, processing, shipped, delivered, cancelled)
    → Selectivity: 1/5 = 20% (low)
  - user_id: 1M values (1M unique users)
    → Selectivity: 1/1M = 0.0001% (high)

Query: WHERE user_id = 12345 AND status = 'pending'
```

**Option A: idx_userid_status ON (user_id, status)**
```
Index structure: Sorted by user_id, then status

Query plan:
1. Navigate to user_id = 12345 in B-Tree → O(log N) = ~23 comparisons
2. Scan all entries for user_id = 12345 (avg 10 orders per user)
3. Filter status = 'pending' (typically 1 pending order)

Cost: log(10M) + 10 = 23 + 10 = 33 operations
Result: 1 matching order
```

**Option B: idx_status_userid ON (status, user_id)**
```
Index structure: Sorted by status, then user_id

Query plan:
1. Navigate to status = 'pending' in B-Tree → O(log N) = ~23 comparisons
2. Scan 'pending' entries (2M rows, 20% of table!)
3. Filter user_id = 12345 within 'pending' slice

Cost: log(10M) + 2M = 23 + 2,000,000 operations ❌
Result: 1 matching order (same result, WAY slower!)
```

**Winner: Option A (user_id first)**
- High-selectivity column first narrows search space immediately
- Rule: **Most selective column → Least selective column**

**Exception: Query Pattern Frequency**

```
Queries:
- 95% of queries: WHERE status = 'pending' (no user_id filter)
- 5% of queries: WHERE user_id = X AND status = 'pending'

If most queries only filter status, idx_status_userid is better!
- 95% queries: Navigate to status directly ✅
- 5% queries: Acceptable slower (still uses index partially)

Decision: **Query pattern overrides selectivity** (sometimes)
```

---

## 4. Best Practices

### Practice 1: Selective Column First, Then Sort Column

**Optimal Column Order Formula:**

```
Index design: (filter_cols, sort_cols, include_cols)

1. filter_cols: Columns in WHERE clause, ordered by selectivity (most → least)
2. sort_cols: Columns in ORDER BY clause
3. include_cols: SELECT columns not in (1) or (2), for covering index

Example query:
SELECT order_id, total, items
FROM orders
WHERE user_id = 12345 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 10;

Index design:
CREATE INDEX idx_orders_optimal 
ON orders(user_id, status, created_at DESC)
INCLUDE (order_id, total, items);

Breakdown:
- user_id: High selectivity (1M users) → First
- status: Lower selectivity (5 values) → Second
- created_at DESC: Sort column → Third (provides ORDER BY sort)
- INCLUDE: Covering columns → No heap fetch needed ✅
```

### Practice 2: Avoid Redundant Indexes

**Problem: Too Many Overlapping Indexes**

```sql
-- Redundant indexes (wasteful):
CREATE INDEX idx_a ON table(col_a);
CREATE INDEX idx_ab ON table(col_a, col_b);
CREATE INDEX idx_abc ON table(col_a, col_b, col_c);

-- Analysis:
-- idx_a: Redundant! idx_ab can answer "WHERE col_a = ?" (leftmost prefix)
-- idx_ab: Redundant! idx_abc can answer "WHERE col_a = ?" AND "WHERE col_a = ? AND col_b = ?"
-- idx_abc: Only index needed for all three patterns ✅

-- Waste:
-- - Storage: 3 indexes * 500MB each = 1.5GB (vs 500MB for idx_abc alone)
-- - Write overhead: INSERT updates 3 indexes (vs 1)
-- - Maintenance: VACUUM, ANALYZE, REINDEX on 3 indexes
```

**Optimization: Drop Redundant Indexes**

```sql
-- Keep only idx_abc:
DROP INDEX idx_a;
DROP INDEX idx_ab;
-- idx_abc handles all queries using leftmost prefix rule ✅

-- Savings:
-- - Storage: 1GB saved
-- - INSERT time: 3× faster (1 index update vs 3)
```

**Exception: Different Sort Orders**

```sql
-- NOT redundant:
CREATE INDEX idx_ab_asc ON table(col_a, col_b ASC);
CREATE INDEX idx_ab_desc ON table(col_a, col_b DESC);

-- idx_ab_asc: For ORDER BY col_a, col_b ASC
-- idx_ab_desc: For ORDER BY col_a, col_b DESC

-- Cannot scan idx_ab_asc backward to get DESC order efficiently (B-Tree limitation)
-- Both indexes needed if both sort orders are common ✅
```

### Practice 3: Monitor Query Patterns to Choose Column Order

**Data-Driven Index Design:**

```sql
-- Step 1: Log queries (PostgreSQL example):
ALTER SYSTEM SET log_statement = 'all';
SELECT pg_reload_conf();

-- OR use pg_stat_statements extension:
CREATE EXTENSION pg_stat_statements;

-- Step 2: Analyze query patterns:
SELECT 
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
WHERE query LIKE '%WHERE%user_id%status%'
ORDER BY calls DESC
LIMIT 10;

-- Results:
-- Query A: WHERE user_id = ? AND status = ? (10K calls/day)
-- Query B: WHERE status = ? (5K calls/day)  
-- Query C: WHERE user_id = ? (2K calls/day)

-- Decision:
-- - Query A most frequent (10K calls) → Optimize for (user_id, status)
-- - Query B significant (5K calls) → (status) alone also needed
-- - Query C less frequent (2K calls) → Covered by (user_id, status) partial use

-- Index strategy:
CREATE INDEX idx_user_status ON orders(user_id, status);  ← Query A
CREATE INDEX idx_status ON orders(status);                ← Query B (not covered by first index)

-- Alternative: Create wider index if storage acceptable:
CREATE INDEX idx_status_user ON orders(status, user_id);
-- Covers Query B perfectly, Query A less efficiently (status less selective)
-- Trade-off: Slightly slower Query A, but eliminates need for idx_user_status
```

### Practice 4: Include Covering Columns for Hot Queries

**Transform Index Scan → Index-Only Scan:**

```sql
-- Query (10K calls/sec, hot path):
SELECT order_id, total, status
FROM orders
WHERE user_id = ? AND created_at > ?
ORDER BY created_at DESC
LIMIT 20;

-- Option A: Standard composite index
CREATE INDEX idx_user_date ON orders(user_id, created_at DESC);

-- Plan:
Index Scan using idx_user_date
  - Index access: Navigate to (user_id, created_at)
  - Heap fetch: Fetch 20 tuples to get order_id, total, status
  - I/O: 20 index page reads + 20 heap page reads = 40 reads

-- Option B: Covering composite index
CREATE INDEX idx_user_date_covering 
ON orders(user_id, created_at DESC)
INCLUDE (order_id, total, status);

-- Plan:
Index Only Scan using idx_user_date_covering
  - Index access: Navigate to (user_id, created_at)
  - Data retrieval: All columns in index leaf pages (no heap fetch!) ✅
  - I/O: 20 index page reads + 0 heap page reads = 20 reads (50% faster!)

-- Performance:
-- - Latency: 2ms → 1ms (50% reduction)
-- - Throughput: 10K qps × 1ms = 10 CPU-sec/sec → 1000% CPU load
-- - vs Option A: 10K qps × 2ms = 20 CPU-sec/sec → 2000% CPU (2× higher!)

-- Trade-off:
-- - Index size: 500MB → 800MB (60% larger)
-- - Worth it? Yes! (50% latency reduction for hot path)
```

---

## 5. Common Mistakes

### Mistake 1: Wrong Column Order

**Problem: Least Selective Column First**

```sql
-- Table: orders (10M rows)
-- Columns:
--   status: 5 distinct values (pending, processing, shipped, delivered, cancelled)
--   user_id: 1M distinct values (1M users)

-- Query:
SELECT * FROM orders WHERE user_id = 12345 AND status = 'pending';

-- Bad index:
CREATE INDEX idx_status_userid ON orders(status, user_id);

-- Plan:
Index Scan using idx_status_userid
  Index Cond: (status = 'pending' AND user_id = 12345)

-- Process:
-- 1. Navigate to status = 'pending' (2M rows, 20% of table)
-- 2. Scan 2M entries, filtering user_id = 12345 → Find 1 matching
-- Cost: log(10M) + 2M = 2,000,023 operations ❌

-- Good index:
CREATE INDEX idx_userid_status ON orders(user_id, status);

-- Plan:
Index Scan using idx_userid_status
  Index Cond: (user_id = 12345 AND status = 'pending')

-- Process:
-- 1. Navigate to user_id = 12345 (10 rows average per user)
-- 2. Scan 10 entries, filtering status = 'pending' → Find 1 matching
-- Cost: log(10M) + 10 = 33 operations ✅

-- Speedup: 2,000,023 / 33 = 60,000× FASTER!
```

**Rule Simplified:**
- **High cardinality first** (user_id: 1M values) ✅
- **Low cardinality last** (status: 5 values) ✅

### Mistake 2: Ignoring Query Patterns (Creating Unused Indexes)

**Problem:**

```sql
-- Developer creates index based on gut feeling:
CREATE INDEX idx_user_created_status ON orders(user_id, created_at, status);

-- Actual query patterns (from pg_stat_statements):
-- Query A (90% of traffic): WHERE status = 'pending' ORDER BY created_at DESC
-- Query B (10% of traffic): WHERE user_id = X

-- Problem:
-- - idx_user_created_status optimized for: WHERE user_id = ? AND created_at > ? AND status = ?
-- - But this query pattern doesn't exist! (0% of traffic)
-- - Query A: Cannot use index (status not first column)
-- - Query B: Uses index (user_id first), but created_at, status wasted space

-- Result:
-- - Index never used optimally (0 impact on performance)
-- - Storage wasted: 500MB unused index
-- - Write overhead: Every INSERT updates useless index
```

**Solution: Data-Driven Design**

```sql
-- Analyze query patterns:
SELECT query, calls FROM pg_stat_statements WHERE query LIKE '%orders%' ORDER BY calls DESC;

-- Create indexes matching ACTUAL queries:
CREATE INDEX idx_status_created ON orders(status, created_at DESC);  -- Query A ✅
CREATE INDEX idx_userid ON orders(user_id);                           -- Query B ✅

-- Drop unused index:
DROP INDEX idx_user_created_status;  -- Never used, wasting space

-- Validation: Check index usage after 1 week:
SELECT 
    indexrelname AS index_name,
    idx_scan AS scans,
    idx_tup_read AS tuples_read
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan;

-- If idx_scan = 0 after 1 week → Index unused → Drop it!
```

### Mistake 3: Too Many Columns in Composite Index

**Problem: Kitchen Sink Index**

```sql
-- Developer adds "everything" to index:
CREATE INDEX idx_kitchen_sink 
ON orders(user_id, status, created_at, updated_at, total, shipping_address, notes);

-- Problems:
-- 1. Huge index size: 7 columns × average 50 bytes = 350 bytes per entry
--    10M orders × 350 bytes = 3.5GB index ❌

-- 2. Rarely uses all columns:
--    Query: WHERE user_id = X AND status = Y → Only uses 2 columns, 5 wasted!

-- 3. Write amplification:
--    UPDATE notes column → Entire 350-byte index entry rewritten (even though user_id, status unchanged)

-- 4. Low cache efficiency:
--    3.5GB index doesn't fit in 512MB buffer pool → Frequent evictions

-- 5. Maintenance overhead:
--    VACUUM, REINDEX take 10× longer
```

**Solution: Keep Composite Indexes Focused**

```sql
-- Guideline: 2-4 columns maximum (exceptions for covering indexes)

-- Replace kitchen sink with targeted indexes:
CREATE INDEX idx_user_status ON orders(user_id, status, created_at);  -- 3 columns ✅
INCLUDE (total);  -- Add covering column if needed

-- Benefits:
-- - Index size: 3 key columns + 1 include ≈ 80 bytes per entry → 800MB (4× smaller!)
-- - Write efficiency: UPDATE notes doesn't touch index (notes not included)
-- - Cache friendly: 800MB fits in buffer pool
-- - Maintenance: 4× faster VACUUM/REINDEX
```

### Mistake 4: Not Testing Index with EXPLAIN

**Problem: Assumption vs Reality**

```sql
-- Developer creates index:
CREATE INDEX idx_created_status ON orders(created_at, status);

-- Assumed query pattern:
SELECT * FROM orders WHERE created_at > '2024-01-01' AND status = 'pending';

-- Developer expectation: "Index will make this fast!"

-- Reality check with EXPLAIN:
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders WHERE created_at > '2024-01-01' AND status = 'pending';

-- Result:
Seq Scan on orders  ← Index NOT used! ❌
  Filter: (created_at > '2024-01-01' AND status = 'pending')
  Rows: 50000
  Buffers: shared hit=10000

-- Reason: created_at > '2024-01-01' matches 9M rows (90% of table)
-- Optimizer: "Seq scan cheaper than index scan for 90% of rows" → Ignores index

-- Fix: Swap column order
DROP INDEX idx_created_status;
CREATE INDEX idx_status_created ON orders(status, created_at);

-- EXPLAIN again:
Index Scan using idx_status_created  ← Using index now! ✅
  Index Cond: (status = 'pending' AND created_at > '2024-01-01')
  Rows: 50000
  Buffers: shared hit=150  ← 66× fewer buffer reads!

-- Lesson: Always verify with EXPLAIN, don't assume!
```

---

## 6. Security Considerations

### 1. Composite Index Column Order Leaks Information

**Risk:**

```sql
-- Index reveals business logic:
CREATE INDEX idx_creditcard_user_expiry 
ON payments(credit_card_number, user_id, expiry_date);

-- Attacker queries system catalog:
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'payments';

-- Results:
-- idx_creditcard_user_expiry ON (credit_card_number, user_id, expiry_date)

-- Inference:
-- - "credit_card_number is first column → High selectivity → Likely lookups by card number"
-- - "Company stores credit cards (PCI-DSS compliance question)"
-- - "Column order suggests card-to-user lookup (fraud detection?)"

-- Exploitation: Social engineering ("I work in fraud detection, need card lookup access...")
```

**Mitigation:**
- Obfuscate index names: `idx_pmt_001` instead of `idx_creditcard_...`
- Restrict catalog access: `REVOKE SELECT ON pg_indexes FROM public_role;`
- Hash/tokenize sensitive columns (don't index raw credit card numbers)

### 2. Timing Attack on Composite Index Column Order

**Risk:**

```sql
-- Index: idx_status_userid ON (status, user_id)

-- Attacker sends queries:
-- Query A:
SELECT * FROM orders WHERE user_id = 12345;
-- Response: 50ms (slow, user_id not first column → Uses index inefficiently)

-- Query B:
SELECT * FROM orders WHERE status = 'pending' AND user_id = 12345;
-- Response: 2ms (fast, status first column → Uses index efficiently)

-- Inference:
-- - Query B faster → status likely first column in index
-- - Attacker learns: "Database optimized for status-based queries" (business intelligence)
-- - If status = 'vip': Attacker identifies VIP users (targets for phishing)
```

**Mitigation:**
- Constant-time query padding (artificial delays)
- Rate limiting (prevent timing measurements)
- Use covering indexes (make both queries fast, eliminate timing difference)

---

## 7. Performance Optimization

### Optimization 1: Partial Composite Index for Skewed Workload

**Combine Partial + Composite for Maximum Efficiency:**

```sql
-- Scenario:
-- Table: orders (100M rows)
-- Query pattern: 95% of queries filter WHERE status IN ('pending', 'processing')
-- Distribution: 1M active orders (1%), 99M historical orders (99%)

-- Standard composite index:
CREATE INDEX idx_status_user_date ON orders(status, user_id, created_at);
-- Size: 100M rows × 30 bytes = 3GB

-- Optimized partial composite index:
CREATE INDEX idx_active_user_date 
ON orders(status, user_id, created_at)
WHERE status IN ('pending', 'processing');
-- Size: 1M rows × 30 bytes = 30MB (100× SMALLER!)

-- Benefits:
-- - 95% of queries covered (active orders)
-- - 100× smaller index (fits in cache 100% of time)
-- - 99% of INSERTs skip index (delivered/cancelled orders)
-- - Performance: 50ms → 0.5ms (100× FASTER!)

-- Full index still needed for 5% of historical queries:
CREATE INDEX idx_status_date ON orders(status, created_at);  -- For non-active statuses
```

### Optimization 2: Descending Indexes for ORDER BY DESC

**Match Sort Order to Query:**

```sql
-- Query: Latest orders first (common pattern)
SELECT * FROM orders
WHERE user_id = 12345
ORDER BY created_at DESC
LIMIT 10;

-- Standard index (ascending):
CREATE INDEX idx_user_created ON orders(user_id, created_at ASC);

-- Plan:
Index Scan Backward using idx_user_created
  -- Scans index in reverse → Less efficient on some storage engines
  -- Cost: log(N) + 10 rows + backward scan overhead

-- Optimized index (descending):
CREATE INDEX idx_user_created_desc ON orders(user_id, created_at DESC);

-- Plan:
Index Scan using idx_user_created_desc  ← Forward scan (natural direction) ✅
  -- Cost: log(N) + 10 rows (no backward scan overhead)

-- Performance difference:
-- - Ascending: 2ms (backward scan penalty on rotational disks)
-- - Descending: 1ms (forward scan, optimal)
-- - Modern SSDs: Negligible difference (<5%)
-- - Rotational HDDs: 20-50% faster with DESC index
```

**Syntax:**

```sql
-- PostgreSQL / SQL Server:
CREATE INDEX idx_name ON table(col_a ASC, col_b DESC);

-- MySQL (MySQL 8.0+):
CREATE INDEX idx_name ON table(col_a ASC, col_b DESC);

-- Oracle:
CREATE INDEX idx_name ON table(col_a ASC, col_b DESC);
```

### Optimization 3: Index-Only Scans with Composite Covering Indexes

**Eliminate Heap Access Entirely:**

```sql
-- Query (hot path, 50K qps):
SELECT order_id, total, created_at
FROM orders
WHERE user_id = ?
  AND status IN ('pending', 'processing')
ORDER BY created_at DESC
LIMIT 20;

-- Standard composite index:
CREATE INDEX idx_user_status_created ON orders(user_id, status, created_at DESC);

-- Plan:
Index Scan using idx_user_status_created
  - Index access: 20 index entries
  - Heap fetch: 20 tuples to get order_id, total ❌
  - I/O: 20 index reads + 20 heap reads = 40 reads

-- Optimized covering composite index:
CREATE INDEX idx_user_status_created_covering
ON orders(user_id, status, created_at DESC)
INCLUDE (order_id, total);

-- Plan:
Index Only Scan using idx_user_status_created_covering  ✅
  - Index access: 20 index entries (all data in index!)
  - Heap fetch: 0 (no heap access!)
  - I/O: 20 index reads + 0 heap reads = 20 reads (50% faster!)

-- Performance:
-- - Latency: 2ms → 1ms (50% improvement)
-- - Throughput: 50K qps × 2ms → 50K qps × 1ms (50% less CPU)
-- - Savings: 50K qps × 0.001s × $0.05/CPU-hour = $2.50/hour × 24 × 30 = $1800/month

-- Trade-off:
-- - Index size: 500MB → 750MB (50% larger)
-- - Write overhead: Slightly higher (UPDATE total updates index)
-- - Decision: Worth it! (50% latency reduction + $1800/month savings)
```

---

## 8. Examples

### Example 1: Date Range + Category Filter

**Scenario: E-commerce product listings**

```sql
CREATE TABLE products (
    product_id INT,
    category VARCHAR(50),
    created_at DATE,
    price DECIMAL(10,2),
    name VARCHAR(255)
);

-- Query: Recent products in category
SELECT product_id, name, price
FROM products
WHERE category = 'Electronics'
  AND created_at > '2024-01-01'
ORDER BY created_at DESC
LIMIT 50;

-- Index design:
CREATE INDEX idx_products_cat_date 
ON products(category, created_at DESC)
INCLUDE (product_id, name, price);

-- Benefits:
-- - category first: High selectivity (100 categories)
-- - created_at DESC: Provides ORDER BY sort (no sort phase)
-- - INCLUDE: Index-only scan (no heap fetch)
-- - Performance: 100ms → 2ms (50× faster)
```

---

### Example 2: Multi-Column Sort

**Scenario: Employee directory sorted by department, then hire date**

```sql
CREATE TABLE employees (
    emp_id INT,
    department VARCHAR(50),
    hire_date DATE,
    salary DECIMAL(10,2)
);

-- Query: Engineering employees, newest first within department
SELECT emp_id, hire_date, salary
FROM employees
WHERE department = 'Engineering'
ORDER BY hire_date DESC
LIMIT 100;

-- Index design:
CREATE INDEX idx_emp_dept_date 
ON employees(department, hire_date DESC)
INCLUDE (emp_id, salary);

-- Plan:
Index Only Scan using idx_emp_dept_date
  Index Cond: (department = 'Engineering')
  -- No sort phase! Index provides sort order ✅
  Limit: 100

-- Performance:
-- - No explicit sort (index pre-sorted)
-- - Index-only scan (all columns in index)
-- - Result: 0.5ms query time
```

---

### Example 3: Composite Index on Foreign Keys

**Scenario: Order items lookup by order_id + product_id**

```sql
CREATE TABLE order_items (
    order_item_id BIGINT,
    order_id BIGINT,
    product_id BIGINT,
    quantity INT,
    price DECIMAL(10,2)
);

-- Query A (95% of traffic): Get items for an order
SELECT * FROM order_items WHERE order_id = 12345;

-- Query B (5% of traffic): Get specific item
SELECT * FROM order_items WHERE order_id = 12345 AND product_id = 67890;

-- Composite index:
CREATE INDEX idx_order_product ON order_items(order_id, product_id);

-- Query A plan:
Index Scan using idx_order_product
  Index Cond: (order_id = 12345)
  -- Uses leftmost prefix (order_id) ✅

-- Query B plan:
Index Scan using idx_order_product
  Index Cond: (order_id = 12345 AND product_id = 67890)
  -- Uses full composite index ✅

-- Single index covers both query patterns!
```

---

## 9. Real-World Use Cases

### Use Case 1: GitHub - Repository Search (Language + Stars)

**Context:**
- Search repositories by programming language + star count
- Queries: "Top JavaScript repos with >1000 stars"

**Implementation:**

```sql
CREATE INDEX idx_repos_lang_stars 
ON repositories(language, stars DESC)
INCLUDE (repo_id, name, description);

-- Query:
SELECT repo_id, name, description, stars
FROM repositories
WHERE language = 'JavaScript'
  AND stars > 1000
ORDER BY stars DESC
LIMIT 50;

-- Benefits:
-- - language first: Filters to 5M JavaScript repos (from 200M total)
-- - stars DESC: Pre-sorted (no sort phase needed)
-- - INCLUDE: Index-only scan
-- - Performance: <5ms (vs 30s seq scan)
```

### Use Case 2: Uber - Driver Matching (Location + Availability)

**Context:**
- Find available drivers near rider location
- Composite index on (geohash, availability_status)

**Implementation:**

```sql
CREATE INDEX idx_drivers_geo_status 
ON drivers(geohash, availability_status)
WHERE availability_status = 'available';

-- Query:
SELECT driver_id, lat, lon, rating
FROM drivers
WHERE geohash = 'dr5regw'  -- San Francisco area
  AND availability_status = 'available';

-- Benefits:
-- - geohash first: Narrows to ~500 drivers in area
-- - availability_status: Filters to ~50 available
-- - Partial index: Only indexes available drivers (90% space savings)
-- - Performance: 50ms → 2ms (25× faster) → Instant driver matching!
```

### Use Case 3: Facebook - User Feed (Friendship + Timestamp)

**Context:**
- News feed: Show posts from friends, newest first
- Composite index on (user_id, created_at DESC)

**Implementation:**

```sql
-- Denormalized table: user_posts_feed
CREATE TABLE user_posts_feed (
    user_id BIGINT,      -- Viewer user ID
    post_id BIGINT,
    author_id BIGINT,    -- Friend who created post
    created_at TIMESTAMP
);

CREATE INDEX idx_feed_user_time 
ON user_posts_feed(user_id, created_at DESC)
INCLUDE (post_id, author_id);

-- Query:
SELECT post_id, author_id, created_at
FROM user_posts_feed
WHERE user_id = 12345  -- Viewer
ORDER BY created_at DESC
LIMIT 25;  -- First page of feed

-- Plan:
Index Only Scan using idx_feed_user_time
  Index Cond: (user_id = 12345)
  Limit: 25

-- Performance:
-- - Index-only scan: No heap access ✅
-- - Pre-sorted: No sort phase ✅
-- - Limit: Stops after 25 entries (doesn't scan all posts)
-- - Result: Sub-millisecond query time for 2 billion daily feed loads
```

---

## 10. Interview Questions

### Q1: Explain the leftmost prefix rule and why column order matters in composite indexes.

**Senior Answer:**

```
**Leftmost Prefix Rule:**

A composite index on (col_a, col_b, col_c) is sorted by col_a first, then col_b within each col_a value, then col_c within each (col_a, col_b) combination. This nested sorting means:

- Queries filtering on (col_a): Can use index ✅
- Queries filtering on (col_a, col_b): Can use index ✅  
- Queries filtering on (col_a, col_b, col_c): Can use index ✅
- Queries filtering on (col_b): Cannot use index efficiently ❌ (col_b not sorted globally)
- Queries filtering on (col_c): Cannot use index efficiently ❌
- Queries filtering on (col_a, col_c): Partially uses index (uses col_a, scans + filters col_c) ⚠️

**Why Column Order Matters:**

Example: Index idx_abc ON (col_a, col_b, col_c)

Physical structure (B-Tree):
[col_a=1, col_b=10, col_c=100, RID]
[col_a=1, col_b=10, col_c=101, RID]
[col_a=1, col_b=20, col_c=200, RID]
[col_a=2, col_b=5, col_c=50, RID]
[col_a=2, col_b=15, col_c=150, RID]

Query: WHERE col_a = 1 AND col_b = 10
  → Navigate to (1, 10, *) in B-Tree → Scan (1, 10, 100), (1, 10, 101) → Efficient ✅

Query: WHERE col_b = 10
  → No efficient navigation! col_b=10 appears in different parts of tree:
     (1, 10, *), (1, 10, *), (5, 10, *), etc.
  → Must scan entire index, filtering col_b=10 → Inefficient ❌

**Selectivity Impact:**

Example: orders table
- user_id: 1M distinct values (high selectivity: 1/1M = 0.0001%)
- status: 5 distinct values (low selectivity: 1/5 = 20%)

Query: WHERE user_id = 12345 AND status = 'pending'

Option A: idx_user_status ON (user_id, status)
  → Navigate to user_id=12345 (10 rows)
  → Filter status='pending' (typically 1 row)
  → Cost: log(10M) + 10 = 33 operations

Option B: idx_status_user ON (status, user_id)
  → Navigate to status='pending' (2M rows, 20% of table!)
  → Filter user_id=12345 (1 row)
  → Cost: log(10M) + 2M = 2,000,023 operations

**Speedup: 60,000× faster with correct column order!**

**Real-World Decision:**

At my previous company, we had an index on (created_at, user_id) for orders table. Query pattern: 90% of queries filtered WHERE user_id = ? AND created_at > ?. Performance was poor (50ms P99).

Analysis with EXPLAIN showed index scanning millions of created_at entries, filtering user_id (created_at not selective enough—90% of orders created in last year).

Fix: Reverse order to (user_id, created_at). Performance improved 100×: 50ms → 0.5ms. Lesson: High-selectivity column first, then sort/range column.
```

---

### Q2: When should you create multiple single-column indexes vs one composite index?

**Staff Answer:**

```
**Create Multiple Single-Column Indexes When:**

1. **Queries filter on columns independently (OR logic)**

Example:
SELECT * FROM users WHERE email = 'alice@example.com';  -- Query A
SELECT * FROM users WHERE username = 'alice123';         -- Query B

Indexes:
CREATE INDEX idx_email ON users(email);     -- Query A
CREATE INDEX idx_username ON users(username); -- Query B

Why not composite (email, username)?
  → Query A: WHERE email = ? (uses leftmost prefix) ✅
  → Query B: WHERE username = ? (cannot use index, username not first) ❌

Separate indexes cover both queries independently.

2. **OR queries with bitmap scanning**

Query: WHERE email = ? OR username = ?

Indexes: idx_email, idx_username

Plan: Bitmap Index Scan
  1. Scan idx_email → Bitmap A
  2. Scan idx_username → Bitmap B
  3. Bitmap OR: A ∪ B
  4. Fetch heap tuples from combined bitmap

Composite index (email, username) cannot help (no single leftmost column matches OR condition).

3. **Different query patterns with no overlap**

Example: Analytics table
  - Query A (80%): WHERE created_at > ? (time-series analysis)
  - Query B (20%): WHERE status = ? (status reporting)

Indexes:
CREATE INDEX idx_created ON table(created_at);
CREATE INDEX idx_status ON table(status);

No composite index useful (queries don't filter on same columns together).

---

**Create Composite Index When:**

1. **Queries filter on multiple columns together (AND logic)**

Example:
SELECT * FROM orders WHERE user_id = ? AND status = ?;  -- 95% of queries

Index:
CREATE INDEX idx_user_status ON orders(user_id, status);

Why not separate?
  - Separate indexes: Bitmap Index Scan on both, combine bitmaps, fetch heap (slower)
  - Composite: Direct navigation to (user_id, status) in one index (faster)

2. **Queries have WHERE + ORDER BY on different columns**

Example:
SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 10;

Index:
CREATE INDEX idx_user_created ON orders(user_id, created_at DESC);

Benefits:
  - Filters user_id (first column)
  - Provides sort order for created_at (second column)
  - No explicit sort phase needed ✅

Separate indexes cannot provide both filtering + sorting efficiently.

3. **High-frequency query with consistent column combination**

Example: E-commerce product search
  - Query: WHERE category = ? AND price BETWEEN ? AND ? (10K qps)

Index:
CREATE INDEX idx_category_price ON products(category, price);

Benefits:
  - Optimizes hot path (10K qps)
  - Single index navigation (faster than bitmap combining)
  - Smaller index than sum of separate indexes

---

**Real-World Decision Matrix:**

| Scenario | Index Strategy | Reason |
|----------|----------------|--------|
| WHERE email = ? OR username = ? | Separate: idx_email, idx_username | OR logic, bitmap combine |
| WHERE user_id = ? AND status = ? | Composite: idx_user_status | AND logic, consistent pattern |
| WHERE user_id = ? with ORDER BY created_at | Composite: idx_user_created | Filter + sort efficiency |
| WHERE created_at > ? (80%) vs WHERE status = ? (20%) | Separate: idx_created, idx_status | Independent queries |
| WHERE category = ? AND price BETWEEN ? (hot path) | Composite: idx_category_price | Hot path optimization |

---

**Trade-Offs:**

**Separate Indexes:**
- Pros: Covers independent query patterns, simpler maintenance
- Cons: Larger total storage (2 indexes vs 1), slower bitmap combining

**Composite Index:**
- Pros: Faster for AND queries, smaller storage (1 index vs 2), provides sorting
- Cons: Only helps if WHERE matches leftmost columns, less flexible

**Interview Example:**

"At my previous company, we had an orders table with separate indexes on user_id and status. Query: WHERE user_id = ? AND status = ? (hot path, 20K qps).

Performance: 10ms P99 (bitmap combining 2 indexes, then heap fetch).

Optimization: Created composite index idx_user_status. Performance: 10ms → 1ms (10× faster). Throughput increased from 20K qps at 90% CPU to 20K qps at 20% CPU.

Storage: Separate indexes 500MB each = 1GB total. Composite: 600MB (40% savings).

Lesson: Composite indexes essential for hot-path multi-column AND queries. Separate indexes better for independent/OR queries."
```

---

### Q3: How do you determine the optimal column order in a composite index?

**Principal Answer:**

```
**Framework for Optimal Column Order:**

**Step 1: Analyze Query Patterns**

Extract query logs and identify dominant patterns:

SELECT 
    query,
    calls,
    mean_exec_time
FROM pg_stat_statements
WHERE query LIKE '%table_name%'
ORDER BY calls * mean_exec_time DESC  -- High impact queries
LIMIT 20;

Example results:
- Query A (80%): WHERE user_id = ? AND status = ? ORDER BY created_at
- Query B (15%): WHERE status = ? ORDER BY created_at
- Query C (5%): WHERE user_id = ? ORDER BY created_at

**Step 2: Selectivity Analysis**

Calculate column cardinality:

SELECT 
    COUNT(DISTINCT user_id) AS user_cardinality,
    COUNT(DISTINCT status) AS status_cardinality,
    COUNT(*) AS total_rows
FROM orders;

Results:
- user_id: 1M distinct (cardinality: 1M / 10M = 0.0001 selectivity per value)
- status: 5 distinct (cardinality: 5 / 10M = 0.0002 selectivity per value)
- user_id more selective ✅

**Step 3: Apply Column Order Rules**

**Rule 1: Equality before Range**

Query: WHERE user_id = ? AND created_at > ?

Index order: (user_id, created_at)  ✅
  - user_id equality filter narrows immediately
  - created_at range scan within user_id subset

Wrong order: (created_at, user_id)  ❌
  - created_at > '2024-01-01' matches 9M rows (90%)
  - user_id filter applied after (huge scan)

**Rule 2: High Selectivity before Low Selectivity**

Query: WHERE user_id = ? AND status = ?

Column selectivity:
- user_id: 1/1M = 0.0001% per value (high selectivity)
- status: 1/5 = 20% per value (low selectivity)

Index order: (user_id, status)  ✅
  - Narrows to ~10 rows via user_id
  - Filters status within 10 rows (trivial)

Wrong order: (status, user_id)  ❌
  - Narrows to 2M rows via status (20%)
  - Filters user_id within 2M rows (expensive)

**Rule 3: Frequent Queries before Rare Queries**

Query frequency:
- Query A (80%): WHERE user_id = ?
- Query B (20%): WHERE status = ?

Index order: (user_id, status)  ✅
  - Query A: Uses leftmost prefix (80% coverage)
  - Query B: Cannot use index efficiently (only 20% queries affected)

Trade-off: Optimize for 80% case, accept suboptimal 20%.

Alternative: Create two indexes if 20% queries are critical:
  - idx_user_status: For Query A (80%)
  - idx_status: For Query B (20%)

**Rule 4: Filter Columns before Sort Columns**

Query: WHERE user_id = ? ORDER BY created_at

Index order: (user_id, created_at)  ✅
  - user_id filters rows
  - created_at provides sort order (no sort phase needed)

Wrong order: (created_at, user_id)  ❌
  - created_at not useful for filtering (WHERE not on created_at directly)
  - user_id not first → Cannot use leftmost prefix efficiently

**Rule 5: Sort Column Last (provides ORDER BY)**

Query: WHERE status = 'pending' ORDER BY created_at DESC LIMIT 10

Index order: (status, created_at DESC)  ✅
  - Filters status
  - Scans in created_at DESC order (LIMIT stops after 10)
  - No explicit sort phase ✅

Benefits:
  - Query cost: log(N) + 10 (LIMIT stops early)
  - vs idx(status, created_at ASC): Need explicit sort DESC (slower)

---

**Step 4: Decision Tree**

Query: WHERE col_a = ? AND col_b = ? AND col_c BETWEEN ? AND ? ORDER BY col_d

Column properties:
- col_a: High selectivity, equality
- col_b: Low selectivity, equality
- col_c: Medium selectivity, range
- col_d: Sort column

Index order: (col_a, col_b, col_c, col_d)

Reasoning:
1. col_a first: High selectivity + equality (Rule 2)
2. col_b second: Low selectivity + equality, but still equality before range (Rule 1)
3. col_c third: Range condition (after equalities, Rule 1)
4. col_d last: Sort column (Rule 5)

---

**Step 5: Validate with EXPLAIN**

CREATE INDEX idx_test ON orders(user_id, status, created_at DESC);

EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders
WHERE user_id = 12345 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 10;

Check for:
  - Index Scan (not Seq Scan) ✅
  - No explicit Sort step ✅
  - Low buffer reads ✅

If plan suboptimal → Adjust column order, retest.

---

**Real-World Example:**

Company: SaaS analytics platform
Table: events (1B rows)
Columns: tenant_id (10K values), event_type (50 values), created_at (timestamps)

Query (90% traffic): 
WHERE tenant_id = ? AND event_type = ? AND created_at > NOW() - INTERVAL '7 days'

Initial index: idx_created_type_tenant (created_at, event_type, tenant_id)
  - Created by junior dev, thinking "time-series data, date first"
  - Performance: 500ms P99 (seq scan on date range, then filter tenant + type)

Optimized index: idx_tenant_type_created (tenant_id, event_type, created_at)

Reasoning:
  1. tenant_id first: 10K values → 100K rows per tenant (high selectivity)
  2. event_type second: 50 values → 2K rows per (tenant, type)
  3. created_at last: Range filter on small subset (2K → 200 rows for 7 days)

Result: 500ms → 1ms (500× FASTER!)

Savings: Scaled from 4 servers to 1 server (75% infrastructure cost reduction).

Lesson: **Selectivity + equality before ranges. Date columns rarely belong first unless queries are purely time-based with no other filters.**
```

---

## 11. Summary

### Composite Index Key Principles

**1. Sorted Structure**
- Composite index sorted by col1, then col2, then col3 (nested sorting)
- Physical layout: [col1_val, col2_val, col3_val, RID]
- Enables efficient multi-column filtering + sorting

**2. Leftmost Prefix Rule**
- Index usable for: (col1), (col1, col2), (col1, col2, col3)
- Index NOT usable for: (col2), (col3), (col2, col3)
- Query WHERE must start with leftmost column(s)

**3. Column Order Determines Performance**
- High selectivity → Low selectivity (most → least selective)
- Equality filters before range filters
- Filter columns before sort columns
- Optimize for dominant query pattern (80%+ traffic)

**4. Single Index Covers Multiple Patterns**
- Index idx(user_id, status) covers:
  - WHERE user_id = ? (leftmost prefix)
  - WHERE user_id = ? AND status = ? (full index)
  - ORDER BY within user_id subset

### When to Use Composite Indexes

**✅ Create Composite Index:**
- Multi-column AND queries (WHERE col_a = ? AND col_b = ?)
- WHERE + ORDER BY on different columns
- High-frequency consistent query pattern
- Need to avoid explicit sort phase
- Covering index (add INCLUDE columns for index-only scan)

**❌ Use Separate Single-Column Indexes:**
- OR queries (WHERE col_a = ? OR col_b = ?)
- Independent query patterns with no overlap
- Queries filter on different columns at different times

### Optimal Column Order Rules

| Priority | Rule | Example |
|----------|------|---------|
| 1 | Equality before range | (user_id =, created_at >) not (created_at >, user_id =) |
| 2 | High selectivity before low selectivity | (user_id, status) not (status, user_id) |
| 3 | Frequent queries before rare queries | Optimize for 80% traffic pattern |
| 4 | Filter columns before sort columns | (status, created_at) not (created_at, status) |
| 5 | Sort column last | (filter_col1, filter_col2, sort_col) |

### Performance Characteristics

| Metric | Separate Indexes | Composite Index | Improvement |
|--------|------------------|-----------------|-------------|
| Query time (AND filter) | 10ms (bitmap combine) | 1ms (direct seek) | 10× faster |
| Storage | 2× (two indexes) | 1× (one index) | 50% savings |
| Index-only scan | Difficult | Easy (INCLUDE cols) | 2-10× speedup |
| Sort elimination | No | Yes (sort col last) | 2-5× speedup |

### Implementation Checklist

**Creating Composite Indexes:**
- [ ] Analyze query patterns (pg_stat_statements, slow query log)
- [ ] Calculate column selectivity (COUNT(DISTINCT col) / COUNT(*))
- [ ] Order columns: Equality → Range → Sort
- [ ] Order by selectivity: High → Low
- [ ] Add INCLUDE columns for covering (if read-heavy)
- [ ] Create index: `CREATE INDEX idx ON table(col1, col2, col3) INCLUDE (col4, col5);`
- [ ] Run ANALYZE to update statistics
- [ ] Verify with EXPLAIN: Check "Index Scan" + no "Sort" step

**Monitoring:**
- [ ] Check index usage: `pg_stat_user_indexes.idx_scan > 0`
- [ ] Verify leftmost prefix usage (queries filtering col1, col1+col2, etc.)
- [ ] Monitor index bloat after heavy UPDATEs
- [ ] Drop redundant indexes (shorter prefix indexes covered by longer ones)

### Quick Wins

**1. Add composite index for hot AND query (1 hour, 10-100× speedup)**
```sql
-- Identify hot query:
SELECT query, calls FROM pg_stat_statements ORDER BY calls DESC LIMIT 5;

-- Create composite index:
CREATE INDEX idx_name ON table(col1, col2);  -- col1 most selective first
```

**2. Add INCLUDE columns for index-only scan (30 min, 2-5× speedup)**
```sql
CREATE INDEX idx_name ON table(filter_col1, filter_col2)
INCLUDE (select_list_col1, select_list_col2);
```

**3. Drop redundant indexes (15 min, storage savings + faster writes)**
```sql
-- Redundant: idx_a(col_a) when idx_ab(col_a, col_b) exists
DROP INDEX idx_a;  -- idx_ab covers (col_a) via leftmost prefix
```

### Database-Specific Syntax

**PostgreSQL:**
```sql
CREATE INDEX idx ON table(col1, col2 DESC) INCLUDE (col3, col4);
```

**MySQL:**
```sql
CREATE INDEX idx ON table(col1, col2 DESC);
-- No INCLUDE support (use index all columns instead)
```

**SQL Server:**
```sql
CREATE INDEX idx ON table(col1, col2 DESC) INCLUDE (col3, col4);
```

**Oracle:**
```sql
CREATE INDEX idx ON table(col1, col2 DESC);
-- No INCLUDE (workaround: Add columns to index key, waste space)
```

### Key Takeaway

**"Composite indexes are essential for multi-column AND queries. Column order determines performance: high-selectivity equality columns first, ranges next, sort column last. One well-designed composite index can eliminate 5 single-column indexes while delivering 10-100× speedup."**

Example decision: Query `WHERE user_id = ? AND status = ? ORDER BY created_at DESC`
- user_id: 1M values (high selectivity) → First
- status: 5 values (low selectivity) → Second  
- created_at: Sort column → Last

Index: `CREATE INDEX idx ON orders(user_id, status, created_at DESC);`

Result: 50ms → 0.5ms (100× faster), no sort phase needed! ✅

---

**Next:** `07_Full_Text_Indexes.md` - GIN, GiST, and inverted indexes for text search

