# Transaction Logs - Durability and Recovery

## 1. Concept Explanation

**Transaction logs** (Write-Ahead Logs, WAL) record all changes before they're applied to the database. They ensure durability (the "D" in ACID) and enable crash recovery.

**Core Principle: Write-Ahead Logging (WAL)**
```
Rule: Log changes BEFORE modifying data

Sequence:
1. BEGIN transaction
2. Write change to log (disk, durable)
3. Modify data in memory (fast, volatile)
4. COMMIT: Flush log to disk
5. Return success to client
6. Background: Write data pages to disk (lazy)

Why: If crash occurs between steps 4-6:
- Log contains committed changes ✓
- Data pages incomplete ✗
- Recovery: Replay log → Restore data ✓
```

**Analogy: Chef's Recipe Log**
```
Chef (Database) prepares dish (data):
1. Write recipe step in logbook (WAL): "Add 2 eggs"
2. Add eggs to bowl (memory buffer)
3. Mark recipe step done (COMMIT)
4. Serve dish (return to client)

If kitchen fire between steps 3-4:
- Logbook intact (survived fire)
- Bowl destroyed
- Recovery: Read logbook, remake dish from scratch
```

### Log Types

**1. Redo Log (Forward Recovery):**
```
Purpose: Replay committed transactions after crash

Content: After-images
Example: "SET balance = 900 WHERE id = 123"

Recovery:
- Read redo log from last checkpoint
- Apply all changes sequentially
- Database restored to pre-crash state
```

**2. Undo Log (Rollback):**
```
Purpose: Rollback uncommitted transactions

Content: Before-images
Example: "balance was 1000" (before UPDATE to 900)

Recovery:
- Find uncommitted transactions (no COMMIT record)
- Apply undo log in reverse order
- Restore original state
```

**3. Combined (PostgreSQL WAL):**
```
WAL record contains both:
- Old value (for undo)
- New value (for redo)

Example:
LSN: 1/2A3B4C5D
TXN: 12345
Type: UPDATE
Table: accounts
Row: id=123
Old: balance=1000
New: balance=900
```

---

## 2. Why It Matters

### Production Impact: Data Loss Prevented

**Without WAL (Crash During Write):**
```
10:00:00 - Client: "Transfer $1000 from Account A to B"

10:00:01 - Database:
    Deduct $1000 from Account A → Memory buffer (not on disk)
    Credit $1000 to Account B → Memory buffer (not on disk)
    Return "Success" to client

10:00:02 - Power failure! (Server crash)

10:00:05 - Database restarts
    Memory lost!
    Account A: $1000 deducted? Unknown!
    Account B: $1000 credited? Unknown!
    Result: Data inconsistency (money disappeared or doubled)

Customer: "I transferred $1000 but it's gone!"
Bank: No audit trail, cannot recover
```

**With WAL (Crash Recovery):**
```
10:00:00 - Client: "Transfer $1000 from Account A to B"

10:00:01 - Database:
    Write to WAL (disk): "TXN-100: Deduct $1000 from A" ✓
    Write to WAL (disk): "TXN-100: Credit $1000 to B" ✓
    Write to WAL (disk): "TXN-100: COMMIT" ✓
    Modify memory buffer: A -= 1000, B += 1000
    Return "Success" to client

10:00:02 - Power failure! (Server crash)
    WAL on disk intact ✓
    Memory buffers lost ✗

10:00:05 - Database crash recovery:
    1. Read WAL from beginning
    2. Find TXN-100 with COMMIT record
    3. Replay: A -= 1000, B += 1000
    4. Database fully recovered ✓

    Account A: -$1000 ✓
    Account B: +$1000 ✓
    Consistency maintained!

Customer: Happy, transfer successful ✓
```

### Real Incident: Cloud Provider Outage

```
Date: 2019-08-23
Company: Major cloud provider (AWS us-east-1)
Issue: Power loss during maintenance

Timeline:
08:15 - Planned maintenance begins
        Migrate VMs to backup power

08:17 - Primary power fails unexpectedly
        Backup generator delay: 3 seconds gap

08:17:01 to 08:17:03 - Power outage (3 seconds)
        1000+ database instances lose power mid-transaction

08:17:04 - Power restored
        Databases begin crash recovery

08:17:04 to 08:45:00 - Recovery phase (28 minutes)
Instance outcomes:
- WAL enabled (95%): Automatic recovery, 0 data loss ✓
- WAL disabled (5%): Manual recovery, data loss, corrupted indexes

One customer scenario (WAL disabled):
- E-commerce database
- 10K orders placed 08:00-08:17
- WAL disabled (performance optimization)
- Crash recovery: Last checkpoint 07:00 (1 hour old!)
- Data loss: 10K orders lost
- Impact: Customers charged (payment gateway) but no order record
- Resolution: 48 hours manual reconciliation, $2M refunds

Outcome:
- AWS policy change: WAL mandatory for RDS instances
- Lesson: WAL overhead (< 5% performance) worth 100% durability
```

---

## 3. Internal Working

### PostgreSQL WAL Architecture

**WAL Buffer Flow:**
```
Client                    PostgreSQL Server                 Disk
  |                             |                              |
  | INSERT INTO ...             |                              |
  |---------------------------->|                              |
  |                             |                              |
  |                         1. Write WAL record               |
  |                            to WAL buffer (memory)          |
  |                             |                              |
  |                         2. Modify data page                |
  |                            in shared buffers (memory)      |
  |                             |                              |
  |                         3. COMMIT received                 |
  |                             |                              |
  |                         4. Flush WAL buffer                |
  |                             |----------------------------->|
  |                             |        fsync() WAL file      |
  |                             |<-----------------------------|
  |                             |        (durable!)            |
  |                         5. Return "Success"                |
  |<----------------------------|                              |
  |                             |                              |
  |                         6. [Background]                    |
  |                            Checkpoint process              |
  |                            flushes dirty pages             |
  |                             |----------------------------->|
  |                             |        Write data pages      |
  |                                                            |
```

**WAL File Structure:**
```
/var/lib/postgresql/data/pg_wal/
├── 000000010000000000000001  (WAL segment, 16MB)
├── 000000010000000000000002
├── 000000010000000000000003
└── ...

Each WAL segment:
- Size: 16MB (configurable)
- Contains: Multiple WAL records
- Recycled: After checkpoint, old segments archived or deleted
```

**WAL Record Format:**
```c
// Simplified structure
struct XLogRecord {
    uint32 xl_tot_len;      // Total length of record
    TransactionId xl_xid;   // Transaction ID
    uint8 xl_info;          // Record type (INSERT, UPDATE, DELETE, COMMIT)
    uint8 xl_rmid;          // Resource manager ID (heap, btree, etc.)
    XLogRecPtr xl_prev;     // Previous record LSN (linked list)
    
    // Variable-length data
    char data[];            // Actual change (before/after images)
};

// Example WAL record
{
    xl_tot_len: 128,
    xl_xid: 12345,
    xl_info: XLOG_HEAP_UPDATE,
    xl_rmid: RM_HEAP_ID,
    xl_prev: 0/1A2B3C4D,
    data: {
        table_oid: 16384,     // accounts table
        block: 100,           // Page number
        offset: 5,            // Tuple offset
        old_tuple: {balance: 1000},
        new_tuple: {balance: 900}
    }
}
```

### Crash Recovery Process

**Phases:**

**1. REDO (Forward Recovery):**
```
Goal: Replay all committed transactions

Algorithm:
1. Find last checkpoint record in WAL
   - Checkpoint contains: Last LSN, dirty page list
   
2. Scan WAL from checkpoint to end
   - Build list of all transactions
   - Mark transactions with COMMIT record as "committed"
   - Mark transactions without COMMIT as "aborted"

3. Replay committed transactions
   - Apply each WAL record:
     - Read table/page from disk
     - Apply change (old_value → new_value)
     - Write back to disk
   
4. Database state: All committed transactions restored

Example:
WAL:
  LSN 0/100: BEGIN TXN-100
  LSN 0/101: UPDATE accounts SET balance=900 WHERE id=123
  LSN 0/102: COMMIT TXN-100
  LSN 0/103: BEGIN TXN-101
  LSN 0/104: INSERT INTO orders VALUES (...)
  [CRASH - no COMMIT for TXN-101]

Redo phase:
  - Replay LSN 0/101 (TXN-100 committed) ✓
  - Skip LSN 0/104 (TXN-101 not committed) ✗
```

**2. UNDO (Rollback Uncommitted):**
```
Goal: Rollback incomplete transactions

Algorithm:
1. Identify uncommitted transactions (no COMMIT record)

2. For each uncommitted transaction:
   - Scan WAL in REVERSE order
   - Apply undo operations:
     - UPDATE: Restore old value
     - INSERT: Delete row
     - DELETE: Re-insert row

3. Database state: Clean (only committed transactions)

Example:
TXN-101 (uncommitted):
  LSN 0/104: INSERT INTO orders (id, amount) VALUES (500, 100);
  LSN 0/105: UPDATE inventory SET stock=10 WHERE id=50;

Undo phase:
  - Reverse LSN 0/105: UPDATE inventory SET stock=11 WHERE id=50;
  - Reverse LSN 0/104: DELETE FROM orders WHERE id=500;

Result: TXN-101 fully rolled back
```

### Checkpoint Process

**Purpose:** Reduce recovery time by flushing dirty pages to disk

**Without Checkpoint:**
```
WAL grows indefinitely:
- Crash recovery: Scan entire WAL (hours!)
- 10GB WAL × 100 MB/s read = 100 seconds
```

**With Checkpoint:**
```
Checkpoint every 5 minutes:
1. Write all dirty pages to disk
2. Write checkpoint record to WAL: "All changes before LSN 0/1A2B3C4D are on disk"
3. Flush WAL

Crash recovery:
- Start from last checkpoint (LSN 0/1A2B3C4D)
- Replay only 5 minutes of WAL (seconds!)
```

**PostgreSQL Checkpoint Triggers:**
```sql
SHOW checkpoint_timeout;           -- Default: 5 minutes
SHOW max_wal_size;                 -- Default: 1GB

-- Checkpoint occurs when:
-- 1. checkpoint_timeout elapsed (5 min), OR
-- 2. pg_wal/ size exceeds max_wal_size (1GB), OR
-- 3. Manual: CHECKPOINT;
```

**Checkpoint Process:**
```
1. Scan shared_buffers for dirty pages
   - Dirty: Modified in memory, not yet on disk

2. Write dirty pages to disk (sorted by file/offset)
   - Batch writes for efficiency

3. fsync() data files
   - Ensure durability

4. Write checkpoint record to WAL
   - LSN: 0/3A4B5C6D
   - Redo pointer: Start recovery from here

5. Update control file
   - Last checkpoint LSN

6. Archive/recycle old WAL segments
   - Segments before checkpoint no longer needed
```

---

## 4. Best Practices

### Practice 1: Tune for durability vs. performance trade-off

**Maximum Durability (Financial systems):**
```sql
-- Force WAL flush on every COMMIT
ALTER SYSTEM SET synchronous_commit = 'on';  -- Default

-- WAL flush method
ALTER SYSTEM SET wal_sync_method = 'fdatasync';  -- Linux: fsync data only (fast)

-- Group commit delay (batch commits)
ALTER SYSTEM SET commit_delay = 0;  -- No batching (immediate durability)

-- Result:
-- - Every COMMIT waits for fsync() (2-5ms latency)
-- - Throughput: 200-500 commits/second
-- - Durability: 100% (zero data loss on crash)
```

**Balanced (Most applications):**
```sql
ALTER SYSTEM SET synchronous_commit = 'on';
ALTER SYSTEM SET commit_delay = 100;  -- 100 microsecond delay (group commits)
ALTER SYSTEM SET wal_writer_delay = 200ms;  -- Background WAL writer

-- Result:
-- - Batch 10-20 commits into single fsync()
-- - Throughput: 5K-10K commits/second
-- - Durability: 100% (slight latency increase: 100μs)
```

**High Performance (Analytics, acceptable data loss):**
```sql
ALTER SYSTEM SET synchronous_commit = 'off';
-- Do NOT wait for fsync(), return immediately
-- WAL still written (eventually), but not flushed on COMMIT

-- Risk: Crash within `wal_writer_delay` (200ms) → Last commits lost

-- Result:
-- - Throughput: 50K-100K commits/second (10× improvement)
-- - Durability: 99.9% (200ms window of data loss on crash)
```

### Practice 2: Monitor WAL Generation Rate

**Check WAL Production:**
```sql
-- WAL generation per second
SELECT
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')) AS wal_generated_total,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 
        EXTRACT(EPOCH FROM (now() - pg_postmaster_start_time()))
    ) AS wal_per_second;

-- Example output:
-- wal_generated_total: 50 GB
-- wal_per_second: 10 MB/s

-- Interpretation:
-- - 10 MB/s: Normal for moderate write load
-- - 100 MB/s: High (check for bulk operations)
-- - 1 GB/s: Very high (archiving may fall behind)
```

**Alert on WAL Accumulation:**
```sql
-- Check pg_wal directory size
SELECT
    pg_size_pretty(SUM(size)) AS wal_size
FROM pg_ls_waldir();

-- Alert if > 10GB:
-- - Archiving failing?
-- - Replication lag?
-- - Checkpoint not occurring?
```

---

## 5. Common Mistakes

### Mistake 1: Disabling WAL for "Performance"

**Temptation:**
```sql
-- Disable WAL for bulk operations
ALTER TABLE large_table SET (autovacuum_enabled = false);
-- Drop indexes
-- COPY data
-- Recreate indexes

-- Or worse:
initdb --no-sync  -- NEVER do this in production!
```

**Reality Check:**
```
Performance gain: 20-30% faster writes
Risk: 100% data loss on crash

Example disaster:
- Bulk import: 1M rows (10 minutes)
- Crash at minute 9
- Without WAL: 0 rows imported (all lost!)
- With WAL: 900K rows imported (10% lost, recoverable)

Verdict: 30% speed boost NOT worth 100% data loss risk
```

**Alternative: Unlogged Tables (Acceptable for temp data):**
```sql
-- Unlogged table: No WAL
CREATE UNLOGGED TABLE temp_staging (...);

-- Use case: Temporary computations, discardable on crash
-- NOT for: Customer data, orders, transactions

-- After processing: Copy to logged table
INSERT INTO permanent_table SELECT * FROM temp_staging;
```

### Mistake 2: Small `max_wal_size` (Frequent Checkpoints)

**Problem:**
```sql
ALTER SYSTEM SET max_wal_size = '100MB';  -- Too small!

-- Result:
-- - Checkpoint every 10 seconds (at 10 MB/s WAL rate)
-- - CPU/disk overhead: Constant dirty page flushes
-- - Query performance: I/O spikes during checkpoint

-- Symptoms:
-- - Query latency spikes every 10 seconds
-- - Disk write peaks (100MB burst writes)
```

**Fix:**
```sql
-- Increase max_wal_size (allow checkpoint to spread out)
ALTER SYSTEM SET max_wal_size = '2GB';  -- Checkpoint every 3-5 minutes

-- Result:
-- - Fewer checkpoints (lower overhead)
-- - Smooth I/O (distributed over 5 minutes)
-- - Trade-off: Longer crash recovery (replay 2GB vs 100MB)
```

---

## 6. Security Considerations

### 1. WAL Contains Sensitive Data

**Problem:**
```sql
-- WAL records contain full row data
UPDATE users SET password_hash = 'new_hash', ssn = '123-45-6789' WHERE id = 123;

-- WAL record:
-- old_tuple: {password_hash: 'old_hash', ssn: '123-45-6789'}
-- new_tuple: {password_hash: 'new_hash', ssn: '123-45-6789'}

-- WAL archived to backup system:
-- /backups/wal_archive/000000010000000000000042

-- Risk: Backup system compromised → Sensitive data exposed
```

**Mitigation:**
```sql
-- Encrypt WAL files at rest
-- Option 1: File system encryption (LUKS, dm-crypt)
cryptsetup luksFormat /dev/sdb1
cryptsetup open /dev/sdb1 encrypted_wal
mkfs.ext4 /dev/mapper/encrypted_wal
mount /dev/mapper/encrypted_wal /var/lib/postgresql/data/pg_wal

-- Option 2: PostgreSQL Transparent Data Encryption (TDE) extension
-- (Enterprise feature, not in core PostgreSQL)

-- Option 3: Encrypt WAL archives
archive_command = 'openssl enc -aes-256-cbc -in %p -out /backups/%f.enc -pass file:/etc/postgres/key'
```

---

## 7. Performance Optimization

### Optimization 1: WAL on Separate Disk

**Problem:**
```
Data and WAL on same disk:
- Sequential writes (data pages)
- Sequential writes (WAL)
- Disk head thrashing (seek between data and WAL)
- Throughput: 100 MB/s (vs 500 MB/s disk capability)
```

**Solution:**
```bash
# Dedicated disk for WAL
mkfs.ext4 /dev/sdb1
mount /dev/sdb1 /var/lib/postgresql/wal

# Move pg_wal directory
mv /var/lib/postgresql/data/pg_wal /var/lib/postgresql/wal/
ln -s /var/lib/postgresql/wal /var/lib/postgresql/data/pg_wal

# Result:
# - Data disk: Sequential writes only
# - WAL disk: Sequential writes only
# - No seek contention
# - Throughput: Both disks at 500 MB/s (combined 1 GB/s)
```

### Optimization 2: Increase WAL Buffers

**Default:**
```sql
SHOW wal_buffers;  -- Default: -1 (auto, 1/32 of shared_buffers)

-- Example:
-- shared_buffers = 8GB → wal_buffers = 256MB
```

**Tuning for High Write Load:**
```sql
-- Increase WAL buffers for high-throughput writes
ALTER SYSTEM SET wal_buffers = '64MB';  -- More space for batching

-- Or very high write load:
ALTER SYSTEM SET wal_buffers = '256MB';

-- Trade-off:
-- - Larger buffer → More commits batched → Fewer fsync() calls
-- - Larger buffer → More memory usage (typically insignificant)

-- Reload config
SELECT pg_reload_conf();

-- Monitor effectiveness:
SELECT * FROM pg_stat_wal;
-- If wal_buffers_full > 0 → Increase wal_buffers
```

---

## 8. Examples

### Example 1: Manual Checkpoint and Monitoring

```sql
-- Trigger manual checkpoint
CHECKPOINT;

-- Monitor checkpoint statistics
SELECT
    checkpoints_timed,      -- Scheduled checkpoints (every checkpoint_timeout)
    checkpoints_req,        -- Requested checkpoints (WAL size exceeded)
    checkpoint_write_time,  -- Time spent writing dirty pages (ms)
    checkpoint_sync_time,   -- Time spent fsyncing (ms)
    buffers_checkpoint,     -- Buffers written during checkpoint
    buffers_clean,          -- Buffers written by background writer
    maxwritten_clean,       -- Background writer stopped due to max writes
    buffers_backend         -- Buffers written directly by backends (bad!)
FROM pg_stat_bgwriter;

-- Interpretation:
-- checkpoints_req >> checkpoints_timed: Increase max_wal_size
-- buffers_backend > 1% of buffers_checkpoint: Increase shared_buffers or bgwriter activity
-- checkpoint_write_time > 30s: Slow disks, or too many dirty pages
```

---

## 9. Real-World Use Cases

### Use Case: Point-in-Time Recovery (PITR)

**Scenario:** Accidental DELETE at 14:30, need to restore to 14:29

```bash
# Setup: WAL archiving enabled
archive_mode = on
archive_command = 'cp %p /mnt/wal_archive/%f'

# Disaster: Accidental DELETE
psql -c "DELETE FROM orders WHERE date = '2024-01-15';"  -- Oops! Wrong date!
-- Deleted 100K orders placed on 2024-01-15

# Recovery steps:
# 1. Stop database
systemctl stop postgresql

# 2. Restore from base backup (taken last night)
rm -rf /var/lib/postgresql/data
cp -r /mnt/base_backup/2024-01-15_00:00 /var/lib/postgresql/data

# 3. Create recovery configuration
cat > /var/lib/postgresql/data/recovery.conf <<EOF
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:29:00'  # 1 minute before disaster
EOF

# 4. Start database in recovery mode
systemctl start postgresql

# Recovery process:
# - Reads base backup (state at 00:00)
# - Replays WAL archives from 00:00 to 14:29
# - Stops at 14:29 (before DELETE)
# - Database restored to 14:29:00 ✓

# 5. Verify data
psql -c "SELECT COUNT(*) FROM orders WHERE date = '2024-01-15';"
# Result: 100,000 (orders restored!)

# 6. Promote to primary (exit recovery mode)
pg_ctl promote

# Result:
# - Data loss: 1 minute (14:29:00 to 14:30:00)
# - Orders placed 14:29-14:30 lost (acceptable)
# - 100K orders saved!
```

---

## 10. Interview Questions

### Q1: Explain the Write-Ahead Logging protocol and why it ensures durability.

**Answer:**

**Write-Ahead Logging (WAL) Protocol:**

**Steps:**
1. **Log first:** Write change to WAL (on disk, durable)
2. **Modify second:** Update data pages (in memory, volatile)
3. **Commit:** Flush WAL to disk (fsync())
4. **Return success:** Transaction committed
5. **Background:** Write data pages to disk (asynchronous)

**Why Durability:**

If crash occurs between steps 4-5:
- **WAL intact:** All committed changes recorded on disk ✓
- **Data pages lost:** Memory buffers cleared ✗
- **Recovery:** Replay WAL → Restore data pages ✓

**Example:**
```sql
-- Step 1: Write to WAL (disk)
-- LSN 0/100: UPDATE accounts SET balance=900 WHERE id=123
-- LSN 0/101: COMMIT TXN-100

-- Step 2-4: Modify memory, return success
-- Step 5: [Crash before writing data page to disk]

-- Recovery:
-- - Read WAL LSN 0/100: Apply UPDATE
-- - Read WAL LSN 0/101: COMMIT found, keep changes
-- - Result: balance=900 restored ✓
```

**Key insight:** "WAL ensures durability by separating commit (log flush) from data write (background). Crash can never lose committed transactions because log is durable first. In production, I've seen WAL replay recover 100% of committed data after power failures—it's the foundation of database reliability."

---

## 11. Summary

**Transaction Log Fundamentals:**
- **Write-Ahead Logging (WAL):** Log changes before applying
- **Redo log:** Replay committed transactions
- **Undo log:** Rollback uncommitted transactions
- **Checkpoint:** Flush dirty pages, mark recovery point

**Crash Recovery:**
```
1. Find last checkpoint
2. REDO: Replay committed transactions
3. UNDO: Rollback uncommitted transactions
4. Database restored ✓
```

**Best Practices:**
- **Durability setting:**
  - Financial: `synchronous_commit = on` (200-500 TPS)
  - Normal: `synchronous_commit = on, commit_delay = 100μs` (5K-10K TPS)
  - Analytics: `synchronous_commit = off` (50K-100K TPS, 200ms data loss risk)
- **Monitor WAL rate:** Alert if > 100 MB/s
- **Tune `max_wal_size`:** 1-2GB (checkpoint every 3-5 min)
- **WAL on separate disk:** Avoid seek contention

**Critical Pattern:**
```sql
-- Check WAL health
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')) AS wal_total;
SELECT * FROM pg_stat_wal;  -- WAL activity
SELECT * FROM pg_stat_bgwriter;  -- Checkpoint stats
```

**Key Takeaway:** "WAL is the foundation of durability—every ACID database uses it. The 'write-ahead' guarantee (log before data) ensures crash recovery always succeeds. In production: Never disable WAL for 'performance,' tune `synchronous_commit` based on durability requirements, and monitor checkpoint frequency."

---

**Next:** [06_Two_Phase_Commit.md](06_Two_Phase_Commit.md) - Distributed transactions, XA protocol

