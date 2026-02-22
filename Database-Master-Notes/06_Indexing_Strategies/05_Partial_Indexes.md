# Partial Indexes - Indexing Subsets for Efficiency

## 1. Concept Explanation

**Partial indexes** (also called **filtered indexes**) index only a subset of rows matching a WHERE clause, dramatically reducing index size and improving performance for queries that filter on that same condition. They're perfect for skewed data distributions where most queries target a small,

 consistent subset.

Think of a partial index like a specialized filing cabinet:
- **Full index:** Cabinet storing all documents (active + archived + deleted)
- **Partial index:** Cabinet storing only active documents (90% smaller, faster to search)

### Partial vs Full Index

```
Table: orders (10M rows)
Distribution: status column
  - 'pending': 100K rows (1%)
  - 'processing': 50K rows (0.5%)
  - 'shipped': 2M rows (20%)
  - 'delivered': 7.8M rows (78%)
  - 'cancelled': 50K rows (0.5%)

Query pattern: 95% of queries filter WHERE status IN ('pending', 'processing')
```

**Full Index:**
```sql
CREATE INDEX idx_orders_status ON orders(status);

-- Index size: 10M rows × 20 bytes ≈ 200MB
-- Query: WHERE status = 'pending'
--   Scan index: 100K matching entries from 10M total
--   I/O: Navigate tree + scan entries
```

**Partial Index:**
```sql
CREATE INDEX idx_orders_active ON orders(status, created_at)
WHERE status IN ('pending', 'processing');

-- Index size: 150K rows × 20 bytes ≈ 3MB (66× SMALLER!)
-- Query: WHERE status = 'pending'
--   Scan index: 100K matching entries from 150K total
--   I/O: Navigate tree (shallower!) + scan entries

-- Benefits:
-- - 66× smaller index (fits in cache!)
-- - Shallower tree: 3 levels vs 4
-- - Faster inserts: Only 1.5% of INSERTs update index
```

**Key Properties:**
- **WHERE clause:** Index predicate filters rows at creation time
- **Query matching:** Query WHERE must match or be subset of index WHERE
- **Space efficiency:** Only index relevant data
- **Write efficiency:** Fewer rows = fewer index updates

---

## 2. Why It Matters

### Production Impact: Active Orders Dashboard

**Scenario: E-commerce orders (100M total, 1M active)**

```sql
-- Dashboard query (runs 1K times/sec):
SELECT order_id, user_id, items, total, created_at
FROM orders
WHERE status IN ('pending', 'processing')
ORDER BY created_at DESC
LIMIT 50;
```

**Before: Full Index**
```sql
CREATE INDEX idx_orders_status_date ON orders(status, created_at DESC);

-- Index size: 100M rows × 30 bytes = 3GB
-- Buffer pool: 16GB RAM
-- Index cached: 3GB / 16GB = 18.75% (part of index, frequent evictions)

-- Query performance:
Index Scan using idx_orders_status_date
  Index Cond: (status IN ('pending', 'processing'))
  Rows: 1M (scanned) → 50 (returned)
  Buffers: shared hit=5000, shared read=1000  ← Cache misses!
  Time: 50ms (cache thrashing)

-- Throughput: 1K queries/sec × 50ms = 50 CPU-seconds/sec → 5000% CPU!
```

**After: Partial Index**
```sql
CREATE INDEX idx_orders_active ON orders(status, created_at DESC)
WHERE status IN ('pending', 'processing');

-- Index size: 1M rows × 30 bytes = 30MB (100× SMALLER!)
-- Buffer pool: 16GB RAM
-- Index cached: 30MB / 16GB = 0.19% (entire index fits in cache!)

-- Query performance:
Index Scan using idx_orders_active
  Index Cond: (status IN ('pending', 'processing'))
  Rows: 1M (scanned) → 50 (returned)
  Buffers: shared hit=200, shared read=0  ← 100% cache hits! ✅
  Time: 2ms (25× FASTER!)

-- Throughput: 1K queries/sec × 2ms = 2 CPU-seconds/sec → 200% CPU

-- Savings:
-- - Storage: 3GB → 30MB (99% reduction)
-- - CPU: 5000% → 200% (96% reduction)
-- - P99 latency: 50ms → 2ms (96% improvement)
-- - Infrastructure cost: $2000/month → $80/month
```

### Real Numbers: Write Performance Impact

**Scenario: 10K orders/sec insertion rate**

| Metric | Full Index | Partial Index | Improvement |
|--------|------------|---------------|-------------|
| Index size | 3GB | 30MB | 100× smaller |
| % INSERTs indexed | 100% | 1% | 99% skip index |
| INSERT time | 5ms | 0.5ms | 10× faster |
| Index page splits | 10K/sec | 100/sec | 100× fewer |
| Disk writes | 50MB/sec | 0.5MB/sec | 100× less |

**Impact:**
- Write throughput: +900% (10× faster INSERTs)
- Disk wear (SSD): Reduced by 99% (10 years → 1000 years lifespan!)
- Replication lag: 5 seconds → 0.05 seconds (replica catch-up)

---

## 3. Internal Working

### Index Predicate Evaluation During INSERT

**INSERT Algorithm with Partial Index:**

```
Partial index: idx_orders_active WHERE status IN ('pending', 'processing')

INSERT INTO orders (order_id, status, ...) VALUES (12345, 'pending', ...);

Steps:
1. Insert tuple into heap
2. For each index on table:
     a. Check if index is partial (has WHERE clause)
     b. If partial: Evaluate WHERE predicate against new tuple
          - status = 'pending' IN ('pending', 'processing') → TRUE
          - Proceed to step c
     c. If TRUE or index is full: Insert entry into index
     d. If FALSE: Skip index update ✅

Result: Index updated (status='pending' matches predicate)

---

INSERT INTO orders (order_id, status, ...) VALUES (67890, 'delivered', ...);

Steps:
1. Insert tuple into heap
2. For each index:
     a. Check predicate: status = 'delivered' IN ('pending', 'processing') → FALSE
     b. Skip index update ✅

Result: Index NOT updated (99% of INSERTs skip this index)

---

Write amplification comparison:

Full index:
- Heap write: 500 bytes
- Index write: 30 bytes (every INSERT)
- Total: 530 bytes per INSERT

Partial index (1% match rate):
- Heap write: 500 bytes
- Index write: 30 bytes (only 1% of INSERTs)
- Total: 500.3 bytes average per INSERT (94% reduction in index writes!)
```

### Query Planner: Index Match Detection

**Optimizer Rules for Using Partial Index:**

```
Partial index: WHERE status = 'pending'
Query: WHERE status = 'pending' AND created_at > '2024-01-01'

Planner logic:
1. Query WHERE must IMPLY index WHERE (query is subset)
   - Query: status='pending' AND created_at>'2024-01-01'
   - Index: status='pending'
   - Implication: YES (query includes index condition)
   - Result: Index is candidate ✅

2. Check selectivity:
   - Index selectivity: 1% of table
   - Query selectivity: 0.5% of table (pending + date filter)
   - Index helps narrow down to 1%, then filter further
   - Result: Use partial index ✅

---

Query: WHERE status = 'delivered'

Planner logic:
1. Query WHERE must IMPLY index WHERE
   - Query: status='delivered'
   - Index: status='pending'
   - Implication: NO (delivered != pending)
   - Result: Index not applicable ❌

2. Fallback: Use different index or seq scan
```

**Implication Check Examples:**

| Index WHERE | Query WHERE | Usable? | Reason |
|-------------|-------------|---------|--------|
| `status='pending'` | `status='pending'` | ✅ Yes | Exact match |
| `status='pending'` | `status='pending' AND user_id=5` | ✅ Yes | Query subset |
| `status IN ('pending','processing')` | `status='pending'` | ✅ Yes | Query implies index |
| `status='pending'` | `status IN ('pending','processing')` | ❌ No | Query superset (includes processing) |
| `created_at > '2024-01-01'` | `created_at > '2024-02-01'` | ✅ Yes | Query subset (Feb > Jan) |
| `created_at > '2024-02-01'` | `created_at > '2024-01-01'` | ❌ No | Query superset |

### Storage: Only Matching Rows Indexed

**Physical Structure:**

```
Table: orders (heap)
[order_id: 1, status: 'delivered', ...]  ← Not in partial index
[order_id: 2, status: 'pending', ...]    ← IN partial index ✅
[order_id: 3, status: 'delivered', ...]  ← Not in partial index
[order_id: 4, status: 'processing', ...] ← IN partial index ✅
[order_id: 5, status: 'delivered', ...]  ← Not in partial index

Partial index: idx_orders_active WHERE status IN ('pending', 'processing')

Index B-Tree:
Root: [Key: 2, 4, ...]  ← Only orders with matching status

Leaf pages:
[order_id: 2, status: 'pending', created_at: '2024-02-20'] → RID: (page=1, offset=2)
[order_id: 4, status: 'processing', created_at: '2024-02-21'] → RID: (page=2, offset=1)
... (only 1M entries, not 100M)

Index size: 1M entries × 30 bytes = 30MB
vs full index: 100M entries × 30 bytes = 3GB
```

---

## 4. Best Practices

### Practice 1: Index Small, Frequently Queried Subsets

**❌ Bad: Partial Index on Large Subset**
```sql
-- 80% of orders are 'delivered'
CREATE INDEX idx_orders_delivered ON orders(order_id)
WHERE status = 'delivered';

-- Problems:
-- - Index still large: 80M rows (80% of table)
-- - Small savings: 3GB → 2.4GB (only 20% reduction)
-- - Cache efficiency: Doesn't fit in memory anyway
-- - Most queries target other statuses (pending, processing)
-- - Wrong subset indexed!
```

**✅ Good: Partial Index on Small Subset**
```sql
-- 1% of orders are 'pending' or 'processing'
-- 95% of queries target these statuses (dashboard, active order management)

CREATE INDEX idx_orders_active ON orders(status, created_at)
WHERE status IN ('pending', 'processing');

-- Benefits:
-- - Index tiny: 1M rows (1% of table) = 30MB
-- - Fits entirely in cache (100% hit rate)
-- - Covers 95% of query patterns
-- - 99% of INSERTs skip index (fast writes)
```

**Selection Criteria:**
- **Subset size:** <10% of table rows (smaller is better)
- **Query frequency:** >80% of queries filter on same subset
- **WHERE stability:** Predicate doesn't change (avoid reindex)

### Practice 2: Combine Partial with Covering

**Power Combo: Partial + Covering = Maximum Efficiency**

```sql
-- Query: Active order details
SELECT order_id, user_id, total, created_at, items
FROM orders
WHERE status IN ('pending', 'processing')
ORDER BY created_at DESC
LIMIT 100;

-- Partial + Covering index:
CREATE INDEX idx_orders_active_covering 
ON orders(status, created_at DESC)
INCLUDE (order_id, user_id, total, items)
WHERE status IN ('pending', 'processing');

-- Benefits:
-- 1. Partial: Only 1% of rows indexed (small index)
-- 2. Covering: No heap access (index-only scan)
-- 3. Combined: Tiny index (30MB) + zero heap fetches = ultra-fast!

-- Performance:
-- - Index-only scan: 0 heap fetches ✅
-- - Index size: 30MB (fits in cache)
-- - Query time: 0.5ms (vs 50ms without optimization)
-- - Speedup: 100× FASTER!
```

### Practice 3: Use Partial Indexes for Null Filtering

**Scenario: Nullable Column with Sparse Non-Null Values**

```sql
-- Table: users (10M rows)
-- Column: deleted_at (TIMESTAMPTZ, nullable)
-- Distribution: 99% NULL (active users), 1% NOT NULL (deleted users)

-- Query pattern: Only query active users
SELECT * FROM users WHERE deleted_at IS NULL AND email = ?;
```

**❌ Bad: Full Index (Indexes NULLs Unnecessarily)**
```sql
CREATE INDEX idx_users_email ON users(email);

-- Index size: 10M rows × 50 bytes = 500MB
-- Includes 10M NULL deleted_at entries (wasted space)
```

**✅ Good: Partial Index (Exclude Deleted Users)**
```sql
CREATE INDEX idx_users_active_email ON users(email)
WHERE deleted_at IS NULL;

-- Index size: 9.9M rows × 50 bytes = 495MB
-- Slight savings, but better approach:

CREATE INDEX idx_users_active_email ON users(email, user_id)
WHERE deleted_at IS NULL;

-- Benefits:
-- - Only indexes active users (99% of queries)
-- - Deleted user queries use different index or seq scan (acceptable)
-- - Write efficiency: UPDATE ... SET deleted_at = NOW() removes from index (less bloat)

-- Better semantics:
-- "This index is for active users" (explicit intent)
```

**Common Pattern:**
```sql
-- Soft delete pattern:
WHERE deleted_at IS NULL

-- Pending items:
WHERE completed_at IS NULL

-- Unprocessed records:
WHERE processed_at IS NULL

-- All benefit from partial indexes!
```

### Practice 4: Update Statistics After Data Distribution Changes

**Problem: Optimizer Doesn't Know About Partial Index Selectivity**

```sql
CREATE INDEX idx_orders_active ON orders(status)
WHERE status IN ('pending', 'processing');

-- Initially: 1M pending orders (1% selectivity)
-- Statistics reflect: 1% selectivity

-- 6 months later: Business grows, 10M pending orders (10% selectivity)
-- BUT: Statistics stale, still show 1% selectivity

-- Query:
SELECT * FROM orders WHERE status = 'pending';

-- Optimizer decision (with stale stats):
-- "Index covers 1% of data (based on old stats) → Use index"
-- Reality: Index covers 10% of data → Seq scan might be faster!

-- Result: Wrong plan chosen, poor performance
```

**Solution: Regular ANALYZE**
```sql
-- After significant data changes:
ANALYZE orders;

-- Schedule in maintenance window:
-- Daily ANALYZE for high-churn tables
SELECT cron.schedule('analyze-orders', '0 3 * * *', 'ANALYZE orders');

-- Or tune autovacuum/autoanalyze:
ALTER TABLE orders SET (
    autovacuum_analyze_threshold = 50,
    autovacuum_analyze_scale_factor = 0.01  -- Analyze after 1% change
);

-- Verify statistics:
SELECT
    tablename,
    last_analyze,
    last_autoanalyze,
    n_live_tup,
    n_mod_since_analyze
FROM pg_stat_user_tables
WHERE tablename = 'orders';

-- If n_mod_since_analyze > 10% of n_live_tup: Run ANALYZE
```

---

## 5. Common Mistakes

### Mistake 1: Partial Index WHERE Doesn't Match Query WHERE

**Problem:**
```sql
-- Partial index:
CREATE INDEX idx_orders_recent ON orders(order_id)
WHERE created_at > '2024-01-01';

-- Query A:
SELECT * FROM orders 
WHERE order_id = 12345 
  AND created_at > '2024-02-01';  ← Stricter condition (Feb > Jan)

-- Planner: Query implies index WHERE (Feb > Jan implies > Jan 1) → Use index ✅

-- Query B:
SELECT * FROM orders
WHERE order_id = 12345
  AND created_at > '2023-06-01';  ← Broader condition (June < Jan)

-- Planner: Query does NOT imply index WHERE (June > 2023 does NOT imply > Jan 2024) → Cannot use index ❌

-- Result: Index not used, seq scan instead!
```

**Solution: Match Query Patterns**
```sql
-- Analyze actual query patterns:
-- 80% of queries: WHERE created_at > NOW() - INTERVAL '30 days'
-- 15% of queries: WHERE created_at > NOW() - INTERVAL '90 days'
-- 5% of queries: All-time queries

-- Create partial index matching 80% case:
CREATE INDEX idx_orders_recent ON orders(created_at, order_id)
WHERE created_at > NOW() - INTERVAL '30 days';

-- Update daily (since NOW() is static at index creation):
-- Option 1: Use static date, rebuild monthly
CREATE INDEX idx_orders_recent ON orders(created_at, order_id)
WHERE created_at > '2024-02-01';  -- Rebuild on 2024-03-01

-- Option 2: Use expression (PostgreSQL 11+)
CREATE INDEX idx_orders_recent ON orders(created_at, order_id)
WHERE created_at > CURRENT_DATE - INTERVAL '30 days';
-- Warning: This recreates index daily implicitly (not recommended)

-- Best: Use fixed date, rebuild periodically
```

### Mistake 2: Creating Too Many Partial Indexes

**Problem: Over-Optimization**
```sql
-- Developer creates partial index for every query:

-- Query A (50% traffic):
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'pending';

-- Query B (30% traffic):
CREATE INDEX idx_orders_processing ON orders(created_at) WHERE status = 'processing';

-- Query C (15% traffic):
CREATE INDEX idx_orders_shipped ON orders(created_at) WHERE status = 'shipped';

-- Query D (5% traffic):
CREATE INDEX idx_orders_cancelled ON orders(created_at) WHERE status = 'cancelled';

-- Problems:
-- - 4 indexes to maintain (INSERT must check predicate for each)
-- - Most queries covered by first 2 indexes (pending + processing)
-- - Last 2 indexes rarely used (5-15% traffic not worth overhead)
-- - Write amplification: 1-4 index updates per INSERT (probabilistic)
```

**Solution: Consolidate Common Patterns**
```sql
-- Combine pending + processing (covers 80% of queries):
CREATE INDEX idx_orders_active ON orders(status, created_at)
WHERE status IN ('pending', 'processing');

-- For shipped/cancelled: Use full index or accept seq scan (infrequent queries)
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- Total: 2 indexes (vs 4)
-- Coverage: 100% of queries
-- Write overhead: Lower (fewer predicate checks)
```

### Mistake 3: Partial Index on Frequently Changing Predicate Column

**Problem:**
```sql
-- Partial index:
CREATE INDEX idx_orders_active ON orders(order_id)
WHERE status IN ('pending', 'processing');

-- Business logic: Orders transition states frequently
-- INSERT: status='pending' → Index updated ✅
-- UPDATE: status='pending' → 'processing' → Index stays (both in IN clause)
-- UPDATE: status='processing' → 'shipped' → Index must DELETE entry! ❌

-- High churn:
-- 10M transitions/day from 'processing' → 'shipped'
-- Each transition: Remove from partial index → Causes index page updates, potential fragmentation

-- Performance:
-- - UPDATE time: 5ms → 15ms (3× slower due to index deletion)
-- - Index fragmentation: 30% empty space after 1 week
-- - Need frequent REINDEX: Weekly rebuilds required
```

**Solution: Avoid Partial Indexes on High-Churn Predicates**
```sql
-- Option 1: Use full index (accept larger size)
CREATE INDEX idx_orders_status_id ON orders(status, order_id);
-- Slightly larger, but no DELETE overhead

-- Option 2: Use different predicate (immutable column)
CREATE INDEX idx_orders_pending_date ON orders(created_at)
WHERE created_at > NOW() - INTERVAL '7 days';
-- Indexes recent orders (correlates with 'pending' status)
-- created_at never changes (no DELETE from index)

-- Option 3: Separate mutable and immutable data
CREATE TABLE orders_metadata (order_id, status);  -- Mutable
CREATE TABLE orders_data (order_id, ...);         -- Immutable

CREATE INDEX idx_orders_meta_active ON orders_metadata(status)
WHERE status IN ('pending', 'processing');
-- Small table, partial index OK
```

### Mistake 4: Partial Index with Non-Deterministic WHERE

**Problem:**
```sql
-- Non-deterministic function in WHERE:
CREATE INDEX idx_orders_recent ON orders(order_id)
WHERE created_at > NOW();

-- Error: cannot use non-immutable function in index predicate

-- Why?
-- NOW() returns different value each second
-- Index would need to rebuild every second to stay correct!

-- Attempted workaround:
CREATE INDEX idx_orders_recent ON orders(order_id)
WHERE created_at > (SELECT MAX(created_at) - INTERVAL '30 days' FROM orders);

-- Error: cannot use subquery in index predicate

-- These are invalid in PostgreSQL (and most databases)
```

**Solution: Use Constants or Immutable Expressions**
```sql
-- Option 1: Static date (rebuild periodically)
CREATE INDEX idx_orders_recent ON orders(order_id, created_at)
WHERE created_at > '2024-02-01';

-- Rebuild monthly with new date:
DROP INDEX idx_orders_recent;
CREATE INDEX idx_orders_recent ON orders(order_id, created_at)
WHERE created_at > '2024-03-01';

-- Option 2: Use CURRENT_DATE (immutable within transaction)
CREATE INDEX idx_orders_recent ON orders(order_id, created_at)
WHERE created_at > CURRENT_DATE - 30;

-- Note: CURRENT_DATE is immutable (doesn't change during index build)
-- But index predicate still fixed after creation (doesn't update daily)

-- Option 3: Partitioning (instead of partial index)
CREATE TABLE orders (
    order_id INT,
    created_at DATE,
    ...
) PARTITION BY RANGE (created_at);

-- Drop old partitions instead of partial index:
DROP TABLE orders_2023_01;  -- Archive old data
-- Recent partitions automatically "filtered" by partition pruning
```

---

## 6. Security Considerations

### 1. Information Leakage via Index Existence

**Risk:**
```sql
-- Partial index reveals business logic:
CREATE INDEX idx_users_premium ON users(user_id)
WHERE subscription_tier = 'premium';

-- Attacker queries system catalog:
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'users';

-- Results:
-- idx_users_premium | WHERE subscription_tier = 'premium'

-- Inference:
-- "Company has premium subscription tier"
-- "Database optimized for premium user queries" → They're important/profitable

-- Exploitation:
-- - Target premium users (phishing, social engineering)
-- - Competitor intelligence (pricing tiers)
```

**Mitigation:**
- Restrict catalog access: `REVOKE SELECT ON pg_indexes FROM PUBLIC;`
- Obfuscate index names: `idx_001` instead of `idx_users_premium`
- Document indexes outside database (internal wiki, not in index names)

### 2. Timing Attacks Reveal Predicate

**Risk:**
```sql
-- Partial index:
CREATE INDEX idx_orders_highvalue ON orders(order_id)
WHERE total > 10000;

-- Attacker sends queries:
-- Query A:
SELECT * FROM orders WHERE order_id = 12345;  -- Assume total = $50
-- Response: 50ms (seq scan or different index)

-- Query B:
SELECT * FROM orders WHERE order_id = 67890;  -- Assume total = $15K
-- Response: 1ms (partial index used → Fast!)

-- Inference: Order 67890 is high-value (>$10K)
-- Attacker learns: Sensitive financial information through timing
```

**Mitigation:**
- Constant-time queries (add artificial delay)
- Use full index (not partial) for sensitive predicates
- Rate limiting + CAPTCHA for external queries

---

## 7. Performance Optimization

### Optimization 1: Partial Index on Partition

**Combine Partitioning + Partial Indexes**

```sql
-- Scenario: 1B orders, partitioned by year
CREATE TABLE orders (
    order_id BIGINT,
    user_id BIGINT,
    status VARCHAR(20),
    created_at DATE
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2023 PARTITION OF orders 
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders 
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Query pattern: 98% of queries target currentyear + active orders

-- Strategy: Partial index only on 2024 partition
CREATE INDEX idx_orders_2024_active 
ON orders_2024(status, created_at)
WHERE status IN ('pending', 'processing');

-- No partial index on 2023 partition (historical data, rare queries)

-- Benefits:
-- - 2024 partition: 100M rows, partial index 1M rows = 10MB
-- - 2023 partition: 900M rows, no index = 0MB
-- - Total index storage: 10MB (vs 10GB for full index on all partitions)
-- - Query performance: Partition pruning + partial index = double filtering!

-- Query:
SELECT * FROM orders
WHERE status = 'pending' AND created_at > '2024-02-01';

-- Plan:
-- 1. Partition pruning: Only scan orders_2024 partition (900M rows excluded)
-- 2. Partial index: Only scan 'pending' rows within partition (99M rows excluded)
-- 3. Date filter: created_at > '2024-02-01' (further filtering)
-- Result: Scans 50K rows (from 1B), 20,000× fewer rows!
```

### Optimization 2: Partial Index with Included Columns

**Triple Combo: Partial + Covering + Composite**

```sql
--Query (hot path, 10K/sec):
SELECT order_id, user_id, total, items
FROM orders
WHERE status = 'pending'
  AND created_at > NOW() - INTERVAL '24 hours'
ORDER BY created_at DESC
LIMIT 100;

-- Optimized partial index:
CREATE INDEX idx_orders_pending_24h_covering
ON orders(created_at DESC)
INCLUDE (order_id, user_id, total, items)
WHERE status = 'pending' 
  AND created_at > CURRENT_DATE - 1;  -- Last 24 hours

-- Benefits:
-- 1. Partial: Only 10K rows (pending + last 24h)
-- 2. Covering: No heap access (all columns in index)
-- 3. Sorted: created_at DESC (no sort phase needed)
-- Result: Index size 500KB, query time 0.2ms, 100% cache hit ✅

-- Maintenance: Rebuild daily (since CURRENT_DATE changes):
-- Scheduled job:
DROP INDEX idx_orders_pending_24h_covering;
CREATE INDEX idx_orders_pending_24h_covering
ON orders(created_at DESC)
INCLUDE (order_id, user_id, total, items)
WHERE status = 'pending' 
  AND created_at > CURRENT_DATE - 1;
```

### Optimization 3: Expression-Based Partial Index

**Index Computed Expressions with Filtering**

```sql
-- Table: users
-- Column: email (lowercase and uppercase mixed)
-- Query: Case-insensitive email lookup

SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- Problem: Function on column prevents index usage

-- Solution: Expression index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- But: Index all 10M emails (300MB)

-- Optimization: Partial expression index (active users only)
CREATE INDEX idx_users_active_email_lower 
ON users(LOWER(email))
WHERE deleted_at IS NULL;

-- Index size: 9.9M rows × 30 bytes = 297MB (vs 300MB, slight savings)

-- Better: Combine with email domain filtering
CREATE INDEX idx_users_gmail_lower
ON users(LOWER(email))
WHERE deleted_at IS NULL 
  AND email LIKE '%@gmail.com';

-- Index size: 3M Gmail users × 30 bytes = 90MB (70% smaller!)
-- Use case: If 80% of queries are Gmail lookups (common for consumer apps)
```

---

## 8. Examples

### Example 1: SaaS Multi-Tenant Database

**Scenario: 10K tenants, 1 billion records total**

```sql
CREATE TABLE events (
    event_id BIGINT,
    tenant_id INT,
    event_type VARCHAR(50),
    data JSONB,
    created_at TIMESTAMPTZ
);

-- Distribution:
-- - 10 "whale" tenants: 500M events (50% of data)
-- - 990 "regular" tenants: 490M events (49% of data)
-- - 9000 "trial" tenants: 10M events (1% of data)

-- Query pattern:
-- - Whale tenants: Dedicated queries, use tenant_id index
-- - Regular tenants: Standard queries
-- - Trial tenants: 90% of queries (checking recent activity)

-- Partial index for trial tenants:
CREATE INDEX idx_events_trial_recent
ON events(tenant_id, created_at DESC)
WHERE tenant_id >= 10000  -- Trial tenant IDs start at 10000
  AND created_at > CURRENT_DATE - 30;  -- Last 30 days

-- Index size: 10M events × 20 bytes = 200MB
-- vs full index: 1B events × 20 bytes = 20GB (100× smaller!)

-- Benefits:
-- - Trial tenant queries: Fast (200MB index in cache)
-- - Regular/whale queries: Use different indexes
-- - Storage savings: 19.8GB saved
```

---

### Example 2: GitHub - Pull Request Listing

**Scenario: Open PRs vs Merged/Closed PRs**

```sql
CREATE TABLE pull_requests (
    pr_id BIGINT,
    repo_id BIGINT,
    state VARCHAR(20),  -- 'open', 'merged', 'closed'
    title VARCHAR(255),
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ
);

-- Distribution:
-- - open: 5M PRs (1%)
-- - merged: 450M PRs (90%)
-- - closed: 45M PRs (9%)

-- Query pattern:
-- - List open PRs: 95% of queries
-- - List merged/closed: 5% of queries (historical research)

-- Partial index for open PRs:
CREATE INDEX idx_prs_open 
ON pull_requests(repo_id, updated_at DESC)
INCLUDE (pr_id, title, created_at)
WHERE state = 'open';

-- Index size: 5M × 300 bytes = 1.5GB
-- vs full index: 500M × 300 bytes = 150GB (100× smaller!)

-- Query:
SELECT pr_id, title, created_at, updated_at
FROM pull_requests
WHERE repo_id = 12345 
  AND state = 'open'
ORDER BY updated_at DESC
LIMIT 30;

-- Plan: Index-only scan on idx_prs_open
-- Performance: 0.5ms (vs 50ms with full index)
-- Speedup: 100× FASTER!
```

---

## 9. Real-World Use Cases

### Use Case 1: Stripe - Active Subscriptions

**Context:**
- Millions of subscriptions
- States: active, canceled, paused, past_due
- 95% of queries target active subscriptions

**Implementation:**
```sql
CREATE INDEX idx_subscriptions_active 
ON subscriptions(customer_id, current_period_end)
INCLUDE (plan_id, status, amount)
WHERE status = 'active';

-- Benefits:
-- - Index tiny (5% of subscriptions)
-- - Dashboard queries instant (<1ms)
-- - Billing queries fast (active subscriptions only)
-- - Write efficiency: Canceled subscriptions removed from index automatically
```

### Use Case 2: Twitter - Recent Tweets Index

**Context:**
- Billions of tweets
- Timeline queries: Last 7 days of tweets from followed users

**Implementation:**
```sql
CREATE INDEX idx_tweets_recent 
ON tweets(user_id, created_at DESC)
INCLUDE (tweet_id, text, media_urls)
WHERE created_at > CURRENT_DATE - 7;

-- Rebuild daily (drop old tweets from index):
-- Cron job at midnight:
REINDEX INDEX CONCURRENTLY idx_tweets_recent;

-- Benefits:
-- - Index covers 1% of tweets (last 7 days)
-- - Timeline queries instant (all data in small index)
-- - Old tweets excluded (not needed for timelines)
-- - Storage: 99% savings vs full index
```

---

## 10. Interview Questions

### Q1: When should you use a partial index instead of a full index?

**Senior Answer:**

```
**Use Partial Index When:**

1. **Skewed data distribution + Skewed query pattern**
   - Example: 1% of rows are 'pending' status, 95% of queries filter on 'pending'
   - Partial index: 100× smaller, covers 95% of queries
   - Full index: 100× larger, covers 100% of queries (5% extra not worth cost)

2. **High write volume on non-target rows**
   - Example: 10K inserts/sec, but only 1% are 'pending' (99 queries)
   - Partial index: 100 index updates/sec (1% of inserts)
   - Full index: 10K index updates/sec (every insert)
   - Write throughput: 100× faster with partial

3. **Cache efficiency critical**
   - Example: 16GB buffer pool, full index 20GB, partial index 200MB
   - Partial: 100% cache hit rate (entire index in memory)
   - Full: 80% cache hit rate (frequent evictions)
   - Query latency: P99 reduced 90% (no cold page reads)

4. **Storage constraints**
   - Example: SSD space expensive, partial index 99% smaller
   - Savings: $50K/year in storage costs (cloud)

**Avoid Partial Index When:**

1. **Query patterns unpredictable**
   - Example: Ad-hoc analytics, different WHERE clauses each query
   - Partial index: Covers 20% of queries (predicate doesn't match)
   - Full index: Covers 100% of queries

2. **Subset size >50% of table**
   - Example: Partial index on 60% of rows
   - Savings: Only 40% smaller (not worth complexity)
   - Rule: Partial index useful when subset <10% of rows

3. **Predicate column frequently updated**
   - Example: status changes 10/sec  → Rows added/removed from index → Fragmentation
   - Full index: Stays in index (no DELETE/INSERT, just heap update)

4. **Maintenance complexity unacceptable**
   - Partial index with NOW(): Needs daily rebuild
   - Full index: Set and forget

**Real-world Decision Matrix:**

| Scenario | Subset Size | Query Coverage | Write:Read Ratio | Index Type | Reason |
|----------|-------------|----------------|------------------|------------|--------|
| Active orders | 1% | 95% | 1:100 | Partial ✅ | Small subset, high coverage |
| Recent orders (30d) | 20% | 80% | 1:50 | Partial ⚠️ | Medium subset, OK if cache-critical |
| Delivered orders | 80% | 10% | 1:10 | Full ❌ | Large subset, low coverage |
| Premium users | 5% | 90% | 1:200 | Partial ✅ | Small subset, high coverage |

**Interview Example:**

"At my previous company, our orders table had 100M rows: 1M active, 99M historical. Dashboard queries (5K/sec) only targeted active orders. We had a full index (500MB) that was partlyevicted from our 256MB buffer pool, causing 40% cache misses and P99 latency of 50ms.

We created a partial index WHERE status IN ('pending','processing'), size 5MB. This fit entirely in cache (100% hit rate), reducing P99 to 2ms (25× improvement). The 5% of queries needing historical data used a different index (acceptable slower performance for rare queries).

Storage savings: 495MB. Performance gain: 25×. Maintenance: Same as full index (actually easier—fewer splits). Clear win."
```

---

### Q2: How does the optimizer decide whether to use a partial index?

**Staff Answer:**

```
**Optimizer Decision Process:**

**Step 1: Predicate Implication Check**

The query WHERE clause must logically IMPLY the index WHERE clause.

```
Mathematical implication: Query → Index
If Query is true, then Index must be true.

Examples:

Index WHERE: status = 'pending'
Query WHERE: status = 'pending' AND user_id = 12345
Implication: YES ✅
  If status='pending' AND user_id=12345, then status='pending' is true.
  Use index: Yes

Query WHERE: status IN ('pending', 'processing')
Implication: NO ❌
  status IN (...) could be 'processing', which does NOT imply status='pending'
  Use index: No

Index WHERE: created_at > '2024-01-01'
Query WHERE: created_at > '2024-02-01'
Implication: YES ✅
  If created_at > Feb 1, then created_at > Jan 1 is true.
  Use index: Yes

Query WHERE: created_at > '2023-06-01'
Implication: NO ❌
  created_at > Jun 2023 does NOT imply created_at > Jan 2024 (could be Nov 2023)
  Use index: No
```

**Step 2: Cost Estimation**

If implication check passes, optimizer estimates cost:

```
Cost factors:
1. Index size (smaller = fewer pages to scan)
2. Selectivity (% of index rows matching query)
3. Heap fetch cost (if not covering index)

Example:

Query: SELECT * FROM orders WHERE status = 'pending' AND user_id = 12345;

Option A: Partial index idx_pending WHERE status='pending'
  - Index size: 1M rows (1% of table)
  - Selectivity: user_id=12345 matches 10 rows in 1M → 0.001%
  - Index scan: 1M rows × 0.001% = 10 rows
  - Cost: log(1M) + 10 heap fetches = 20 + 10 = 30

Option B: Full index idx_status_user ON (status, user_id)
  - Index size: 100M rows (entire table)
  - Selectivity: status='pending' AND user_id=12345 → 10 rows
  - Index scan: Navigate to (pending, 12345) directly → 10 rows
  - Cost: log(100M) + 10 heap fetches = 27 + 10 = 37

Option A wins (partial index): Lower cost (30 < 37)
Reason: Smaller index → More cache-friendly → More likely in memory

Caveat: If partial index not in cache and full index is:
  Option A: 30 (disk read penalty: +100) = 130
  Option B: 37 (cached) = 37
  Option B wins!
```

**Step 3: Statistics Accuracy**

Optimizer relies on statistics (pg_stats, pg_class):

```
Stale statistics can misguide optimizer:

Real data: 10M pending orders (10% of table)
Statistics: 1M pending orders (1%, outdated)

Optimizer thinks:
  "Partial index covers 1% → Very selective → Fast scan"

Reality:
  Partial index covers 10% → Less selective → Could be slower than expected

Solution: Regular ANALYZE
ANALYZE orders;
-- Updates statistics → Optimizer makes better decisions
```

**Step 4: Hint/Force Index (Emergency)**

If optimizer chooses wrong index, force usage:

```sql
-- PostgreSQL (requires pg_hint_plan):
/*+ IndexScan(orders idx_pending) */
SELECT * FROM orders WHERE status = 'pending';

-- MySQL:
SELECT * FROM orders FORCE INDEX (idx_pending) WHERE status = 'pending';

-- SQL Server:
SELECT * FROM orders WITH (INDEX(idx_pending)) WHERE status = 'pending';

-- Warning: Hints override optimizer → Use sparingly (optimizer usually right)
```

**Real-World Debugging:**

```sql
-- Check if partial index is being used:
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-02-01';

-- Look for:
Index Scan using idx_pending ON orders  ← Using partial index ✅
  Index Cond: (status = 'pending' AND created_at > '2024-02-01')
  Rows: 10
  Buffers: shared hit=5

-- If not using partial index:
Seq Scan on orders  ← Not using index! ❌
  Filter: (status = 'pending' AND created_at > '2024-02-01')

-- Debugging steps:
-- 1. Check implication: Does query WHERE imply index WHERE?
SELECT '2024-02-01'::date > '2024-01-01'::date;  -- true → Implication holds

-- 2. Check statistics:
SELECT * FROM pg_stats WHERE tablename = 'orders' AND attname = 'status';
-- If null_frac or n_distinct way off → Run ANALYZE

-- 3. Check index validity:
SELECT * FROM pg_index WHERE indexrelid = 'idx_pending'::regclass;
-- If indisvalid = false → Index broken, REINDEX needed

-- 4. Force index (temporary workaround):
/*+ IndexScan(orders idx_pending) */
-- If this works → Optimizer issue (stale stats or cost miscalculation)
-- If this doesn't help → Index genuinely not useful for this query
```

**Interview Insight:**

"In a production incident, queries suddenly stopped using our partial index idx_orders_active (WHERE status IN ('pending','processing')). EXPLAIN showed Seq Scan instead of Index Scan.

Investigation:
1. Implication check: Query had WHERE status='pending' → Implies index predicate ✅
2. Statistics: Ran pg_stats query → n_distinct showed 5 (correct), but most_common_freqs showed [0.9, 0.05, ...] (90% pending, 5% processing). Reality: Shift in business led to 50/50 split.
3. Root cause: Stale statistics made optimizer think idx_orders_active covered 95% of rows → Too large → seq scan cheaper.

Fix: ANALYZE orders; → Statistics updated → Optimizer saw 55% coverage (still good) → Used partial index again.

Lesson: Even with correct implication, stale statistics can break optimizer decisions. ANALYZE regularly!"
```

---

## 11. Summary

### Partial Index Key Principles

1. **Index Subset of Rows**
   - WHERE clause filters which rows included
   - Only matching rows stored in index
   - Size: N × (fraction matching predicate)

2. **Query Must Imply Index Predicate**
   - Query WHERE → Index WHERE (logical implication)
   - Optimizer checks implication before using index
   - Example: Query `status='pending'` implies index `status IN ('pending','processing')` → NO

3. **Ideal for Skewed Data + Skewed Queries**
   - 1% of rows, 95% of queries → Perfect use case
   - 50% of rows, 50% of queries → Marginal benefit
   - Rule: Subset <10% of rows for maximum benefit

4. **Write Efficiency Bonus**
   - Inserts skip index if predicate doesn't match
   - Example: 99% of inserts skip partial index → 100× fewer index writes

### When to Use Partial Indexes

**✅ Create Partial Index:**
- Subset <10% of table rows (smaller = better)
- >80% of queries filter on same predicate (high coverage)
- Read-heavy workload (>10:1 read:write ratio)
- Cache-critical queries (small index fits in memory)
- Stable predicate (column rarely updated, avoiding fragmentation)

**❌ Avoid Partial Index:**
- Subset >50% of table (marginal savings)
- <50% query coverage (many queries can't use index)
- Write-heavy predicate column (high churn → fragmentation)
- Non-deterministic WHERE (NOW(), random(), subqueries)
- Unknown query patterns (ad-hoc analytics)

### Performance Characteristics

| Metric | Full Index | Partial Index (1% subset) | Improvement |
|--------|------------|---------------------------|-------------|
| Index size | 1GB | 10MB | 100× smaller |
| Cache hit rate | 50% (partial in cache) | 100% (entire index cached) | 2× fewer cold reads |
| INSERT time | 5ms (always update) | 0.5ms (99% skip update) | 10× faster writes |
| Query time | 2ms | 0.5ms | 4× faster |
| Storage cost | $10/month | $0.10/month | 100× cheaper |

### Implementation Checklist

**Creating Partial Indexes:**
- [ ] Identify skewed column (1-10% of rows match condition)
- [ ] Verify query pattern (>80% of queries filter on same condition)
- [ ] Write WHERE clause matching query pattern
- [ ] Ensure WHERE uses immutable values (no NOW(), subqueries)
- [ ] Create index: `CREATE INDEX ... WHERE ...`
- [ ] Run ANALYZE to update statistics
- [ ] Verify usage: `EXPLAIN` should show partial index

**Monitoring:**
- [ ] Check index usage: `pg_stat_user_indexes.idx_scan > 100`
- [ ] Verify implication: Query WHERE implies index WHERE
- [ ] Monitor predicate selectivity (should remain <10%)
- [ ] Run ANALYZE after data distribution changes
- [ ] Watch for fragmentation if predicate column updated frequently

### Quick Wins

**1. Partial index on sparse subset (1 hour, 100× smaller)**
```sql
-- Find sparse subsets:
SELECT status, COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS pct
FROM orders
GROUP BY status
ORDER BY pct;

-- Create partial index on <10% subsets frequently queried:
CREATE INDEX idx_orders_active ON orders(status, created_at)
WHERE status IN ('pending', 'processing');
```

**2. Combine partial + covering (1 hour, ultra-fast queries)**
```sql
CREATE INDEX idx_name ON table(key_col)
INCLUDE (select_list_cols)
WHERE predicate;
```

**3. Update statistics after creation (5 min, ensure optimizer awareness)**
```sql
ANALYZE table_name;
```

### Database-Specific Syntax

**PostgreSQL:**
```sql
CREATE INDEX idx_name ON table(column) WHERE predicate;
```

**SQL Server:**
```sql
CREATE INDEX idx_name ON table(column) WHERE predicate;
-- Same syntax as PostgreSQL ✅
```

**MySQL:**
```sql
-- No native partial indexes ❌
-- Workaround: Triggers + separate table
```

**Oracle:**
```sql
-- No native partial indexes ❌
-- Workaround: Function-based index with CASE
CREATE INDEX idx_name ON table(CASE WHEN predicate THEN column END);
```

### Key Takeaway

**"Partial indexes are THE optimization for skewed workloads. Index the 1%, query the 1%, save 99% storage and 10× query time. But only if queries match the predicate—otherwise, it's wasted complexity."**

Example decision: 100M orders, 1M active orders (1%), 95% of queries target active orders.
- Full index: 1GB storage, 50% cache hit, 2ms queries
- Partial index: 10MB storage, 100% cache hit, 0.2ms queries
- Savings: 990MB storage, 10× faster queries, 100× fewer index writes
- Decision: Partial index clear win! ✅

---

**Next:** `06_Composite_Indexes.md` - Multi-column index design and column ordering strategies

