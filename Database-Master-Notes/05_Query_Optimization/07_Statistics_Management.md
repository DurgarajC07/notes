# Statistics Management - Keeping the Optimizer Smart

## 1. Concept Explanation

**Statistics** are metadata about your data that the query optimizer uses to estimate query costs and choose execution plans. Think of statistics as the optimizer's "eyes"—without accurate statistics, it's blind, making random guesses instead of informed decisions.

### What Statistics Track

```
Per-Table Statistics:
├─ Row count (n_live_tup)
├─ Dead tuples (n_dead_tup)
├─ Table size (pages, avg_row_length)
└─ Last analyze timestamp

Per-Column Statistics:
├─ Number of distinct values (n_distinct)
├─ NULL fraction (null_frac)
├─ Average column width (avg_width)
├─ Most common values (MCV) and frequencies
├─ Histogram buckets (distribution of values)
└─ Correlation (physical vs logical ordering)
```

### How Optimizer Uses Statistics

```sql
-- Query:
SELECT * FROM orders WHERE status = 'pending';

-- Optimizer's thought process:
-- 1. How many rows in orders? → n_live_tup: 1,000,000
-- 2. How many distinct values in status? → n_distinct: 5
-- 3. How common is 'pending'? → MCV: 10% of rows
-- 4. Estimated result: 1,000,000 × 0.10 = 100,000 rows
-- 5. Cost of index scan: 100,000 × 4.0 (random_page_cost) = 400,000
-- 6. Cost of seq scan: 10,000 pages × 1.0 (seq_page_cost) = 10,000
-- 7. Decision: Sequential scan cheaper ✅
```

**Without statistics:**
```
-- Optimizer guesses: Maybe 50% of rows? Or 1%? Who knows?
-- Random plan selection
-- 50% chance of catastrophically slow query
```

---

## 2. Why It Matters

### Production Impact

**Stale Statistics = Wrong Plans**

```sql
-- Table grew from 10K to 10M rows (1000× growth)
-- Statistics last updated 6 months ago (still thinks it's 10K)

SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at >= '2024-01-01';

-- Optimizer thinks:
-- - orders: 10K rows (WRONG! Actually 10M)
-- - Result after filter: 1K rows
-- - Choose: Nested Loop Join (good for small result sets)

-- Reality:
-- - orders: 10M rows
-- - Result after filter: 5M rows
-- - Nested Loop: 5M × 200K iterations (catastrophic!)

-- Time: 45 minutes (vs 5 seconds with correct plan)
```

**After ANALYZE:**
```sql
ANALYZE orders;

-- Now optimizer knows:
-- - orders: 10M rows ✅
-- - Result after filter: 5M rows
-- - Choose: Hash Join (correct for large result sets)

-- Time: 5 seconds ✅
```

### Real Scenario: E-commerce Black Friday

**Background:**
- Normal traffic: 100 orders/hour
- Black Friday: 10,000 orders/hour (100× spike)
- Auto-ANALYZE runs every 24h (not triggered yet)

**Incident Timeline:**

**00:00 (Black Friday starts):**
```sql
-- Statistics still say: 50K orders total
-- Reality: 500K orders (10× growth in 1 day)

-- Dashboard query:
SELECT COUNT(*), AVG(total), SUM(total)
FROM orders
WHERE created_at >= CURRENT_DATE;

-- Plan (based on stale stats):
-- Estimated rows: 100 (normal day)
-- Actual rows: 10,000 (Black Friday)
-- Index scan chosen (wrong for 10K rows)

-- Time: 15 seconds (vs normal 0.5s)
-- Dashboard: Timeouts, customer complaints
```

**04:00 (Auto-ANALYZE runs):**
```sql
-- Auto-ANALYZE triggers (threshold reached)
ANALYZE orders;

-- Statistics updated: 500K orders ✅
-- New plan: Sequential scan (correct)
-- Time: 0.6 seconds ✅

-- Dashboard: Back to normal
```

**Lesson:** Statistics lag reality. Critical period: growth spike before auto-ANALYZE.

---

## 3. Internal Working

### ANALYZE Algorithm

```
ANALYZE Process:
1. Sample random rows (default: 30,000 rows per table)
   ↓
2. For each column:
   a. Count distinct values
   b. Identify NULL fraction
   c. Find most common values (MCV)
   d. Build histogram for value distribution
   ↓
3. Calculate correlation (physical vs logical order)
   ↓
4. Store in system catalogs (pg_stats, sys.stats, etc.)
   ↓
5. Optimizer uses stats for future queries
```

### Histogram Construction

**Purpose:** Estimate rows for range queries

```sql
-- Column: price (100K rows)
-- Values: $1 to $10,000

-- Histogram (10 buckets):
Bucket 1:  $1     - $1,000   (10% of rows)
Bucket 2:  $1,000 - $2,000   (8% of rows)
Bucket 3:  $2,000 - $3,000   (12% of rows)
Bucket 4:  $3,000 - $4,000   (15% of rows)
Bucket 5:  $4,000 - $5,000   (20% of rows)  ← Most common
Bucket 6:  $5,000 - $6,000   (10% of rows)
Bucket 7:  $6,000 - $7,000   (8% of rows)
Bucket 8:  $7,000 - $8,000   (7% of rows)
Bucket 9:  $8,000 - $9,000   (5% of rows)
Bucket 10: $9,000 - $10,000  (5% of rows)

-- Query: WHERE price BETWEEN $4,000 AND $6,000
-- Estimate: Bucket 5 (20%) + Bucket 6 (10%) = 30% of rows
-- Result: 100K × 0.30 = 30,000 rows ✅
```

**Without histogram:**
```
-- Uniform distribution assumption:
-- $4,000-$6,000 is 20% of range ($1-$10,000)
-- Estimate: 100K × 0.20 = 20,000 rows
-- Actual: 30,000 rows (50% error!)
```

### Most Common Values (MCV)

**Purpose:** Track frequent outliers

```sql
-- Column: country (1M users)
-- Distribution:
--   US: 500K (50%)
--   UK: 200K (20%)
--   CA: 100K (10%)
--   Others: 200K countries, <1K users each (20% total)

-- MCV list (top 10):
'US': 0.50
'UK': 0.20
'CA': 0.10
'DE': 0.03
'FR': 0.02
'AU': 0.02
'JP': 0.02
'IN': 0.02
'BR': 0.02
'MX': 0.02

-- Query: WHERE country = 'US'
-- Estimate: 1M × 0.50 = 500K rows (from MCV) ✅

-- Query: WHERE country = 'Andorra'
-- Not in MCV, estimate from n_distinct: 1M / 200 = 5K rows ✅
```

### Correlation

**Purpose:** Estimate cost of index scan vs seq scan

```sql
-- Correlation: How well physical storage order matches logical order

-- Perfect correlation (+1.0):
-- Orders inserted chronologically, clustered by created_at
-- Index scan on created_at: Sequential I/O (fast)

created_at on disk: [2024-01-01, 2024-01-02, 2024-01-03, ...]
                     ↓ Sequential reads

-- No correlation (0.0):
-- Orders inserted randomly
-- Index scan on created_at: Random I/O (slow)

created_at on disk: [2024-03-15, 2024-01-08, 2024-02-20, ...]
                     ↓ Random jumps

-- Negative correlation (-1.0):
-- Orders inserted in reverse chronological order

-- Optimizer uses correlation to adjust I/O cost:
-- High correlation → Lower cost (sequential I/O)
-- Low correlation → Higher cost (random I/O)
```

---

## 4. Best Practices

### Practice 1: Run ANALYZE After Bulk Changes

**❌ Bad: Bulk Insert Without ANALYZE**
```sql
-- Import 10M rows
COPY orders FROM '/data/orders.csv';

-- Statistics still think: 50K rows (before import)
-- Optimizer: Makes plans for 50K rows
-- Reality: 10.05M rows
-- Result: All queries slow until next auto-ANALYZE
```

**✅ Good: ANALYZE After Bulk Operations**
```sql
-- Import 10M rows
COPY orders FROM '/data/orders.csv';

-- Update statistics immediately
ANALYZE orders;

-- Now optimizer knows: 10.05M rows ✅
-- Queries use correct plans immediately
```

**When to Manual ANALYZE:**
- After bulk INSERT/COPY (>5% table growth)
- After bulk DELETE (>10% rows removed)
- After bulk UPDATE to indexed columns
- After CREATE INDEX
- Before major reports/queries (data warehouse)

### Practice 2: Increase Statistics Target for Critical Columns

**❌ Bad: Default Statistics (100 buckets)**
```sql
-- customer_id has extreme skew:
--   99% of customers: 1-50 orders
--   1% VIP customers: 1K-100K orders
-- Default histogram (100 buckets): Can't capture outliers

-- Query:
WHERE customer_id = 99999;  -- VIP with 100K orders

-- Optimizer estimate: 50 orders (average)
-- Actual: 100K orders (2000× off!)
-- Wrong plan chosen
```

**✅ Good: Increase Statistics Target**
```sql
-- Increase histogram buckets for skewed column
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;

-- Re-analyze with higher detail
ANALYZE orders;

-- Check results:
SELECT
    tablename,
    attname,
    n_distinct,
    array_length(most_common_vals, 1) as n_mcv,
    histogram_bounds
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'customer_id';

-- histogram_bounds: Now 1000 buckets (vs 100 before)
-- MCV list: Now includes VIP customers ✅

-- Query with VIP customer:
WHERE customer_id = 99999;
-- Optimizer estimate: 98,500 orders (from detailed histogram)
-- Actual: 100K orders (1.5% error, acceptable!)
-- Correct plan chosen ✅
```

**Statistics Target Guidelines:**
- Default: 100 (sufficient for most columns)
- Moderately skewed: 200-500
- Highly skewed: 1000-10000
- Cost: Higher target = longer ANALYZE time, more storage

**Check if higher target helps:**
```sql
-- Before:
SELECT COUNT(*) FROM orders WHERE customer_id = 99999;
-- EXPLAIN: Estimated rows: 50, Actual: 100,000 (2000× off)

-- Increase target:
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
ANALYZE orders;

-- After:
SELECT COUNT(*) FROM orders WHERE customer_id = 99999;
-- EXPLAIN: Estimated rows: 98,500, Actual: 100,000 (1.5% error) ✅
```

### Practice 3: Monitor Auto-ANALYZE Triggers

**PostgreSQL Auto-ANALYZE Threshold:**
```
Trigger when:
  (n_dead_tup + n_ins_since_last_analyze) > (autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × n_live_tup)

Defaults:
  autovacuum_analyze_threshold = 50
  autovacuum_analyze_scale_factor = 0.1

Example (1M row table):
  Trigger = 50 + (0.1 × 1,000,000) = 100,050 changes
  → Auto-ANALYZE runs after 100K inserts/deletes
```

**Problem: Large tables take long to trigger**
```sql
-- Table: 10M rows
-- Auto-ANALYZE trigger: 50 + (0.1 × 10M) = 1,000,050 changes

-- Black Friday: +500K orders (growth)
-- Auto-ANALYZE: Hasn't triggered yet (need 1M changes)
-- Statistics: Stale for hours
```

**Solution: Adjust Scale Factor for Large Tables**
```sql
-- Per-table auto-ANALYZE tuning
ALTER TABLE orders SET (autovacuum_analyze_scale_factor = 0.01);  -- 1% instead of 10%

-- Now triggers after:
-- 50 + (0.01 × 10M) = 100,050 changes (100K instead of 1M)

-- Black Friday: +500K orders
-- Auto-ANALYZE: Triggers after first 100K (much sooner!) ✅
```

**Monitor Auto-ANALYZE Activity:**
```sql
-- PostgreSQL: Check last analyze timestamp
SELECT
    schemaname,
    tablename,
    n_live_tup,
    n_dead_tup,
    n_mod_since_analyze,
    last_analyze,
    last_autoanalyze,
    CASE
        WHEN n_live_tup > 0 THEN
            round(100.0 * n_mod_since_analyze / n_live_tup, 2)
        ELSE 0
    END as pct_changed_since_analyze
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_mod_since_analyze DESC;

-- Look for:
-- - pct_changed_since_analyze > 20% (stale stats)
-- - last_autoanalyze more than 1 day ago (large table)
```

### Practice 4: Use Extended Statistics for Correlated Columns

**Problem: Optimizer Assumes Column Independence**
```sql
-- Table: orders (1M rows)
-- Columns:
--   country: 'US' (50%), 'UK' (30%), 'CA' (20%)
--   payment_method: 'credit_card' (60%), 'paypal' (40%)

-- Reality: Strong correlation:
--   US customers: 90% credit_card, 10% paypal
--   UK customers: 30% credit_card, 70% paypal
--   CA customers: 50% credit_card, 50% paypal

-- Query:
WHERE country = 'UK' AND payment_method = 'credit_card';

-- Optimizer assumption (independence):
-- Selectivity = P(country='UK') × P(payment='credit_card')
--             = 0.30 × 0.60 = 0.18
-- Estimated rows: 1M × 0.18 = 180,000

-- Reality (correlated):
-- UK customers with credit_card: 30% × 30% = 9%
-- Actual rows: 1M × 0.09 = 90,000

-- Estimate 2× off → Wrong plan (nested loop vs hash join)
```

**Solution: Extended Statistics (PostgreSQL 10+)**
```sql
-- Create extended statistics to capture correlation
CREATE STATISTICS orders_country_payment_stats (dependencies)
ON country, payment_method
FROM orders;

-- Re-analyze
ANALYZE orders;

-- Check extended stats:
SELECT
    stxname,
    stxkeys,
    stxdependencies
FROM pg_statistic_ext
WHERE stxname = 'orders_country_payment_stats';

-- Now query:
WHERE country = 'UK' AND payment_method = 'credit_card';

-- Optimizer uses dependency information:
-- Estimated rows: 95,000 (vs 180,000 before) ✅
-- Actual rows: 90,000 (5% error, acceptable!)
```

**Types of Extended Statistics:**

**1. Dependencies (functional dependencies)**
```sql
-- When one column determines another
CREATE STATISTICS user_stats (dependencies)
ON country, timezone
FROM users;

-- Example: country='US' → timezone highly correlated
```

**2. N-Distinct (multivariate distinct counts)**
```sql
-- When combination of columns has fewer distinct values than expected
CREATE STATISTICS order_stats (ndistinct)
ON user_id, product_id
FROM orders;

-- Example: Each user orders few distinct products (not all combinations)
```

**3. MCV (multivariate most common values)**
```sql
-- Track frequent value combinations
CREATE STATISTICS order_stats (mcv)
ON country, payment_method, status
FROM orders;

-- Example: ('US', 'credit_card', 'completed') is very common
```

---

## 5. Common Mistakes

### Mistake 1: Never Running Manual ANALYZE

**Problem:**
```sql
-- Data warehouse: Nightly ETL job
-- 1. Delete old aggregates (1M rows)
-- 2. Insert new aggregates (2M rows)
-- 3. Run reports immediately

-- Statistics after step 1: Still show 10M rows (before delete)
-- Reality: 9M rows
-- Reports: Use wrong plans, slow

-- Statistics after step 2: Still show 9M rows
-- Reality: 11M rows
-- Reports: Still use wrong plans
```

**Solution:**
```sql
-- ETL script:
BEGIN;
  DELETE FROM daily_aggregates WHERE date < CURRENT_DATE - 30;
  ANALYZE daily_aggregates;  ← After DELETE

  INSERT INTO daily_aggregates SELECT ...;
  ANALYZE daily_aggregates;  ← After INSERT
COMMIT;

-- Now reports use correct statistics ✅
```

### Mistake 2: Running ANALYZE Too Frequently

**Problem:**
```sql
-- Overzealous script:
while true; do
    psql -c "ANALYZE orders;"  -- Every second!
    sleep 1;
done

-- Impact:
-- - ANALYZE takes 5 seconds for large table
-- - Locks table (briefly) during sampling
-- - I/O overhead (reads random pages)
-- - No benefit (data doesn't change every second)
```

**Solution:**
```sql
-- Reasonable schedule:
-- - OLTP tables (frequent writes): Every 5-15 minutes
-- - Data warehouse (batch loads): After each load
-- - Reporting tables (read-heavy): Daily or weekly

-- PostgreSQL: Set auto-ANALYZE, don't add manual cron
-- Only manual ANALYZE after bulk operations
```

**ANALYZE Cost:**
- Samples 30K rows (default)
- Random I/O for sampling
- Brief lock (ACCESS SHARE, allows reads)
- Time: ~100ms to 10s depending on table size

### Mistake 3: Not Checking Statistics Accuracy

**Problem:**
```sql
-- Assume ANALYZE fixes everything, never verify

-- Query slow, check plan:
EXPLAIN SELECT * FROM orders WHERE status = 'pending';

-- Output:
-- Estimated rows: 100
-- Actual rows: 500,000 (5000× off!)

-- Statistics are wrong, but nobody checks!
```

**Solution: Regular Statistics Audit**
```sql
-- Check estimation vs reality for important queries
EXPLAIN (ANALYZE, VERBOSE) [your query];

-- Look for large discrepancies:
-- - Estimated rows: 100, Actual: 100,000 (1000× off: RED FLAG)
-- - Estimated rows: 1000, Actual: 1100 (10% off: acceptable)

-- If large discrepancy:
-- 1. Check n_distinct: SELECT n_distinct FROM pg_stats WHERE ...
-- 2. Increase statistics target: ALTER TABLE ... SET STATISTICS 1000;
-- 3. Create extended statistics for correlated columns
-- 4. Re-ANALYZE
-- 5. Verify improvement
```

**Automated Monitoring:**
```sql
-- Compare estimated vs actual for recent queries
-- (Requires pg_stat_statements)

SELECT
    query,
    calls,
    mean_exec_time,
    stddev_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- Queries > 1s
ORDER BY mean_exec_time DESC
LIMIT 20;

-- For each slow query:
-- - Run EXPLAIN (ANALYZE)
-- - Check estimated vs actual rows
-- - If 10× off: Statistics problem
```

### Mistake 4: Using Default Statistics Target for Everything

**Problem:**
```sql
-- All columns use default (100 buckets):
-- - user_id: Uniform distribution (100 buckets enough)
-- - created_at: Uniform distribution (100 buckets enough)
-- - product_id: Power-law distribution (100 buckets NOT enough)

-- Query:
WHERE product_id = 12345;  -- Top 1% product

-- Estimated: 50 rows (average)
-- Actual: 50,000 rows (1000× off, product is outlier)
```

**Solution: Tune Per Column**
```sql
-- Identify skewed columns:
SELECT
    tablename,
    attname,
    n_distinct,
    null_frac,
    correlation,
    array_length(most_common_vals, 1) as n_mcv
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY n_distinct DESC;

-- For columns with high n_distinct + skew (power-law):
ALTER TABLE orders ALTER COLUMN product_id SET STATISTICS 1000;

-- For uniform distribution (no skew):
-- Keep default (100)

-- Re-analyze:
ANALYZE orders;
```

---

## 6. Security Considerations

### 1. Statistics Expose Data Distribution

**Risk:**
```sql
-- Read-only user can infer sensitive information
SELECT
    tablename,
    attname,
    most_common_vals,
    most_common_freqs
FROM pg_stats
WHERE tablename = 'users';

-- Output shows:
-- Column: salary
-- most_common_vals: {50000, 60000, 75000, 90000, 150000}
-- most_common_freqs: {0.30, 0.25, 0.20, 0.15, 0.05}

-- Attacker learns:
-- - 30% of users earn $50K
-- - Only 5% earn $150K+
-- - Salary distribution exposed
```

**Mitigation:**
```sql
-- Restrict access to statistics views
REVOKE SELECT ON pg_stats FROM public;
REVOKE SELECT ON pg_statistic FROM public;

-- Grant only to DBAs:
GRANT SELECT ON pg_stats TO dba_role;
```

### 2. Statistics Query Timing Attacks

**Risk:**
```sql
-- Attacker can infer data existence via query timing

-- Query 1:
SELECT * FROM users WHERE email = 'admin@company.com';
-- Fast (0.01ms) if doesn't exist (index shows immediately)
-- Slower (0.1ms) if exists (fetches row)

-- Attacker enumerates valid emails by timing
```

**Mitigation:**
- Constant-time lookups for sensitive data
- Rate limiting
- Query result caching (constant time regardless)

---

## 7. Performance Optimization

### Optimization 1: Parallel ANALYZE

**Problem: Large Table ANALYZE is Slow**
```sql
-- Table: 100M rows
-- ANALYZE time: 5 minutes (samples 30K rows randomly)

-- During ANALYZE: Table locked (ACCESS SHARE, allows reads but blocks vacuums)
```

**Solution: Partition Table, Analyze Partitions in Parallel**
```sql
-- PostgreSQL 11+: Partitioned table
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    created_at TIMESTAMP,
    ...
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ...12 partitions total

-- Analyze partitions in parallel:
-- Terminal 1:
ANALYZE orders_2024_01;

-- Terminal 2:
ANALYZE orders_2024_02;

-- Terminal 3:
ANALYZE orders_2024_03;

-- ...

-- Total time: 30 seconds (vs 5 minutes serial)
-- 10× FASTER!

-- Note: Don't run too many in parallel (I/O contention)
-- Rule of thumb: Max 4 parallel ANALYZE
```

### Optimization 2: Incremental Statistics (Oracle)

**Problem: Full Table ANALYZE Expensive**
```sql
-- Table: 1B rows
-- ANALYZE time: 1 hour

-- 1% of table changed (10M rows)
-- Re-analyzing entire 1B rows wasteful
```

**Solution: Incremental Statistics (Oracle)**
```sql
-- Enable incremental statistics
EXEC DBMS_STATS.SET_TABLE_PREFS('schema', 'orders', 'INCREMENTAL', 'TRUE');

-- Initial analyze (expensive):
EXEC DBMS_STATS.GATHER_TABLE_STATS('schema', 'orders');

-- Later: Only analyze changed partitions:
EXEC DBMS_STATS.GATHER_TABLE_STATS('schema', 'orders');
-- Skips unchanged partitions automatically
-- Time: 5 minutes (vs 1 hour) ✅
```

### Optimization 3: Sampling Rate Tuning

**Default Sampling:**
```sql
-- PostgreSQL: 30,000 rows (fixed sample size)
-- For 100M row table: 0.03% sample rate

-- Advantage: Fast
-- Disadvantage: May miss outliers in small groups
```

**Increase Sample Size:**
```sql
-- Increase sample size for better accuracy
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
-- Now samples more for histogram (1000 buckets vs 100)

-- Or: Increase default statistics target globally
ALTER SYSTEM SET default_statistics_target = 200;  -- Default: 100
-- Doubles sample size for all columns
```

**Trade-off:**
- Higher target: More accurate, slower ANALYZE
- Lower target: Faster ANALYZE, less accurate

**Guidelines:**
- OLTP: default_statistics_target = 100 (default, fast)
- Data warehouse: default_statistics_target = 200-500 (accuracy matters)
- Specific skewed columns: SET STATISTICS 1000-10000

---

## 8. Examples

### Example 1: Fixing Cardinality Mismatch with Extended Statistics

**Scenario:** Multi-tenant SaaS, queries filtered by tenant_id + date

**Problem:**
```sql
-- Table: events (100M rows)
-- 1000 tenants, heavily skewed:
--   - Tenant 1: 50M events (50%, enterprise customer)
--   - Tenant 2-10: 40M events (40%, mid-market)
--   - Tenant 11-1000: 10M events (10%, small businesses)

-- Query:
SELECT event_type, COUNT(*)
FROM events
WHERE tenant_id = 1
  AND event_date >= '2024-01-01'
GROUP BY event_type;

-- Optimizer assumption (independence):
-- - Selectivity(tenant_id=1): 1/1000 = 0.001
-- - Selectivity(date >= 2024-01-01): 50% (half the data)
-- - Combined: 0.001 × 0.5 = 0.0005
-- - Estimated rows: 100M × 0.0005 = 50,000

-- WRONG! Reality:
-- - Tenant 1 has 50M events (50% of table, not 0.1%)
-- - Last 30 days: 45M events (tenant 1 is very active recently)
-- - Actual rows: 45M (900× more than estimate!)

-- Chosen plan: Nested Loop Join (disaster for 45M rows)
-- Time: 45 minutes ❌
```

**Fix: Extended Statistics**
```sql
-- Step 1: Create extended statistics
CREATE STATISTICS events_tenant_date_stats (dependencies, ndistinct)
ON tenant_id, event_date
FROM events;

-- Step 2: Increase statistics target for tenant_id (capture outliers)
ALTER TABLE events ALTER COLUMN tenant_id SET STATISTICS 1000;

-- Step 3: Re-analyze
ANALYZE events;

-- Step 4: Verify
EXPLAIN (ANALYZE) SELECT ...;

-- Output:
Hash Aggregate (cost=500000.00..600000.00 rows=100) (actual time=8500.123..8600.456 rows=25)
  Group Key: event_type
  -> Seq Scan on events (cost=0.00..450000.00 rows=45000000)
       Filter: (tenant_id = 1 AND event_date >= '2024-01-01')

-- Estimated rows: 45M ✅ (was 50K before)
-- Actual rows: 45M ✅
-- Correct plan chosen: Sequential Scan + Hash Aggregate
-- Time: 8.6 seconds ✅ (was 45 minutes before, 314× FASTER!)
```

---

### Example 2: Auto-ANALYZE Tuning for Fast-Growing Table

**Scenario:** Real-time analytics table, 1M events/hour

**Problem:**
```sql
-- Table: realtime_events
-- Current size: 100M rows
-- Growth: +1M rows/hour (24/7)

-- Auto-ANALYZE threshold:
-- 50 + (0.1 × 100M) = 10,000,050 changes
-- → Triggers after 10M changes ≈ 10 hours

-- During 10-hour window:
-- - Statistics show 100M rows
-- - Reality: 110M rows (10% stale)
-- - Queries use outdated plans
-- - Dashboards slow
```

**Fix: Adjust Auto-ANALYZE Scale Factor**
```sql
-- Step 1: Lower scale factor (trigger more frequently)
ALTER TABLE realtime_events SET (
    autovacuum_analyze_scale_factor = 0.01,  -- 1% instead of 10%
    autovacuum_analyze_threshold = 50
);

-- Now triggers after:
-- 50 + (0.01 × 100M) = 1,000,050 changes ≈ 1 hour

-- Step 2: Monitor auto-ANALYZE activity
SELECT
    schemaname,
    tablename,
    n_live_tup,
    n_mod_since_analyze,
    last_autoanalyze,
    autovacuum_count,
    autoanalyze_count
FROM pg_stat_user_tables
WHERE tablename = 'realtime_events';

-- Expected: autoanalyze_count increases every hour

-- Step 3: Verify query plans stay accurate
-- (Run hourly check with EXPLAIN ANALYZE)
```

**Result:**
- Statistics updated every hour (vs every 10 hours)
- Query plans stay accurate
- Dashboard stays fast ✅

---

## 9. Real-World Use Cases

### Use Case 1: Data Warehouse ETL Pipeline

**Scenario:** Nightly ETL loads 50M rows

**Strategy:**
```sql
-- ETL script:

-- 1. Drop old partitions (fast, no ANALYZE needed)
DROP TABLE IF EXISTS sales_2023_01;

-- 2. Load new data
CREATE TABLE sales_2024_02 (LIKE sales_template);
COPY sales_2024_02 FROM '/data/sales_2024_02.csv';

-- 3. ANALYZE immediately after load
ANALYZE sales_2024_02;

-- 4. Create indexes
CREATE INDEX idx_sales_2024_02_date ON sales_2024_02(sale_date);
CREATE INDEX idx_sales_2024_02_product ON sales_2024_02(product_id);

-- 5. Attach partition
ALTER TABLE sales ATTACH PARTITION sales_2024_02
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- 6. ANALYZE parent table (updates partition metadata)
ANALYZE sales;

-- Now queries use accurate statistics for new partition ✅
```

**Without ANALYZE:**
- Queries on new partition use default stats (1000 rows assumed)
- Wrong plans chosen
- Reports 100× slower

### Use Case 2: Multi-Tenant Query Optimization

**Scenario:** SaaS with 10K tenants, query: "tenant's orders last 30 days"

**Problem: Optimizer Doesn't Know Tenant Sizes**
```sql
-- Query:
SELECT * FROM orders
WHERE tenant_id = ?
  AND created_at >= CURRENT_DATE - 30;

-- Tenant sizes vary wildly:
-- - Small tenants: 10 orders/month
-- - Enterprise tenants: 100K orders/month

-- Single statistics for all tenants:
-- - Avg: 5000 orders per tenant
-- - Optimizer always chooses plan for 5000 rows
-- - Wrong for both small (overkill) and enterprise (underkill)
```

**Solution: Extended Statistics + Multiple Query Variants**
```sql
-- Step 1: Extended statistics
CREATE STATISTICS orders_tenant_stats (dependencies, ndistinct, mcv)
ON tenant_id, created_at
FROM orders;

ALTER TABLE orders ALTER COLUMN tenant_id SET STATISTICS 1000;

ANALYZE orders;

-- Step 2: Check tenant sizes, create separate query variants
-- Small tenants (<1000 orders/month):
SELECT * FROM orders
WHERE tenant_id = ?
  AND created_at >= CURRENT_DATE - 30;
-- Plan: Index scan (good for small result)

-- Enterprise tenants (>50K orders/month):
SELECT /*+ SeqScan(orders) */ * FROM orders
WHERE tenant_id = ?
  AND created_at >= CURRENT_DATE - 30;
-- Plan: Sequential scan (better for large result)

-- Application logic chooses variant based on tenant tier ✅
```

---

## 10. Interview Questions

### Q1: How would you diagnose why a query is choosing the wrong execution plan?

**Staff Answer:**

```
**Step-by-step diagnosis:**

**1. Verify plan is actually wrong**
```sql
EXPLAIN (ANALYZE, BUFFERS) [slow query];

-- Compare estimated vs actual:
-- Node: Nested Loop
--   → Estimated rows: 100
--   → Actual rows: 500,000 (5000× off: PROBLEM!)
--   → Actual time: 45231.456ms (slow)
```

**2. Check statistics freshness**
```sql
-- When were statistics last updated?
SELECT
    schemaname,
    tablename,
    n_live_tup,
    n_mod_since_analyze,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE tablename IN ('orders', 'customers');

-- Red flags:
-- - last_autoanalyze more than 1 day ago (for active table)
-- - n_mod_since_analyze > 20% of n_live_tup (stale)
-- - n_live_tup much smaller/larger than expected (growth/shrinkage)
```

**3. Check column statistics accuracy**
```sql
-- Examine the filtered columns
SELECT
    attname,
    n_distinct,
    null_frac,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats
WHERE tablename = 'orders'
  AND attname IN ('customer_id', 'status', 'created_at');

-- Look for:
-- - n_distinct = -1 (estimated, not measured accurately)
-- - most_common_vals missing important values
-- - histogram_bounds not covering data range
```

**4. Test with fresh statistics**
```sql
-- Run ANALYZE
ANALYZE orders;
ANALYZE customers;

-- Re-run EXPLAIN (ANALYZE)
-- Did estimates improve?
-- Is plan better now?
```

**5. Check for correlated columns**
```sql
-- Query filters on multiple columns:
WHERE status = 'completed' AND country = 'US'

-- Are these correlated?
-- - If independent: P(A AND B) = P(A) × P(B)
-- - If correlated: P(A AND B) ≠ P(A) × P(B)

-- Check correlation:
SELECT
    COUNT(*) as total,
    COUNT(*) FILTER (WHERE status = 'completed') as completed,
    COUNT(*) FILTER (WHERE country = 'US') as us,
    COUNT(*) FILTER (WHERE status = 'completed' AND country = 'US') as completed_us
FROM orders;

-- Calculate:
-- Expected (if independent): (completed/total) × (us/total) × total
-- Actual: completed_us
-- If 2× difference: Columns are correlated!

-- Fix: Extended statistics
CREATE STATISTICS orders_status_country_stats (dependencies)
ON status, country FROM orders;
ANALYZE orders;
```

**6. Check for outliers/skew**
```sql
-- Does query target outlier?
-- Example: Customer #99999 has 1M orders (outlier)

-- Check distribution:
SELECT customer_id, COUNT(*)
FROM orders
GROUP BY customer_id
ORDER BY COUNT(*) DESC
LIMIT 10;

-- If top customers have 1000× more orders than average:
-- → Increase statistics target
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
ANALYZE orders;
```

**7. Check query plan cache**
```sql
-- PostgreSQL prepared statements cache plans
-- Cached plan may be stale

-- Check if plan is cached:
SELECT * FROM pg_prepared_statements;

-- Invalidate cache:
DEALLOCATE ALL;
-- Or restart connection

-- Re-test query
```

**Real-world example:**

"At my previous company, we had a multi-tenant app where queries suddenly became 100× slower for one enterprise customer. Investigation revealed:

1. EXPLAIN showed 5000× row estimate miss (thought 100 rows, actually 500K)
2. last_autoanalyze was 3 days ago (customer onboarded 1M users over weekend)
3. Ran ANALYZE manually → Fixed immediately
4. Root cause: autovacuum was disabled during data migration, never re-enabled
5. Permanent fix: Re-enabled autovacuum, lowered scale_factor for multi-tenant tables"
```

### Q2: Explain extended statistics and when to use them.

**Principal Answer:**

```
**Extended statistics** capture correlations between columns that standard statistics miss.

**Problem: Independence Assumption**

By default, optimizer assumes columns are independent:

```sql
WHERE status = 'completed' AND country = 'US'

-- Optimizer calculates:
-- P(status='completed' AND country='US')
--   = P(status='completed') × P(country='US')
--   = 0.90 × 0.50 = 0.45 (independence assumption)

-- Reality: US orders complete faster (correlation):
-- P(status='completed' | country='US') = 0.95
-- Actual: 0.95 × 0.50 = 0.475

-- 5% error (acceptable)
```

But with strong correlation:

```sql
-- Reality: 99% of US orders complete, 10% of international orders complete

-- Optimizer (independence):
-- P(completed AND US) = 0.55 × 0.50 = 0.275 (27.5% of rows)

-- Reality (correlated):
-- P(completed AND US) = 0.99 × 0.50 = 0.495 (49.5% of rows)

-- 80% error (catastrophic!) → Wrong plan
```

**Extended Statistics Types:**

**1. Dependencies (functional dependencies)**
```sql
CREATE STATISTICS user_location_stats (dependencies)
ON country, city, zip_code
FROM users;

-- Captures: country → city → zip_code
-- If country='US', city='Seattle', zip_code must be 98XXX

-- Query: WHERE country = 'US' AND city = 'Seattle'
-- Without: Estimate 0.02 × 0.001 = 0.00002 (0.002% of rows)
-- With: Uses dependency, estimate 0.002% (closer to reality)
```

**2. N-Distinct (multivariate distinct counts)**
```sql
CREATE STATISTICS product_user_stats (ndistinct)
ON product_id, user_id
FROM purchases;

-- Standard assumption:
-- - 1000 distinct products
-- - 10,000 distinct users
-- - Combinations: 1000 × 10,000 = 10M (assumes all combinations possible)

-- Reality:
-- - Each user purchases from 5-10 products (not all 1000)
-- - Actual distinct combinations: 50,000 (not 10M)

-- Query: GROUP BY product_id, user_id
-- Without: Estimates 10M groups (allocates huge hash table)
-- With: Estimates 50K groups (correct hash table size) ✅
```

**3. MCV (multivariate most common values)**
```sql
CREATE STATISTICS order_status_stats (mcv)
ON status, payment_method, country
FROM orders;

-- Captures frequent combinations:
-- ('completed', 'credit_card', 'US'): 35%
-- ('completed', 'paypal', 'UK'): 15%
-- ('pending', 'bank_transfer', 'DE'): 5%
-- ...

-- Query: WHERE status='completed' AND payment='credit_card' AND country='US'
-- Without: 0.90 × 0.60 × 0.50 = 0.27 (27% estimate)
-- With: Uses MCV list, 0.35 (35% estimate) ✅
```

**When to use extended statistics:**

**Use when:**

1. **Queries filter on multiple columns**
```sql
WHERE col1 = ? AND col2 = ? AND col3 = ?
```

2. **EXPLAIN shows large row estimate misses (10× or more)**
```sql
-- Estimated rows: 10,000
-- Actual rows: 150,000 (15× off)
```

3. **Columns have known correlation**
   - Geography: (country, state, city)
   - Time: (year, month, day)
   - Product: (category, subcategory, brand)
   - Multi-tenant: (tenant_id, <any other column>)

4. **GROUP BY multiple columns with few actual groups**
```sql
GROUP BY product_id, user_id
-- Expected: 10M groups (product × user cardinality)
-- Actual: 50K groups (each user buys few products)
```

**Don't use when:**

1. **Columns are truly independent**
   - user_id and random_id (no correlation)

2. **Single-column queries**
   - Extended stats don't help

3. **OLTP high-write tables**
   - Extended stats increase ANALYZE time
   - Write overhead (maintaining additional statistics)

**Creating extended statistics:**

```sql
-- PostgreSQL 10+:
CREATE STATISTICS stats_name (dependencies, ndistinct, mcv)
ON col1, col2, col3
FROM table_name;

-- Re-analyze to populate:
ANALYZE table_name;

-- Verify:
SELECT stxname, stxkeys, stxkind, stxdependencies, stxndistinct
FROM pg_statistic_ext
WHERE stxrelid = 'table_name'::regclass;
```

**Real-world example:**

"We had a multi-tenant analytics platform where queries filtered by tenant_id + date_range. Optimizer consistently estimated 10K rows, but enterprise customers had 5M rows for the same query. Created extended statistics on (tenant_id, event_date), and errors dropped from 500× to 1.5×. Query went from timing out (60s) to completing in 2s by choosing hash join instead of nested loop."
```

---

## 11. Summary

### Statistics Management Checklist

- [ ] **After bulk operations:** Run ANALYZE immediately
- [ ] **High-skew columns:** Increase statistics target (1000+)
- [ ] **Correlated columns:** Create extended statistics
- [ ] **Fast-growing tables:** Lower autovacuum_analyze_scale_factor (0.01)
- [ ] **Query plan issues:** Check estimated vs actual rows (EXPLAIN ANALYZE)
- [ ] **Regular audits:** Monitor last_autoanalyze timestamps
- [ ] **Partitioned tables:** ANALYZE parent after adding partitions

### Statistics Types

| Type | Purpose | When to Use |
|------|---------|-------------|
| **n_distinct** | # of unique values | Distinct counts, GROUP BY estimates |
| **MCV** | Most common values + frequencies | Skewed data (zipcode, country, status) |
| **Histogram** | Value distribution | Range queries (WHERE price BETWEEN) |
| **Correlation** | Physical vs logical ordering | Index scan cost estimation |
| **Extended (dependencies)** | Column correlations | Multi-column filters (country + city) |
| **Extended (ndistinct)** | Multivariate distinct counts | GROUP BY multiple columns |
| **Extended (mcv)** | Multivariate common values | Frequent value combinations |

### Auto-ANALYZE Tuning

**Default Threshold:**
```
50 + (0.1 × n_live_tup)
```

**Recommendations:**
- Small tables (<100K rows): Keep default (0.1)
- Medium tables (1M-10M rows): Lower to 0.05
- Large tables (>10M rows): Lower to 0.01
- Fast-growing tables: Lower to 0.005-0.01

**Check if tuning needed:**
```sql
SELECT
    tablename,
    n_live_tup,
    n_mod_since_analyze,
    round(100.0 * n_mod_since_analyze / NULLIF(n_live_tup, 0), 2) as pct_changed,
    last_autoanalyze,
    NOW() - last_autoanalyze as time_since_analyze
FROM pg_stat_user_tables
WHERE n_live_tup > 10000
  AND n_mod_since_analyze::float / NULLIF(n_live_tup, 0) > 0.20
ORDER BY n_mod_since_analyze DESC;
```

### Key Principles

1. **Statistics = Optimizer's Eyes**
   - Stale statistics → Blind optimizer → Wrong plans

2. **ANALYZE After Bulk Changes**
   - Bulk INSERT/DELETE/UPDATE: Always ANALYZE after
   - Don't rely on auto-ANALYZE (may take hours to trigger)

3. **Tune for Skewed Data**
   - Increase statistics target (100 → 1000)
   - Create extended statistics for correlated columns
   - Use partial indexes for extreme skew

4. **Monitor, Don't Assume**
   - Check estimated vs actual rows (EXPLAIN ANALYZE)
   - Automate statistics audits (weekly checks)
   - Alert on stale statistics (>20% changed since analyze)

5. **Balance Accuracy vs Cost**
   - Higher statistics target = more accurate, slower ANALYZE
   - Extended statistics = better estimates, more ANALYZE overhead
   - OLTP: Favor speed (default target)
   - OLAP: Favor accuracy (higher target)

**Remember:** Statistics are the foundation of query optimization. Wrong statistics → wrong plans → slow queries. Keep them fresh, detailed (for skewed data), and validated (EXPLAIN ANALYZE).

---

**Next Reading:**
- `08_N_Plus_One_Problem.md` - Application-level query patterns causing performance issues
- `06_Query_Hints.md` - When statistics alone aren't enough, force better plans
- `05_Index_Selection.md` - How statistics influence index selection
