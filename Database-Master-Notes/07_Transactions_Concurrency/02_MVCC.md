# MVCC - Multi-Version Concurrency Control

## 1. Concept Explanation

**Multi-Version Concurrency Control (MVCC)** is a concurrency control method that allows multiple transactions to access the same data simultaneously without blocking each other. Instead of locking rows, MVCC creates multiple "versions" of each row—each transaction sees a consistent snapshot of the database as it existed when the transaction started.

Think of MVCC like viewing historical documents:
```
Database Row (account balance):
- Version 1 (time 10:00): balance = $1000
- Version 2 (time 10:05): balance = $900  (after $100 withdrawal)
- Version 3 (time 10:10): balance = $1100 (after $200 deposit)

Transaction A (started 10:03): Sees Version 1 ($1000)
Transaction B (started 10:07): Sees Version 2 ($900)
Transaction C (started 10:12): Sees Version 3 ($1100)

Each transaction sees a consistent snapshot—no locks needed for reads!
```

### Traditional Locking vs MVCC

**Traditional Locking (MySQL MyISAM, old databases):**
```
Writer blocks readers:
- Transaction A: UPDATE accounts SET balance = 900 WHERE id = 123;
  → Acquires WRITE LOCK on row
- Transaction B: SELECT balance FROM accounts WHERE id = 123;
  → BLOCKED! Must wait for A to finish
  
Result: Poor concurrency (readers wait for writers)
```

**MVCC (PostgreSQL, MySQL InnoDB, Oracle):**
```
Writer doesn't block readers:
- Transaction A: UPDATE accounts SET balance = 900 WHERE id = 123;
  → Creates NEW VERSION (old version kept)
- Transaction B: SELECT balance FROM accounts WHERE id = 123;
  → Reads OLD VERSION (not blocked!)
  
Result: Excellent concurrency (readers never wait for writers)
```

### How MVCC Works (Simplified)

```
Row Structure in MVCC:

+----------------+------------------+---------------+
| Transaction ID | Data             | Visibility    |
+----------------+------------------+---------------+
| TXN-100        | balance = $1000  | Valid for TXN < 101 |
| TXN-105        | balance = $900   | Valid for TXN >= 105 |
| TXN-110        | balance = $1100  | Valid for TXN >= 110 |
+----------------+------------------+---------------+

When Transaction (TXN-107) reads:
1. Check row versions
2. Find version valid for TXN-107 (version created by TXN-105)
3. Return balance = $900
4. Never blocked by ongoing writes!
```

---

## 2. Why It Matters

### Production Impact: Blocking vs Non-Blocking Reads

**Scenario: High-Traffic E-commerce Site**

**Without MVCC (Traditional Locking):**
```sql
-- Session 1 (checkout process)
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 123;
-- Takes 5 seconds to process payment
COMMIT;

-- Session 2, 3, 4... (product page views)
SELECT * FROM products WHERE id = 123;
-- BLOCKED for 5 seconds! Page won't load!

-- Result:
-- - Product page: 5-second delay
-- - User frustrated, abandons site
-- - Revenue lost: ~$500/hour (slow pages = lost sales)
```

**With MVCC (PostgreSQL/InnoDB):**
```sql
-- Session 1 (checkout process)
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 123;
-- Creates new version: stock = 9
-- Takes 5 seconds to process payment
COMMIT;

-- Session 2, 3, 4... (product page views)
SELECT * FROM products WHERE id = 123;
-- NOT BLOCKED! Reads old version: stock = 10
-- Returns immediately (< 10ms)

-- Result:
-- - Product page: Fast (10ms response time)
-- - Users happy, continue browsing
-- - Revenue maintained ✓
```

### Real Incident: Database Deadlock Cascade

```
Company: SaaS analytics platform
Issue: Reporting queries block customer writes

Without MVCC (MySQL MyISAM):
- 10:00 AM: Admin runs long report (SELECT * FROM events... takes 10 minutes)
  → Acquires READ LOCK on 100M rows
- 10:01 AM: Customer tries INSERT (new event)
  → BLOCKED by report (waiting for READ LOCK release)
- 10:02 AM: More customers INSERT
  → BLOCKED (queue builds: 1000 waiting)
- 10:05 AM: Connection pool exhausted (500 connections waiting)
- 10:06 AM: Application crashes (no available connections)
- 10:10 AM: Report finishes, locks released
  → 1000 queued inserts execute simultaneously
  → Database CPU: 100% (overload)
  → Cascade failure: takes 2 hours to recover

Cost:
- 2 hours downtime
- $200K revenue lost
- 5000 customer complaints
- Regulatory fines (SLA breach)

Fix: Migrate to PostgreSQL (MVCC)
- Admin reports: READ old versions (no locks!)
- Customer INSERTs: Never blocked
- Throughput: 10K writes/sec (vs 100 writes/sec before)
- Downtime: 0 hours ✓
```

---

## 3. Internal Working

### PostgreSQL MVCC Implementation

**Tuple Structure (Row Versioning):**

Every row in PostgreSQL contains hidden system columns:
```
CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL
);

-- Actual storage (with hidden columns):
+--------+---------+---------+---------+-----------+
| xmin   | xmax    | id      | balance | ctid      |
+--------+---------+---------+---------+-----------+
| 100    | 0       | 123     | 1000    | (0,1)     |
+--------+---------+---------+---------+-----------+

xmin: Transaction ID that created this version
xmax: Transaction ID that deleted/updated this version (0 = still valid)
ctid: Physical location of row version
```

**UPDATE Operation (Creates New Version):**

```sql
-- Initial state (TXN-100)
accounts: xmin=100, xmax=0, balance=1000

-- Transaction 105: UPDATE
BEGIN;  -- TXN-105
UPDATE accounts SET balance = 900 WHERE id = 123;

-- MVCC doesn't modify existing row!
-- Instead: Creates NEW version, marks old version as deleted

-- After UPDATE (before COMMIT):
Old version: xmin=100, xmax=105, balance=1000  -- Marked deleted by TXN-105
New version: xmin=105, xmax=0,   balance=900   -- Created by TXN-105

COMMIT;  -- TXN-105 now committed
```

**Snapshot Isolation (Which Version to Read?):**

```sql
-- Transaction 107 starts
BEGIN;  -- TXN-107

-- What does this see?
SELECT balance FROM accounts WHERE id = 123;

-- Visibility rules:
-- 1. Row created (xmin) BEFORE my transaction started? ✓
-- 2. Row not deleted (xmax) OR deleted AFTER my transaction started? ✓
-- 3. Creating transaction (xmin) committed? ✓

-- TXN-107 visibility:
Old version (xmin=100, xmax=105):
  - Created by TXN-100 (committed, before TXN-107) ✓
  - Deleted by TXN-105 (committed, before TXN-107) ✗ → NOT VISIBLE

New version (xmin=105, xmax=0):
  - Created by TXN-105 (committed, before TXN-107) ✓
  - Not deleted (xmax=0) ✓ → VISIBLE

Result: TXN-107 sees balance = 900
```

### MySQL InnoDB MVCC Implementation

**Undo Logs (Rollback Segments):**

InnoDB stores old versions in undo logs, not in the table itself:

```
Table (clustered index): Contains latest version only
accounts: id=123, balance=900  (latest)

Undo log (rollback segment):
  Version 1: balance=1000, TRX_ID=100
  Version 2: balance=900,  TRX_ID=105

When Transaction 103 reads (started before TXN-105):
1. Read latest version from table: balance=900
2. Check TRX_ID (105) > my snapshot (103) → Too new!
3. Walk undo log backwards
4. Find version with TRX_ID <= 103 → balance=1000
5. Return balance=1000 (consistent read)
```

**InnoDB Row Format:**

```
Clustered Index Row:
+----------+---------+---------+------------------+
| DB_TRX_ID| DB_ROLL_PTR | id  | balance          |
+----------+---------+---------+------------------+
| 105      | 0x1A2B  | 123     | 900              |
+----------+---------+---------+------------------+

DB_TRX_ID: Transaction that last modified row
DB_ROLL_PTR: Pointer to undo log (previous versions)
```

### Garbage Collection (VACUUM in PostgreSQL)

**Problem: Old Versions Accumulate**

```sql
-- After 1000 updates:
Old versions: 1000 rows (xmax ≠ 0, deleted)
New version: 1 row (xmax = 0, current)

Total storage: 1001 rows (only 1 is visible!)
Disk space: 10× bloat
```

**Solution: VACUUM**

```sql
-- Manual vacuum
VACUUM accounts;

-- What VACUUM does:
-- 1. Scan table for dead tuples (xmax < oldest active transaction)
-- 2. Mark space as reusable (don't delete immediately)
-- 3. Update statistics

-- Auto-vacuum (background process)
SELECT name, setting FROM pg_settings WHERE name LIKE 'autovacuum%';

autovacuum = on
autovacuum_naptime = 1min  -- Check every minute
autovacuum_vacuum_threshold = 50  -- Min dead tuples
autovacuum_vacuum_scale_factor = 0.2  -- 20% of table size
```

---

## 4. Best Practices

### Practice 1: Monitor Bloat (PostgreSQL)

**Check Table Bloat:**
```sql
-- Bloat query (requires pgstattuple extension)
CREATE EXTENSION pgstattuple;

SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_table_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_table_size(schemaname||'.'||tablename)) AS indexes_size,
    round(100 * pgstattuple(schemaname||'.'||tablename).dead_tuple_percent, 2) AS dead_tuple_percent
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- Output:
--  tablename  | total_size | dead_tuple_percent 
-- ------------+------------+--------------------
--  orders     | 5000 MB    | 45% ← BLOATED! VACUUM needed!
--  products   | 1200 MB    | 5%  ← Healthy
```

**Fix Bloat:**
```sql
-- Heavy bloat (> 30%): Full vacuum (blocks writes)
VACUUM FULL orders;

-- Light bloat (< 30%): Regular vacuum (doesn't block)
VACUUM orders;

-- Reindex to defragment
REINDEX TABLE orders;
```

### Practice 2: Monitor Long-Running Transactions

**Long transactions = Old versions can't be cleaned:**

```sql
-- PostgreSQL: Find long-running transactions
SELECT
    pid,
    now() - xact_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
    AND xact_start IS NOT NULL
ORDER BY duration DESC
LIMIT 10;

-- Output:
--  pid  | duration  |  state  |         query                
-- ------+-----------+---------+------------------------------
--  1234 | 02:15:30  | idle in | BEGIN; SELECT * FROM orders...
--       |           | transaction |

-- Problem: This transaction started 2+ hours ago
-- → All row versions created in last 2 hours CANNOT BE VACUUMED
-- → Bloat accumulates!

-- Solution: Kill long transaction
SELECT pg_terminate_backend(1234);
```

**Set Transaction Timeout:**
```sql
-- Kill idle transactions after 5 minutes
SET idle_in_transaction_session_timeout = '5min';

-- Kill long-running queries after 30 seconds
SET statement_timeout = '30s';
```

### Practice 3: Use VACUUM ANALYZE Regularly

```sql
-- Manual vacuum + statistics update
VACUUM ANALYZE orders;

-- Auto-vacuum configuration (postgresql.conf)
autovacuum = on
autovacuum_max_workers = 3  -- Parallel workers
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.1  -- Vacuum at 10% dead tuples

-- Per-table tuning (for high-churn tables)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- More aggressive (5%)
    autovacuum_vacuum_threshold = 1000
);
```

### Practice 4: Monitor Transaction ID Wraparound

**PostgreSQL uses 32-bit transaction IDs (wraps after 2 billion):**

```sql
-- Check age of oldest transaction
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    2147483648 - age(datfrozenxid) AS xids_until_wraparound
FROM pg_database
ORDER BY xid_age DESC;

-- Output:
--  datname  | xid_age    | xids_until_wraparound 
-- ----------+------------+-----------------------
--  mydb     | 1500000000 | 647483648  ← OK
--  olddb    | 2100000000 | 47483648   ← DANGER! Vacuum soon!

-- If xid_age approaches 2 billion:
-- → Database enters read-only mode (protect data integrity)
-- → Must VACUUM to freeze old transactions

-- Emergency vacuum
VACUUM FREEZE;
```

---

## 5. Common Mistakes

### Mistake 1: Not Running VACUUM on High-Update Tables

**Problem:**
```sql
-- Table with frequent updates
UPDATE sessions SET last_active = NOW() WHERE user_id = 123;
-- Executed 1M times/day

-- After 1 week:
SELECT pg_size_pretty(pg_table_size('sessions'));
-- Result: 50 GB (was 5 GB initially!)

-- Dead tuple percent
SELECT dead_tuple_percent FROM pgstattuple('sessions');
-- Result: 85% bloat!

-- Query performance
EXPLAIN SELECT * FROM sessions WHERE user_id = 123;
-- Seq Scan: 10 seconds (was 100ms!)
```

**Solution:**
```sql
-- Aggressive auto-vacuum for high-churn tables
ALTER TABLE sessions SET (
    autovacuum_vacuum_scale_factor = 0.02,  -- Vacuum at 2% dead tuples
    autovacuum_vacuum_threshold = 500
);

-- Manual vacuum
VACUUM sessions;

-- Result:
-- - Table size: 50 GB → 6 GB
-- - Query time: 10s → 100ms
```

### Mistake 2: Long-Running Analytics Queries Block VACUUM

**Problem:**
```sql
-- Session 1: Long analytics query (started 10:00 AM)
BEGIN;
SELECT * FROM orders WHERE date > '2020-01-01';  -- Takes 2 hours

-- Session 2: Normal operations (10:01 AM - 12:00 PM)
-- 1M rows updated during these 2 hours

-- VACUUM runs (11:00 AM)
VACUUM orders;
-- Can't remove versions newer than Session 1's snapshot!
-- Result: VACUUM does nothing (bloat continues)

-- 12:00 PM: Session 1 finally finishes
-- Bloat accumulated: 1M dead tuples (can't clean until now)
```

**Solution:**
```sql
-- Analytics queries: Use read replicas (not primary)
-- On read replica:
SELECT * FROM orders WHERE date > '2020-01-01';
-- Doesn't affect primary's VACUUM

-- Or: Set transaction timeout
SET statement_timeout = '1hour';
BEGIN;
SELECT * FROM orders WHERE date > '2020-01-01';
-- Killed after 1 hour, VACUUM can proceed
```

### Mistake 3: Not Monitoring Transaction ID Wraparound

**Real Incident:**
```
Company: Social media platform
Impact: Database entered read-only mode

Timeline:
- Database running for 18 months
- No VACUUM FREEZE ever run
- Transaction ID age: 2.1 billion (approaching wraparound)
- PostgreSQL safety mechanism: ENTER READ-ONLY MODE
- All writes blocked: INSERT, UPDATE, DELETE → ERROR

Recovery:
1. Emergency VACUUM FREEZE (took 36 hours!)
2. During this time: Read-only mode (no new posts, likes, comments)
3. Revenue lost: $5M (3 days mostly read-only)
4. Reputation damage: "Site down" trending on Twitter

Prevention:
-- Monitor XID age (alerting)
SELECT age(datfrozenxid) FROM pg_database WHERE datname = 'mydb';
-- Alert when age > 1.5 billion

-- Regular VACUUM FREEZE (monthly)
VACUUM FREEZE;
```

---

## 6. Security Considerations

### 1. Sensitive Data in Old Versions

**Problem:**
```sql
-- Update password (GDPR compliance)
UPDATE users SET password_hash = 'new_hash' WHERE id = 123;

-- Old version still visible to long-running transactions!
-- Session started before UPDATE:
BEGIN;  -- Started 1 hour ago
SELECT password_hash FROM users WHERE id = 123;
-- Returns OLD password_hash! (MVCC snapshot)

-- Security risk: Compromised password still accessible
```

**Solution:**
```sql
-- Force VACUUM immediately after sensitive updates
UPDATE users SET password_hash = 'new_hash' WHERE id = 123;
VACUUM users;  -- Remove old versions

-- Or: Use advisory locks to prevent long transactions
SELECT pg_advisory_lock(123);  -- Lock before sensitive operation
UPDATE users SET password_hash = 'new_hash' WHERE id = 123;
SELECT pg_advisory_unlock(123);
```

### 2. Deleted Data Remains on Disk

**Problem:**
```sql
-- Delete sensitive record (e.g., right to be forgotten)
DELETE FROM user_data WHERE user_id = 123;
COMMIT;

-- Data still on disk (until VACUUM)!
-- Forensic recovery possible

-- VACUUM only marks space reusable (doesn't zero it)
VACUUM user_data;
-- Data ghosts remain on disk blocks!
```

**Solution:**
```sql
-- Secure deletion (PostgreSQL 11+)
DELETE FROM user_data WHERE user_id = 123;

-- Immediate vacuum
VACUUM FULL user_data;  -- Rewrites table (slower but secure)

-- Or: Overwrite before delete
UPDATE user_data SET 
    name = 'REDACTED',
    email = 'REDACTED',
    phone = 'REDACTED'
WHERE user_id = 123;

DELETE FROM user_data WHERE user_id = 123;
VACUUM user_data;
```

---

## 7. Performance Optimization

### Optimization 1: HOT Updates (Heap-Only Tuples)

**Problem: Every UPDATE creates new version:**
```sql
UPDATE orders SET status = 'shipped' WHERE id = 123;
-- Creates new row version
-- Updates ALL indexes (expensive!)
```

**Solution: HOT Updates (PostgreSQL optimization):**
```sql
-- Conditions for HOT update:
-- 1. Updated column NOT indexed
-- 2. New version fits on same page

-- Example: status column not indexed
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status VARCHAR(20),  -- Not indexed
    customer_id INT
);

CREATE INDEX idx_customer ON orders(customer_id);

-- HOT update (fast)
UPDATE orders SET status = 'shipped' WHERE id = 123;
-- New version on same page, index NOT updated!

-- Non-HOT update (slow)
UPDATE orders SET customer_id = 456 WHERE id = 123;
-- customer_id is indexed → Index updated (expensive)
```

**Monitor HOT Updates:**
```sql
SELECT
    schemaname,
    tablename,
    n_tup_upd AS total_updates,
    n_tup_hot_upd AS hot_updates,
    round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) AS hot_update_percent
FROM pg_stat_user_tables
WHERE n_tup_upd > 0
ORDER BY n_tup_upd DESC;

-- Output:
--  tablename | total_updates | hot_updates | hot_update_percent 
-- -----------+---------------+-------------+--------------------
--  orders    | 1000000       | 850000      | 85% ← Good! (HOT working)
--  users     | 500000        | 50000       | 10% ← Bad! (too many indexes?)
```

### Optimization 2: Fillfactor (Leave Space for Updates)

```sql
-- Default fillfactor: 100 (pack pages completely)
-- Problem: No room for HOT updates

-- Solution: Reserve space for updates
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status VARCHAR(20),
    amount DECIMAL
) WITH (fillfactor = 70);  -- Leave 30% free space per page

-- Benefits:
-- - HOT updates more likely (new version fits on same page)
-- - Fewer page splits
-- - Better performance for UPDATE-heavy tables

-- Recommendation:
-- - Read-only tables: fillfactor = 100
-- - Moderate updates: fillfactor = 80-90
-- - Frequent updates: fillfactor = 70
```

---

## 8. Examples

### Example 1: Snapshot Isolation Demonstration

```sql
-- Terminal 1: Start long transaction
BEGIN;  -- TXN-100
SELECT * FROM accounts WHERE id = 123;
-- balance = 1000

-- Terminal 2: Update balance
BEGIN;  -- TXN-101
UPDATE accounts SET balance = 900 WHERE id = 123;
COMMIT;

-- Terminal 3: New transaction sees update
BEGIN;  -- TXN-102
SELECT * FROM accounts WHERE id = 123;
-- balance = 900 (sees TXN-101's update)
COMMIT;

-- Terminal 1: Still sees old snapshot
SELECT * FROM accounts WHERE id = 123;
-- balance = 1000 (snapshot from TXN-100 start!)

-- Commit to see new version
COMMIT;

-- Terminal 1: New transaction
BEGIN;  -- TXN-103
SELECT * FROM accounts WHERE id = 123;
-- balance = 900 (now sees TXN-101's update)
```

### Example 2: Write Skew Anomaly

```sql
-- Constraint: At least 1 doctor on-call per shift

-- Initial state:
doctors: id=1, on_call=true  (Alice)
         id=2, on_call=true  (Bob)

-- Transaction A (Alice requests time off)
BEGIN;  -- TXN-100
SELECT COUNT(*) FROM doctors WHERE on_call = true;
-- Result: 2 (Alice + Bob)

-- Transaction B (Bob requests time off) [concurrent]
BEGIN;  -- TXN-101
SELECT COUNT(*) FROM doctors WHERE on_call = true;
-- Result: 2 (Alice + Bob) [same snapshot!]

-- Transaction A: Alice goes off-call
UPDATE doctors SET on_call = false WHERE id = 1;
-- Count still 2 in my snapshot, so OK
COMMIT;

-- Transaction B: Bob goes off-call
UPDATE doctors SET on_call = false WHERE id = 2;
-- Count still 2 in my snapshot, so OK
COMMIT;

-- Final state:
doctors: id=1, on_call=false  (Alice off)
         id=2, on_call=false  (Bob off)
-- CONSTRAINT VIOLATED! No doctors on-call!

-- Solution: Use SELECT FOR UPDATE (explicit locking)
BEGIN;
SELECT COUNT(*) FROM doctors WHERE on_call = true FOR UPDATE;
-- Locks rows, prevents concurrent updates
```

---

## 9. Real-World Use Cases

### Use Case 1: GitHub - MVCC for Zero-Downtime Migrations

**Challenge:**
- Migrate 50TB database with zero downtime
- Must serve reads during migration (24/7 availability)

**Solution: MVCC enables online schema changes:**

```sql
-- Traditional approach (blocks reads):
ALTER TABLE repositories ADD COLUMN visibility VARCHAR(10);
-- Acquires EXCLUSIVE lock → Blocks all reads/writes (hours!)

-- MVCC approach (doesn't block):
-- Step 1: Create new column (doesn't block, uses MVCC)
ALTER TABLE repositories ADD COLUMN visibility VARCHAR(10);

-- Step 2: Backfill in batches (old transactions see old schema)
FOR batch IN batches LOOP
    UPDATE repositories
    SET visibility = CASE
        WHEN private THEN 'private'
        ELSE 'public'
    END
    WHERE id BETWEEN batch_start AND batch_end;
    -- Each UPDATE creates new version
    -- Old transactions read old versions (no visibility column)
    -- New transactions read new versions (with visibility column)
END LOOP;

-- Step 3: Make column NOT NULL (after backfill complete)
ALTER TABLE repositories ALTER COLUMN visibility SET NOT NULL;

-- Result: Migration completed in 72 hours, zero downtime!
```

---

## 10. Interview Questions

### Q1: Explain how MVCC prevents read-write conflicts. Give an example.

**Answer:**

MVCC prevents conflicts by maintaining multiple versions of each row. Readers access old versions while writers create new versions—no locking needed between readers and writers.

**Example:**
```sql
-- Time T1: Initial state
accounts: xmin=100, xmax=0, balance=1000

-- Time T2: Transaction A starts (TXN-200)
BEGIN;
-- Snapshot: TXN-200 sees versions created by TXN ≤ 199

-- Time T3: Transaction B updates (TXN-201)
UPDATE accounts SET balance = 900 WHERE id = 123;
COMMIT;
-- Creates new version: xmin=201, xmax=0, balance=900
-- Old version marked deleted: xmin=100, xmax=201, balance=1000

-- Time T4: Transaction A reads
SELECT balance FROM accounts WHERE id = 123;
-- Sees old version (xmin=100, still visible to TXN-200)
-- Returns balance = 1000 (consistent snapshot)

-- No conflict! A reads old version, B wrote new version.
```

**Key Insight:** "MVCC trades space for concurrency. Extra versions consume disk, but eliminate read-write conflicts, improving throughput 10-100× in read-heavy workloads."

---

## 11. Summary

**MVCC Principles:**
- Multiple versions per row (writers don't block readers)
- Snapshot isolation (each transaction sees consistent view)
- Garbage collection needed (VACUUM removes old versions)

**Visibility Rules:**
- Row visible if: xmin ≤ my TXN AND (xmax = 0 OR xmax > my TXN)
- Each transaction has snapshot (set of visible TXN IDs)

**Best Practices:**
- Monitor bloat (VACUUM when dead_tuple_percent > 30%)
- Limit long transactions (set timeouts)
- Tune autovacuum (aggressive for high-churn tables)
- Watch transaction ID wraparound (age < 2 billion)

**Performance:**
- HOT updates (keep updated columns un-indexed)
- Fillfactor = 70 for UPDATE-heavy tables
- VACUUM ANALYZE regularly

**Key Takeaway:** "MVCC is the foundation of modern database concurrency. It enables thousands of concurrent readers without locks, but requires active maintenance (VACUUM) to prevent bloat. In production: Monitor dead tuple percentage, tune autovacuum, and kill long-running transactions."

---

**Next:** [03_Locking_Mechanisms.md](03_Locking_Mechanisms.md) - When MVCC isn't enough
