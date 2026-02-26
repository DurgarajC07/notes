# Lock Granularity - Row, Page, and Table Locking

## 1. Concept Explanation

**Lock Granularity** refers to the size of the database object being locked during a transaction. Smaller granularity (row-level) allows more concurrency but higher overhead. Larger granularity (table-level) has lower overhead but blocks more transactions.

**Three Main Levels:**

**1. Row-Level Locking:**
```
Transaction A locks: Row 123 only

+-------+----------+
| ID    | Balance  | Lock |
+-------+----------+------+
| 122   | 1000     | -    |  ← Transaction B can access
| 123   | 500      | 🔒   |  ← Transaction A locked
| 124   | 750      | -    |  ← Transaction B can access
+-------+----------+------+

Concurrency: HIGH (other rows accessible)
Overhead: MEDIUM (manage many locks)
```

**2. Page-Level Locking:**
```
Transaction A locks: Entire page (8KB, ~100 rows)

Page 1 (Rows 1-100):   🔒 Locked
Page 2 (Rows 101-200): -  Accessible
Page 3 (Rows 201-300): -  Accessible

Concurrency: MEDIUM (whole page blocked)
Overhead: LOW (fewer locks to manage)
```

**3. Table-Level Locking:**
```
Transaction A locks: Entire table

+-------------------+
| Accounts Table    | 🔒 Locked
| - Row 1-1,000,000 | 
+-------------------+

Concurrency: LOW (everything blocked)
Overhead: VERY LOW (single lock)
```

**Analogy: Parking Lot Security**
```
Row-level = Individual car locks
- Each car locked separately
- High security (others can access their cars)
- High overhead (manage 1000 locks)

Page-level = Section gates
- Lock parking section (10 cars)
- Medium security (section blocked)
- Medium overhead (manage 100 gates)

Table-level = Entire lot gate
- Lock whole parking lot
- Low security (everyone blocked)
- Low overhead (single gate)
```

---

## 2. Why It Matters

### Production Impact: Page-Level Locking Disaster

**Scenario: SQL Server Retail Database**

```
Company: National retail chain
Database: SQL Server 2016 (page-level locking by default for some operations)
Issue: Black Friday checkout slowdown

Timeline:
Black Friday 2019, 10:00 AM - Peak traffic

Table: customers (5 million rows, clustered index on customer_id)
Page size: 8KB (approximately 100 customer rows per page)

Normal operation (row-level locking):
-- Transaction 1: Update customer 123 address
BEGIN;
UPDATE customers SET address = '123 Main St' WHERE customer_id = 123;
-- Locks: Row 123 only
-- Other customers: Accessible ✓
COMMIT;

Black Friday (lock escalation to page-level):
-- SQL Server: "Too many row locks (> 5000), escalating to page locks"

-- Transaction 1: Update customer 123 (on Page 10, rows 1000-1099)
BEGIN;
UPDATE customers SET address = '123 Main St' WHERE customer_id = 123;
-- Lock escalated: Entire Page 10 (rows 1000-1099) locked! ⚠️

-- Transaction 2: Update customer 1050 (also on Page 10!)
BEGIN;
UPDATE customers SET loyalty_points = 500 WHERE customer_id = 1050;
-- BLOCKED waiting for Transaction 1 (same page!) ✗

-- Transaction 3: Update customer 1025 (Page 10 again!)
BEGIN;
UPDATE customers SET email = 'new@email.com' WHERE customer_id = 1025;
-- BLOCKED waiting for Transaction 1 ✗

COMMIT;  -- Transaction 1 (after 5 seconds processing)

-- Transactions 2 and 3 now proceed (waited 5 seconds)

Result:
- Query time: 20ms → 5+ seconds (250× slower!)
- Checkout page: Users see "Processing..." spinning for 5-10 seconds
- Abandoned carts: 25% increase (users think site crashed)
- Revenue loss: $500K (Black Friday sales lost)

Investigation:
- Lock monitor: 50,000+ page locks (normally 0)
- Lock escalation: 5000+ row locks → page locks
- Contention: 100× increase in lock waits

Root cause: SQL Server lock escalation (row → page → table)

Fix 1: Disable lock escalation
ALTER TABLE customers SET (LOCK_ESCALATION = DISABLE);
-- Keep row-level locks, don't escalate ✓

Fix 2: Partition table
-- Partition by customer_id range (reduces rows per page accessed)
CREATE PARTITION FUNCTION customer_range (INT)
AS RANGE LEFT FOR VALUES (1000000, 2000000, 3000000, 4000000);

Post-fix:
- Query time: 20ms (back to normal) ✓
- Abandoned carts: 10% (normal baseline) ✓
- Revenue recovered: Full Black Friday potential ✓
```

### Real Incident: MySQL Table-Level Lock Kills Production

```
Date: June 2018
Company: SaaS helpdesk platform
Database: MySQL 5.5 (MyISAM storage engine, table-level locking)
Issue: Daily backup locks entire database

Scenario: Nightly backup job

Table: tickets (MyISAM engine, 10 million rows)
Backup time: 02:00 AM - 04:00 AM (2 hours)

02:00:00 - Backup starts
BEGIN;
SELECT * FROM tickets;  -- Full table scan for backup

-- MyISAM: Acquires table-level WRITE lock

02:05:00 - Customer creates support ticket (Europe timezone, daytime)
BEGIN;
INSERT INTO tickets (customer_id, subject, body) VALUES (123, 'Issue', 'Help!');
-- BLOCKED waiting for backup to release table lock! ⏳

02:10:00 - 50 more customers try to create tickets
-- All BLOCKED (table locked for 2 hours!) ✗

02:30:00 - Support agents try to view tickets
SELECT * FROM tickets WHERE status = 'open';
-- BLOCKED (table still locked!) ✗

03:00:00 - CEO checks ticket queue
-- Dashboard blank (queries blocked) ✗

03:30:00 - Monitoring alert: "100+ database connections waiting"

04:00:00 - Backup completes, table lock released
-- 100 INSERT/SELECT queries execute simultaneously
-- Database CPU: 100% (spike)
-- Query time: 10 seconds (thundering herd)

Result:
- Downtime: 2 hours (02:00 AM - 04:00 AM)
- Customer impact: European customers (prime business hours)
- Tickets lost: 150 (customers gave up)
- Revenue loss: $50K (customers churned)
- Reputation: 200 negative reviews "Database locks during daytime"

Root cause: MyISAM table-level locking (no row-level locks)

Fix: Migrate to InnoDB (row-level locking)
ALTER TABLE tickets ENGINE=InnoDB;

-- New behavior:
-- Backup: Reads rows using MVCC (no locks!)
-- Inserts: Lock individual rows only
-- Concurrency: Backup + inserts run simultaneously ✓

Post-fix:
- Downtime: 0 hours (backup doesn't block inserts)
- Customer impact: None ✓
- Tickets lost: 0 ✓
```

---

## 3. Internal Working

### Row-Level Locking (PostgreSQL, InnoDB)

**How Row Locks Work:**
```sql
-- PostgreSQL row lock

-- Transaction A
BEGIN;
UPDATE accounts SET balance = 900 WHERE id = 123;
-- Internal: Row lock acquired

-- Lock stored in row header (xmax field)
-- Row tuple:
-- xmin: 1000 (transaction that created row)
-- xmax: 1501 (transaction A holding lock)
-- t_infomask: HEAP_XMAX_EXCL_LOCK (exclusive lock)

-- Transaction B [concurrent]
BEGIN;
UPDATE accounts SET balance = 800 WHERE id = 123;
-- Checks xmax: 1501 (locked)
-- Waits for Transaction 1501 to commit/rollback ⏳

-- Transaction A
COMMIT;  -- Row lock released (xmax cleared)

-- Transaction B
-- Lock acquired, update proceeds ✓
```

**Lock Queue (Multiple Waiters):**
```
Row 123 locked by Transaction A

Lock queue:
1. Transaction A: HOLDING LOCK
2. Transaction B: WAITING FOR EXCLUSIVE LOCK
3. Transaction C: WAITING FOR EXCLUSIVE LOCK

Transaction A commits:
- Transaction B acquires lock (first in queue)
- Transaction C continues waiting

Transaction B commits:
- Transaction C acquires lock
```

### Page-Level Locking (SQL Server)

**How Page Locks Work:**
```sql
-- SQL Server

-- Transaction A
BEGIN;
UPDATE customers SET name = 'John' WHERE customer_id = 123;

-- Internal: Acquire page lock
-- Page 10 (rows 100-199) locked

-- Lock manager entry:
-- Resource: PAGE (Database 5, File 1, Page 10)
-- Mode: EXCLUSIVE
-- Duration: TRANSACTION

-- Transaction B [concurrent]
BEGIN;
UPDATE customers SET name = 'Jane' WHERE customer_id = 150;
-- Same page (Page 10)!

-- Checks lock manager: Page 10 locked by Transaction A
-- Waits... ⏳

-- Transaction A
COMMIT;  -- Page lock released

-- Transaction B
-- Acquires page lock, update proceeds ✓
```

**Lock Escalation (Row → Page → Table):**
```
Scenario: Update 10,000 rows

Initial: Row-level locks
- Lock 1: Row 1
- Lock 2: Row 2
- ...
- Lock 5000: Row 5000

SQL Server: "Too many locks (> 5000 threshold), escalating"

Escalation: Page-level locks
- Lock 1: Page 1 (rows 1-100)
- Lock 2: Page 2 (rows 101-200)
- ...
- Lock 50: Page 50 (rows 4901-5000)

SQL Server: "Still too many locks (> 5000 pages), escalating again"

Escalation: Table-level lock
- Lock 1: Entire table

Benefit: Reduce lock manager memory (1 lock vs 10,000 locks)
Cost: Block all concurrent access to table ✗
```

### Table-Level Locking (MyISAM, Explicit LOCK TABLES)

**How Table Locks Work:**
```sql
-- MySQL MyISAM (automatic table locking)

-- Transaction A
BEGIN;
UPDATE tickets SET status = 'closed' WHERE id = 123;

-- MyISAM: Acquires table-level WRITE lock automatically
-- Lock manager:
-- Resource: TABLE `tickets`
-- Mode: WRITE
-- Duration: Until COMMIT

-- Transaction B [concurrent]
BEGIN;
SELECT * FROM tickets WHERE customer_id = 456;
-- BLOCKED (table locked for WRITE) ⏳

-- Transaction C [concurrent]
BEGIN;
INSERT INTO tickets (subject) VALUES ('New ticket');
-- BLOCKED (table locked) ⏳

-- Transaction A
COMMIT;  -- Table lock released

-- Transactions B and C proceed simultaneously
```

---

## 4. Best Practices

### Practice 1: Use Row-Level Locking Databases

**Modern Choice:**
```sql
-- Prefer: PostgreSQL, MySQL InnoDB, Oracle
-- These use row-level locks by default

-- Example: InnoDB
CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL(10, 2)
) ENGINE=InnoDB;  -- Row-level locking ✓

-- Transaction A
UPDATE accounts SET balance = 900 WHERE id = 123;
-- Locks: Row 123 only

-- Transaction B [concurrent]
UPDATE accounts SET balance = 800 WHERE id = 456;
-- Proceeds (different row) ✓
```

**Avoid:**
```sql
-- Avoid: MyISAM, explicit LOCK TABLES

-- MyISAM (table-level locks)
CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL(10, 2)
) ENGINE=MyISAM;  -- ✗ Table-level locking

-- Transaction A
UPDATE accounts SET balance = 900 WHERE id = 123;
-- Locks: Entire table ✗

-- Transaction B [concurrent]
UPDATE accounts SET balance = 800 WHERE id = 456;
-- BLOCKED (table locked) ✗
```

### Practice 2: Monitor Lock Escalation (SQL Server)

**Check Lock Escalation:**
```sql
-- SQL Server: Check lock escalation events

SELECT
    object_name(object_id) AS table_name,
    index_id,
    partition_number,
    lock_escalation_desc
FROM sys.tables
WHERE lock_escalation_desc != 'TABLE';

-- Disable escalation for specific tables
ALTER TABLE high_traffic_table SET (LOCK_ESCALATION = DISABLE);
```

**Monitor Lock Waits:**
```sql
-- PostgreSQL: Check lock waits
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### Practice 3: Batch Updates with Locks

**Anti-Pattern (locks all rows for long time):**
```sql
-- Update 1 million rows (holds locks for 10 minutes!)
BEGIN;

UPDATE accounts SET status = 'inactive'
WHERE last_login < NOW() - INTERVAL '1 year';
-- Locks: 1 million rows for 10 minutes ✗

COMMIT;

-- Problem: Blocks all concurrent updates to these rows
```

**Best Practice (batch in small transactions):**
```sql
-- Batch 1: Update 1000 rows
BEGIN;

UPDATE accounts SET status = 'inactive'
WHERE id IN (
    SELECT id FROM accounts
    WHERE last_login < NOW() - INTERVAL '1 year'
    LIMIT 1000
);
-- Locks: 1000 rows for 1 second ✓

COMMIT;  -- Release locks

-- Batch 2: Next 1000 rows
BEGIN;
UPDATE accounts SET status = 'inactive' WHERE id IN (...);
COMMIT;

-- Repeat until done

-- Benefit:
-- - Locks released frequently (reduces contention)
-- - Other transactions can acquire locks between batches
-- - Total time same (1000 batches × 1s = 10 min)
-- - Concurrency dramatically improved ✓
```

---

## 5. Common Mistakes

### Mistake 1: Using SELECT FOR UPDATE on Full Table

**Problem:**
```sql
-- Lock all rows in table (millions of rows!)
BEGIN;

SELECT * FROM orders FOR UPDATE;
-- Locks: ALL rows in orders table ✗

-- Process each order (takes 10 minutes)
FOR order IN orders:
    -- Update order
    UPDATE orders SET status = 'processed' WHERE id = order.id;

COMMIT;  -- Locks held for 10 minutes

-- Impact:
-- - All concurrent SELECT FOR UPDATE: BLOCKED
-- - All concurrent UPDATE: BLOCKED
-- - Table effectively read-only for 10 minutes ✗
```

**Fix:**
```sql
-- Lock specific rows only
BEGIN;

SELECT * FROM orders
WHERE status = 'pending'
  AND created_at < NOW() - INTERVAL '1 hour'
FOR UPDATE
LIMIT 100;  -- Batch of 100 rows only ✓

-- Process batch (1 second)
UPDATE orders SET status = 'processed' WHERE id IN (...);

COMMIT;  -- Locks released after 1 second ✓

-- Other transactions can access other rows concurrently ✓
```

### Mistake 2: Not Understanding Lock Compatibility

**Problem (conflicting locks):**
```sql
-- Transaction A
BEGIN;
SELECT * FROM accounts WHERE id = 123 FOR SHARE;
-- Acquires: SHARED lock (allows other readers)

-- Transaction B
BEGIN;
SELECT * FROM accounts WHERE id = 123 FOR SHARE;
-- Acquires: SHARED lock (compatible with Transaction A) ✓

-- Transaction C
BEGIN;
UPDATE accounts SET balance = 900 WHERE id = 123;
-- Needs: EXCLUSIVE lock
-- BLOCKED by Transaction A and B's SHARED locks ⏳

-- Transaction A and B hold locks indefinitely (reading data)
-- Transaction C waits forever ✗
```

**Lock Compatibility Matrix:**
```
               | SHARE | EXCLUSIVE |
---------------|-------|-----------|
SHARE          | ✓ OK  | ✗ BLOCK   |
EXCLUSIVE      | ✗ BLOCK | ✗ BLOCK |

Key insight:
- Multiple SHARE locks: Compatible (readers don't block readers)
- SHARE + EXCLUSIVE: Incompatible (readers block writers)
- Multiple EXCLUSIVE: Incompatible (writers block writers)
```

**Fix:**
```sql
-- Use FOR UPDATE SKIP LOCKED (PostgreSQL 9.5+)
BEGIN;

SELECT * FROM accounts
WHERE status = 'pending'
FOR UPDATE SKIP LOCKED  -- Skip locked rows ✓
LIMIT 10;

-- Benefits:
-- - Doesn't block on locked rows (skips them)
-- - Processes available rows immediately
-- - Other workers process other rows concurrently ✓

COMMIT;
```

---

## 6. Security Considerations

### Preventing Denial-of-Service via Lock Escalation

**Attack Scenario:**
```sql
-- Attacker runs malicious query
BEGIN;

-- Update millions of rows (triggers lock escalation)
UPDATE users SET last_seen = NOW()
WHERE user_type = 'free';  -- 5 million rows

-- SQL Server: Lock escalation (row → page → table)
-- Entire `users` table locked ✗

-- Impact:
-- - All login queries: BLOCKED
-- - All user profile updates: BLOCKED
-- - Application: Effectively down ✗

-- Attacker holds transaction open for 1 hour (DoS)
SLEEP(3600);

COMMIT;
```

**Mitigation:**
```sql
-- Option 1: Disable lock escalation
ALTER TABLE users SET (LOCK_ESCALATION = DISABLE);

-- Option 2: Set lock timeout
SET LOCK_TIMEOUT 5000;  -- 5 seconds max wait

BEGIN;
UPDATE users SET last_seen = NOW() WHERE user_type = 'free';
-- If lock timeout exceeded: ERROR (transaction aborted) ✓
COMMIT;

-- Option 3: Rate limit large updates (application layer)
-- Batch updates: Max 1000 rows per transaction
-- Max 10 transactions per second per user
```

---

## 7. Performance Optimization

### Benchmark: Lock Granularity Performance

**Test Setup:**
```
Workload: 1000 concurrent transactions
Each transaction: Update 1 random row
Table: 1 million rows
```

**Results:**

| Lock Granularity | Throughput (TPS) | Latency (p50) | Latency (p95) | Lock Waits |
|------------------|------------------|---------------|---------------|------------|
| **Row-level (InnoDB)** | 10,000 TPS | 10ms | 30ms | 5% |
| **Page-level (SQL Server)** | 3,000 TPS | 30ms | 200ms | 40% |
| **Table-level (MyISAM)** | 100 TPS | 500ms | 2000ms | 99% |

**Key Insights:**
```
1. Row-level: 100× faster than table-level
2. Page-level: 3× faster than table-level, 3× slower than row-level
3. Table-level: Serial execution (no concurrency)

Conclusion: Always use row-level locking (InnoDB, PostgreSQL)
```

### Optimization: Reduce Lock Contention

**Hot Spot Problem:**
```sql
-- Counter table (single row updated by all transactions)
CREATE TABLE counters (
    name VARCHAR(50) PRIMARY KEY,
    value BIGINT
);

INSERT INTO counters (name, value) VALUES ('page_views', 0);

-- 1000 concurrent transactions
UPDATE counters SET value = value + 1 WHERE name = 'page_views';
-- Serial execution (all wait for same row lock) ✗
-- Throughput: ~100 TPS
```

**Fix 1: Sharding (Distribute Lock Contention):**
```sql
-- Multiple counter shards
CREATE TABLE counter_shards (
    name VARCHAR(50),
    shard_id INT,
    value BIGINT,
    PRIMARY KEY (name, shard_id)
);

-- Insert 100 shards
INSERT INTO counter_shards (name, shard_id, value)
SELECT 'page_views', generate_series(1, 100), 0;

-- Update random shard (distribute contention)
UPDATE counter_shards
SET value = value + 1
WHERE name = 'page_views' AND shard_id = (random() * 99 + 1)::INT;
-- Throughput: ~10,000 TPS (100× improvement) ✓

-- Read total
SELECT SUM(value) FROM counter_shards WHERE name = 'page_views';
```

**Fix 2: Batch Updates (Application-Side Buffering):**
```python
# Buffer increments in application
counter_buffer = defaultdict(int)

def increment_counter(name):
    counter_buffer[name] += 1
    
    # Flush every 100 increments
    if counter_buffer[name] % 100 == 0:
        db.execute(
            "UPDATE counters SET value = value + :increment WHERE name = :name",
            increment=counter_buffer[name], name=name
        )
        counter_buffer[name] = 0

# Result: 100× fewer database updates (less lock contention) ✓
```

---

## 8. Examples

### Example 1: Job Queue Processing (SKIP LOCKED)

```sql
-- Job queue table
CREATE TABLE job_queue (
    id SERIAL PRIMARY KEY,
    payload JSONB,
    status VARCHAR(20) DEFAULT 'pending',
    worker_id INT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Worker function (PostgreSQL)
CREATE OR REPLACE FUNCTION claim_job(p_worker_id INT)
RETURNS TABLE(job_id INT, payload JSONB) AS $$
BEGIN
    RETURN QUERY
    UPDATE job_queue
    SET status = 'processing', worker_id = p_worker_id
    WHERE id IN (
        SELECT id FROM job_queue
        WHERE status = 'pending'
        ORDER BY created_at
        FOR UPDATE SKIP LOCKED  -- Skip jobs locked by other workers ✓
        LIMIT 1
    )
    RETURNING id, payload;
END;
$$ LANGUAGE plpgsql;

-- Multiple workers run concurrently
-- Worker 1:
SELECT * FROM claim_job(1);  -- Returns job 101 (locks it)

-- Worker 2: [concurrent]
SELECT * FROM claim_job(2);  -- Returns job 102 (skips locked 101) ✓

-- Worker 3: [concurrent]
SELECT * FROM claim_job(3);  -- Returns job 103 (skips locked 101, 102) ✓

-- Benefit: Workers don't block each other (process jobs in parallel) ✓
```

### Example 2: Inventory Reservation (Row-Level Locking)

```sql
CREATE OR REPLACE FUNCTION reserve_inventory(
    p_product_id INT,
    p_quantity INT
) RETURNS TEXT AS $$
DECLARE
    v_available INT;
BEGIN
    -- Lock product row (row-level lock, not whole table!)
    SELECT stock INTO v_available
    FROM inventory
    WHERE product_id = p_product_id
    FOR UPDATE;  -- Exclusive lock on this row only ✓
    
    -- Other products remain accessible to concurrent transactions ✓
    
    -- Check availability
    IF v_available < p_quantity THEN
        RAISE EXCEPTION 'Insufficient stock: % available, % requested',
            v_available, p_quantity;
    END IF;
    
    -- Reserve
    UPDATE inventory
    SET stock = stock - p_quantity
    WHERE product_id = p_product_id;
    
    RETURN 'Reserved ' || p_quantity || ' units';
END;
$$ LANGUAGE plpgsql;

-- Concurrent reservations for different products:
-- Transaction 1: reserve_inventory(100, 5) → Locks product 100 only
-- Transaction 2: reserve_inventory(200, 3) → Locks product 200 only
-- Both proceed concurrently (different rows) ✓
```

---

## 9. Real-World Use Cases

### Use Case: Stripe Payment Processing

**Stripe's Lock Granularity Strategy:**

```python
class StripePaymentProcessor:
    def process_payment(self, customer_id, amount):
        """
        Process payment with row-level locking
        Only locks customer row, not entire table
        """
        with db.transaction():
            # Lock customer row only (PostgreSQL row-level lock)
            customer = db.query("""
                SELECT id, balance, currency
                FROM customers
                WHERE id = :customer_id
                FOR UPDATE  -- Row-level lock ✓
            """, customer_id=customer_id).first()
            
            # Other customers' payments proceed concurrently ✓
            
            # Check balance
            if customer.balance < amount:
                raise InsufficientFunds()
            
            # Deduct balance
            db.execute("""
                UPDATE customers
                SET balance = balance - :amount
                WHERE id = :customer_id
            """, amount=amount, customer_id=customer_id)
            
            # Create charge
            charge_id = db.execute("""
                INSERT INTO charges (customer_id, amount, status, created_at)
                VALUES (:customer_id, :amount, 'succeeded', NOW())
                RETURNING id
            """, customer_id=customer_id, amount=amount)
            
            return charge_id

# Concurrent payments processing:
# Thread 1: process_payment(customer_123, $100) → Locks row 123
# Thread 2: process_payment(customer_456, $200) → Locks row 456
# Thread 3: process_payment(customer_789, $50)  → Locks row 789
# All proceed in parallel (different rows, no contention) ✓

# Throughput: 10,000+ payments/second
# If using table-level locks: ~100 payments/second (100× slower) ✗
```

---

## 10. Interview Questions

### Q1: Explain the trade-offs between row-level, page-level, and table-level locking. When would you use each?

**Answer:**

**Trade-Offs:**

| Feature | Row-Level | Page-Level | Table-Level |
|---------|-----------|------------|-------------|
| **Concurrency** | ✅ High (lock specific rows) | ⚠️ Medium (lock ~100 rows) | ❌ Low (lock entire table) |
| **Overhead** | ⚠️ Medium (manage many locks) | ✅ Low (fewer locks) | ✅ Very low (single lock) |
| **Throughput** | ✅ High (parallel updates) | ⚠️ Medium | ❌ Low (serial execution) |
| **Lock Escalation** | ⚠️ Possible (SQL Server) | N/A | N/A |

**When to Use Each:**

**Row-Level (99% of cases):**
```sql
-- OLTP applications (high concurrency)
-- E-commerce, banking, SaaS

-- Use case: Update customer profile
UPDATE customers SET email = 'new@email.com' WHERE id = 123;
-- Locks: Row 123 only (other customers accessible) ✓

-- Databases: PostgreSQL, MySQL InnoDB, Oracle
```

**Page-Level (legacy systems):**
```sql
-- SQL Server default (with lock escalation prevention)
-- Useful when: Medium concurrency, memory-constrained

-- Use case: Batch update related rows on same page
UPDATE orders SET status = 'shipped'
WHERE order_date = '2024-01-15';  -- ~100 orders on same page

-- Benefit: Fewer locks to manage (memory efficient)
-- Cost: More rows locked than necessary
```

**Table-Level (rare, specific cases):**
```sql
-- Use case 1: Full table rebuild
BEGIN;
LOCK TABLE products IN EXCLUSIVE MODE;
-- Rebuild indexes, reorganize data
-- No concurrent access needed (maintenance window)
COMMIT;

-- Use case 2: Bulk data load (no concurrent writes)
BEGIN;
LOCK TABLE staging_data IN EXCLUSIVE MODE;
COPY staging_data FROM '/tmp/data.csv';
COMMIT;

-- Use case 3: Read-heavy, write-rarely (MyISAM acceptable)
-- Example: Archive data, log tables (append-only)
```

**Interview insight:** "Always use row-level locking for OLTP workloads. At [Company], when we migrated from MyISAM (table-level) to InnoDB (row-level), throughput increased from 200 TPS to 15,000 TPS (75× improvement) on our highest-traffic tables. The only time we intentionally use table-level locks is during maintenance windows for full-table operations like VACUUM FULL or table rebuilds. We've implemented lock timeout policies (5 seconds max) to prevent long-running transactions from escalating locks and causing cascading failures. We monitor lock wait events in our database metrics and alert if > 1% of queries wait on locks."

---

## 11. Summary

**Lock Granularity:**
- **Row-Level:** Lock individual rows (PostgreSQL, InnoDB default)
- **Page-Level:** Lock 8KB pages ~100 rows (SQL Server, some operations)
- **Table-Level:** Lock entire table (MyISAM, explicit LOCK TABLES)

**Key Behavior:**
```sql
-- Row-level locking (InnoDB)
UPDATE accounts SET balance = 900 WHERE id = 123;
-- Locks: Row 123 only ✓
-- Concurrent update to row 456: Proceeds immediately ✓

-- Table-level locking (MyISAM)
UPDATE accounts SET balance = 900 WHERE id = 123;
-- Locks: Entire accounts table ✗
-- Concurrent update to row 456: BLOCKED ✗
```

**Performance Comparison:**

| Granularity | Throughput | Latency | Concurrency | Use Case |
|-------------|------------|---------|-------------|----------|
| **Row-level** | 10,000 TPS | 10ms | High | OLTP (99% of apps) |
| **Page-level** | 3,000 TPS | 30ms | Medium | Legacy SQL Server |
| **Table-level** | 100 TPS | 500ms | Low | Bulk operations, maintenance |

**Best Practices:**
✅ Use row-level locking databases (PostgreSQL, InnoDB)
✅ Disable lock escalation on high-traffic tables (SQL Server)
✅ Batch large updates (1000 rows per transaction max)
✅ Use FOR UPDATE SKIP LOCKED for job queues (PostgreSQL)
✅ Monitor lock waits (alert if > 1% queries wait)
❌ Avoid SELECT FOR UPDATE on full table scan
❌ Don't use MyISAM for high-concurrency tables
❌ Avoid long-running transactions holding locks

**Common Pitfalls:**
1. **Lock escalation:** SQL Server escalates row → page → table (disable on hot tables)
2. **Full table scan locks:** FOR UPDATE without WHERE clause (locks all rows)
3. **MyISAM in production:** Table-level locks kill concurrency (migrate to InnoDB)
4. **Conflicting locks:** Multiple SHARE locks block single EXCLUSIVE lock (use SKIP LOCKED)

**Critical Pattern:**
```sql
-- Job queue pattern (high concurrency)
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
FOR UPDATE SKIP LOCKED  -- Skip locked rows ✓
LIMIT 10;

-- Multiple workers process jobs in parallel ✓
-- Each worker locks different rows (no contention) ✓
```

**Lock Escalation Prevention (SQL Server):**
```sql
-- Disable escalation on hot tables
ALTER TABLE high_traffic_table SET (LOCK_ESCALATION = DISABLE);

-- Monitor escalation events
SELECT * FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL)
WHERE index_lock_promotion_count > 0;
```

**Key Takeaway:** "Lock granularity is the difference between 100 TPS and 10,000 TPS. Row-level locking is non-negotiable for modern OLTP applications. The 2010s migration from MyISAM (table-level) to InnoDB (row-level) was primarily motivated by concurrency improvements. For SQL Server, disabling lock escalation on hot tables prevents catastrophic performance degradation under load. Always use FOR UPDATE SKIP LOCKED for job queues and work distribution—it's the pattern that enables horizontal scaling of background workers. Monitor lock wait metrics: if > 1% queries wait on locks, investigate lock escalation, long-running transactions, or missing indexes causing table scans with locks."

---

**Project:** [08_Isolation_Levels_And_Locking](README.md) - Full section overview and learning paths
