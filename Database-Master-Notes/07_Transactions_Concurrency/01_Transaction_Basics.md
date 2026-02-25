# Transaction Basics - Foundation of Database Reliability

## 1. Concept Explanation

**A transaction** is a logical unit of work that groups one or more database operations into a single, atomic operation. Either all operations in the transaction succeed (commit), or all fail (rollback)—there is no partial success.

Think of transactions like banking transfers:
```
Transfer $100 from Account A to Account B:
1. Read balance of Account A
2. Subtract $100 from Account A
3. Read balance of Account B
4. Add $100 to Account B

Without transaction: Steps 1-2 succeed, then system crashes → Money disappears!
With transaction: All steps succeed together, or all rollback → Money safe!
```

### Core Transaction Operations

**BEGIN/START TRANSACTION:**
```sql
BEGIN;  -- PostgreSQL, MySQL
-- or
START TRANSACTION;  -- Standard SQL
-- or
BEGIN TRANSACTION;  -- SQL Server
```
Marks the start of a transaction. All subsequent statements are part of this transaction until COMMIT or ROLLBACK.

**COMMIT:**
```sql
COMMIT;
```
Makes all changes permanent. Once committed, changes are visible to other transactions and survive system crashes.

**ROLLBACK:**
```sql
ROLLBACK;
```
Undoes all changes made in the transaction. Database returns to the state before BEGIN.

**SAVEPOINT:**
```sql
SAVEPOINT savepoint_name;
-- Later, rollback to savepoint
ROLLBACK TO SAVEPOINT savepoint_name;
```
Creates a checkpoint within a transaction. Allows partial rollback without aborting the entire transaction.

### Visualization

```
Transaction Lifecycle:

[Application]
    |
    | BEGIN
    v
[Transaction Started]
    |
    | INSERT INTO accounts...
    | UPDATE accounts...
    | DELETE FROM logs...
    |
    +-- System crash? -----> [ROLLBACK] (automatic)
    |                            |
    |                            v
    |                      [Transaction Aborted]
    |
    | All operations successful?
    |
    +-- NO --------------> [ROLLBACK] (manual)
    |                            |
    |                            v
    |                      [Transaction Aborted]
    |
    +-- YES -------------> [COMMIT]
                                |
                                v
                           [Changes Permanent]
                                |
                                v
                           [Visible to other sessions]
```

---

## 2. Why It Matters

### Production Impact: Without Transactions

**Scenario: E-commerce Order Placement**

```sql
-- Without transaction (DANGEROUS!)
-- Step 1: Deduct inventory
UPDATE products SET stock = stock - 1 WHERE product_id = 123;

-- Step 2: Create order (CRASH HERE!)
-- System crashes before this executes

-- Step 3: Charge customer
INSERT INTO orders (customer_id, product_id, amount) VALUES (456, 123, 99.99);

-- Result:
-- - Inventory decremented ✓
-- - Order NOT created ✗
-- - Customer NOT charged ✗
-- - Product appears sold but no revenue!
-- - Inventory mismatch: phantom stock reduction
```

**With transaction (SAFE!):**
```sql
BEGIN;

-- Step 1: Deduct inventory
UPDATE products SET stock = stock - 1 WHERE product_id = 123;

-- Step 2: Create order
INSERT INTO orders (customer_id, product_id, amount) VALUES (456, 123, 99.99);

-- Step 3: Charge customer
INSERT INTO payments (order_id, amount, status) VALUES (LAST_INSERT_ID(), 99.99, 'charged');

-- If ANY step fails or system crashes → ROLLBACK (automatic)
-- All steps succeed → COMMIT

COMMIT;

-- Result: Either all succeed or none do (atomic!)
```

### Real Incident: Payment Processing Failure

```
Company: Online ticketing platform
Issue: Race condition in ticket sales

Without proper transactions:
1. User A and User B both try to buy last ticket (seat 12A)
2. Both read: "seat available = true"
3. User A: UPDATE seats SET reserved = true WHERE seat = '12A'
4. User B: UPDATE seats SET reserved = true WHERE seat = '12A'
5. Both succeed! Ticket sold twice!

Result:
- 2 customers charged for same seat
- Airport gate: 2 people with valid tickets for seat 12A
- Customer service nightmare
- Regulatory fines: $50K
- Reputation damage

Root cause: No transaction isolation, no row locking

Fix with transactions:
BEGIN;
SELECT * FROM seats WHERE seat = '12A' FOR UPDATE;  -- Lock the row
-- Check availability
IF available THEN
    UPDATE seats SET reserved = true WHERE seat = '12A';
    INSERT INTO bookings (user_id, seat, price) VALUES (?, '12A', 149.99);
    COMMIT;
ELSE
    ROLLBACK;
    RETURN "Seat no longer available";
END;

Result:
- User A locks row first → Succeeds
- User B waits for lock → Gets "seat unavailable" after A commits
- No double-booking! ✓
```

---

## 3. Internal Working

### Transaction States (Finite State Machine)

```
Transaction State Transitions:

                    BEGIN
                      |
                      v
            [1. ACTIVE]
                 |
                 | All operations executing
                 |
         +-------+-------+
         |               |
    COMMIT           ROLLBACK
         |               |
         v               v
  [2. PARTIALLY    [5. ABORTED]
     COMMITTED]           |
         |               | Undo changes
         |               v
         |        [6. ROLLED BACK]
         |
         | Write to disk (durable)
         v
  [3. COMMITTED]
         |
         | Transaction complete
         v
  [4. TERMINATED]
```

**State Definitions:**

1. **ACTIVE:** Transaction is executing operations
2. **PARTIALLY COMMITTED:** All operations completed, but changes not yet written to disk
3. **COMMITTED:** Changes permanently written to disk (durable)
4. **TERMINATED:** Transaction completed successfully
5. **ABORTED:** Transaction failed or explicitly rolled back
6. **ROLLED BACK:** All changes undone, database restored to pre-transaction state

### How COMMIT Works (Write-Ahead Logging - WAL)

```sql
BEGIN;
INSERT INTO orders (id, customer_id, total) VALUES (1001, 500, 249.99);
UPDATE customers SET last_order_date = NOW() WHERE id = 500;
COMMIT;
```

**Internal Steps During COMMIT:**

```
1. Application: COMMIT command sent
   |
   v
2. Database: Flush all transaction logs to disk (WAL)
   - Write INSERT log entry → WAL file (durable)
   - Write UPDATE log entry → WAL file (durable)
   - fsync() to force disk write (expensive!)
   |
   v
3. Database: Write COMMIT record to WAL
   - Transaction marked as committed in log
   - fsync() again (ensure commit record on disk)
   |
   v
4. Return success to application
   - Client receives "COMMIT successful"
   - Changes now visible to other transactions
   |
   v
5. Background: Eventually write data pages to disk
   - Actual table data written (asynchronous)
   - Can happen seconds/minutes later
   - If crash before this → Replay WAL on restart

Key insight: COMMIT returns as soon as WAL is on disk, NOT when data pages are written!
This is why WAL enables fast commits (sequential writes) vs slow data page updates (random writes).
```

### How ROLLBACK Works (Undo Logs)

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 123;
-- Oh no, wrong account!
ROLLBACK;
```

**Internal Steps During ROLLBACK:**

```
1. Database maintains undo information during transaction:
   Original operation:
     UPDATE accounts SET balance = 1000 WHERE account_id = 123;
   
   Undo log entry (created before update):
     account_id=123, old_balance=1100
   
   After update:
     balance = 1000 (new value in buffer)

2. ROLLBACK command:
   - Read undo log entries
   - Apply inverse operations
   - Restore balance = 1100 (old value)

3. Discard transaction:
   - Release all locks held by transaction
   - Discard undo log entries
   - Discard redo log entries (not yet on disk)
   - Return "ROLLBACK successful"

4. Result:
   - Database state as if transaction never happened
   - No changes visible to other transactions
   - Resources (locks, memory) released
```

---

## 4. Best Practices

### Practice 1: Keep Transactions Short

**❌ Bad: Long-Running Transaction (Holds Locks)**
```sql
BEGIN;

-- Step 1: Update 1M rows (takes 30 seconds)
UPDATE orders SET processed = true WHERE created_at < '2024-01-01';

-- Step 2: Send email to each customer (takes 10 minutes!)
FOR order IN (SELECT * FROM orders WHERE processed = true) LOOP
    PERFORM send_email(order.customer_email, order.order_id);
    -- Network call for EACH email (blocking!)
END LOOP;

-- Step 3: Archive to cold storage (takes 5 minutes)
CALL archive_old_orders();

COMMIT;  -- 15 minutes later!

-- Problems:
-- - Locks held for 15 minutes (blocks other transactions)
-- - Connection timeout risk
-- - Rollback would take ages (undo 1M rows + emails sent!)
-- - If crash during email loop → All progress lost
```

**✅ Good: Break Into Smaller Transactions**
```sql
-- Transaction 1: Update orders (fast, ~5 seconds)
BEGIN;
UPDATE orders SET processed = true WHERE created_at < '2024-01-01';
COMMIT;

-- Non-transactional: Send emails (outside transaction)
-- Use background job queue (Sidekiq, Celery, RabbitMQ)
FOR order IN (SELECT * FROM orders WHERE processed = true) LOOP
    enqueue_email_job(order.customer_email, order.order_id);
END LOOP;

-- Transaction 2: Archive (separate transaction)
BEGIN;
CALL archive_old_orders();
COMMIT;

-- Benefits:
-- - Locks held for 5 seconds each (vs 15 minutes)
-- - Emails continue even if archive fails
-- - Better concurrency (other transactions not blocked)
-- - Rollback is fast (small scope)
```

### Practice 2: Always Use Explicit Transactions for Multi-Step Operations

**❌ Bad: Implicit Transactions (Autocommit Mode)**
```sql
-- PostgreSQL default: autocommit ON
-- Each statement is its own transaction

-- Transfer money between accounts
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;  -- COMMIT (implicit)
-- CRASH HERE! Money deducted but not credited!
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;  -- Never executes

-- Result: $100 vanished into thin air!
```

**✅ Good: Explicit Transaction**
```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

-- Verify invariant: Total money unchanged
SELECT SUM(balance) FROM accounts;  -- Must equal original sum

COMMIT;  -- Both updates succeed together, or both rollback

-- If crash after first update BUT before COMMIT:
-- → Automatic ROLLBACK on restart
-- → Balance of account 1 restored
-- → Money safe!
```

### Practice 3: Use SAVEPOINTs for Partial Rollback

**Scenario: Bulk Import with Error Handling**

```sql
BEGIN;

-- Import 10K records in batches of 1000
FOR batch_num IN 1..10 LOOP
    SAVEPOINT batch_savepoint;
    
    BEGIN
        -- Insert 1000 records
        INSERT INTO products (id, name, price)
        SELECT id, name, price
        FROM staging_products
        WHERE batch = batch_num;
        
        -- Success: Release savepoint
        RELEASE SAVEPOINT batch_savepoint;
        
    EXCEPTION WHEN OTHERS THEN
        -- Error in this batch: Rollback only this batch
        ROLLBACK TO SAVEPOINT batch_savepoint;
        
        -- Log error
        INSERT INTO import_errors (batch, error_message)
        VALUES (batch_num, SQLERRM);
        
        -- Continue with next batch (don't abort entire transaction)
    END;
END LOOP;

COMMIT;  -- Commit successful batches

-- Result:
-- - Batches 1-7: Imported successfully (7000 records)
-- - Batch 8: Failed (1000 records skipped, error logged)
-- - Batches 9-10: Imported successfully (2000 records)
-- - Total: 9000 records imported, 1000 failed (not atomic, but intentional)
```

### Practice 4: Handle Deadlocks Gracefully

**Application-Level Retry Logic**

```python
import psycopg2
import time

def transfer_money(from_account, to_account, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            conn = get_db_connection()
            cursor = conn.cursor()
            
            cursor.execute("BEGIN;")
            
            # Deduct from sender
            cursor.execute(
                "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                (amount, from_account)
            )
            
            # Credit to recipient
            cursor.execute(
                "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                (amount, to_account)
            )
            
            cursor.execute("COMMIT;")
            return True  # Success!
            
        except psycopg2.extensions.TransactionRollbackError as e:
            # Deadlock detected (PostgreSQL error code 40P01)
            if "deadlock detected" in str(e):
                conn.rollback()
                
                if attempt < max_retries - 1:
                    # Exponential backoff: wait 100ms, 200ms, 400ms
                    wait_time = 0.1 * (2 ** attempt)
                    time.sleep(wait_time)
                    continue  # Retry
                else:
                    raise  # Max retries exceeded
            else:
                raise  # Different error, don't retry
                
        except Exception as e:
            conn.rollback()
            raise

# Usage
try:
    transfer_money(from_account=123, to_account=456, amount=100.00)
except Exception as e:
    print(f"Transfer failed after retries: {e}")
```

---

## 5. Common Mistakes

### Mistake 1: Forgetting to COMMIT

**Problem:**
```sql
-- Developer forgets COMMIT
BEGIN;
UPDATE users SET email = 'new@example.com' WHERE id = 123;
-- No COMMIT or ROLLBACK!

-- Session disconnects → Automatic ROLLBACK
-- Change is lost!
```

**In psql (PostgreSQL CLI):**
```
postgres=# BEGIN;
BEGIN
postgres=# UPDATE users SET email = 'new@example.com' WHERE id = 123;
UPDATE 1
postgres=# \q  -- Quit without COMMIT
-- WARNING: Changes were rolled back!
```

**Solution: Always END with COMMIT or ROLLBACK**
```sql
BEGIN;
UPDATE users SET email = 'new@example.com' WHERE id = 123;
COMMIT;  -- Don't forget!
```

### Mistake 2: Long Transactions Holding Locks

**Problem:**
```sql
-- Transaction 1 (User A's session)
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 123;
-- User A goes to lunch for 1 hour (transaction still open!)
-- Locks held for 1 hour!

-- Transaction 2 (User B's session)
SELECT * FROM accounts WHERE account_id = 123 FOR UPDATE;
-- BLOCKED! Waiting for User A's lock
-- After 30 seconds: timeout error
-- User B frustrated, calls support
```

**Observability:**
```sql
-- PostgreSQL: Find long-running transactions
SELECT
    pid,
    now() - xact_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Output:
--  pid  | duration  |  state  |              query                
-- ------+-----------+---------+-----------------------------------
--  1234 | 01:15:32  | idle in | UPDATE accounts SET balance...
--       |           | transaction |

-- Solution: Kill long transaction (as admin)
SELECT pg_terminate_backend(1234);
```

**Fix: Set Statement Timeout**
```sql
-- Application configuration
SET statement_timeout = '30s';  -- Kill queries after 30 seconds
SET idle_in_transaction_session_timeout = '5min';  -- Kill idle transactions

BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 123;
-- If developer forgets to commit and goes to lunch:
-- → After 5 minutes: Automatic ROLLBACK
-- → Locks released
-- → Other transactions can proceed
```

### Mistake 3: Not Handling Transaction Failures

**❌ Bad: No Error Handling**
```python
def create_order(customer_id, items):
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute("BEGIN;")
    cursor.execute("INSERT INTO orders (customer_id) VALUES (%s)", (customer_id,))
    order_id = cursor.lastrowid
    
    for item in items:
        cursor.execute("INSERT INTO order_items (order_id, item_id) VALUES (%s, %s)",
                      (order_id, item['id']))
    
    cursor.execute("COMMIT;")
    # What if INSERT fails? No error handling!
    # Transaction might be left open (idle in transaction)
```

**✅ Good: Proper Error Handling**
```python
def create_order(customer_id, items):
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        cursor.execute("BEGIN;")
        
        cursor.execute("INSERT INTO orders (customer_id) VALUES (%s)", (customer_id,))
        order_id = cursor.lastrowid
        
        for item in items:
            cursor.execute("INSERT INTO order_items (order_id, item_id) VALUES (%s, %s)",
                          (order_id, item['id']))
        
        cursor.execute("COMMIT;")
        return order_id
        
    except Exception as e:
        cursor.execute("ROLLBACK;")
        logging.error(f"Order creation failed: {e}")
        raise
        
    finally:
        cursor.close()
        conn.close()  # Always close connection
```

### Mistake 4: Transactions in Loops (N+1 Problem)

**❌ Bad: Transaction Per Row**
```sql
-- Update 10K users (one transaction per user)
FOR user IN (SELECT * FROM users) LOOP
    BEGIN;
    UPDATE users SET last_login = NOW() WHERE id = user.id;
    COMMIT;  -- 10K commits!
END LOOP;

-- Performance:
-- - 10K BEGIN/COMMIT pairs
-- - 10K fsync() calls to disk
-- - 10K transaction log entries
-- - Time: 5 minutes (slow!)
```

**✅ Good: Single Transaction for Batch**
```sql
-- Update 10K users in one transaction
BEGIN;

UPDATE users SET last_login = NOW() WHERE id IN (
    SELECT id FROM users WHERE last_login < NOW() - INTERVAL '7 days'
);

COMMIT;  -- Single commit

-- Performance:
-- - 1 BEGIN/COMMIT pair
-- - 1 fsync() call
-- - 1 transaction log entry
-- - Time: 3 seconds (100× faster!)
```

---

## 6. Security Considerations

### 1. SQL Injection via Transaction Control

**Vulnerable Code:**
```python
# NEVER DO THIS!
account_id = request.GET['account_id']  # User input: "123; DROP TABLE accounts; --"

cursor.execute("BEGIN;")
cursor.execute(f"UPDATE accounts SET balance = balance - 100 WHERE id = {account_id}")
# Executes: UPDATE accounts SET balance = balance - 100 WHERE id = 123; DROP TABLE accounts; --
cursor.execute("COMMIT;")

# Result: accounts table deleted!
```

**Secure Code:**
```python
# Use parameterized queries
account_id = request.GET['account_id']

cursor.execute("BEGIN;")
cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = %s", (account_id,))
# SQL injection impossible (parameter properly escaped)
cursor.execute("COMMIT;")
```

### 2. Privilege Escalation via Transaction

**Attack Scenario:**
```sql
-- Attacker has limited privileges
-- Normal query: Allowed
SELECT * FROM products WHERE public = true;

-- Attacker tries privilege escalation
BEGIN;

-- Create temporary function with SECURITY DEFINER
CREATE OR REPLACE FUNCTION temp_admin_func() RETURNS void AS $$
BEGIN
    UPDATE users SET role = 'admin' WHERE id = 999;  -- Attacker's account
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;  -- Runs with function owner's privileges!

-- Call function
SELECT temp_admin_func();

COMMIT;

-- Attacker now admin!
```

**Mitigation:**
```sql
-- Principle of least privilege
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE CREATE ON DATABASE mydb FROM PUBLIC;

-- Only admins can create functions
GRANT CREATE ON SCHEMA public TO dba_role;

-- Audit function creation
CREATE EVENT TRIGGER log_function_creation
ON ddl_command_end
WHEN TAG IN ('CREATE FUNCTION')
EXECUTE FUNCTION audit_log_function_creation();
```

---

## 7. Performance Optimization

### Optimization 1: Batch Operations

**Slow: Individual Transactions**
```sql
-- Insert 10K records (one transaction per insert)
FOR i IN 1..10000 LOOP
    BEGIN;
    INSERT INTO logs (message) VALUES ('Log entry ' || i);
    COMMIT;
END LOOP;

-- Performance:
-- - 10K transactions
-- - 10K fsync() calls (disk writes)
-- - Time: 120 seconds
```

**Fast: Batched Inserts**
```sql
-- Insert 10K records in one transaction
BEGIN;

INSERT INTO logs (message)
SELECT 'Log entry ' || generate_series(1, 10000);

COMMIT;

-- Performance:
-- - 1 transaction
-- - 1 fsync() call
-- - Time: 1 second (120× faster!)
```

### Optimization 2: Read-Only Transactions

```sql
-- Mark transaction as read-only (optimizer benefit)
START TRANSACTION READ ONLY;

SELECT * FROM orders WHERE customer_id = 123;
SELECT * FROM order_items WHERE order_id IN (...);

COMMIT;

-- Benefits:
-- - No locks acquired (faster)
-- - Can execute on read replica (load balancing)
-- - Faster snapshot creation (no locking needed)
-- - Attempt to write → Error (safety)
```

### Optimization 3: Asynchronous Commit (PostgreSQL)

```sql
-- Trade durability for performance (careful!)
SET synchronous_commit = OFF;

BEGIN;
INSERT INTO analytics_events (event_type, data) VALUES ('click', '...');
COMMIT;  -- Returns immediately (doesn't wait for disk fsync)

-- Behavior:
-- - COMMIT returns before WAL is on disk
-- - If crash within ~200ms → Last few commits lost
-- - Acceptable for analytics (not money transfers!)

-- Performance gain: 10× faster commits (100 commits/sec → 1000 commits/sec)

-- Use for:
-- ✓ Analytics events (losing a few is OK)
-- ✓ Session logs
-- ✓ Audit trails (non-critical)

-- DO NOT use for:
-- ✗ Financial transactions
-- ✗ User data
-- ✗ Inventory updates
```

---

## 8. Examples

### Example 1: E-commerce Order Processing

```sql
-- Complete order placement transaction
CREATE OR REPLACE FUNCTION process_order(
    p_customer_id INT,
    p_items JSONB,  -- [{"product_id": 123, "quantity": 2, "price": 29.99}, ...]
    p_payment_method VARCHAR
) RETURNS INT AS $$
DECLARE
    v_order_id INT;
    v_item JSONB;
    v_product_stock INT;
    v_total_amount DECIMAL(10,2) := 0;
BEGIN
    -- Start transaction (implicit, but shown for clarity)
    -- All operations atomic: succeed together or rollback together
    
    -- Step 1: Create order
    INSERT INTO orders (customer_id, order_date, status)
    VALUES (p_customer_id, NOW(), 'pending')
    RETURNING id INTO v_order_id;
    
    -- Step 2: Process each item
    FOR v_item IN SELECT * FROM jsonb_array_elements(p_items)
    LOOP
        -- Check stock availability (with row lock)
        SELECT stock INTO v_product_stock
        FROM products
        WHERE id = (v_item->>'product_id')::INT
        FOR UPDATE;  -- Lock row (prevent concurrent purchases)
        
        IF v_product_stock < (v_item->>'quantity')::INT THEN
            -- Insufficient stock: Rollback entire order
            RAISE EXCEPTION 'Product % out of stock', v_item->>'product_id';
        END IF;
        
        -- Deduct inventory
        UPDATE products
        SET stock = stock - (v_item->>'quantity')::INT
        WHERE id = (v_item->>'product_id')::INT;
        
        -- Add order item
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        VALUES (
            v_order_id,
            (v_item->>'product_id')::INT,
            (v_item->>'quantity')::INT,
            (v_item->>'price')::DECIMAL
        );
        
        -- Accumulate total
        v_total_amount := v_total_amount + 
            ((v_item->>'quantity')::INT * (v_item->>'price')::DECIMAL);
    END LOOP;
    
    -- Step 3: Update order total
    UPDATE orders SET total_amount = v_total_amount WHERE id = v_order_id;
    
    -- Step 4: Process payment (simulate)
    INSERT INTO payments (order_id, amount, payment_method, status)
    VALUES (v_order_id, v_total_amount, p_payment_method, 'completed');
    
    -- Step 5: Mark order complete
    UPDATE orders SET status = 'confirmed' WHERE id = v_order_id;
    
    -- Success: Return order ID (implicit COMMIT)
    RETURN v_order_id;
    
EXCEPTION
    WHEN OTHERS THEN
        -- Any error: Rollback entire order (implicit ROLLBACK)
        RAISE NOTICE 'Order processing failed: %', SQLERRM;
        RAISE;  -- Re-raise exception to caller
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT process_order(
    p_customer_id := 456,
    p_items := '[
        {"product_id": 123, "quantity": 2, "price": 29.99},
        {"product_id": 456, "quantity": 1, "price": 99.99}
    ]'::jsonb,
    p_payment_method := 'credit_card'
);

-- Result: Order ID 1001 (or exception if any step fails)
```

---

### Example 2: Banking Transfer with Savepoints

```sql
-- Complex transaction with multiple savepoints
CREATE OR REPLACE FUNCTION process_batch_transfers(
    p_transfers JSONB  -- [{"from": 123, "to": 456, "amount": 100}, ...]
) RETURNS TABLE (transfer_id INT, success BOOLEAN, error_message TEXT) AS $$
DECLARE
    v_transfer JSONB;
    v_transfer_num INT := 0;
    v_from_balance DECIMAL;
BEGIN
    FOR v_transfer IN SELECT * FROM jsonb_array_elements(p_transfers)
    LOOP
        v_transfer_num := v_transfer_num + 1;
        
        -- Savepoint for this transfer
        EXECUTE format('SAVEPOINT transfer_%s', v_transfer_num);
        
        BEGIN
            -- Check balance
            SELECT balance INTO v_from_balance
            FROM accounts
            WHERE id = (v_transfer->>'from')::INT
            FOR UPDATE;
            
            IF v_from_balance < (v_transfer->>'amount')::DECIMAL THEN
                RAISE EXCEPTION 'Insufficient funds';
            END IF;
            
            -- Deduct from sender
            UPDATE accounts
            SET balance = balance - (v_transfer->>'amount')::DECIMAL
            WHERE id = (v_transfer->>'from')::INT;
            
            -- Credit to recipient
            UPDATE accounts
            SET balance = balance + (v_transfer->>'amount')::DECIMAL
            WHERE id = (v_transfer->>'to')::INT;
            
            -- Log successful transfer
            INSERT INTO transfer_log (from_account, to_account, amount, status)
            VALUES (
                (v_transfer->>'from')::INT,
                (v_transfer->>'to')::INT,
                (v_transfer->>'amount')::DECIMAL,
                'success'
            );
            
            -- Release savepoint (success)
            EXECUTE format('RELEASE SAVEPOINT transfer_%s', v_transfer_num);
            
            -- Return success row
            RETURN QUERY SELECT v_transfer_num, TRUE, NULL::TEXT;
            
        EXCEPTION WHEN OTHERS THEN
            -- Rollback only this transfer (not entire batch)
            EXECUTE format('ROLLBACK TO SAVEPOINT transfer_%s', v_transfer_num);
            
            -- Log failure
            INSERT INTO transfer_log (from_account, to_account, amount, status, error)
            VALUES (
                (v_transfer->>'from')::INT,
                (v_transfer->>'to')::INT,
                (v_transfer->>'amount')::DECIMAL,
                'failed',
                SQLERRM
            );
            
            -- Return failure row
            RETURN QUERY SELECT v_transfer_num, FALSE, SQLERRM;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM process_batch_transfers('[
    {"from": 123, "to": 456, "amount": 100},
    {"from": 789, "to": 234, "amount": 50},
    {"from": 123, "to": 789, "amount": 999999}
]'::jsonb);

-- Output:
--  transfer_id | success |      error_message       
-- -------------+---------+--------------------------
--            1 | t       | 
--            2 | t       | 
--            3 | f       | Insufficient funds
-- 
-- Result:
-- - Transfers 1 and 2 committed
-- - Transfer 3 rolled back (but 1 and 2 remain)
-- - Audit log contains all attempts
```

---

## 9. Real-World Use Cases

### Use Case 1: Stripe - Payment Processing

**Challenge:**
- Process millions of payments per day
- Must be ACID-compliant (money cannot vanish)
- Handle concurrent operations (many customers paying simultaneously)
- Support retries (network failures, temporary errors)

**Solution: Idempotent Transactions with Idempotency Keys**

```sql
-- Stripe's approach (simplified)
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE,  -- Client-provided key
    customer_id INT,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Payment processing function
CREATE OR REPLACE FUNCTION process_payment(
    p_idempotency_key VARCHAR,
    p_customer_id INT,
    p_amount DECIMAL
) RETURNS TABLE (payment_id BIGINT, status VARCHAR) AS $$
DECLARE
    v_existing_payment_id BIGINT;
BEGIN
    -- Check if payment already processed (idempotency)
    SELECT id INTO v_existing_payment_id
    FROM payments
    WHERE idempotency_key = p_idempotency_key;
    
    IF v_existing_payment_id IS NOT NULL THEN
        -- Payment already exists: Return existing payment (safe retry)
        RETURN QUERY
        SELECT id, status
        FROM payments
        WHERE id = v_existing_payment_id;
        RETURN;
    END IF;
    
    -- New payment: Process atomically
    BEGIN
        -- Reserve idempotency key immediately
        INSERT INTO payments (idempotency_key, customer_id, amount, status)
        VALUES (p_idempotency_key, p_customer_id, p_amount, 'processing')
        RETURNING id INTO v_existing_payment_id;
        
        -- Call external payment gateway (Stripe API)
        -- If this fails → Transaction rolls back, idempotency key released
        PERFORM charge_credit_card(p_customer_id, p_amount);
        
        -- Update payment status
        UPDATE payments
        SET status = 'succeeded'
        WHERE id = v_existing_payment_id;
        
        -- Deduct balance
        UPDATE customers
        SET balance = balance - p_amount
        WHERE id = p_customer_id;
        
        -- Success
        RETURN QUERY
        SELECT id, status
        FROM payments
        WHERE id = v_existing_payment_id;
        
    EXCEPTION WHEN OTHERS THEN
        -- Payment failed: Record failure
        UPDATE payments
        SET status = 'failed'
        WHERE id = v_existing_payment_id;
        
        RAISE;
    END;
END;
$$ LANGUAGE plpgsql;

-- Client usage (with retry)
-- Request 1 (initial attempt)
SELECT * FROM process_payment('unique-key-123', 456, 99.99);
-- Result: payment_id=1001, status='succeeded'

-- Request 2 (retry due to network timeout, but payment already succeeded)
SELECT * FROM process_payment('unique-key-123', 456, 99.99);
-- Result: payment_id=1001, status='succeeded' (same payment, no double-charge!)

-- Key insight: Idempotency key + transactions = Safe retries
```

### Use Case 2: Uber - Ride State Transitions

**Challenge:**
- Ride goes through multiple states (requested → matched → pickup → trip → completed → paid)
- Must maintain consistency (no invalid state transitions)
- Handle concurrent updates (driver and rider actions)
- Audit all state changes

**Solution: State Machine with Transactions**

```sql
CREATE TABLE rides (
    id BIGSERIAL PRIMARY KEY,
    driver_id INT,
    rider_id INT,
    status VARCHAR(20) CHECK (status IN ('requested', 'matched', 'pickup', 'trip', 'completed', 'paid', 'cancelled')),
    status_updated_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT valid_state_transitions CHECK (
        -- Define valid transitions in database
    )
);

-- State transition function
CREATE OR REPLACE FUNCTION transition_ride_status(
    p_ride_id BIGINT,
    p_new_status VARCHAR,
    p_actor VARCHAR  -- 'driver' or 'rider'
) RETURNS BOOLEAN AS $$
DECLARE
    v_current_status VARCHAR;
    v_valid_transition BOOLEAN := FALSE;
BEGIN
    -- Lock ride row (prevent concurrent updates)
    SELECT status INTO v_current_status
    FROM rides
    WHERE id = p_ride_id
    FOR UPDATE;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Ride % not found', p_ride_id;
    END IF;
    
    -- Validate transition (state machine logic)
    IF (v_current_status = 'requested' AND p_new_status = 'matched') THEN
        v_valid_transition := TRUE;
    ELSIF (v_current_status = 'matched' AND p_new_status = 'pickup' AND p_actor = 'driver') THEN
        v_valid_transition := TRUE;
    ELSIF (v_current_status = 'pickup' AND p_new_status = 'trip' AND p_actor = 'driver') THEN
        v_valid_transition := TRUE;
    ELSIF (v_current_status = 'trip' AND p_new_status = 'completed' AND p_actor = 'driver') THEN
        v_valid_transition := TRUE;
    ELSIF (v_current_status = 'completed' AND p_new_status = 'paid') THEN
        v_valid_transition := TRUE;
    ELSIF (p_new_status = 'cancelled') THEN
        v_valid_transition := TRUE;  -- Can cancel from most states
    ELSE
        RAISE EXCEPTION 'Invalid transition: % -> %', v_current_status, p_new_status;
    END IF;
    
    IF v_valid_transition THEN
        -- Update ride status
        UPDATE rides
        SET status = p_new_status,
            status_updated_at = NOW()
        WHERE id = p_ride_id;
        
        -- Audit log
        INSERT INTO ride_status_log (ride_id, old_status, new_status, actor, timestamp)
        VALUES (p_ride_id, v_current_status, p_new_status, p_actor, NOW());
        
        RETURN TRUE;
    END IF;
    
    RETURN FALSE;
END;
$$ LANGUAGE plpgsql;

-- Usage
-- Driver arrives at pickup location
SELECT transition_ride_status(
    p_ride_id := 1001,
    p_new_status := 'pickup',
    p_actor := 'driver'
);  -- Success (matched → pickup)

-- Invalid transition (trip → requested)
SELECT transition_ride_status(
    p_ride_id := 1001,
    p_new_status := 'requested',
    p_actor := 'driver'
);  -- ERROR: Invalid transition

-- Key insight: Transaction + row lock + state validation = Consistent state machine
```

---

## 10. Interview Questions

### Q1: Explain the difference between COMMIT and ROLLBACK. When would you use each?

**Answer:**

**COMMIT:**
- **Purpose:** Makes all changes in the transaction permanent and visible to other sessions
- **Internal behavior:** Flushes Write-Ahead Log (WAL) to disk, marks transaction as committed
- **When to use:**
  - All operations succeeded
  - Data validation passed
  - Business logic constraints satisfied
  
**Example:**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 123;
UPDATE accounts SET balance = balance + 100 WHERE id = 456;
-- Verify total balance unchanged (invariant check)
SELECT SUM(balance) FROM accounts;  -- If correct → COMMIT
COMMIT;
```

**ROLLBACK:**
- **Purpose:** Undoes all changes, restores database to state before BEGIN
- **Internal behavior:** Applies undo log entries, releases locks, discards transaction
- **When to use:**
  - Operation failed (constraint violation, deadlock)
  - Business logic validation failed
  - User cancellation
  - Exception occurred

**Example:**
```sql
BEGIN;
UPDATE inventory SET stock = stock - 10 WHERE product_id = 123;
-- Check if stock went negative
SELECT stock INTO v_stock FROM inventory WHERE product_id = 123;
IF v_stock < 0 THEN
    ROLLBACK;  -- Insufficient stock, undo deduction
ELSE
    COMMIT;
END IF;
```

**Key Difference:**
- COMMIT = "Save changes permanently"
- ROLLBACK = "Undo changes, pretend transaction never happened"

**Interview insight:** "In a previous project, I implemented order processing with explicit ROLLBACK when inventory checks failed. We validated each step before committing, ensuring no partial orders entered the system. Transaction success rate: 99.2%, failed orders properly rolled back with clear error messages."

---

### Q2: How would you design a transaction for a banking system that transfers money between accounts? What edge cases would you handle?

**Staff-Level Answer:**

```sql
CREATE OR REPLACE FUNCTION transfer_money(
    p_from_account INT,
    p_to_account INT,
    p_amount DECIMAL(12,2),
    p_transaction_id VARCHAR  -- Idempotency key
) RETURNS TABLE (success BOOLEAN, error_message TEXT) AS $$
DECLARE
    v_from_balance DECIMAL;
    v_existing_transfer BOOLEAN;
BEGIN
    -- Edge Case 1: Duplicate transfer detection (idempotency)
    SELECT EXISTS(
        SELECT 1 FROM transfers WHERE transaction_id = p_transaction_id
    ) INTO v_existing_transfer;
    
    IF v_existing_transfer THEN
        RETURN QUERY SELECT TRUE, 'Transfer already processed'::TEXT;
        RETURN;
    END IF;
    
    -- Edge Case 2: Validate inputs
    IF p_amount <= 0 THEN
        RETURN QUERY SELECT FALSE, 'Amount must be positive'::TEXT;
        RETURN;
    END IF;
    
    IF p_from_account = p_to_account THEN
        RETURN QUERY SELECT FALSE, 'Cannot transfer to same account'::TEXT;
        RETURN;
    END IF;
    
    -- Edge Case 3: Lock accounts in consistent order (prevent deadlock)
    -- Always lock smaller account_id first
    IF p_from_account < p_to_account THEN
        PERFORM * FROM accounts WHERE id = p_from_account FOR UPDATE;
        PERFORM * FROM accounts WHERE id = p_to_account FOR UPDATE;
    ELSE
        PERFORM * FROM accounts WHERE id = p_to_account FOR UPDATE;
        PERFORM * FROM accounts WHERE id = p_from_account FOR UPDATE;
    END IF;
    
    -- Edge Case 4: Check source account exists and has sufficient balance
    SELECT balance INTO v_from_balance
    FROM accounts
    WHERE id = p_from_account;
    
    IF v_from_balance IS NULL THEN
        RETURN QUERY SELECT FALSE, 'Source account not found'::TEXT;
        RETURN;
    END IF;
    
    IF v_from_balance < p_amount THEN
        RETURN QUERY SELECT FALSE, format('Insufficient funds: %s < %s', v_from_balance, p_amount)::TEXT;
        RETURN;
    END IF;
    
    -- Edge Case 5: Check destination account exists
    IF NOT EXISTS(SELECT 1 FROM accounts WHERE id = p_to_account) THEN
        RETURN QUERY SELECT FALSE, 'Destination account not found'::TEXT;
        RETURN;
    END IF;
    
    -- Edge Case 6: Check account status (frozen, closed, etc.)
    IF EXISTS(SELECT 1 FROM accounts WHERE id = p_from_account AND status = 'frozen') THEN
        RETURN QUERY SELECT FALSE, 'Source account frozen'::TEXT;
        RETURN;
    END IF;
    
    -- Perform transfer
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from_account;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to_account;
    
    -- Edge Case 7: Audit trail
    INSERT INTO transfers (transaction_id, from_account, to_account, amount, timestamp)
    VALUES (p_transaction_id, p_from_account, p_to_account, p_amount, NOW());
    
    -- Edge Case 8: Post-transfer invariant check
    DECLARE
        v_from_balance_after DECIMAL;
        v_to_balance_after DECIMAL;
    BEGIN
        SELECT balance INTO v_from_balance_after FROM accounts WHERE id = p_from_account;
        SELECT balance INTO v_to_balance_after FROM accounts WHERE id = p_to_account;
        
        -- Verify balances make sense
        IF v_from_balance_after < 0 THEN
            RAISE EXCEPTION 'Negative balance after transfer: %', v_from_balance_after;
        END IF;
    END;
    
    RETURN QUERY SELECT TRUE, 'Transfer successful'::TEXT;
    
EXCEPTION
    WHEN OTHERS THEN
        -- Edge Case 9: Log failure
        INSERT INTO transfer_failures (transaction_id, error, timestamp)
        VALUES (p_transaction_id, SQLERRM, NOW());
        
        RETURN QUERY SELECT FALSE, SQLERRM;
END;
$$ LANGUAGE plpgsql;
```

**Edge Cases Handled:**

1. **Idempotency:** Duplicate transfer detection (network retries)
2. **Input Validation:** Positive amount, different accounts
3. **Deadlock Prevention:** Lock accounts in consistent order
4. **Account Existence:** Both accounts must exist
5. **Sufficient Balance:** Check before deducting
6. **Account Status:** Frozen/closed accounts
7. **Audit Trail:** Record all transfers (compliance)
8. **Invariant Verification:** Check balances after transfer
9. **Error Logging:** Record failures for debugging
10. **Atomicity:** All-or-nothing (no partial transfers)

**Interview insight:** "In production, I added a 10th edge case: daily transfer limits. If cumulative transfers exceeded $10K/day, we blocked with error message and sent fraud alert. This caught 23 fraudulent accounts in first month, saving $2M+ in losses."

---

## 11. Summary

### Transaction Fundamentals

**Core Operations:**
- **BEGIN:** Start transaction, enters ACTIVE state
- **COMMIT:** Make changes permanent, fsyncs WAL to disk
- **ROLLBACK:** Undo all changes, apply undo log
- **SAVEPOINT:** Checkpoint within transaction, allows partial rollback

**Key Principles:**

1. **Atomicity:** All operations succeed together or fail together (no partial state)
2. **Explicit Transactions:** Always wrap multi-step operations in BEGIN/COMMIT
3. **Short Transactions:** Minimize lock duration, release resources quickly
4. **Error Handling:** Always ROLLBACK on exception, log failures
5. **Idempotency:** Use unique keys to safely retry transactions

### Transaction States

```
BEGIN → ACTIVE → (success) → PARTIALLY COMMITTED → COMMITTED → TERMINATED
                  |
                  └→ (failure) → ABORTED → ROLLED BACK → TERMINATED
```

### Best Practices Checklist

- [ ] Use explicit transactions (BEGIN/COMMIT) for multi-step operations
- [ ] Keep transactions short (< 5 seconds ideal, < 30 seconds maximum)
- [ ] Always handle exceptions (try/catch with ROLLBACK)
- [ ] Use SAVEPOINTs for partial rollback in batch operations
- [ ] Lock rows in consistent order (prevent deadlocks)
- [ ] Set timeouts (statement_timeout, idle_in_transaction_timeout)
- [ ] Implement idempotency (use unique keys for retries)
- [ ] Monitor long-running transactions (pg_stat_activity)
- [ ] Batch operations (avoid transactions in loops)
- [ ] Validate invariants before COMMIT (check business rules)

### Common Patterns

**Pattern 1: Transfer (Debit/Credit)**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**Pattern 2: Order Processing**
```sql
BEGIN;
INSERT INTO orders (...) RETURNING id INTO order_id;
INSERT INTO order_items (...);
UPDATE inventory SET stock = stock - quantity;
COMMIT;
```

**Pattern 3: Batch with Savepoints**
```sql
BEGIN;
FOR batch IN batches LOOP
    SAVEPOINT batch_savepoint;
    BEGIN
        -- Process batch
        RELEASE SAVEPOINT batch_savepoint;
    EXCEPTION WHEN OTHERS THEN
        ROLLBACK TO SAVEPOINT batch_savepoint;
    END;
END LOOP;
COMMIT;
```

### Performance Tips

1. **Batch operations:** 1 transaction for N rows, not N transactions
2. **Read-only transactions:** Use `START TRANSACTION READ ONLY` for queries
3. **Asynchronous commit:** Trade durability for speed (carefully!)
4. **Connection pooling:** Reuse connections, avoid BEGIN/COMMIT overhead

### Key Takeaway

**"Transactions are the foundation of database reliability. Always use explicit transactions for multi-step operations, keep them short, and handle failures gracefully. In production: Monitor long-running transactions, implement idempotency, and validate business invariants before committing. When designing: Think about deadlocks (lock order matters), retries (idempotency keys), and audit trails (log everything)."**

---

**Next Reading:**
- [02_MVCC.md](02_MVCC.md) - How databases handle concurrent transactions
- [03_Locking_Mechanisms.md](03_Locking_Mechanisms.md) - Row locks, table locks, deadlocks
- [04_Deadlocks.md](04_Deadlocks.md) - Detection, prevention, resolution strategies
