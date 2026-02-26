# Read Uncommitted - The Weakest Isolation Level

## 1. Concept Explanation

**Read Uncommitted** (also called "dirty read") is the lowest isolation level where transactions can read data that has been modified but not yet committed by other transactions.

**Core Behavior:**
```
Transaction A:
BEGIN;
UPDATE accounts SET balance = 500 WHERE id = 123;
-- NOT committed yet!

Transaction B (Read Uncommitted):
SELECT balance FROM accounts WHERE id = 123;
-- Returns: 500 (uncommitted data!)

Transaction A:
ROLLBACK;  -- Undo changes

-- Problem: Transaction B read "dirty" data (500) that was never committed
-- Reality: balance still 1000, but B saw 500
```

**Analogy: Reading Draft Documents**
```
Author writes draft: "Revenue: $10 million"
You read draft: See "$10 million"
Author revises: "Revenue: $5 million" (oops, typo!)
You already made decisions based on $10M (wrong data!)

Same as reading uncommitted data—changes may be rolled back
```

### Dirty Reads

**What is a Dirty Read?**
Reading data from a row that has been modified by another transaction but not yet committed.

**Example:**
```sql
-- Time 0: Initial state
balance = 1000

-- Time 1: Transaction A
BEGIN;
UPDATE accounts SET balance = 0 WHERE id = 123;  -- Deduct full balance

-- Time 2: Transaction B (Read Uncommitted)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
SELECT balance FROM accounts WHERE id = 123;
-- Returns: 0 (dirty read!)

IF balance = 0 THEN
    SEND_EMAIL("Account empty, please deposit funds");
END IF;
COMMIT;

-- Time 3: Transaction A rolls back
ROLLBACK;  -- Undo deduction, balance back to 1000

-- Result:
-- - Balance: 1000 (never changed)
-- - Email sent: "Account empty" (based on dirty read!)
-- - Customer confused: "My balance is 1000, why the email?"
```

---

## 2. Why It Matters

### Production Impact: Financial Reporting Disaster

**Scenario: Monthly Revenue Report**

```
Company: SaaS platform
Task: Generate monthly revenue report for investors

Timeline:
2024-01-31 23:50:00 - Month-end closing begins
    - Accountant starts bulk update (100K transactions)
    
BEGIN;
-- Update all December transactions
UPDATE transactions 
SET status = 'closed', 
    revenue = adjusted_revenue,
    tax = adjusted_tax
WHERE date BETWEEN '2024-12-01' AND '2024-12-31';
-- Takes 10 minutes to complete

23:55:00 - CEO runs revenue report [Read Uncommitted!]
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT SUM(revenue) FROM transactions 
WHERE date BETWEEN '2024-12-01' AND '2024-12-31';
-- Returns: $12.5 million

-- CEO emails board: "Great month! $12.5M revenue"

23:58:00 - Accountant discovers error in adjustment
    - Tax calculation wrong for 50K transactions
    - Needs to recalculate

ROLLBACK;  -- Undo all changes

-- Correct process (1 hour later):
BEGIN;
UPDATE transactions SET ... (with correct tax);
COMMIT;

01:00:00 - Final revenue: $10.2 million

Result:
- CEO reported: $12.5M (based on dirty read)
- Actual revenue: $10.2M
- Difference: $2.3M overstatement (18% error!)
- Board meeting disaster: "We reported $12.5M, now it's $10.2M?"
- Stock impact: Investor confidence shaken
- Cost: Market cap drop $50M on earnings correction

Root cause: Read Uncommitted allowed CEO query to see uncommitted data
```

### Real Incident: E-commerce Inventory Glitch

```
Date: Black Friday 2019
Company: Major e-commerce platform
Issue: Overselling due to dirty reads

Timeline:
10:00:00 - Flash sale begins: 100 items in stock

10:00:01 - Order processing system (Read Uncommitted)
Transaction 1: Customer A orders 50 items
    BEGIN;
    UPDATE inventory SET stock = stock - 50 WHERE product_id = 123;
    -- NOT committed yet (waiting for payment API)
    
10:00:02 - Transaction 2: Customer B orders 60 items [concurrent]
    SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
    SELECT stock FROM inventory WHERE product_id = 123;
    -- Dirty read: stock = 50 (sees uncommitted change!)
    
    IF stock >= 60 THEN
        -- Check passes (50 >= 60? NO! But read uncommitted data)
        -- Actually reads: 50 free + 50 reserved = allows order
        UPDATE inventory SET stock = stock - 60 WHERE product_id = 123;
        COMMIT;
    END IF;

10:00:05 - Transaction 1 payment fails
    ROLLBACK;  -- Customer A order cancelled, stock back to 100

Final state:
- Inventory: 40 items (100 - 60 from Customer B)
- Customer A: Order cancelled ✗
- Customer B: Order confirmed ✓ (60 items)
- Reality: Should have 100 items in stock, sold 60
- But system showed: 40 items remaining

10:30:00 - 100 more customers order remaining 40 items
    - All orders accepted (40 items available)
    - Warehouse ships: 40 items total
    - 100 customers waiting for 100 items!
    - Oversold: 60 items

Result:
- Customer complaints: 60 customers got "delayed shipping" email
- Refunds issued: 60 × $200 average = $12K
- Customer service: 100 hours × $20/hour = $2K
- Reputation damage: -500 negative reviews
- Total cost: ~$50K (including lost future sales)

Fix: Changed to Read Committed isolation level
Result: 0 overselling incidents after fix
```

---

## 3. Internal Working

### Read Uncommitted Implementation

**PostgreSQL:**
```sql
-- PostgreSQL does NOT support Read Uncommitted!
-- Silently upgrades to Read Committed

SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SHOW transaction_isolation;
-- Returns: "read committed" (upgraded automatically)

-- Reason: PostgreSQL MVCC design prevents dirty reads at architecture level
```

**MySQL/InnoDB:**
```sql
-- MySQL supports Read Uncommitted
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Internal behavior:
-- - No locks acquired on SELECT
-- - Reads latest version from buffer pool (even if uncommitted)
-- - Bypasses MVCC snapshot isolation
```

**SQL Server:**
```sql
-- SQL Server supports Read Uncommitted
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Or use NOLOCK hint (same effect)
SELECT * FROM accounts WITH (NOLOCK);

-- Internal:
-- - Bypasses shared locks (doesn't wait for exclusive locks)
-- - Reads uncommitted data pages directly
```

### What Read Uncommitted Bypasses

**Normal (Read Committed) behavior:**
```
Transaction A writes:
1. Acquire exclusive lock on row
2. Modify row in buffer
3. Write to transaction log
4. COMMIT: Make visible to others
5. Release lock

Transaction B reads (Read Committed):
1. Wait for exclusive lock release
2. Read committed version only
```

**Read Uncommitted behavior:**
```
Transaction A writes:
1. Acquire exclusive lock
2. Modify row in buffer
-- Transaction B can read NOW (doesn't wait!)

Transaction B reads (Read Uncommitted):
1. Skip lock check (no waiting!)
2. Read current buffer value (uncommitted!)
```

---

## 4. Best Practices

### Practice 1: Never Use for Financial or Critical Data

**❌ NEVER:**
```sql
-- Reporting on financial transactions
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT SUM(amount) FROM payments WHERE date = CURRENT_DATE;
-- May include uncommitted (possibly rolled back) payments!

-- Balance calculations
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT balance FROM accounts WHERE id = 123;
-- May see temporary deductions that get rolled back
```

**✅ ALWAYS USE READ COMMITTED OR HIGHER:**
```sql
-- Default isolation (Read Committed)
SELECT SUM(amount) FROM payments WHERE date = CURRENT_DATE;
-- Only sees committed data ✓

-- Or explicit
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM accounts WHERE id = 123;
```

### Practice 2: Use Only for Non-Critical Approximate Reports

**Acceptable use cases:**
```sql
-- 1. Approximate row counts (dashboards)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT COUNT(*) FROM large_table;
-- Dirty read acceptable: ~10M rows (exact count unimportant)

-- 2. Monitoring queries (rough metrics)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT 
    COUNT(*) AS active_sessions,
    AVG(duration) AS avg_duration
FROM sessions
WHERE last_active > NOW() - INTERVAL '5 minutes';
-- Approximation acceptable for real-time monitoring

-- 3. Development/testing environments
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM debug_logs ORDER BY timestamp DESC LIMIT 100;
-- Development only, not production!
```

### Practice 3: Document Usage Explicitly

**If you must use Read Uncommitted:**
```sql
-- BAD: No documentation
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT COUNT(*) FROM orders;

-- GOOD: Explicit reasoning
/*
 * READ UNCOMMITTED used intentionally for performance
 * Context: Dashboard approximate count (exact value unimportant)
 * Risk: May include uncommitted orders (acceptable for dashboard)
 * Benefit: No lock contention, 10× faster query
 * Reviewed: 2024-01-15 by @senior-dba
 */
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT COUNT(*) AS approximate_order_count FROM orders;
```

---

## 5. Common Mistakes

### Mistake 1: Using NOLOCK Hint Without Understanding Risk

**Problem:**
```sql
-- SQL Server developers often use NOLOCK for "performance"
SELECT * FROM orders WITH (NOLOCK)
WHERE customer_id = 123;

-- Risks:
-- 1. Dirty reads (uncommitted data)
-- 2. Missing rows (page splits during scan)
-- 3. Duplicate rows (page splits)
-- 4. Incorrect aggregates

-- Horror story:
-- Query: SELECT SUM(amount) FROM orders WITH (NOLOCK);
-- During scan: Orders table being updated (page splits)
-- Result: Reads some rows twice, skips some rows
-- SUM(...) returns incorrect value (even committed data wrong!)
```

**Fix:**
```sql
-- Use Read Committed (default)
SELECT * FROM orders WHERE customer_id = 123;

-- If performance critical, use other techniques:
-- Option 1: Add index
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Option 2: Read from replica
SELECT * FROM orders_replica WHERE customer_id = 123;

-- Option 3: Use snapshot isolation (SQL Server)
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
SELECT * FROM orders WHERE customer_id = 123;
-- No dirty reads, no locks, consistent snapshot
```

### Mistake 2: Assuming Read Uncommitted is "Faster"

**Myth:**
```
"Read Uncommitted is 10× faster because no locks!"
```

**Reality:**
```sql
-- Benchmark: PostgreSQL
-- Read Uncommitted (silently upgraded to Read Committed):
SELECT COUNT(*) FROM large_table;
-- Time: 2.5 seconds

-- Read Committed:
SELECT COUNT(*) FROM large_table;
-- Time: 2.5 seconds (same!)

-- Reason: PostgreSQL uses MVCC (no read locks anyway)

-- Benchmark: MySQL
-- Read Uncommitted:
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT COUNT(*) FROM large_table;
-- Time: 2.3 seconds

-- Read Committed:
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT COUNT(*) FROM large_table;
-- Time: 2.4 seconds

-- Difference: 4% (negligible!)
-- Risk: Dirty reads (not worth 4% gain)
```

**Truth:**
- Modern databases (PostgreSQL, InnoDB) use MVCC
- MVCC: Readers don't block writers (no locks on SELECT anyway!)
- Read Uncommitted gains: < 5% in most cases
- Cost: Data integrity risk (not worth it!)

---

## 6. Security Considerations

### 1. Data Leakage via Dirty Reads

**Attack Vector:**
```sql
-- Attacker exploits Read Uncommitted to bypass security checks

-- Scenario: Admin temporarily grants elevated permissions
BEGIN;
UPDATE users SET role = 'admin' WHERE id = 123; -- Testing role change
-- NOT committed yet!

-- Attacker query (Read Uncommitted):
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM users WHERE id = 123;
-- Sees: role = 'admin' (dirty read!)

-- Attacker uses this info to craft privilege escalation attack
-- Knows user 123 being considered for admin (intelligence gathering)

-- Admin rolls back
ROLLBACK;  -- User 123 not actually admin
```

**Mitigation:**
```sql
-- Never use Read Uncommitted for security-sensitive queries
-- Always use Read Committed or higher
SELECT * FROM users WHERE id = 123;  -- Default isolation
```

---

## 7. Performance Optimization

### Optimization 1: Use Covering Indexes Instead of Read Uncommitted

**Problem:**
```sql
-- Slow query (1 second)
SELECT customer_name, order_total 
FROM orders 
WHERE status = 'pending';

-- "Solution": NOLOCK for speed
SELECT customer_name, order_total 
FROM orders WITH (NOLOCK)
WHERE status = 'pending';
-- Time: 0.8 seconds (20% faster, but dirty reads!)
```

**Better Solution:**
```sql
-- Create covering index
CREATE INDEX idx_orders_status_covering 
ON orders(status) 
INCLUDE (customer_name, order_total);

-- Query (no NOLOCK, Read Committed)
SELECT customer_name, order_total 
FROM orders 
WHERE status = 'pending';
-- Time: 0.1 seconds (10× faster than NOLOCK!)
-- Benefit: No dirty reads ✓
```

---

## 8. Examples

### Example 1: Monitoring Dashboard (Acceptable Use)

```sql
-- Dashboard: Approximate active user count
-- Requirement: Update every 5 seconds, exact count unimportant

CREATE OR REPLACE FUNCTION get_approximate_active_users()
RETURNS INTEGER AS $$
DECLARE
    approx_count INTEGER;
BEGIN
    -- Use Read Uncommitted for speed (locks would slow down dashboard)
    SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
    
    SELECT COUNT(*) INTO approx_count
    FROM user_sessions
    WHERE last_active > NOW() - INTERVAL '15 minutes';
    
    -- Risk: May include uncommitted sessions (acceptable)
    -- Benefit: Dashboard responsive, no lock contention
    
    RETURN approx_count;
END;
$$ LANGUAGE plpgsql;

-- Dashboard refresh (every 5 seconds)
-- Shows: ~12,543 active users (approximate)
```

### Example 2: Financial Report (WRONG Use)

```sql
-- ❌ WRONG: Financial report with Read Uncommitted
CREATE OR REPLACE FUNCTION get_daily_revenue(report_date DATE)
RETURNS NUMERIC AS $$
DECLARE
    total_revenue NUMERIC;
BEGIN
    -- WRONG: Read Uncommitted for financial data!
    SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
    
    SELECT SUM(amount) INTO total_revenue
    FROM transactions
    WHERE date = report_date AND status = 'completed';
    
    -- Risk: Includes uncommitted (possibly rolled back) transactions
    -- Result: Incorrect revenue figure (disaster!)
    
    RETURN total_revenue;
END;
$$ LANGUAGE plpgsql;

-- ✅ CORRECT: Use Read Committed (default)
CREATE OR REPLACE FUNCTION get_daily_revenue_correct(report_date DATE)
RETURNS NUMERIC AS $$
DECLARE
    total_revenue NUMERIC;
BEGIN
    -- Default isolation: Read Committed ✓
    SELECT SUM(amount) INTO total_revenue
    FROM transactions
    WHERE date = report_date AND status = 'completed';
    
    -- Guaranteed: Only committed transactions included ✓
    
    RETURN total_revenue;
END;
$$ LANGUAGE plpgsql;
```

---

## 9. Real-World Use Cases

### Use Case: Analytics Query on Replica

**Scenario:** Large analytics query on production data

```python
# Production setup: Primary DB + read replica
# Analytics queries run on replica (not primary)

class AnalyticsService:
    def get_user_growth_stats(self):
        """
        Get approximate user growth metrics for dashboard
        Read from replica: Data may be seconds behind (acceptable)
        """
        # Connect to read replica
        replica_conn = get_replica_connection()
        
        # Read Uncommitted acceptable here because:
        # 1. Replica lag already means slightly stale data
        # 2. Exact counts unimportant for trend charts
        # 3. Avoids lock contention on replica
        
        query = """
        SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
        
        SELECT 
            DATE_TRUNC('day', created_at) AS signup_date,
            COUNT(*) AS new_users
        FROM users
        WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
        GROUP BY DATE_TRUNC('day', created_at)
        ORDER BY signup_date;
        """
        
        rows = replica_conn.execute(query).fetchall()
        
        # Returns approximate daily signups for 30-day chart
        # Slight inaccuracy acceptable (< 1% typically)
        return [{"date": r[0], "users": r[1]} for r in rows]
```

**When NOT to use Read Uncommitted:**
```python
def get_user_balance(user_id):
    """
    Get user account balance for display/withdrawal
    NEVER use Read Uncommitted for financial data!
    """
    # ❌ WRONG:
    # query = "SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED; ..."
    
    # ✅ CORRECT: Default isolation (Read Committed)
    query = "SELECT balance FROM accounts WHERE user_id = %s"
    balance = db.execute(query, (user_id,)).fetchone()[0]
    
    return balance  # Guaranteed committed value ✓
```

---

## 10. Interview Questions

### Q1: What are dirty reads, and when might you intentionally use Read Uncommitted despite the risks?

**Answer:**

**Dirty Reads:**
Reading data modified by another transaction that hasn't committed yet. If that transaction rolls back, you read data that "never existed."

**Example:**
```sql
-- Transaction A
BEGIN;
UPDATE inventory SET stock = 0 WHERE product_id = 100;
-- Not committed!

-- Transaction B (Read Uncommitted)
SELECT stock FROM inventory WHERE product_id = 100;
-- Returns: 0 (dirty read!)

-- Transaction A rolls back
ROLLBACK;  -- Stock back to original value

-- Transaction B used incorrect data (0 instead of actual stock)
```

**Acceptable Use Cases:**

1. **Approximate counts for dashboards:**
   - Use case: "~10,543 active users" (exact count unimportant)
   - Benefit: No lock contention, fast query
   - Risk: Count may be off by < 1% (acceptable)

2. **Development/testing environments:**
   - Use case: Debugging queries
   - Benefit: See WIP data during testing
   - Risk: None (not production)

3. **Read replicas with lag:**
   - Use case: Analytics on replica already seconds behind
   - Benefit: Slightly faster queries
   - Risk: Minimal (replica already has stale data)

**Never Use For:**
- Financial calculations (revenue, balances)
- Inventory/stock levels (overselling risk)
- User authentication/authorization
- Any data where accuracy matters

**Interview insight:** "At [Company], we use Read Committed as default (99.9% of queries). The 0.1% using Read Uncommitted are explicitly documented approximate dashboard counts on read replicas. Any financial query uses Read Committed minimum, typically Repeatable Read for reports spanning multiple queries."

---

## 11. Summary

**Read Uncommitted:**
- **Lowest isolation level** (dirtiest reads possible)
- **Allows dirty reads:** See uncommitted data (may be rolled back)
- **No locks on reads:** Fast, but unsafe
- **Risk:** Data integrity violations, incorrect results

**Dirty Read Example:**
```sql
TXN-A: UPDATE balance = 0 (not committed)
TXN-B: SELECT balance → Returns 0 (dirty read!)
TXN-A: ROLLBACK (balance back to 1000)
TXN-B: Made decisions based on incorrect data (0)
```

**When to Use:**
✅ Approximate counts (dashboards)
✅ Non-critical monitoring queries
✅ Development/testing only
❌ Financial data
❌ Inventory management
❌ Any critical business logic

**Database Support:**
- **PostgreSQL:** NOT supported (silently upgrades to Read Committed)
- **MySQL/InnoDB:** Supported (use cautiously!)
- **SQL Server:** Supported (NOLOCK hint)

**Best Practices:**
1. **Default to Read Committed** (never change without reason)
2. **Document explicitly** if using Read Uncommitted
3. **Use covering indexes** instead of NOLOCK for performance
4. **Never use for financial/critical data**
5. **Monitor for dirty read incidents** (alerts on data anomalies)

**Critical Pattern:**
```sql
-- DON'T DO THIS (unless explicitly justified)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT COUNT(*) FROM table;

-- DO THIS (default, safe)
SELECT COUNT(*) FROM table;  -- Read Committed
```

**Key Takeaway:** "Read Uncommitted trades data integrity for marginal performance gains (< 5%). In modern MVCC databases (PostgreSQL, InnoDB), Read Committed has no lock overhead either, making Read Uncommitted obsolete. Use Read Committed as minimum—the tiny performance gain of Read Uncommitted never justifies the data corruption risk."

---

**Next:** [02_Read_Committed.md](02_Read_Committed.md) - Default isolation level in most databases
