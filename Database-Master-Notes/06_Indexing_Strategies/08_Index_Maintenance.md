# Index Maintenance - Keeping Indexes Healthy and Performant

## 1. Concept Explanation

**Index maintenance** refers to

 the ongoing tasks required to keep database indexes performing optimally. Like a car needing regular oil changes, indexes degrade over time due to fragmentation, bloat, and stale statistics, requiring periodic maintenance to restore performance.

Think of index maintenance like maintaining a library:
- **Initial state:** Books alphabetically organized, easy to find
- **After 6 months:** Books misshelved, gaps on shelves, catalog outdated
- **Maintenance:** Reorganize books, fill gaps, update catalog
- **Result:** Fast lookups restored

### Why Indexes Degrade

```
Day 1: Freshly created B-Tree index
Page 1: [1,  2,  3,  4,  5]  100% full, sequential
Page 2: [6,  7,  8,  9, 10]  100% full, sequential
Page 3: [11, 12, 13, 14, 15] 100% full, sequential

After 6 months of INSERTs/UPDATEs/DELETEs:
Page 1: [1, __, 3, __, __]  40% full (fragmented)
Page 7: [2, 9]               20% full (out of order!)
Page 2: [6, 7, 8]            60% full
Page 5: [4, 5, 11]           60% full (mixed ranges)
Page 3: [12, __, 14, __, __] 40% full

Problems:
- Fragmentation: Data scattered across more pages
- Bloat: Pages half-empty (wasted space)
- Out-of-order pages: Range scans jump randomly
- Missing statistics: Optimizer makes wrong decisions
```

**Types of Index Degradation:**

1. **Page splits** → Fragmentation
2. **Deletes** → Bloat (empty space)
3. **Random inserts** → Logical order ≠ physical order
4. **Stale statistics** → Bad query plans

---

## 2. Why It Matters

### Production Impact: Before vs After Maintenance

**Scenario: E-commerce Orders Table**

```sql
-- Table: orders (50M rows), B-Tree index on created_at
-- After 1 year without maintenance:

-- Query: Recent orders
SELECT * FROM orders 
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY created_at DESC
LIMIT 100;
```

**Before Maintenance:**
```
Index Scan using idx_orders_created on orders
  → Index pages read: 1,500 pages (fragmented!)
  → Heap pages read: 800 pages (random access)
  → Buffers: shared hit=0 read=2300
  → Time: 2.5 seconds

Analysis:
- Index fragmentation: 75% (logical sequences scattered)
- Index bloat: 60% (pages half-empty after deletes)
- Actual index size: 800MB (should be 320MB)
- Cache efficiency: Poor (pages not sequential, cache misses)
```

**After Maintenance (REINDEX):**
```
Index Scan using idx_orders_created on orders
  → Index pages read: 150 pages (compact!)
  → Heap pages read: 100 pages (better locality)
  → Buffers: shared hit=200 read=50
  → Time: 0.15 seconds (16× FASTER!)

Analysis:
- Index fragmentation: 0% (freshly rebuilt)
- Index bloat: 0% (optimal fill factor)
- Actual index size: 320MB (60% reduction)
- Cache efficiency: Excellent (sequential pages)
```

### Cost of Not Maintaining Indexes

**Real Incident: Payment Processing Slowdown**

```
Timeline:
- Month 0: System deployed, indexes fresh
  - Query time: 50ms average
  - Throughput: 1000 transactions/sec

- Month 3: Performance degrading
  - Query time: 150ms average (3× slower)
  - Throughput: 600 transactions/sec
  - Users complaining about checkout delays

- Month 6: Crisis
  - Query time: 500ms average (10× slower)
  - Throughput: 200 transactions/sec
  - Black Friday approaching, system can't handle load

Root cause analysis:
- Index fragmentation: 85%
- Index bloat: 70% empty space
- Query plan choosing seq scan over index (optimizer thinks index too expensive)
- Database I/O saturated (reading 10× more pages than necessary)

Solution:
- Emergency maintenance window: REINDEX all tables
- Duration: 2 hours (50GB indexes rebuilt)
- Result: Query times back to 50ms, throughput 1000/sec
- Prevention: Scheduled monthly REINDEX
```

---

## 3. Internal Working

### Index Fragmentation Measurement

**Logical Fragmentation:**
```
Ideal (0% fragmentation):
Logical order:  [Page 1] → [Page 2] → [Page 3] → [Page 4]
Physical order: [Page 1]   [Page 2]   [Page 3]   [Page 4]
Sequential reads: ✅ Fast

Fragmented (80% fragmentation):
Logical order:  [Page 1] → [Page 5] → [Page 2] → [Page 9] → [Page 3]
Physical order: [Page 1]   [Page 2]   [Page 3]   [Page 4]   [Page 5]...
Sequential reads: ❌ Random I/O (slow!)

Measurement:
Fragmentation % = (Pages out of sequence / Total pages) × 100
```

**B-Tree Page Density (Bloat):**
```
Fresh index (100% density):
Page: [K1, K2, K3, K4, K5, K6, K7, K8]
Free space: 0%
Efficiency: Optimal

After 1000 deletes (40% density):
Page: [K1, __, __, K4, __, K6, __, K8]
Free space: 60%
Efficiency: Poor (2.5× more pages to scan)

Calculation:
Page density = (Used space / Total space) × 100
Bloat = 100% - Page density
```

### REINDEX Algorithm

```
PostgreSQL REINDEX:

FUNCTION reindex_table(table_name):
    # Step 1: Scan existing index, extract all entries
    entries = []
    FOR each page in old_index:
        FOR each entry in page:
            IF entry is not deleted:
                entries.append((key, row_id))
    
    # Step 2: Sort entries by key
    entries.sort(key=lambda x: x[0])
    
    # Step 3: Build new B-Tree from sorted entries
    new_index = build_btree(entries, fill_factor=90)
    
    # Step 4: Atomically swap old index with new
    BEGIN TRANSACTION:
        LOCK TABLE (ACCESS EXCLUSIVE)  # Blocks reads/writes
        DROP old_index
        RENAME new_index TO old_index_name
        COMMIT
    
    # Step 5: Update statistics
    ANALYZE table_name

Complexity:
- Scan: O(N) where N = index entries
- Sort: O(N log N)
- Build: O(N)
- Total: O(N log N)

Downtime:
- REINDEX: Full table lock (blocks all operations)
- REINDEX CONCURRENTLY: Allows reads, blocks writes only during swap
```

**REBUILD Online (SQL Server):**
```
SQL Server ONLINE REBUILD:

FUNCTION rebuild_index_online(index_name):
    # Step 1: Create shadow index (copy)
    shadow_index = create_index_copy(index_name)
    
    # Step 2: Apply ongoing changes to both indexes
    enable_dual_write_mode()  # New inserts/updates go to both
    
    # Step 3: Catch up shadow index
    WHILE delta_changes > 0:
        apply_changes_to_shadow(delta_changes)
        delta_changes = get_remaining_changes()
    
    # Step 4: Quick swap with brief exclusive lock
    BEGIN TRANSACTION:
        LOCK INDEX (EXCLUSIVE)  # Brief lock (milliseconds)
        swap_indexes(original, shadow)
        COMMIT
    
    # Step 5: Drop old index
    DROP shadow_index (now points to old index)

Downtime:
- Minimal: < 100ms exclusive lock during swap
- Reads: Unaffected throughout
- Writes: Brief block during swap only
```

---

## 4. Best Practices

### Practice 1: Regular Monitoring of Index Health

**PostgreSQL: Monitor Bloat and Fragmentation**
```sql
-- Check index size and usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pg_relation_size(indexrelid) AS size_bytes
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Detect bloated indexes (requires pgstattuple extension)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pgstatindex(indexrelid)::text AS index_stats;

-- Parse relevant stats:
SELECT
    indexname,
    avg_leaf_density,  -- Should be > 70%
    leaf_fragmentation -- Should be < 30%
FROM
(
    SELECT
        indexname,
        (pgstatindex(indexrelid)).* 
    FROM pg_stat_user_indexes
    WHERE schemaname = 'public'
) AS stats
WHERE avg_leaf_density < 70 OR leaf_fragmentation > 30;

-- Indexes needing maintenance ^^^
```

**SQL Server: Fragmentation Check**
```sql
-- Check fragmentation across all indexes
SELECT 
    OBJECT_NAME(ips.object_id) AS table_name,
    i.name AS index_name,
    ips.index_type_desc,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.avg_page_space_used_in_percent,
    ips.record_count
FROM sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'DETAILED'
) AS ips
JOIN sys.indexes AS i
    ON ips.object_id = i.object_id
    AND ips.index_id = i.index_id
WHERE 
    ips.avg_fragmentation_in_percent > 10  -- Threshold
    AND ips.page_count > 1000  -- Skip small indexes
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- Maintenance recommendations:
-- Fragmentation 10-30%: REORGANIZE (online, lightweight)
-- Fragmentation > 30%: REBUILD (more thorough, can be online)
```

### Practice 2: Scheduled Maintenance Windows

**❌ Bad: Reactive Maintenance (Wait Until Performance Degrades)**
```sql
-- No monitoring, no schedule
-- Performance slowly degrades over 6 months
-- Users complain → Emergency investigation
-- Find 80% fragmented indexes
-- Force maintenance during business hours (downtime!)
```

**✅ Good: Proactive Scheduled Maintenance**
```sql
-- Weekly maintenance window: Sunday 2 AM - 6 AM

-- Maintenance script (PostgreSQL):
DO $$
DECLARE
    idx RECORD;
    start_time TIMESTAMP;
BEGIN
    -- Only run during maintenance window
    IF EXTRACT(DOW FROM NOW()) != 0 OR EXTRACT(HOUR FROM NOW()) NOT BETWEEN 2 AND 5 THEN
        RAISE NOTICE 'Not in maintenance window, skipping';
        RETURN;
    END IF;
    
    -- Find indexes needing maintenance
    FOR idx IN
        SELECT
            schemaname,
            tablename,
            indexname,
            pg_relation_size(indexrelid) AS size_bytes
        FROM pg_stat_user_indexes
        WHERE idx_scan > 100  -- Only used indexes
        ORDER BY pg_relation_size(indexrelid) DESC
    LOOP
        start_time := clock_timestamp();
        
        RAISE NOTICE 'Reindexing %.% (Size: %)',
            idx.tablename,
            idx.indexname,
            pg_size_pretty(idx.size_bytes);
        
        -- Use CONCURRENTLY to avoid locking
        EXECUTE format('REINDEX INDEX CONCURRENTLY %I.%I',
            idx.schemaname, idx.indexname);
        
        RAISE NOTICE 'Completed in %',
            clock_timestamp() - start_time;
    END LOOP;
    
    -- Update statistics
    ANALYZE;
END $$;
```

**SQL Server: Adaptive Maintenance**
```sql
-- Ola Hallengren's maintenance solution (industry standard)
-- https://ola.hallengren.com/

DECLARE @fragmentation INT = 10;  -- Threshold percentages
DECLARE @rebuild_threshold INT = 30;

-- Reorganize lightly fragmented indexes
EXECUTE dbo.IndexOptimize
    @Databases = 'USER_DATABASES',
    @FragmentationLow = NULL,
    @FragmentationMedium = 'INDEX_REORGANIZE',
    @FragmentationHigh = 'INDEX_REBUILD_ONL INE',
    @FragmentationLevel1 = @fragmentation,
    @FragmentationLevel2 = @rebuild_threshold,
    @UpdateStatistics = 'ALL',
    @OnlyModifiedStatistics = 'Y';
```

### Practice 3: Use REINDEX CONCURRENTLY (When Available)

**❌ Bad: Regular REINDEX (Blocks All Access)**
```sql
-- PostgreSQL < 12
REINDEX INDEX idx_orders_created;

-- Behavior:
-- 1. Acquires ACCESS EXCLUSIVE lock on table
-- 2. Blocks all SELECT, INSERT, UPDATE, DELETE
-- 3. Duration: 10 minutes for 50GB index
-- 4. Downtime: 10 minutes (unacceptable for 24/7 systems

!)
```

**✅ Good: REINDEX CONCURRENTLY (PostgreSQL 12+)**
```sql
-- PostgreSQL 12+
REINDEX INDEX CONCURRENTLY idx_orders_created;

-- Behavior:
-- 1. Creates new index with temporary name
-- 2. Validates new index
-- 3. Brief exclusive lock (milliseconds) to swap indexes
-- 4. Drops old index
-- Duration: 12 minutes (20% overhead vs non-concurrent)
-- Downtime: < 100ms (acceptable!)

-- Caveat: Requires 2× disk space temporarily (old + new index)
```

**SQL Server: ONLINE Rebuild**
```sql
-- Rebuild online (Enterprise Edition)
ALTER INDEX idx_orders_created ON orders
REBUILD WITH (ONLINE = ON, MAXDOP = 4);

-- ONLINE = ON:
-- - Allows SELECT queries during rebuild
-- - Allows INSERT/UPDATE/DELETE (slightly slower)
-- - Brief schema lock at start and end (< 100ms)

-- Standard Edition: ONLINE not available, must use REORGANIZE
ALTER INDEX idx_orders_created ON orders REORGANIZE;
-- REORGANIZE:
-- - Online, no blocking
-- - Less thorough than REBUILD (defragments but doesn't rebuild from scratch)
-- - Good for fragmentation 10-30%
```

### Practice 4: Don't Rebuild Unused Indexes

**Waste of Resources:**
```sql
-- Check index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Never used!
ORDER BY pg_relation_size(indexrelid) DESC;

-- Example output:
-- indexname: idx_users_middle_name
-- idx_scan: 0 (never used in 6 months)
-- size: 2 GB

-- Decision: DROP, don't maintain!
DROP INDEX idx_users_middle_name;

-- Benefits:
-- - 2GB disk space freed
-- - Faster INSERTs/UPDATEs (one fewer index to update)
-- - Less maintenance work
```

---

## 5. Common Mistakes

### Mistake 1: Rebuilding All Indexes Blindly

**Problem:**
```sql
-- Maintenance script rebuilds EVERY index every week
FOR idx IN (SELECT indexname FROM pg_indexes)
LOOP
    REINDEX INDEX idx.indexname;
END LOOP;

-- Issues:
-- 1. Wastes time on small indexes (<100MB, rebuild takes seconds but not needed)
-- 2. Wastes time on unused indexes (idx_scan = 0)
-- 3. Wastes time on fresh indexes (created yesterday, 0% fragmentation)
-- 4. Burns I/O budget on low-priority indexes

-- Result:
-- - Maintenance window: 6 hours
-- - 80% of time spent on unnecessary rebuilds
-- - High-priority indexes delayed or skipped
```

**Solution: Selective Maintenance**
```sql
-- Only rebuild indexes that need it
SELECT
    schemaname,
    tablename,
    indexname,
    pg_relation_size(indexrelid) AS size_bytes,
    idx_scan,
    (pgstatindex(indexrelid)).avg_leaf_density AS density,
    (pgstatindex(indexrelid)).leaf_fragmentation AS fragmentation
FROM pg_stat_user_indexes
WHERE 
    idx_scan > 100  -- Used recently
   AND pg_relation_size(indexrelid) > 100 * 1024 * 1024  -- > 100MB
    AND (
        (pgstatindex(indexrelid)).avg_leaf_density < 70  -- Bloat > 30%
        OR (pgstatindex(indexrelid)).leaf_fragmentation > 30  -- Fragmentation > 30%
    )
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild only these indexes
-- Maintenance window: 1 hour (vs 6 hours before)
```

### Mistake 2: REINDEX Without ANALYZE

**Problem:**
```sql
-- Rebuild indexes but forget statistics
REINDEX TABLE orders;

-- Index is now optimal, BUT:
-- - Statistics still stale (last ANALYZE was 6 months ago)
-- - Optimizer thinks table has 1M rows (actually 10M now)
-- - Cardinality estimates wrong
-- - Query planner chooses seq scan instead of using new index!

-- Query performance: Still slow! (despite fresh index)
```

**Solution: Always ANALYZE After REINDEX**
```sql
-- Rebuild + update statistics
REINDEX TABLE orders;
ANALYZE orders;

-- Or: Combined command
REINDEX TABLE orders;
VACUUM ANALYZE orders;  -- VACUUM + ANALYZE together

-- Now:
-- - Index optimal (fragmentation 0%)
-- - Statistics fresh (cardinality accurate)
-- - Optimizer chooses index scan
-- - Queries fast! ✅
```

### Mistake 3: Maintenance During Peak Load

**Disaster Scenario:**
```sql
-- Maintenance script runs at 2 PM (peak traffic)
REINDEX TABLE orders;

-- Impact:
-- 1. ACCESS EXCLUSIVE lock acquired
-- 2. All queries on orders table blocked
-- 3. Connection pool fills up (threads waiting)
-- 4. Application timeouts (30s wait → failure)
-- 5. Cascading failures (other services depend on this)
-- 6. Site down for 15 minutes (duration of REINDEX)

-- Business impact:
-- - Lost orders: $50K revenue
-- - Customer complaints: 5000+
-- - reputation damage
```

**Solution: Maintenance Windows + CONCURRENTLY**
```sql
-- Option 1: Off-peak maintenance
-- Run at 3 AM when traffic is 5% of peak

-- Option 2: REINDEX CONCURRENTLY (minimal disruption)
REINDEX INDEX CONCURRENTLY idx_orders_created;
-- Online, runs during business hours safely

-- Option 3: Rolling maintenance (multi-replica setup)
-- 1. Remove replica-1 from load balancer
-- 2. REINDEX on replica-1
-- 3. Add replica-1 back, remove replica-2
-- 4. REINDEX on replica-2
-- 5. Repeat for all replicas
-- 6. Promote replica as new primary (zero downtime!)
```

### Mistake 4: Not Monitoring Maintenance Duration

**Problem:**
```sql
-- Maintenance script has no time limits
-- Index grows 10× over 2 years
-- REINDEX now takes 6 hours (used to take 30 minutes)
-- Exceeds maintenance window (4 hours)
-- Maintenance incomplete, index still fragmented

-- Scenario:
-- 3 AM: Maintenance starts
-- 7 AM: Maintenance still running (exceeded window)
-- 8 AM: Peak traffic starts, system slow (fragmented index)
-- 9 AM: Manual intervention required (kill long-running REINDEX)
```

**Solution: Monitor and Timeout**
```sql
-- Set statement timeout
SET statement_timeout = '2h';  -- Kill queries exceeding 2 hours

-- Log progress
DO $$
DECLARE
    idx RECORD;
    start_time TIMESTAMP;
    duration INTERVAL;
BEGIN
    FOR idx IN SELECT * FROM indexes_to_rebuild
    LOOP
        start_time := clock_timestamp();
        
        BEGIN
            EXECUTE 'REINDEX INDEX CONCURRENTLY ' || idx.indexname;
            duration := clock_timestamp() - start_time;
            
            INSERT INTO maintenance_log (index_name, duration, status)
            VALUES (idx.indexname, duration, 'SUCCESS');
            
            IF duration > INTERVAL '1 hour' THEN
                -- Alert: Index taking too long, might need partitioning
                PERFORM pg_notify('maintenance_alert', 
                    'Index ' || idx.indexname || ' took ' || duration);
            END IF;
        EXCEPTION WHEN OTHERS THEN
            INSERT INTO maintenance_log (index_name, duration, status, error)
            VALUES (idx.indexname, clock_timestamp() - start_time, 'FAILED', SQLERRM);
        END;
    END LOOP;
END $$;
```

---

## 6. Security Considerations

### 1. Maintenance Privilege Escalation

**Risk:**
```sql
-- Maintenance user has broad privileges
CREATE USER maintenance_user WITH PASSWORD 'secret';
GRANT ALL PRIVILEGES ON DATABASE mydb TO maintenance_user;

-- Attacker compromises maintenance_user credentials
-- Can now:
-- - DROP indexes (causes performance degradation)
-- - REINDEX during peak hours (DOS)
-- - Access/modify data
```

**Mitigation:**
```sql
-- Principle of least privilege: Only grant REINDEX permission
CREATE USER maintenance_user;

-- Grant only necessary permissions
GRANT CONNECT ON DATABASE mydb TO maintenance_user;
GRANT USAGE ON SCHEMA public TO maintenance_user;

-- Grant REINDEX permission (PostgreSQL 14+)
GRANT MAINTAIN ON ALL TABLES IN SCHEMA public TO maintenance_user;

-- Revoke data access
REVOKE SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public FROM maintenance_user;

-- Now maintenance_user can:
-- ✅ REINDEX
-- ✅ VACUUM
-- ✅ ANALYZE
-- ❌ SELECT data
-- ❌ Modify data
-- ❌ DROP tables
```

### 2. Audit Maintenance Operations

**Tracking:**
```sql
-- Enable audit logging for maintenance operations
ALTER SYSTEM SET log_statement = 'ddl';  -- Log all DDL (REINDEX, etc.)
ALTER SYSTEM SET log_duration = on;      -- Log query duration

-- Create audit table
CREATE TABLE maintenance_audit (
    id SERIAL PRIMARY KEY,
    operation VARCHAR(50),
    object_name VARCHAR(200),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    duration INTERVAL,
    performed_by VARCHAR(100),
    status VARCHAR(20)
);

-- Log all maintenance
CREATE OR REPLACE FUNCTION log_maintenance()
RETURNS EVENT_TRIGGER AS $$
BEGIN
    INSERT INTO maintenance_audit (
        operation,
        object_name,
        started_at,
        performed_by
    ) VALUES (
        TG_TAG,  -- REINDEX, VACUUM, etc.
        TG_OBJECT_NAME,
        NOW(),
        SESSION_USER
    );
END;
$$ LANGUAGE plpgsql;

CREATE EVENT TRIGGER mainten ance_audit_trigger
ON ddl_command_end
EXECUTE FUNCTION log_maintenance();
```

---

## 7. Performance Optimization

### Optimization 1: Parallel Index Rebuild

**Sequential (Slow):**
```sql
-- Rebuild indexes one at a time
REINDEX INDEX idx1;  -- 30 minutes
REINDEX INDEX idx2;  -- 25 minutes
REINDEX INDEX idx3;  -- 20 minutes
-- Total: 75 minutes
```

**Parallel (Fast):**
```sql
-- PostgreSQL: Run concurrent sessions
-- Session 1:
REINDEX INDEX CONCURRENTLY idx1;

-- Session 2 (simultaneously):
REINDEX INDEX CONCURRENTLY idx2;

-- Session 3 (simultaneously):
REINDEX INDEX CONCURRENTLY idx3;

-- Total: 30 minutes (limited by slowest index)
-- Speedup: 2.5× faster!

-- Caveat: Monitor system resources (I/O, CPU)
-- Don't overwhelm system (limit to 3-4 concurrent rebuilds)
```

**SQL Server: MAXDOP**
```sql
-- Limit parallelism per index rebuild
ALTER INDEX idx_orders_created ON orders
REBUILD WITH (ONLINE = ON, MAXDOP = 4);  -- Use 4 CPU cores

-- System-wide parallel rebuilds:
-- Job 1: REBUILD idx1 WITH (MAXDOP = 4)
-- Job 2: REBUILD idx2 WITH (MAXDOP = 4)
-- Total CPU cores used: 8 (out of 32 available)
```

### Optimization 2: Incremental Maintenance (REORGANIZE vs REBUILD)

**Rebuild Everything (Overkill):**
```sql
-- Every Monday: Full REBUILD of all indexes
-- Duration: 6 hours
-- I/O cost: 500GB writes per week
```

**Tiered Maintenance (Optimized):**
```sql
-- Daily (10 PM): Light maintenance
-- - REORGANIZE fragmented indexes (10-30% fragmentation)
-- - Duration: 30 minutes
-- - Online, no downtime

-- Weekly (Sunday 3 AM): Medium maintenance
-- - REBUILD heavily fragmented indexes (>30%)
-- - Duration: 2 hours

-- Monthly (First Sunday): Deep maintenance
-- - REBUILD all indexes on critical tables
-- - VACUUM FULL (reclaim space)
-- - Duration: 4 hours

-- Total maintenance time: 11.5 hours/month (vs 24 hours with weekly full rebuilds)
```

**Adaptive Script:**
```sql
--  Choose action based on fragmentation level
CREATE OR REPLACE FUNCTION maintain_index(idx_name TEXT)
RETURNS void AS $$
DECLARE
    frag_pct FLOAT;
    density_pct FLOAT;
BEGIN
    -- Get index stats
    SELECT 
        (pgstatindex(idx_name::regclass)).leaf_fragmentation,
        (pgstatindex(idx_name::regclass)).avg_leaf_density
    INTO frag_pct, density_pct;
    
    -- Decision logic
    IF frag_pct > 50 OR density_pct < 50 THEN
        -- Severe: Full rebuild
        EXECUTE 'REINDEX INDEX CONCURRENTLY ' || idx_name;
        RAISE NOTICE '%: REBUILT (frag: %, density: %)', idx_name, frag_pct, density_pct;
    ELSIF frag_pct > 20 OR density_pct < 70 THEN
        -- Moderate: Reorganize (if supported by DB)
        -- PostgreSQL doesn't have REORGANIZE, would use REINDEX
        EXECUTE 'REINDEX INDEX CONCURRENTLY ' || idx_name;
        RAISE NOTICE '%: REORGANIZED (frag: %, density: %)', idx_name, frag_pct, density_pct;
    ELSE
        -- Healthy: Skip
        RAISE NOTICE '%: SKIPPED (healthy, frag: %, density: %)', idx_name, frag_pct, density_pct;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## 8. Examples

### Example 1: E-commerce Production Maintenance Schedule

**Infrastructure:**
- Database: PostgreSQL 15
- Table: orders (50M rows, 100GB)
- Indexes: 8 indexes (40GB total)
- Traffic: 10K orders/day, 50K queries/sec peak

**Maintenance Schedule:**

```sql
-- DAILY (11 PM - 12 AM): Light maintenance
-- Target: Indexes with moderate issues
SELECT maintenance_daily();

CREATE OR REPLACE FUNCTION maintenance_daily()
RETURNS void AS $$
DECLARE
    idx RECORD;
BEGIN
    -- Only moderate fragmentation
    FOR idx IN
        SELECT indexname
        FROM pg_stat_user_indexes psi
        JOIN (
            SELECT indexname, (pgstatindex(indexrelid)).leaf_fragmentation AS frag
            FROM pg_stat_user_indexes
            WHERE schemaname = 'public'
        ) stats USING (indexname)
        WHERE stats.frag BETWEEN 20 AND 40
        AND psi.idx_scan > 1000  -- Only used indexes
    LOOP
        -- Quick REINDEX (no CONCURRENTLY, during low traffic)
        EXECUTE 'REINDEX INDEX ' || idx.indexname;
    END LOOP;
    
    -- Update statistics (fast on recently changed tables)
    ANALYZE;
END;
$$ LANGUAGE plpgsql;

-- WEEKLY (Sunday 3 AM - 6 AM): Deep maintenance
-- Target: All problematic indexes
SELECT maintenance_weekly();

CREATE OR REPLACE FUNCTION maintenance_weekly()
RETURNS void AS $$
DECLARE
    idx RECORD;
BEGIN
    -- High fragmentation or bloat
    FOR idx IN
        SELECT indexname
        FROM pg_stat_user_indexes psi
        JOIN (
            SELECT 
                indexname,
                (pgstatindex(indexrelid)).leaf_fragmentation AS frag,
                (pgstatindex(indexrelid)).avg_leaf_density AS density
            FROM pg_stat_user_indexes
            WHERE schemaname = 'public'
        ) stats USING (indexname)
        WHERE (stats.frag > 40 OR stats.density < 60)
        AND psi.idx_scan > 100
        ORDER BY pg_relation_size(psi.indexrelid) DESC
    LOOP
        EXECUTE 'REINDEX INDEX CONCURRENTLY ' || idx.indexname;
    END LOOP;
    
    -- Full table analyze
    VACUUM ANALYZE;
END;
$$ LANGUAGE plpgsql;

-- MONTHLY (First Sunday 2 AM - 7 AM): Full maintenance
-- Target: All tables and indexes
SELECT maintenance_monthly();

CREATE OR REPLACE FUNCTION maintenance_monthly()
RETURNS void AS $$
BEGIN
    -- Rebuild all indexes on critical tables
    REINDEX TABLE CONCURRENTLY orders;
    REINDEX TABLE CONCURRENTLY products;
    REINDEX TABLE CONCURRENTLY customers;
    
    -- Reclaim space
    VACUUM FULL ANALYZE orders;
    
    -- Drop unused indexes
    DELETE FROM pg_indexes
    WHERE indexname IN (
        SELECT indexname
        FROM pg_stat_user_indexes
        WHERE idx_scan = 0
        AND indexrelid::regclass::text NOT LIKE '%_pkey'  -- Keep primary keys
    );
END;
$$ LANGUAGE plpgsql;
```

### Example 2: Monitoring Dashboard Queries

**Maintenance Health Dashboard:**

```sql
-- Create view for real-time monitoring
CREATE OR REPLACE VIEW index_health AS
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    CASE 
        WHEN idx_scan = 0 THEN 'UNUSED'
        WHEN (pgstatindex(indexrelid)).leaf_fragmentation > 50 THEN 'CRITICAL'
        WHEN (pgstatindex(indexrelid)).leaf_fragmentation > 30 THEN 'WARNING'
        WHEN (pgstatindex(indexrelid)).avg_leaf_density < 50 THEN 'CRITICAL'
        WHEN (pgstatindex(indexrelid)).avg_leaf_density < 70 THEN 'WARNING'
        ELSE 'HEALTHY'
    END AS health_status,
    ROUND((pgstatindex(indexrelid)).leaf_fragmentation::numeric, 2) AS fragmentation_pct,
    ROUND((pgstatindex(indexrelid)).avg_leaf_density::numeric, 2) AS density_pct,
    CASE
        WHEN idx_scan = 0 THEN 'Drop index'
        WHEN (pgstatindex(indexrelid)).leaf_fragmentation > 50 THEN 'REINDEX NOW'
        WHEN (pgstatindex(indexrelid)).leaf_fragmentation > 30 THEN 'Schedule REINDEX'
        WHEN (pgstatindex(indexrelid)).avg_leaf_density < 50 THEN 'REINDEX NOW'
        WHEN (pgstatindex(indexrelid)).avg_leaf_density < 70 THEN 'Schedule REINDEX'
        ELSE 'No action needed'
    END AS recommendation
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Query dashboard
SELECT * FROM index_health WHERE health_status != 'HEALTHY';

-- Example output:
-- table_name   | index_name          | size   | health   | frag% | density% | recommendation
-- -------------|---------------------|--------|----------|-------|----------|------------------
-- orders       | idx_orders_created  | 5 GB   | CRITICAL | 75.3  | 45.2     | REINDEX NOW
-- products     | idx_products_category| 800 MB | WARNING | 35.1  | 68.5     | Schedule REINDEX
-- customers    | idx_customers_unused | 2 GB   | UNUSED   | N/A   | N/A      | Drop index
```

---

## 9. Real-World Use Cases

### Use Case 1: GitHub - Massive Index Maintenance

**Scale:**
- Database: MySQL (later PostgreSQL)
- Tables: 300+ tables, 50TB+
- Indexes: 1000+ indexes, 20TB
- Challenge: Maintenance without downtime

**Strategy:**
1. **Online schema changes** (pt-online-schema-change for MySQL)
2. **Gradual rollout** (one table at a time)
3. **Monitor replication lag** (ensure replicas keep up)
4. **Automated rollback** (if errors detected)

**Maintenance approach:**
```sql
-- Staggered maintenance over 7 days
-- Day 1: Top 10 largest tables (20% of data)
-- Day 2-7: Remaining tables in batches

-- Each table:
-- 1. Check replication lag < 5 seconds
-- 2. REBUILD indexes (online)
-- 3. ANALYZE
-- 4. Verify query performance
-- 5. If issues: Rollback, investigate
-- 6. If success: Proceed to next table
```

**Result:**
- Zero downtime
- Query performance improved 30%
- Disk space reclaimed: 5TB (25%)

### Use Case 2: Stripe - Payment Processing Uptime

**Requirements:**
- 99.99% uptime (52 minutes downtime/year allowed)
- Maintenance windows: Not acceptable (24/7 payment processing)

**Solution: Rolling Maintenance Across Replicas**

```sql
-- Infrastructure: 1 primary + 3 replicas

-- Step 1: Remove replica-1 from load balancer
-- - Traffic now distributed across primary + 2 replicas

-- Step 2: Maintenance on replica-1 (safe, offline from traffic)
REINDEX DATABASE stripe_payments;  -- Can use blocking REINDEX (faster)
VACUUM FULL ANALYZE;

-- Step 3: Verify replica-1 caught up with replication
SELECT pg_last_wal_replay_lsn() = pg_current_wal_lsn();  -- Should be TRUE

-- Step 4: Add replica-1 back to load balancer
-- Step 5: Repeat for replica-2, replica-3

-- Step 6: Promote replica-1 as new primary (planned failover)
-- - Old primary becomes replica
-- - Maintenance on old primary (now replica)

-- Total downtime: 0 seconds ✅
-- Maintenance duration: 2 hours per replica = 8 hours total spread over time
```

---

## 10. Interview Questions

### Q1: How do you prioritize which indexes to rebuild in a limited maintenance window?

**Senior Answer:**

```
**Prioritization Framework:**

**1. Impact Assessment (Score 1-10):**
- Query frequency: How many queries use this index per second?
- Query importance: User-facing vs batch jobs?
- Current performance: How slow are queries now?

Example scoring:
- idx_orders_checkout: 1000 queries/sec, user-facing, 5s slow → Score: 10
- idx_analytics_batch: 1 query/hour, background, 2min slow → Score: 3

**2. Severity Measurement:**
- Fragmentation %: > 50% = critical, 30-50% = high, < 30% = low
- Bloat %: > 60% = critical, 40-60% = high, < 40% = low
- Size: Larger indexes have bigger impact when slow

**3. Maintenance Cost:**
- Rebuild duration: idx_orders (5GB) = 30 min, idx_logs (50GB) = 5 hours
- Lock implications: Can we use CONCURRENTLY?
- Resource usage: I/O and CPU impact

**4. Combined Priority Score:**
Priority = (Impact × Severity) / Maintenance Cost

Example:
idx_orders_checkout:
- Impact: 10 (critical user-facing)
- Severity: 8 (60% fragmented)
- Cost: 30 minutes
- Priority: (10 × 8) / 30 = 2.67

idx_logs_archive:
- Impact: 2 (rarely queried)
- Severity: 9 (80% fragmented)
- Cost: 300 minutes
- Priority: (2 × 9) / 300 = 0.06

→ Rebuild idx_orders_checkout first!

**5. Execution Strategy:**

```sql
-- Query to generate prioritized list
WITH index_metrics AS (
    SELECT
        indexname,
        idx_scan AS queries_per_day,
        CASE 
            WHEN tablename IN ('orders', 'payments', 'users') THEN 10
            WHEN tablename IN ('products', 'inventory') THEN 7
            ELSE 3
        END AS business_importance,
        (pgstatindex(indexrelid)).leaf_fragmentation AS frag_pct,
        pg_relation_size(indexrelid) / (1024*1024*1024.0) AS size_gb
    FROM pg_stat_user_indexes
    WHERE schemaname = 'public'
),
scored AS (
    SELECT
        *,
        queries_per_day * business_importance AS impact_score,
        CASE 
            WHEN frag_pct > 50 THEN 10
            WHEN frag_pct > 30 THEN 7
            WHEN frag_pct > 10 THEN 3
            ELSE 0
        END AS severity_score,
        size_gb * 15 AS estimated_minutes  -- 15 min per GB
    FROM index_metrics
)
SELECT
    indexname,
    impact_score,
    severity_score,
    estimated_minutes,
    ROUND((impact_score * severity_score)::numeric / estimated_minutes, 2) AS priority
FROM scored
WHERE frag_pct > 10  -- Only consider fragmented indexes
ORDER BY priority DESC
LIMIT 20;  -- Top 20 for 4-hour maintenance window
```

**Real-world example:**

4-hour maintenance window = 240 minutes

Prioritized list:
1. idx_orders_status (30 min, Priority: 5.2) [REBUILD]
2. idx_payments_user (20 min, Priority: 4.8) [REBUILD]
3. idx_products_category (45 min, Priority: 3.1) [REBUILD]
4. idx_users_email (60 min, Priority: 2.5) [REBUILD]
5. idx_logs_timestamp (180 min, Priority: 0.3) [SKIP - not enough time]

Total: 155 minutes of 240 available

Decision: Rebuild top 4, defer idx_logs to next window or run separately online.

**Interview insight:**

"In a previous role, we had a 2-hour Sunday maintenance window for a 10TB database with 500 indexes. I implemented this scoring system and immediately identified that 80% of user-visible performance issues came from just 12 indexes (the 80/20 rule). We focused maintenance on those 12, completing in 90 minutes, and query performance improved 10× for 95% of users. The remaining 488 indexes were either barely used or performed fine despite fragmentation."
```

### Q2: What's the difference between REORGANIZE and REBUILD, and when would you use each?

**Staff Answer:**

```
**REORGANIZE (Online Defragmentation):**

**How it works:**
- Scans index leaf pages
- Moves rows within pages to eliminate gaps
- Compacts pages by combining partial pages
- Does NOT rebuild index from scratch
- Online operation (no exclusive locks)

**Characteristics:**
- Always online (allows reads and writes)
- Slower than REBUILD (incremental vs rebuild from scratch)
- Doesn't require extra disk space
- Doesn't reset fill factor
- Best for fragmentation 10-30%

**Algorithm:**
```
FOR each leaf page in index:
    IF page is < 70% full:
        Find adjacent pages that are also sparse
        Merge rows from sparse pages into fewer pages
        Mark empty pages for reuse
    Update page pointers to maintain logical order
```

**Example:**
```sql
-- SQL Server
ALTER INDEX idx_orders_created ON orders REORGANIZE;

-- MySQL (OPTIMIZE TABLE does this)
OPTIMIZE TABLE orders;

-- Duration: 45 minutes for 5GB index
-- Disk space: 0 extra (in-place)
-- Locking: None (online)
-- Result: Fragmentation 35% → 15%
```

---

**REBUILD (Full Reconstruction):**

**How it works:**
- Creates entirely new index structure
- Scans table, extracts all keys
- Sorts keys
- builds new B-Tree from scratch
- Atomically replaces old index with new
- Can be online (REBUILD ONLINE) or offline

**Characteristics:**
- Offline: Exclusive lock (blocks everything)
- Online: Allows reads, brief lock for swap
- Faster than REORGANIZE for severe fragmentation
- Requires 2× disk space temporarily (old + new)
- Resets fill factor to configured value
- Best for fragmentation > 30%

**Algorithm:**
```
1. Scan table, extract all (key, row_id) pairs
2. Sort pairs by key
3. Build new B-Tree with optimal fill factor
4. Create new index structure on disk
5. [ONLINE only] Shadow index approach:
   - Track ongoing changes during build
   - Apply changes to new index
6. Acquire brief exclusive lock (<100ms)
7. Swap new index for old
8. Drop old index structure
```

**Example:**
```sql
-- SQL Server: Offline rebuild (blocks all access)
ALTER INDEX idx_orders_created ON orders REBUILD;
-- Duration: 20 minutes for 5GB index
-- Disk space: 10GB (old + new)
-- Locking: Exclusive (blocks all queries)

-- SQL Server: Online rebuild (Enterprise Edition)
ALTER INDEX idx_orders_created ON orders REBUILD WITH (ONLINE = ON);
-- Duration: 25 minutes (25% overhead vs offline)
-- Disk space: 10GB (old + new)
-- Locking: Minimal (< 100ms exclusive lock during swap)

-- PostgreSQL: Always requires brief exclusive lock at end
REINDEX INDEX CONCURRENTLY idx_orders_created;
-- Duration: 25 minutes
-- Disk space: 10GB (old + new)
-- Locking: Brief exclusive lock (< 100ms)
```

---

**Decision Matrix:**

| Factor | REORGANIZE | REBUILD |
|--------|------------|---------|
| **Fragmentation level** | 10-30% | > 30% |
| **Downtime tolerance** | None (always online) | Depends (REBUILD ONLINE = minimal) |
| **Disk space available** | Low (no extra space) | High (needs 2× index size) |
| **Time constraint** | Longer (45 min) | Faster (20 min) |
| **Fill factor reset** | No (keeps old fill factor) | Yes (applies new fill factor) |
| **Database edition** | Standard/Express | Enterprise (for ONLINE) |

**Real-world decision tree:**

```
Fragmentation detected?
├─ Yes → How severe?
│   ├─ 10-20%: Monitor, no action yet
│   ├─ 20-30%: REORGANIZE
│   │   ├─ Can't spare disk space? REORGANIZE
│   │   └─ Need it fast? REORGANIZE (online, no downtime)
│   └─ > 30%: REBUILD
│       ├─ Production system? REBUILD WITH (ONLINE = ON)
│       │   ├─ Enterprise Edition? Yes → REBUILD ONLINE
│       │   └─ Standard Edition? Use REORGANIZE (best you can do)
│       └─ Maintenance window available? REBUILD (offline, faster)
└─ No → No action needed, monitor
```

**Production example:**

Scenario: SaaS application, 24/7 uptime required, SQL Server Standard Edition

Problem: idx_transactions (10GB) at 45% fragmentation

Decision process:
1. Fragmentation > 30% → REBUILD preferred
2. BUT: Standard Edition → No REBUILD ONLINE
3. Alternative: REORGANIZE
   - Online: ✅
   - Reduces fragmentation: 45% → 15% (acceptable)
   - Duration: 60 minutes (acceptable, online)
4. Decision: REORGANIZE

If Enterprise Edition available:
- REBUILD WITH (ONLINE = ON)
- Duration: 30 minutes
- Result: 0% fragmentation (vs 15% with REORGANIZE)
- Extra cost: License upgrade ($15K/year), disk space (10GB temp)

**Interview insight:**

"I worked on a financial trading platform where we couldn't afford ANY downtime (every second = $$$). We were on SQL Server Standard Edition, so REBUILD ONLINE wasn't available. We implemented a staged approach: (1) REORGANIZE critical indexes weekly during low-traffic hours, (2) scheduled REBUILD ONLINE migration by upgrading to Enterprise Edition during yearly maintenance, (3) for the most critical table, we used read replicas—took one replica offline, REBUILD there, promoted it to primary, then REBUILD the old primary. This gave us REBUILD performance with zero user-facing downtime, at the cost of operational complexity."
```

---

## 11. Summary

### Index Maintenance Key Principles

1. **Indexes Degrade Over Time**
   - Fragmentation: Pages out of order → Random I/O
   - Bloat: Empty space → Wasted disk and cache
   - Stale statistics → Wrong query plans
   - Performance degradation: 2-10× slowdown

2. **Regular Monitoring Essential**
   - Fragmentation > 30%: REBUILD
   - Fragmentation 10-30%: REORGANIZE
   - Bloat > 50%: REBUILD
   - Unused indexes (idx_scan = 0): DROP

3. **Maintenance Strategies**
   - Scheduled: Weekly/monthly windows
   - Adaptive: Based on fragmentation metrics
   - Online: REINDEX CONCURRENTLY, REBUILD ONLINE
   - Tiered: Daily light, weekly deep, monthly full

4. **Always Pair with Statistics**
   - REINDEX + ANALYZE
   - Update statistics after maintenance
   - Optimizer needs fresh cardinality

### Maintenance Checklist

**Weekly (30 min - 2 hours):**
- [ ] Check index fragmentation
- [ ] REORGANIZE indexes 10-30% fragmented
- [ ] REBUILD indexes > 30% fragmented (if time permits)
- [ ] ANALYZE tables
- [ ] Review unused indexes (idx_scan = 0)

**Monthly (2-6 hours):**
- [ ] REBUILD all critical table indexes
- [ ] VACUUM FULL (reclaim space)
- [ ] DROP unused indexes
- [ ] Update statistics on all tables
- [ ] Check index bloat
- [ ] Review maintenance duration trends

**Quarterly (4-8 hours):**
- [ ] Deep maintenance on all indexes
- [ ] Partition old data (if using partitioning)
- [ ] Review index strategies (still needed?)
- [ ] Performance baseline comparison
- [ ] Disk space cleanup

### Command Quick Reference

**PostgreSQL:**
```sql
-- Check fragmentation
SELECT (pgstatindex('idx_name')).leaf_fragmentation;

-- Rebuild (blocks)
REINDEX INDEX idx_name;

-- Rebuild (online, PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_name;

-- Update statistics
ANALYZE table_name;
```

**SQL Server:**
```sql
-- Check fragmentation
SELECT avg_fragmentation_in_percent 
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('table'), NULL, NULL, 'DETAILED');

-- Reorganize (online)
ALTER INDEX idx_name ON table REORGANIZE;

-- Rebuild (offline)
ALTER INDEX idx_name ON table REBUILD;

-- Rebuild (online, Enterprise)
ALTER INDEX idx_name ON table REBUILD WITH (ONLINE = ON);

-- Update statistics
UPDATE STATISTICS table WITH FULLSCAN;
```

**MySQL:**
```sql
-- Check fragmentation
SHOW TABLE STATUS LIKE 'table';

-- Rebuild (OPTIMIZE rebuilds)
OPTIMIZE TABLE table;

-- Analyze
ANALYZE TABLE table;
```

### Performance Impact Summary

| Maintenance Type | Downtime | Duration (5GB index) | Result |
|------------------|----------|---------------------|---------|
| **None** | 0 | 0 | Query time: 2.5s (degraded) |
| **REORGANIZE** | 0 (online) | 45 min | Query time: 0.5s (60% better) |
| **REBUILD** | 20 min (offline) | 20 min | Query time: 0.15s (90% better) |
| **REBUILD ONLINE** | < 100ms | 25 min | Query time: 0.15s (90% better, minimal downtime!) |

### Key Takeaway

**"Index maintenance is like changing your car's oil—skip it at your peril. A 2-hour monthly maintenance window can save you from a 48-hour weekend emergency. Measure, schedule, automate."**

In production: Monitor index health weekly, schedule maintenance monthly, and always use CONCURRENTLY/ONLINE options when available. Your 3 AM self will thank you.

---

**Next Reading:**
- [09_Write_Amplification.md](09_Write_Amplification.md) - Understanding the cost of indexes on write performance
- [10_README.md](10_README.md) - Section overview and index selection decision matrix
- [05_Query_Optimization/07_Statistics_Management.md](../05_Query_Optimization/07_Statistics_Management.md) - Deep dive on statistics
