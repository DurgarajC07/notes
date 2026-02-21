# Index Selection - Choosing the Right Index

## 1. Concept Explanation

**Index selection** is the process by which the query optimizer chooses which index (if any) to use for a given query. With multiple indexes on a table, the optimizer must decide: Which index provides the best access path?

Think of it like choosing between different maps to find a location: a street map,elevation map, or satellite view—each optimized for different types of searches.

### The Decision Process

```
Query: SELECT * FROM users WHERE email = 'user@example.com' AND country = 'US';

Available indexes:
1. idx_users_email (email)
2. idx_users_country (country)
3. idx_users_email_country (email, country)  ← Composite
4. idx_users_country_email (country, email)  ← Composite (different order)

Optimizer evaluates each:
- Cost of idx_users_email: 100 (selectivity: 1/1M = 0.000001)
- Cost of idx_users_country: 50000 (selectivity: 200K/1M = 0.2)
- Cost of idx_users_email_country: 5 (covers both columns, minimal heap access)
- Cost of idx_users_country_email: 150 (less selective prefix)

Winner: idx_users_email_country ✅
```

---

## 2. Why It Matters

### Production Impact

**Without Proper Index Selection:**
```sql
-- 3 indexes exist, but optimizer chooses wrong one
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at);
CREATE INDEX idx_orders_user ON orders(user_id);

SELECT * FROM orders 
WHERE status = 'completed' 
  AND created_at >= '2024-01-01'
  AND user_id = 12345;

-- Chooses idx_orders_status (wrong!)
-- Filters 990K rows (99% are 'completed')
-- Then scans 990K rows for other conditions
-- Time: 15 seconds
```

**With Optimal Index:**
```sql
-- Add composite index with most selective column first
CREATE INDEX idx_orders_user_created_status ON orders(user_id, created_at, status);

-- Now uses idx_orders_user_created_status
-- Filters to 50 rows immediately
-- No additional heap scans needed (covering index)
-- Time: 0.005 seconds (3000× FASTER!)
```

### Real Scenario: E-commerce Search

**Query:**
```sql
SELECT * FROM products
WHERE category_id = 5
  AND price BETWEEN 100 AND 500
  AND in_stock = true
ORDER BY rating DESC
LIMIT 20;
```

**Available Indexes:**
1. `idx_products_category` (category_id)
2. `idx_products_price` (price)
3. `idx_products_instock` (in_stock)
4. `idx_products_rating` (rating DESC)
5. `idx_products_category_price_stock` (category_id, price, in_stock)

**Bad Selection: idx_products_rating**
```
-- Uses rating index for ORDER BY
Index Scan using idx_products_rating on products  (cost=0.43..50000.00 rows=20)
  Filter: (category_id = 5 AND price >= 100 AND price <= 500 AND in_stock = true)
  Rows Removed by Filter: 980000  ← Scans 980K rows to find 20!

-- Reads all products sorted by rating, filters one by one
-- Time: 25 seconds
```

**Good Selection: idx_products_category_price_stock**
```
-- Uses composite index to filter efficiently
Limit  (cost=0.43..100.00 rows=20)
  ->  Sort  (cost=100.00..150.00 rows=5000)
        Sort Key: rating DESC
        ->  Index Scan using idx_products_category_price_stock on products
              Index Cond: (category_id = 5 AND price >= 100 AND price <= 500 AND in_stock = true)
              Rows: 5000  ← Filters to 5K first, then sorts

-- Filters first (5K rows), then sorts, then limits
-- Time: 0.2 seconds (125× FASTER!)
```

---

## 3. Internal Working

### Index Selection Algorithm

```
1. Parse query → Identify filterable columns
       ↓
2. Find candidate indexes
   - Single-column indexes on filterable columns
   - Multi-column indexes with matching prefix
   - Covering indexes
       ↓
3. Estimate cost for each index path
   - Selectivity of index columns
   - Cost of index scan vs heap access
   - Cost of additional filters
       ↓
4. Compare with sequential scan cost
       ↓
5. Choose lowest cost option
```

### Selectivity Calculation

**Formula:**
```
Selectivity = Rows matching condition / Total rows

Cost Estimate = (Index Access Cost) + (Heap Access Cost × Rows After Index)
```

**Example:**
```sql
-- Table: orders (1M rows)
-- Query: WHERE status = 'pending' AND user_id = 12345

-- Option 1: idx_orders_status
Selectivity(status='pending') = 10K / 1M = 0.01
Rows after index: 10K
Cost = (Index scan: 100) + (Heap scan: 10K × 4.0) = 40,100

-- Option 2: idx_orders_user_id
Selectivity(user_id=12345) = 50 / 1M = 0.00005
Rows after index: 50
Cost = (Index scan: 50) + (Heap scan: 50 × 4.0) = 250

-- Winner: idx_orders_user_id (250 < 40,100) ✅
```

### Composite Index Prefix Matching

**B-Tree Composite Index Structure:**
```
idx_users_country_city_age (country, city, age)

Lexicographic ordering:
('US', 'NYC', 25)
('US', 'NYC', 30)
('US', 'NYC', 35)
('US', 'SF', 25)
('US', 'SF', 30)
('UK', 'London', 25)
('UK', 'London', 30)
```

**Index Can Be Used For:**
```sql
-- ✅ Full prefix match
WHERE country = 'US' AND city = 'NYC' AND age = 30

-- ✅ Partial prefix match (left-aligned)
WHERE country = 'US' AND city = 'NYC'  -- Uses first 2 columns
WHERE country = 'US'  -- Uses first column

-- ✅ Range on last prefix column
WHERE country = 'US' AND city = 'NYC' AND age > 25

-- ❌ Skips prefix column (can't use index efficiently)
WHERE city = 'NYC' AND age = 30  -- No country filter

-- ❌ Range on non-last prefix column
WHERE country = 'US' AND city > 'M' AND age = 30  -- Can't use age column
```

**Leftmost Prefix Rule:** Composite index usable only if query filters on leftmost columns first.

---

## 4. Best Practices

### Practice 1: Order Composite Index Columns by Selectivity

**❌ Bad: Low Selectivity First**
```sql
-- Status has only 3 values: 'pending', 'completed', 'cancelled'
-- User_id has 1M unique values
CREATE INDEX idx_orders_status_user ON orders(status, user_id);

-- Query:
WHERE status = 'completed' AND user_id = 12345;

-- Index scan:
-- 1. Filter status='completed' → 990K rows (low selectivity)
-- 2. Filter user_id=12345 within those 990K → 50 rows
-- Cost: High (scans large intermediate result)
```

**✅ Good: High Selectivity First**
```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Query:
WHERE user_id = 12345 AND status = 'completed';

-- Index scan:
-- 1. Filter user_id=12345 → 50 rows (high selectivity)
-- 2. Filter status='completed' within those 50 → 45 rows
-- Cost: Low (small intermediate result)
```

**Selectivity Rule:** Order composite index columns from most selective to least selective.

**Exception:** If query always filters on low-selectivity column (e.g., status='pending'), put it first to enable index usage.

### Practice 2: Create Covering Indexes for Hot Queries

**❌ Bad: Index + Heap Access**
```sql
CREATE INDEX idx_products_category ON products(category_id);

SELECT id, name, price
FROM products
WHERE category_id = 5;

-- Plan:
Index Scan using idx_products_category on products  (cost=0.43..800.00 rows=5000)
  Index Cond: (category_id = 5)
  
-- Steps:
-- 1. Scan index: Find 5000 matching index entries
-- 2. For each entry: Random heap access to get (id, name, price)
-- Total: 5000 random I/Os (slow!)
```

**✅ Good: Covering Index**
```sql
CREATE INDEX idx_products_category_covering ON products(category_id) INCLUDE (name, price);
-- Or: CREATE INDEX ... ON products(category_id, name, price);

-- Plan:
Index Only Scan using idx_products_category_covering on products  (cost=0.43..300.00 rows=5000)
  Index Cond: (category_id = 5)
  
-- Steps:
-- 1. Scan index: Find 5000 matching entries
-- 2. All needed columns in index (no heap access!)
-- Total: Sequential index scan (2× FASTER!)
```

**Covering Index Benefits:**
- No heap access (faster)
- Less I/O (especially important for SSD/NVMe)
- Works even if table has many columns

**Trade-off:** Larger index size (more disk space, slower writes)

### Practice 3: Drop Redundant Indexes

**❌ Bad: Redundant Indexes**
```sql
-- These indexes are redundant:
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_email_country ON users(email, country);  ← Covers idx_users_email
CREATE INDEX idx_users_email_country_age ON users(email, country, age);  ← Covers both above

-- Maintenance overhead:
-- - Every INSERT updates 3 indexes (redundantly)
-- - Every UPDATE on email updates 3 indexes
-- - Extra disk space: 3× overhead
```

**✅ Good: Drop Redundant Indexes**
```sql
-- Keep only the most comprehensive index
DROP INDEX idx_users_email;
DROP INDEX idx_users_email_country;
CREATE INDEX idx_users_email_country_age ON users(email, country, age);  ← Keeps this

-- Query: WHERE email = 'user@example.com'
-- Still uses idx_users_email_country_age (leftmost prefix rule) ✅

-- Benefits:
-- - 1 index update per write instead of 3
-- - 66% less disk space
-- - Faster writes, same read performance
```

**Detection Query:**
```sql
-- PostgreSQL: Find redundant indexes
SELECT 
    indrelid::regclass AS table_name,
    indexrelid::regclass AS index_name,
    array_agg(attname ORDER BY attnum) AS columns
FROM pg_index i
JOIN pg_attribute a ON a.attrelid = i.indrelid AND a.attnum = ANY(i.indkey)
WHERE i.indrelid = 'users'::regclass
GROUP BY table_name, index_name, i.indexrelid
ORDER BY table_name, array_length(i.indkey, 1) DESC;

-- Look for indexes that are prefixes of others
```

### Practice 4: Monitor Index Usage

**Track Unused Indexes:**
```sql
-- PostgreSQL: Find indexes never used
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(indexrelid) DESC;

-- Drop unused indexes:
-- DROP INDEX idx_unused;
```

**Track Index Effectiveness:**
```sql
-- Indexes with low hit rate (scanning but not finding much)
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    CASE WHEN idx_tup_read > 0 
         THEN round(100.0 * idx_tup_fetch / idx_tup_read, 2)
         ELSE 0 
    END AS hit_rate_pct
FROM pg_stat_user_indexes
WHERE idx_scan > 100  -- Only consider used indexes
  AND idx_tup_read > 0
ORDER BY hit_rate_pct ASC, idx_scan DESC;

-- Low hit_rate_pct → Index scanned but few rows fetched → Possible inefficiency
```

---

## 5. Common Mistakes

### Mistake 1: Creating Index on Low-Selectivity Column

**Problem:**
```sql
-- gender column has only 3 distinct values: 'M', 'F', 'Other'
CREATE INDEX idx_users_gender ON users(gender);

-- Query:
SELECT * FROM users WHERE gender = 'F';

-- Plan:
Seq Scan on users  (cost=0.00..20000.00 rows=500000)
  Filter: (gender = 'F')

-- Index ignored! Why?
-- - 50% of rows match (low selectivity)
-- - Index scan cost: 4.0 × 500K random heap reads = 2,000,000
-- - Seq scan cost: 1.0 × 10K sequential page reads = 10,000
-- - Seq scan 200× cheaper!
```

**Solution:**
```sql
-- Drop useless index
DROP INDEX idx_users_gender;

-- If gender is frequently combined with other filters, use composite:
CREATE INDEX idx_users_gender_country ON users(gender, country);  -- country adds selectivity
```

**Selectivity Threshold:**
- Index useful when selectivity < 5-10% of table
- Above that, sequential scan usually faster

### Mistake 2: Wrong Column Order in Composite Index

**Problem:**
```sql
CREATE INDEX idx_orders_created_user ON orders(created_at, user_id);

-- Query 1: Uses index
WHERE created_at >= '2024-01-01' AND user_id = 12345;  ✅

-- Query 2: CANNOT use index efficiently
WHERE user_id = 12345;  ❌
-- Can't skip created_at column (leftmost prefix rule broken)

-- Plan:
Seq Scan on orders  (cost=0.00..50000.00 rows=50)
  Filter: (user_id = 12345)
```

**Solution:**
```sql
-- Analyze query patterns first
-- If 80% of queries filter by user_id:
DROP INDEX idx_orders_created_user;
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- Now both queries work:
WHERE user_id = 12345;  ✅ (uses first column)
WHERE user_id = 12345 AND created_at >= '2024-01-01';  ✅ (uses both)
```

**Rule:** Put most-frequently-queried column first, unless selectivity demands otherwise.

### Mistake 3: Over-Indexing (Write Amplification)

**Problem:**
```sql
-- Table with 10 indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_country ON users(country);
CREATE INDEX idx_users_city ON users(city);
CREATE INDEX idx_users_age ON users(age);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_users_updated ON users(updated_at);
CREATE INDEX idx_users_last_login ON users(last_login);
CREATE INDEX idx_users_email_verified ON users(email_verified);

-- INSERT performance:
-- Single row insert updates:
--   - 1 table heap page
--   - 10 index B-Tree pages (worst case)
-- Write amplification: 10×

-- Benchmark:
INSERT INTO users (...) VALUES (...);  -- 50ms per insert (slow!)
```

**Solution:**
```sql
-- Audit index usage
SELECT indexname, idx_scan FROM pg_stat_user_indexes WHERE tablename = 'users';

-- Results:
-- idx_users_email: 1,000,000 scans  ← Keep
-- idx_users_phone: 50,000 scans  ← Keep
-- idx_users_country: 10,000 scans  ← Keep
-- idx_users_age: 50 scans  ← Drop (rarely used)
-- idx_users_email_verified: 0 scans  ← Drop (never used)
-- ...

-- Drop low-value indexes
DROP INDEX idx_users_age;
DROP INDEX idx_users_email_verified;

-- New INSERT performance:
INSERT INTO users (...) VALUES (...);  -- 35ms per insert (1.4× faster)
```

**Guideline:**
- OLTP tables: 3-5 indexes typical, 10+ red flag
- Data warehouse: More indexes acceptable (read-heavy)

### Mistake 4: Not Using Partial Indexes

**Problem:**
```sql
-- Index includes deleted records (90% of table)
CREATE INDEX idx_users_country ON users(country);

-- Query only cares about active users:
SELECT * FROM users WHERE country = 'US' AND status = 'active';

-- Index includes 900K deleted rows (wasted space)
-- Index size: 50MB (90% waste)
```

**Solution:**
```sql
-- Partial index on active users only
CREATE INDEX idx_users_country_active ON users(country) WHERE status = 'active';

-- Now:
-- Index size: 5MB (10× smaller)
-- Faster scans (less data to read)
-- Faster writes (deleted users don't update index)

-- Query must include WHERE status = 'active' to use partial index
SELECT * FROM users WHERE country = 'US' AND status = 'active';  ✅
```

---

## 6. Security Considerations

### 1. Index Visibility Exposes Schema

**Risk:**
```sql
-- Read-only user can infer schema from indexes
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'public';

-- Output reveals:
-- idx_users_ssn → SSN column exists (sensitive!)
-- idx_users_salary → Salary column exists
-- idx_transactions_cc_last4 → Credit card data
```

**Mitigation:**
```sql
-- Restrict access to system catalogs
REVOKE SELECT ON pg_indexes FROM public;
REVOKE SELECT ON pg_stat_user_indexes FROM public;
```

### 2. Timing Attacks via Index Selection

**Risk:**
```sql
-- Different execution times reveal existence of data
-- With index on email:
SELECT * FROM users WHERE email = 'admin@company.com';
-- Fast (0.01ms) → Email exists in index

SELECT * FROM users WHERE email = 'nonexistent@example.com';
-- Even faster (0.001ms) → Not in index

-- Attacker can enumerate valid emails
```

**Mitigation:**
- Constant-time operations for sensitive lookups
- Rate limiting
- Add artificial delays

---

## 7. Performance Optimization

### Optimization 1: Expression Indexes

**Problem: Function Prevents Index Usage**
```sql
CREATE INDEX idx_users_email ON users(email);

-- Query with function:
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- Plan:
Seq Scan on users  (cost=0.00..25000.00 rows=5000)
  Filter: (lower(email) = 'user@example.com')

-- Index ignored! Function disables index usage.
```

**Solution: Function-Based Index**
```sql
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Now query uses index:
Index Scan using idx_users_email_lower on users  (cost=0.43..8.45 rows=1)
  Index Cond: (lower(email) = 'user@example.com')

-- 3000× faster!
```

**Common Expression Indexes:**
```sql
-- Case-insensitive text search
CREATE INDEX idx_products_name_lower ON products(LOWER(name));

-- Date truncation
CREATE INDEX idx_orders_date_trunc ON orders(DATE_TRUNC('day', created_at));

-- JSON field extraction
CREATE INDEX idx_metadata_user_id ON events((metadata->>'user_id'));

-- Computed columns
CREATE INDEX idx_products_discount_pct ON products((price * discount / 100));
```

### Optimization 2: Index Fill Factor for Write-Heavy Tables

**Problem: B-Tree Page Splits**
```sql
-- Default fill factor: 100% (pages completely full)
CREATE INDEX idx_users_email ON users(email);

-- INSERT new user with email = 'alice@example.com'
-- Page is full → Split page (expensive operation)
-- Result: Index fragmentation, slower writes
```

**Solution: Lower Fill Factor**
```sql
-- Set 80% fill factor (leaves 20% free space per page)
CREATE INDEX idx_users_email ON users(email) WITH (fillfactor = 80);

-- Now INSERT has room without page split
-- Trade-off: 20% larger index, but 50% faster writes
```

**When to Use:**
- Writeit-heavy tables (high INSERT rate)
- B-Tree indexes with random key distribution
- UUIDs, email addresses, random strings

**When NOT to Use:**
- Read-heavy tables (wasted space)
- Append-only data (sequential inserts don't cause splits)

### Optimization 3: Separate Indexes for Different Query Patterns

**Anti-Pattern: One Index for Everything**
```sql
-- Trying to satisfy all queries with one index
CREATE INDEX idx_orders_everything ON orders(user_id, status, created_at, total);

-- Query 1:
WHERE user_id = 12345;  ✅ Works

-- Query 2:
WHERE status = 'pending';  ❌ Can't use (user_id not filtered)

-- Query 3:
WHERE created_at >= '2024-01-01';  ❌ Can't use (user_id, status not filtered)
```

**Better: Targeted Indexes**
```sql
-- Analyze query patterns:
-- - 60% queries: WHERE user_id = ?
-- - 30% queries: WHERE created_at >= ? AND status = ?
-- - 10% queries: WHERE total > ? ORDER BY created_at

-- Create 2-3 targeted indexes:
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_status ON orders(created_at, status);
CREATE INDEX idx_orders_total_created ON orders(total, created_at);  -- For sorted queries

-- Now all query patterns covered efficiently
```

---

## 8. Examples

### Example 1: Optimizing Multi-Column Filter Query

**Scenario:** Find recent orders for high-value customers

**Query:**
```sql
SELECT o.order_id, o.total, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at >= '2024-01-01'
  AND c.lifetime_value > 10000
  AND o.status = 'completed';
```

**Initial Indexes:**
```sql
-- orders table
idx_orders_created (created_at)
idx_orders_status (status)
idx_orders_customer (customer_id)

-- customers table
idx_customers_id (id) -- Primary key
```

**Bad Plan (25 seconds):**
```
Hash Join  (cost=50000.00..100000.00 rows=5000) (actual time=12345.678..25678.901 rows=5000)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..80000.00 rows=500000)  ← Seq scan!
        Filter: (created_at >= '2024-01-01' AND status = 'completed')
        Rows Removed by Filter: 9500000
  ->  Hash  (cost=40000.00..40000.00 rows=10000)
        ->  Seq Scan on customers c  (cost=0.00..40000.00 rows=10000)  ← Seq scan!
              Filter: (lifetime_value > 10000)
              Rows Removed by Filter: 990000
```

**Analysis:**
1. Sequential scans on both tables (slow)
2. Filters 9.5M orders to 500K
3. Filters 990K customers to 10K
4. Then joins 500K × 10K

**Solution: Add Composite Indexes**
```sql
-- Orders: Composite index on frequently-filtered columns
CREATE INDEX idx_orders_created_status_customer 
ON orders(created_at, status, customer_id);

-- Customers: Index on lifetime_value
CREATE INDEX idx_customers_ltv ON customers(lifetime_value);

ANALYZE orders, customers;
```

**Optimized Plan (0.8 seconds):**
```
Hash Join  (cost=5000.00..15000.00 rows=5000) (actual time=300.123..850.456 rows=5000)
  Hash Cond: (o.customer_id = c.id)
  ->  Index Scan using idx_orders_created_status_customer on orders o
        (cost=0.56..8000.00 rows=5000)
        Index Cond: (created_at >= '2024-01-01' AND status = 'completed')
  ->  Hash  (cost=2000.00..2000.00 rows=10000)
        ->  Index Scan using idx_customers_ltv on customers c
              (cost=0.43..2000.00 rows=10000)
              Index Cond: (lifetime_value > 10000)

-- Result: 25s → 0.8s (31× FASTER!)
```

---

### Example 2: Covering Index for Reporting Query

**Scenario:** Daily sales report (runs every morning)

**Query:**
```sql
SELECT 
    DATE(created_at) as sale_date,
    COUNT(*) as order_count,
    SUM(total) as total_sales,
    AVG(total) as avg_order_value
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
  AND status = 'completed'
GROUP BY DATE(created_at)
ORDER BY sale_date;
```

**Initial Index:**
```sql
CREATE INDEX idx_orders_created ON orders(created_at);
```

**Bad Plan (15 seconds):**
```
GroupAggregate  (cost=100000.00..150000.00 rows=30) (actual time=8000.123..15000.456 rows=30)
  Group Key: (date(created_at))
  ->  Sort  (cost=100000.00..110000.00 rows=500000)
        Sort Key: (date(created_at))
        ->  Index Scan using idx_orders_created on orders
              (cost=0.56..80000.00 rows=500000)
              Index Cond: (created_at >= (CURRENT_DATE - '30 days'::interval))
              Filter: (status = 'completed')
              Rows Removed by Filter: 50000  ← Extra filter after index scan

-- Problem: Heap access for every row to check status and get total
-- 500K random heap reads!
```

**Solution: Covering Index**
```sql
CREATE INDEX idx_orders_created_status_total_covering 
ON orders(created_at, status) INCLUDE (total);
-- Or: ON orders(created_at, status, total) for older PostgreSQL

ANALYZE orders;
```

**Optimized Plan (1.5 seconds):**
```
GroupAggregate  (cost=8000.00..12000.00 rows=30) (actual time=800.123..1500.456 rows=30)
  Group Key: (date(created_at))
  ->  Sort  (cost=8000.00..9000.00 rows=450000)
        Sort Key: (date(created_at))
        ->  Index Only Scan using idx_orders_created_status_total_covering on orders
              (cost=0.56..5000.00 rows=450000)
              Index Cond: (created_at >= (CURRENT_DATE - '30 days'::interval) AND status = 'completed')

-- No heap access! All data from index.
-- Result: 15s → 1.5s (10× FASTER!)
```

---

## 9. Real-World Use Cases

### Use Case 1: Time-Series Data (Append-Only Pattern)

**Scenario:** IoT sensor data, log events, financial ticks

**Data Pattern:**
- Inserts always at current time (no updates)
- Queries filter by time range (last hour, last day)
- Natural clustering by timestamp

**Optimal Index Strategy:**
```sql
-- Table: sensor_readings (1B rows, append-only)
CREATE TABLE sensor_readings (
    id BIGSERIAL,
    sensor_id INTEGER,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    value DECIMAL,
    PRIMARY KEY (timestamp, sensor_id)  ← Clustered by time
);

-- Index strategy:
CREATE INDEX idx_sensor_readings_sensor_ts ON sensor_readings(sensor_id, timestamp DESC);

-- Benefits:
-- 1. Primary key clusters data by time → Range queries fast
-- 2. Secondary index supports per-sensor queries
-- 3. No fragmentation (append-only, no page splits)
-- 4. Old partitions can be dropped (partition by month)

-- Query performance:
SELECT * FROM sensor_readings
WHERE sensor_id = 12345
  AND timestamp >= NOW() - INTERVAL '1 hour';

-- Uses idx_sensor_readings_sensor_ts
-- Scans only recent data (hot data in cache)
-- Time: 0.05s for 10K rows ✅
```

### Use Case 2: E-commerce Search with Multiple Filters

**Scenario:** Product search with faceted filters

**Query Pattern:**
```sql
SELECT * FROM products
WHERE category_id = 5
  AND price BETWEEN 100 AND 500
  AND brand_id IN (10, 20, 30)
  AND in_stock = true
  AND rating >= 4.0
ORDER BY popularity DESC
LIMIT 20;
```

**Challenge:** 5 filters, can't optimize for all combinations

**Solution: Bitmap Index Scans (PostgreSQL Combines Indexes)**
```sql
-- Create individual indexes:
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_brand ON products(brand_id);
CREATE INDEX idx_products_instock ON products(in_stock) WHERE in_stock = true;  ← Partial
CREATE INDEX idx_products_rating ON products(rating);
CREATE INDEX idx_products_popularity ON products(popularity DESC);

-- PostgreSQL automatically combines indexes:
-- Plan:
Limit  (cost=5000.00..5050.00 rows=20)
  ->  Sort  (cost=5000.00..5500.00 rows=5000)
        Sort Key: popularity DESC
        ->  BitmapAnd  (cost=3000.00..4000.00 rows=5000)  ← Combines indexes!
              ->  Bitmap Index Scan on idx_products_category
                    Index Cond: (category_id = 5)
              ->  Bitmap Index Scan on idx_products_price
                    Index Cond: (price >= 100 AND price <= 500)
              ->  Bitmap Index Scan on idx_products_brand
                    Index Cond: (brand_id = ANY ('{10,20,30}'::integer[]))
              ->  Bitmap Index Scan on idx_products_instock
              ->  Bitmap Index Scan on idx_products_rating
                    Index Cond: (rating >= 4.0)

-- Execution: 0.2s (combines 5 indexes efficiently)
```

---

## 10. Interview Questions

### Q1: How would you debug a query that's not using an index?

**Senior Answer:**

```
Step-by-step debugging process:

1. **Verify index exists and is relevant**
```sql
-- Check indexes on table
\d table_name  -- PostgreSQL
SHOW INDEX FROM table_name;  -- MySQL

-- Ensure index columns match WHERE clause
```

2. **Check if index is actually being ignored**
```sql
EXPLAIN SELECT * FROM orders WHERE status = 'pending';

-- If shows "Seq Scan" instead of "Index Scan":
-- → Index not used

-- Reasons:

**A.Cost-based decision (optimizer chose seq scan as cheaper)**
```sql
-- Force index to see if it's actually slower:
SET enable_seqscan = off;
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';

-- Compare execution times:
-- - Seq scan: 1000ms
-- - Index scan: 5000ms
-- → Seq scan was correct choice! (status='pending' is 90% of rows)
```

**B. Stale statistics**
```sql
-- Check last analyze:
SELECT last_analyze, n_live_tup FROM pg_stat_user_tables WHERE tablename = 'orders';

-- If old, update:
ANALYZE orders;

-- Re-run EXPLAIN
```

**C. Function on indexed column**
```sql
-- This won't use index on email:
WHERE LOWER(email) = 'user@example.com'

-- Fix: Function-based index
CREATE INDEX idx_email_lower ON users(LOWER(email));
```

**D. Type mismatch (implicit cast)**
```sql
-- user_id is INTEGER, but query uses STRING
WHERE user_id = '12345'  ← Forces cast, disables index

-- Fix: Use correct type
WHERE user_id = 12345
```

**E. Wrong column order in composite index**
```sql
-- Index: (country, city)
-- Query: WHERE city = 'NYC'  ← Can't use (skips country)

-- Fix: Add single-column index on city, or reorder composite index
```

3. **Verify fix**
```sql
-- After applying fix:
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- Confirm:
-- - Index Scan (not Seq Scan)
-- - Actual execution time improved
-- - Buffers: shared hit increased (using cache)
```
```

### Q2: When would you create a partial (filtered) index?

**Staff Answer:**

```
Create partial indexes when:

**1. Heavily skewed data (filtering majority)**
```sql
-- 99% of orders are 'completed', only 1% 'pending'
-- Query only cares about pending:
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'pending';

-- Benefits:
-- - Index 100× smaller (1% of rows)
-- - Faster scans (less data)
-- - Faster writes (completed orders don't update index)
-- - Query MUST include WHERE status = 'pending' to use index
```

**2. Soft-delete pattern**
```sql
-- 90% of rows have deleted_at IS NOT NULL
-- Queries only want active records:
CREATE INDEX idx_users_email_active ON users(email) WHERE deleted_at IS NULL;

-- Query:
SELECT * FROM users WHERE email = 'user@example.com' AND deleted_at IS NULL;
-- Uses partial index (10× smaller than full index)
```

**3. Multi-tenant with VIP tenants**
```sql
-- 1% of tenants are VIP, need fastest response
CREATE INDEX idx_events_tenant_vip ON events(tenant_id, created_at) 
WHERE tenant_id IN (1, 5, 10, 23);  -- VIP tenants

-- VIP queries get dedicated index (no contention with regular tenants)
```

**4. Boolean flags (low cardinality)**
```sql
-- is_premium: false (95%), true (5%)
-- Query only cares about premium users:
CREATE INDEX idx_users_premium ON users(id) WHERE is_premium = true;

-- Without partial index:
-- - Full index on 1M rows
-- - Query scans 1M rows, filters to 50K

-- With partial index:
-- - Partial index on 50K rows
-- - Query scans 50K rows directly
```

**When NOT to use partial indexes:**

- Data evenly distributed
- Queries don't have consistent WHERE clause
- Condition changes frequently (index needs rebuild)

**Trade-off:**
- Pros: Smaller, faster, less write overhead
- Cons: Only works for queries matching WHERE condition exactly
```

### Q3: Explain how composite index column order affects query performance.

**Principal Answer:**

```
Composite index column order critical for two reasons:

**1. Leftmost Prefix Rule (Index Usability)**

Index: idx_users (country, city, age)

```sql
-- ✅ Can use index (all columns, left to right)
WHERE country = 'US' AND city = 'NYC' AND age = 30

-- ✅ Can use index (first 2 columns)
WHERE country = 'US' AND city = 'NYC'

-- ✅ Can use index (first column only)
WHERE country = 'US'

-- ❌ CANNOT use index (skips country)
WHERE city = 'NYC' AND age = 30

-- ⚠️ Can use index, but only for country (range on city prevents age usage)
WHERE country = 'US' AND city > 'M' AND age = 30
```

**B-Tree structure:** Keys sorted lexicographically (country, city, age).
Jumping to city='NYC' without country is like searching a phone book by first name.

**2. Selectivity and Intermediate Result Size**

Order determines scan efficiency.

Example: Table with 1M rows

```sql
-- Scenario 1: Low selectivity first
CREATE INDEX idx_orders_status_user (status, user_id);

WHERE status = 'completed' AND user_id = 12345;

-- Scan:
-- 1. status='completed' → 990K rows (low selectivity)
-- 2. user_id=12345 within 990K → 50 rows
-- Intermediate result: 990K (inefficient)

-- Scenario 2: High selectivity first
CREATE INDEX idx_orders_user_status (user_id, status);

WHERE user_id = 12345 AND status = 'completed';

-- Scan:
-- 1. user_id=12345 → 50 rows (high selectivity)
-- 2. status='completed' within 50 → 45 rows
-- Intermediate result: 50 (efficient!)
```

**Decision Framework:**

1. **Analyze query patterns:**
```sql
-- 70% of queries: WHERE user_id = ?
-- 20% of queries: WHERE user_id = ? AND status = ?
-- 10% of queries: WHERE status = ?

-- Order: (user_id, status) ← Satisfies 90% of queries
```

2. **Consider selectivity:**
```sql
-- Check distinct values:
SELECT 
    COUNT(DISTINCT user_id) as user_selectivity,  -- 100K distinct
    COUNT(DISTINCT status) as status_selectivity;  -- 3 distinct

-- user_id more selective → Put first
```

3. **Handle range queries:**
```sql
-- Query: WHERE created_at >= ? AND status = ?

-- If index: (created_at, status)
-- - Can use both columns ✅

-- If index: (status, created_at)
-- - If status has few values, must scan large intermediate result ❌
-- - But if status is selective, this is better ✅

-- Rule: Range column as last (or second-to-last) in index
```

4. **Test both orders:**
```sql
-- Create both, compare:
CREATE INDEX idx_test1 ON orders(user_id, status);
CREATE INDEX idx_test2 ON orders(status, user_id);

EXPLAIN (ANALYZE, BUFFERS) [your common queries];

-- Keep better-performing index, drop other
```

**Real-world example: Log search**

```sql
-- Query: Logs for specific service in last 24 hours
WHERE service_id = 'api-gateway' AND timestamp >= NOW() - INTERVAL '24 hours';

-- Option 1: (service_id, timestamp)
-- - Filters to service first (~10% of logs)
-- - Then filters by timestamp within service
-- - Range scan on timestamp works well
-- ✅ Best for queries filtering by specific service

-- Option 2: (timestamp, service_id)
-- - Filters to last 24h first (~1% of logs)
-- - Then filters by service within that time range
-- - ✅✅ Best if querying recent data across all services frequently

-- Decision: Depends on query pattern!
```
```

---

## 11. Summary

### Index Selection Decision Matrix

| Query Pattern | Best Index Strategy | Example |
|---------------|---------------------|---------|
| Single column, high selectivity | Single-column index | WHERE email = '' |
| Multiple AND conditions | Composite index (selectivity order) | WHERE user_id = ? AND status = ? |
| OR conditions | Separate indexes + Bitmap OR | WHERE cat=5 OR price<100 |
| Frequent query, few columns needed | Covering index | SELECT id, name WHERE cat=? |
| Range + equality | Composite (equality first) | WHERE user_id=? AND date>=? |
| Sorted output | Index on sort column | ORDER BY created_at DESC |
| Skewed data | Partial index | WHERE status='pending' (1% of rows) |
| Case-insensitive search | Expression index | WHERE LOWER(email)=? |

### Key Principles

1. **Leftmost Prefix Rule**
   - Composite index (A, B, C) usable for: A, (A,B), (A,B,C)
   - NOT usable for: B, C, (B,C), (A,C) where C is BETWEEN

2. **Selectivity Ordering**
   - Most selective column first (smallest intermediate result)
   - Exception: Query patterns may demand different order

3. **Covering Indexes**
   - Include all query columns → eliminate heap access
   - Trade-off: Larger index, slower writes

4. **Index Maintenance**
   - Drop unused indexes (check pg_stat_user_indexes)
   - Drop redundant indexes (composite covers single-column)
   - Monitor write overhead (too many indexes = slow writes)

### Optimization Checklist

- [ ] Run EXPLAIN ANALYZE to verify index usage
- [ ] Check selectivity of filter columns (pg_stats.n_distinct)
- [ ] Order composite index by selectivity (highest first)
- [ ] Consider covering indexes for hot queries
- [ ] Use partial indexes for skewed data (soft deletes, status flags)
- [ ] Create expression indexes for function-based filters
- [ ] Monitor index usage (drop unused indexes after 30 days)
- [ ] Set fillfactor=80 for write-heavy tables
- [ ] Update statistics after bulk changes (ANALYZE)
- [ ] Test index changes on production-like data volume

**Remember:** More indexes ≠ better performance. Each index costs disk space and slows writes. Create indexes strategically based on query patterns, and drop unused/redundant ones ruthlessly.

---

**Next Reading:**
- `06_Query_Hints.md` - When and how to override optimizer's index selection
- `07_Statistics_Management.md` - Keeping cardinality estimates accurate for better index selection
- `08_N_Plus_One_Problem.md` - Application-level query optimization
