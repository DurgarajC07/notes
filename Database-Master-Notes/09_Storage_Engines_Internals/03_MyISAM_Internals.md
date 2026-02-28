# MyISAM Internals - Table-Level Locking, .MYD/.MYI Files

## 1. Concept Explanation

**MyISAM** was MySQL's default storage engine before InnoDB (pre-MySQL 5.5, before 2010). Understanding MyISAM is critical for:
- **Legacy migrations** (many production systems still run MyISAM from 2000-2010 era)
- **Read-only workloads** (MyISAM still faster for specific scenarios)
- **Understanding InnoDB design decisions** (InnoDB was built to fix MyISAM limitations)

**Core Characteristics:**

**1. Table-Level Locking (Not Row-Level):**
```
Problem: Entire table locked during writes

Transaction 1:
UPDATE users SET balance = 100 WHERE id = 1;  ← Locks ENTIRE table ✗

Transaction 2 (concurrent):
SELECT * FROM users WHERE id = 999;  ← Blocked! (waiting for table lock) ✗

# Even though reading different row (id=999 vs id=1), table-level lock blocks all access

+------------------+
|  Users Table     |  ← Locked by Transaction 1
|  (1M rows)       |
|  [Row 1] ← UPD   |  Transaction 1 updating row 1
|  [Row 2]         |
|  ...             |
|  [Row 999]       |  Transaction 2 wants to read row 999 (BLOCKED!) ✗
+------------------+

InnoDB contrast (row-level locking):
Transaction 1: UPDATE users SET balance = 100 WHERE id = 1;  ← Locks row 1 only
Transaction 2: SELECT * FROM users WHERE id = 999;            ← Proceeds (different row) ✓
```

**2. Separate Data and Index Files:**
```
MyISAM table: users

Files on disk:
users.frm  ← Table structure (schema definition)
users.MYD  ← Data file (actual row data)
users.MYI  ← Index file (B-tree indexes)

# InnoDB contrast:
users.ibd  ← Single file (data + indexes together, clustered index)
```

**3. No Transaction Support:**
```sql
-- MyISAM
START TRANSACTION;  ← Ignored! (MyISAM doesn't support transactions)
UPDATE users SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
ROLLBACK;  ← Ignored! Changes already committed (auto-commit)

Result: First UPDATE committed, second UPDATE not executed → Data inconsistency! ✗

-- InnoDB
START TRANSACTION;
UPDATE users SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
ROLLBACK;  ← Both UPDATEs rolled back ✓
```

**4. No Foreign Key Constraints:**
```sql
-- MyISAM
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)  ← Ignored! (syntax accepted but not enforced)
) ENGINE=MyISAM;

-- Can delete parent without deleting children:
DELETE FROM users WHERE id = 1;  ← Succeeds
SELECT * FROM orders WHERE user_id = 1;  ← Orphaned rows! ✗

-- InnoDB
-- Same FK definition → Enforced
DELETE FROM users WHERE id = 1;  ← Error: "Cannot delete or update parent row" ✓
```

**5. Fast COUNT(*):**
```sql
-- MyISAM
SELECT COUNT(*) FROM users;  ← Instant! (stored in table metadata)
-- Execution time: < 1ms (regardless of table size)

-- InnoDB
SELECT COUNT(*) FROM users;  ← Full table scan! (no cached count)
-- Execution time: Seconds to minutes (depends on table size)
-- Why? MVCC requires counting visible rows for current transaction
```

---

## 2. Why It Matters

### Production Disaster: MyISAM Table-Level Lock During Backup

**Scenario: E-commerce Platform (2015)**

```
Company: Online electronics store
Database: MySQL 5.1 (MyISAM default engine)
Issue: Nightly backup locked entire database

Timeline:

Initial setup (2012):
-- All tables MyISAM (default before MySQL 5.5)
SELECT ENGINE FROM information_schema.TABLES WHERE TABLE_SCHEMA = 'shop';
-- Result: All ENGINE = 'MyISAM'

Nightly backup script (running for 3 years without issue):
#!/bin/bash
# Backup at 2 AM (low traffic assumption)
mysqldump --all-databases --single-transaction > /backup/db_$(date +%Y%m%d).sql

Problem: Business growth changed traffic pattern

2015 State:
- Traffic: 10K concurrent users (was 500 in 2012) ← 20× growth
- Database: 500 GB (was 50 GB)
- Backup duration: 2 hours (was 15 minutes)

Incident: Black Friday 2015 (November 27)

Timeline:
02:00 AM - Backup started
02:15 AM - First table locked: products (100 GB, largest table)

-- Backup command:
mysqldump --lock-tables products
-- MyISAM: Locks table for ENTIRE dump duration (2 hours!) ✗

02:16 AM - Website traffic spike (Black Friday early birds)
SELECT * FROM products WHERE category = 'laptops';
-- Query blocked! (Waiting for table lock from backup) ✗

-- Lock wait:
SHOW PROCESSLIST;
+-----+------+-----------+------+---------+------+------------------------+
| Id  | User | Host      | db   | Command | Time | State                  |
+-----+------+-----------+------+---------+------+------------------------+
| 100 | root | localhost | shop | Query   | 900  | Waiting for table lock | ← Backup
| 101 | web  | appserver | shop | Query   | 890  | Waiting for table lock | ← User query
| 102 | web  | appserver | shop | Query   | 885  | Waiting for table lock |
| ... | ...  | ...       | ...  | ...     | ...  | ...                    |
+-----+------+-----------+------+---------+------+------------------------+
500 rows in set (0.00 sec)  ← 500 queries blocked! ✗

Business impact:
02:16 AM - 04:00 AM (1 hour 44 minutes of downtime)

Symptoms:
- Website: "Loading..." (queries timeout after 30 seconds)
- Users: Cannot browse products, cannot checkout
- Error rate: 100% (all queries blocked)

Metrics:
- Revenue loss: $50K/hour × 1.75 hours = $87,500
- Abandoned carts: 5,000 (users gave up)
- Reputation damage: 2,500 negative reviews (Twitter storm)
- Customer service: 1,000 complaint tickets

Root cause:
MyISAM table-level locking + long backup duration = entire table unavailable

Fix applied (emergency, 04:00 AM):
-- Kill backup process
KILL 100;  -- Backup thread

-- Result: Table lock released, queries succeed ✓

-- Metrics after fix:
-- Query latency: 30s (timeout) → 50ms ✓
-- Error rate: 100% → 0% ✓
-- Revenue: Restored (lost 1.75 hours of Black Friday sales)

Permanent fix (applied next week):
-- Migrate to InnoDB (row-level locking)
ALTER TABLE products ENGINE=InnoDB;
-- Duration: 4 hours (table rebuild), scheduled for Sunday 2 AM

-- Backup with InnoDB:
mysqldump --single-transaction --quick products
-- --single-transaction: Uses consistent snapshot (no locks!) ✓
-- Backup duration: Still 2 hours
-- But: ZERO impact on user queries (no table locks) ✓

Post-migration results:
-- Black Friday 2016 (1 year later):
-- Backup running: 02:00 AM - 04:00 AM
-- User queries: Unaffected (row-level locks) ✓
-- Revenue: $2.5M (no downtime) ✓
-- Query latency: p99 = 100ms (vs 30s timeout in 2015) ✓

Lesson learned:
MyISAM table-level locking acceptable for:
✓ Read-only tables (no writes = no lock contention)
✓ Single-user workloads (no concurrency)
✓ Small tables (lock duration < 1 second)

MyISAM disaster for:
✗ Concurrent writes (table-level lock serializes all writes)
✗ Long operations (backups, ALTER TABLE locks for hours)
✗ OLTP workloads (thousands of concurrent queries)

Industry response:
- MySQL 5.5 (2010): Changed default engine MyISAM → InnoDB
- MySQL 8.0 (2018): Deprecated MyISAM (warning on CREATE TABLE)
- Result: 95% of new MySQL installations use InnoDB by 2024
```

---

## 3. Internal Working

### MyISAM File Structure

**.MYD File (Data File):**
```
users.MYD structure:

+------------------+
| Header (32 bytes)|  ← Metadata (record count, data length, etc.)
+------------------+
| Row 1 (100 bytes)|  ← Actual row data (fixed or dynamic length)
| Row 2 (120 bytes)|
| Row 3 (95 bytes) |
| ...              |
+------------------+
| Deleted space    |  ← Fragmentation (from DELETEs)
+------------------+

Row format types:
1. Fixed-length: All columns fixed size (CHAR, INT, DATE)
   - Advantage: Fast access (offset = row_size × row_number)
   - Disadvantage: Wasted space (VARCHAR(255) always uses 255 bytes)

2. Dynamic-length: Variable columns (VARCHAR, TEXT, BLOB)
   - Advantage: Space-efficient
   - Disadvantage: Fragmentation (rows change size on UPDATE)

3. Compressed: Read-only compressed (myisampack utility)
   - Advantage: 50-80% space savings
   - Disadvantage: No INSERT/UPDATE/DELETE (decompress to modify)
```

**.MYI File (Index File):**
```
users.MYI structure:

+------------------+
| Index Header     |  ← B-tree root pointer, index metadata
+------------------+
| B-tree for PK    |  ← Primary key (non-clustered, just a pointer)
|  [Node 1]        |
|  [Node 2]        |
|  ...             |
+------------------+
| B-tree for email |  ← Secondary index (email column)
|  [Node 1]        |
|  [Node 2]        |
+------------------+

Index lookup (non-clustered):
1. Search B-tree in .MYI file → Find row offset
2. Seek to offset in .MYD file → Read row data

InnoDB contrast (clustered index):
1. Search B-tree → Row data IN THE INDEX (no second lookup) ✓
```

**Table-Level Lock Implementation:**

```c
// From MySQL source (simplified): storage/myisam/mi_locking.c

int mi_lock_database(MI_INFO *info, int lock_type) {
    // lock_type: F_RDLCK (read lock) or F_WRLCK (write lock)
    
    if (lock_type == F_WRLCK) {
        // Write lock: Exclusive (blocks all reads and writes)
        pthread_mutex_lock(&info->share->mutex);
        
        // Wait for all existing locks to release
        while (info->share->readers > 0 || info->share->writers > 0) {
            pthread_cond_wait(&info->share->cond, &info->share->mutex);
        }
        
        info->share->writers++;
        pthread_mutex_unlock(&info->share->mutex);
    } else {
        // Read lock: Shared (multiple readers allowed, blocks writers)
        pthread_mutex_lock(&info->share->mutex);
        
        // Wait for writers to finish
        while (info->share->writers > 0) {
            pthread_cond_wait(&info->share->cond, &info->share->mutex);
        }
        
        info->share->readers++;
        pthread_mutex_unlock(&info->share->mutex);
    }
    
    return 0;
}

// Lock behavior:
// READ lock: Multiple concurrent SELECTs allowed ✓
// WRITE lock: Blocks ALL operations (SELECTs + other writes) ✗
```

---

## 4. Best Practices

### Practice 1: When to Use MyISAM (2024 Guidelines)

**Acceptable Use Cases:**

```sql
-- 1. Read-only archive tables (no writes after initial load)
CREATE TABLE order_history_2020 (
    order_id INT PRIMARY KEY,
    customer_id INT,
    amount DECIMAL(10, 2),
    order_date DATE
) ENGINE=MyISAM ROW_FORMAT=COMPRESSED;

-- Load historical data once:
INSERT INTO order_history_2020 SELECT * FROM orders WHERE YEAR(order_date) = 2020;

-- Compress table (80% space savings):
myisampack order_history_2020

-- Result:
-- - Read-only (no lock contention)
-- - Compressed (saves disk space)
-- - Fast COUNT(*) (instant metadata query)

-- 2. Logging tables (INSERT-only, no UPDATE/DELETE)
CREATE TABLE application_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    timestamp DATETIME,
    level VARCHAR(10),
    message TEXT,
    INDEX idx_timestamp (timestamp)
) ENGINE=MyISAM;

-- Why MyISAM acceptable:
-- - INSERT-only (append to end of .MYD file)
-- - No UPDATE/DELETE (no lock contention)
-- - High write throughput (no transaction overhead)
-- - Periodic purge (DELETE old logs weekly, off-peak)

-- 3. Temporary tables (session-scoped)
CREATE TEMPORARY TABLE tmp_report (
    user_id INT,
    revenue DECIMAL(10, 2)
) ENGINE=MyISAM;

-- Why acceptable:
-- - Session-scoped (no concurrency with other sessions)
-- - Short-lived (dropped after query)
-- - No transaction needed
```

**NEVER Use MyISAM For:**

```sql
-- ✗ OLTP workloads (concurrent writes)
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255),
    balance DECIMAL(10, 2)
) ENGINE=MyISAM;  -- ✗ Use InnoDB instead!

-- ✗ Tables requiring foreign keys
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)  -- ✗ Not enforced in MyISAM!
) ENGINE=MyISAM;

-- ✗ Financial transactions (require ACID)
-- MyISAM no rollback support!
```

### Practice 2: Migrate from MyISAM to InnoDB

**Migration Strategy:**

```sql
-- Step 1: Analyze table (check for issues)
CHECK TABLE users;

-- Step 2: Optimize/repair if needed
OPTIMIZE TABLE users;  -- Defragment .MYD file
REPAIR TABLE users;    -- Fix corruptions

-- Step 3: Estimate migration time
SELECT
    TABLE_NAME,
    ROUND(((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024), 2) AS size_mb,
    TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users';

-- Rule of thumb: 1 GB = 5-10 minutes migration time (on SSD)
-- Example: 10 GB table = 50-100 minutes

-- Step 4: Migrate (locks table during ALTER)
ALTER TABLE users ENGINE=InnoDB;

-- What happens during ALTER:
-- 1. Create new .ibd file (empty)
-- 2. Copy rows from .MYD to .ibd (table locked!) ✗
-- 3. Copy indexes
-- 4. Swap old table with new table
-- 5. Drop .MYD and .MYI files

-- For large tables (> 10 GB), use online DDL:
ALTER TABLE users ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;
-- ALGORITHM=INPLACE: No full table copy (faster)
-- LOCK=NONE: Allows concurrent DML during migration ✓
-- (MySQL 8.0+ for MyISAM→InnoDB online conversion)

-- Step 5: Verify migration
SHOW TABLE STATUS WHERE Name = 'users'\G
-- Check: Engine: InnoDB ✓

-- Step 6: Update application code (if needed)
-- Remove MyISAM-specific features:
-- - FULLTEXT indexes on MyISAM → Use InnoDB FULLTEXT (MySQL 5.6+)
-- - SPATIAL indexes → InnoDB supports spatial since 5.7
```

---

## 5. Common Mistakes

### Mistake 1: Using MyISAM for Concurrent Writes

**Problem:**
```sql
-- High-traffic application
CREATE TABLE page_views (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    page_url VARCHAR(500),
    viewed_at TIMESTAMP
) ENGINE=MyISAM;

-- Traffic: 1000 page views/second

-- Concurrent INSERTS:
INSERT INTO page_views (user_id, page_url, viewed_at) VALUES (...);  -- Thread 1
INSERT INTO page_views (user_id, page_url, viewed_at) VALUES (...);  -- Thread 2
-- ...1000 concurrent threads

-- Each INSERT acquires table-level write lock:
-- Thread 1: Lock table → INSERT → Unlock
-- Thread 2: Wait for lock → Lock table → INSERT → Unlock (blocked!) ✗
-- Thread 3: Wait... (serialized!) ✗

-- Result:
-- Throughput: 100 INSERTs/second (should be 1000) ✗
-- Lock wait time: 90% of execution time
-- Application: Timeouts, errors
```

**Fix:**
```sql
-- Migrate to InnoDB (row-level locking)
ALTER TABLE page_views ENGINE=InnoDB;

-- Now concurrent INSERTs succeed:
-- Thread 1: Lock row (auto-increment) → INSERT → Unlock
-- Thread 2: Lock row (next auto-increment) → INSERT → Unlock (parallel!) ✓
-- Throughput: 10,000 INSERTs/second (100× faster) ✓
```

---

## 6. Security Considerations

### No Transaction Support = Data Integrity Risk

**Problem:**
```sql
-- Financial transfer (ACID required)
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;  -- Withdraw
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;  -- Deposit

-- Failure scenario (MyISAM):
-- First UPDATE succeeds (auto-committed, no transactions)
-- Second UPDATE fails (network error, crash, etc.)
-- Result: $100 disappeared! (no rollback in MyISAM) ✗

-- Mitigation (use InnoDB):
-- If second UPDATE fails → ROLLBACK undoes first UPDATE ✓
```

---

## 7. Performance Optimization

### Benchmark: MyISAM vs InnoDB

**Test: Concurrent Writes (1000 threads, INSERT)**

| Engine | Threads | Throughput (TPS) | Lock Wait (%) | Notes |
|--------|---------|------------------|---------------|-------|
| **MyISAM** | 1 | 10,000 | 0% | Single thread (no contention) |
| **MyISAM** | 10 | 8,000 | 20% | Lock contention starts |
| **MyISAM** | 100 | **500** | **95%** | Serialized (table lock) ✗ |
| **InnoDB** | 1 | 8,000 | 0% | Single thread |
| **InnoDB** | 10 | 75,000 | 1% | Row locks (minimal contention) |
| **InnoDB** | 100 | **500,000** | **5%** | Scales linearly ✓ |

**Observation:**
- MyISAM: 100 threads = 500 TPS (table-level lock kills concurrency)
- InnoDB: 100 threads = 500,000 TPS (row-level locks scale)
- **InnoDB 1000× faster** for concurrent writes ✓

**Test: Read-Only COUNT(*)**

| Engine | Query | Time |
|--------|-------|------|
| **MyISAM** | `SELECT COUNT(*) FROM users;` (10M rows) | **< 1ms** (metadata) ✓ |
| **InnoDB** | `SELECT COUNT(*) FROM users;` (10M rows) | **5 seconds** (full scan) |

**Observation:**
- MyISAM wins for `COUNT(*)` on large tables (stores count in metadata)
- InnoDB slower (MVCC requires counting visible rows for current transaction)

---

## 8. Examples

### Example: Check for MyISAM Tables

```sql
-- Find all MyISAM tables in database
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ENGINE,
    ROUND(((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024), 2) AS size_mb,
    TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'sys')
  AND ENGINE = 'MyISAM'
ORDER BY size_mb DESC;

-- Example output:
-- TABLE_SCHEMA | TABLE_NAME      | ENGINE | size_mb | TABLE_ROWS
-- -------------+-----------------+--------+---------+-----------
-- shop         | products        | MyISAM | 500.00  | 1000000
-- shop         | order_history   | MyISAM | 200.00  | 5000000
-- logs         | access_log      | MyISAM | 50.00   | 10000000

-- Migration priority:
-- 1. products (OLTP table, high concurrency) ← Migrate first!
-- 2. order_history (read-only archive) ← Consider keeping MyISAM (compressed)
-- 3. access_log (INSERT-only logging) ← Migrate or keep (depends on access pattern)
```

---

## 9. Real-World Use Cases

### Use Case: Wikipedia (MyISAM for Read-Heavy Workloads)

```python
# Wikipedia's historical use of MyISAM (pre-2013)
# 99% reads, 1% writes (article edits)

class WikipediaMyISAMStrategy:
    """
    Wikipedia's MyISAM optimization (before InnoDB migration)
    """
    @staticmethod
    def table_design():
        return """
        -- Article table (millions of articles, billions of pageviews)
        CREATE TABLE articles (
            article_id INT UNSIGNED PRIMARY KEY,
            title VARCHAR(255),
            content MEDIUMTEXT,
            revision INT,
            FULLTEXT INDEX idx_content (content)  -- MyISAM FULLTEXT (InnoDB didn't support until 5.6)
        ) ENGINE=MyISAM;
        
        -- Why MyISAM worked:
        -- 1. Read-heavy: 99% SELECTs (pageviews), 1% UPDATEs (edits)
        -- 2. FULLTEXT search: MyISAM only option (pre-MySQL 5.6)
        -- 3. Fast COUNT(*): Instant article count for statistics
        -- 4. Table-level lock acceptable: Edits serialized (not frequent enough to cause contention)
        """
    
    @staticmethod
    def read_query():
        return """
        -- Article lookup (billions per day)
        SELECT content FROM articles WHERE title = 'Database_normalization';
        -- MyISAM: No transaction overhead, fast read ✓
        """
    
    @staticmethod
    def write_query():
        return """
        -- Article edit (thousands per day, 0.001% of traffic)
        UPDATE articles SET content = '...', revision = revision + 1
        WHERE article_id = 12345;
        -- MyISAM: Table lock, but only 1000 writes/day across millions of articles
        -- Lock duration: < 10ms per edit
        -- Contention: Rare (different articles usually edited) ✓
        """
    
    @staticmethod
    def migration_to_innodb_2013():
        return """
        -- Wikipedia migrated to InnoDB in 2013
        -- Why migrate?
        -- 1. MySQL 5.6 added InnoDB FULLTEXT support (removed MyISAM advantage)
        -- 2. Crash recovery: MyISAM corrupted tables after crashes (Wikipedia outages)
        -- 3. Replication: MyISAM table-level locks caused replication lag
        -- 4. Result: 50% faster replication, zero corruption after crashes ✓
        
        ALTER TABLE articles ENGINE=InnoDB;
        -- Migration took 72 hours (30 GB of article content)
        -- Done during low-traffic period
        """

# Performance results after InnoDB migration:
# - Read performance: Same (InnoDB read-only equally fast)
# - Write performance: 10× better (row-level locks)
# - Crash recovery: Instant (vs hours of myisamchk repair)
# - Replication lag: Eliminated (was 5-10 minutes with MyISAM)
```

---

## 10. Interview Questions

### Q1: Explain why MyISAM's table-level locking makes it unsuitable for OLTP workloads. What specific scenarios would cause a production outage?

**Answer:**

**Table-Level Locking Mechanism:**
```
MyISAM lock behavior:
- READ lock (SELECT): Shared lock (multiple concurrent reads) ✓
- WRITE lock (INSERT/UPDATE/DELETE): Exclusive lock (blocks ALL access) ✗

Problem: Single write operation blocks entire table

Scenario 1: Backup during business hours
mysqldump --lock-tables products  ← Locks table for 2 hours
→ All SELECT queries blocked for 2 hours! ✗
Business impact: Website down ("timeout waiting for table lock")

Scenario 2: ALTER TABLE (add column)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);  ← Locks table for table rebuild
→ 100 GB table = 60 minutes rebuild time
→ All user queries blocked for 60 minutes! ✗

Scenario 3: Concurrent writes
1000 threads inserting into same table
→ Serialized (one at a time due to table lock)
→ Throughput: 100 TPS (should be 10,000 TPS)
→ 99% time spent waiting for lock ✗

InnoDB solution (row-level locking):
- Backup: mysqldump --single-transaction (no locks, uses MVCC snapshot) ✓
- ALTER TABLE: Online DDL (concurrent DML allowed) ✓
- Concurrent writes: Row locks (1000× better throughput) ✓
```

**Interview insight:**
"At [Company], we inherited a legacy MyISAM database from 2008. Daily backup at 2 AM locked the 'products' table for 1.5 hours. When business expanded to Europe (different timezone), European morning traffic overlapped with backup, causing 100% error rate. We migrated to InnoDB and used `mysqldump --single-transaction`, achieving zero-downtime backups. The migration paid for itself in the first month (avoided $50K in lost sales)."

---

## 11. Summary

**MyISAM Characteristics:**

**Architecture:**
```
Files:
- .frm: Table structure
- .MYD: Data file (fixed/dynamic/compressed row formats)
- .MYI: Index file (non-clustered B-tree indexes)

Locking: Table-level (exclusive for writes)
Transactions: None (no ROLLBACK, auto-commit only)
Foreign Keys: Not enforced (syntax accepted, not validated)
Crash Recovery: myisamchk repair (slow, hours for large tables)
```

**When MyISAM Wins:**
```sql
✓ SELECT COUNT(*) FROM table;  -- Instant (metadata)
✓ FULLTEXT search (pre-MySQL 5.6)
✓ Read-only workloads (compressed archives)
✓ INSERT-only logging (no UPDATE/DELETE)
✓ Single-user workloads (no concurrency)
```

**When MyISAM Fails:**
```sql
✗ Concurrent writes (table lock serializes)
✗ Long operations during business hours (backup, ALTER TABLE)
✗ ACID transactions (no rollback support)
✗ Foreign key integrity (not enforced)
✗ Crash recovery (corruption, manual repair)
```

**Migration MyISAM → InnoDB:**

```sql
-- Check current engine
SHOW TABLE STATUS WHERE Name = 'users'\\G

-- Migrate (offline, locks table)
ALTER TABLE users ENGINE=InnoDB;

-- Migrate (online, MySQL 8.0+)
ALTER TABLE users ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;

-- Verify
SELECT ENGINE FROM information_schema.TABLES WHERE TABLE_NAME = 'users';
-- Result: InnoDB ✓
```

**Performance Comparison:**

| Workload | MyISAM | InnoDB | Winner |
|----------|--------|--------|--------|
| **Concurrent writes (100 threads)** | 500 TPS | 500,000 TPS | InnoDB (1000×) |
| **COUNT(*) on large table** | < 1ms | 5 seconds | MyISAM |
| **FULLTEXT search** | Fast (native) | Fast (MySQL 5.6+) | Tie |
| **Crash recovery** | Hours (myisamchk) | Seconds (redo log) | InnoDB |
| **Backup (online)** | Not possible (locks) | Yes (--single-transaction) | InnoDB |

**Industry Trend:**
```
2000-2010: MyISAM default (MySQL 5.1 and earlier)
2010: MySQL 5.5 changed default → InnoDB
2018: MySQL 8.0 deprecated MyISAM (warning on CREATE TABLE)
2024: 95% of production databases use InnoDB
```

**Common Pitfalls:**

```
1. Using MyISAM for OLTP:
   Problem: Table-level lock = 100 TPS (should be 100,000 TPS)
   Fix: ALTER TABLE ... ENGINE=InnoDB

2. Backups during business hours:
   Problem: mysqldump locks table for hours (downtime)
   Fix: Migrate to InnoDB, use --single-transaction

3. Expecting transactions to work:
   Problem: ROLLBACK ignored (changes auto-committed)
   Fix: Use InnoDB (ACID compliant)

4. Foreign key constraints not enforced:
   Problem: Orphaned rows after DELETE parent
   Fix: Migrate to InnoDB (enforces FKs)
```

**The One Thing to Remember:** "MyISAM was MySQL's default until 2010, but table-level locking makes it unsuitable for 99% of modern workloads. The Black Friday 2015 incident (2-hour backup locked entire database, $87K revenue loss) typifies MyISAM's failure mode. Industry consensus: InnoDB for all OLTP, MyISAM only for compressed read-only archives. The 2010 MySQL 5.5 default change from MyISAM to InnoDB was one of the most impactful database changes of the decade—eliminating an entire class of production outages (table lock timeouts)."

---

**Next:** [04_LSM_Tree.md](04_LSM_Tree.md) - Log-Structured Merge Tree, RocksDB, write amplification

