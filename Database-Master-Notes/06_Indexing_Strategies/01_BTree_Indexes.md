# B-Tree Indexes - The Foundation of Database Performance

## 1. Concept Explanation

**B-Tree indexes** (specifically B+Tree in most databases) are the most common index structure in relational databases. They organize data in a balanced tree structure that maintains sorted order and provides efficient O(log N) lookups, insertions, and deletions.

Think of a B-Tree like a filing cabinet:
- **Root/Internal nodes:** Directory tabs telling you which drawer to open
- **Leaf nodes:** The actual files, sorted alphabetically
- **Balanced structure:** All drawers are at the same depth (equal access time)

### B-Tree vs B+Tree

Most databases use **B+Tree** (an enhancement of B-Tree):

```
B-Tree:
- Data stored in both internal and leaf nodes
- Less efficient range scans (must traverse tree for each value)

B+Tree (what databases actually use):
- Data stored ONLY in leaf nodes
- Internal nodes store only keys (more keys fit per page)
- Leaf nodes linked together (efficient range scans)
- All paths from root to leaf are same length (balanced)
```

### Structure Visualization

```
                    [50]                    ← Root (level 0)
                   /    \
                 /        \
            [20, 35]      [75, 90]          ← Internal nodes (level 1)
           /   |   \      /   |   \
         /     |     \  /     |     \
    [10,15] [25,30] [40,45] [60,70] [80,85] [95,100]  ← Leaf nodes (level 2)
       ↓       ↓       ↓       ↓       ↓       ↓
    (data)  (data)  (data)  (data)  (data)  (data)

Leaf nodes linked: [10,15] ↔ [25,30] ↔ [40,45] ↔ ...
                   (Enables efficient range scans)
```

**Key Properties:**
- **Order (fanout):** Each node stores N keys and N+1 child pointers
- **Balanced:** All leaf nodes at same depth
- **Sorted:** Keys within nodes are sorted
- **Self-balancing:** Automatically rebalances on insert/delete

---

## 2. Why It Matters

### Production Impact

**Without Index:**
```sql
-- Find user by email (1M users)
SELECT * FROM users WHERE email = 'alice@example.com';

-- Plan: Sequential Scan
-- Reads: 1,000,000 rows
-- I/O: 10,000 pages (assuming 100 rows/page)
-- Time: 5 seconds (reading all pages)
```

**With B-Tree Index:**
```sql
CREATE INDEX idx_users_email ON users(email);

SELECT * FROM users WHERE email = 'alice@example.com';

-- Plan: Index Scan using idx_users_email
-- Tree depth: 3 levels (1 root + 1 internal + 1 leaf)
-- Reads: 3 index pages + 1 table page = 4 pages
-- Time: 0.001 seconds (1ms)

-- Speedup: 5000× FASTER!
```

### Real Scenario: E-commerce Product Search

**Query:**
```sql
-- Search products by SKU
SELECT * FROM products WHERE sku = 'LAPTOP-X1-256GB';
```

**Before Index (10M products):**
```
Sequential Scan on products  (cost=0.00..200000.00 rows=1)
  Filter: (sku = 'LAPTOP-X1-256GB')
  Rows Removed by Filter: 9999999

-- Execution: 15 seconds
-- I/O: 100,000 pages read
-- CPU: 100% (scanning all rows)
```

**After B-Tree Index:**
```sql
CREATE UNIQUE INDEX idx_products_sku ON products(sku);

Index Scan using idx_products_sku on products  (cost=0.43..8.45 rows=1)
  Index Cond: (sku = 'LAPTOP-X1-256GB')

-- Execution: 0.003 seconds (3ms)
-- I/O: 4 pages (3 index + 1 heap)
-- CPU: 1%

-- Result: 5000× faster, site responsive ✅
```

---

## 3. Internal Working

### B+Tree Node Structure

**Internal Node (non-leaf):**
```
+------------------+------------------+------------------+
| Keys: [25, 50]   |                  |                  |
+------------------+------------------+------------------+
| Pointers: [P0, P1, P2]              |                  |
+------------------+------------------+------------------+

P0 → child with keys < 25
P1 → child with keys >= 25 and < 50
P2 → child with keys >= 50

Size: Up to ~300 keys per 8KB page (depending on key size)
```

**Leaf Node:**
```
+------------------+------------------+------------------+
| Keys: [40, 42, 45, 47, 49]          |                  |
+------------------+------------------+------------------+
| Row IDs: [rid1, rid2, rid3, ...]    |                  |
+------------------+------------------+------------------+
| Next Leaf Pointer: → (next leaf)    |                  |
+------------------+------------------+------------------+

Row ID (RID) points to heap page + offset where actual row lives

Size: Up to ~200 entries per 8KB page (key + pointer)
```

### Search Algorithm

```
FUNCTION btree_search(value):
    current_node = root
    
    WHILE current_node is not leaf:
        # Binary search within node to find child pointer
        child_index = binary_search(current_node.keys, value)
        current_node = follow_pointer(current_node.pointers[child_index])
    
    # Now at leaf node
    position = binary_search(current_node.keys, value)
    
    IF current_node.keys[position] == value:
        RETURN current_node.row_ids[position]  # Found!
    ELSE:
        RETURN NOT_FOUND

# Complexity: O(log N) where N = total keys
# Example: 1M keys, depth 3 → 3 page reads
```

**Example Trace:**
```sql
-- Search for: email = 'charlie@example.com'
-- Assuming 100 keys per node, 1M total emails

Step 1: Read root node
  Keys: [a...@..., j...@..., q...@..., z...@...]
  Binary search: 'charlie' between 'a' and 'j'
  Follow pointer to child node

Step 2: Read internal node (level 1)
  Keys: [a...@..., c...@..., e...@..., g...@..., i...@...]
  Binary search: 'charlie' between 'c' and 'e'
  Follow pointer to child node

Step 3: Read leaf node (level 2)
  Keys: ['bob@...', 'charlie@...', 'david@...']
  Binary search: Found 'charlie@example.com' at position 1!
  Row ID: (page=12345, offset=89)

Step 4: Read heap page
  Fetch row from page 12345, offset 89

Total I/O: 4 pages (3 index + 1 heap)
```

### Insert Algorithm with Page Splits

```
FUNCTION btree_insert(key, value):
    # Find appropriate leaf node
    leaf = search_leaf_node(key)
    
    IF leaf has space:
        # Simple case: Insert into leaf
        leaf.insert_sorted(key, value)
    ELSE:
        # Page is full → SPLIT!
        split_leaf_node(leaf, key, value)

FUNCTION split_leaf_node(leaf, key, value):
    # Create new leaf node
    new_leaf = allocate_leaf_node()
    
    # Add new key to temporary buffer
    temp = leaf.keys + [key]
    temp.sort()
    
    # Split keys in half
    mid = len(temp) / 2
    leaf.keys = temp[0:mid]
    new_leaf.keys = temp[mid:]
    
    # Update linked list
    new_leaf.next = leaf.next
    leaf.next = new_leaf
    
    # Promote middle key to parent (may cause parent split)
    promote_to_parent(leaf.parent, temp[mid], new_leaf)

FUNCTION promote_to_parent(parent, key, new_child):
    IF parent has space:
        parent.insert(key, new_child)
    ELSE:
        # Parent full → Split parent (recursive)
        split_internal_node(parent, key, new_child)
```

**Split Example:**
```
Before insert (leaf full):
Leaf: [10, 20, 30, 40, 50]  (max 5 keys)

Insert: 35

Temporary buffer: [10, 20, 30, 35, 40, 50]
Split at mid (3): [10, 20, 30] | [35, 40, 50]

After split:
Old leaf: [10, 20, 30]
New leaf: [35, 40, 50]

Promote to parent: key=35, pointer to new leaf

Parent before:
  [50]
  [P0, P1]

Parent after:
  [35, 50]
  [P0, P1, P2]
  
Where P0 → old leaf [10,20,30]
      P1 → new leaf [35,40,50]
      P2 → existing leaf [60,70,80]
```

### Range Scan Optimization

**Why B+Tree is Perfect for Range Queries:**
```sql
-- Find users created in date range
SELECT * FROM users WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';

-- B+Tree traversal:
1. Navigate to leaf with '2024-01-01' → O(log N)
2. Follow leaf pointers sequentially until '2024-01-31'
3. No tree traversal needed for remaining keys!

-- Complexity: O(log N + K) where K = result size
-- Example: 1M users, 10K in Jan → log(1M) + 10K ≈ 20 + 10,000 = 10,020 operations
```

**Sequential vs B-Tree for Range:**
```
Sequential Scan:
- Read all 1M users, filter by date
- I/O: 100,000 pages (all)
- Time: 10 seconds

B+Tree Index Scan:
- Navigate to first key: 3 pages (tree depth)
- Read sequential leaf pages with matching keys: 100 pages
- Read heap pages for matching rows: 10,000 pages
- Total I/O: 10,103 pages
- Time: 1 second (10× faster)
```

---

## 4. Best Practices

### Practice 1: Choose Appropriate Fill Factor

**❌ Bad: Default Fill Factor (100%)**
```sql
-- Default: Pages completely full
CREATE INDEX idx_orders_id ON orders(id);

-- Insert new order:
-- If page full → Split page (expensive)
-- Write amplification: Split writes 2 pages instead of 1
```

**✅ Good: Lower Fill Factor for Write-Heavy Tables**
```sql
-- Set 70% fill factor (leaves 30% free space)
-- PostgreSQL:
CREATE INDEX idx_orders_id ON orders(id) WITH (fillfactor = 70);

-- SQL Server:
CREATE INDEX idx_orders_id ON orders(id) WITH (FILLFACTOR = 70);

-- Benefits:
-- - Fewer page splits (free space absorbs inserts)
-- - Less fragmentation
-- - Trade-off: 30% more disk space

-- When to use:
-- - High INSERT/UPDATE rate
-- - Random key distribution (UUIDs, emails)
-- - When NOT to use: Read-only tables (wasted space)
```

**Fill Factor Guidelines:**
| Workload | Fill Factor | Reason |
|----------|-------------|--------|
| Read-only / OLAP | 100% | No updates, maximize caching |
| Moderate writes (10% INSERT/UPDATE) | 80-90% | Balance space/split frequency |
| Heavy writes (>50% INSERT/UPDATE) | 70-80% | Minimize splits |
| Append-only (sequential keys) | 100% | No splits (keys always at end) |

### Practice 2: Index Column Order for Composite Indexes

**Composite B-Tree Structure:**
```
Index on (country, city, age):

Sorted as: (country, city, age) lexicographically

('US', 'NYC', 25)
('US', 'NYC', 30)
('US', 'SF', 25)
('US', 'SF', 30)
('UK', 'London', 25)
('UK', 'London', 30)
```

**❌ Bad: Wrong Column Order**
```sql
-- Index: (age, country, city)
CREATE INDEX idx_users_bad ON users(age, country, city);

-- Query:
WHERE country = 'US' AND city = 'NYC' AND age = 30;

-- Cannot use index efficiently!
-- Must scan all ages, then filter (not selective)
```

**✅ Good: Selective Column First**
```sql
-- Index: (country, city, age)
CREATE INDEX idx_users_good ON users(country, city, age);

-- Query:
WHERE country = 'US' AND city = 'NYC' AND age = 30;

-- Uses all 3 columns efficiently:
-- 1. Navigate to 'US' subtree (eliminates 50% of data)
-- 2. Within 'US', navigate to 'NYC' (eliminates 95% of 'US' data)
-- 3. Within 'US'+'NYC', find age=30 (final filter)

-- Rule: Most selective column first
-- Exception: If query always filters on specific column, put it first
```

### Practice 3: Monitor and Rebuild Fragmented Indexes

**Problem: Fragmentation Over Time**
```
Initial state (sequential inserts):
Page 1: [keys 1-100]
Page 2: [keys 101-200]
Page 3: [keys 201-300]
(Sequential, no fragmentation)

After 1 year (random inserts/updates/deletes):
Page 1: [keys 5, 78, 199]  ← Sparse (70% empty)
Page 7: [keys 10, 102]     ← Out of order
Page 2: [keys 50, 110, 250] ← Mixed ranges

↓ Result:
- Range scans read many pages (not sequential)
- Cache inefficient (jumps between pages)
- Wasted disk space (pages 70% empty)
```

**Detect Fragmentation:**
```sql
-- PostgreSQL:
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan > 100  -- Only check used indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Check bloat:
SELECT
    tablename,
    ROUND((CASE WHEN otta=0 THEN 0.0 
           ELSE sml.relpages::FLOAT/otta END) * 100, 1) AS bloat_pct
FROM (
    -- Complex query from pgstattuple extension
    SELECT schemaname, tablename, cc.relpages, ...
    FROM pg_class cc
    JOIN pg_namespace nn ON cc.relnamespace = nn.oid
    ...
) AS sml
WHERE bloat_pct > 20;  -- Bloat > 20% = rebuild candidate

-- SQL Server:
SELECT 
    object_name(ips.object_id) AS table_name,
    i.name AS index_name,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30  -- >30% fragmentation
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

**Rebuild Fragmented Indexes:**
```sql
-- PostgreSQL: REINDEX (locks table)
REINDEX INDEX idx_orders_created;
-- Or: REINDEX TABLE orders;  (all indexes)

-- PostgreSQL 12+: REINDEX CONCURRENTLY (no lock)
REINDEX INDEX CONCURRENTLY idx_orders_created;

-- SQL Server: REBUILD (offline)
ALTER INDEX idx_orders_created ON orders REBUILD;

-- SQL Server: REBUILD ONLINE (minimal lock)
ALTER INDEX idx_orders_created ON orders REBUILD WITH (ONLINE = ON);

-- MySQL: Rebuild via OPTIMIZE TABLE
OPTIMIZE TABLE orders;
```

**Rebuild Schedule:**
```sql
-- Weekly maintenance window:
DO $$
DECLARE
    idx RECORD;
BEGIN
    FOR idx IN 
        SELECT indexname 
        FROM pg_stat_user_indexes 
        WHERE idx_scan > 1000  -- Only used indexes
    LOOP
        EXECUTE 'REINDEX INDEX CONCURRENTLY ' || idx.indexname;
    END LOOP;
END $$;
```

### Practice 4: Use Covering Indexes for Hot Queries

**❌ Bad: Index + Heap Lookup**
```sql
CREATE INDEX idx_orders_user ON orders(user_id);

-- Query:
SELECT order_id, total, created_at
FROM orders
WHERE user_id = 12345;

-- Plan:
Index Scan using idx_orders_user on orders
  Index Cond: (user_id = 12345)
  
-- Steps:
-- 1. B-Tree search: Find user_id=12345 in index (3 pages)
-- 2. For each matching index entry (50 orders):
--      Read heap page to get (order_id, total, created_at)
-- Total I/O: 3 (index) + 50 (heap) = 53 pages
```

**✅ Good: Covering Index (Index-Only Scan)**
```sql
-- PostgreSQL:
CREATE INDEX idx_orders_user_covering 
ON orders(user_id) 
INCLUDE (order_id, total, created_at);

-- SQL Server:
CREATE INDEX idx_orders_user_covering 
ON orders(user_id) 
INCLUDE (order_id, total, created_at);

-- MySQL/Older PostgreSQL:
CREATE INDEX idx_orders_user_covering 
ON orders(user_id, order_id, total, created_at);

-- Query:
SELECT order_id, total, created_at
FROM orders
WHERE user_id = 12345;

-- Plan:
Index Only Scan using idx_orders_user_covering on orders
  Index Cond: (user_id = 12345)
  Heap Fetches: 0  ← No heap access!

-- Steps:
-- 1. B-Tree search: Find user_id=12345 in index (3 pages)
-- 2. Read all needed columns from index leaf pages (5 pages)
-- Total I/O: 3 + 5 = 8 pages (vs 53 before)

-- Result: 6× fewer I/O, 5× faster queries
```

---

## 5. Common Mistakes

### Mistake 1: Creating Index on Every Column

**Problem:**
```sql
-- "More indexes = faster!" (WRONG!)
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_name ON users(name);
CREATE INDEX idx_users_age ON users(age);
CREATE INDEX idx_users_country ON users(country);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_users_updated ON users(updated_at);
-- ...20 indexes total

-- Impact:
-- - INSERT: Updates 20 indexes (write amplification: 20×)
-- - Disk space: 5× table size in indexes
-- - Cache pollution: 20 indexes compete for buffer cache
-- - Planning overhead: Optimizer evaluates 20 index options
```

**Solution:**
```sql
-- Audit which indexes are actually used:
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC;

-- Drop unused indexes:
-- idx_users_age: 0 scans, 50MB → DROP
-- idx_users_status: 5 scans in 6 months → DROP
DROP INDEX idx_users_age;
DROP INDEX idx_users_status;

-- Guidelines:
-- - OLTP tables: 3-5 indexes typical
-- - Data warehouse: 5-10 indexes acceptable
-- - >15 indexes: Red flag (audit needed)
```

### Mistake 2: Indexing Low-Cardinality Columns

**Problem:**
```sql
-- status has only 3 values: 'active', 'inactive', 'deleted'
CREATE INDEX idx_users_status ON users(status);

-- Query:
SELECT * FROM users WHERE status = 'active';

-- Optimizer decision:
Sequential Scan on users  (cost=0.00..20000.00 rows=900000)
  Filter: (status = 'active')

-- Index ignored! Why?
-- - 90% of users are 'active' (low selectivity)
-- - Index scan cost: 900K random heap reads = 3,600,000 cost
-- - Seq scan cost: 10K sequential page reads = 10,000 cost
-- - Seq scan 360× cheaper!
```

**Solution:**
```sql
-- Option 1: Drop index (not useful)
DROP INDEX idx_users_status;

-- Option 2: Partial index for rare value (if querying that)
CREATE INDEX idx_users_deleted ON users(status) WHERE status = 'deleted';
-- Only indexes 10% of rows (deleted users)
-- Query WHERE status = 'deleted' is 10× faster
-- Query WHERE status = 'active' still uses seq scan (correct choice)

-- Option 3: Composite index with selective column
CREATE INDEX idx_users_country_status ON users(country, status);
-- country narrows down first (selective)
-- status filters within country
```

**Guideline:**
- Index useful when selectivity < 5-10% of table
- Above that threshold, seq scan usually faster

### Mistake 3: Not Considering Index Size

**Problem:**
```sql
-- Index on TEXT column with large values
CREATE INDEX idx_documents_content ON documents(content);
-- content: Average 100KB per document

-- With 1M documents:
-- Index size: 1M × 100KB = 100GB
-- vs table size: 1M × 100KB = 100GB
-- Total: 200GB (2× storage)

-- Performance issues:
-- - Index doesn't fit in memory (cache thrashing)
-- - B-Tree depth: 5-6 levels (more I/O per lookup)
-- - Page splits: Expensive (moving 100KB values)
```

**Solution:**
```sql
-- Option 1: Hash index on content (for equality only)
CREATE INDEX idx_documents_content_hash ON documents USING HASH (md5(content));
-- Stores 32-byte hash instead of full content

-- Option 2: Full-text index (for search)
CREATE INDEX idx_documents_content_fts ON documents USING GIN (to_tsvector('english', content));
-- Inverted index, stores tokens not full text

-- Option 3: Prefix index (first N characters)
-- MySQL:
CREATE INDEX idx_documents_content_prefix ON documents(content(100));
-- Indexes first 100 characters only

-- Option 4: Don't index, use external search (Elasticsearch)
```

### Mistake 4: Ignoring Index Bloat After Mass Deletes

**Problem:**
```sql
-- Delete 90% of old orders
DELETE FROM orders WHERE created_at < '2020-01-01';  -- Deletes 9M / 10M rows

-- Index still contains dead entries:
-- Index size before: 800MB
-- Index size after DELETE: 800MB (unchanged!)

-- Why?
-- - PostgreSQL: Dead tuples marked as deleted, not immediately removed
-- - Index pages mostly empty (90% dead entries)
-- - Queries scan large index with few live entries (slow)
```

**Solution:**
```sql
-- After mass DELETE/UPDATE, rebuild indexes:
REINDEX TABLE orders;

-- Or use VACUUM FULL (reclaims space):
VACUUM FULL orders;

-- After rebuild:
-- Index size: 80MB (90% reduction, only live entries)
-- Queries: 10× faster (compact index)

-- Schedule: After major data cleanup operations
```

---

## 6. Security Considerations

### 1. Index Data Leakage

**Risk:**
```sql
-- Index stores column values
CREATE INDEX idx_users_ssn ON users(ssn);  -- Social Security Number

-- Database backup includes indexes
-- Attacker with backup file can read index:
-- - SSNs stored in index leaf pages
-- - Even if table data encrypted, indexes might not be
```

**Mitigation:**
```sql
-- Option 1: Don't index sensitive columns
-- Use application-level caching instead

-- Option 2: Encrypt at column level
CREATE INDEX idx_users_ssn_encrypted ON users(pgp_sym_encrypt(ssn, 'key'));
-- Index stores encrypted values
-- Query: WHERE pgp_sym_encrypt(ssn, 'key') = pgp_sym_encrypt('123-45-6789', 'key')

-- Option 3: Hash-based index (irreversible)
CREATE INDEX idx_users_ssn_hash ON users(sha256(ssn::bytea));
-- Cannot reverse hash to get SSN
-- Query: WHERE sha256(ssn::bytea) = sha256('123-45-6789'::bytea)
```

### 2. Timing Attacks via Index Presence

**Risk:**
```sql
-- Attacker can infer data existence via response time

-- With index:
SELECT * FROM users WHERE email = 'admin@company.com';
-- Fast (1ms) → Email exists

SELECT * FROM users WHERE email = 'nonexistent@example.com';
-- Very fast (0.1ms) → Email doesn't exist (early termination)

-- Attacker can enumerate valid emails via timing
```

**Mitigation:**
- Constant-time queries (add artificial delay)
- Rate limiting on authentication endpoints
- Use CAPTCHA after N failed attempts

---

## 7. Performance Optimization

### Optimization 1: Index on Foreign Keys

**Problem: Joins Without Index on FK**
```sql
-- Schema:
-- orders(order_id, customer_id, ...)  10M rows
-- customers(customer_id, ...)          1M rows

-- Query:
SELECT c.customer_name, COUNT(o.order_id)
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name;

-- Without index on orders.customer_id:
Hash Join  (cost=500000.00..800000.00 rows=1000000) (actual time=35000..75000 rows=1000000)
  Hash Cond: (o.customer_id = c.customer_id)
  -> Seq Scan on orders o  (cost=0.00..200000.00 rows=10000000)  ← Full table scan!
       (actual time=0.5..8000 rows=10000000)
  -> Hash  (cost=50000.00..50000.00 rows=1000000)
       -> Seq Scan on customers c
  
-- Execution: 75 seconds (terrible!)
```

**Solution: Index Foreign Keys**
```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
ANALYZE orders;

-- Now:
Hash Join  (cost=50000.00..120000.00 rows=1000000) (actual time=500..2500 rows=1000000)
  Hash Cond: (o.customer_id = c.customer_id)
  -> Index Scan using idx_orders_customer_id on orders o
       (cost=0.43..80000.00 rows=10000000)
       (actual time=0.1..1500 rows=10000000)
  -> Hash  (cost=50000.00..50000.00 rows=1000000)
       -> Seq Scan on customers c

-- Execution: 2.5 seconds (30× FASTER!)

-- Rule: ALWAYS index foreign key columns
-- Why: Foreign keys are JOIN columns (used in 80% of multi-table queries)
```

### Optimization 2: Clustered Index Pattern

**Problem: Logical vs Physical Order Mismatch**
```sql
-- Index on created_at (sorted)
CREATE INDEX idx_orders_created ON orders(created_at);

-- But table rows physically ordered by order_id (random for created_at)
-- Correlation between index and table: 0.05 (5%, nearly random)

-- Query:
SELECT * FROM orders WHERE created_at >= '2024-01-01' LIMIT 1000;

-- Index scan reads:
-- - Index pages: 3 (navigate) + 10 (range scan) = 13 pages
-- - Heap pages: 1000 random pages (each row on different page)
-- Total I/O: 1013 pages

-- Time: 1.5 seconds (random I/O expensive)
```

**Solution: CLUSTER Table by Index**
```sql
-- Reorder table rows to match index order
CLUSTER orders USING idx_orders_created;
-- Physically sorts table by created_at

-- After CLUSTER:
-- Rows with adjacent created_at values are on same pages

-- Same query:
SELECT * FROM orders WHERE created_at >= '2024-01-01' LIMIT 1000;

-- Index scan reads:
-- - Index pages: 13 pages (same as before)
-- - Heap pages: 10 pages (1000 rows / 100 rows per page, sequential)
-- Total I/O: 23 pages (vs 1013 before)

-- Time: 0.05 seconds (30× FASTER!)

-- Caveat: CLUSTER is one-time operation
-- - Future INSERTs append to end (order degrades over time)
-- - Solution: Re-CLUSTER periodically (weekly/monthly)
```

### Optimization 3: Index Compression (PostgreSQL)

**Problem: Large Indexes**
```sql
-- Index on UUID column (16 bytes each)
CREATE INDEX idx_events_uuid ON events(event_uuid);

-- 100M events:
-- Index size: 100M × (16 bytes UUID + 8 bytes pointer) = 2.4GB

-- Issues:
-- - Buffer cache: 2.4GB of RAM for one index
-- - Tree depth: 4-5 levels (more I/O per lookup)
```

**Solution: Use B-Tree Compression**
```sql
-- PostgreSQL doesn't auto-compress, but we can optimize:

-- Option 1: Use BRIN index for sequential data
CREATE INDEX idx_events_created_brin ON events USING BRIN (created_at);
-- BRIN: Block Range INdex (stores min/max per page range)
-- Size: 1MB (vs 2.4GB for B-Tree)
-- Trade-off: Slightly less accurate (scans page ranges)

-- Option 2: Partial index (if querying subset)
CREATE INDEX idx_events_recent ON events(event_uuid) WHERE created_at >= NOW() - INTERVAL '30 days';
-- Only indexes recent events (95% smaller)

-- Option 3: Hash of UUID (for equality only)
CREATE INDEX idx_events_uuid_hash ON events USING HASH (event_uuid);
-- Hash index: 8 bytes per entry (vs 16 for UUID)
-- Trade-off: No range scans
```

---

## 8. Examples

### Example 1: Composite Index for Complex Query

**Scenario:** Multi-tenant SaaS, query users by tenant + status

**Query:**
```sql
-- Find active users for specific tenant
SELECT user_id, email, last_login
FROM users
WHERE tenant_id = 'acme-corp'
  AND status = 'active'
  AND subscription_tier = 'premium'
ORDER BY last_login DESC
LIMIT 50;
```

**Attempt 1: Single-Column Indexes**
```sql
CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_tier ON users(subscription_tier);

-- Plan:
Bitmap Heap Scan on users  (cost=5000.00..15000.00 rows=50)
  Recheck Cond: (tenant_id = 'acme-corp' AND status = 'active' AND subscription_tier = 'premium')
  -> BitmapAnd
       -> Bitmap Index Scan on idx_users_tenant
       -> Bitmap Index Scan on idx_users_status
       -> Bitmap Index Scan on idx_users_tier
  -> Sort
       Sort Key: last_login DESC

-- Execution: 0.5 seconds
-- Issues:
-- - Combines 3 indexes (overhead)
-- - Bitmap scans less efficient than index scan
-- - Still needs sort (last_login not indexed)
```

**Attempt 2: Composite Index (Optimal)**
```sql
DROP INDEX idx_users_tenant;
DROP INDEX idx_users_status;
DROP INDEX idx_users_tier;

-- Create composite index with all filter columns + sort column
CREATE INDEX idx_users_tenant_status_tier_login 
ON users(tenant_id, status, subscription_tier, last_login DESC);

-- Plan:
Index Scan using idx_users_tenant_status_tier_login on users
  Index Cond: (tenant_id = 'acme-corp' AND status = 'active' AND subscription_tier = 'premium')
  Rows: 5000
  Limit: 50

-- Execution: 0.005 seconds (100× FASTER!)

-- Why faster:
-- - Single index scan (no bitmap combination)
-- - All filters use index efficiently
-- - Results already sorted by last_login (no sort phase)
-- - Index-only scan possible if adding email to INCLUDE
```

---

### Example 2: Fixing Slow Pagination Query

**Scenario:** REST API pagination on large table

**Query:**
```sql
-- Page 1000 of users (50 per page)
SELECT user_id, username, email, created_at
FROM users
ORDER BY created_at DESC
LIMIT 50 OFFSET 49950;  -- Page 1000: skip 49,950 rows
```

**Naive Approach (Slow):**
```sql
CREATE INDEX idx_users_created ON users(created_at);

-- Plan:
Limit  (cost=50000.00..50100.00 rows=50)
  -> Index Scan Backward using idx_users_created on users
       (cost=0.43..100000.00 rows=1000000)
       Rows Removed by Offset: 49950  ← Scans and discards 49,950 rows!

-- Execution: 2 seconds (scans 49,950 rows to skip them)
-- Problem: OFFSET is slow for high page numbers
```

**Optimized: Keyset Pagination**
```sql
-- Instead of OFFSET, use WHERE filter on last seen key

-- Page 1:
SELECT user_id, username, email, created_at
FROM users
ORDER BY created_at DESC
LIMIT 50;
-- Returns users with created_at: 2024-01-31 23:59:59 (last row)

-- Page 2:
SELECT user_id, username, email, created_at
FROM users
WHERE created_at < '2024-01-31 23:59:59'  -- Continue from last row
ORDER BY created_at DESC
LIMIT 50;

-- Page 1000 (same query pattern):
SELECT user_id, username, email, created_at
FROM users
WHERE created_at < '2023-12-01 00:00:00'  -- Last row from page 999
ORDER BY created_at DESC
LIMIT 50;

-- Plan:
Index Scan Backward using idx_users_created on users
  Index Cond: (created_at < '2023-12-01 00:00:00')
  Limit: 50

-- Execution: 0.05 seconds (40× FASTER!)
-- Scans only 50 rows (not 49,950)

-- Benefits:
-- - Constant time regardless of page number
-- - Works for any page (page 1 or page 10,000)
-- - Requires passing "cursor" (last row's created_at) to client
```

---

## 9. Real-World Use Cases

### Use Case 1: Time-Series Data (IoT Sensors)

**Scenario:** 1B sensor readings, query last 24h frequently

**Table Structure:**
```sql
CREATE TABLE sensor_readings (
    reading_id BIGSERIAL,
    sensor_id INTEGER,
    timestamp TIMESTAMPTZ,
    value DECIMAL,
    PRIMARY KEY (timestamp, sensor_id)  -- Composite primary key
);
```

**Index Strategy:**
```sql
-- Primary key already creates index on (timestamp, sensor_id)
-- This supports both queries:

-- Query 1: All sensors in time range
SELECT * FROM sensor_readings
WHERE timestamp >= NOW() - INTERVAL '24 hours';
-- Uses primary key index efficiently (timestamp is first column)

-- Query 2: Specific sensor over time
SELECT * FROM sensor_readings
WHERE sensor_id = 12345 AND timestamp >= NOW() - INTERVAL '7 days';
-- Uses primary key index (sensor_id second column, but range on timestamp still helps)

-- If Query 2 is slow, add reverse index:
CREATE INDEX idx_sensor_readings_sensor_time ON sensor_readings(sensor_id, timestamp);

-- Now both queries optimal with respective indexes
```

**Partition Strategy (Bonus):**
```sql
-- Partition by time (1 partition per day)
CREATE TABLE sensor_readings (
    ...
) PARTITION BY RANGE (timestamp);

CREATE TABLE sensor_readings_2024_02_21 PARTITION OF sensor_readings
    FOR VALUES FROM ('2024-02-21') TO ('2024-02-22');

-- Each partition has its own index (smaller B-Tree)
-- Query on recent data only touches 1-7 partitions (fast)
-- Old partitions can be dropped (data retention policy)
```

### Use Case 2: Multi-Tenant SaaS Database

**Scenario:** 10K tenants, varying sizes (10-1M users per tenant)

**Table:**
```sql
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,
    tenant_id UUID NOT NULL,
    email VARCHAR(255),
    status VARCHAR(20),
    created_at TIMESTAMPTZ,
    ...
);
```

**Index Strategy:**
```sql
-- All queries filter by tenant_id first (tenant isolation)
-- Composite indexes start with tenant_id:

-- Index 1: Tenant + email (login)
CREATE UNIQUE INDEX idx_users_tenant_email ON users(tenant_id, email);

-- Index 2: Tenant + status (list active users)
CREATE INDEX idx_users_tenant_status ON users(tenant_id, status) WHERE status = 'active';
-- Partial index: Only active users (90% smaller)

-- Index 3: Tenant + created (recent users)
CREATE INDEX idx_users_tenant_created ON users(tenant_id, created_at DESC);

-- Benefits:
-- - All queries filtered to tenant first (99% data reduction)
-- - Small tenants: Index scan (fast)
-- - Large tenants: Might use seq scan within tenant (correct choice)
```

---

## 10. Interview Questions

### Q1: Explain how B-Tree handles concurrent inserts. Do page splits lock the entire tree?

**Staff Answer:**

```
**Concurrent Insert Mechanism:**

B-Trees use **latch coupling** (aka **lock coupling**) to allow concurrent operations without locking the entire tree.

**Algorithm:**

```
INSERT operation:
1. Acquire SHARE latch on root
2. Navigate down tree:
   - Acquire SHARE latch on child node
   - Release latch on parent node
   - Repeat until leaf node reached

3. At leaf node:
   a. Upgrade to EXCLUSIVE latch on leaf
   b. If leaf has space:
      - Insert key-value pair
      - Release latch
      - DONE (no escalation)
   
   c. If leaf is full (needs split):
      - Keep EXCLUSIVE latch on leaf
      - Acquire EXCLUSIVE latch on parent
      - Perform split:
         * Allocate new leaf node
         * Move half the keys to new leaf
         * Insert pointer to new leaf in parent
      - Release latches
```

**Example with contention:**

```
Thread 1: INSERT key = 42
Thread 2: INSERT key = 45 (concurrent)

Time T0:
  Thread 1: SHARE latch on root
  Thread 2: SHARE latch on root  ← Both allowed (SHARE)

Time T1:
  Thread 1: Navigate to leaf node L1, acquire SHARE
  Thread 2: Navigate to leaf node L1, acquire SHARE  ← Both reading same leaf

Time T2:
  Thread 1: Upgrade to EXCLUSIVE on L1... BLOCKED (Thread 2 has SHARE)
  Thread 2: Upgrade to EXCLUSIVE on L1... BLOCKED (Thread 1 trying same)

Time T3:
  Deadlock detection: One thread aborts and retries
  Thread 1: Gets EXCLUSIVE on L1
  Thread 2: Waiting...

Time T4:
  Thread 1: Insert key=42, release EXCLUSIVE
  Thread 2: Gets EXCLUSIVE on L1, insert key=45, release
```

**Page Split Locking:**

When a page split occurs:

1. **Leaf split:** Only leaf node and immediate parent locked
   - Tree mostly available for other operations
   - Readers can still access other branches

2. **Cascading split (parent full):**
   - Locks propagate upward (leaf → parent → grandparent)
   - In worst case: Locks path from leaf to root
   - BUT: Other branches still accessible

3. **Root split (tree grows):**
   - Must lock root exclusively
   - New root created (brief exclusive lock)
   - Most contentious operation, but rare

**Performance characteristics:**

- **Most inserts:** Only leaf-level latch (no contention)
- **Split at leaf:** Locks 2 nodes (leaf + parent) briefly
- **Root split:** Global lock, but very rare (tree depth 3→4 might happen once per million inserts)

**Real-world impact:**

```
Benchmark: 1000 concurrent INSERT threads on 10M row table

Without latch coupling (coarse-grained lock):
  - Lock entire tree per insert
  - Throughput: 1000 inserts/sec (serialized)

With latch coupling:
  - Lock only path from root to leaf
  - Throughput: 50,000 inserts/sec (50× faster)
  - Contention only when threads insert into same leaf
```

**Database-specific implementations:**

- **PostgreSQL:** Uses buffer locks (lightweight latches), no deadlock detection (careful ordering instead)
- **InnoDB (MySQL):** Uses latch coupling with deadlock detection
- **SQL Server:** Uses latch hints to avoid deadlocks (optimistic)

**Follow-up insight:**

"In production, I've seen page split storms under high insert load cause brief latency spikes. The solution was two-fold: (1) Increase fill factor to 70% to provide free space, reducing split frequency, and (2) Use partitioning to distribute inserts across multiple tree structures, reducing contention."
```

### Q2: Why do databases use B+Tree instead of hash tables for indexes?

**Senior Answer:**

```
**B+Tree vs Hash Table Trade-offs:**

| Feature | B+Tree | Hash Table |
|---------|--------|------------|
| **Exact match (WHERE x = ?)** | O(log N) | O(1)* faster |
| **Range query (WHERE x > ?)** | O(log N + K) ✅ | Not supported ❌ |
| **Sorted output (ORDER BY)** | Free (already sorted) ✅ | Requires sort ❌ |
| **Prefix match (LIKE 'abc%')** | Supported ✅ | Not supported ❌ |
| **MIN/MAX** | O(log N) read leftmost/rightmost ✅ | O(N) full scan ❌ |
| **Disk I/O pattern** | Sequential (predictable) ✅ | Random (cache-unfriendly) |
| **Space efficiency** | 70-90% page fill | 50-70% (hash collisions) |
| **Concurrency** | Lock small subtrees | Lock hash buckets (more contention) |

(*O(1) assumes no hash collisions; worst case O(N))

**Why B+Tree Wins in Practice:**

**1. Range queries are everywhere**
```sql
-- 80% of real-world queries use ranges:
WHERE created_at >= '2024-01-01'
WHERE price BETWEEN 100 AND 500
WHERE name LIKE 'A%'
WHERE age > 18

-- Hash can't handle any of these!
```

**2. Sorted order matters**
```sql
-- B+Tree: Already sorted, no extra work
SELECT * FROM users ORDER BY email LIMIT 100;

-- Hash: Must sort 1M rows, then take top 100
-- Cost: O(N log N) sort vs O(log N) B-Tree lookup
```

**3. Disk I/O patterns**

B+Tree read pattern (range scan):
```
Pages: [1] → [2] → [3] → [4]  ← Sequential I/O (fast)
SSD: 500 MB/s throughput
HDD: 100 MB/s (still decent)
```

Hash table read pattern (multiple lookups):
```
Pages: [1] → [847] → [23] → [1203] ← Random I/O (slow)
SSD: 50 MB/s (10× slower due to random seeks)
HDD: 2 MB/s (50× slower)
```

**4. Practical example:**

```sql
-- Query: Find orders in date range
SELECT * FROM orders 
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';

-- B+Tree index on created_at:
-- 1. Seek to '2024-01-01' (3 page reads: root → internal → leaf)
-- 2. Scan sequentially until '2024-01-31' (100 pages)
-- Total: 103 pages, sequential
-- Time: 0.1 seconds

-- Hash index on created_at:
-- 1. Hash('2024-01-01'), hash('2024-01-02'), ..., hash('2024-01-31')
--    → 31 hash lookups to different buckets
-- 2. Each lookup: Random page read (3 pages avg)
-- Total: 93 random pages
-- Time: 2 seconds (20× slower due to random I/O)
```

**When Hash Index IS Better:**

```sql
-- Exact match on high-cardinality column (UUID, email)
SELECT * FROM users WHERE id = '550e8400-e29b-41d4-a716-446655440000';

-- Hash: 1-2 page reads (constant)
-- B+Tree: 3-4 page reads (log N)

-- Difference: Marginal (3 vs 2 pages)
-- Not worth losing range query support
```

**Real-world decision:**

Most databases default to B+Tree for ALL indexes because:
- 80% of queries need ranges/sorting
- B+Tree "good enough" for exact matches (log N ≈ constant for practical sizes)
- Storage engines optimized for B+Tree (decades of optimization)
- Hash collisions can degrade O(1) to O(N) in worst case

**Exceptions where hash is used:**

1. **In-memory databases:** Redis, memcached (no disk I/O, O(1) matters more)
2. **PostgreSQL hash indexes:** Only for exact match (rare use case)
3. **Hash partitioning:** Distribute data across servers (different purpose)

**Interview follow-up:**

"In my experience, I've only created a hash index once: an in-memory lookup table for session IDs (UUIDs), where we only ever did exact match lookups, never ranges. For 99% of cases, B+Tree is the right choice because most queries filter on ranges or need sorted output at some point."
```

---

## 11. Summary

### B-Tree Key Principles

1. **Balanced Tree Structure**
   - All leaf nodes at same depth → Predictable O(log N) performance
   - Self-balancing on insert/delete → No manual maintenance

2. **Sorted Order**
   - Keys sorted within nodes → Binary search within node
   - Leaf nodes linked → Efficient range scans (key advantage over hash)

3. **High Fanout**
   - Internal nodes store 100-300 keys → Tree depth stays shallow (3-4 levels for billions of rows)
   - More keys = fewer I/Os per lookup

4. **Range Query Optimization**
   - Seek to start key: O(log N)
   - Sequential scan to end key: O(K) where K = result size
   - Total: O(log N + K) vs O(N) for sequential scan

### Implementation Checklist

**Creating Indexes:**
- [ ] Index foreign key columns (always)
- [ ] Index WHERE clause columns (frequently filtered)
- [ ] Index JOIN columns (both sides)
- [ ] Index ORDER BY columns (if used with LIMIT)
- [ ] Use composite indexes for multi-column filters (selective column first)

**Optimizing Existing Indexes:**
- [ ] Monitor index usage (drop unused indexes)
- [ ] Check fragmentation monthly (rebuild if >30%)
- [ ] Set fill factor for write-heavy tables (70-80%)
- [ ] Use covering indexes for hot queries (INCLUDE columns)
- [ ] Consider partial indexes for skewed data (WHERE clauses)

**Avoiding Common Mistakes:**
- [ ] Don't index every column (write amplification)
- [ ] Don't index low-cardinality columns (seq scan is faster)
- [ ] Don't ignore index size (large indexes = slow, cache thrashing)
- [ ] Don't forget to rebuild after mass deletes (bloat removal)
- [ ] Don't use high fill factor for random inserts (causes splits)

### Performance Characteristics

| Operation | Without Index | With B-Tree Index | Speedup |
|-----------|---------------|-------------------|---------|
| Exact match (WHERE id = ?) | O(N) seq scan | O(log N) | 1000-10,000× |
| Range query (WHERE x > ?) | O(N) seq scan | O(log N + K) | 10-1000× |
| ORDER BY LIMIT | O(N log N) sort | O(log N + K) | 100-1000× |
| MIN/MAX | O(N) scan all | O(log N) leftmost/rightmost | 1000-10,000× |
| JOIN (indexed FK) | O(N × M) nested loop | O(N × log M) | 100-1000× |

### Decision Tree: When to Create Index

```
Column used in WHERE/JOIN/ORDER BY?
├─ No → Don't create index (waste of space)
└─ Yes → Continue...
     ↓
Selectivity < 10%?
├─ No (e.g., 90% of rows match) → Don't create index (seq scan faster)
└─ Yes → Continue...
     ↓
High write volume (>50% INSERT/UPDATE)?
├─ Yes → Consider:
│   - Partial index (if querying subset)
│   - Lower fill factor (70-80%)
│   - Batch inserts, rebuild periodically
└─ No → Safe to create index
     ↓
Multi-column filter?
├─ Yes → Create composite index (most selective column first)
└─ No → Create single-column index
     ↓
Frequently need multiple columns from index?
├─ Yes → Create covering index (INCLUDE additional columns)
└─ No → Basic index sufficient
     ↓
CREATE INDEX!
```

### Quick Wins

**1. Index foreign keys (30 min, 10-100× faster JOINs)**
```sql
SELECT tablename, conname 
FROM pg_constraint 
WHERE contype = 'f';  -- Find all foreign keys

-- For each FK, check if index exists, create if missing
```

**2. Cover hot queries (1 hour, 5-10× faster)**
```sql
-- Find queries doing heap fetches:
EXPLAIN (ANALYZE, BUFFERS) [your top 10 queries];
-- Look for "Heap Fetches: 10000+"

-- Add INCLUDE clause to index
```

**3. Rebuild fragmented indexes (weekly, 2-5× faster)**
```sql
-- Find fragmented indexes:
SELECT indexname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE idx_scan > 100;

-- REINDEX monthly
```

**Remember:** B-Tree indexes are the workhorses of database performance. One well-placed index can make your app 1000× faster. Choose wisely!

---

**Next Reading:**
- `02_Hash_Indexes.md` - When O(1) lookups matter (exact match only)
- `04_Covering_Indexes.md` - Eliminating heap access entirely
- `06_Composite_Indexes.md` - Multi-column index design patterns
