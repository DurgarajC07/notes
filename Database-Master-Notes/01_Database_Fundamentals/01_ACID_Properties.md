# 🔐 ACID Properties - Database Transaction Guarantees

> ACID properties are the foundation of reliable database systems. Understanding ACID is essential for building data-consistent applications and making trade-off decisions in distributed systems.

---

## 📖 1. Concept Explanation

### What is ACID?

**ACID** is an acronym representing four properties that guarantee reliable processing of database transactions:

1. **Atomicity** - All or nothing
2. **Consistency** - Data integrity maintained
3. **Isolation** - Concurrent transactions don't interfere
4. **Durability** - Committed data persists

**Transaction:**

```
BEGIN TRANSACTION
   UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Debit
   UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Credit
COMMIT; -- Both operations succeed or both fail
```

**Why ACID exists:**

- Prevent data corruption during failures
- Handle concurrent users safely
- Guarantee consistency across operations
- Protect critical business data (money, inventory, etc.)

---

### The Four Pillars

```
ACID Properties
│
├── Atomicity ──────> Transaction is all-or-nothing
│                     └─> Partial writes are impossible
│
├── Consistency ────> Data meets all rules/constraints
│                     └─> Invariants are preserved
│
├── Isolation ──────> Concurrent transactions don't interfere
│                     └─> Serializable execution
│
└── Durability ─────> Committed data survives crashes
                      └─> Write-ahead logging (WAL)
```

---

## 🧠 2. Why It Matters in Real Systems

### Without ACID - Real Disasters

**E-commerce nightmare:**

```sql
-- Order placed
INSERT INTO orders (user_id, total) VALUES (123, 500.00);

-- [CRASH HERE] 💥

-- Inventory never updated!
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 456;
```

**Result:** Customer charged, product never shipped, inventory wrong.

---

**Banking catastrophe:**

```sql
-- Transfer $1000 from Account A to B
UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';

-- [CRASH] 💥

-- Credit never happens!
UPDATE accounts SET balance = balance + 1000 WHERE id = 'B';
```

**Result:** Money vanishes from the system. 🔥

---

### With ACID - Guaranteed Safety

```sql
BEGIN TRANSACTION;
   -- Debit account A
   UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';

   -- Credit account B
   UPDATE accounts SET balance = balance + 1000 WHERE id = 'B';
COMMIT;

-- If ANY step fails (crash, constraint violation, etc.)
-- → ENTIRE transaction rolls back
-- → Both accounts remain unchanged ✅
```

---

### Impact on Scale

| Scenario               | With ACID         | Without ACID          |
| ---------------------- | ----------------- | --------------------- |
| **Bank transfers**     | Safe, guaranteed  | Money can disappear   |
| **Inventory systems**  | Stock accurate    | Overselling happens   |
| **Seat reservations**  | No double-booking | Race conditions       |
| **Financial ledgers**  | Audit-compliant   | Cannot be trusted     |
| **Payment processing** | PCI-compliant     | Regulatory violations |

**Cost of ACID:**

- ✅ Data safety and compliance
- ❌ Lower throughput (locking, coordination)
- ❌ Higher latency (fsync, durable writes)
- ❌ Complex distributed implementation

---

## ⚙️ 3. Internal Working

### Atomicity - How It Works

**Implementation: Undo Log / Rollback Segments**

```
Transaction Start:
1. BEGIN TRANSACTION
2. Database writes "undo information" to transaction log
3. Execute operations (UPDATE, INSERT, DELETE)
4. COMMIT or ROLLBACK

If COMMIT:
   → Apply changes permanently
   → Discard undo log

If ROLLBACK (or crash):
   → Use undo log to reverse changes
   → Restore original values
```

**PostgreSQL Example:**

```sql
BEGIN;
   UPDATE accounts SET balance = 500 WHERE id = 1;
   -- PostgreSQL keeps old tuple (balance = 1000) with transaction ID
   -- New tuple (balance = 500) marked with current transaction ID

ROLLBACK;
   -- Old tuple becomes visible again
   -- New tuple is discarded
```

**MySQL InnoDB:**

- Uses **undo tablespace** to store old row versions
- Rollback reads undo log backwards, reversing changes
- Long transactions → large undo logs → performance impact

---

### Consistency - Data Integrity

**Implementation: Constraints + Application Logic**

```sql
-- Database-level constraints
CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL(10,2) CHECK (balance >= 0),  -- No negative balances
    CONSTRAINT unique_account UNIQUE (account_number)
);

-- Foreign key constraints
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)  -- User must exist
);

-- Application-level consistency
BEGIN TRANSACTION;
   -- Rule: Total money in system must remain constant

   DECLARE @sum_before DECIMAL(10,2);
   SELECT @sum_before = SUM(balance) FROM accounts;

   UPDATE accounts SET balance = balance - 100 WHERE id = 1;
   UPDATE accounts SET balance = balance + 100 WHERE id = 2;

   DECLARE @sum_after DECIMAL(10,2);
   SELECT @sum_after = SUM(balance) FROM accounts;

   IF @sum_before != @sum_after THEN
      ROLLBACK;  -- Invariant violated!
   ELSE
      COMMIT;
   END IF;
END;
```

**Consistency checks happen:**

1. **Before transaction starts** - Check current state
2. **During transaction** - Enforce constraints
3. **Before commit** - Final validation

---

### Isolation - Concurrency Control

**Implementation: Locking + MVCC**

**Pessimistic Locking (Traditional):**

```
Transaction T1:
BEGIN;
   SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- Locks row
   -- Other transactions wait here ⏳
   UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;  -- Lock released

Transaction T2 (blocked until T1 commits):
BEGIN;
   SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- Waits for T1
   UPDATE accounts SET balance = balance + 50 WHERE id = 1;
COMMIT;
```

**Optimistic Locking (MVCC - PostgreSQL):**

```sql
-- PostgreSQL creates new tuple version for each UPDATE
-- Each transaction sees a snapshot of data

T1: BEGIN; -- Snapshot at txn_id = 100
T1: SELECT balance FROM accounts WHERE id = 1;  -- Sees balance = 1000

T2: BEGIN; -- Snapshot at txn_id = 101
T2: UPDATE accounts SET balance = 900 WHERE id = 1;  -- Creates new version
T2: COMMIT;

T1: SELECT balance FROM accounts WHERE id = 1;  -- Still sees balance = 1000!
T1: COMMIT;

-- T1 sees consistent snapshot, no locks needed
```

**Internal structures:**

- **Lock tables** - Track who holds which locks
- **Lock queues** - Waiting transactions
- **Transaction ID** - MVCC snapshot versioning
- **Visibility map** - Which tuples are visible to which transactions

---

### Durability - Crash Recovery

**Implementation: Write-Ahead Logging (WAL)**

```
Write Path:
1. Application: UPDATE accounts SET balance = 500 WHERE id = 1;

2. Database writes to WAL (transaction log):
   [WAL Entry: txn_id=100, UPDATE accounts id=1, old=1000, new=500]
   ↓
3. fsync() to disk -- MUST complete before COMMIT returns ⚡
   ↓
4. Update in-memory buffer pool (dirty page)
   ↓
5. COMMIT returns to application ✅
   ↓
6. Later: Background process flushes dirty pages to data files

---

Crash Recovery:
1. Database restarts
2. Reads WAL from last checkpoint
3. Replays committed transactions
4. Rolls back uncommitted transactions
5. Database state restored ✅
```

**PostgreSQL WAL:**

```bash
# WAL files in pg_wal/
$ ls -lh /var/lib/postgresql/data/pg_wal/
-rw------- 1 postgres postgres 16M Feb 2 10:00 000000010000000000000001
-rw------- 1 postgres postgres 16M Feb 2 10:15 000000010000000000000002

# Configuration
wal_level = replica              # Amount of WAL data
fsync = on                       # Force writes to disk (NEVER turn off!)
synchronous_commit = on          # Wait for WAL flush before COMMIT returns
wal_buffers = 16MB              # WAL buffer size
checkpoint_timeout = 5min        # Automatic checkpoint interval
```

**MySQL InnoDB:**

```bash
# Redo log files (ib_logfile0, ib_logfile1)
$ ls -lh /var/lib/mysql/
-rw-rw---- 1 mysql mysql 48M Feb 2 10:00 ib_logfile0
-rw-rw---- 1 mysql mysql 48M Feb 2 10:15 ib_logfile1

# Configuration
innodb_flush_log_at_trx_commit = 1  # Flush log to disk on each commit (safest)
# = 0  Flush every second (fast but can lose 1 sec of data)
# = 2  Flush to OS cache, OS flushes to disk (middle ground)

innodb_log_file_size = 512M          # Redo log size
innodb_log_buffer_size = 16M         # Log buffer
```

---

## ✅ 4. Best Practices

### 1. Keep Transactions Short

```sql
-- ❌ Bad: Long-running transaction
BEGIN;
   SELECT * FROM large_table WHERE user_id = 123;  -- Returns 1M rows
   -- Application processes for 30 seconds...
   UPDATE user_stats SET last_login = NOW() WHERE id = 123;
COMMIT;

-- Holds locks for 30+ seconds, blocks other users!
```

```sql
-- ✅ Good: Minimal transaction scope
-- 1. Read data outside transaction
SELECT * FROM large_table WHERE user_id = 123;

-- 2. Process in application
-- ...

-- 3. Write in short transaction
BEGIN;
   UPDATE user_stats SET last_login = NOW() WHERE id = 123;
COMMIT;
-- Transaction lasts < 1ms
```

---

### 2. Use Appropriate Isolation Levels

```sql
-- ❌ Overkill: SERIALIZABLE for read-only analytics
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*) FROM page_views WHERE date = '2024-02-01';
-- Unnecessary overhead, no concurrent writes

-- ✅ Good: READ COMMITTED for analytics
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT COUNT(*) FROM page_views WHERE date = '2024-02-01';
-- Faster, sufficient consistency
```

---

### 3. Handle Transaction Failures

```python
# ✅ Good: Retry logic with exponential backoff
import time

def transfer_money(from_account, to_account, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            with db.transaction():
                # Debit
                db.execute(
                    "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, from_account)
                )

                # Credit
                db.execute(
                    "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to_account)
                )

                return True  # Success!

        except DeadlockError as e:
            if attempt < max_retries - 1:
                wait_time = (2 ** attempt) * 0.1  # Exponential backoff
                time.sleep(wait_time)
            else:
                raise  # Max retries exceeded

        except IntegrityError as e:
            # Don't retry constraint violations
            raise
```

---

### 4. Monitor Transaction Metrics

```sql
-- PostgreSQL: Long-running transactions
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

-- MySQL: Long-running transactions
SELECT * FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 30 SECOND
ORDER BY trx_started;
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Disabling fsync for Performance

```bash
# ❌ NEVER DO THIS IN PRODUCTION!
fsync = off  # PostgreSQL
innodb_flush_log_at_trx_commit = 0  # MySQL

# "I need more throughput!"
# → Crash happens
# → Committed data LOST
# → Database corrupted
# → Company bankrupt 💥
```

**Why it breaks:**

- OS caches writes in memory
- Crash before OS flushes to disk
- Data loss guaranteed

**Right solution:**

```bash
# ✅ Keep durability guarantees
fsync = on
synchronous_commit = on

# ✅ Optimize elsewhere:
- Batch multiple operations in single transaction
- Use connection pooling
- Add read replicas
- Upgrade hardware (SSD, more RAM)
- Scale horizontally
```

---

### Mistake 2: Forgetting to COMMIT

```python
# ❌ Bad: No explicit commit
def update_user(user_id, name):
    db.execute("UPDATE users SET name = %s WHERE id = %s", (name, user_id))
    # Forgot to commit!
    # Transaction hangs open, holds locks

# ✅ Good: Explicit transaction management
def update_user(user_id, name):
    with db.transaction():  # Auto-commits on exit
        db.execute("UPDATE users SET name = %s WHERE id = %s", (name, user_id))
```

---

### Mistake 3: Ignoring Transaction Boundaries

```python
# ❌ Bad: Transaction inside loop
for order_id in order_ids:  # 10,000 orders
    with db.transaction():
        db.execute("UPDATE orders SET status = 'shipped' WHERE id = %s", (order_id,))
# 10,000 commits! Huge overhead (fsync × 10,000)

# ✅ Good: Single transaction for batch
with db.transaction():
    for order_id in order_ids:
        db.execute("UPDATE orders SET status = 'shipped' WHERE id = %s", (order_id,))
# 1 commit! 100× faster
```

---

### Mistake 4: Mixing Transactional and Non-Transactional

```sql
-- ❌ Bad: MyISAM (non-transactional) mixed with InnoDB (transactional)
BEGIN;
   UPDATE orders SET status = 'paid';  -- InnoDB table
   UPDATE logs SET message = 'Payment processed';  -- MyISAM table (no rollback!)
ROLLBACK;  -- Orders rolled back, but logs NOT rolled back!

-- ✅ Good: Use transactional engines consistently
-- All tables use InnoDB in MySQL or PostgreSQL
```

---

## 🔐 6. Security Considerations

### 1. SQL Injection Prevention

```python
# ❌ Vulnerable: String concatenation
user_input = "1 OR 1=1; DROP TABLE accounts; --"
query = f"SELECT * FROM users WHERE id = {user_input}"
db.execute(query)  # 💥 SQL injection!

# ✅ Safe: Parameterized queries
user_input = "1 OR 1=1; DROP TABLE accounts; --"
db.execute("SELECT * FROM users WHERE id = %s", (user_input,))
# Treated as literal value, not executable SQL
```

---

### 2. Transaction Timeout

```sql
-- ✅ Set statement timeout to prevent long locks
SET statement_timeout = '30s';  -- PostgreSQL

BEGIN;
   UPDATE accounts SET balance = balance - 100 WHERE id = 1;
   -- If takes > 30s, automatic rollback
COMMIT;
```

---

### 3. Audit Logging

```sql
-- ✅ Track who made changes
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    operation VARCHAR(10),
    user_id INT,
    old_values JSONB,
    new_values JSONB,
    timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- Trigger on sensitive tables
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_values, new_values)
        VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), row_to_json(NEW));
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER accounts_audit
AFTER UPDATE ON accounts
FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

---

## 🚀 7. Performance Optimization Techniques

### 1. Batch Operations

```sql
-- ❌ Slow: Row-by-row
for user_id in user_ids:
    BEGIN;
       UPDATE users SET last_login = NOW() WHERE id = user_id;
    COMMIT;
-- 10,000 users = 10,000 commits = 10,000 × fsync overhead

-- ✅ Fast: Batch update
BEGIN;
   UPDATE users
   SET last_login = NOW()
   WHERE id IN (1, 2, 3, ..., 10000);  -- Or use temporary table
COMMIT;
-- 1 commit = 1 × fsync overhead (100× faster)
```

---

### 2. Read-Only Transactions

```sql
-- ✅ PostgreSQL: Declare read-only for better performance
BEGIN TRANSACTION READ ONLY;
   SELECT * FROM large_table WHERE date = '2024-02-01';
COMMIT;

-- Database can optimize:
- No undo log needed
- No locking overhead
- Can read from replicas
```

---

### 3. Savepoints for Partial Rollback

```sql
-- ✅ Advanced: Savepoints within transactions
BEGIN;
   UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Success

   SAVEPOINT before_risky_operation;

   UPDATE accounts SET balance = balance - 999999 WHERE id = 2;  -- Fails (negative balance)

   ROLLBACK TO SAVEPOINT before_risky_operation;  -- Roll back only this part

   UPDATE accounts SET balance = balance + 100 WHERE id = 3;  -- Continue transaction
COMMIT;
```

---

### 4. Avoid Hot Spots

```sql
-- ❌ Bad: Auto-increment primary key in distributed system
CREATE TABLE events (
    id SERIAL PRIMARY KEY,  -- All inserts contend for next ID
    data TEXT
);

-- ✅ Good: UUID or snowflake ID (distributed generation)
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data TEXT
);
-- No central contention point
```

---

## 🧪 8. Examples

### Example 1: E-Commerce Order Processing

```sql
-- ✅ Complete order placement with ACID guarantees
BEGIN TRANSACTION;

   -- 1. Create order
   INSERT INTO orders (user_id, total, status)
   VALUES (123, 99.99, 'pending')
   RETURNING id INTO @order_id;

   -- 2. Reserve inventory
   UPDATE inventory
   SET quantity = quantity - 1,
       reserved = reserved + 1
   WHERE product_id = 456
     AND quantity >= 1;  -- Atomic check and update

   IF ROW_COUNT() = 0 THEN
      ROLLBACK;  -- Out of stock
      SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Product out of stock';
   END IF;

   -- 3. Charge payment
   INSERT INTO payments (order_id, amount, status)
   VALUES (@order_id, 99.99, 'captured');

   -- 4. Update order status
   UPDATE orders SET status = 'confirmed' WHERE id = @order_id;

COMMIT;  -- All or nothing!
```

---

### Example 2: Banking Transfer

```sql
-- ✅ Money transfer with checks
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

   -- 1. Check source balance
   SELECT balance INTO @current_balance
   FROM accounts
   WHERE id = @source_account
   FOR UPDATE;  -- Lock row

   IF @current_balance < @transfer_amount THEN
      ROLLBACK;
      SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Insufficient funds';
   END IF;

   -- 2. Debit source
   UPDATE accounts
   SET balance = balance - @transfer_amount
   WHERE id = @source_account;

   -- 3. Credit destination
   UPDATE accounts
   SET balance = balance + @transfer_amount
   WHERE id = @destination_account;

   -- 4. Log transaction
   INSERT INTO transaction_log (from_account, to_account, amount, timestamp)
   VALUES (@source_account, @destination_account, @transfer_amount, NOW());

COMMIT;
```

---

### Example 3: Seat Reservation System

```sql
-- ✅ Prevent double-booking
BEGIN TRANSACTION;

   -- 1. Check seat availability
   SELECT status INTO @seat_status
   FROM seats
   WHERE event_id = 100 AND seat_number = 'A15'
   FOR UPDATE;  -- Lock seat row

   IF @seat_status != 'available' THEN
      ROLLBACK;
      SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Seat already reserved';
   END IF;

   -- 2. Reserve seat
   UPDATE seats
   SET status = 'reserved',
       reserved_by = @user_id,
       reserved_at = NOW()
   WHERE event_id = 100 AND seat_number = 'A15';

   -- 3. Create reservation record
   INSERT INTO reservations (user_id, event_id, seat_number)
   VALUES (@user_id, 100, 'A15');

COMMIT;
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: Financial Systems

**Requirements:**

- ACID compliance mandatory (regulatory)
- Audit trails required
- Zero data loss tolerance

**Implementation:**

```sql
-- Ledger system with dual-entry bookkeeping
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

   -- Every transaction has two entries (debit and credit)
   INSERT INTO ledger (account, type, amount, transaction_id)
   VALUES
      (@account_from, 'DEBIT', 100.00, @txn_id),
      (@account_to, 'CREDIT', 100.00, @txn_id);

   -- Invariant check: Sum of all ledger entries = 0
   IF (SELECT SUM(CASE WHEN type='DEBIT' THEN amount ELSE -amount END)
       FROM ledger) != 0 THEN
      ROLLBACK;
   END IF;

COMMIT;
```

---

### Use Case 2: Inventory Management

**Challenge:** Prevent overselling during flash sales

```sql
-- ✅ Optimistic locking with version column
BEGIN;

   SELECT quantity, version INTO @qty, @ver
   FROM inventory
   WHERE product_id = 123;

   IF @qty < 1 THEN
      ROLLBACK;
      RAISE 'Out of stock';
   END IF;

   -- Update only if version matches (no one else modified)
   UPDATE inventory
   SET quantity = quantity - 1,
       version = version + 1
   WHERE product_id = 123
     AND version = @ver;

   IF ROW_COUNT() = 0 THEN
      ROLLBACK;  -- Someone else bought last item
      RAISE 'Concurrent modification';
   END IF;

COMMIT;
```

---

### Use Case 3: Distributed Transactions (Saga Pattern)

**Challenge:** ACID across microservices

```python
# ✅ Saga pattern: Compensating transactions
def process_order():
    try:
        # Step 1: Reserve inventory
        inventory_service.reserve(product_id, quantity)

        # Step 2: Charge payment
        payment_service.charge(user_id, amount)

        # Step 3: Create shipment
        shipping_service.create(order_id)

    except InventoryError:
        # No compensation needed (first step)
        raise

    except PaymentError:
        # Compensate: Release inventory
        inventory_service.release(product_id, quantity)
        raise

    except ShippingError:
        # Compensate: Refund payment + release inventory
        payment_service.refund(user_id, amount)
        inventory_service.release(product_id, quantity)
        raise
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: Explain ACID properties with a real-world example.

**Answer:**

Imagine a bank transfer of $100 from Alice to Bob:

- **Atomicity:** Either both debit ($100 from Alice) and credit ($100 to Bob) happen, or neither. No "Alice debited but Bob not credited" scenario.
- **Consistency:** The total money in the bank remains constant (sum of all balances before = sum after).
- **Isolation:** If another transaction checks balances during the transfer, it sees either pre-transfer or post-transfer state, never "in-between" inconsistent state.
- **Durability:** Once Alice's app shows "Transfer complete", even if the bank's server crashes immediately, the transfer persists.

---

### Q2: How does a database guarantee atomicity?

**Answer:**

**Two mechanisms:**

1. **Undo logging:** Before modifying data, write "undo information" (old values) to transaction log. On rollback or crash, replay undo log backwards to restore original state.

2. **Shadow paging:** Write changes to new pages, keep old pages. On commit, atomically switch pointer to new pages. On rollback, discard new pages.

Most modern databases use undo logging with write-ahead logging (WAL).

---

### Q3: What's the difference between ACID and BASE?

**Answer:**

| Property         | ACID (RDBMS)                  | BASE (NoSQL)                  |
| ---------------- | ----------------------------- | ----------------------------- |
| **Consistency**  | Strong, immediate             | Eventual, delayed             |
| **Availability** | Lower (needs coordination)    | Higher (partition tolerance)  |
| **Durability**   | Guaranteed on commit          | Async replication             |
| **Use case**     | Financial, critical data      | Social media, analytics       |
| **Trade-off**    | Consistency over availability | Availability over consistency |

ACID: "I need this to be correct RIGHT NOW"  
BASE: "I can tolerate temporary inconsistency for speed/availability"

---

### Q4: Can you have ACID in a distributed system?

**Answer:**

**Technically yes, practically with trade-offs:**

**Approaches:**

1. **Two-Phase Commit (2PC):**
   - Coordinator asks all nodes: "Can you commit?"
   - If all say yes, coordinator tells them to commit
   - Problem: Blocking protocol, coordinator is single point of failure

2. **Paxos/Raft consensus:**
   - Used in Google Spanner, CockroachDB
   - Provides linearizability (strong consistency)
   - Cost: Higher latency (network round-trips), lower availability during partitions

3. **Saga pattern:**
   - Break transaction into local transactions with compensating actions
   - Eventual consistency, not true ACID

**Real-world:** Most distributed systems sacrifice strong consistency (use eventual consistency) for availability and partition tolerance (CAP theorem).

---

### Q5: How do you troubleshoot a slow transaction?

**Answer:**

**Step-by-step:**

1. **Identify long-running transactions:**

```sql
-- PostgreSQL
SELECT pid, now() - xact_start AS duration, state, query
FROM pg_stat_activity
WHERE state != 'idle' AND xact_start IS NOT NULL
ORDER BY duration DESC;
```

2. **Check for locks:**

```sql
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

3. **Analyze query plans:**

```sql
EXPLAIN ANALYZE <slow_query>;
```

4. **Common causes:**

- Missing indexes → Full table scans
- Lock contention → FOR UPDATE on hot rows
- Long transaction scope → Reduce transaction time
- Deadlocks → Retry logic needed

---

## 🧩 11. Design Patterns

### Pattern 1: Unit of Work

**Problem:** Managing transaction boundaries in application code

**Solution:**

```python
class UnitOfWork:
    def __enter__(self):
        self.conn = db.connect()
        self.conn.begin()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.conn.rollback()
        else:
            self.conn.commit()
        self.conn.close()

# Usage
with UnitOfWork() as uow:
    uow.conn.execute("INSERT INTO orders ...")
    uow.conn.execute("UPDATE inventory ...")
    # Automatic commit on success, rollback on exception
```

**Pros:**

- ✅ Clear transaction boundaries
- ✅ Automatic cleanup
- ✅ Exception-safe

**Cons:**

- ❌ One connection per transaction (not pooling-friendly)

---

### Pattern 2: Optimistic Locking

**Problem:** Minimize lock contention for high-concurrency workloads

**Solution:**

```sql
-- Add version column
ALTER TABLE products ADD COLUMN version INT DEFAULT 0;

-- Application code
BEGIN;
   SELECT price, version FROM products WHERE id = 123;
   -- User edits price in UI...

   UPDATE products
   SET price = 19.99,
       version = version + 1
   WHERE id = 123
     AND version = 0;  -- Only update if version unchanged

   IF ROW_COUNT() = 0 THEN
      ROLLBACK;
      RAISE 'Concurrent modification detected, please retry';
   END IF;
COMMIT;
```

**Pros:**

- ✅ No locks held during user think-time
- ✅ High concurrency

**Cons:**

- ❌ Requires retry logic
- ❌ Not suitable for frequent conflicts

**Use when:**

- Long user think-time (editing forms)
- Low conflict probability
- Read-heavy workload

---

### Pattern 3: Idempotent Operations

**Problem:** Retry-safe transactions

**Solution:**

```sql
-- Use idempotency key
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    idempotency_key VARCHAR(100) UNIQUE NOT NULL,
    amount DECIMAL(10,2),
    status VARCHAR(20)
);

-- Application
BEGIN;
   INSERT INTO payments (idempotency_key, amount, status)
   VALUES ('order-123-payment', 99.99, 'pending')
   ON CONFLICT (idempotency_key) DO NOTHING;

   -- Even if retried 100 times, only 1 payment created ✅
COMMIT;
```

**Pros:**

- ✅ Safe retries
- ✅ Prevents duplicate charges

**Use when:**

- External API calls (payment gateways)
- Network can fail mid-request

---

## 📚 Summary

**ACID guarantees:**

- ✅ **Atomicity:** All-or-nothing transactions
- ✅ **Consistency:** Data integrity preserved
- ✅ **Isolation:** Concurrent safety
- ✅ **Durability:** Committed data persists

**Trade-offs:**

- ✅ Data safety and correctness
- ❌ Lower throughput and higher latency
- ❌ Complex distributed implementation

**When to use ACID:**

- Financial systems
- Inventory management
- Booking/reservation systems
- Any system where correctness > speed

**When to relax ACID (use BASE):**

- Social media feeds
- Analytics dashboards
- Content delivery
- High-throughput logging

---

**Key takeaway:** ACID is not a checkbox—it's a spectrum. Understand the trade-offs and choose the right consistency level for your use case.
