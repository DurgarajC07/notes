# Optimistic Locking - Detect Conflicts at Commit Time

## 1. Concept Explanation

**Optimistic locking** assumes conflicts are rare and detects them at commit time rather than preventing them with locks. Instead of blocking concurrent access, it allows multiple transactions to proceed and rejects conflicting updates.

**Philosophy:**
```
Pessimistic locking: "Lock first, ask questions later"
Optimistic locking: "Proceed freely, detect conflicts at commit"
```

**Core Mechanism: Version Numbers**
```
Every row has a version column:

Initial state:
id | balance | version
123| 1000    | 1

Transaction A reads:
SELECT balance, version FROM accounts WHERE id = 123;
-- balance=1000, version=1

Transaction B reads (concurrent):
SELECT balance, version FROM accounts WHERE id = 123;
-- balance=1000, version=1 (same!)

Transaction A updates:
UPDATE accounts 
SET balance = 900, version = version + 1 
WHERE id = 123 AND version = 1;  -- Check version!
-- 1 row updated ✓ (version now 2)

Transaction B updates (tries):
UPDATE accounts 
SET balance = 1100, version = version + 1 
WHERE id = 123 AND version = 1;  -- Check old version!
-- 0 rows updated ✗ (version mismatch: expected 1, actual 2)
-- Conflict detected! Retry required.
```

**Analogy: Wikipedia Edit Conflict**
```
You edit Wiki page (based on revision 42):
1. Read page revision 42
2. Make changes locally (5 minutes)
3. Save changes

Meanwhile, someone else edited:
- They saved revision 43 (1 minute ago)

Your save attempt:
- Expected revision: 42
- Actual revision: 43
- Result: "Edit conflict! Please merge changes and retry"

Same principle as optimistic locking!
```

---

## 2. Why It Matters

### Production Impact: High-Read, Low-Write Scenarios

**Scenario: Product Catalog (1000 reads/second, 1 write/minute)**

**With Pessimistic Locking (SELECT FOR UPDATE):**
```sql
-- Every READ acquires shared lock
SELECT * FROM products WHERE id = 100 FOR SHARE;
-- 1000 reads/sec × shared lock overhead = High contention

-- Write acquires exclusive lock
UPDATE products SET price = 99.99 WHERE id = 100;
-- Waits for 1000 ongoing reads to release locks
-- Write latency: 100-500ms (unacceptable for admin updates)
```

**With Optimistic Locking (Version Column):**
```sql
-- Reads: No locks!
SELECT * FROM products WHERE id = 100;
-- 1000 reads/sec, 0 lock overhead ✓

-- Write: Check version
UPDATE products 
SET price = 99.99, version = version + 1 
WHERE id = 100 AND version = 42;
-- No waiting (version check is instant)
-- Write latency: < 5ms ✓

-- If conflict (rare, 1 write/minute):
-- Retry (read latest version, apply change)
```

**Result:**
- Read throughput: 10× improvement (no lock overhead)
- Write latency: 100ms → 5ms (95% reduction)
- Conflict rate: < 0.01% (1 write conflicts with another write)

### Real Incident: E-commerce Cart Race Condition

```
Company: E-commerce platform
Issue: Lost cart updates without locks

Timeline:
10:00:00 - Customer A adds Item X to cart
           Initial: cart = []
           Client: Add Item X
           
10:00:00.5 - Customer A adds Item Y (concurrent, slow network)
           Client reads: cart = [] (stale! No lock)
           Client: Add Item Y
           
10:00:01 - Item X update reaches server
           UPDATE cart SET items = '["X"]' WHERE user_id = 123;
           Result: cart = ["X"] ✓
           
10:00:02 - Item Y update reaches server
           UPDATE cart SET items = '["Y"]' WHERE user_id = 123;
           -- Overwrites! (no version check)
           Result: cart = ["Y"] ✗

Customer complaint: "I added Item X but it's gone!"

Root cause: No concurrency control (optimistic or pessimistic)

Fix with optimistic locking:
10:00:00 - Item X:
           Read: cart = [], version = 1
           UPDATE cart SET items = '["X"]', version = 2 WHERE user_id = 123 AND version = 1;
           -- Success ✓
           
10:00:02 - Item Y:
           Read (stale): cart = [], version = 1
           UPDATE cart SET items = '["Y"]', version = 2 WHERE user_id = 123 AND version = 1;
           -- 0 rows updated (version now 2, expected 1)
           -- Conflict detected!
           
           Retry:
           Read (fresh): cart = ["X"], version = 2
           UPDATE cart SET items = '["X", "Y"]', version = 3 WHERE user_id = 123 AND version = 2;
           -- Success ✓

Result: cart = ["X", "Y"] (both items preserved!)
```

---

## 3. Internal Working

### Version Column Strategy

**Implementation:**
```sql
-- Add version column to table
ALTER TABLE accounts ADD COLUMN version INT DEFAULT 0;

-- Application update pattern
BEGIN;

-- Step 1: Read current version
SELECT id, balance, version 
INTO v_id, v_balance, v_version
FROM accounts 
WHERE id = 123;

-- Step 2: Business logic
v_new_balance := v_balance - 100;

-- Step 3: Update with version check
UPDATE accounts 
SET balance = v_new_balance, version = v_version + 1 
WHERE id = 123 AND version = v_version;

-- Step 4: Check if update succeeded
IF SQL%ROWCOUNT = 0 THEN
    RAISE EXCEPTION 'Optimistic lock conflict: row modified by another transaction';
END IF;

COMMIT;
```

**ORM Support (JPA/Hibernate):**
```java
@Entity
public class Account {
    @Id
    private Long id;
    
    private BigDecimal balance;
    
    @Version  // Hibernate manages version automatically
    private Integer version;
}

// Usage (automatic version checking!)
Account account = entityManager.find(Account.class, 123L);
account.setBalance(account.getBalance().subtract(new BigDecimal("100")));
entityManager.merge(account);
// Hibernate generates:
// UPDATE accounts SET balance = ?, version = version + 1 
// WHERE id = ? AND version = ?

// If version mismatch:
// Throws OptimisticLockException
```

### Timestamp-Based Strategy

**Implementation:**
```sql
-- Use last_modified timestamp
ALTER TABLE accounts ADD COLUMN last_modified TIMESTAMP DEFAULT NOW();

-- Update pattern
BEGIN;

-- Read current timestamp
SELECT balance, last_modified 
INTO v_balance, v_last_modified
FROM accounts 
WHERE id = 123;

-- Update with timestamp check
UPDATE accounts 
SET balance = v_balance - 100, 
    last_modified = NOW() 
WHERE id = 123 AND last_modified = v_last_modified;

IF SQL%ROWCOUNT = 0 THEN
    RAISE EXCEPTION 'Concurrent modification detected';
END IF;

COMMIT;
```

**Limitation: Clock Skew**
```
Problem:
- Server A clock: 10:00:00
- Server B clock: 10:00:05 (5 seconds ahead, clock skew)

Transaction A on Server A:
UPDATE accounts SET last_modified = '10:00:00' WHERE id = 123;

Transaction B on Server B:
UPDATE accounts SET last_modified = '10:00:05' WHERE id = 123;

Result: B always "wins" (later timestamp) even if A was actually later!

Solution: Use version numbers (monotonic, clock-independent)
```

### Compare-And-Swap (CAS)

**Atomic CAS in Application:**
```python
# Redis CAS (WATCH + MULTI)
def increment_counter(key):
    while True:
        # Watch key for changes
        redis.watch(key)
        
        # Read current value
        current = int(redis.get(key) or 0)
        
        # Begin transaction
        pipe = redis.pipeline()
        
        # Increment
        pipe.set(key, current + 1)
        
        try:
            # Execute transaction (commits only if key unchanged)
            pipe.execute()
            return current + 1  # Success!
        except redis.WatchError:
            # Key modified by another client → Retry
            continue
```

**PostgreSQL CAS with RETURNING:**
```sql
-- Atomic CAS (single statement)
UPDATE accounts 
SET balance = balance - 100, version = version + 1 
WHERE id = 123 AND version = 42
RETURNING balance, version;

-- Returns:
-- balance | version
-- 900     | 43

-- If no rows returned: Version conflict, retry
```

---

## 4. Best Practices

### Practice 1: Handle Conflicts with Exponential Backoff Retry

**❌ Bad: Immediate Retry (Thundering Herd):**
```python
def withdraw(account_id, amount):
    for attempt in range(100):  # 100 retries!
        account = db.query("SELECT * FROM accounts WHERE id = %s", account_id)
        
        if account.balance >= amount:
            rows = db.execute("""
                UPDATE accounts 
                SET balance = balance - %s, version = version + 1 
                WHERE id = %s AND version = %s
            """, (amount, account_id, account.version))
            
            if rows == 1:
                return True  # Success
        
        # Immediate retry (no delay!)
    
    return False  # Failed after 100 retries

# Problem:
# 100 concurrent transactions retry simultaneously
# CPU: 100% (busy loop)
# Success rate: Low (all conflict with each other)
```

**✅ Good: Exponential Backoff:**
```python
import time
import random

def withdraw_with_backoff(account_id, amount, max_retries=5):
    for attempt in range(max_retries):
        try:
            account = db.query("SELECT * FROM accounts WHERE id = %s", account_id)
            
            if account.balance < amount:
                raise InsufficientFunds()
            
            rows = db.execute("""
                UPDATE accounts 
                SET balance = balance - %s, version = version + 1 
                WHERE id = %s AND version = %s
            """, (amount, account_id, account.version))
            
            if rows == 1:
                return True  # Success!
            
            # Version conflict → Retry with backoff
            if attempt < max_retries - 1:
                # Exponential backoff: 10ms, 20ms, 40ms, 80ms, 160ms
                wait_time = 0.01 * (2 ** attempt)
                # Add jitter (randomness to spread retries)
                jitter = wait_time * random.uniform(0.5, 1.5)
                time.sleep(jitter)
        
        except OptimisticLockException:
            # Retry
            continue
    
    raise MaxRetriesExceeded(f"Failed after {max_retries} attempts")

# Benefits:
# - Conflicts spread out (not all retry simultaneously)
# - Success rate: 99%+ (even with high concurrency)
# - CPU: Efficient (sleep during backoff)
```

### Practice 2: Choose Optimistic vs Pessimistic Based on Contention

**Decision Matrix:**

| Contention Level | Read:Write Ratio | Strategy | Reason |
|------------------|------------------|----------|--------|
| Low (< 1% conflict) | Any | Optimistic | Low overhead, rare retries |
| Medium (1-10%) | > 10:1 (read-heavy) | Optimistic | Read performance outweighs retry cost |
| Medium (1-10%) | < 1:1 (write-heavy) | Pessimistic | Too many retries, lock is cheaper |
| High (> 10%) | Any | Pessimistic | Retry storm, lock prevents wasted work |

**Hybrid Approach:**
```python
def update_account(account_id, operation):
    # Check historical conflict rate
    conflict_rate = get_conflict_rate(account_id)  # e.g., 0.05 (5%)
    
    if conflict_rate < 0.01:
        # Low conflict: Use optimistic locking
        return optimistic_update(account_id, operation)
    else:
        # High conflict: Use pessimistic locking
        return pessimistic_update(account_id, operation)

def optimistic_update(account_id, operation):
    account = db.query("SELECT * FROM accounts WHERE id = %s", account_id)
    new_balance = operation(account.balance)
    
    rows = db.execute("""
        UPDATE accounts SET balance = %s, version = version + 1 
        WHERE id = %s AND version = %s
    """, (new_balance, account_id, account.version))
    
    if rows == 0:
        raise OptimisticLockConflict()

def pessimistic_update(account_id, operation):
    account = db.query("SELECT * FROM accounts WHERE id = %s FOR UPDATE", account_id)
    new_balance = operation(account.balance)
    
    db.execute("""
        UPDATE accounts SET balance = %s WHERE id = %s
    """, (new_balance, account_id))
```

---

## 5. Common Mistakes

### Mistake 1: Not Handling OptimisticLockException (Silent Failures)

**Problem:**
```python
# Bad: Ignores version conflict
def update_user_profile(user_id, updates):
    user = User.objects.get(id=user_id)
    user.name = updates['name']
    user.email = updates['email']
    user.save()  # May silently fail on version conflict!
    
    return "Updated successfully"  # Lies! Maybe not updated

# Result:
# - Client thinks update succeeded
# - Database unchanged (version conflict)
# - User confused: "I changed my name but it's still wrong!"
```

**Fix:**
```python
from django.db import transaction
from django.db.models import F

def update_user_profile(user_id, updates, max_retries=3):
    for attempt in range(max_retries):
        try:
            with transaction.atomic():
                # Lock by version
                user = User.objects.select_for_update().get(id=user_id)
                
                # Apply updates
                user.name = updates['name']
                user.email = updates['email']
                user.version = F('version') + 1  # Increment version
                user.save()
                
                return "Updated successfully"
        
        except OptimisticLockException:
            if attempt == max_retries - 1:
                raise ConcurrentModificationError("Update failed due to conflict")
            # Retry with backoff
            time.sleep(0.01 * (2 ** attempt))
    
    raise MaxRetriesExceeded()
```

### Mistake 2: Incrementing Version on Every Read

**Problem:**
```sql
-- Mistaken version increment
SELECT * FROM accounts WHERE id = 123;

-- Wrong: Increment version on read (!)
UPDATE accounts SET version = version + 1 WHERE id = 123;

-- Result:
-- - Version increases with every SELECT
-- - False conflicts: Version changed even though balance didn't
-- - Retry storm: All concurrent reads cause version conflicts
```

**Correct:**
```sql
-- Only increment version on WRITE
SELECT * FROM accounts WHERE id = 123;
-- Version unchanged ✓

UPDATE accounts SET balance = 900, version = version + 1 WHERE id = 123;
-- Version incremented only on write ✓
```

---

## 6. Security Considerations

### 1. Version Number Overflow

**Problem:**
```sql
-- Version column: INT (4 bytes, max 2,147,483,647)
-- High-frequency updates: 1000 updates/second
-- Overflow: 2.1B / 1000 / 86400 = 24 days

-- Day 25:
UPDATE accounts SET version = version + 1 WHERE id = 123;
-- version = 2,147,483,648 → Overflow! (becomes -2,147,483,648)

-- Next update:
UPDATE accounts SET version = version + 1 WHERE id = -2,147,483,648 AND ...;
-- 0 rows updated (negative version never matches!)
-- All updates fail ✗
```

**Fix:**
```sql
-- Use BIGINT (8 bytes, max 9.2 quintillion)
ALTER TABLE accounts ALTER COLUMN version TYPE BIGINT;

-- Overflow: 9.2 × 10^18 / 1000 / 86400 / 365 = 292 million years ✓
```

---

## 7. Performance Optimization

### Optimization 1: Batch Version Checks

**Slow: Individual Version Checks**
```python
# Update 100 rows
for row_id in range(1, 101):
    row = db.query("SELECT * FROM data WHERE id = %s", row_id)
    db.execute("""
        UPDATE data SET value = %s, version = version + 1 
        WHERE id = %s AND version = %s
    """, (new_value, row_id, row.version))

# 100 SELECT + 100 UPDATE = 200 queries (slow!)
```

**Fast: Batch Update with CTE**
```sql
-- Single query updates all rows with version checks
WITH current_data AS (
    SELECT id, version FROM data WHERE id = ANY(%s)
),
updates AS (
    SELECT 
        unnest(%s::int[]) AS id,
        unnest(%s::int[]) AS expected_version,
        unnest(%s::text[]) AS new_value
)
UPDATE data
SET value = updates.new_value, version = data.version + 1
FROM updates
WHERE data.id = updates.id 
    AND data.version = updates.expected_version
RETURNING data.id, data.version;

-- Returns IDs of successfully updated rows
-- Missing IDs = version conflicts (retry individually)
```

---

## 8. Examples

### Example 1: Inventory Management with Optimistic Locking

```python
class InventoryService:
    def reserve_stock(self, product_id, quantity, max_retries=5):
        """Reserve stock with optimistic locking."""
        for attempt in range(max_retries):
            # Read current stock and version
            product = db.query("""
                SELECT id, stock, version 
                FROM inventory 
                WHERE id = %s
            """, product_id)
            
            if product.stock < quantity:
                raise InsufficientStock(f"Only {product.stock} available")
            
            # Attempt to update with version check
            rows_updated = db.execute("""
                UPDATE inventory 
                SET stock = stock - %s, version = version + 1 
                WHERE id = %s AND version = %s
            """, (quantity, product_id, product.version))
            
            if rows_updated == 1:
                # Success! Stock reserved
                return {
                    'product_id': product_id,
                    'reserved': quantity,
                    'remaining': product.stock - quantity
                }
            
            # Version conflict → Retry with backoff
            if attempt < max_retries - 1:
                wait_time = 0.01 * (2 ** attempt) * random.uniform(0.8, 1.2)
                time.sleep(wait_time)
        
        raise OptimisticLockError(f"Failed to reserve stock after {max_retries} attempts")

# Usage
try:
    result = service.reserve_stock(product_id=100, quantity=5)
    print(f"Reserved {result['reserved']}, {result['remaining']} remaining")
except InsufficientStock as e:
    print(f"Out of stock: {e}")
except OptimisticLockError as e:
    print(f"High contention, try again: {e}")
```

---

## 9. Real-World Use Cases

### Use Case: Google Docs Collaborative Editing

**Challenge:** 10 users edit same document simultaneously

**Strategy: Operational Transformation + Optimistic Locking**
```javascript
// Each edit has version number
{
    document_id: 'doc_123',
    version: 42,
    edit: {
        position: 100,
        insert: 'Hello ',
        user: 'alice@example.com'
    }
}

// Server applies edit:
function applyEdit(edit) {
    // Read current document version
    const doc = readDocument(edit.document_id);
    
    if (edit.version !== doc.version) {
        // Version conflict: Transform edit to new version
        const transformed = transformEdit(edit, doc.recentEdits);
        edit = transformed;
    }
    
    // Apply edit with optimistic lock
    const success = updateDocument({
        id: edit.document_id,
        content: applyOperation(doc.content, edit),
        version: doc.version + 1,
        expectedVersion: doc.version  // Optimistic lock check
    });
    
    if (!success) {
        // Conflict (another edit applied first) → Retry
        return applyEdit(edit);
    }
    
    // Broadcast to all users
    broadcast(edit);
}

// Result:
// - 10 concurrent edits/sec
// - Version conflicts: < 1% (rare, handled transparently)
// - User experience: Smooth, real-time collaboration
```

---

## 10. Interview Questions

### Q1: When would you choose optimistic locking over pessimistic locking, and vice versa?

**Answer:**

**Choose Optimistic Locking When:**

1. **Read-heavy workload (> 10:1 read:write ratio):**
   - Example: Product catalog (1000 reads/sec, 1 update/min)
   - Reason: No lock overhead on reads, rare conflicts
   - Benefit: 10× read throughput

2. **Low contention (< 1% write conflicts):**
   - Example: User profile updates (each user updates own profile)
   - Reason: Conflicts extremely rare, retry cost negligible
   - Benefit: Simplified code, no deadlocks

3. **Long-running operations:**
   - Example: User edits form for 5 minutes before saving
   - Reason: Holding lock for 5 minutes blocks others
   - Optimistic: Check version at save time (instant)

**Choose Pessimistic Locking When:**

1. **Write-heavy workload (< 2:1 read:write ratio):**
   - Example: High-frequency counter (1000 increments/sec)
   - Reason: Constant conflicts → Retry storm
   - Pessimistic: Lock prevents wasted work

2. **High contention (> 10% write conflicts):**
   - Example: Limited inventory (100 users buying last 5 items)
   - Reason: Optimistic → 95% retry rate (wasteful)
   - Pessimistic: Queue users (first-come-first-served)

3. **Critical operations (no retry acceptable):**
   - Example: Financial transfer (regulatory requirement: no retry delays)
   - Reason: Business requirement for immediate consistency
   - Pessimistic: Lock guarantees success

**Hybrid Example:**
```python
def update_balance(account_id, amount):
    # Check account contention
    if is_hot_account(account_id):  # e.g., merchant account with 1000 TPS
        # Pessimistic locking
        with db.transaction():
            account = Account.objects.select_for_update().get(id=account_id)
            account.balance += amount
            account.save()
    else:
        # Optimistic locking
        retry_with_exponential_backoff(
            lambda: update_with_version_check(account_id, amount)
        )
```

**Interview insight:** "At [Company], we use optimistic locking for 95% of tables (user profiles, orders, products). For the 5% hot tables (inventory, counters), we use pessimistic locking. Decision based on metrics: if retry rate > 5%, we switch to pessimistic. Monitor with `conflict_rate = failed_updates / total_updates`."

---

## 11. Summary

**Optimistic Locking:**
- **Assumption:** Conflicts are rare
- **Method:** Detect conflicts at commit time (version check)
- **Benefits:** High read throughput, no lock overhead, no deadlocks
- **Trade-offs:** Retry overhead, eventual consistency

**Implementation:**
```sql
-- Version column
ALTER TABLE table_name ADD COLUMN version INT DEFAULT 0;

-- Update pattern
UPDATE table_name 
SET value = new_value, version = version + 1 
WHERE id = target_id AND version = expected_version;

-- Check rows_affected == 1 (success) or 0 (conflict)
```

**When to Use:**
- ✅ Read-heavy workloads (> 10:1 read:write)
- ✅ Low contention (< 1% conflicts)
- ✅ Long-running user operations (held form for minutes)
- ❌ Write-heavy workloads (retry storm)
- ❌ High contention (> 10% conflicts)
- ❌ Critical no-retry operations (financial)

**Best Practices:**
- Retry with exponential backoff (avoid thundering herd)
- Use BIGINT for version (prevent overflow)
- Handle OptimisticLockException explicitly (don't ignore!)
- Monitor conflict rate: Switch to pessimistic if > 5%

**Critical Pattern:**
```python
for attempt in range(max_retries):
    entity = read_entity(id)
    new_value = compute(entity.value)
    
    if update_with_version_check(id, new_value, entity.version):
        return  # Success!
    
    # Conflict → Exponential backoff
    time.sleep(0.01 * (2 ** attempt) * random.uniform(0.8, 1.2))

raise MaxRetriesExceeded()
```

**Key Takeaway:** "Optimistic locking trades lock overhead for retry cost. Ideal for read-heavy, low-contention workloads. In production: Monitor conflict rate and switch to pessimistic if retries exceed 5%. Hybrid approach often best: optimistic for most tables, pessimistic for hot tables."

---

**Next:** [08_README.md](08_README.md) - Section overview and concurrency pattern decision guide

