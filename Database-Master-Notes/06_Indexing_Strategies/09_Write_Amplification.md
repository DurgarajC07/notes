# Write Amplification - Understanding Index Write Costs

## 1. Concept Explanation

**Write amplification** is the phenomenon where a single logical write (e.g., INSERT one row) results in multiple physical writes to disk due to indexes, logs, and data structures. Every index on a table multiplies the write cost, making write-heavy workloads significantly slower.

Think of write amplification like carbon copies:
- **Without indexes:** Write 1 form → 1 physical write
- **With 5 indexes:** Write 1 form → 6 physical writes (1 table + 5 indexes)
- **With 20 indexes:** Write 1 form → 21 physical writes (1 table + 20 indexes)

### Write Amplification Formula

```
Write Amplification Factor (WAF) = Physical Writes / Logical Writes

Example:
- INSERT 1 row into table
- Table has 10 indexes
- Each INSERT triggers:
  - 1 table write (heap)
  - 10 index writes
  - 1 WAL (Write-Ahead Log) write
  - Total: 12 physical writes

WAF = 12 / 1 = 12×

If each write is 8KB:
- Logical write: 8KB
- Physical writes: 12 × 8KB = 96KB
- Amplification: 12× more data written to disk!
```

### Visualization

```
Single INSERT operation:

User application:
INSERT INTO orders (customer_id, amount, status, created_at) VALUES (12345, 99.99, 'pending', NOW());

Physical writes triggered:

1. Heap table write:
   [Write row data to table page: 8KB]

2. Primary key index (id):  
   [Update B-Tree leaf page: 8KB]
   [Update parent node if split: 8KB]  ← Page split!

3. Index on customer_id:
   [Update B-Tree leaf page: 8KB]

4. Index on status:
   [Update B-Tree leaf page: 8KB]

5. Index on created_at:
   [Update B-Tree leaf page: 8KB]

6. Composite index (customer_id, created_at):
   [Update B-Tree leaf page: 8KB]

7. Write-Ahead Log (WAL):
   [Append to transaction log: 2KB]

Total physical I/O: 6 pages × 8KB + 2KB WAL = 50KB
Single row size: 100 bytes

Write amplification: 50KB / 100 bytes = 512×!
```

**Key Factors:**
- **Number of indexes:** More indexes = more writes
- **Page splits:** B-Tree page full → Split → Extra writes
- **WAL/Redo logs:** Every change logged for durability
- **Replication:** Changes replicated to standby servers

---

## 2. Why It Matters

### Production Impact: Write-Heavy Workloads

**Scenario: High-Frequency Trading System**

```sql
-- Table: trades (10K inserts/second)
CREATE TABLE trades (
    trade_id BIGSERIAL PRIMARY KEY,
    ticker VARCHAR(10),
    price DECIMAL(10,2),
    volume INT,
    trader_id INT,
    timestamp TIMESTAMPTZ
);
```

**Case 1: Minimal Indexes (WAF = 3×)**
```sql
-- Only primary key
CREATE INDEX idx_trades_pkey ON trades(trade_id);

-- Writes per INSERT:
-- 1. Table heap: 1 write
-- 2. Primary key index: 1 write
-- 3. WAL: 1 write
-- Total: 3 writes

-- Throughput: 10K inserts/sec
-- Physical writes: 30K writes/sec
-- Disk I/O: 30K × 8KB = 240MB/sec (manageable)
```

**Case 2: Over-Indexed (WAF = 21×)**
```sql
-- Developer adds index on every column
CREATE INDEX idx_trades_pkey ON trades(trade_id);
CREATE INDEX idx_trades_ticker ON trades(ticker);
CREATE INDEX idx_trades_price ON trades(price);
CREATE INDEX idx_trades_volume ON trades(volume);
CREATE INDEX idx_trades_trader ON trades(trader_id);
CREATE INDEX idx_trades_timestamp ON trades(timestamp);
-- Plus 15 composite indexes for various query patterns...
-- Total: 20 indexes

-- Writes per INSERT:
-- 1. Table heap: 1 write
-- 2. 20 indexes: 20 writes (best case, no page splits)
-- 3. WAL: 1 write
-- Total: 22 writes (reality: 30+ with page splits)

-- Throughput: 10K inserts/sec
-- Physical writes: 10K × 30 = 300K writes/sec
-- Disk I/O: 300K × 8KB = 2.4GB/sec (SSD saturated!)

-- Result:
-- - INSERT latency: 5ms → 150ms (30× slower!)
-- - Replication lag: 0s → 5 minutes (replicas can't keep up)
-- - Disk wear: SSD lifetime reduced by 10×
-- - System instability: Timeouts, queue buildup, cascading failures
```

### Real Incident: Over-Indexing Disaster

```
Company: Social media analytics platform
Table: events (user actions)
Initial state:
- 3 indexes
- 50K events/sec
- INSERT latency: 2ms

Developer adds analytics features:
- Creates 25 indexes (one for each dashboard metric)
- "More indexes = faster queries, right?"

24 hours later:
- ALERT: Database write throughput critically low
- INSERT latency: 2ms → 500ms (250× slower!)
- Replication lag: 6 hours (replicas 6 hours behind primary)
- Disk space: 500GB → 2TB (indexes + bloat)
- SSD endurance warning: Writes exceeding rated limit

Investigation:
- WAF increased from 5× to 35×
- 50K events/sec × 35 writes = 1.75M physical writes/sec
- SSD rated for 100K IOPS → Running at 1750% of capacity

Emergency fix:
1. Drop 20 least-used indexes (keep only 5)
2. VACUUM to reclaim space
3. Restore performance: latency back to 3ms

Cost:
- 6 hours of degraded service
- 20TB of excess SSD writes (shortened drive life)
- $50K in lost analytics accuracy (data lag)

Lesson:
- Each index has a cost
- Measure before adding indexes
- WAF monitoring essential for write-heavy systems
```

---

## 3. Internal Working

### Write Amplification Sources

**1. Table Heap Write**
```
Single INSERT:
- Allocate space in heap page
- Write row data
- Update page header
- Cost: 1 page write (8KB)
```

**2. Index Updates**
```
For each index:
- Locate correct leaf page in B-Tree
- Insert key-value pair
- If page full → Page split:
  - Allocate new page
  - Split keys between old and new page
  - Update parent node pointer
  - Possibly cascade split upward
  - Cost: 2-5 page writes per split

Best case (no split): 1 write per index
Worst case (split): 5 writes per index

Total: N indexes × 1-5 writes
```

**3. WAL (Write-Ahead Log)**
```
Before actual data write:
- Append change to transaction log
- Contains: old value, new value, transaction ID
- Ensures durability (crash recovery)
- Cost: 1-2KB per operation

Benefit: Sequential writes (fast)
Cost: Extra physical write
```

**4. Replication**
```
Streaming replication:
- Primary writes to WAL
- WAL shipped to standbys
- Standbys apply changes
- Effective WAF × (1 + number of replicas)

Example:
- 1 primary + 2 replicas
- WAF on primary: 10×
- Total cluster writes: 10× × 3 = 30×
```

### LSM-Tree Write Amplification

**B-Tree (Traditional) Write Amplification:**
```
INSERT:
1. Find leaf page: 4 reads (tree depth 4)
2. Modify leaf page: 1 write
3. If page full: Split page: 3-5 writes
4. Update parent: 1 write
5. WAL: 1 write

Total: 1 read + 6-8 writes (best case)
WAF: 7-9×
```

**LSM-Tree (Log-Structured Merge Tree) Write Amplification:**
```
INSERT (Short-term):
1. Append to in-memory memtable: 0 writes (RAM)
2. Append to WAL: 1 sequential write
3. When memtable full: Flush to L0 SSTable: 1 sequential write

Total immediate writes: 2 (both sequential)
WAF immediate: 2× (excellent!)

Compaction (Background):
- Merge L0 → L1: Read all L0 + L1, write merged L1
  - WAF: 2× (read + write)
- Merge L1 → L2: Read all L1 + L2, write merged L2
  - WAF: 2×
- Cascade through levels: L0 → L1 → L2 → L3 → L4
  - Total WAF: 2× per level × 5 levels = 10×

Amortized WAF over time: 10-30× (depends on compaction strategy)

Trade-off:
- Writes: Fast initially (sequential), expensive later (compaction)
- Reads: Slower (must check multiple SSTables)
```

**LSM-Tree Compaction Strategies:**

```
1. Size-Tiered Compaction (Cassandra default):
   - Group SSTables of similar size
   - Merge when N tables of same size exist
   - WAF: 10-20×
   - Pro: Lower write amplification
   - Con: Higher space amplification (duplicate data across levels)

2. Leveled Compaction (RocksDB default):
   - Strict level hierarchy: L0 < L1 < L2 < L3
   - Merge into next level when level full
   - WAF: 20-30×
   - Pro: Lower space amplification (less duplication)
   - Con: Higher write amplification

3. Time-Window Compaction (time-series data):
   - Group by time window (e.g., 1 hour)
   - Compact within window, drop old windows
   - WAF: 2-5×
   - Pro: Very low WAF for time-series
   - Con: Only works for time-based data
```

---

## 4. Best Practices

### Practice 1: Index Only What You Query

**❌ Bad: Index Everything**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    date_of_birth DATE,
    phone VARCHAR(20),
    address VARCHAR(200),
    city VARCHAR(50),
    state VARCHAR(2),
    zip VARCHAR(10),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    last_login TIMESTAMP
);

-- Developer creates index on every column (14 indexes!)
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_first_name ON users(first_name);
CREATE INDEX idx_users_last_name ON users(last_name);
CREATE INDEX idx_users_dob ON users(date_of_birth);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_address ON users(address);
CREATE INDEX idx_users_city ON users(city);
CREATE INDEX idx_users_state ON users(state);
CREATE INDEX idx_users_zip ON users(zip);
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_users_updated ON users(updated_at);
CREATE INDEX idx_users_last_login ON users(last_login);

-- WAF calculation:
-- 1 table write + 14 index writes + 1 WAL write = 16 writes
-- WAF: 16×

-- Reality: Many indexes never used!
SELECT indexname, idx_scan FROM pg_stat_user_indexes WHERE tablename = 'users';

-- Result:
-- idx_users_first_name: 0 scans (NEVER USED!)
-- idx_users_address: 2 scans in 6 months
-- idx_users_zip: 15 scans
-- ... only 3 indexes actually used frequently
```

**✅ Good: Index Based on Query Patterns**
```sql
-- Audit actual queries (run for 1 week)
SELECT query, calls FROM pg_stat_statements 
WHERE query LIKE '%users%' 
ORDER BY calls DESC 
LIMIT 20;

-- Result: 95% of queries filter by:
-- 1. email (login)
-- 2. username (profile lookup)
-- 3. created_at (recent users report)

-- Create only necessary indexes
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_created ON users(created_at);

-- WAF: 1 table + 3 indexes + 1 WAL = 5 writes
-- WAF: 5× (vs 16× before)

-- INSERT throughput improved:
-- Before: 5K inserts/sec
-- After: 15K inserts/sec (3× faster!)
```

### Practice 2: Use Partial Indexes for Filtered Queries

**Reduce write amplification by indexing only relevant rows**

**❌ Bad: Full Index**
```sql
-- Index all 100M rows
CREATE INDEX idx_orders_status ON orders(status);

-- Status distribution:
-- - completed: 95M rows (95%)
-- - pending: 4M rows (4%)
-- - cancelled: 1M rows (1%)

-- Queries only filter by pending/cancelled (5% of data)
WHERE status = 'pending'   -- 95% of status queries
WHERE status = 'cancelled' -- 5% of status queries

-- Problem:
-- - Index includes 95M "completed" rows (never queried)
-- - Every INSERT of "completed" order updates index (wasted write)
-- - Index size: 1.2GB (95% wasted space)
```

**✅ Good: Partial Index**
```sql
-- Index only pending and cancelled orders
CREATE INDEX idx_orders_pending ON orders(status) WHERE status IN ('pending', 'cancelled');

-- Index size: 60MB (only 5M rows)
-- 95% size reduction!

-- Writes:
-- INSERT completed order: No index update (not in filter)
-- INSERT pending order: Index updated (needed)

-- WAF improvement:
-- Before: 95% of INSERTs update index (wasted)
-- After: 5% of INSERTs update index (only when needed)
-- WAF reduction: 1.9× improvement (from 10× to 5.3×)
```

### Practice 3: Batch Writes and Bulk Load

**Amortize write amplification across many rows**

**❌ Bad: Single-Row Inserts**
```sql
-- Insert 1M rows one at a time
FOR i IN 1..1000000 LOOP
    INSERT INTO events (user_id, event_type, timestamp)
    VALUES (i, 'pageview', NOW());
    COMMIT;  -- Commit each insert
END LOOP;

-- Overhead per INSERT:
-- - Transaction overhead: BEGIN + COMMIT
-- - Index rebalancing: Immediate for each insert
-- - WAL sync: fsync() per COMMIT (slow!)

-- Time: 2 hours
-- Physical writes: 10M writes (10× amplification)
```

**✅ Good: Batched Inserts**
```sql
-- Insert in batches of 1000
INSERT INTO events (user_id, event_type, timestamp)
VALUES
    (1, 'pageview', NOW()),
    (2, 'pageview', NOW()),
    ...
    (1000, 'pageview', NOW());
COMMIT;  -- One commit per 1000 rows

-- Benefits:
-- - Single transaction overhead per 1000 rows
-- - Index updates amortized (batch insertions more efficient)
-- - WAL sync once per batch (fsync expensive)

-- Time: 10 minutes (12× faster!)
-- Physical writes: 3M writes (3× amplification, vs 10× before)
```

**Best: COPY/LOAD DATA (Bulk Load)**
```sql
-- PostgreSQL: COPY
COPY events (user_id, event_type, timestamp)
FROM '/data/events.csv'
WITH (FORMAT csv);

-- MySQL: LOAD DATA
LOAD DATA INFILE '/data/events.csv'
INTO TABLE events
FIELDS TERMINATED BY ','
(user_id, event_type, timestamp);

-- Optimizations:
-- - Minimal transaction overhead
-- - Indexes updated in bulk (optimal page packing)
-- - WAL buffered (fewer fsyncs)

-- Time: 2 minutes (60× faster than single inserts!)
-- Physical writes: 1.5M writes (1.5× amplification)
```

### Practice 4: Drop Indexes During Bulk Import, Rebuild After

**Eliminate write amplification for large data loads**

```sql
-- Scenario: Load 1 billion rows into table with 10 indexes

-- ❌ Naive approach:
-- COPY data with indexes in place
-- Time: 12 hours
-- WAF: 15× (1 table + 10 indexes + WAL + rebalancing)

-- ✅ Optimized approach:
BEGIN;

-- Step 1: Drop all indexes except primary key
DROP INDEX idx_users_email;
DROP INDEX idx_users_city;
DROP INDEX idx_users_created;
... (drop all secondary indexes)

-- Step 2: Bulk load data
COPY users FROM '/data/users.csv';
-- Time: 1 hour (no index updates!)
-- WAF: 2× (table + WAL only)

-- Step 3: Rebuild indexes
CREATE INDEX idx_users_email ON users(email);       -- 30 min
CREATE INDEX idx_users_city ON users(city);         -- 25 min
CREATE INDEX idx_users_created ON users(created_at); -- 20 min
...
-- Total index rebuild time: 2 hours

COMMIT;

-- Total time: 3 hours (vs 12 hours)
-- Speedup: 4× faster!

-- Why faster:
-- - No incremental index updates during load (expensive rebalancing avoided)
-- - Indexes built from sorted data in one pass (optimal)
-- - Lower WAF during bulk load phase
```

---

## 5. Common Mistakes

### Mistake 1: Adding Indexes Without Measuring Impact

**Problem:**
```sql
-- Developer adds index to speed up one query
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- Query is now 10× faster (good!)
-- But...

-- Measure write impact:
-- Before index:
-- - INSERT throughput: 10K/sec
-- - INSERT latency: 5ms

-- After index:
-- - INSERT throughput: 8K/sec (20% slower!)
-- - INSERT latency: 7ms (40% slower!)
-- - Replication lag: 0s → 30s (replica can't keep up)

-- Trade-off:
-- - Read speedup: 10× (one query type)
-- - Write slowdown: 1.4× (all inserts)

-- Decision depends on workload:
-- - Read-heavy (90% reads): Worth it
-- - Write-heavy (50% writes): Might not be worth it
```

**Solution: Always Measure**
```sql
-- Before adding index:
-- 1. Benchmark current performance
SELECT
    NOW() AS baseline_time,
    pg_stat_get_tuples_inserted(oid) AS inserts_so_far
FROM pg_class
WHERE relname = 'orders';

-- 2. Add index
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- 3. Re-benchmark (after 1 hour)
SELECT
    NOW() AS after_index_time,
    pg_stat_get_tuples_inserted(oid) - baseline_inserts AS new_inserts,
    (NOW() - baseline_time) AS duration
FROM pg_class
WHERE relname = 'orders';

-- Calculate throughput change:
-- Before: 10K inserts / hour = 2.78 inserts/sec
-- After: 8K inserts / hour = 2.22 inserts/sec
-- Impact: -20% throughput

-- 4. Decide:
-- - If read benefit > write cost → Keep index
-- - If write cost too high → Drop index, optimize differently
```

### Mistake 2: Not Using Covering Indexes (Redundant Index Writes)

**Problem:**
```sql
-- Multiple indexes on same columns
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_email_created ON users(email, created_at);

-- Every INSERT updates BOTH indexes:
INSERT INTO users (email, created_at) VALUES ('user@example.com', NOW());

-- Write amplification:
-- - Both indexes contain email
-- - Redundant writes (idx_users_email unnecessary if idx_users_email_created exists)

-- WAF: 2 index writes (when 1 would suffice)
```

**Solution:**
```sql
-- Use covering index, drop single-column index
DROP INDEX idx_users_email;
-- Keep only: idx_users_email_created

-- Query: WHERE email = 'user@example.com'
-- Plan: Uses idx_users_email_created (email is leftmost column)

-- Query: WHERE email = 'user@example.com' AND created_at > '2024-01-01'
-- Plan: Uses idx_users_email_created (both columns in index)

-- WAF improvement: 2× → 1× (50% reduction in index writes)
```

### Mistake 3: Over-Indexing in Event Logging Systems

**Problem:**
```sql
-- Event logging table (1M events/sec)
CREATE TABLE events (
    event_id BIGSERIAL PRIMARY KEY,
    user_id INT,
    session_id VARCHAR(36),
    event_type VARCHAR(50),
    url TEXT,
    timestamp TIMESTAMPTZ,
    user_agent TEXT,
    ip_address INET,
    payload JSONB
);

-- Developer adds 15 indexes:
CREATE INDEX idx_events_user_id ON events(user_id);
CREATE INDEX idx_events_session_id ON events(session_id);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_url ON events(url);
CREATE INDEX idx_events_timestamp ON events(timestamp);
CREATE INDEX idx_events_ip ON events(ip_address);
CREATE INDEX idx_events_payload ON events USING GIN (payload);
... (8 more composite indexes)

-- WAF: 17× (1 table + 15 indexes + WAL)
-- Physical writes: 1M events/sec × 17 = 17M writes/sec
-- SSD lifespan: 5 years → 6 months (28× faster wear!)
```

**Solution: Time-Series Approach**
```sql
-- Strategy 1: Partition by time, minimal indexes
CREATE TABLE events (
    event_id BIGSERIAL,
    user_id INT,
    event_type VARCHAR(50),
    timestamp TIMESTAMPTZ,
    payload JSONB,
    PRIMARY KEY (timestamp, event_id)  -- Clustered by time
) PARTITION BY RANGE (timestamp);

-- Minimal indexes (only frequently queried columns)
CREATE INDEX idx_events_user_id ON events(user_id);
CREATE INDEX idx_events_type ON events(event_type);
-- Total: 3 indexes (vs 15)

-- WAF: 5× (vs 17×)

-- Strategy 2: Write to fast storage, ETL to warehouse
-- 1. INSERT to PostgreSQL with minimal indexes (fast)
-- 2. Hourly ETL to ClickHouse/BigQuery (analytics)
-- 3. Query analytics from warehouse (columnar, optimized for reads)

-- Benefits:
-- - PostgreSQL: Optimized for writes (low WAF)
-- - Warehouse: Optimized for reads (many indexes, materialized views)
-- - Best of both worlds!
```

### Mistake 4: Forgetting About Replication Amplification

**Problem:**
```sql
-- Production setup:
-- 1 primary + 3 read replicas

-- WAF on primary: 10×
-- Writes replicated to all 3 replicas
-- Effective WAF across cluster: 10× × 4 = 40×!

-- Scenario: 10K writes/sec on primary
-- Cluster-wide physical writes: 10K × 40 = 400K writes/sec
-- Total I/O: 400K × 8KB = 3.2GB/sec

-- Problem: Replicas can't keep up
-- Replication lag: 0s → 10 minutes → 1 hour
-- Eventually: Replicas fall too far behind, crash
```

**Solution:**
```sql
-- Option 1: Reduce indexes on replicas
-- Primary: Full indexes (write-optimized)
-- Replicas: Subset of indexes (read-optimized for specific queries)

-- This is NOT standard (most DBs replicate index structure)
-- Alternative: Use logical replication (selective table/column replication)

-- Option 2: Add more replicas with better hardware
-- If replication lag = 10min with 3 replicas
-- Add 2 more replicas with faster SSDs
-- Spread read load across 5 replicas
-- Replication lag: 10min → 2min

-- Option 3: Reduce indexes on primary (best solution)
-- Audit and drop 50% of indexes
-- WAF: 10× → 5×
-- Cluster writes: 400K → 100K writes/sec (4× reduction)
-- Replication lag: 1 hour → 0s ✅
```

---

## 6. Security Considerations

### 1. DoS via Write Amplification

**Attack:**
```sql
-- Attacker discovers table has 20 indexes
-- Sends many INSERT requests

-- Each INSERT:
-- - 1 row inserted (100 bytes)
-- - 20 index updates (8KB each)
-- - Total write: 160KB per 100-byte logical insert
-- - Amplification: 1600×!

-- Attacker sends 1K inserts/sec
-- Physical I/O: 1K × 160KB = 160MB/sec
-- Sustained for 10 minutes: 96GB written
-- SSD daily write budget: 100GB
-- SSD lifetime reduced: 10%/day → Failure in 10 days!
```

**Mitigation:**
```sql
-- 1. Rate limiting at application layer
-- Max 100 inserts/sec per user

-- 2. Connection pooling with limits
-- Max 1000 active connections

-- 3. Monitor write I/O
CREATE OR REPLACE FUNCTION check_write_iops()
RETURNS void AS $$
DECLARE
    writes_per_sec INT;
BEGIN
    SELECT pg_stat_get_blks_written(oid)
    FROM pg_class
    WHERE relname = 'high_traffic_table'
    INTO writes_per_sec;
    
    IF writes_per_sec > 10000 THEN
        -- Alert ops team
        PERFORM pg_notify('write_amplification_alert',
            'Excessive write I/O: ' || writes_per_sec || ' blocks/sec');
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Run every minute
SELECT cron.schedule('check_writes', '* * * * *', 'SELECT check_write_iops()');
```

---

## 7. Performance Optimization

### Optimization 1: Use LSM-Tree for Write-Heavy Workloads

**B-Tree (Write-Optimized SQL) vs LSM-Tree (Write-Heavy NoSQL)**

**Scenario: Time-series sensor data (100K writes/sec)**

```sql
-- PostgreSQL B-Tree:
CREATE TABLE sensor_data (
    sensor_id INT,
   timestamp TIMESTAMPTZ,
    value DECIMAL,
    PRIMARY KEY (sensor_id, timestamp)
);

-- Write pattern:
-- - Random sensor_id
-- - Monotonically increasing timestamp

-- WAF:
-- - Heap write: 1
-- - Primary key B-Tree: 2-3 (page splits frequent due to many sensors)
-- - WAL: 1
-- Total WAF: 4-5×

-- Performance:
-- - INSERT throughput: 20K/sec (bottleneck: random B-Tree updates)
-- - SSD writes: 100MB/sec

---

-- Cassandra LSM-Tree:
CREATE TABLE sensor_data (
    sensor_id INT,
    timestamp TIMESTAMP,
    value DECIMAL,
    PRIMARY KEY (sensor_id, timestamp)
);

-- Write pattern:
-- - Append to memtable (in-memory)
-- - Flush to SSTable (sequential write)
-- - Background compaction

-- WAF:
-- - Immediate: 2× (memtable + WAL, both sequential)
-- - Amortized: 10× (including compaction)

-- Performance:
-- - INSERT throughput: 100K/sec (5× faster!)
-- - SSD writes: 200MB/sec (but sequential, less wear)

-- Trade-off:
-- - Writes: Much faster (sequential)
-- - Reads: Slightly slower (must query multiple SSTables)
-- - For write-heavy workloads: LSM wins!
```

### Optimization 2: Use Clustered Index (InnoDB)

**Reduce write amplification by colocating data and primary key**

**PostgreSQL (Heap Table):**
```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    total DECIMAL
);

-- Structure:
-- - Heap table (random order)
-- - Primary key index (separate B-Tree)

-- INSERT:
-- 1. Write to heap (random location)
-- 2. Write to primary key index
-- WAF: 2×
```

**InnoDB (Clustered Index):**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    total DECIMAL
) ENGINE=InnoDB;

-- Structure:
-- - Data stored IN primary key B-Tree (clustered)
-- - No separate heap

-- INSERT:
-- 1. Write to primary key B-Tree (data included)
-- WAF: 1× (single write!)

-- Benefit:
-- - 50% write reduction vs heap table
-- - Better read performance (no heap lookup)

-- Caveat:
-- - Secondary indexes store primary key (larger indexes)
-- - Page splits expensive (move entire rows, not just pointers)
```

---

## 8. Examples

### Example 1: E-commerce Checkout Optimization

**Before: Over-Indexed Orders Table**

```sql
CREATE TABLE orders (
    order_id BIGSERIAL PRIMARY KEY,
    customer_id INT,
    total_amount DECIMAL,
    status VARCHAR(20),
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ,
    shipping_address TEXT,
    payment_method VARCHAR(50)
);

-- Indexes (15 total):
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at);
CREATE INDEX idx_orders_updated ON orders(updated_at);
CREATE INDEX idx_orders_total ON orders(total_amount);
CREATE INDEX idx_orders_payment ON orders(payment_method);
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
CREATE INDEX idx_orders_customer_created ON orders(customer_id, created_at);
CREATE INDEX idx_orders_status_created ON orders(status, created_at);
... (6 more composite indexes)

-- WAF: 1 table + 15 indexes + 1 WAL = 17×

-- Performance:
-- - INSERT throughput: 500 orders/sec
-- - Checkout page: CREATE ORDER takes 150ms
-- - Customer complaint: "Checkout is slow!"
```

**After: Strategic Index Reduction**

```sql
-- Audit index usage (1 week):
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE tablename = 'orders'
ORDER BY idx_scan DESC;

-- Results:
-- idx_orders_customer: 1.2M scans ✅ Keep
-- idx_orders_customer_created: 500K scans ✅ Keep
-- idx_orders_status_created: 200K scans ✅ Keep
-- idx_orders_created: 50K scans (covered by composites) ❌ Drop
-- idx_orders_updated: 100 scans ❌ Drop
-- idx_orders_total: 0 scans ❌ Drop
-- idx_orders_payment: 0 scans ❌ Drop
-- ... (7 more with idx_scan < 1000) ❌ Drop

-- Drop 10 unused indexes:
DROP INDEX idx_orders_total;
DROP INDEX idx_orders_payment;
DROP INDEX idx_orders_updated;
... (drop 7 more)

-- Keep only 5 most-used indexes:
-- 1. idx_orders_customer (single-column, frequently used alone)
-- 2. idx_orders_customer_created (customer order history)
-- 3. idx_orders_status_created (admin dashboard: recent orders by status)
-- 4. idx_orders_customer_status (customer orders by status)
-- 5. Primary key

-- WAF: 1 table + 5 indexes + 1 WAL = 7×

-- Performance after optimization:
-- - INSERT throughput: 500 → 1200 orders/sec (2.4× faster!)
-- - Checkout page: 150ms → 60ms (2.5× faster!)
-- - Customer satisfaction: Complaints dropped 80%

-- Side effects:
-- - Some admin queries slower (but admin queries only 1% of traffic)
-- - Total index size: 50GB → 15GB (70% disk space freed)
-- - Maintenance time: 6 hours/week → 2 hours/week
```

---

### Example 2: IoT Sensor Data Pipeline

**Challenge: 500K sensor writes/sec**

```sql
-- Initial design (PostgreSQL):
CREATE TABLE sensor_readings (
    sensor_id INT,
    timestamp TIMESTAMPTZ,
    temperature DECIMAL,
    humidity DECIMAL,
    pressure DECIMAL,
    PRIMARY KEY (sensor_id, timestamp)
);

CREATE INDEX idx_readings_timestamp ON sensor_readings(timestamp);
CREATE INDEX idx_readings_sensor ON sensor_readings(sensor_id);

-- WAF: 1 table + 3 indexes + 1 WAL = 5×
-- Physical writes: 500K × 5 = 2.5M writes/sec
-- SSD: 100K IOPS rated → Running at 2500% of capacity ❌
-- System unstable, frequent crashes
```

**Solution: Hybrid Architecture**

```sql
-- Phase 1: Fast ingestion (TimescaleDB, PostgreSQL extension)
CREATE TABLE sensor_readings (
    sensor_id INT,
    timestamp TIMESTAMPTZ,
    temperature DECIMAL,
    humidity DECIMAL,
    pressure DECIMAL
);

-- Convert to hypertable (automatic time partitioning)
SELECT create_hypertable('sensor_readings', 'timestamp');

-- Minimal indexes (only primary key, clustered by time)
-- WAF: 2× (table + WAL, no secondary indexes)

-- Performance:
-- - INSERT throughput: 500K/sec (achievable!)
-- - Physical writes: 1M writes/sec (within SSD capacity)
-- - Replication lag: 0s (fast enough)

-- Phase 2: Offline aggregation (hourly)
CREATE MATERIALIZED VIEW sensor_readings_hourly AS
SELECT
    sensor_id,
    time_bucket('1 hour', timestamp) AS hour,
    AVG(temperature) AS avg_temp,
    AVG(humidity) AS avg_humidity,
    AVG(pressure) AS avg_pressure
FROM sensor_readings
GROUP BY sensor_id, hour;

CREATE INDEX idx_hourly_sensor_hour ON sensor_readings_hourly(sensor_id, hour);

-- Queries now target aggregated table:
-- - 99% of queries: Last 24 hours (aggregated data)
-- - 1% of queries: Real-time (last 5 minutes, raw data)

-- Result:
-- - Write performance: 500K/sec sustained ✅
-- - Query performance: Sub-second for 99% of queries ✅
-- - Cost: $2K/month (vs $20K for over-provisioned OLTP database)
```

---

## 9. Real-World Use Cases

### Use Case 1: Twitter - Timeline Ingestion

**Challenge:**
- 500M tweets/day
- 6K tweets/sec average, 50K/sec peak
- Every tweet triggers:  
  - INSERT into tweets table
  - Update user stats (tweet_count++)
  - Fanout to follower timelines

**Write Amplification Problem:**
```
Single tweet by @celebrity (100M followers):
1. INSERT into tweets: 1 write
2. Update user stats: 1 write
3. Fanout to 100M timelines: 100M writes

Total: 100M+ writes for one tweet!
WAF: 100,000,000×!
```

**Solution: Deferred Fanout + Caching**
```sql
-- Phase 1: Fast ingestion (no indexes)
CREATE TABLE tweets_ingestion (
    tweet_id BIGINT,
    user_id BIGINT,
    content TEXT,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);
-- No secondary indexes (only partition key)
-- WAF: 2× (table + WAL)

-- Phase 2: Async fanout (background workers)
-- - Read from tweets_ingestion
-- - Compute timeline updates
-- - Batch write to timelines table
-- - Rate limited (don't overwhelm system)

-- Phase 3: Cache timelines (Redis)
-- - Most recent 1000 tweets per user cached
-- - 99% of timeline reads from cache
-- - Database only for historical tweets

-- Result:
-- - Ingestion: 50K tweets/sec (sustained)
-- - Fanout: Async, rate-limited (doesn't block ingestion)
-- - Read performance: < 10ms (cached)
```

### Use Case 2: Uber - Trip State Machine

**Challenge:**
- Trip goes through 12 states: requested → matched → pickup → trip → dropoff → payment → complete
- Each state transition = UPDATE trips table
- 20M trips/day = 240M updates/day
- Table has 25 indexes (various query patterns)

**Write Amplification:**
```sql
-- Every state transition:
UPDATE trips SET status = 'pickup', updated_at = NOW() WHERE trip_id = 12345;

-- Writes triggered:
-- 1. Heap table (update row)
-- 2. 25 index updates (status column indexed many ways)
-- 3. Old row version (MVCC, PostgreSQL)
-- 4. WAL entry
-- Total: 28 writes per UPDATE

-- 240M updates/day × 28 writes = 6.7 billion writes/day
-- 77K writes/sec average, 500K writes/sec peak
```

**Solution: Separate Hot and Cold Data**
```sql
-- trips_active: Only active trips (1% of data), minimal indexes
CREATE TABLE trips_active (
    trip_id BIGINT PRIMARY KEY,
    driver_id INT,
    rider_id INT,
    status VARCHAR(20),
    updated_at TIMESTAMP
);
-- Indexes: Only 3 (driver, rider, status)
-- WAF: 5× (1 table + 3 indexes + 1 WAL)

-- trips_archive: Completed trips (99% of data), many indexes
CREATE TABLE trips_archive (
    trip_id BIGINT PRIMARY KEY,
    driver_id INT,
    rider_id INT,
    status VARCHAR(20),
    created_at TIMESTAMP,
    completed_at TIMESTAMP,
    ... (50 more columns)
);
-- Indexes: 25 (all analytics queries)
-- But: No writes (read-only after archive)

-- Workflow:
-- 1. Trip starts: INSERT into trips_active (WAF: 5×)
-- 2. State transitions: UPDATE trips_active (WAF: 5×)
-- 3. Trip completes: 
--    - INSERT into trips_archive (WAF: 27×, but only once)
--    - DELETE from trips_active
-- 4. Analytics: Query trips_archive (cold data)

-- Result:
-- - Active writes: 240M × 5 = 1.2B writes/day (vs 6.7B before)
-- - 82% write reduction!
-- - Peak writes: 500K → 100K writes/sec
```

---

## 10. Interview Questions

### Q1: How would you detect write amplification in a production database?

**Senior Answer:**

```
**Detection Methods:**

**1. Monitor Physical vs Logical Writes:**

PostgreSQL:
```sql
-- Logical writes (rows inserted)
SELECT
    schemaname,
    relname,
    n_tup_ins AS rows_inserted,
    n_tup_upd AS rows_updated,
    n_tup_del AS rows_deleted
FROM pg_stat_user_tables
ORDER BY (n_tup_ins + n_tup_upd) DESC;

-- Physical writes (blocks written)
SELECT
    relname,
    heap_blks_hit,  -- Cache hits
    heap_blks_read,  -- Disk reads
    idx_blks_hit,    -- Index cache hits
    idx_blks_read    -- Index disk reads
FROM pg_statio_user_tables
ORDER BY (idx_blks_read + heap_blks_read) DESC;

-- Calculate WAF:
WAF = (Total blocks written) / (Rows inserted/updated)
```

SQL Server:
```sql
-- Use sys.dm_db_index_operational_stats
SELECT
    OBJECT_NAME(ios.object_id) AS table_name,
    i.name AS index_name,
    ios.leaf_insert_count,
    ios.leaf_update_count,
    ios.leaf_ghost_count,  -- Deleted rows
    ios.range_scan_count,
    ios.singleton_lookup_count,
    ios.page_latch_wait_count,  -- Contention indicator
    ios.page_io_latch_wait_count  -- I/O wait indicator
FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL) AS ios
JOIN sys.indexes AS i
    ON ios.object_id = i.object_id
    AND ios.index_id = i.index_id
WHERE ios.leaf_insert_count > 10000
ORDER BY (ios.leaf_insert_count + ios.leaf_update_count) DESC;
```

**2. Monitor Disk I/O:**
```bash
# iostat: Check write throughput
iostat -x 5  # 5-second intervals

# Look for:
# - w/s: writes per second (high = write amplification)
# - wMB/s: write megabytes per second
# - %util: disk utilization (>80% = saturation)

# Example suspicious output:
# Device: sda
# w/s: 15000  ← 15K writes/sec
# wMB/s: 500  ← 500MB/sec sustained
# %util: 95%  ← Disk saturated

# Calculate:
# Application writes: 1K rows/sec × 100 bytes = 100KB/sec
# Physical writes: 500MB/sec
# WAF: 500MB / 100KB = 5000× ❌ Extreme amplification!
```

**3. Monitor Replication Lag:**
```sql
-- PostgreSQL: Replication lag indicates write pressure
SELECT
    client_addr,
    application_name,
    state,
    sync_state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    NOW() - pg_last_xact_replay_timestamp() AS lag_time
FROM pg_stat_replication;

-- If lag_time grows → Replica can't keep up → Excessive writes
-- Example:
-- lag_time: 0s → 30s → 5min → 1 hour (growing!)
-- Root cause: Write amplification on primary overwhelming replica
```

**4. Check SSD Endurance:**
```bash
# SMART data shows SSD wear
sudo smartctl -a /dev/sda

# Look for:
# - Media_Wearout_Indicator: 100 (new) → 0 (worn out)
# - Total_LBAs_Written: Number of 512-byte sectors written

# Example:
# Total_LBAs_Written: 5,000,000,000,000 (2.5 PB)
# Time in service: 6 months
# Daily writes: 2.5 PB / 180 days = 14 TB/day

# If SSD rated for 1 TBW (terabyte written) per day:
# Actual: 14 TB/day → 14× over spec!
# Remaining lifespan: 5 years → 4 months
# Cause: Write amplification!
```

**5. Profile Query Patterns:**
```sql
-- PostgreSQL: Find queries causing most writes
SELECT
    query,
    calls,
    total_time,
    rows,
    100.0 * shared_blks_written / NULLIF(shared_blks_hit + shared_blks_read + shared_blks_written, 0) AS write_pct
FROM pg_stat_statements
WHERE shared_blks_written > 1000
ORDER BY shared_blks_written DESC
LIMIT 20;

-- Identify:
-- - INSERT-heavy queries
-- - Queries updating many rows
-- - Queries triggering many index updates
```

**Red Flags Checklist:**

✅ Table has > 15 indexes
✅ INSERT latency increasing over time (index bloat)
✅ Replication lag growing despite idle reads
✅ Disk write throughput >> application write rate
✅ SSD wearing out faster than expected (SMART data)
✅ Page split rate high (B-Tree fragmentation)
✅ WAL generation rate > 1GB/min

**Interview insight:**

"In a previous role, I detected severe write amplification by correlating three metrics: (1) Application INSERT rate was 5K/sec × 200 bytes = 1MB/sec, (2) iostat showed 800MB/sec physical writes, and (3) SSD SMART data showed daily writes 100× above spec. Investigation revealed 47 indexes on a single table, most unused. We dropped 40 indexes, wrote amplification dropped from 800× to 10×, and SSD lifespan went from 'weeks remaining' to '5+ years remaining'."
```

### Q2: Explain the trade-off between B-Tree and LSM-Tree in terms of write amplification.

**Staff Answer:**

```
**B-Tree Write Amplification:**

**Mechanism:**
- In-place updates: Modify existing pages on disk
- Page splits: When page full, split into two pages
- Random writes: Updates scattered across B-Tree

**Write Amplification Sources:**
1. **Page overhead:** Update 1 byte → Write 8KB page (8000× amp!)
2. **Page splits:** 1 insert → Split page → 3-5 page writes (cascading splits)
3. **WAL:** Every modification logged
4. **Index updates:** Each secondary index updated separately

**Formula:**
```
WAF_btree = (pages_written × page_size) / logical_data_size

Example:
- INSERT 100 bytes
- Page size: 8KB
- Page partially full → Write 1 page
- Update 3 indexes → Write 3 pages
- WAL → Write 1 KB
- Total: 4 × 8KB + 1KB = 33KB

WAF = 33KB / 100 bytes = 330×
```

**Best case:** No page splits, writesamortized → WAF ≈ 5-10×
**Worst case:** Many page splits, random writes → WAF ≈ 50-100×

---

**LSM-Tree Write Amplification:**

**Mechanism:**
- Append-only: Write to log, never modify existing data
- Memtable: In-memory writes (fast)
- Immutable SSTables: Periodic flush to disk
- Compaction: Background merging of SSTables

**Write Amplification Sources:**
1. **Initial write:** Append to WAL (sequential)
2. **Memtable flush:** Write to L0 SSTable (sequential)
3. **Compaction:** Merge L0 → L1, L1 → L2, ... (repeated reads + writes)

**Formula:**
```
WAF_lsm = 1 (initial) + sum(compaction_overhead per level)

Size-Tiered Compaction (Cassandra):
- L0 → L1: Read 10 SSTables + Write 1 merged SSTable = 2× amp (read+write)
- Occurs once per 10 writes
- Amortized WAF: 1 + (2 × 0.1) = 1.2× per level
- 5 levels → WAF ≈ 6-12×

Leveled Compaction (RocksDB):
- L0 → L1: Read L0 + overlapping L1, write merged L1 = 2×
- L1 → L2: Read L1 + all L2 (10× larger), write merged L2 = 2×
- ... cascade to L5
- WAF ≈ 20-30× (more compaction overhead)
```

**Best case:** Size-tiered, infrequent compaction → WAF ≈ 5-10×
**Worst case:** Leveled, frequent compaction → WAF ≈ 20-40×

---

**Comparison Table:**

| Factor | B-Tree | LSM-Tree |
|--------|--------|----------|
| **Write pattern** | Random (in-place) | Sequential (append) |
| **Immediate WAF** | 5-10× (index update) | 2× (memtable + WAL) |
| **Amortized WAF** | 10-50× (includes splits) | 10-30× (includes compaction) |
| **Write throughput** | 10K-50K writes/sec | 50K-500K writes/sec |
| **Read performance** | Fast (single B-Tree) | Slower (multiple SSTables) |
| **Space amplification** | Low (no duplication) | High (duplicate keys across levels) |
| **Use case** | OLTP, mixed workload | Write-heavy, time-series |

---

**Key Insights:**

**1. Short-term vs Long-term:**
- **LSM short-term:** 2× immediate WAF (very fast)
- **LSM long-term:** 10-30× after compaction (amortized over many writes)
- **B-Tree any time:** 10-50× consistent

**2. Write burst handling:**
- **LSM:** Absorbs bursts in memtable (no disk I/O), then catches up during compaction
- **B-Tree:** All writes go to disk immediately (slower burst handling)

**3. Sequential vs Random I/O:**
- **LSM:** Sequential writes (fast, SSD-friendly)
- **B-Tree:** Random writes (slow, causes fragmentation)

**4. Read performance trade-off:**
- **B-Tree:** O(log N) single tree traverse
- **LSM:** O(k log N) where k = number of SSTables per level

---

**Real-world Decision Framework:**

```
Choose LSM-Tree when:
- Write:Read ratio > 80:20
- Write throughput > 50K/sec
- Time-series data (append-mostly)
- Can tolerate eventual compaction overhead
- Examples: Metrics (Prometheus), Logs (Elasticsearch), Sensor data (Cassandra)

Choose B-Tree when:
- Write:Read ratio < 50:50
- Need low-latency reads (< 1ms)
- ACID transactions required
- Space efficiency important (no duplication)
- Examples: OLTP (PostgreSQL, MySQL), Financial systems, User databases

Use both (Hybrid):
- LSM for ingestion (fast writes)
- B-Tree for queries (fast reads)
- ETL from LSM → B-Tree periodically
- Example: Logs → Elasticsearch (LSM), Analytics → PostgreSQL (B-Tree)
```

**Interview insight:**

"I worked on a IoT telemetry system ingesting 200K sensor readings/sec. Initially PostgreSQL B-Tree, write amplification was 40×, causing 8M physical writes/sec and SSD saturation. We migrated to Cassandra LSM-Tree: immediate WAF dropped from 40× to 2× (20× improvement), write throughput increased to 500K/sec, and SSDs lasted 10× longer. Trade-off: Some dashboard queries were 2-3× slower (had to query multiple SSTables), but we pre-aggregated data to offset this. For write-heavy time-series, LSM was the right choice."
```

---

## 11. Summary

### Write Amplification Key Principles

1. **Every Index Has a Cost**
   - 1 index = 1× write amplification
   - 10 indexes = 10× write amplification
   - Measure before adding indexes

2. **WAF Components**
   - Table heap: 1×
   - Each index: 1-5× (depending on page splits)
   - WAL: 1×
   - Replication: × (number of replicas)

3. **Write-Heavy Workload Optimization**
   - Minimize indexes (only what you query)
   - Use partial indexes (filter condition)
   - Batch writes (amortize overhead)
   - Consider LSM-tree databases

4. **Monitoring Essential**
   - Track: logical writes vs physical writes
   - Monitor: replication lag growth
   - Check: SSD endurance (SMART data)
   - Alert: WAF > 20× for OLTP

### Decision Matrix

| Workload | Index Strategy | Expected WAF |
|----------|----------------|--------------|
| **Read-heavy (80% reads)** | 10-15 indexes | 15-20× acceptable |
| **Balanced (50/50)** | 5-8 indexes | 8-12× target |
| **Write-heavy (80% writes)** | 2-4 indexes | 4-6× critical |
| **Append-only (logs, metrics)** | 1-2 indexes or LSM | 2-5× optimal |

### Quick Reference: Reducing WAF

**Short-term fixes (hours):**
1. Drop unused indexes (idx_scan = 0)
2. Replace single-column indexes with covering indexes
3. Batch INSERT statements (1000 rows per batch)

**Medium-term (days/weeks):**
1. Audit query patterns, remove redundant indexes
2. Use partial indexes for filtered queries
3. Implement partitioning (reduce index size per partition)

**Long-term (months):**
1. Migrate write-heavy tables to LSM databases (Cassandra, RocksDB)
2. Separate hot (active) and cold (archived) data
3. Use materialized views for precomputed aggregates

### Commands

**Check Write Amplification (PostgreSQL):**
```sql
-- Logical writes
SELECT n_tup_ins + n_tup_upd AS logical_writes
FROM pg_stat_user_tables WHERE relname = 'tablename';

-- Physical writes
SELECT heap_blks_hit + idx_blks_hit AS blocks_written
FROM pg_statio_user_tables WHERE relname = 'tablename';

-- WAF = blocks_written * 8KB / (logical_writes * avg_row_size)
```

**SSD Endurance (Linux):**
```bash
sudo smartctl -a /dev/sda | grep -E 'Total_LBAs_Written|Media_Wearout'
```

### Key Takeaway

**"Write amplification is the hidden cost of indexes. One poorly chosen index can 10× your disk writes, wearing out SSDs years early and overwhelming replication. Measure WAF, audit indexes, and optimize for your workload."**

In production: Keep WAF < 10× for OLTP, < 5× for write-heavy workloads. Monitor SSD endurance monthly. Drop indexes aggressively—you can always add them back, but you can't restore SSD lifespan.

---

**Next Reading:**
- [10_README.md](10_README.md) - Section overview and index selection decision tree
- [08_Index_Maintenance.md](08_Index_Maintenance.md) - Keeping indexes healthy
- [../07_Transactions_Concurrency/03_Locking_Mechanisms.md](../07_Transactions_Concurrency/03_Locking_Mechanisms.md) - How writes lock resources
