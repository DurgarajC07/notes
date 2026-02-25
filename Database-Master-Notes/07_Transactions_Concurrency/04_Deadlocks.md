# Deadlocks - When Transactions Wait Forever

## 1. Concept Explanation

A **deadlock** occurs when two or more transactions wait for each other to release locks, creating a circular dependency where none can proceed.

**Classic Example:**
```
Transaction A holds Lock 1, wants Lock 2
Transaction B holds Lock 2, wants Lock 1

A waits for B to release Lock 2
B waits for A to release Lock 1
→ Both wait forever (deadlock!)
```

**Visual:**
```
     [Transaction A]
          |
    Holds Lock 1
          |
    Wants Lock 2
          ↓
    ← [WAIT] ←
    ↓          ↑
[Lock 2]    [Lock 1]
    ↑          ↓
    → [WAIT] →
          ↑
    Wants Lock 1
          |
    Holds Lock 2
          |
     [Transaction B]

Circular wait = Deadlock!
```

### Four Conditions for Deadlock (Coffman Conditions)

1. **Mutual Exclusion:** Only one transaction can hold exclusive lock
2. **Hold and Wait:** Transaction holds lock while waiting for another
3. **No Preemption:** Locks cannot be forcibly taken
4. **Circular Wait:** Cycle in wait-for graph

**All four must be true for deadlock to occur.**

---

## 2. Why It Matters

### Production Impact: Payment Processing Freeze

```
Company: E-commerce platform
Issue: Payment processing deadlock during Black Friday sale

Timeline:
14:00:00 - Black Friday sale begins (1000 orders/minute)

14:15:23 - First deadlock detected
Transaction A (Order 1001):
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 100;  -- Lock product 100
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 200;  -- Wants lock product 200

Transaction B (Order 1002):
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 200;  -- Lock product 200
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 100;  -- Wants lock product 100

Deadlock: A waits for B, B waits for A

14:15:24 - Database detects deadlock after 1 second
          - Aborts Transaction B (victim)
          - Transaction A completes successfully

14:15:24 to 14:45:00 - Deadlock cascade
    - 10K deadlocks in 30 minutes
    - Each deadlock: 1 second wait + abort + retry
    - Customer impact: 3-5 second page load
    - Retry logic causes: 2× load spike

14:45:00 - Emergency fix deployed
    - Sort product IDs before locking (consistent order)
    - Deadlocks drop to 0 immediately

Cost:
- Revenue lost (30 min × 1000 orders/min × $50 × 20% abandon) = $300K
- Customer support (1000 tickets × 5 min × $15/hour) = $1.25K
- Engineering response ($50K/year × 4 engineers × 4 hours) = $400
- Total: ~$302K
```

**Root cause:** Inconsistent lock order (product IDs not sorted)

---

## 3. Internal Working

### Deadlock Detection Algorithm

**Wait-For Graph (WFG):**

Nodes = Transactions
Edges = "Waits for" relationship

```
Example:
TXN-A → TXN-B: A waits for lock held by B
TXN-B → TXN-C: B waits for lock held by C

If graph contains cycle → Deadlock!
```

**Cycle Detection:**
```
Initial state:
TXN-A locks Row 1, wants Row 2
TXN-B locks Row 2, wants Row 1

Wait-For Graph:
  TXN-A → TXN-B (A waits for B)
  TXN-B → TXN-A (B waits for A)

Cycle detected: A → B → A

Database response:
1. Choose victim (transaction with least cost)
2. Abort victim (ROLLBACK)
3. Release victim's locks
4. Wake up remaining transaction
```

**PostgreSQL Implementation:**
```c
// Simplified deadlock detection
void DeadlockCheck() {
    // Run every deadlock_timeout (default: 1 second)
    if (time_since_lock_wait() > deadlock_timeout) {
        BuildWaitForGraph();
        if (CycleDetected()) {
            // Choose victim
            PGPROC *victim = ChooseVictim();
            // Abort transaction
            AbortTransaction(victim);
            // Release locks
            ReleaseLocks(victim);
        }
    }
}
```

### PostgreSQL Deadlock Detection

**Configuration:**
```sql
-- deadlock_timeout: How long to wait before checking for deadlock
SHOW deadlock_timeout;  -- Default: 1s

-- Why not check immediately?
-- - Checking for cycles is expensive (O(n²)Graph traversal)
-- - Most lock waits resolve quickly (< 100ms)
-- - Wait 1 second: Avoid expensive checks for short waits

-- Adjust for high-contention workloads
ALTER DATABASE mydb SET deadlock_timeout = '100ms';
```

**Deadlock Log:**
```
2024-01-15 10:23:45 UTC [12345] ERROR:  deadlock detected
2024-01-15 10:23:45 UTC [12345] DETAIL:  Process 12345 waits for ShareLock on transaction 1001; blocked by process 12346.
Process 12346 waits for ShareLock on transaction 1002; blocked by process 12345.
2024-01-15 10:23:45 UTC [12345] HINT:  See server log for query details.
2024-01-15 10:23:45 UTC [12345] CONTEXT:  while updating tuple (0,10) in relation "accounts"
2024-01-15 10:23:45 UTC [12345] STATEMENT:  UPDATE accounts SET balance = balance - 100 WHERE id = 2;
```

---

## 4. Best Practices

### Practice 1: Lock Resources in Consistent Order

**❌ Bad: Random Lock Order:**
```python
# Transfer money from account1 to account2
def transfer(account1_id, account2_id, amount):
    conn.execute(f"UPDATE accounts SET balance = balance - {amount} WHERE id = {account1_id}")
    conn.execute(f"UPDATE accounts SET balance = balance + {amount} WHERE id = {account2_id}")
    conn.commit()

# Deadlock scenario:
# Transfer A→B: Lock A, then Lock B
# Transfer B→A: Lock B, then Lock A (reverse order!)
# → Deadlock!
```

**✅ Good: Consistent Lock Order (Sort IDs):**
```python
def transfer(account1_id, account2_id, amount):
    # Always lock smaller ID first
    from_id, to_id = sorted([account1_id, account2_id])
    
    if account1_id == from_id:
        debit_id, credit_id = account1_id, account2_id
        debit_amt, credit_amt = amount, -amount
    else:
        debit_id, credit_id = account2_id, account1_id
        debit_amt, credit_amt = -amount, amount
    
    # Lock in consistent order (by ID)
    conn.execute(f"""
        UPDATE accounts SET balance = balance + CASE
            WHEN id = {from_id} THEN {debit_amt}
            WHEN id = {to_id} THEN {credit_amt}
        END
        WHERE id IN ({from_id}, {to_id})
        ORDER BY id;  -- Critical: Order by ID!
    """)
    conn.commit()

# Now:
# Transfer A→B: Lock A (smaller), then Lock B
# Transfer B→A: Lock A (smaller), then Lock B (same order!)
# → No deadlock!
```

### Practice 2: Use Timeouts with Retry Logic

**Without Timeout:**
```sql
-- Wait forever if deadlock undetected
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Hangs indefinitely if victim not chosen
```

**With lock_timeout:**
```sql
SET lock_timeout = '5s';

BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- If lock not acquired in 5 seconds → ERROR: "lock timeout"
COMMIT;
```

**Application Retry Logic:**
```python
import psycopg2
import time

def transfer_with_retry(account1_id, account2_id, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            conn = psycopg2.connect(...)
            cursor = conn.cursor()
            
            # Set timeout
            cursor.execute("SET lock_timeout = '5s'")
            
            # Perform transfer (consistent lock order)
            cursor.execute("""
                UPDATE accounts 
                SET balance = balance + CASE
                    WHEN id = %s THEN -%s
                    WHEN id = %s THEN %s
                END
                WHERE id IN (%s, %s)
                ORDER BY id
            """, (account1_id, amount, account2_id, amount, account1_id, account2_id))
            
            conn.commit()
            return True  # Success
            
        except psycopg2.errors.DeadlockDetected as e:
            conn.rollback()
            if attempt < max_retries - 1:
                # Exponential backoff: 100ms, 200ms, 400ms
                wait_time = 0.1 * (2 ** attempt)
                print(f"Deadlock detected, retry {attempt + 1}/{max_retries} in {wait_time}s")
                time.sleep(wait_time)
            else:
                raise  # Max retries exceeded
        
        except psycopg2.errors.LockNotAvailable as e:
            conn.rollback()
            raise Exception(f"Lock timeout after {max_retries} retries")
    
    return False
```

### Practice 3: Keep Transactions Short

**❌ Bad: Long Transaction with Multiple Locks:**
```sql
BEGIN;

-- Lock 100 products
UPDATE inventory SET stock = stock - 1 WHERE product_id IN (1, 2, ..., 100);

-- Process payment (takes 5 seconds, external API call)
-- Meanwhile: Other transactions try to update same products → Deadlock risk!

-- Send confirmation email (takes 2 seconds)

COMMIT;  -- Hold locks for 7+ seconds!
```

**✅ Good: Short Transactions, External Calls Outside:**
```sql
-- Step 1: Reserve inventory (fast, < 100ms)
BEGIN;
UPDATE inventory SET stock = stock - 1 WHERE product_id IN (1, 2, ..., 100);
INSERT INTO reservations (order_id, products) VALUES (...);
COMMIT;  -- Release locks quickly!

-- Step 2: Process payment (outside transaction, no locks held)
result = call_payment_api();

-- Step 3: Confirm or rollback reservation
BEGIN;
IF result.success THEN
    UPDATE reservations SET status = 'confirmed' WHERE order_id = ...;
ELSE
    UPDATE inventory SET stock = stock + 1 WHERE product_id IN (...);
    DELETE FROM reservations WHERE order_id = ...;
END IF;
COMMIT;

-- Benefits:
-- - Locks held < 100ms per transaction (vs 7 seconds)
-- - Reduced deadlock risk (fewer concurrent long-held locks)
```

---

## 5. Common Mistakes

### Mistake 1: Implicit Lock Order (Hidden Deadlock)

**Problem:**
```sql
-- Application code looks safe:
BEGIN;
UPDATE users SET last_login = NOW() WHERE id = 100;  -- Lock user 100
UPDATE sessions SET expires_at = NOW() + INTERVAL '1 hour' WHERE user_id = 100;
COMMIT;

-- But... table has foreign key with ON UPDATE CASCADE:
ALTER TABLE sessions ADD CONSTRAINT fk_user 
    FOREIGN KEY (user_id) REFERENCES users(id) ON UPDATE CASCADE;

-- Hidden behavior:
-- UPDATE users → Locks users(100)
-- ON UPDATE CASCADE → Locks sessions(user_id=100) [implicit!]
-- UPDATE sessions → Tries to lock sessions again [self-deadlock!]

-- Deadlock log:
-- Process waits for RowExclusiveLock on relation "sessions"
-- blocked by itself!
```

**Fix:**
```sql
-- Remove CASCADE (explicit control)
ALTER TABLE sessions DROP CONSTRAINT fk_user;
ALTER TABLE sessions ADD CONSTRAINT fk_user 
    FOREIGN KEY (user_id) REFERENCES users(id);  -- No CASCADE

-- Explicit updates (predictable lock order)
BEGIN;
UPDATE users SET last_login = NOW() WHERE id = 100;
UPDATE sessions SET expires_at = NOW() + INTERVAL '1 hour' WHERE user_id = 100;
COMMIT;
```

### Mistake 2: Index Scans Locking Wrong Rows

**Problem:**
```sql
-- Table: accounts(id, email, balance)
-- Index: idx_email on email

-- Transaction A
BEGIN;
UPDATE accounts SET balance = balance - 100 
WHERE email = 'alice@example.com';  -- Uses idx_email

-- Behind the scenes:
-- 1. Index scan on email → Locks index entry 'alice@example.com'
-- 2. Fetch row → Locks row with id=123
-- 3. Update row

-- Transaction B [concurrent]
BEGIN;
UPDATE accounts SET balance = balance + 50 
WHERE email = 'bob@example.com';  -- Uses idx_email

-- Behind the scenes:
-- 1. Index scan on email → Locks index entry 'bob@example.com'
-- 2. If 'alice@' and 'bob@' entries are on SAME INDEX PAGE
--    → Locks overlap → Deadlock risk!
```

**Why:**
Index pages hold multiple entries. Locking entry 'alice@...' may lock page containing 'bob@...', causing false conflicts.

**Fix:**
```sql
-- Use primary key when possible (row-level locking)
UPDATE accounts SET balance = balance - 100 WHERE id = 123;

-- Or: Use row-level locking explicitly
SELECT id FROM accounts WHERE email = 'alice@example.com' FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = ?;
```

---

## 6. Security Considerations

### Denial of Service via Deadlock Storm

**Attack:**
```python
# Attacker script: Create intentional deadlocks
import threading

def attack_thread(id1, id2):
    # Transfer in random order (guaranteed deadlocks)
    transfer(id1, id2, 1.00)

# Spawn 1000 threads
for i in range(1000):
    t1 = threading.Thread(target=attack_thread, args=(1, 2))
    t2 = threading.Thread(target=attack_thread, args=(2, 1))
    t1.start()
    t2.start()

# Result:
# - 500+ deadlocks per second
# - CPU 100% (deadlock detection overhead)
# - Legitimate transactions blocked
# - Database effectively DoS'd
```

**Mitigation:**
```sql
-- 1. Rate limiting per user
CREATE TABLE request_log (
    user_id INT,
    timestamp TIMESTAMP DEFAULT NOW()
);

-- Rate limit: 10 requests/second
SELECT COUNT(*) FROM request_log 
WHERE user_id = ? AND timestamp > NOW() - INTERVAL '1 second';
-- If > 10 → REJECT

-- 2. Statement timeout (prevent infinite lock waits)
ALTER DATABASE mydb SET statement_timeout = '10s';

-- 3. Monitor deadlock rate
SELECT COUNT(*) FROM pg_stat_database WHERE deadlocks > 100;
-- Alert if excessive

-- 4. Block abusive IPs
-- Application-level firewall rule
```

---

## 7. Performance Optimization

### Optimization 1: Partition Hot Tables

**Problem:**
```sql
-- Hot table: 1M updates/second, concentrated on few rows
UPDATE counters SET count = count + 1 WHERE page_id = 'homepage';

-- All updates hit same row → High lock contention → Deadlocks
```

**Solution: Shard Updates:**
```sql
-- Create multiple counter rows per page
CREATE TABLE counters (
    page_id VARCHAR,
    shard_id INT,  -- 0-9 (10 shards)
    count BIGINT,
    PRIMARY KEY (page_id, shard_id)
);

-- Update random shard
UPDATE counters 
SET count = count + 1 
WHERE page_id = 'homepage' 
    AND shard_id = FLOOR(RANDOM() * 10);

-- Read: Sum all shards
SELECT page_id, SUM(count) AS total_count
FROM counters
WHERE page_id = 'homepage'
GROUP BY page_id;

-- Benefits:
-- - Lock contention reduced 10× (10 shards)
-- - Deadlocks reduced 10× (fewer collisions)
-- - Throughput: 10K updates/sec (vs 1K before)
```

---

## 8. Examples

### Example 1: Bank Transfer with Deadlock Prevention

```sql
CREATE OR REPLACE FUNCTION safe_transfer(
    p_from_account INT,
    p_to_account INT,
    p_amount NUMERIC
) RETURNS BOOLEAN AS $$
DECLARE
    v_lock_order INT[];
BEGIN
    -- Step 1: Lock accounts in consistent order (by ID)
    v_lock_order := ARRAY[LEAST(p_from_account, p_to_account), 
                          GREATEST(p_from_account, p_to_account)];
    
    -- Step 2: Lock both accounts (ordered)
    PERFORM balance FROM accounts 
    WHERE id = ANY(v_lock_order)
    ORDER BY id  -- Critical: Consistent order!
    FOR UPDATE;
    
    -- Step 3: Validate from_account balance
    IF (SELECT balance FROM accounts WHERE id = p_from_account) < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds';
    END IF;
    
    -- Step 4: Perform transfer
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from_account;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to_account;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Usage:
BEGIN;
SELECT safe_transfer(123, 456, 100.00);  -- Account 123 → 456
COMMIT;

-- Concurrent transfer (reverse direction):
BEGIN;
SELECT safe_transfer(456, 123, 50.00);   -- Account 456 → 123
-- Both transactions lock accounts in SAME order (123, then 456)
-- → No deadlock!
COMMIT;
```

---

## 9. Real-World Use Cases

### Use Case: Uber Trip Matching

**Problem:** Drivers and riders matched concurrently

```
Scenario:
- Driver A available
- Driver B available
- Rider 1 requests ride
- Rider 2 requests ride

Matching algorithm:
Thread 1: Assign Driver A to Rider 1
    1. Lock Driver A
    2. Lock Rider 1
    3. Create trip

Thread 2: Assign Driver B to Rider 2 [concurrent]
    1. Lock Driver B
    2. Lock Rider 2
    3. Create trip

No deadlock (different resources)

But complex scenario:
Thread 1: Assign Driver A to Rider 1
    1. Lock Driver A
    2. Check Driver B availability (backup)
    3. Lock Driver B [waiting...]

Thread 2: Assign Driver B to Rider 2
    1. Lock Driver B
    2. Check Driver A availability (backup)
    3. Lock Driver A [waiting...]

Deadlock: Thread 1 waits for B, Thread 2 waits for A
```

**Uber's Solution:**
```python
def match_driver_to_rider(driver_id, rider_id):
    # Always lock drivers before riders (consistent order)
    # And lock drivers by ID order
    
    with transaction():
        # Lock driver
        driver = Driver.objects.select_for_update().get(id=driver_id)
        
        if not driver.is_available:
            raise DriverUnavailable()
        
        # Lock rider
        rider = Rider.objects.select_for_update().get(id=rider_id)
        
        if not rider.is_waiting:
            raise RiderNotWaiting()
        
        # Create trip
        trip = Trip.objects.create(driver=driver, rider=rider)
        
        # Update status
        driver.is_available = False
        driver.current_trip = trip
        driver.save()
        
        rider.is_waiting = False
        rider.current_trip = trip
        rider.save()
        
        return trip

# Consistent order prevents deadlocks:
# - Always lock drivers before riders
# - Always lock drivers by ID
# - Result: 0 deadlocks in matching system
```

---

## 10. Interview Questions

### Q1: How would you design a system to minimize deadlocks in a high-traffic financial application?

**Answer:**

**Deadlock Prevention Strategies:**

1. **Consistent Lock Order:**
```python
def transfer(from_account, to_account, amount):
    # Always lock accounts in ascending ID order
    lock_order = sorted([from_account.id, to_account.id])
    
    # Lock both accounts
    accounts = Account.objects.filter(
        id__in=lock_order
    ).select_for_update().order_by('id')  # ORDER BY id critical!
    
    # Perform transfer
    ...
```

2. **Reduce Lock Duration:**
```python
# Bad: Hold locks during external API calls
with transaction():
    account.balance -= 100
    account.save()  # Lock held
    call_payment_api()  # Slow! Lock still held

# Good: Release locks before external calls
with transaction():
    account.balance -= 100
    account.save()
# Lock released!
call_payment_api()  # Outside transaction
```

3. **Partition Hot Resources:**
```sql
-- Hot table: All transactions touch same row
UPDATE global_counter SET count = count + 1;

-- Solution: Shard counter
CREATE TABLE counter_shards (
    shard_id INT PRIMARY KEY,
    count BIGINT
);

-- Update random shard (reduces contention)
UPDATE counter_shards SET count = count + 1 
WHERE shard_id = FLOOR(RANDOM() * 100);
```

4. **Retry with Exponential Backoff:**
```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=0.1, min=0.1, max=2),
    retry=retry_if_exception_type(psycopg2.errors.DeadlockDetected)
)
def transfer_money(from_id, to_id, amount):
    # Will retry on deadlock: 100ms, 200ms, 400ms
    ...
```

**Interview insight:** "In production at [Company], we eliminated 99% of deadlocks by enforcing consistent lock order via database ID. The remaining 1% we handle with exponential backoff retry logic. Key metric: Deadlock rate < 0.01% of transactions."

---

## 11. Summary

**Deadlock Definition:**
- Circular wait: A waits for B, B waits for A
- All 4 Coffman conditions must be true

**Detection:**
- Wait-for graph with cycle detection
- PostgreSQL checks after `deadlock_timeout` (default: 1s)
- Victim chosen (lowest cost transaction aborted)

**Prevention:**
- **Consistent lock order** (by ID, always)
- **Short transactions** (no external API calls)
- **Timeouts** (lock_timeout, statement_timeout)
- **Retry logic** (exponential backoff)

**Critical Pattern:**
```python
# Always sort resource IDs before locking
resources = sorted([r1, r2, r3], key=lambda r: r.id)
for resource in resources:
    lock(resource)  # Consistent order prevents deadlocks
```

**Monitoring:**
```sql
-- Deadlock rate
SELECT datname, deadlocks FROM pg_stat_database;

-- Lock waits
SELECT COUNT(*) FROM pg_locks WHERE NOT granted;
```

**Key Takeaway:** "Deadlocks are inevitable in high-concurrency systems. Prevention (consistent lock order) is better than detection (abort and retry). In production: Always sort resource IDs before locking, keep transactions < 100ms, and implement exponential backoff retry logic."

---

**Next:** [05_Transaction_Logs.md](05_Transaction_Logs.md) - WAL, redo/undo logs, crash recovery

