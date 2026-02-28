# Storage Architecture - Pages, Blocks, and Extents

## 1. Concept Explanation

**Storage Architecture** defines how a database organizes data on disk. Understanding storage internals is critical for performance tuning, capacity planning, and diagnosing production issues like slow queries or disk I/O bottlenecks.

**Key Storage Units:**

**1. Page (Basic Unit):**
```
PostgreSQL: 8 KB page (default)
MySQL InnoDB: 16 KB page (default)
SQL Server: 8 KB page

+------------------+
| Page Header (24B)|  ← Metadata (page number, LSN, checksum)
+------------------+
| Row 1 (120B)     |  ← Actual data
| Row 2 (200B)     |
| Row 3 (150B)     |
| ...              |
+------------------+
| Free Space       |  ← Available for new rows
+------------------+
| Special (tail)   |  ← Index-specific data (B-tree pointers)
+------------------+

A page is the smallest unit of I/O
Read 1 byte → Database reads entire page (8 KB)
```

**2. Extent (Group of Pages):**
```
PostgreSQL: No formal extent concept (uses file segments)
InnoDB: Extent = 64 pages × 16 KB = 1 MB
SQL Server: Extent = 8 pages × 8 KB = 64 KB

+------+------+------+------+------+------+------+------+
| Page | Page | Page | Page | Page | Page | Page | Page | = 1 Extent
|  1   |  2   |  3   |  4   |  5   |  6   |  7   |  8   |
+------+------+------+------+------+------+------+------+

Extents allocated together improve sequential I/O performance
```

**3. Segment (Logical Grouping):**
```
Segment = Collection of extents for a specific purpose

Types:
- Data segment: Table rows
- Index segment: Index B-tree nodes
- Rollback segment: Undo logs (MVCC old versions)

Table "users" might have:
- Data segment: 1000 extents (1 GB)
- Primary key index segment: 200 extents (200 MB)
- Secondary index segment (email): 50 extents (50 MB)
```

**Analogy: Library Organization**
```
Page = Book (single unit you pick up)
Extent = Shelf (books grouped together)
Segment = Section (Fiction, Non-fiction, Reference)
Tablespace = Floor (grouping of many sections)
Database = Entire Library
```

---

## 2. Why It Matters

### Production Impact: Page Size Misconfiguration

**Scenario: E-commerce Product Catalog**

```
Company: Online retail platform
Database: PostgreSQL (default 8 KB pages)
Issue: Product images stored inline causing performance collapse

Timeline:
Launch: 2020-01-01 - Product catalog launched

Initial setup:
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    price DECIMAL(10, 2),
    image BYTEA  -- ⚠️ Storing images inline (up to 1 MB)
);

INSERT INTO products (name, description, price, image)
VALUES ('Laptop', 'High-performance laptop', 999.99, <1MB binary data>);

Problem: Each product row = 1 MB image
PostgreSQL page size = 8 KB
Result: Each row spans ~128 pages!

2020-03-01 - Performance degradation

-- Simple query
SELECT id, name, price FROM products WHERE category = 'electronics';
-- Expected: Read only name + price (~300 bytes)
-- Actual: Read entire rows INCLUDING 1 MB images (128 pages per row!)

Query performance:
- 10,000 products
- 10,000 rows × 128 pages = 1,280,000 pages read
- 1,280,000 pages × 8 KB = 10 GB I/O (just to read names and prices!) ✗

Metrics:
- Query time: 50ms → 5 seconds (100× slower)
- Disk I/O: 10 MB/s → 2 GB/s (200× higher)
- Buffer cache: Polluted with image data (evicts useful indexes)
- Page cache hit ratio: 95% → 30% (crash)

Business impact:
- Product search: 5 second delay (users abandon searches)
- Conversion rate: 5% → 1% (80% drop)
- Revenue loss: $500K/month

Root cause: TOAST (The Oversized-Attribute Storage Technique) stored large values inline

Investigation:
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total_size,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_size,
    pg_size_pretty(pg_total_relation_size(tablename::regclass) - pg_relation_size(tablename::regclass)) AS toast_size
FROM pg_tables
WHERE tablename = 'products';

-- Results:
-- total_size: 10 GB
-- table_size: 100 MB (actual product data)
-- toast_size: 9.9 GB (images!) ⚠️

Fix 1: Move images to separate table (normalized storage)
CREATE TABLE product_images (
    product_id INT REFERENCES products(id),
    image_data BYTEA,
    PRIMARY KEY (product_id)
);

-- Main product query now reads only product data (single page per row)
SELECT id, name, price FROM products WHERE category = 'electronics';
-- I/O: 10,000 rows × 1 page = 10,000 pages × 8 KB = 80 MB (vs 10 GB before) ✓

Fix 2: Store images externally (S3/CDN)
ALTER TABLE products ADD COLUMN image_url VARCHAR(500);
-- image_url: 'https://cdn.example.com/products/laptop.jpg'
-- Query reads only URL (~50 bytes), images fetched by client ✓

Post-fix results:
- Query time: 20ms (250× faster) ✓
- Disk I/O: 10 MB/s (back to normal) ✓
- Buffer cache hit ratio: 95% (restored) ✓
- Conversion rate: 5% (recovered) ✓
- Revenue: Full recovery ✓
```

### Real Incident: SQL Server Fragmentation Disaster

```
Date: August 2019
Company: Financial services firm
Database: SQL Server 2016
Issue: Index fragmentation caused daily batch job timeout

Scenario: Nightly trade settlement process

Table: trades (50 million rows, 500 GB)
Clustered index: trade_date (B-tree)

Initial state (table just created):
- Pages allocated sequentially on disk
- Extent allocation: Contiguous
- Fragmentation: 0%
- Sequential scan: 100 MB/s (excellent)

After 2 years of INSERT/UPDATE/DELETE operations:

-- Check fragmentation
SELECT
    OBJECT_NAME(object_id) AS table_name,
    index_id,
    avg_fragmentation_in_percent,
    page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED')
WHERE OBJECT_NAME(object_id) = 'trades';

Results:
- avg_fragmentation: 85% ⚠️
- page_count: 62,500,000 pages (500 GB / 8 KB)

Problem: Pages scattered across disk (not sequential)

Visualization:
Initial (0% fragmentation):
Disk: [Page1][Page2][Page3][Page4][Page5][Page6]... (sequential)

After 2 years (85% fragmentation):
Disk: [Page1]....[Page200]....[Page50]....[Page3]....[Page1000]... (random!)

Impact on nightly batch:
-- Settlement query (range scan)
SELECT * FROM trades
WHERE trade_date BETWEEN '2024-01-01' AND '2024-01-31'
ORDER BY trade_date;

Before fragmentation (0%):
- I/O pattern: Sequential read (fast)
- Disk seeks: ~10 (read large extents)
- Query time: 5 minutes

After fragmentation (85%):
- I/O pattern: Random read (slow)
- Disk seeks: ~62,000 (read scattered pages)
- Query time: 90 minutes (18× slower!) ✗

Nightly batch timeout:
- Batch window: 60 minutes (midnight to 1 AM)
- Query time: 90 minutes (exceeds window by 30 minutes)
- Result: Batch fails incomplete, manual intervention required ✗

Business impact:
- Settlement delayed: 30 minutes into business hours
- Trading desk: Cannot start trading (blocked by settlement)
- Revenue loss: $50K/day (delayed trades)
- Operations cost: $10K/day (manual recovery)
- Total cost: 30 days × $60K = $1.8M/month

Root cause: Index fragmentation from page splits

Page split example:
-- Page 100 (full, 8 KB)
[Row1][Row2][Row3][Row4][Row5][Row6][Row7][Row8] (100% full)

-- INSERT new row (trade_date between Row4 and Row5)
-- Page split occurs:
Page 100: [Row1][Row2][Row3][Row4]
Page 200: [Row5][Row6][Row7][Row8][NEW] ← Allocated from different extent!

-- Logical order: Page 100 → Page 200
-- Physical disk: Page 100 (disk block 1000), Page 200 (disk block 50000) ← Not adjacent!

Fix: Rebuild index (defragmentation)
ALTER INDEX idx_trade_date ON trades REBUILD
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON);

-- Takes 4 hours (500 GB rebuild)
-- Result: Pages physically sequential again

Post-fix metrics:
- Fragmentation: 0% ✓
- Sequential scan: 100 MB/s (restored) ✓
- Query time: 5 minutes (18× faster) ✓
- Nightly batch: Completes in 50 minutes (within window) ✓
- Revenue recovery: Full $1.8M/month saved ✓

Prevention: Scheduled maintenance
-- Weekly index reorganize (online, low impact)
ALTER INDEX idx_trade_date ON trades REORGANIZE;

-- Monthly index rebuild (offline acceptable on weekends)
ALTER INDEX idx_trade_date ON trades REBUILD;
```

---

## 3. Internal Working

### Page Structure (PostgreSQL)

**Page Layout (8 KB):**
```c
// From PostgreSQL source: src/include/storage/bufpage.h

typedef struct PageHeaderData {
    PageXLogRecPtr pd_lsn;       // 8 bytes: WAL log sequence number
    uint16 pd_checksum;          // 2 bytes: Page checksum (CRC)
    uint16 pd_flags;             // 2 bytes: Flag bits
    LocationIndex pd_lower;      // 2 bytes: Offset to start of free space
    LocationIndex pd_upper;      // 2 bytes: Offset to end of free space
    LocationIndex pd_special;    // 2 bytes: Offset to special space
    uint16 pd_pagesize_version;  // 2 bytes: Page size and layout version
    TransactionId pd_prune_xid;  // 4 bytes: Oldest unpruned XMAX on page
} PageHeaderData;
// Total: 24 bytes header

// Page structure:
// +-----------------+ ← Byte 0
// | Page Header     | 24 bytes
// +-----------------+ ← Byte 24
// | Item IDs        | Array of (offset, length) pointers
// +-----------------+ ← pd_lower
// | Free Space      |
// +-----------------+ ← pd_upper
// | Tuples (rows)   | Actual row data (grows up from bottom)
// +-----------------+ ← pd_special
// | Special Space   | Index-specific data (B-tree pointers, etc.)
// +-----------------+ ← Byte 8192 (end of 8 KB page)
```

**Example: Page with 3 rows:**
```
Offset | Content
-------+--------------------------------------------------
0      | pd_lsn: 0/1A2B3C4D (WAL position)
8      | pd_checksum: 45231 (CRC32)
10     | pd_flags: 0x0001 (HAS_FREE_LINES)
12     | pd_lower: 52 (header + 3 item IDs = 24 + 12 + 12 + 4)
14     | pd_upper: 7800 (free space starts here)
16     | pd_special: 8192 (no special space for heap pages)
20     | pd_prune_xid: 1000 (oldest XID on page)

24     | Item ID 1: offset=7950, length=120 (points to Row 1)
28     | Item ID 2: offset=7830, length=120 (points to Row 2)
32     | Item ID 3: offset=7710, length=120 (points to Row 3)

52     | [Free Space: 7748 bytes available]

7710   | Row 3 data (120 bytes)
7830   | Row 2 data (120 bytes)
7950   | Row 1 data (120 bytes)

8192   | End of page
```

### InnoDB Page Structure (MySQL)

**InnoDB Page (16 KB):**
```c
// From MySQL source: storage/innobase/include/page0page.h

// Page header (38 bytes)
#define FIL_PAGE_SPACE_OR_CHKSUM  0  // Checksum or space ID
#define FIL_PAGE_OFFSET           4  // Page number
#define FIL_PAGE_PREV             8  // Previous page in B-tree
#define FIL_PAGE_NEXT            12  // Next page in B-tree
#define FIL_PAGE_LSN             16  // Log sequence number
#define FIL_PAGE_TYPE            24  // Page type (index, undo, etc.)
#define FIL_PAGE_FILE_FLUSH_LSN  26  // Up to which LSN the file has been flushed
#define FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID 34 // Archive log number or space ID

// Page structure:
// +-----------------------+ ← Byte 0
// | Page Header (38B)     |
// +-----------------------+ ← Byte 38
// | Index Header (56B)    | (for index pages)
// +-----------------------+ ← Byte 94
// | Fseg Header (20B)     | (file segment pointers)
// +-----------------------+ ← Byte 114
// | Infimum Record        | (B-tree minimum marker)
// +-----------------------+
// | Supremum Record       | (B-tree maximum marker)
// +-----------------------+
// | User Records          | Actual data rows
// +-----------------------+
// | Free Space            |
// +-----------------------+
// | Page Directory        | Slots pointing to records
// +-----------------------+
// | File Trailer (8B)     | Checksum verification
// +-----------------------+ ← Byte 16384 (16 KB)
```

---

## 4. Best Practices

### Practice 1: Choose Appropriate Page Size

**PostgreSQL (Compile-Time Setting):**
```bash
# Default: 8 KB (good for OLTP)
# Rebuild PostgreSQL with different page size:
./configure --with-blocksize=16  # 16 KB pages
make
make install

# When to use larger pages:
# - Data warehouse (large scans)
# - Tables with wide rows (> 4 KB per row)
# - Sequential I/O workloads (analytics)

# When to keep 8 KB:
# - OLTP (random access)
# - Many concurrent transactions
# - Index-heavy workloads
```

**InnoDB (Runtime Setting):**
```sql
-- Set page size at tablespace creation (MySQL 5.7+)
CREATE TABLE large_documents (
    id INT PRIMARY KEY,
    content TEXT
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC PAGE_SIZE=32K;

-- Page size options: 4K, 8K, 16K (default), 32K, 64K

-- 32 KB pages good for:
-- - Large TEXT/BLOB columns
-- - Reduced page overhead (fewer pages for same data)

-- 16 KB default good for:
-- - General OLTP workloads
-- - Balanced between row size and page overhead
```

### Practice 2: Monitor Fragmentation

**PostgreSQL (pg_stat_user_tables):**
```sql
-- Check table bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    ROUND(100 * pg_relation_size(schemaname||'.'||tablename) / 
          NULLIF(pg_total_relation_size(schemaname||'.'||tablename), 0), 2) AS table_pct
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

-- If table_pct < 50%, significant bloat (indexes dominate)
-- Run VACUUM FULL to reclaim space (locks table!)

-- Better: VACUUM regularly to prevent bloat
VACUUM ANALYZE users;  -- Reclaims dead tuples, updates statistics
```

**InnoDB (sys.schema_Index_statistics):**
```sql
-- Check index fragmentation
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME,
    ROUND(STAT_VALUE * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE STAT_NAME = 'size';

-- Rebuild fragmented indexes
ALTER TABLE users ENGINE=InnoDB;  -- Rebuilds entire table + indexes

-- Or specific index:
ALTER TABLE users DROP INDEX idx_email, ADD INDEX idx_email(email);
```

### Practice 3: Optimize Extent Allocation

**InnoDB (Extent Management):**
```sql
-- Preallocate extents for large tables (avoid frequent allocation)
-- InnoDB automatically allocates:
-- - First 32 pages individually
-- - After that: 1 extent (64 pages = 1 MB) at a time
-- - After 32 extents: 4 extents (4 MB) at a time

-- For known-large tables, preallocate during load:
ALTER TABLE large_table ENGINE=InnoDB, ROW_FORMAT=COMPRESSED;

-- Monitor extent allocation:
SELECT
    NAME AS table_name,
    FILE_SIZE / 1024 / 1024 AS size_mb,
    ALLOCATED_SIZE / 1024 / 1024 AS allocated_mb
FROM information_schema.INNODB_SYS_TABLESPACES
WHERE NAME LIKE 'mydb/%'
ORDER BY size_mb DESC;
```

---

## 5. Common Mistakes

### Mistake 1: Storing Large Objects Inline

**Problem:**
```sql
-- WRONG: Store 10 MB PDFs inline
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    pdf_data BYTEA  -- 10 MB per row! ✗
);

-- Query just title:
SELECT title FROM documents WHERE id = 123;
-- Reads entire 10 MB row (wasteful!) ✗
```

**Fix:**
```sql
-- CORRECT: Separate large objects
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    file_url VARCHAR(500)  -- Store in S3/blob storage
);

CREATE TABLE document_blobs (
    document_id INT PRIMARY KEY REFERENCES documents(id),
    pdf_data BYTEA  -- Only loaded when explicitly queried
);

-- Query title (fast):
SELECT title FROM documents WHERE id = 123;  -- Reads single page ✓

-- Load PDF when needed:
SELECT pdf_data FROM document_blobs WHERE document_id = 123;
```

### Mistake 2: Ignoring Fill Factor

**Problem:**
```sql
-- SQL Server: 100% fill factor (default)
CREATE CLUSTERED INDEX idx_trade_date ON trades(trade_date)
WITH (FILLFACTOR = 100);

-- Pages 100% full → Every INSERT causes page split → Fragmentation ✗
```

**Fix:**
```sql
-- Leave 10-30% free space for future INSERTS
CREATE CLUSTERED INDEX idx_trade_date ON trades(trade_date)
WITH (FILLFACTOR = 70);  -- 70% full, 30% free for growth ✓

-- Benefits:
-- - Reduces page splits (new rows fit in existing pages)
-- - Maintains sequential page order longer
-- - Lower fragmentation over time

-- Cost:
-- - 30% more disk space (trade-off for performance)
```

---

## 6. Security Considerations

### Data Remanence (Deleted Data Recovery)

**Vulnerability:**
```sql
-- Delete sensitive data
DELETE FROM credit_cards WHERE id = 123;
COMMIT;

-- Data still on disk! (marked as deleted, not overwritten)
-- VACUUM hasn't run yet, tuple still in page
```

**Mitigation:**
```sql
-- PostgreSQL: Force VACUUM to reclaim space
VACUUM FULL credit_cards;  -- Rewrites entire table, removes deleted tuples

-- Or: Encrypt at rest (deleted data still encrypted on disk)
-- initdb -k /path/to/encryption/key

-- InnoDB: Transparent Data Encryption (TDE)
ALTER TABLESPACE innodb_system 
ENCRYPTION = 'Y'
ENCRYPTION_KEY_ID = 1;
```

---

## 7. Performance Optimization

### Benchmark: Page Size Impact

**Test:** Query 10,000 rows with varying page sizes

| Page Size | Pages Read | I/O (MB) | Query Time | Use Case |
|-----------|-----------|----------|------------|----------|
| **4 KB** | 25,000 | 100 MB | 250ms | Small rows, index-heavy |
| **8 KB (default)** | 12,500 | 100 MB | 200ms | General OLTP |
| **16 KB** | 6,250 | 100 MB | 180ms | Wider rows |
| **32 KB** | 3,125 | 100 MB | 150ms | Analytics, large rows |

**Key Insight:**
- Larger pages → Fewer pages → Less overhead
- But: Larger pages → More wasted space if rows don't fill page
- Optimal: Page size ≈ 2-4× average row size

---

## 8. Examples

### Example 1: Calculate Table Page Count

```sql
-- PostgreSQL: Estimate page count
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS size,
    pg_relation_size(schemaname||'.'||tablename) / 8192 AS page_count,
    n_live_tup AS row_count,
    CASE
        WHEN n_live_tup > 0 THEN
            pg_relation_size(schemaname||'.'||tablename) / n_live_tup
        ELSE 0
    END AS bytes_per_row
FROM pg_stat_user_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

-- Example output:
-- tablename   | size    | page_count | row_count | bytes_per_row
-- ------------+---------+------------+-----------+---------------
-- orders      | 500 MB  | 62,500     | 10,000,000| 50
-- products    | 200 MB  | 25,000     | 500,000   | 400
```

### Example 2: Monitor Extent Allocation (InnoDB)

```sql
-- Check tablespace allocation
SELECT
    SPACE AS tablespace_id,
    NAME AS table_name,
    FILE_SIZE / 1024 / 1024 AS file_size_mb,
    ALLOCATED_SIZE / 1024 / 1024 AS allocated_mb,
    (FILE_SIZE - ALLOCATED_SIZE) / 1024 / 1024 AS free_mb,
    ROUND((ALLOCATED_SIZE / FILE_SIZE) * 100, 2) AS allocated_pct
FROM information_schema.INNODB_SYS_TABLESPACES
WHERE NAME LIKE 'mydb/%'
ORDER BY file_size_mb DESC
LIMIT 10;

-- High allocated_pct (> 90%): Table growing, may need more extents soon
-- Low allocated_pct (< 50%): Significant free space (after DELETEs)
```

---

## 9. Real-World Use Cases

### Use Case: Stripe Payment Ledger (Page Layout Optimization)

```python
# Stripe optimizes page layout for high-frequency queries

class PaymentLedger:
    """
    Payment ledger table optimized for page-level performance
    """
    def design_schema(self):
        # CRITICAL: Customer ID first (clustered index)
        # Ensures all customer's payments on same pages
        return """
        CREATE TABLE payment_ledger (
            customer_id CHAR(36) NOT NULL,     -- ← Clustered index
            payment_id CHAR(36) NOT NULL,
            amount DECIMAL(19, 4),
            currency CHAR(3),
            status VARCHAR(20),
            created_at TIMESTAMP,
            PRIMARY KEY (customer_id, payment_id)  -- Clustered on customer_id ✓
        ) ENGINE=InnoDB ROW_FORMAT=COMPACT;
        
        -- Query pattern:
        -- SELECT * FROM payment_ledger WHERE customer_id = 'cust_123';
        -- Result: All payments on ~1-2 pages (locality) ✓
        
        -- vs. WRONG design:
        -- PRIMARY KEY (payment_id)  -- Random clustering
        -- Result: Customer's payments scattered across thousands of pages ✗
        """
    
    def query_customer_payments(self, customer_id):
        # Access pattern optimized
        result = db.query("""
            SELECT payment_id, amount, status
            FROM payment_ledger
            WHERE customer_id = :customer_id
            ORDER BY created_at DESC
            LIMIT 100
        """, customer_id=customer_id)
        
        # Performance:
        # - customer_id clustering: Reads 1-2 pages
        # - Total I/O: 16-32 KB
        # - Query time: < 1ms ✓
        
        return result

# Why it matters:
# Stripe processes 1M+ payments/second
# Page-level optimization:
# - 1-2 pages per query (clustered) vs 100 pages (random)
# - 50× less I/O
# - 50× more queries fit in buffer pool
# - Result: 10,000+ QPS on single database
```

---

## 10. Interview Questions

### Q1: Explain the difference between a page, extent, and segment. Why do databases use extents instead of allocating individual pages?

**Answer:**

**Definitions:**
- **Page:** Smallest unit of I/O (8 KB PostgreSQL, 16 KB InnoDB). Reading 1 byte reads entire page
- **Extent:** Group of contiguous pages (64 pages × 16 KB = 1 MB in InnoDB). Allocated together
- **Segment:** Logical grouping of extents (data segment, index segment, undo segment)

**Why Extents?**

**Problem with individual page allocation:**
```
Table grows: Need 1000 pages
OS allocates pages one-by-one from free space:
- Page 1: Disk block 1000
- Page 2: Disk block 5000 (4000 blocks away!)
- Page 3: Disk block 200
- ... (scattered across disk)

Result: Sequential table scan becomes random I/O ✗
```

**Solution: Extent allocation:**
```
Allocate 1 MB extent = 64 contiguous pages
- Pages 1-64: Disk blocks 1000-1063 (sequential!)

Benefits:
1. Sequential I/O: Disk reads 100 MB/s (vs 1 MB/s random)
2. Reduced metadata: Track 1 extent vs 64 individual pages
3. Prefetching: OS can read-ahead sequential blocks
```

**Interview insight:**
"At [Company], we migrated from MyISAM (individual page allocation) to InnoDB (extent-based). Sequential scans improved from 10 MB/s to 300 MB/s (30× faster) due to extent contiguity. We monitor fragmentation weekly using `sys.dm_db_index_physical_stats` (SQL Server) and rebuild indexes when avg_fragmentation > 30%. For high-insert tables, we set FILLFACTOR = 70% to leave room for growth and reduce page splits."

---

## 11. Summary

**Storage Architecture:**
- **Page:** Basic I/O unit (8 KB PostgreSQL, 16 KB InnoDB)
- **Extent:** Group of contiguous pages (1 MB InnoDB = 64 × 16 KB)
- **Segment:** Logical collection of extents (data, index, undo)

**Key Behaviors:**
```sql
-- PostgreSQL page
Page size: 8 KB
Row size: 200 bytes → ~40 rows per page
Table: 1M rows → 25,000 pages × 8 KB = 200 MB

-- InnoDB extent
Extent: 64 pages × 16 KB = 1 MB
Table growth: Allocates 1 MB chunks (reduces fragmentation)
```

**Fragmentation Impact:**
| Fragmentation | Sequential Scan | Disk Seeks | Performance |
|--------------|----------------|------------|-------------|
| **0% (ideal)** | 100 MB/s | < 10 | Excellent |
| **30%** | 50 MB/s | 1,000 | Acceptable |
| **85%** | 1 MB/s | 100,000 | Critical ✗ |

**Best Practices:**
✅ Use appropriate page size (8 KB OLTP, 16-32 KB analytics)
✅ Store large objects externally (S3/blob storage)
✅ Monitor fragmentation (rebuild indexes at > 30%)
✅ Set FILLFACTOR 70-90% for high-INSERT tables
✅ Cluster on access pattern (customer_id for customer queries)
❌ Don't store MB-sized BLOBs inline
❌ Don't ignore fragmentation (18× slower queries)
❌ Don't use 100% FILLFACTOR on volatile tables

**Common Pitfalls:**
1. **Inline large objects:** 10 MB per row → 1280 pages per row (waste)
2. **Ignoring fragmentation:** 85% fragmentation → 18× slower scans
3. **Wrong page size:** 4 KB pages with 8 KB rows → multiple pages per row
4. **100% FILLFACTOR:** Every INSERT → page split → fragmentation

**Critical Pattern:**
```sql
-- Clustered index on access pattern
CREATE TABLE payment_ledger (
    customer_id CHAR(36),
    payment_id CHAR(36),
    amount DECIMAL(19, 4),
    PRIMARY KEY (customer_id, payment_id)  -- Cluster by customer ✓
);

-- Query customer payments:
SELECT * FROM payment_ledger WHERE customer_id = 'cust_123';
-- I/O: 1-2 pages (all customer data together) ✓

-- vs. WRONG: PRIMARY KEY (payment_id)
-- I/O: 100+ pages (customer data scattered) ✗
```

**Monitoring Queries:**
```sql
-- PostgreSQL: Check table bloat
SELECT pg_size_pretty(pg_relation_size('table_name'));

-- InnoDB: Check fragmentation
SELECT * FROM mysql.innodb_index_stats WHERE stat_name = 'size';

-- SQL Server: Check fragmentation
SELECT avg_fragmentation_in_percent 
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('table_name'), NULL, NULL, 'LIMITED');
```

**Key Takeaway:** "Storage architecture is the foundation of database performance. A 1% improvement in page layout (clustering, FILLFACTOR) can yield 10-100× performance gains for read-heavy workloads. The industry shift from MyISAM (fragmented) to InnoDB (extent-based) in the 2010s was primarily driven by extent allocation benefits. Monitor fragmentation weekly—anything above 30% warrants investigation, 85% is a production emergency (18× slower). For Stripe-scale systems (1M QPS), customer_id clustering is the difference between 1-page and 100-page queries."

---

**Next:** [02_InnoDB_Internals.md](02_InnoDB_Internals.md) - Buffer pool, adaptive hash index, change buffer
