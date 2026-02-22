# Bitmap Indexes - Optimizing Low-Cardinality Columns

## 1. Concept Explanation

**Bitmap indexes** store data as bit arrays (bitmaps) where each bit represents whether a row contains a specific value. They excel at indexing low-cardinality columns (few distinct values, many duplicates) commonly found in data warehouse and OLAP workloads.

Think of a bitmap index like a matrix spreadsheet:
- **Rows:** Each row in the table
- **Columns:** Each distinct value in the indexed column
- **Cell values:** 1 if that row has that value, 0 otherwise

### Bitmap vs B-Tree

```
Scenario: 1M users, gender column (3 values: 'M', 'F', 'other')

B-Tree Index on gender:
- Storage: 1M entries × (8 bytes pointer + 1 byte value) ≈ 9MB
- Lookup WHERE gender='M': Navigate tree, scan 500K matching entries

Bitmap Index on gender:
- Storage: 3 bitmaps × 1M bits / 8 bits per byte ≈ 375KB (24× smaller!)
- Lookup WHERE gender='M': Read bitmap #1, return set bits
- Operations: Fast bitwise AND/OR/XOR for complex queries
```

### Structure Visualization

```
Table: users (1M rows)
Column: status ('active', 'inactive', 'deleted')

Bitmap Index Structure:

                Row 0  Row 1  Row 2  Row 3  Row 4  Row 5  ...  Row 999,999
Bitmap 'active'    1      1      0      1      0      1  ...      1
Bitmap 'inactive'  0      0      1      0      1      0  ...      0  
Bitmap 'deleted'   0      0      0      0      0      0  ...      0

Query: WHERE status = 'active'
→ Return rows where bitmap 'active' = 1
→ Positions: {0, 1, 3, 5, ..., 999999}

Query: WHERE status = 'active' OR status = 'inactive'
→ Bitwise OR: bitmap_active | bitmap_inactive
→ Result bitmap: [1, 1, 1, 1, 1, 1, ..., 1]
→ Fast bitwise operation (millions of rows/sec)
```

**Key Properties:**
- **Space-efficient:** 1 bit per row per distinct value
- **Fast bitwise operations:** AND, OR, XOR, NOT (hardware-accelerated)
- **Compression:** Long runs of 0s or 1s compress extremely well (Run-Length Encoding)
- **Read-optimized:** Perfect for OLAP, terrible for OLTP (updates expensive)

---

## 2. Why It Matters

### Production Impact: Data Warehouse Queries

**Scenario: E-commerce analytics (100M orders)**

```sql
-- Query: Count orders by status and priority
SELECT status, priority, COUNT(*)
FROM orders
WHERE order_date >= '2024-01-01'
  AND (status = 'shipped' OR status = 'delivered')
  AND priority IN ('high', 'medium')
GROUP BY status, priority;
```

**Without Bitmap Indexes (B-Tree):**
```
-- B-Tree indexes on status, priority, order_date
-- Execution plan:

Bitmap Heap Scan on orders  (cost=100000..500000 rows=2000000)
  Recheck Cond: ((status = 'shipped' OR status = 'delivered') 
                  AND priority IN ('high','medium') 
                  AND order_date >= '2024-01-01')
  -> BitmapOr
       -> Bitmap Index Scan on idx_status
            Index Cond: (status = 'shipped')
       -> Bitmap Index Scan on idx_status
            Index Cond: (status = 'delivered')
  -> BitmapAnd
       -> Bitmap Index Scan on idx_priority
            Index Cond: (priority = 'high' OR priority = 'medium')
       -> Bitmap Index Scan on idx_order_date
            Index Cond: (order_date >= '2024-01-01')

-- Execution: 12 seconds
-- I/O: 200,000 pages (combining multiple B-Tree scans, heap fetches)
```

**With Bitmap Indexes (Oracle/PostgreSQL):**
```
-- Bitmap indexes on status, priority

Bitmap Heap Scan on orders
  -> BitmapAnd
       -> Bitmap Index Scan on bm_idx_status
            Bitmaps: status='shipped' OR status='delivered'
       -> Bitmap Index Scan on bm_idx_priority
            Bitmaps: priority='high' OR priority='medium'
       -> Bitmap Index Scan on bm_idx_order_date
            Index Cond: (order_date >= '2024-01-01')

-- Execution: 1.5 seconds (8× FASTER!)
-- I/O: 25,000 pages (compressed bitmaps + heap fetch)

-- Why faster?
-- 1. Bitwise operations (AND, OR) are hardware-accelerated
-- 2. Bitmaps compress well (RLE encoding)
-- 3. One pass through bitmaps vs multiple B-Tree navigations
```

### Real Numbers: Storage Savings

**Table: products (10M rows, 20 attributes)**

| Column | Cardinality | B-Tree Size | Bitmap Size | Compression Ratio |
|--------|-------------|-------------|-------------|-------------------|
| category | 50 | 120MB | 6MB | 20:1 |
| brand | 500 | 115MB | 60MB | 2:1 |
| country | 200 | 118MB | 25MB | 5:1 |
| in_stock | 2 | 100MB | 2.5MB | 40:1 |
| is_featured | 2 | 100MB | 2.5MB | 40:1 |
| **Total** | **754** | **553MB** | **96MB** | **5.8:1** |

**Impact:**
- 6× less disk space for indexes
- More indexes fit in buffer cache (better hit rate)
- Faster backups (less data to transfer)

---

## 3. Internal Working

### Bitmap Storage with Run-Length Encoding (RLE)

**Raw Bitmap (uncompressed):**
```
Bitmap for status='active' (1M rows):
[1,1,1,1,1,1,1,1,0,0,0,0,0,0,0,0,1,1,1,1,...]

Size: 1M bits = 125KB
```

**RLE Compressed Bitmap:**
```
Instead of storing individual bits, store runs:

Format: (value, count)
[(1, 8), (0, 8), (1, 4), ...]
       ↑       ↑       ↑
   8 ones  8 zeros  4 ones

Compression ratio depends on clustering:
- Clustered data (sorted by indexed column): 100:1 compression!
- Random distribution: 2:1 compression
```

**Example with High Compression:**
```
Table sorted by status:
[active, active, active, ..., inactive, inactive, ..., deleted, deleted]
  ↑ 500K active rows       ↑ 499K inactive      ↑ 1K deleted

Bitmap 'active':
Raw: [1,1,1,1,1,...(500K times)...,1,0,0,0,...(499K times)...,0,0,0,...(1K times)...,0]
Size: 1M bits = 125KB

RLE: [(1, 500000), (0, 500000)]
Size: 2 entries × 8 bytes = 16 bytes (7800× COMPRESSION!)

-- Query: WHERE status='active'
-- Read: 16 bytes (not 125KB)
-- Decompress: Generate bitmap of first 500K bits set to 1
-- Result: Instant (hardware-accelerated)
```

### Bitwise Operations for Complex Queries

**Query with Multiple Conditions:**
```sql
SELECT * FROM users
WHERE status = 'active' 
  AND country = 'US'
  AND age >= 18;
```

**Bitmap Execution:**
```
Step 1: Load bitmaps
  bitmap_active:   [1,0,1,1,0,1,1,1,0,1]
  bitmap_us:       [1,1,0,1,0,1,0,1,1,0]
  bitmap_age_18+:  [1,1,1,0,1,1,1,1,0,1]

Step 2: Bitwise AND (all conditions must be true)
  result = bitmap_active & bitmap_us & bitmap_age_18+
  
  Operation (hardware-accelerated, 64 bits at a time):
  [1,0,1,1,0,1,1,1,0,1]
& [1,1,0,1,0,1,0,1,1,0]
& [1,1,1,0,1,1,1,1,0,1]
= [1,0,0,0,0,1,0,1,0,0]
   ↑           ↑     ↑
  Row 0      Row 5  Row 7 match all conditions

Step 3: Use result bitmap to fetch heap tuples
  Fetch rows: {0, 5, 7}

-- Performance:
-- AND operation: 1M bits in 125KB
-- CPU: 0.5ms (250 MB/sec throughput, hardware SIMD)
-- vs B-Tree: 50ms (combining 3 index scans)
-- Speedup: 100× FASTER for bitmap operations!
```

**Bitwise OR (Any Condition):**
```sql
WHERE status = 'active' OR status = 'pending'

bitmap_active:   [1,0,1,1,0,1,1,1,0,1]
bitmap_pending:  [0,1,0,0,1,0,0,0,1,0]

result = bitmap_active | bitmap_pending
= [1,1,1,1,1,1,1,1,1,1]  ← Any row matching either condition
```

### Word-Aligned Bitmap (WAH Compression)

**Problem with Naive RLE:**
- RLE compresses well but decompress before operations (slow)

**Solution: Word-Aligned Hybrid (WAH):**
```
Store bitmaps in 32-bit or 64-bit words
Allow bitwise operations on compressed format!

Format:
- Literal word: Next 31 bits are raw bitmap data
  [0 | 31 bits of data]
- Fill word: Next N words are all 0s or all 1s
  [1 | fill_bit | 30-bit count]

Example:
Uncompressed: [1111...1111 (32 ones), 0000...0000 (32 zeros), 1010...1010 (mixed), ...]

WAH compressed:
  Word 0: [1 | 1 | 0000000000000000000000000000001]  ← Fill: 1 word of all 1s
  Word 1: [1 | 0 | 0000000000000000000000000000001]  ← Fill: 1 word of all 0s
  Word 2: [0 | 1010101010101010101010101010101]      ← Literal: mixed data
  
Advantage: Can perform AND/OR directly on compressed words!
```

---

## 4. Best Practices

### Practice 1: Use Bitmap Indexes for Low-Cardinality Columns

**Rule of Thumb: Cardinality < 0.1% of rows**

```sql
-- Decision matrix for 1M row table:

-- Column: gender (2-3 distinct values, 0.0003% cardinality)
-- ✅ Excellent bitmap candidate
CREATE BITMAP INDEX idx_users_gender ON users(gender);

-- Column: status (5 distinct values, 0.0005% cardinality)
-- ✅ Good bitmap candidate
CREATE BITMAP INDEX idx_orders_status ON orders(status);

-- Column: country (200 distinct values, 0.02% cardinality)
-- ⚠️ Marginal bitmap candidate (might still help if queries use OR/AND)
CREATE BITMAP INDEX idx_users_country ON users(country);

-- Column: email (990K distinct values, 99% cardinality)
-- ❌ Bad bitmap candidate (use B-Tree)
CREATE INDEX idx_users_email ON users(email);  -- B-Tree

-- Why email fails:
-- Bitmap size: 990K bitmaps × 1M bits = 123GB (uncompressed!)
-- vs B-Tree: 1M entries × 100 bytes = 100MB
-- Bitmap 1200× LARGER than B-Tree!
```

**Cardinality Guidelines:**

| Cardinality | Distinct Values (1M rows) | Recommendation |
|-------------|---------------------------|----------------|
| < 10 | < 10 | ✅ Excellent (boolean, enum) |
| 10-100 | 10-100 | ✅ Good (status, category) |
| 100-1000 | 100-1,000 | ⚠️ Consider (country, product type) |
| 1K-10K | 1,000-10,000 | ❌ Unlikely (use B-Tree) |
| > 10K | > 10,000 | ❌ Never (use B-Tree) |

### Practice 2: Combine Multiple Bitmap Indexes

**Power of Bitmap Indexes: Efficient Multi-Column Filtering**

```sql
-- Instead of composite index, create separate bitmap indexes
-- Optimizer combines them dynamically!

-- Traditional approach (B-Tree composite index):
CREATE INDEX idx_products_composite ON products(category, brand, in_stock);

-- Problem: Only useful for queries with all 3 columns (or leftmost prefix)
WHERE category = ? AND brand = ? AND in_stock = ?  ✅ Uses index
WHERE category = ? AND brand = ?                   ✅ Uses index
WHERE category = ?                                 ✅ Uses index
WHERE brand = ? AND in_stock = ?                   ❌ Index not used!
WHERE in_stock = ?                                 ❌ Index not used!

-- Bitmap approach (separate indexes):
CREATE BITMAP INDEX idx_products_category ON products(category);
CREATE BITMAP INDEX idx_products_brand ON products(brand);
CREATE BITMAP INDEX idx_products_in_stock ON products(in_stock);

-- Now ALL combinations work efficiently:
WHERE category = ?                      ✅ Uses idx_products_category
WHERE brand = ?                         ✅ Uses idx_products_brand
WHERE in_stock = ?                      ✅ Uses idx_products_in_stock
WHERE category = ? AND brand = ?        ✅ Combines 2 bitmaps (bitwise AND)
WHERE brand = ? AND in_stock = ?        ✅ Combines 2 bitmaps (bitwise AND)
WHERE category = ? AND brand = ? AND in_stock = ?  ✅ Combines 3 bitmaps

-- Any combination of filters works!
-- Optimizer dynamically picks which bitmaps to combine
```

**Real Example: Ad-Hoc Analytics**
```sql
-- User interface with multiple filters:
-- - Category dropdown
-- - Brand checkboxes  
-- - In stock toggle
-- - Price range slider

-- Query changes based on user selections:
-- Query A: WHERE category='Electronics' AND in_stock=true
-- Query B: WHERE brand='Apple' AND price > 500
-- Query C: WHERE category='Electronics' AND brand='Samsung' AND in_stock=true

-- Bitmap indexes: Work for all combinations ✅
-- Composite B-Tree: Would need multiple indexes for all combinations ❌
--   Need: (category, in_stock), (brand, price), (category, brand, in_stock), ...
--   Total: 2^N - 1 indexes for N columns! (exponential explosion)
```

### Practice 3: Cluster Data by Indexed Column

**Maximize Compression Ratio**

```sql
-- Problem: Random data distribution
-- Table: orders (unsorted)
-- Column: status: ['pending', 'shipped', 'pending', 'delivered', 'shipped', ...]

-- Bitmap for 'pending' (random distribution):
[1,0,1,0,0,1,1,0,1,0,1,0,...]  
-- RLE: [(1,1),(0,1),(1,1),(0,2),(1,2),(0,1),(1,1),(0,1),...]
-- Compression: 2:1 (many short runs)

-- Solution: CLUSTER by status
CLUSTER orders USING idx_orders_status;

-- After clustering: All 'pending' together, then 'shipped', then 'delivered'
-- Column: status: ['pending', 'pending', ..., 'shipped', 'shipped', ..., 'delivered', ...]

-- Bitmap for 'pending' (clustered):
[1,1,1,1,1,1,1,1,1,1,...(500K times)...,0,0,0,0,...(rest)...]
-- RLE: [(1, 500000), (0, 500000)]
-- Compression: 7800:1 (two long runs!)

-- Benefits:
-- - 39× better compression (2:1 → 7800:1)
-- - Faster queries (sequential I/O for matching rows)
-- - Smaller index (fits in cache)
```

**Clustering Strategy:**
```sql
-- For data warehouse: Cluster by most frequently filtered column

-- Example: Orders table
-- Query pattern: 80% filter by date, 15% filter by status, 5% other

-- Cluster by date (primary filter):
CREATE INDEX idx_orders_date ON orders(order_date);
CLUSTER orders USING idx_orders_date;

-- Bitmap index on status (secondary filter):
CREATE BITMAP INDEX idx_orders_status ON orders(status);

-- Result:
-- - Date filters: Sequential I/O (clustered)
-- - Status bitmaps: Still compressed well (rows near same date have similar status)
```

### Practice 4: Avoid Bitmap Indexes on OLTP Tables

**❌ Bad: Bitmap Index on Transactional Table**

```sql
-- High UPDATE/INSERT rate (OLTP)
CREATE BITMAP INDEX idx_orders_status ON orders(status);

-- Problem: Every UPDATE requires modifying bitmap

-- UPDATE operation:
UPDATE orders SET status = 'shipped' WHERE order_id = 12345;

-- Behind the scenes:
1. Find row 12345 in table → Row #500,000
2. Old status: 'pending'
   - Flip bitmap_pending[500000] from 1 → 0
   - Write entire 32-bit word containing bit 500000 (32 bits = 4 bytes)
3. New status: 'shipped'
   - Flip bitmap_shipped[500000] from 0 → 1
   - Write entire 32-bit word
   
-- Lock contention:
-- - Each bitmap word is a separate lock
-- - If 2 concurrent UPDATEs modify rows in same word (rows 500000-500031):
--     Thread A: UPDATE row 500000
--     Thread B: UPDATE row 500015
--     → Both need exclusive lock on same word → Serialized! ❌

-- Performance impact:
-- - Single UPDATE: 2-5× slower than B-Tree (lock contention)
-- - Concurrent UPDATEs: 10-100× slower (serialization)
-- - Bitmap fragmentation over time (RLE compression degrades)
```

**✅ Good: Bitmap Index on Read-Heavy OLAP Table**

```sql
-- Data warehouse: Nightly batch load, no daytime updates
CREATE BITMAP INDEX idx_sales_category ON sales(category);

-- Workload:
-- - Load: Once per day (1M rows bulk insert)
-- - Queries: 10K/sec during day (all reads)

-- INSERT impact manageable:
-- - Batch rebuild bitmaps after load (one-time cost)
-- - No lock contention (single batch operation)

-- Query benefits:
-- - 10× faster than B-Tree for multi-condition queries
-- - 90% less index storage (more data in cache)
```

**Decision Matrix:**

| Workload | Read:Write Ratio | Bitmap Index | Rationale |
|----------|------------------|--------------|-----------|
| OLTP (e-commerce checkout) | 20:80 | ❌ Never | High write contention |
| OLTP (user profile queries) | 95:5 | ⚠️ Maybe | If updates are batch (nightly) |
| OLAP (analytics dashboard) | 99.9:0.1 | ✅ Excellent | Read-dominated |
| Data warehouse (BI queries) | 100:0 (batch load) | ✅ Perfect | No concurrent writes |

---

## 5. Common Mistakes

### Mistake 1: Creating Bitmap Index on High-Cardinality Column

**Problem:**
```sql
-- Email: 1M distinct values (100% unique)
CREATE BITMAP INDEX idx_users_email ON users(email);

-- Storage calculation:
-- - 1M distinct values → 1M bitmaps
-- - Each bitmap: 1M bits = 125KB
-- - Total: 1M × 125KB = 125GB uncompressed!

-- With low compression (random distribution):
-- - Compression ratio: 2:1 (best case)
-- - Actual size: 62.5GB

-- vs B-Tree:
-- - 1M entries × 100 bytes (email + pointer) = 100MB
-- - Bitmap is 625× LARGER! ❌

-- Query performance:
WHERE email = 'alice@example.com'

-- Bitmap: Load 1M bitmap headers, find 'alice@example.com' bitmap, scan for set bit
-- Time: 500ms (huge index scan)

-- B-Tree: Navigate 4-level tree, find exact entry
-- Time: 0.5ms (1000× FASTER!)
```

**Solution:**
```sql
DROP BITMAP INDEX idx_users_email;
CREATE INDEX idx_users_email ON users(email);  -- B-Tree
```

### Mistake 2: Not Rebuilding Bitmap After Major Data Changes

**Problem: Degraded Compression Over Time**

```sql
-- Initial: Data loaded in sorted order
CREATE BITMAP INDEX idx_sales_region ON sales(region);

-- Initial compression:
-- - Data clustered by region: [NORTH, NORTH, ..., SOUTH, SOUTH, ...]
-- - Bitmap 'NORTH': [(1, 500000), (0, 500000)]
-- - RLE compression: 7800:1
-- - Size: 16 bytes

-- After 6 months: Random INSERTs
-- - New data scattered: [NORTH, EAST, NORTH, SOUTH, WEST, ...]
-- - Bitmap 'NORTH': [(1, 500000), (0, 100000), (1, 100), (0, 200), (1, 50), ...]
-- - RLE compression: 10:1 (degraded!)
-- - Size: 12KB (750× LARGER than initial)

-- Query performance:
-- - Bitmap size increased 750×
-- - No longer fits in cache
-- - Decompress time: 0.01ms → 5ms (500× slower)
```

**Solution: Periodic Rebuild**

```sql
-- Schedule quarterly rebuild:
-- 1. Re-cluster data (optional, for maximum compression)
CLUSTER sales USING idx_sales_date;  -- Cluster by date (query pattern)

-- 2. Rebuild bitmap indexes
ALTER INDEX idx_sales_region REBUILD;

-- After rebuild:
-- - Compression restored: 7000:1
-- - Size: 18 bytes (back to optimal)
-- - Query performance restored

-- Automate in maintenance window:
-- PostgreSQL example:
DO $$
DECLARE
    idx RECORD;
BEGIN
    FOR idx IN 
        SELECT indexname 
        FROM pg_indexes 
        WHERE schemaname = 'public' 
          AND indexdef LIKE '%USING bitmap%'  -- Bitmap indexes only
    LOOP
        EXECUTE 'REINDEX INDEX ' || idx.indexname;
        RAISE NOTICE 'Rebuilt bitmap index: %', idx.indexname;
    END LOOP;
END $$;
```

### Mistake 3: Using Bitmap for Columns that Change Frequently

**Problem:**
```sql
-- Column: last_login (updated every login)
CREATE BITMAP INDEX idx_users_last_login ON users(last_login);

-- User login:
UPDATE users SET last_login = NOW() WHERE user_id = 12345;

-- Bitmap impact:
-- 1. Find old value: last_login = '2024-01-15 10:30:00'
--    - Locate corresponding bitmap (1 of 50K distinct timestamps)
--    - Flip bit #12345 from 1 → 0
--    - If this was last '2024-01-15 10:30:00', delete empty bitmap
-- 2. Insert new value: last_login = '2024-02-22 14:15:00'
--    - Find/create bitmap for '2024-02-22 14:15:00'
--    - Flip bit #12345 from 0 → 1
-- 3. Lock contention: Bitmap header lock (serializes concurrent logins)

-- Performance:
-- - Single login: 5ms (vs 0.5ms with B-Tree)
-- - 1000 concurrent logins/sec: Serialized! (each waits for lock)
-- - Result: Login throughput drops to 200/sec (5× SLOWER!)

-- Bitmap proliferation:
-- - New distinct value per login → 10M bitmaps after 10M logins
-- - Index size explodes: 10M × 1M bits = 1.25TB!
```

**Solution:**
```sql
DROP BITMAP INDEX idx_users_last_login;

-- Option 1: Don't index (if not frequently queried)
-- Option 2: B-Tree index (if needed)
CREATE INDEX idx_users_last_login ON users(last_login);

-- Option 3: Bucket timestamps (reduce cardinality)
CREATE BITMAP INDEX idx_users_last_login_day 
ON users(DATE_TRUNC('day', last_login));
-- Only 365 distinct values per year (much better!)
```

### Mistake 4: Mixing Bitmap and B-Tree in Same Query Carelessly

**Problem: Optimizer Confusion**

```sql
-- Indexes:
CREATE BITMAP INDEX idx_orders_status ON orders(status);  -- Low cardinality
CREATE INDEX idx_orders_customer ON orders(customer_id);  -- High cardinality, B-Tree

-- Query:
SELECT * FROM orders
WHERE status = 'pending'
  AND customer_id = 12345;

-- Optimizer decision challenge:
-- Option A: Use bitmap index on status, then filter by customer_id
--   - Scan bitmap 'pending' (100K rows)
--   - Filter 100K rows for customer_id=12345 (likely 1 row)
--   - Cost: 100K row checks

-- Option B: Use B-Tree index on customer_id, then filter by status
--   - B-Tree lookup customer_id=12345 (10 orders)
--   - Filter 10 orders for status='pending' (likely 3 rows)
--   - Cost: 10 row checks ✅ Much better!

-- But optimizer might choose Option A if statistics are stale!
```

**Solution: Guide Optimizer**

```sql
-- Option 1: Update statistics
ANALYZE orders;

-- Option 2: Composite index
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
-- Clearly signals: Use customer_id first (selective), then status

-- Option 3: Query hint (emergency only)
-- Oracle:
SELECT /*+ INDEX(orders idx_orders_customer) */ *
FROM orders
WHERE status = 'pending' AND customer_id = 12345;

-- PostgreSQL (requires pg_hint_plan):
/*+ IndexScan(orders idx_orders_customer) */
SELECT * FROM orders
WHERE status = 'pending' AND customer_id = 12345;
```

---

## 6. Security Considerations

### 1. Information Leakage via Bitmap Size

**Risk:**
```sql
-- Attacker can infer data distribution from bitmap sizes

-- Query system catalog:
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE schemaname = 'sensitive_db';

-- Results:
-- bm_idx_users_gender: 2MB
-- bm_idx_users_country: 50MB
-- bm_idx_users_is_premium: 20MB

-- Inferences:
-- - gender: 2MB suggests balanced distribution (50% M, 50% F)
-- - is_premium: 20MB suggests imbalanced (90% free, 10% premium = poor compression)
--   → Attacker learns: ~10% of users are premium customers

-- Exploitation:
-- - Target premium users (higher phishing ROI)
-- - Infer revenue (10% × users × avg_premium_price)
```

**Mitigation:**
- Restrict catalog access: `REVOKE SELECT ON pg_stat_user_indexes FROM PUBLIC;`
- Encrypt index files at rest
- Use uniform compression (pad bitmaps to fixed size)

### 2. Timing Attacks on Bitmap Operations

**Risk:**
```sql
-- Query response time reveals data distribution

-- Query A:
SELECT COUNT(*) FROM users WHERE country = 'US';
-- Response: 50ms (large bitmap, many results)

-- Query B:
SELECT COUNT(*) FROM users WHERE country = 'Fiji';
-- Response: 1ms (small bitmap, few results)

-- Attacker infers: US >> Fiji (data location leakage)
```

**Mitigation:**
- Add random delay jitter: `SELECT ... FROM ... WHERE ... AND pg_sleep(random() * 0.1);`
- Rate limiting per IP
- Constant-time responses (pad response time to maximum)

---

## 7. Performance Optimization

### Optimization 1: Bitmap Join Indexes

**Concept: Pre-Join Bitmaps for Star Schema Queries**

```sql
-- Star schema:
-- fact_sales (1B rows): sale_id, product_id, store_id, date_id, amount
-- dim_products: product_id, category, brand
-- dim_stores: store_id, region, city

-- Common query:
SELECT SUM(amount)
FROM fact_sales s
JOIN dim_products p ON s.product_id = p.product_id
WHERE p.category = 'Electronics'
  AND p.brand = 'Sony';

-- Without bitmap join index:
-- 1. Scan dim_products for matching products (10K products)
-- 2. Hash join with fact_sales (1B rows) → Expensive!

-- Bitmap join index: Pre-compute join!
CREATE BITMAP INDEX idx_sales_product_category
ON fact_sales(p.category) 
FROM fact_sales s, dim_products p
WHERE s.product_id = p.product_id;

-- Now query directly uses bitmap on fact table:
-- - Bitmap 'Electronics': Marks all fact_sales rows for Electronics category
-- - No join needed at query time! (pre-joined)
-- - Query: 5 seconds → 0.5 seconds (10× FASTER!)
```

**Oracle Syntax:**
```sql
CREATE BITMAP INDEX idx_sales_electronics
ON fact_sales(products.category)
FROM fact_sales, dim_products products
WHERE fact_sales.product_id = products.product_id;
```

### Optimization 2: Bitmap Index Organized Tables (Oracle)

**Concept: Store Entire Table as Bitmaps**

```sql
-- Traditional: Table (heap) + Bitmap indexes
-- Problem: Bitmap points to heap → Still need heap access

-- Solution: Bitmap Index Organized Table
-- - No heap storage
-- - All columns stored in bitmap format
-- - Extreme compression (100:1+)

-- Example: Sensor data (append-only)
CREATE TABLE sensor_readings (
    sensor_id NUMBER,
    timestamp DATE,
    value NUMBER,
    status VARCHAR2(10)
) ORGANIZATION INDEX COMPRESS;

-- Compressed storage:
-- - sensor_id: 100 distinct sensors → 100 bitmaps
-- - timestamp: Sorted by time → High compression
-- - status: 5 distinct values → 5 bitmaps
-- - value: Stored as exception list (if numeric ranges)

-- Storage: 1B rows
-- - Traditional heap: 100GB
-- - Bitmap organized: 10GB (10× compression)

-- Query performance:
SELECT AVG(value)
FROM sensor_readings
WHERE sensor_id = 42
  AND timestamp >= SYSDATE - 7
  AND status = 'OK';

-- No heap access needed! All data in bitmaps
-- Query time: 0.1 seconds (vs 5 seconds with heap)
```

### Optimization 3: Parallel Bitmap Scans

**Leverage Multiple CPU Cores**

```sql
-- Enable parallel query:
-- PostgreSQL:
SET max_parallel_workers_per_gather = 8;
ALTER TABLE orders SET (parallel_workers = 8);

-- Query:
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*)
FROM orders
WHERE status = 'delivered'
  AND priority = 'high'
  AND region = 'US';

-- Plan:
Finalize Aggregate  (actual time=50.123..50.125 rows=1)
  -> Gather  (actual time=49.456..51.234 rows=8)
       Workers Planned: 8
       Workers Launched: 8
       -> Partial Aggregate  (actual time=48.234..48.235 rows=1)
            -> Parallel Bitmap Heap Scan on orders
                 Recheck Cond: (status = 'delivered' AND priority = 'high' AND region = 'US')
                 -> BitmapAnd
                      -> Bitmap Index Scan on bm_idx_status
                      -> Bitmap Index Scan on bm_idx_priority
                      -> Bitmap Index Scan on bm_idx_region
                           Rows per worker: 125000

-- Execution:
-- - Single-threaded: 400ms
-- - 8 workers: 50ms (8× SPEEDUP!)

-- Best practices:
-- - Table size > 10GB (small tables don't benefit)
-- - Query selectivity < 10% (large result sets parallelize well)
-- - Available CPU cores (don't oversubscribe)
```

---

## 8. Examples

### Example 1: E-Commerce Product Filtering

**Scenario:** Product catalog with faceted search (filters)

```sql
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    category VARCHAR(50),        -- 20 categories
    brand VARCHAR(100),          -- 500 brands
    color VARCHAR(30),           -- 15 colors
    size VARCHAR(10),            -- 10 sizes
    in_stock BOOLEAN,            -- 2 values
    is_featured BOOLEAN,         -- 2 values
    price DECIMAL(10,2)
);

-- 10M products

-- Create bitmap indexes on filter columns:
CREATE BITMAP INDEX bm_idx_category ON products(category);
CREATE BITMAP INDEX bm_idx_brand ON products(brand);
CREATE BITMAP INDEX bm_idx_color ON products(color);
CREATE BITMAP INDEX bm_idx_size ON products(size);
CREATE BITMAP INDEX bm_idx_in_stock ON products(in_stock);
CREATE BITMAP INDEX bm_idx_featured ON products(is_featured);

-- Query: User applies filters
SELECT product_id, name, price
FROM products
WHERE category IN ('Electronics', 'Computers')
  AND brand IN ('Apple', 'Samsung', 'Dell')
  AND color = 'Black'
  AND in_stock = TRUE
  AND is_featured = TRUE
ORDER BY price DESC
LIMIT 50;

-- Execution plan:
Limit
  -> Sort by price DESC
      -> Bitmap Heap Scan on products
            -> BitmapAnd
                 -> BitmapOr  -- category IN (...)
                      -> Bitmap Index Scan (category='Electronics')
                      -> Bitmap Index Scan (category='Computers')
                 -> BitmapOr  -- brand IN (...)
                      -> Bitmap Index Scan (brand='Apple')
                      -> Bitmap Index Scan (brand='Samsung')
                      -> Bitmap Index Scan (brand='Dell')
                 -> Bitmap Index Scan (color='Black')
                 -> Bitmap Index Scan (in_stock=TRUE)
                 -> Bitmap Index Scan (is_featured=TRUE)

-- Performance:
-- - 6 bitmaps combined with AND/OR operations
-- - Bitwise ops: 5ms (hardware-accelerated)
-- - Heap fetch: 10ms (500 matching products)
-- - Sort + Limit: 2ms (top 50)
-- - Total: 17ms ✅

-- vs B-Tree approach:
-- - Would need composite index: (category, brand, color, in_stock, is_featured)
-- - But query permutations: User might apply ANY combination of filters
-- - Need 2^6 = 64 indexes to cover all combinations! ❌
-- - Bitmap: 6 indexes cover all 64 combinations ✅
```

---

### Example 2: Data Warehouse Reporting (Sales Cube)

**Scenario:** OLAP cube with dimensions

```sql
CREATE TABLE fact_sales (
    sale_id BIGINT,
    date_id INT,
    product_id INT,
    store_id INT,
    customer_id INT,
    quantity INT,
    amount DECIMAL(10,2)
) PARTITION BY RANGE (date_id);
-- 1B sales records

-- Dimension tables (small):
CREATE TABLE dim_date (date_id INT, year INT, quarter INT, month INT, day_of_week INT);
CREATE TABLE dim_product (product_id INT, category VARCHAR, subcategory VARCHAR);
CREATE TABLE dim_store (store_id INT, region VARCHAR, country VARCHAR);

-- Bitmap indexes on fact table foreign keys + low-cardinality columns:
CREATE BITMAP INDEX bm_idx_sales_year 
ON fact_sales(d.year)
FROM fact_sales s, dim_date d
WHERE s.date_id = d.date_id;  -- Bitmap join index!

CREATE BITMAP INDEX bm_idx_sales_quarter 
ON fact_sales(d.quarter)
FROM fact_sales s, dim_date d
WHERE s.date_id = d.date_id;

CREATE BITMAP INDEX bm_idx_sales_category
ON fact_sales(p.category)
FROM fact_sales s, dim_product p
WHERE s.product_id = p.product_id;

CREATE BITMAP INDEX bm_idx_sales_region
ON fact_sales(st.region)
FROM fact_sales s, dim_store st
WHERE s.store_id = st.store_id;

-- Query: Sales report by region and category
SELECT 
    region,
    category,
    SUM(amount) AS total_sales,
    COUNT(*) AS num_transactions
FROM fact_sales s
JOIN dim_date d ON s.date_id = d.date_id
JOIN dim_product p ON s.product_id = p.product_id
JOIN dim_store st ON s.store_id = st.store_id
WHERE d.year = 2024
  AND d.quarter = 1
  AND st.region IN ('North America', 'Europe')
  AND p.category = 'Electronics'
GROUP BY region, category;

-- Execution plan (bitmap join indexes):
HashAggregate
  Group Key: region, category
  -> Bitmap Heap Scan on fact_sales
        -> BitmapAnd
             -> Bitmap Index Scan bm_idx_sales_year (year=2024)
             -> Bitmap Index Scan bm_idx_sales_quarter (quarter=1)
             -> BitmapOr
                  -> Bitmap Index Scan bm_idx_sales_region (region='North America')
                  -> Bitmap Index Scan bm_idx_sales_region (region='Europe')
             -> Bitmap Index Scan bm_idx_sales_category (category='Electronics')

-- Performance:
-- - No joins needed! (bitmap join indexes pre-computed joins)
-- - 4 bitmaps combined: 50ms
-- - Aggregate: 100ms
-- - Total: 150ms ✅

-- Without bitmap join indexes:
-- - Join fact_sales (1B rows) with 3 dimension tables
-- - Hash joins: 30 seconds
-- - Aggregate: 5 seconds
-- - Total: 35 seconds ❌

-- Speedup: 233× FASTER!
```

---

## 9. Real-World Use Cases

### Use Case 1: Oracle Exadata - Smart Scan with Bitmap Indexes

**Context:**
- Oracle Exadata: Database appliance with storage servers
- "Smart Scan" pushes query predicates to storage layer
- Bitmap indexes enable predicate pruning at storage

**Implementation:**
```sql
-- Exadata configuration:
-- - 10TB fact table (partitioned by date)
-- - Storage servers: 10 nodes with flash cache
-- - Bitmap indexes on all dimension foreign keys

CREATE BITMAP INDEX bm_idx_sales_product 
ON sales(product_id) LOCAL;  -- Local to each partition

-- Query:
SELECT SUM(amount)
FROM sales
WHERE order_date >= DATE '2024-01-01'
  AND product_category = 'Electronics'
  AND customer_region = 'US';

-- Execution:
-- 1. Storage nodes receive query predicate
-- 2. Each storage node checks local bitmap indexes:
--      - Bitmap 'Electronics': Identifies matching 8K blocks
--      - Bitmap 'US': Identifies matching 8K blocks
--      - Bitwise AND: Only blocks matching both conditions
-- 3. Storage nodes return ONLY matching blocks (not entire partitions)
-- 4. Database server aggregates results

-- Performance:
-- - Traditional: Read 10TB, filter in database layer → 10 minutes
-- - Smart Scan: Read 500GB (5% selectivity), filter at storage → 30 seconds
-- - Speedup: 20× FASTER

-- Key advantage: Reduce data transfer (storage → database)
-- - 10TB × 5GB/sec = 33 minutes of I/O
-- - 500GB × 5GB/sec = 1.6 minutes of I/O
```

### Use Case 2: Vertica - Columnar Storage with Bitmap Indexes

**Context:**
- Vertica: Columnar database (each column stored separately)
- Bitmap indexes integrated into columnar format
- Automatic bitmap creation for low-cardinality columns

**Implementation:**
```sql
-- Vertica table (automatically columnar):
CREATE TABLE events (
    event_id INT,
    user_id INT,
    event_type VARCHAR(50),  -- 20 distinct values
    country VARCHAR(50),     -- 200 distinct values
    device VARCHAR(20),      -- 10 distinct values
    timestamp TIMESTAMP
);

-- Vertica automatically creates bitmaps for low-cardinality columns
-- No explicit index creation needed!

-- Storage layout:
-- Column 'event_type' (compressed):
--   - 20 bitmaps (one per distinct value)
--   - RLE compressed
--   - Size: 2MB (vs 50MB raw column)

-- Column 'country' (mixed):
--   - 200 bitmaps (borderline cardinality)
--   - Compressed but larger
--   - Size: 30MB

-- Column 'user_id' (high cardinality):
--   - No bitmap (too many distinct values)
--   - Stored as compressed dictionary encoding
--   - Size: 200MB

-- Query:
SELECT COUNT(*)
FROM events
WHERE event_type = 'page_view'
  AND country = 'US'
  AND device = 'mobile';

-- Execution:
-- - Read only 3 columns (event_type, country, device)
-- - Bitwise AND of 3 bitmaps: 5ms
-- - Count set bits: 1ms
-- - Total: 6ms (vs 500ms with row storage)

-- Speedup: 83× FASTER
-- - Columnar: Read 3 columns only (not all 6)
-- - Bitmap: Fast bitwise operations
-- - Compression: Less data to read
```

---

## 10. Interview Questions

### Q1: When would you choose a bitmap index over a B-Tree index?

**Senior Answer:**

```
**Use Bitmap Index When:**

1. **Low cardinality (<100 distinct values)**
   - Example: gender (2-3 values), status (5-10 values), country (200 values)
   - Bitmap compression works best with few distinct values

2. **Read-heavy workload (OLAP, data warehouse)**
   - Minimal UPD ATES (bitmap updates are expensive)
   - Many SELECT queries with complex filters (AND/OR)
   - Bitmap operations (bitwise AND/OR) extremely fast

3. **Ad-hoc query patterns**
   - Users apply arbitrary combinations of filters
   - Example: Product catalog with category, brand, color, size filters
   - Bitmap: 4 indexes cover 2^4 = 16 filter combinations
   - B-Tree: Would need 16 composite indexes

4. **Star schema queries**
   - Fact table joins with dimension tables
   - Bitmap join indexes pre-compute joins
   - Avoid expensive hash joins at query time

5. **Storage constraints**
   - Bitmap indexes 5-50× smaller than B-Tree for low-cardinality
   - More indexes fit in buffer cache

**Use B-Tree Index When:**

1. **High cardinality (>1000 distinct values)**
   - Example: email, phone, SSN, UUID
   - Bitmap would be LARGER than B-Tree (1M bitmaps × 1M bits each)

2. **Write-heavy workload (OLTP)**
   - Frequent INSERTs/UPDATEs cause lock contention on bitmap words
   - B-Tree updates are more granular (single leaf node)

3. **Range queries**
   - WHERE price BETWEEN 100 AND 500
   - Bitmap doesn't support ranges (keys not sorted)
   - B-Tree maintains sorted order (range scans efficient)

4. **Sorting/ORDER BY**
   - Bitmap: Random order (must sort results)
   - B-Tree: Already sorted (ORDER BY is free)

5. **Unique constraints**
   - Primary keys, unique columns
   - Bitmap can't enforce uniqueness efficiently

**Decision Matrix:**

| Column | Cardinality | Queries | Workload | Index Type |
|--------|-------------|---------|----------|------------|
| gender | 3 | = only | OLAP | Bitmap ✅ |
| status | 5 | =, IN | OLAP | Bitmap ✅ |
| country | 200 | =, IN | OLAP | Bitmap ⚠️ (marginal) |
| age | 100 | =, >, < | Mixed | B-Tree (ranges needed) |
| email | 1M | =, LIKE | OLTP | B-Tree ✅ |
| user_id | 1M | =, > | OLTP | B-Tree ✅ |

**Real-world example:**

"At my previous company, we had a product catalog with 20M products and 100+ filter attributes (category, brand, color, size, material, etc.). Users could apply any combination of filters in the UI.

Initially, we used B-Tree composite indexes: (category, brand), (category, brand, color), etc. We needed 50+ indexes to cover common query patterns, totaling 200GB storage.

We switched to bitmap indexes on each low-cardinality attribute (20 indexes, 30GB total). Query performance improved 10× (bitwise operations vs B-Tree traversals) and storage dropped 85%. The ad-hoc query support was perfect — any filter combination worked efficiently."
```

---

### Q2: Explain how bitmap compression works and why clustered data compresses better.

**Staff Answer:**

```
**Bitmap Compression: Run-Length Encoding (RLE)**

**Basic RLE:**

Instead of storing individual bits (0 or 1), store runs of consecutive values:

```
Uncompressed bitmap:
[1,1,1,1,1,0,0,0,0,1,1,0,0,0,0,0,0]

RLE compressed:
[(1, 5), (0, 4), (1, 2), (0, 6)]
  ↑ Value, Count

Storage:
- Uncompressed: 17 bits = 3 bytes (rounded up)
- Compressed: 4 runs × 8 bytes per run = 32 bytes

Wait, this is WORSE! Why?
```

**RLE Only Helps with Long Runs:**

```
Worst case (alternating bits):
[1,0,1,0,1,0,1,0,1,0]

RLE:
[(1,1), (0,1), (1,1), (0,1), (1,1), (0,1), (1,1), (0,1), (1,1), (0,1)]
10 runs × 8 bytes = 80 bytes vs 10 bits = 2 bytes
→ 40× EXPANSION! ❌

Best case (long runs):
[1,1,1,...1 (1M times)...,0,0,0,...0 (1M times)...]

RLE:
[(1, 1000000), (0, 1000000)]
2 runs × 8 bytes = 16 bytes vs 2M bits = 250KB
→ 15,625× COMPRESSION! ✅
```

**Why Clustered Data Compresses Better:**

**Unclustered (random) data:**
```
Table: orders (1M rows, unsorted)
Column: status: ['shipped', 'pending', 'shipped', 'delivered', 'pending', ...]

Bitmap for 'shipped' (random distribution):
[1, 0, 1, 0, 0, 1, 0, 1, 1, 0, 1, 0, ...]

Run analysis:
- Average run length: 2 bits (very short runs)
- Number of runs: 500K runs (alternates frequently)
- Compression ratio: 2:1 (marginal)

RLE size:
- 500K runs × 8 bytes = 4MB
- vs raw: 1M bits = 125KB
- Useful compression, but not spectacular
```

**Clustered (sorted) data:**
```
Table: orders CLUSTERED BY status
Column: status: ['delivered', 'delivered', ..., 'pending', 'pending', ..., 'shipped', 'shipped', ...]
                  ↑ 100K delivered           ↑ 400K pending                ↑ 500K shipped

Bitmap for 'shipped' (clustered):
[0,0,0,...0 (500K zeros)...,1,1,1,...1 (500K ones)...]

Run analysis:
- Number of runs: 2 (one run of 0s, one run of 1s)
- Average run length: 500K bits (very long runs)
- Compression ratio: 7,800:1 (excellent!)

RLE size:
- 2 runs × 8 bytes = 16 bytes
- vs raw: 1M bits = 125KB
- 7,800× compression!
```

**Mathematical Explanation:**

```
Compression ratio = (Number of bits) / (Number of runs × bytes per run)

Unclustered:
- Bits: 1,000,000
- Runs: 500,000 (alternating)
- Compression: 1,000,000 / (500,000 × 8) = 0.25 (EXPANSION!)

Clustered:
- Bits: 1,000,000
- Runs: 2 (two long runs)
- Compression: 1,000,000 / (2 × 8 × 8 bits/byte) = 7,812.5 (massive compression)

Improvement: 7,812.5 / 0.25 = 31,250× better compression!
```

**Real-World Impact:**

```sql
-- Before clustering:
-- - 10 bitmap indexes
-- - Average compression: 2:1
-- - Total size: 500MB

CLUSTER orders USING idx_orders_status;
REINDEX TABLE orders;

-- After clustering:
-- - Same 10 bitmap indexes
-- - Average compression: 1000:1 (sorted by status, other columns benefit too)
-- - Total size: 1MB (500× smaller!)

-- Query performance:
-- - Before: 50% of bitmaps fit in cache (500MB cache, 500MB indexes)
-- - After: 100% of bitmaps fit in cache (500MB cache, 1MB indexes)
-- - Query time: 1 second → 0.01 seconds (100× faster)
```

**Advanced: Word-Aligned Hybrid (WAH) Compression:**

```
Problem: RLE must decompress before operations
Solution: WAH compresses in 32-bit words, allows operations on compressed data!

WAH format:
- Literal word: [0 | 31 bits of raw data]
- Fill word: [1 | fill_bit | 30-bit count]

Example:
[1111...1111 (64 ones), 0000...0000 (64 zeros), 1010...mixed (32 bits)]

WAH encoding:
  Word 0: [1 | 1 | 00...010]  ← Fill: 2 words of all 1s (64 bits)
  Word 1: [1 | 0 | 00...010]  ← Fill: 2 words of all 0s (64 bits)
  Word 2: [0 | 1010101010101010101010101010101]  ← Literal: 31 bits of mixed data

Compressed size: 3 words × 4 bytes = 12 bytes
vs uncompressed: 160 bits = 20 bytes
Compression: 1.67:1

Advantage: Bitwise AND on compressed words directly!

Compressed AND:
  [1|1|010] & [1|1|010] = [1|1|010]  ← Fill & Fill = Fill (no decompression!)
  [0|0101...] & [0|1010...] = [0|0000...]  ← Literal & Literal (32 AND ops in one instruction)

vs RLE: Must decompress both bitmaps, then AND → 10× slower
```

**Interview insight:**

"In a data warehouse project, we had a 10TB fact table that was initially unsorted. Bitmap indexes were 50GB total with poor compression. After analyzing query patterns, we found 80% of queries filtered by date.

We CLUSTERed the table by date (monthly partitions), which sorted data chronologically. This indirectly sorted other columns too (e.g., all January orders were together, and most January orders had similar status distributions).

After clustering, bitmap index size dropped from 50GB to 2GB (25× compression). Even better, queries that combined date + status filters became 100× faster because both bitmaps were highly compressed and cache-friendly. This one change saved $50K/year in storage costs and eliminated our query performance issues."
```

---

## 11. Summary

### Bitmap Index Key Principles

1. **Low-Cardinality Optimization**
   - Excellent: <10 distinct values (boolean, small enums)
   - Good: 10-100 distinct values (status, category)
   - Marginal: 100-1000 distinct values (country, product type)
   - Poor: >1000 distinct values (use B-Tree instead)

2. **Bitwise Operations are Fast**
   - AND, OR, XOR operations hardware-accelerated
   - Combine multiple filters efficiently (any combination)
   - 10-100× faster than combining B-Tree indexes

3. **Compression is Critical**
   - RLE compression ratio depends on data clustering
   - Clustered data: 1000:1 compression (excellent)
   - Random data: 2:1 compression (marginal)
   - **Action:** CLUSTER table by frequently filtered column

4. **Read-Optimized, Write-Penalized**
   - Perfect for OLAP/data warehouse (99%+ reads)
   - Terrible for OLTP (high write contention)
   - Bitmap updates lock entire word (32 rows)

### When to Use Bitmap Indexes

**✅ Ideal Scenarios:**
- Data warehouse / OLAP workloads (batch loads, no concurrent writes)
- Ad-hoc queries with arbitrary filter combinations
- Low-cardinality columns (<100 distinct values)
- Storage-constrained environments (bitmap 5-50× smaller)
- Star schema with dimension foreign keys

**❌ Avoid When:**
- OLTP workloads (high UPDATE rate causes lock contention)
- High-cardinality columns (>1000 distinct values → bitmap larger than B-Tree)
- Range queries needed (bitmap doesn't maintain sorted order)
- Need ORDER BY support (bitmap unsorted)

### Performance Characteristics

| Operation | Bitmap Index | B-Tree Index | Winner |
|-----------|--------------|--------------|--------|
| Exact match (=) | O(1) bitmap lookup | O(log N) tree traversal | Bitmap (marginal) |
| Multi-condition AND | O(K) bitwise AND (K = conditions) | O(K × log N) combine indexes | Bitmap (10×) |
| Multi-condition OR | O(K) bitwise OR | O(K × log N) combine indexes | Bitmap (10×) |
| Range query (>, <) | Not supported ❌ | O(log N + K) | B-Tree (only option) |
| ORDER BY | Not supported ❌ | Free (sorted) | B-Tree (only option) |
| INSERT/UPDATE | Slow (lock contention) | Fast (granular locks) | B-Tree |

### Storage Comparison (1M rows)

| Column | Cardinality | B-Tree Size | Bitmap Size (unclustered) | Bitmap Size (clustered) |
|--------|-------------|-------------|---------------------------|-------------------------|
| gender | 3 | 10MB | 400KB (25×) | 20KB (500×) |
| status | 5 | 10MB | 600KB (17×) | 30KB (333×) |
| country | 200 | 12MB | 20MB (0.6×) ❌ | 1MB (12×) |
| user_id | 1M | 100MB | 125GB (1250×) ❌ | N/A |

### Implementation Checklist

**Creating Bitmap Indexes:**
- [ ] Verify cardinality <100 distinct values
- [ ] Confirm read-heavy workload (>95% SELECT)
- [ ] Cluster table by frequently filtered column (maximize compression)
- [ ] Create separate bitmap indexes (not composite) for filter flexibility
- [ ] Monitor index size (should be <10% of table size)

**Maintenance:**
- [ ] REINDEX after significant data growth (restore compression)
- [ ] CLUSTER periodically (quarterly) if data unsorted
- [ ] Monitor query plans (ensure bitmaps are being used)
- [ ] Watch for lock contention (sign of write workload mismatch)

### Quick Wins

**1. Convert low-cardinality B-Tree to Bitmap (30 min, 5-10× smaller)**
```sql
-- Find candidates:
SELECT column_name, COUNT(DISTINCT column_name) AS cardinality
FROM table_name
GROUP BY column_name
HAVING COUNT(DISTINCT column_name) < 100;

DROP INDEX idx_old;
CREATE BITMAP INDEX idx_new ON table(column);
```

**2. Cluster table by query pattern (2 hours, 10-100× compression)**
```sql
CLUSTER table_name USING index_on_common_filter_column;
REINDEX TABLE table_name;
```

**3. Replace multiple composite indexes with bitmaps (1 hour, flexible filtering)**
```sql
-- Before: 3 composite indexes (cover some combos)
-- After: 3 bitmap indexes (cover ALL combos)
```

### Database-Specific Notes

**PostgreSQL:**
- No native bitmap indexes (planned, not yet implemented as of 2026)
- Uses "Bitmap Index Scan" internally (converts B-Tree to bitmap temporarily)

**Oracle:**
- Full bitmap index support (gold standard)
- Bitmap join indexes for star schemas
- Bitmap index organized tables (extreme compression)

**SQL Server:**
- No bitmap indexes
- Uses "rowstore" vs "columnstore" indexes instead

**MySQL:**
- No bitmap indexes
- InnoDB uses B-Tree only

**Remember:** Bitmap indexes are a specialized tool. Use for OLAP/data warehouse with low-cardinality columns. For most OLTP workloads, B-Tree is the right choice.

---

**Next:** `04_Covering_Indexes.md` - Eliminating heap access with Index-Only Scans
