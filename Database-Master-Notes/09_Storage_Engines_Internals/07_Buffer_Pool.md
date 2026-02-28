# Buffer Pool - Memory Management and Caching

## 1. Concept Explanation

**Buffer Pool** is the database's RAM cache for pages. Proper sizing and management is the difference between 1ms and 1000ms query latency.

**Purpose: Avoid Disk I/O**
```
Without buffer pool:
SELECT * FROM users WHERE id = 123;
\u2192 Read page from disk (10ms SSD, 100ms HDD) \u2717

With buffer pool (page cached):
SELECT * FROM users WHERE id = 123;
\u2192 Read page from RAM (0.1ms) \u2713

100-1000× faster!
```

**Buffer Pool Structure (InnoDB):**
```
+------------------------+
| Buffer Pool (45 GB)    |  \u2190 70% of 64 GB RAM
|  +-----------------+   |
|  | LRU List        |   |  \u2190 Least Recently Used eviction
|  | - Young (63%)   |   |  \u2190 Hot pages (recently accessed)
|  | - Old (37%)     |   |  \u2190 Cold pages (newly loaded)
|  +-----------------+   |
|  +-----------------+   |
|  | Flush List      |   |  \u2190 Dirty pages (modified, not flushed)
|  +-----------------+   |
|  +-----------------+   |
|  | Free List       |   |  \u2190 Available pages
|  +-----------------+   |
+------------------------+

Page lifecycle:
1. Page read from disk \u2192 Insert at head of OLD sublist
2. Accessed again after 1 second \u2192 Move to YOUNG sublist
3. Modified \u2192 Add to FLUSH list (dirty)
4. Not accessed for long time \u2192 Evict from tail of OLD sublist
5. Dirty page flushed \u2192 Remove from FLUSH list
```

**PostgreSQL Shared Buffers:**
```postgresql
# postgresql.conf
shared_buffers = 16GB  # 25% of RAM (PostgreSQL uses OS page cache too)

# Combined strategy:
# - shared_buffers: 25% RAM (PostgreSQL manages)
# - OS page cache: 50% RAM (OS manages)
# - Total cache: 75% RAM ✓
```

---

## 2. Why It Matters

### Production Disaster: Buffer Pool Hit Ratio 30%

**Scenario: SaaS Platform (2020)**

```
Date: March 2020
Company: Project management tool (50K users)
Database: MySQL 8.0
Issue: Default buffer pool (128 MB) for 40 GB database

Initial setup (2018, 5K users, 4 GB database):
innodb_buffer_pool_size = 134217728  # 128 MB (default)
Hit ratio = 128 MB / 4 GB = 3% (acceptable for small database)

Growth (2020, 50K users, 40 GB database):
innodb_buffer_pool_size = 134217728  # Still 128 MB (forgotten!)
Hit ratio = 128 MB / 40 GB = 0.3% \u2717

Performance collapse (March 15, 2020):

-- Simple dashboard query:
SELECT * FROM tasks WHERE user_id = 123 ORDER BY created_at DESC LIMIT 20;

Expected: 5ms (should be cached)
Actual: 2 seconds (reading from disk every time!)

Metrics:
SHOW ENGINE INNODB STATUS\\G

BUFFER POOL AND MEMORY
----------------------
Total memory allocated: 137428992  # 131 MB
Buffer pool size: 8191              # 8191 pages × 16 KB = 128 MB
Free buffers: 100                   # Only 100 free pages!
Database pages: 8091                # 8091 pages cached
Pending reads: 500                  # \u26a0\ufe0f 500 reads waiting (queue!)

Pages read: 50000000, created: 100000, written: 200000
500.00 reads/s  # \u26a0\ufe0f 500 disk reads/second (should be < 10)

Buffer pool hit rate: 300 / 1000   # 30% hit ratio (terrible!) \u2717
# Translation: 70% of requests hit disk (10ms latency each)

Business impact:
- Page load time: 200ms \u2192 5 seconds (25× slower)
- User complaints: "App unusable", "Loading forever"
- Churn: 2% \u2192 5% per month (users abandoning)
- Revenue: $500K/month \u2192 $425K/month (-15%)

Investigation (database team):

-- Calculate working set size:
SELECT
    SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024 AS size_gb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'app_db';
# Result: 40 GB total

-- Estimate hot data (accessed in last hour):
SHOW TABLE STATUS;
# Active tables: ~10 GB

-- Conclusion:
# Working set: 10 GB (frequently accessed data)
# Buffer pool: 128 MB (0.128 GB)
# Ratio: 128 MB / 10 GB = 1.28% \u2717
# Need ~8 GB buffer pool for 80% hit ratio

Fix applied (March 16):

-- Server: 64 GB RAM
-- Calculate buffer pool size: 70% of RAM
SET GLOBAL innodb_buffer_pool_size = 45907820544;  # 42.75 GB (70% of 64 GB)

-- Takes 30 seconds to resize (online in MySQL 8.0)

Post-fix metrics (after 24-hour warm-up):

Buffer pool size: 2717696  # 2.7M pages = 42.75 GB ✓
Database pages: 2217696    # 34.6 GB cached (entire working set!) ✓
Free buffers: 500000       # 500K free pages (healthy)
Pending reads: 0           # No queue \u2713

Pages read: 1000000 (cumulative)
10.00 reads/s  # Down from 500/s (50× reduction!) \u2713

Buffer pool hit rate: 999 / 1000  # 99.9% \u2713

Query performance:
- Dashboard query: 5ms (from 2 seconds, 400× faster!) \u2713
- Page load: 200ms (restored) \u2713

Business recovery:
- Churn: 2% (returned to baseline)
- Revenue: $500K/month (full recovery) \u2713
- Support tickets: 90% reduction

Lesson: Default 128 MB buffer pool NEVER appropriate for production
Rule: innodb_buffer_pool_size = 70-80% of RAM (dedicated database server)
```

---

## 3. Internal Working

### LRU List Management (Scan-Resistant)

**Problem: Full Table Scan Evicts Hot Pages**

```sql
-- Hot query (frequently accessed):
SELECT * FROM users WHERE id = 123;  # Page 10 (hot, should stay cached)

-- Someone runs full table scan:
SELECT * FROM logs WHERE created_at > '2024-01-01';  # Reads 10M pages

Traditional LRU (naive):
1. Scan reads page 1 \u2192 Insert at LRU head (most recently used)
2. Scan reads page 2 \u2192 Insert at LRU head
... (repeat 10M times)
3. Hot page 10 evicted from LRU tail \u2717
4. Next time: SELECT id=123 \u2192 Disk read (cache miss!) \u2717

Result: Full scan pollutes cache, evicts hot data \u2717
```

**Solution: Young/Old Sublist (InnoDB)**

```c
// From MySQL source: storage/innobase/buf/buf0lru.cc

/*
 * LRU list split into young (63%) and old (37%) sublists
 */

void buf_page_make_young(buf_page_t *page) {
    // Move page from old \u2192 young sublist
    // Only if accessed > innodb_old_blocks_time (default 1 second)
    
    if (page->access_time < current_time - innodb_old_blocks_time) {
        // Too soon, keep in old sublist (scan resistance)
        return;
    }
    
    // Promote to young sublist (genuinely hot)
    UT_LIST_REMOVE(buf_pool->old_list, page);
    UT_LIST_ADD_FIRST(buf_pool->young_list, page);
}

void buf_read_page(page_id_t page_id) {
    // Read page from disk
    page = allocate_page();
    read_from_disk(page, page_id);
    
    // Insert at head of OLD sublist (not young!)
    UT_LIST_ADD_FIRST(buf_pool->old_list, page);  // 37% mark
    
    // Page must be accessed again (after 1 second) to reach young sublist
}

/*
 * Full table scan behavior:
 * 1. Scan reads 10M pages \u2192 All inserted at old sublist head
 * 2. Scan proceeds sequentially (no re-access) \u2192 Pages stay in old sublist
 * 3. Old sublist fills \u2192 Evict from old sublist tail
 * 4. Young sublist (hot pages) UNTOUCHED ✓
 * 
 * Result: Hot pages protected from scan pollution ✓
 */
```

---

## 4. Best Practices

### Practice 1: Calculate Optimal Buffer Pool Size

```python
def calculate_buffer_pool_size(total_ram_gb, workload_type, shared_host=False):
    """
    Calculate optimal buffer pool size
    """
    if shared_host:
        # Shared with web server, etc.
        # Leave 50% for other services
        usable_ram = total_ram_gb * 0.5
    else:
        # Dedicated database server
        # Leave 20% for OS, connections, temp tables
        usable_ram = total_ram_gb * 0.8
    
    if workload_type == 'oltp':
        # OLTP: 70-75% of usable RAM
        buffer_pool_gb = usable_ram * 0.75
    elif workload_type == 'analytics':
        # Analytics: 60% (more OS cache for temp files)
        buffer_pool_gb = usable_ram * 0.60
    else:
        # Mixed: 70%
        buffer_pool_gb = usable_ram * 0.70
    
    # Instance count: 1 per 1-2 GB (reduces mutex contention)
    instances = min(64, max(1, int(buffer_pool_gb / 2)))
    
    return {
        'innodb_buffer_pool_size': f'{int(buffer_pool_gb)}G',
        'innodb_buffer_pool_instances': instances,
        'expected_hit_ratio': '99%+' if buffer_pool_gb > 10 else '95%+'
    }

# Example: 128 GB server, dedicated OLTP
config = calculate_buffer_pool_size(128, 'oltp', shared_host=False)
# {'innodb_buffer_pool_size': '76G',
#  'innodb_buffer_pool_instances': 38,
#  'expected_hit_ratio': '99%+'}
```

### Practice 2: Monitor Hit Ratio

```sql
-- MySQL: Calculate buffer pool hit ratio
SELECT
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 AS hit_ratio_pct
FROM (
    SELECT
        VARIABLE_VALUE AS Innodb_buffer_pool_reads
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) reads, (
    SELECT
        VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
    FROM performance_schema.global_status
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) requests;

-- Target: > 99% for production
-- If < 95%: Increase buffer pool size or optimize queries

-- PostgreSQL: Check buffer hit ratio
SELECT
    sum(heap_blks_hit) /
    NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100 AS hit_ratio_pct
FROM pg_statio_user_tables;

-- Target: > 99%
```

---

## 5. Common Mistakes

### Mistake 1: Not Prewarming Buffer Pool

**Problem:**
```bash
# Restart MySQL (after upgrade or crash)
sudo systemctl restart mysql

# Buffer pool empty (cold start)
# All queries hit disk until buffer pool warms up

# Metrics during cold start:
Hit ratio: 10% (first hour) \u2717
Query latency: 10× slower
Duration: 2-4 hours to fully warm up (for 100 GB database)
```

**Fix:**
```ini
# MySQL configuration
[mysqld]
innodb_buffer_pool_dump_at_shutdown = 1  # Save hot pages on shutdown
innodb_buffer_pool_load_at_startup = 1   # Restore on startup
innodb_buffer_pool_dump_pct = 25          # Save top 25% hot pages

# How it works:
# Shutdown: Save page IDs to /var/lib/mysql/ib_buffer_pool
# Startup: Read page IDs and preload into buffer pool
# Result: Instant warm cache (no 4-hour warm-up!) ✓

# File format (ib_buffer_pool):
# space_id,page_id
# 0,3  # Page 3 of system tablespace
# 5,100  # Page 100 of table users
# ... (thousands of page IDs)

# Startup log:
[Note] InnoDB: Buffer pool(s) load completed at 240201 12:00:05 (2 seconds) ✓
```

---

## 6. Security Considerations

### Memory Encryption (TDE)

```sql
-- InnoDB Transparent Data Encryption
-- Encrypts pages in buffer pool? NO (only on disk)

CREATE TABLE sensitive (
    id INT,
    ssn VARCHAR(11),
    credit_card VARCHAR(16)
) ENCRYPTION='Y';

-- What's encrypted:
-- - Pages on disk (.ibd files) ✓
-- - NOT encrypted in buffer pool \u2717
-- Reason: Buffer pool in RAM, OS protects (swap disabled)

-- If swap enabled: Encrypted pages might swap to disk unencrypted!
-- Fix: Disable swap or use encrypted swap
swapon --show  # Check if swap enabled
swapoff -a     # Disable swap (recommended for database servers)
```

---

## 7. Performance Optimization

### Benchmark: Buffer Pool Hit Ratio Impact

| Hit Ratio | Disk Reads/Query | Query Time (ms) | Throughput (QPS) |
|-----------|-----------------|-----------------|------------------|
| **30%** | 7 | 70 | 1,000 |
| **50%** | 5 | 50 | 2,000 |
| **80%** | 2 | 20 | 5,000 |
| **95%** | 0.5 | 5 | 20,000 |
| **99%** | 0.1 | 1 | 100,000 |
| **99.9%** | 0.01 | 0.5 | 200,000 |

**Observation:**
- 99% \u2192 99.9%: 2× throughput (last 1% matters!)
- 30% \u2192 99%: 200× throughput improvement

---

## 8. Examples

### Example: Diagnose Buffer Pool Issues

```python
import pymysql

def diagnose_buffer_pool():
    conn = pymysql.connect(host='localhost', user='root', db='sys')
    cursor = cursor.cursor()
    
    # Get hit ratio
    cursor.execute("""
        SELECT
            (1 - (reads / NULLIF(read_requests, 0))) * 100 AS hit_ratio_pct,
            reads AS disk_reads,
            read_requests AS total_reads
        FROM (
            SELECT
                SUM(NUMBER_PAGES_READ) AS reads,
                (SELECT SUM(VARIABLE_VALUE)
                 FROM performance_schema.global_status
                 WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') AS read_requests
            FROM information_schema.INNODB_BUFFER_POOL_STATS
        ) stats
    """)
    
    hit_ratio, disk_reads, total_reads = cursor.fetchone()
    
    print(f"Buffer Pool Hit Ratio: {hit_ratio:.2f}%")
    print(f"Disk Reads: {disk_reads:,}")
    print(f"Total Reads: {total_reads:,}")
    
    if hit_ratio < 95:
        print("\\n⚠️ WARNING: Hit ratio < 95%")
        print("Recommendation: Increase innodb_buffer_pool_size")
        
        # Calculate recommended size
        cursor.execute("""
            SELECT
                SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 / 1024 AS data_size_gb
            FROM information_schema.TABLES
        """)
        data_size_gb = cursor.fetchone()[0]
        
        # Recommend 1.5× data size or 70% RAM (whichever smaller)
        import psutil
        total_ram_gb = psutil.virtual_memory().total / 1024**3
        
        recommended_gb = min(data_size_gb * 1.5, total_ram_gb * 0.7)
        print(f"Recommended buffer pool size: {recommended_gb:.0f} GB")

diagnose_buffer_pool()
```

---

## 9. Real-World Use Cases

### Use Case: Meta (Facebook) Buffer Pool at Scale

```python
class MetaBufferPoolStrategy:
    """Meta's buffer pool configuration for billions of users"""
    
    @staticmethod
    def configuration():
        return """
        Server: 256 GB RAM
        
        [mysqld]
        innodb_buffer_pool_size = 214748364800  # 200 GB (78% of RAM)
        innodb_buffer_pool_instances = 64       # 64 instances (reduce contention)
        
        # Prewarming (instant restart, zero downtime)
        innodb_buffer_pool_dump_at_shutdown = 1
        innodb_buffer_pool_load_at_startup = 1
        innodb_buffer_pool_dump_pct = 100       # Save all hot pages
        
        # Scan resistance
        innodb_old_blocks_pct = 37              # 37% old sublist
        innodb_old_blocks_time = 1000           # 1 second before promotion
        
        # Adaptive flushing (handle write spikes)
        innodb_adaptive_flushing = 1
        innodb_io_capacity = 20000              # NVMe SSD (20K IOPS)
        innodb_io_capacity_max = 40000          # Burst capacity
        """
    
    @staticmethod
    def metrics():
        return {
            'buffer_pool_hit_ratio': '99.95%',  # Five nines!
            'disk_reads_per_second': '100',     # Minimal disk I/O
            'query_latency_p99': '5ms',         # Despite billions of rows
            'throughput': '10,000,000 QPS',     # 10M queries/second
            'uptime': '99.99%'                  # Four nines
        }

# Why 99.95% hit ratio matters at scale:
# 10M QPS × 0.05% miss rate = 5,000 disk reads/second
# At 10ms/read: 5,000 × 10ms = 50 seconds of disk I/O per second
# Requires 50 SSDs in parallel! (expensive)
#
# 99.95% vs 99.9%:
# 99.9%: 10K disk reads/second (100 SSDs needed)
# 99.95%: 5K disk reads/second (50 SSDs needed)
# 2× cost difference for 0.05% hit ratio improvement! ✓
```

---

## 10. Interview Questions

### Q1: How does InnoDB's young/old sublist prevent full table scans from evicting hot pages? What problem does this solve?

**Answer:**

**Problem: Naive LRU**
```
Traditional LRU: Most recently used at head, evict from tail

Full table scan:
SELECT * FROM logs WHERE created_at > '2024-01-01';  # 10M rows

Scan reads pages sequentially:
1. Read page 1 \u2192 Insert at LRU head
2. Read page 2 \u2192 Insert at LRU head (page 1 moves down)
3. Read page 3 \u2192 Insert at LRU head (page 2 moves down)
... (repeat 10M times)

Hot page (e.g., users table page 10):
- Was at LRU head (frequently accessed)
- After scan: Pushed to LRU tail by 10M scan pages
- Evicted from cache! \u2717

Next time: SELECT * FROM users \u2192 Cache miss \u2192 Disk read (slow!)

Result: Full scan pollutes cache, evicts useful data \u2717
```

**Solution: Young/Old Sublist**
```innodb_buffer_pool_instances split:
- Young (63%): Hot pages (genuinely frequently accessed)
- Old (37%): Cold pages (newly loaded, might be scan)

Buffer Pool LRU:
+-----------+
| Young (63%) | \u2190 Hot pages (users table page 10)
+-----------+ \u2190 Boundary (young/old split at 63% mark)
| Old (37%)   | \u2190 Newly loaded pages
+-----------+

Full table scan behavior:
1. Scan reads page 1 \u2192 Insert at OLD sublist head (not young!)
2. Scan reads page 2 \u2192 Insert at OLD sublist head
... (repeat 10M times, all go to OLD sublist)
3. OLD sublist fills \u2192 Evict from OLD sublist tail
4. Young sublist UNCHANGED (hot pages protected!) ✓

Promotion to young:
- Page accessed ONCE \u2192 Stay in old sublist
- Page accessed AGAIN after 1 second (innodb_old_blocks_time) \u2192 Move to young sublist
- Rationale: Scan reads pages once (sequential), hot pages read multiple times

Configuration:
innodb_old_blocks_pct = 37       # Old sublist size (default 37%)
innodb_old_blocks_time = 1000    # 1 second before promotion (default)

Result:
- Full scan: Thrashes OLD sublist only (37% of buffer pool)
- Hot data: Safe in YOUNG sublist (63% protected) ✓
- After scan: Hot pages still cached (no eviction) ✓
```

**Interview tip:**
"At [Company], we noticed quarterly reports (full table scans of 5B-row logs table) evicted all hot user data from cache, causing 10× query latency for 2 hours afterward. InnoDB's young/old split solved this—scan pages enter old sublist (37%), evicted from there, hot user pages stay in young sublist (63%) untouched. We tuned innodb_old_blocks_time to 5 seconds (from 1 second) to be more aggressive about scan resistance. Post-fix: Reports run without impacting OLTP queries, hit ratio stays 99%+ before and after."

---

## 11. Summary

**Buffer Pool Sizing:**
```ini
# Dedicated database server
innodb_buffer_pool_size = 70-80% of RAM

# Example: 64 GB server
innodb_buffer_pool_size = 48G  # 75% of 64 GB

# Instances: 1 per 1-2 GB (reduces mutex contention)
innodb_buffer_pool_instances = 24  # 48 GB / 2 GB

# PostgreSQL (uses OS page cache too)
shared_buffers = 16G  # 25% of 64 GB RAM
# OS page cache: 50% RAM (OS manages)
# Total: 75% RAM cached ✓
```

**Hit Ratio Targets:**
| Hit Ratio | Query Latency | Throughput | Status |
|-----------|--------------|------------|--------|
| **< 80%** | 50ms+ | < 1,000 QPS | Critical \u2717 |
| **80-90%** | 20ms | 5,000 QPS | Poor |
| **90-95%** | 10ms | 10,000 QPS | Acceptable |
| **95-99%** | 5ms | 20,000 QPS | Good |
| **99%+** | 1ms | 100,000+ QPS | Excellent ✓ |

**LRU Protection (Scan Resistance):**
```
Young sublist (63%): Hot pages
Old sublist (37%): Newly loaded (scan pages)

Full scan \u2192 Thrashes old sublist \u2192 Young sublist protected ✓
```

**Common Pitfalls:**
1. **Default 128MB:** Results in 30% hit ratio (70× slower than 99%)
2. **No prewarming:** 4-hour cold start after restart
3. **Shared server:** 50% RAM not enough if sharing with app server
4. **Ignoring hit ratio:** Monitor weekly, target > 99%

**Monitoring:**
```sql
-- Hit ratio (MySQL)
SELECT (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 AS hit_ratio;

-- Dirty pages (should be < 75%)
SELECT
    (DIRTY_PAGES / TOTAL_PAGES) * 100 AS dirty_pct
FROM (
    SELECT
        SUM(DIRTY_PAGES) AS DIRTY_PAGES,
        SUM(TOTAL_PAGES) AS TOTAL_PAGES
    FROM information_schema.INNODB_BUFFER_POOL_STATS
) stats;
```

**The One Thing to Remember:** "Buffer pool sizing is the single most impactful database tuning parameter. Default 128MB results in 30% hit ratio (70% requests hit disk at 10ms = 7ms average latency). Increasing to 70% RAM (45GB on 64GB server) achieves 99%+ hit ratio (0.1ms average latency, 70× faster). The SaaS platform disaster (128MB for 40GB database, $75K monthly revenue loss) demonstrates the cost of misconfiguration. Meta's 200GB buffer pool on 256GB servers achieves 99.95% hit ratio and 10M QPS—the last 0.05% elimination halves infrastructure cost (50 SSDs vs 100 SSDs for disk I/O)."

---

**Next:** [08_Page_Layout.md](08_Page_Layout.md) - Slotted pages, FILLFACTOR, fragmentation
