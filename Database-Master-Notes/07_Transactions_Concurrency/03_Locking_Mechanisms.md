# Locking Mechanisms - Controlling Concurrent Access

## 1. Concept Explanation

**Database locks** are synchronization mechanisms that prevent conflicting concurrent operations on the same data. While MVCC handles most read scenarios without locks, certain operations (writes, schema changes) require explicit locking to maintain data integrity.

Think of locks like bathroom stalls:
```
Shared Lock (S): Multiple people can peek under the door (read)
Exclusive Lock (X): One person inside, door locked (write)
Intent Lock (IX): "I might need exclusive access to a stall" (table scan planning)
```

### Lock Types

**1. Shared Lock (S) - Read Lock:**
```sql
SELECT * FROM accounts WHERE id = 123 FOR SHARE;
-- Multiple transactions can hold shared locks simultaneously
-- Prevents writes (exclusive locks) until shared locks released
```

**2. Exclusive Lock (X) - Write Lock:**
```sql
UPDATE accounts SET balance = 900 WHERE id = 123;
-- Only one transaction can hold exclusive lock
-- Blocks all other locks (shared and exclusive)
```

**3. Row-Level Lock:**
```sql
SELECT * FROM accounts WHERE id = 123 FOR UPDATE;
-- Locks specific row
-- Other rows in table remain accessible
```

**4. Table-Level Lock:**
```sql
LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;
-- Locks entire table
-- Blocks all operations on table
```

**5. Intent Locks (Hierarchical):**
```
Intent locks signal intention to acquire row locks:
- IS (Intent Shared): Planning to acquire shared row locks
- IX (Intent Exclusive): Planning to acquire exclusive row locks
- SIX (Shared + Intent Exclusive): Reading table, will update some rows
```

### Lock Compatibility Matrix

```
               | S  | X  | IS | IX | SIX |
---------------|----|----|----|----|-----|
S (Shared)     | ✓  | ✗  | ✓  | ✗  | ✗   |
X (Exclusive)  | ✗  | ✗  | ✗  | ✗  | ✗   |
IS (Intent S)  | ✓  | ✗  | ✓  | ✓  | ✓   |
IX (Intent X)  | ✗  | ✗  | ✓  | ✓  | ✗   |
SIX            | ✗  | ✗  | ✓  | ✗  | ✗   |

✓ = Compatible (both can be held)
✗ = Incompatible (one must wait)
```

---

## 2. Why It Matters

### Production Impact: Lost Updates

**Without Proper Locking:**
```sql
-- Inventory: stock = 10

-- Session 1: Customer A buys 5 units
BEGIN;
SELECT stock INTO v_stock FROM inventory WHERE product_id = 123;  -- Reads: 10
-- Customer A decides to buy (takes 30 seconds)

-- Session 2: Customer B buys 8 units [concurrent]
BEGIN;
SELECT stock INTO v_stock FROM inventory WHERE product_id = 123;  -- Reads: 10 (same!)
UPDATE inventory SET stock = 10 - 8 WHERE product_id = 123;       -- stock = 2
COMMIT;

-- Session 1: Customer A completes purchase
UPDATE inventory SET stock = 10 - 5 WHERE product_id = 123;       -- stock = 5 (overwrites!)
COMMIT;

-- Final state: stock = 5
-- Expected: stock = -3 (oversold!) or error

-- Reality: Sold 13 units (5 + 8), but stock shows 5
-- Lost update: Customer B's purchase overwritten!
```

**With Proper Locking (SELECT FOR UPDATE):**
```sql
-- Session 1: Customer A
BEGIN;
SELECT stock FROM inventory WHERE product_id = 123 FOR UPDATE;  -- Locks row: 10
-- Customer A decides (takes 30 seconds)

-- Session 2: Customer B [concurrent]
BEGIN;
SELECT stock FROM inventory WHERE product_id = 123 FOR UPDATE;  -- BLOCKED! Waits...

-- Session 1: Completes
UPDATE inventory SET stock = stock - 5 WHERE product_id = 123;  -- stock = 5
COMMIT;  -- Releases lock

-- Session 2: Now acquires lock
SELECT stock FROM inventory WHERE product_id = 123 FOR UPDATE;  -- Reads: 5
UPDATE inventory SET stock = stock - 8 WHERE product_id = 123;  -- stock = -3 → ERROR!
ROLLBACK;

-- Final state: stock = 5 (correct!)
-- Customer B shown "Insufficient stock"
```

### Real Incident: Double-Booking

```
Company: Hotel reservation system
Issue: Same room booked twice on same night

Timeline:
10:00:00 - Guest A searches for availability
           SELECT * FROM rooms WHERE date = '2024-12-25' AND available = true;
           Result: Room 101 available

10:00:01 - Guest B searches for availability [concurrent]
           SELECT * FROM rooms WHERE date = '2024-12-25' AND available = true;
           Result: Room 101 available (same!)

10:00:30 - Guest A books Room 101
           UPDATE rooms SET available = false, guest_id = A WHERE room_id = 101;
           COMMIT;

10:00:31 - Guest B books Room 101 [no lock!]
           UPDATE rooms SET available = false, guest_id = B WHERE room_id = 101;
           COMMIT;  -- Overwrites Guest A!

Result:
- Guest A: Confirmation email sent ✓
- Guest B: Confirmation email sent ✓
- System shows: Room 101 booked by Guest B only
- Guest A arrives: "No reservation found"
- Cost: Free night + $500 compensation + 1-star review

Root cause: No row lock during reservation

Fix:
BEGIN;
SELECT * FROM rooms 
WHERE date = '2024-12-25' AND room_id = 101 
FOR UPDATE;  -- Lock row

IF available THEN
    UPDATE rooms SET available = false, guest_id = ? WHERE room_id = 101;
    COMMIT;
ELSE
    ROLLBACK;
    RETURN "Room no longer available";
END IF;

Result: Guest B gets "Room unavailable" → Prevented double-booking
```

---

## 3. Internal Working

### Lock Acquisition Flow

```
Transaction requests lock:
    |
    v
Check lock table for conflicts
    |
    ├─ No conflict? → Grant lock immediately
    |                  Update lock table
    |                  Continue execution
    |
    └─ Conflict? → Add to wait queue
                   Sleep (release CPU)
                   ↓
                   Holder releases lock
                   ↓
                   Wake up from queue (FIFO)
                   ↓
                   Check conflicts again
                   ↓
                   Grant lock if no new conflicts
```

### PostgreSQL Lock Implementation

**Lock Storage (in shared memory):**
```c
// Simplified structure
typedef struct LOCK {
    LOCKTAG tag;          // Lock identifier (table/row)
    LockMode mode;        // S, X, IS, IX, SIX
    int nRequested;       // Requests pending
    int nGranted;         // Locks granted
    PROCLOCK *waiters;    // Queue of waiting transactions
} LOCK;

// Example
Lock on row (table=accounts, row=123):
  mode = X (Exclusive)
  nGranted = 1 (Transaction TXN-100 holds it)
  waiters = [TXN-101, TXN-102]  // 2 transactions waiting
```

**Lock Wait Queue (FIFO):**
```
Waiting for lock on accounts row 123:

Queue: [TXN-101] -> [TXN-102] -> [TXN-103]
       ↑ Next to acquire lock when TXN-100 releases

When TXN-100 commits:
1. Release exclusive lock
2. Wake up TXN-101 (queue head)
3. TXN-101 acquires lock
4. TXN-102, TXN-103 continue waiting
```

### Lock Escalation (SQL Server)

**Problem: Too Many Row Locks**
```sql
-- Update 100K rows
UPDATE orders SET processed = true WHERE date < '2024-01-01';

-- Naive approach: 100K row locks
-- Memory: 100K × 64 bytes = 6.4 MB per transaction
-- System limit: 5 million locks → 80 concurrent transactions max!
```

**Solution: Lock Escalation**
```
Lock hierarchy:
  Database
    ↓
  Table
    ↓
  Page (8KB, ~100 rows)
    ↓
  Row

Escalation triggers:
- More than 5000 row locks → Escalate to table lock
- More than 40% of table locked → Escalate to table lock
- Lock memory > threshold → Escalate

Example:
1. Transaction starts: Row locks (rows 1-100)
2. Threshold exceeded: Escalate to table lock
3. Block granularity: Entire table (99% not accessed!)
4. Concurrency impact: All other transactions blocked
```

**Prevent Escalation:**
```sql
-- Disable lock escalation for specific table
ALTER TABLE orders SET (LOCK_ESCALATION = DISABLE);

-- Or: Batch updates
BEGIN;
UPDATE orders SET processed = true 
WHERE id BETWEEN 1 AND 1000;  -- Small batch (no escalation)
COMMIT;

BEGIN;
UPDATE orders SET processed = true 
WHERE id BETWEEN 1001 AND 2000;
COMMIT;
-- Repeat...
```

---

## 4. Best Practices

### Practice 1: Always Use SELECT FOR UPDATE for Critical Sections

**❌ Bad: Read then Update (Race Condition):**
```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 123;  -- balance = 1000
-- Application logic (takes 5 seconds)
IF balance >= 100 THEN
    UPDATE accounts SET balance = balance - 100 WHERE id = 123;
END IF;
COMMIT;

-- Problem: Another transaction can modify balance during "application logic"
-- Result: Overdraft possible
```

**✅ Good: Lock During Read:**
```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 123 FOR UPDATE;  -- Lock row: 1000
-- Application logic (takes 5 seconds)
-- No other transaction can modify balance (locked!)
IF balance >= 100 THEN
    UPDATE accounts SET balance = balance - 100 WHERE id = 123;
END IF;
COMMIT;  -- Release lock

-- Guaranteed: Balance check and update are atomic
```

### Practice 2: Lock in Consistent Order (Prevent Deadlocks)

**❌ Bad: Inconsistent Lock Order:**
```sql
-- Transaction A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Lock account 1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Lock account 2
COMMIT;

-- Transaction B [concurrent]
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;   -- Lock account 2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- Lock account 1 → DEADLOCK!
COMMIT;

-- Deadlock: A waits for B, B waits for A → Cycle!
```

**✅ Good: Consistent Lock Order (By ID):**
```sql
-- Transaction A
BEGIN;
-- Lock smaller ID first
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Lock 1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Lock 2
COMMIT;

-- Transaction B [concurrent]
BEGIN;
-- Lock smaller ID first (same order!)
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- Waits for A's lock 1
-- After A commits, B acquires lock 1, then lock 2
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
COMMIT;

-- No deadlock! Both lock in same order (1 → 2)
```

### Practice 3: Use NOWAIT to Avoid Blocking

**Without NOWAIT (Blocks Indefinitely):**
```sql
BEGIN;
SELECT * FROM accounts WHERE id = 123 FOR UPDATE;
-- If row locked → Waits forever (or until statement_timeout)
-- User frustrated, abandons action
```

**With NOWAIT (Fails Immediately):**
```sql
BEGIN;
SELECT * FROM accounts WHERE id = 123 FOR UPDATE NOWAIT;
-- If row locked → ERROR immediately: "could not obtain lock"
-- Application can show: "Resource busy, try again"
-- Better UX: User informed immediately

-- Application retry logic
FOR attempt IN 1..3 LOOP
    BEGIN
        SELECT * FROM accounts WHERE id = 123 FOR UPDATE NOWAIT;
        -- Success: Acquired lock
        EXIT;
    EXCEPTION WHEN lock_not_available THEN
        IF attempt < 3 THEN
            PERFORM pg_sleep(0.1);  -- Wait 100ms
            CONTINUE;  -- Retry
        ELSE
            RAISE;  -- Max retries, give up
        END IF;
    END;
END LOOP;
```

### Practice 4: Lock Only What You Need

**❌ Bad: Lock Entire Table:**
```sql
BEGIN;
LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;  -- Blocks ALL queries!
UPDATE accounts SET balance = balance + 10 WHERE id = 123;
COMMIT;

-- Problem: Entire table locked for single-row update
-- Impact: All concurrent queries blocked
```

**✅ Good: Lock Specific Row:**
```sql
BEGIN;
SELECT * FROM accounts WHERE id = 123 FOR UPDATE;  -- Lock only row 123
UPDATE accounts SET balance = balance + 10 WHERE id = 123;
COMMIT;

-- Impact: Only queries targeting row 123 blocked
-- Other rows: Fully concurrent access
```

---

## 5. Common Mistakes

### Mistake 1: Forgetting WHERE Clause (Locks Entire Table)

**Problem:**
```sql
-- Intended: Lock account 123
SELECT * FOR UPDATE;  -- Missing WHERE!

-- Reality: Locks ALL rows in table!
-- 10M accounts × 1 transaction = System freeze

-- Monitoring shows:
SELECT COUNT(*) FROM pg_locks WHERE granted = true;
-- Result: 10,000,000 locks (!)

-- All other queries blocked:
SELECT * FROM accounts WHERE id = 456;  -- Waits indefinitely
```

**Fix:**
```sql
-- Always specify WHERE clause
SELECT * FROM accounts WHERE id = 123 FOR UPDATE;

-- Lock only what you need
```

### Mistake 2: Holding Locks During External API Calls

**Problem:**
```sql
BEGIN;

-- Step 1: Lock inventory
SELECT stock FROM inventory WHERE product_id = 123 FOR UPDATE;

-- Step 2: Call external payment API (takes 5 seconds)
result = call_payment_api(customer_id, amount);

-- Step 3: Deduct inventory
IF result.status = 'success' THEN
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 123;
END IF;

COMMIT;  -- Hold lock for 5+ seconds!

-- Problem:
-- - Lock held during network I/O (slow!)
-- - If API timeout (30s) → Lock held 30 seconds
-- - All concurrent purchases blocked
-- - Throughput: 1 order/5s = 720 orders/hour (vs 10K+/hour expected)
```

**Fix: Two-Phase Approach:**
```sql
-- Phase 1: Reserve inventory (fast, < 100ms)
BEGIN;
SELECT stock FROM inventory WHERE product_id = 123 FOR UPDATE;
IF stock > 0 THEN
    INSERT INTO reservations (product_id, customer_id, expires_at)
    VALUES (123, 456, NOW() + INTERVAL '5 minutes');
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 123;
END IF;
COMMIT;  -- Release lock (fast!)

-- Phase 2: Process payment (outside transaction)
result = call_payment_api(customer_id, amount);

-- Phase 3: Finalize or release reservation
BEGIN;
IF result.status = 'success' THEN
    UPDATE reservations SET status = 'confirmed' WHERE id = reservation_id;
ELSE
    -- Payment failed: Return inventory
    DELETE FROM reservations WHERE id = reservation_id;
    UPDATE inventory SET stock = stock + 1 WHERE product_id = 123;
END IF;
COMMIT;

-- Benefits:
-- - Lock held < 100ms (vs 5+ seconds)
-- - Throughput: 10K orders/hour ✓
```

---

## 6. Security Considerations

### 1. Lock-Based DoS Attack

**Attack:**
```sql
-- Attacker opens transaction and locks critical table
BEGIN;
SELECT * FROM products FOR UPDATE;  -- Lock all products!
-- Never commit (idle transaction)

-- Legitimate users:
SELECT * FROM products WHERE id = 123;  -- BLOCKED indefinitely
-- Website appears down: "Loading..."
```

**Mitigation:**
```sql
-- Set idle transaction timeout
ALTER DATABASE mydb SET idle_in_transaction_session_timeout = '5min';

-- Attacker's transaction killed after 5 minutes
-- Locks released automatically

-- Monitor long-running transactions
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
    AND xact_start < now() - INTERVAL '1 minute';

-- Kill suspicious transactions
SELECT pg_terminate_backend(pid);
```

---

## 7. Performance Optimization

### Optimization 1: Skip Locked (Process Available Rows)

**Problem: Job Queue Processing**
```sql
-- Worker 1
BEGIN;
SELECT * FROM job_queue WHERE status = 'pending' LIMIT 1 FOR UPDATE;
-- Processes job (takes 10 seconds)
COMMIT;

-- Worker 2 [concurrent]
SELECT * FROM job_queue WHERE status = 'pending' LIMIT 1 FOR UPDATE;
-- BLOCKED! Waits for Worker 1 even though other jobs available
```

**Solution: SKIP LOCKED**
```sql
-- Worker 1
BEGIN;
SELECT * FROM job_queue 
WHERE status = 'pending' 
LIMIT 1 
FOR UPDATE SKIP LOCKED;  -- Skip locked rows, grab next available
-- Gets job ID 100, processes it
COMMIT;

-- Worker 2 [concurrent]
SELECT * FROM job_queue 
WHERE status = 'pending' 
LIMIT 1 
FOR UPDATE SKIP LOCKED;  -- Skips job 100 (locked by Worker 1)
-- Gets job ID 101 immediately (no wait!)
COMMIT;

-- Benefits:
-- - No blocking (workers process in parallel)
-- - Throughput: 10 workers × 10 jobs/sec = 100 jobs/sec
-- - vs blocking: 1 job/sec (sequential)
```

---

## 8. Examples

### Example 1: Ticket Sales (Prevent Overbooking)

```sql
CREATE OR REPLACE FUNCTION buy_ticket(
    p_event_id INT,
    p_customer_id INT,
    p_quantity INT
) RETURNS BOOLEAN AS $$
DECLARE
    v_available INT;
BEGIN
    -- Lock event row (prevent concurrent sales)
    SELECT tickets_available INTO v_available
    FROM events
    WHERE id = p_event_id
    FOR UPDATE;  -- Critical: Lock row!
    
    -- Check availability
    IF v_available < p_quantity THEN
        RAISE EXCEPTION 'Only % tickets available', v_available;
    END IF;
    
    -- Reserve tickets
    UPDATE events
    SET tickets_available = tickets_available - p_quantity
    WHERE id = p_event_id;
    
    -- Create order
    INSERT INTO orders (event_id, customer_id, quantity)
    VALUES (p_event_id, p_customer_id, p_quantity);
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Usage: 10 concurrent customers buying last 5 tickets
-- First 1 succeeds, rest get "Only 0 tickets available"
```

---

## 9. Real-World Use Cases

### Use Case 1: Banking - Concurrent Withdrawals

```sql
-- Two ATMs withdraw simultaneously from same account

-- ATM 1
BEGIN;
SELECT balance FROM accounts WHERE id = 123 FOR UPDATE;  -- Lock: $1000
-- Customer withdraws $800
UPDATE accounts SET balance = balance - 800 WHERE id = 123;
COMMIT;

-- ATM 2 [concurrent, 1 second later]
BEGIN;
SELECT balance FROM accounts WHERE id = 123 FOR UPDATE;  -- Waits for ATM 1's lock
-- After ATM 1 commits: Reads balance = $200
-- Customer tries to withdraw $500 → ERROR: Insufficient funds
ROLLBACK;

-- Result: No overdraft! Lock prevented race condition
```

---

## 10. Interview Questions

### Q1: What is the difference between shared locks and exclusive locks? When would you use each?

**Answer:**

**Shared Locks (S):**
- **Purpose:** Allow multiple readers, block writers
- **Use case:** SELECT statements that need repeatable reads
- **Example:**
```sql
SELECT * FROM inventory WHERE product_id = 123 FOR SHARE;
-- Multiple transactions can read simultaneously
-- No transaction can update until all shared locks released
```

**Exclusive Locks (X):**
- **Purpose:** Single writer, block all others (readers + writers)
- **Use case:** UPDATE, DELETE, INSERT operations
- **Example:**
```sql
UPDATE inventory SET stock = stock - 1 WHERE product_id = 123;
-- Acquires exclusive lock automatically
-- Blocks all concurrent reads and writes
```

**Key Difference:**
- Shared: N readers, 0 writers
- Exclusive: 1 writer, 0 readers/writers

**Interview insight:** "In high-traffic systems, I use shared locks sparingly (they block writes). For read-heavy workloads, MVCC snapshots + SELECT (no locks) achieve better concurrency. Reserve FOR SHARE for critical sections requiring repeatable reads across multiple statements."

---

## 11. Summary

**Lock Fundamentals:**
- **Shared (S):** Multiple readers
- **Exclusive (X):** Single writer
- **Row vs Table:** Granularity matters
- **FOR UPDATE:** Explicit row lock

**Best Practices:**
- Lock only what you need (WHERE clause!)
- Lock in consistent order (prevent deadlocks)
- Use NOWAIT (fail fast)
- Use SKIP LOCKED (job queues)
- Release locks quickly (no external API calls)

**Critical Pattern:**
```sql
BEGIN;
SELECT * FROM resource WHERE id = ? FOR UPDATE;  -- Lock first
-- Validate
-- Modify
COMMIT;  -- Release lock
```

**Key Takeaway:** "Locks are necessary for write consistency but hurt concurrency. Use explicit locking (FOR UPDATE) only when MVCC snapshots aren't enough (read-modify-write patterns). In production: Always specify WHERE clauses, lock in consistent order, and monitor lock wait times."

---

**Next:** [04_Deadlocks.md](04_Deadlocks.md) - Detection, prevention, resolution
