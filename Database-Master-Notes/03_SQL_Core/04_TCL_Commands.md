# ⚡ TCL Commands - Transaction Control Language

> TCL commands manage transactions to ensure data consistency. Master BEGIN, COMMIT, ROLLBACK, and SAVEPOINT for reliable database operations.

---

## 📖 1. Concept Explanation

### What is TCL?

**Transaction Control Language (TCL)** controls transactions—groups of operations that execute as a single atomic unit (all succeed or all fail).

**Core TCL Commands:**
```
TCL COMMANDS
│
├── BEGIN/START TRANSACTION - Start transaction
│
├── COMMIT - Permanently save changes
│
├── ROLLBACK - Undo all changes
│
└── SAVEPOINT - Set checkpoint within transaction
    └── ROLLBACK TO SAVEPOINT - Undo to checkpoint
```

**Transaction Lifecycle:**
```
AUTO-COMMIT MODE
    ↓
BEGIN TRANSACTION
    ↓
SQL Statement 1 ──→ Success ──→ Continue
    ↓                            ↓
SQL Statement 2 ──→ Success ──→ Continue
    ↓                            ↓
SQL Statement 3 ──→ ERROR ───→ ROLLBACK (undo all)
    ↓
SQL Statement 3 (retry) ──→ Success
    ↓
COMMIT ──→ Changes permanent
```

---

## 🧠 2. Why It Matters in Real Systems

### Money Transfer Without Transactions (Disaster)

**❌ Without Transaction:**
```sql
-- Transfer $100 from Account A to Account B
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- ✅ Succeeds
-- [SERVER CRASHES HERE] ❌
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- ❌ Never executes

-- Result: $100 disappeared! Account A lost money, B didn't receive it ❌
```

**✅ With Transaction:**
```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- Both succeed or both fail (atomic) ✅

-- If crash before COMMIT: Automatic ROLLBACK, no money lost ✅
```

---

## ⚙️ 3. Internal Working

### Transaction Internals (InnoDB)

```sql
BEGIN;
    UPDATE users SET balance = balance + 100 WHERE id = 1;
COMMIT;

-- What happens internally:
-- 1. BEGIN: Start transaction context
-- 2. UPDATE: 
--    - Lock row (id=1)
--    - Write to undo log (for ROLLBACK)
--    - Write to redo log (for crash recovery)
--    - Update in-memory buffer pool
-- 3. COMMIT:
--    - Flush redo log to disk
--    - Release locks
--    - Mark transaction complete
--    - Changes visible to other transactions
```

### MVCC (Multi-Version Concurrency Control)

```sql
-- Transaction 1:
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  
-- Sees: balance = 1000

-- Transaction 2 (concurrent):
BEGIN;
UPDATE accounts SET balance = 500 WHERE id = 1;
COMMIT;

-- Transaction 1 (still in progress):
SELECT balance FROM accounts WHERE id = 1;
-- REPEATABLE READ: Still sees 1000 (snapshot isolation)
-- READ COMMITTED: Now sees 500
COMMIT;
```

---

## ✅ 4. Best Practices

### Always Use Transactions for Multi-Statement Operations

```sql
-- ✅ Transaction for consistency
BEGIN;
    INSERT INTO orders (user_id, total) VALUES (123, 150.00);
    UPDATE products SET stock = stock - 1 WHERE id = 456;
    INSERT INTO order_items (order_id, product_id) VALUES (LAST_INSERT_ID(), 456);
COMMIT;

-- All succeed or all fail ✅
```

### Keep Transactions Short

```sql
-- ❌ Bad: Long-running transaction
BEGIN;
    SELECT * FROM huge_table;  -- Takes 10 minutes
    -- Locks held for 10 minutes, blocks other users ❌
COMMIT;

-- ✅ Good: Short transactions
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Completes in milliseconds ✅
```

### Use SAVEPOINT for Complex Logic

```sql
BEGIN;
    INSERT INTO orders (user_id, total) VALUES (123, 150.00);
    SAVEPOINT sp1;
    
    UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 456;
    
    -- Check if inventory went negative
    IF (SELECT quantity FROM inventory WHERE product_id = 456) < 0 THEN
        ROLLBACK TO SAVEPOINT sp1;  -- Undo inventory update only
        -- Order insert is preserved
    END IF;
    
COMMIT;
```

### Handle Errors in Application

```python
# ✅ Python example with error handling
try:
    cursor.execute("BEGIN")
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = %s", (1,))
    cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = %s", (2,))
    cursor.execute("COMMIT")
except Exception as e:
    cursor.execute("ROLLBACK")
    logger.error(f"Transaction failed: {e}")
    raise
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Forgetting to COMMIT

```sql
BEGIN;
    UPDATE products SET price = 19.99 WHERE id = 123;
-- [Forgot COMMIT] ❌

-- Other sessions can't see the change
-- Connection closes: automatic ROLLBACK ❌
```

### Mistake 2: Very Long Transactions

```sql
-- ❌ Bad: Transaction spans user interaction
BEGIN;
    SELECT * FROM cart WHERE user_id = 123;
    -- Wait for user to click "Checkout" (could be 5 minutes) ❌
    UPDATE cart SET status = 'purchased' WHERE user_id = 123;
COMMIT;

-- Locks held for 5 minutes, blocks other users ❌

-- ✅ Good: Transaction only during actual update
-- SELECT outside transaction
$cart = SELECT * FROM cart WHERE user_id = 123;
-- User clicks checkout
BEGIN;
    UPDATE cart SET status = 'purchased' WHERE user_id = 123;
    INSERT INTO orders ...;
COMMIT;
-- Transaction completes in milliseconds ✅
```

### Mistake 3: DDL Inside Transaction (MySQL)

```sql
-- ❌ MySQL: DDL causes implicit COMMIT
BEGIN;
    INSERT INTO users (name) VALUES ('Alice');
    ALTER TABLE users ADD COLUMN age INT;  -- Implicit COMMIT! ❌
    INSERT INTO users (name) VALUES ('Bob');
ROLLBACK;  -- Only Bob is rolled back, Alice is committed! ❌

-- ✅ PostgreSQL: DDL is transactional
BEGIN;
    INSERT INTO users (name) VALUES ('Alice');
    ALTER TABLE users ADD COLUMN age INT;
    INSERT INTO users (name) VALUES ('Bob');
ROLLBACK;  -- Everything rolled back ✅
```

### Mistake 4: Not Handling Deadlocks

```sql
-- Transaction 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Locks row 1
-- Waiting for Transaction 2...
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Tries to lock row 2

-- Transaction 2 (concurrent):
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;  -- Locks row 2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;  -- Tries to lock row 1

-- DEADLOCK! Both waiting for each other ❌

-- ✅ Solution: Retry with exponential backoff
try:
    execute_transaction()
except DeadlockError:
    time.sleep(random.uniform(0.1, 0.5))
    retry_transaction()
```

### Mistake 5: Auto-Commit Confusion

```sql
-- ❌ MySQL default: Auto-commit ON
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Immediately committed, can't rollback ❌

-- ✅ Disable auto-commit for transactions
SET autocommit = 0;
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
SET autocommit = 1;
```

---

## 🔐 6. Security Considerations

### Prevent Transaction Hijacking

```sql
-- ✅ Always close connections properly
BEGIN;
    UPDATE sensitive_data SET value = 'new' WHERE id = 1;
    -- [Connection pool reused without COMMIT/ROLLBACK] ❌
    -- Next request sees uncommitted data!

-- ✅ Use connection pool cleanup
connection.ensure_closed_transaction()  -- Auto ROLLBACK if left open
```

### Transaction Timeout

```sql
-- ✅ Set timeout to prevent blocking
SET SESSION innodb_lock_wait_timeout = 5;  -- 5 seconds

BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    -- [Waiting for locked row]
    -- After 5 seconds: ERROR 1205 (HY000): Lock wait timeout
ROLLBACK;
```

---

## 🚀 7. Performance Optimization

### Batch Operations in Single Transaction

```sql
-- ❌ Slow: 1000 separate transactions
for i in 1..1000:
    BEGIN;
    INSERT INTO logs VALUES (...);
    COMMIT;
-- Time: 10 seconds (1000 × fsync)

-- ✅ Fast: One transaction, batch insert
BEGIN;
    INSERT INTO logs VALUES
        (...),  -- Row 1
        (...),  -- Row 2
        -- ... 1000 rows
    ;
COMMIT;
-- Time: 0.5 seconds (1 × fsync) ✅
```

### Read-Only Transactions

```sql
-- ✅ Optimize for read-only
START TRANSACTION READ ONLY;
    SELECT * FROM users WHERE id = 123;
    SELECT * FROM orders WHERE user_id = 123;
COMMIT;

-- Benefits:
-- - No undo log creation
-- - No lock acquisition
-- - Faster in some databases
```

---

## 🧪 8. Examples

### Bank Transfer

```sql
-- Atomically transfer money
BEGIN;
    -- Debit sender
    UPDATE accounts 
    SET balance = balance - 100 
    WHERE id = 1 AND balance >= 100;  -- Ensure sufficient funds
    
    -- Check if update succeeded
    IF ROW_COUNT() = 0 THEN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Insufficient funds';
    END IF;
    
    -- Credit receiver
    UPDATE accounts 
    SET balance = balance + 100 
    WHERE id = 2;
    
    -- Log transaction
    INSERT INTO transactions (from_account, to_account, amount) 
    VALUES (1, 2, 100);
    
COMMIT;
```

### Order Checkout

```sql
BEGIN;
    -- Create order
    INSERT INTO orders (user_id, total, status) 
    VALUES (123, 150.00, 'pending');
    
    SET @order_id = LAST_INSERT_ID();
    
    -- Add order items
    INSERT INTO order_items (order_id, product_id, quantity, price) VALUES
        (@order_id, 1, 2, 50.00),
        (@order_id, 5, 1, 50.00);
    
    -- Update inventory
    UPDATE products SET stock = stock - 2 WHERE id = 1;
    UPDATE products SET stock = stock - 1 WHERE id = 5;
    
    -- Check for negative stock
    IF EXISTS(SELECT 1 FROM products WHERE id IN (1, 5) AND stock < 0) THEN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Out of stock';
    END IF;
    
    -- Update order status
    UPDATE orders SET status = 'confirmed' WHERE id = @order_id;
    
COMMIT;
```

### Saga Pattern with SAVEPOINT

```sql
BEGIN;
    -- Step 1: Reserve inventory
    INSERT INTO reservations (product_id, quantity) VALUES (123, 5);
    SAVEPOINT sp_inventory;
    
    -- Step 2: Process payment
    INSERT INTO payments (user_id, amount) VALUES (456, 100.00);
    SAVEPOINT sp_payment;
    
    -- Step 3: Create shipment
    INSERT INTO shipments (order_id, status) VALUES (789, 'pending');
    
    -- If shipment fails, undo payment but keep reservation
    IF shipment_failed THEN
        ROLLBACK TO SAVEPOINT sp_payment;
        -- Can retry payment later
    END IF;
    
COMMIT;
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Stripe - Idempotent Payments

```sql
-- Prevent duplicate charges on retry
BEGIN;
    -- Check if payment already processed
    SELECT id FROM payments WHERE idempotency_key = 'unique_key_123';
    
    IF FOUND THEN
        -- Return existing payment
        ROLLBACK;
        RETURN existing_payment;
    ELSE
        -- Create new payment
        INSERT INTO payments (idempotency_key, amount, status) 
        VALUES ('unique_key_123', 5000, 'succeeded');
        
        COMMIT;
        RETURN new_payment;
    END IF;
```

### Case Study 2: Uber - Ride Matching

```sql
BEGIN;
    -- Lock driver (prevent double-booking)
    SELECT * FROM drivers 
    WHERE id = 456 AND status = 'available' 
    FOR UPDATE NOWAIT;  -- Fail immediately if locked
    
    -- If not available, try next driver
    IF NOT FOUND THEN
        ROLLBACK;
        RETURN 'driver_unavailable';
    END IF;
    
    -- Create ride
    INSERT INTO rides (driver_id, rider_id, status) 
    VALUES (456, 123, 'matched');
    
    -- Update driver status
    UPDATE drivers SET status = 'busy' WHERE id = 456;
    
COMMIT;
```

### Case Study 3: GitHub - Distributed Transactions

```sql
-- Two-phase commit across databases
-- Phase 1: PREPARE
BEGIN;
    UPDATE repo_stats SET stars = stars + 1 WHERE id = 123;
    PREPARE TRANSACTION 'txn_123';  -- Save for commit

-- [Coordinator checks other databases]

-- Phase 2: COMMIT
COMMIT PREPARED 'txn_123';  -- All or nothing across all DBs
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What is a transaction?

**Answer:** A transaction is a **group of operations** that execute as a **single atomic unit**—either all succeed or all fail.

**ACID Properties:**
- **Atomicity**: All or nothing
- **Consistency**: Data stays valid
- **Isolation**: Transactions don't interfere
- **Durability**: Committed changes persist

```sql
-- Example: Bank transfer (atomic)
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Both updates happen or neither happens
```

---

### Q2: What's the difference between COMMIT and ROLLBACK?

**Answer:**
- **COMMIT**: Makes changes **permanent**
- **ROLLBACK**: **Undoes** all changes since BEGIN

```sql
BEGIN;
    DELETE FROM users WHERE id = 123;
    -- Oops, wrong user!
ROLLBACK;  -- Undo delete ✅

BEGIN;
    UPDATE users SET status = 'active' WHERE id = 456;
COMMIT;  -- Save change permanently ✅
```

---

### Q3: What is a SAVEPOINT?

**Answer:** A **checkpoint** within a transaction that allows **partial rollback**.

```sql
BEGIN;
    INSERT INTO orders (id) VALUES (1);
    SAVEPOINT sp1;
    
    INSERT INTO orders (id) VALUES (2);
    SAVEPOINT sp2;
    
    INSERT INTO orders (id) VALUES (3);
    
    ROLLBACK TO SAVEPOINT sp2;  -- Undo only #3
    
COMMIT;
-- Result: Orders 1 and 2 saved, 3 discarded
```

---

### Q4: What happens if you don't COMMIT?

**Answer:**
- Changes stay **uncommitted** (other sessions can't see them)
- Connection closes → **automatic ROLLBACK**
- Locks held until COMMIT/ROLLBACK

```sql
BEGIN;
    UPDATE users SET status = 'inactive' WHERE id = 123;
    -- [Forgot COMMIT]
    -- [Connection closes]
-- Automatic ROLLBACK: Change lost ❌
```

---

### Q5: What is a deadlock? How do you handle it?

**Answer:** A deadlock occurs when **two transactions wait for each other's locks**, creating a circular dependency.

```sql
-- Transaction 1:
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;  -- Locks row 1
UPDATE accounts SET balance = balance + 10 WHERE id = 2;  -- Waits for row 2

-- Transaction 2:
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 2;  -- Locks row 2
UPDATE accounts SET balance = balance + 10 WHERE id = 1;  -- Waits for row 1

-- DEADLOCK! ❌
```

**Solution: Retry with exponential backoff**
```python
def transfer_money(from_id, to_id, amount):
    max_retries = 3
    for attempt in range(max_retries):
        try:
            db.execute("BEGIN")
            db.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s", (amount, from_id))
            db.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s", (amount, to_id))
            db.execute("COMMIT")
            return "success"
        except DeadlockError:
            db.execute("ROLLBACK")
            time.sleep(2 ** attempt)  # Exponential backoff
    return "failed"
```

---

## 🧩 11. Design Patterns

### Pattern 1: Compensating Transaction (Saga Pattern)

```sql
-- For distributed systems where 2PC isn't available
BEGIN;
    -- Step 1: Reserve inventory
    UPDATE inventory SET reserved = reserved + 5 WHERE product_id = 123;
    SET @reservation_id = insert_reservation(123, 5);
COMMIT;

-- [Call payment service]
IF payment_failed THEN
    -- Compensating transaction: Undo reservation
    BEGIN;
        UPDATE inventory SET reserved = reserved - 5 WHERE product_id = 123;
        DELETE FROM reservations WHERE id = @reservation_id;
    COMMIT;
END IF;
```

### Pattern 2: Optimistic Locking

```sql
-- Use version number to detect conflicts
BEGIN;
    SELECT version FROM accounts WHERE id = 123;  -- version = 5
    
    -- Update with version check
    UPDATE accounts 
    SET balance = balance + 100, version = version + 1
    WHERE id = 123 AND version = 5;
    
    IF ROW_COUNT() = 0 THEN
        -- Someone else modified it, retry
        ROLLBACK;
        RETURN 'conflict';
    END IF;
    
COMMIT;
```

### Pattern 3: Distributed Transaction (Two-Phase Commit)

```sql
-- Phase 1: PREPARE on all databases
PREPARE TRANSACTION 'global_txn_123';

-- Coordinator checks all databases
-- Phase 2a: If all OK, COMMIT
COMMIT PREPARED 'global_txn_123';

-- Phase 2b: If any fail, ROLLBACK
ROLLBACK PREPARED 'global_txn_123';
```

---

## 📚 Summary

### Key Takeaways

1. **ACID**: Atomicity, Consistency, Isolation, Durability
2. **BEGIN**: Start transaction
3. **COMMIT**: Make changes permanent
4. **ROLLBACK**: Undo all changes
5. **SAVEPOINT**: Checkpoint for partial rollback
6. **Keep transactions short**: Minimize lock duration
7. **Handle errors**: Always ROLLBACK on failure
8. **Deadlocks**: Retry with exponential backoff
9. **Auto-commit**: Disable for multi-statement transactions
10. **DDL**: Causes implicit COMMIT in MySQL

**Transaction Lifecycle:**
```
BEGIN → SQL statements → COMMIT (success) or ROLLBACK (failure)
```

---

**Next:** [05_Subqueries.md](05_Subqueries.md) - Nested queries  
**Related:** [../07_Transactions_And_Concurrency/](../07_Transactions_And_Concurrency/) - Advanced transaction patterns

---

*Last Updated: February 19, 2026*
