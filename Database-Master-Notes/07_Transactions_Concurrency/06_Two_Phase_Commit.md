# Two-Phase Commit - Distributed Transaction Coordination

## 1. Concept Explanation

**Two-Phase Commit (2PC)** is a distributed transaction protocol that ensures all participants in a transaction either commit or abort together, maintaining atomicity across multiple databases.

**Problem: Distributed Atomicity**
```
Scenario: Transfer $100 from Bank A to Bank B (separate databases)

Without 2PC:
1. Deduct $100 from Bank A → Success ✓
2. Network failure
3. Credit $100 to Bank B → Failed ✗
Result: Money disappeared! ($100 vanished)

With 2PC:
1. Ask both banks: "Can you commit?"
2. Both respond: "Yes, ready to commit"
3. Tell both: "COMMIT now"
Result: Either both commit or both abort (atomic!)
```

### Two Phases

**Phase 1: Prepare (Voting)**
```
Coordinator → Participants: "PREPARE to commit"
Participants:
  - Write redo/undo logs
  - Lock resources
  - Respond: "YES (ready)" or "NO (abort)"

Coordinator:
  - If ALL say "YES" → Proceed to Phase 2
  - If ANY say "NO" → Send "ABORT" to all
```

**Phase 2: Commit/Abort (Decision)**
```
Coordinator → Participants: "COMMIT" (or "ABORT")
Participants:
  - Apply changes (commit) or discard (abort)
  - Release locks
  - Respond: "DONE"

Coordinator:
  - Wait for all "DONE" acknowledgments
  - Transaction complete
```

### Visual Flow

```
Timeline:

Coordinator         Participant A             Participant B
    |                      |                         |
    | PREPARE              |                         |
    |--------------------->|                         |
    |                      | Write undo/redo logs    |
    |                      | Lock resources          |
    |                      |                         |
    |       YES (ready)    |                         |
    |<---------------------|                         |
    |                                                |
    | PREPARE                                        |
    |----------------------------------------------->|
    |                                                | Write logs
    |                                                | Lock resources
    |                                  YES (ready)   |
    |<-----------------------------------------------|
    |                                                |
    | [Decision: All voted YES]                      |
    |                                                |
    | COMMIT               |                         |
    |--------------------->|                         |
    |                      | Apply changes           |
    |                      | Release locks           |
    |       DONE           |                         |
    |<---------------------|                         |
    |                                                |
    | COMMIT                                         |
    |----------------------------------------------->|
    |                                                | Apply changes
    |                                                | Release locks
    |                                   DONE         |
    |<-----------------------------------------------|
    |                                                |
    | Transaction complete ✓                         |
```

---

## 2. Why It Matters

### Production Impact: Payment Processing Across Services

**Scenario: Booking System (Hotel + Flight)**
```
User books:
- Hotel room: $500
- Flight: $300
Total: $800

Services:
- Hotel database: db_hotel
- Flight database: db_flight
- Payment database: db_payment

Requirement: All or nothing (atomic booking)
```

**Without 2PC (Partial Failure):**
```
10:00:00 - User clicks "Book"

10:00:01 - Charge payment ($800)
BEGIN;
INSERT INTO payments (user_id, amount) VALUES (123, 800);
COMMIT;  -- Success on db_payment ✓

10:00:02 - Book hotel
BEGIN;
UPDATE rooms SET available = false WHERE id = 501;
COMMIT;  -- Success on db_hotel ✓

10:00:03 - Book flight
BEGIN;
UPDATE seats SET booked = true WHERE id = 'A12';
-- Network timeout! (20 seconds)
ROLLBACK;  -- Failed on db_flight ✗

Result:
- User charged $800 ✓
- Hotel booked ✓
- Flight NOT booked ✗

Customer: "I paid $800 but no flight!"
Support: Manual refund, hotel cancellation (hours of work)
```

**With 2PC (Atomic Across All):**
```
10:00:00 - User clicks "Book"

10:00:01 - Coordinator: Start distributed transaction
PREPARE on db_payment:
    BEGIN;
    INSERT INTO payments (user_id, amount) VALUES (123, 800);
    -- Don't commit yet, write to log
    Response: "YES, ready to commit"

PREPARE on db_hotel:
    BEGIN;
    UPDATE rooms SET available = false WHERE id = 501;
    -- Don't commit yet, write to log
    Response: "YES, ready to commit"

PREPARE on db_flight:
    BEGIN;
    UPDATE seats SET booked = true WHERE id = 'A12';
    -- Network timeout!
    Response: "NO, failed to prepare"

10:00:23 - Coordinator: Flight said "NO" → ABORT all

ABORT to db_payment:
    ROLLBACK;  -- Payment undone ✗

ABORT to db_hotel:
    ROLLBACK;  -- Hotel undone ✗

Flight already failed ✗

Result:
- User NOT charged ✓
- Hotel NOT booked ✓
- Flight NOT booked ✓
- Consistency maintained!

Customer: "Booking failed, please try again"
Support: No manual intervention needed ✓
```

---

## 3. Internal Working

### XA Transaction Protocol (X/Open Distributed Transaction Architecture)

**XA Interface Functions:**
```c
// Phase 1: Prepare
xa_prepare(xid)
// Returns:
//   XA_OK: Ready to commit
//   XA_RBROLLBACK: Vote to abort

// Phase 2: Commit or Rollback
xa_commit(xid, flags)
xa_rollback(xid)

// Recovery
xa_recover(xids[], count)
```

**PostgreSQL Implementation:**
```sql
-- Start distributed transaction
BEGIN;
-- Perform work
UPDATE accounts SET balance = balance - 100 WHERE id = 123;

-- Phase 1: Prepare
PREPARE TRANSACTION 'tx_2024_001';
-- State: Transaction prepared (in limbo)
-- Resources locked, changes logged
-- Waiting for COMMIT or ROLLBACK

-- Query prepared transactions
SELECT * FROM pg_prepared_xacts;
/*
 gid         | prepared           | owner    | database
-------------+--------------------+----------+----------
 tx_2024_001 | 2024-01-15 10:30:00| postgres | mydb
*/

-- Phase 2: Coordinator decides
-- Option A: All voted YES
COMMIT PREPARED 'tx_2024_001';

-- Option B: Any voted NO
ROLLBACK PREPARED 'tx_2024_001';
```

### Coordinator State Machine

```
States:

INIT → PREPARE_SENT → COMMITTED / ABORTED

Transitions:

1. INIT:
   - Coordinator receives transaction request
   - Assigns global transaction ID (XID)

2. PREPARE_SENT:
   - Send PREPARE to all participants
   - Wait for responses (with timeout)
   
   If ALL respond "YES":
       State → COMMIT_SENT
   If ANY respond "NO" OR timeout:
       State → ABORT_SENT

3. COMMIT_SENT:
   - Write "COMMIT decision" to coordinator log (critical!)
   - Send COMMIT to all participants
   - Wait for acknowledgments
   - State → COMMITTED

4. ABORT_SENT:
   - Write "ABORT decision" to coordinator log
   - Send ROLLBACK to all participants
   - Wait for acknowledgments
   - State → ABORTED
```

**Coordinator Log (Durable Decision):**
```
Why log decision before sending COMMIT?
- If coordinator crashes after Phase 1 but before Phase 2
- Participants are in "prepared" state (locked resources)
- On coordinator recovery: Read log → Knows decision → Resend COMMIT/ABORT
```

---

## 4. Best Practices

### Practice 1: Set Reasonable Timeouts

**Problem: Indefinite Wait**
```sql
-- Participant prepared, waits for coordinator decision
PREPARE TRANSACTION 'tx_001';
-- [Coordinator crashes and never recovers]
-- Resources locked FOREVER!
-- Other transactions blocked indefinitely
```

**Solution: Prepared Transaction Timeout**
```sql
-- PostgreSQL: max_prepared_transactions and monitoring
SHOW max_prepared_transactions;  -- Default: 0 (disabled)

-- Enable prepared transactions
ALTER SYSTEM SET max_prepared_transactions = 100;

-- Monitor prepared transactions age
SELECT
    gid,
    prepared,
    NOW() - prepared AS age,
    owner,
    database
FROM pg_prepared_xacts
WHERE NOW() - prepared > INTERVAL '5 minutes';

-- Alert if > 5 minutes old (coordinator likely crashed)
-- Manual resolution: ROLLBACK PREPARED 'gid';
```

### Practice 2: Minimize Prepared Transaction Duration

**❌ Bad: Long-Running Prepared Transaction**
```python
# Participant A
conn_a.execute("BEGIN;")
conn_a.execute("UPDATE inventory SET stock = stock - 1 WHERE id = 100;")
conn_a.execute("PREPARE TRANSACTION 'tx_001';")

# Do expensive computation (5 minutes!)
result = complex_calculation()

# Coordinator decides
if result.success:
    conn_a.execute("COMMIT PREPARED 'tx_001';")
# Locks held for 5 minutes!
```

**✅ Good: Prepare Last**
```python
# Do computation first (no locks held)
result = complex_calculation()

if result.success:
    # THEN start transaction
    conn_a.execute("BEGIN;")
    conn_a.execute("UPDATE inventory SET stock = stock - 1 WHERE id = 100;")
    conn_a.execute("PREPARE TRANSACTION 'tx_001';")
    
    # Immediately commit (locks held < 1 second)
    conn_a.execute("COMMIT PREPARED 'tx_001';")
```

---

## 5. Common Mistakes

### Mistake 1: Coordinator Crash During Phase 2 (Blocking)

**Problem:**
```
Phase 1 complete:
- Participant A: PREPARED (locks held)
- Participant B: PREPARED (locks held)

Coordinator decides: COMMIT
- Writes "COMMIT" to coordinator log ✓
- Sends COMMIT to Participant A ✓
- [Coordinator crashes before sending COMMIT to B]

State:
- Participant A: Committed (locks released) ✓
- Participant B: Prepared (locks held FOREVER!) ✗

Impact:
- All transactions accessing B's locked resources BLOCKED
- Manual intervention required
```

**Resolution:**
```sql
-- On Participant B:
-- Check prepared transactions
SELECT * FROM pg_prepared_xacts;

-- Manual decision (after confirming with Participant A):
-- If A committed → B should commit
COMMIT PREPARED 'tx_001';

-- Locks released, system unblocked
```

**Prevention: Coordinator Replication**
```
Use replicated coordinator (e.g., ZooKeeper, etcd):
- Primary coordinator fails → Secondary takes over
- Secondary reads coordinator log → Knows decision
- Secondary resends COMMIT to Participant B
- Automatic recovery ✓
```

### Mistake 2: Forgetting prepare_transaction_wait_timeout

**Problem:**
```sql
-- Participant waits indefinitely for coordinator
PREPARE TRANSACTION 'tx_001';
-- [Coordinator never sends COMMIT/ROLLBACK]
-- Locks held forever, deadlock cascade
```

**Fix:**
```python
# Application-level timeout
import time
import threading

def prepare_with_timeout(conn, xid, timeout=30):
    prepared = threading.Event()
    
    def prepare():
        conn.execute(f"PREPARE TRANSACTION '{xid}';")
        prepared.set()
    
    thread = threading.Thread(target=prepare)
    thread.start()
    
    if not prepared.wait(timeout):
        # Timeout exceeded
        conn.execute(f"ROLLBACK PREPARED '{xid}';")
        raise TimeoutError(f"Prepare timeout after {timeout}s")
    
    thread.join()

# Usage
try:
    prepare_with_timeout(conn, 'tx_001', timeout=30)
except TimeoutError:
    # Coordinator likely crashed, abort transaction
    pass
```

---

## 6. Security Considerations

### 1. Coordinator Spoofing

**Attack:**
```
Attacker sends fake "COMMIT" messages to participants:
- Participant A receives "COMMIT" for transaction 'tx_malicious'
- Participant A commits (without proper authorization)
- Result: Unauthorized transaction committed
```

**Mitigation:**
```
1. Authentication:
   - Coordinator and participants authenticate via mutual TLS
   - Certificate validation

2. Transaction ID validation:
   - Participants only accept COMMIT for transactions they prepared
   - Reject COMMIT for unknown transaction IDs

3. Coordinator IP whitelist:
   - Participants only accept commands from known coordinator IPs
```

---

## 7. Performance Optimization

### Optimization 1: Optimize for Common Case (All Commit)

**Optimization: Presumed Abort**
```
Optimization:
- Assume most transactions commit (99%+)
- Only log ABORT decisions (rare)
- COMMIT decisions implicit (not logged)

Benefit:
- Reduced coordinator log writes (1% vs 100%)
- Faster commit path

Trade-off:
- On coordinator crash: Participants must query coordinator state
- If no record → Presume ABORT (safe, conservative)
```

**Implementation:**
```python
class Coordinator:
    def commit_transaction(self, xid, participants):
        # Phase 1: Prepare
        responses = []
        for p in participants:
            response = p.prepare(xid)
            responses.append(response)
        
        # Phase 2: Decision
        if all(r == "YES" for r in responses):
            # All YES → COMMIT (no log needed with presumed abort)
            for p in participants:
                p.commit(xid)
            return "COMMITTED"
        else:
            # Any NO → ABORT (log this rare case)
            self.log_abort(xid)  # Explicit abort log
            for p in participants:
                p.rollback(xid)
            return "ABORTED"
    
    def recover_transaction(self, xid):
        # Check abort log
        if self.log_contains_abort(xid):
            return "ABORTED"
        else:
            # Not in log → Presume abort (safe)
            return "PRESUMED_ABORTED"
```

---

## 8. Examples

### Example 1: Distributed Payment System

```python
import psycopg2

class TwoPhaseCoordinator:
    def __init__(self):
        self.participants = {
            'payment_db': psycopg2.connect("host=db1 dbname=payments"),
            'inventory_db': psycopg2.connect("host=db2 dbname=inventory"),
            'order_db': psycopg2.connect("host=db3 dbname=orders")
        }
    
    def execute_order(self, user_id, product_id, amount, price):
        import uuid
        xid = f"order_{uuid.uuid4()}"
        
        try:
            # Phase 1: Prepare all participants
            # Participant 1: Payment
            conn_payment = self.participants['payment_db']
            cursor = conn_payment.cursor()
            cursor.execute("BEGIN;")
            cursor.execute(
                "INSERT INTO payments (user_id, amount) VALUES (%s, %s)",
                (user_id, price)
            )
            cursor.execute(f"PREPARE TRANSACTION '{xid}_payment';")
            
            # Participant 2: Inventory
            conn_inventory = self.participants['inventory_db']
            cursor = conn_inventory.cursor()
            cursor.execute("BEGIN;")
            cursor.execute(
                "UPDATE products SET stock = stock - %s WHERE id = %s AND stock >= %s",
                (amount, product_id, amount)
            )
            if cursor.rowcount == 0:
                raise Exception("Insufficient stock")
            cursor.execute(f"PREPARE TRANSACTION '{xid}_inventory';")
            
            # Participant 3: Order
            conn_order = self.participants['order_db']
            cursor = conn_order.cursor()
            cursor.execute("BEGIN;")
            cursor.execute(
                "INSERT INTO orders (user_id, product_id, amount, price) VALUES (%s, %s, %s, %s)",
                (user_id, product_id, amount, price)
            )
            cursor.execute(f"PREPARE TRANSACTION '{xid}_order';")
            
            # Phase 2: All prepared successfully → COMMIT
            self.participants['payment_db'].cursor().execute(f"COMMIT PREPARED '{xid}_payment';")
            self.participants['inventory_db'].cursor().execute(f"COMMIT PREPARED '{xid}_inventory';")
            self.participants['order_db'].cursor().execute(f"COMMIT PREPARED '{xid}_order';")
            
            return {"status": "success", "order_id": xid}
        
        except Exception as e:
            # Phase 2: Any failure → ROLLBACK all
            try:
                self.participants['payment_db'].cursor().execute(f"ROLLBACK PREPARED '{xid}_payment';")
            except:
                pass
            try:
                self.participants['inventory_db'].cursor().execute(f"ROLLBACK PREPARED '{xid}_inventory';")
            except:
                pass
            try:
                self.participants['order_db'].cursor().execute(f"ROLLBACK PREPARED '{xid}_order';")
            except:
                pass
            
            return {"status": "failed", "error": str(e)}

# Usage
coordinator = TwoPhaseCoordinator()
result = coordinator.execute_order(
    user_id=123,
    product_id=456,
    amount=2,
    price=50.00
)
print(result)
# {"status": "success", "order_id": "order_abc123"}
```

---

## 9. Real-World Use Cases

### Use Case: Banking Consortium Transfer

**Scenario:** Transfer between banks (different systems)

```
Banks:
- Bank A (PostgreSQL)
- Bank B (MySQL)
- Clearing House (Coordinator, Oracle)

Transfer: $10,000 from Bank A Account 123 to Bank B Account 456
```

**Implementation:**
```python
# Clearing House Coordinator
class BankTransferCoordinator:
    def transfer(self, from_bank, from_account, to_bank, to_account, amount):
        xid = f"xfer_{uuid.uuid4()}"
        
        try:
            # Phase 1: Prepare
            # Bank A (PostgreSQL)
            conn_a.execute("BEGIN;")
            conn_a.execute(f"UPDATE accounts SET balance = balance - {amount} WHERE id = {from_account};")
            conn_a.execute(f"PREPARE TRANSACTION '{xid}_bank_a';")
            
            # Bank B (MySQL via XA)
            conn_b.execute(f"XA START '{xid}_bank_b';")
            conn_b.execute(f"UPDATE accounts SET balance = balance + {amount} WHERE id = {to_account};")
            conn_b.execute(f"XA END '{xid}_bank_b';")
            conn_b.execute(f"XA PREPARE '{xid}_bank_b';")
            
            # Phase 2: Commit
            # Log decision first (durability)
            clearing_house_log.write(f"{xid}: COMMIT")
            
            # Execute commit
            conn_a.execute(f"COMMIT PREPARED '{xid}_bank_a';")
            conn_b.execute(f"XA COMMIT '{xid}_bank_b';")
            
            return "SUCCESS"
        
        except Exception as e:
            # Log abort
            clearing_house_log.write(f"{xid}: ABORT - {str(e)}")
            
            # Rollback both
            try:
                conn_a.execute(f"ROLLBACK PREPARED '{xid}_bank_a';")
            except:
                pass
            try:
                conn_b.execute(f"XA ROLLBACK '{xid}_bank_b';")
            except:
                pass
            
            return f"FAILED: {str(e)}"

# Result:
# - Either both banks commit (atomic) ✓
# - Or neither commits (rollback) ✓
# - No partial state (money lost/duplicated) ✓
```

---

## 10. Interview Questions

### Q1: What are the limitations of Two-Phase Commit, and when would you consider alternatives?

**Answer:**

**Limitations of 2PC:**

1. **Blocking:**
   - If coordinator crashes after Phase 1 (participants prepared)
   - Participants hold locks indefinitely
   - System frozen until coordinator recovers

2. **Performance:**
   - Two network round-trips (PREPARE, then COMMIT)
   - Locks held during both phases (longer than single-DB transaction)
   - Throughput: 100-500 TPS (vs 10K+ TPS single-DB)

3. **Availability:**
   - Any participant down → Entire transaction aborts
   - N participants → Availability = (availability_per_participant)^N
   - Example: 3 participants at 99.9% → Combined 99.7% (worse!)

**Alternatives:**

**1. Saga Pattern (Event-Driven Compensation):**
```python
# Instead of atomic 2PC, use compensating transactions

# Forward transactions:
def book_order():
    payment_id = charge_payment()              # T1
    hotel_id = book_hotel()                    # T2
    flight_id = book_flight()                  # T3

# Compensating transactions (if T3 fails):
def compensate():
    cancel_hotel(hotel_id)                     # C2
    refund_payment(payment_id)                 # C1

# Use case: Long-running workflows (minutes/hours)
# Benefit: No locks, high availability
# Trade-off: Eventual consistency (not immediate)
```

**2. Idempotent Operations with Retries:**
```python
# Avoid distributed transactions entirely
# Use idempotent APIs (safe to retry)

def transfer_with_idempotency(transfer_id, from_account, to_account, amount):
    # Check if already processed (idempotency)
    if transfer_log.exists(transfer_id):
        return "ALREADY_PROCESSED"
    
    # Perform transfer (single DB, atomic)
    deduct(from_account, amount)
    credit(to_account, amount)
    transfer_log.record(transfer_id)
    
    # Safe to retry on failure (idempotency check prevents double-apply)

# Use case: Payment processing (Stripe pattern)
```

**Decision Matrix:**
| Criteria | Use 2PC | Use Saga | Use Idempotency |
|----------|---------|----------|-----------------|
| Latency requirement | < 100ms | > 1s | < 100ms |
| Consistency | Strong (immediate) | Eventual | Strong (immediate) |
| Availability | Low (N-way dependency) | High | Medium |
| Complexity | Medium | High | Low |

**Interview insight:** "At [Company], we replaced 2PC with saga pattern for order processing (payment + inventory + shipping). Benefit: Availability improved 99.7% → 99.95% (no blocking). Trade-off: Eventual consistency acceptable for order flow (compensate with refunds if needed). For bank transfers requiring strong consistency, we still use 2PC but minimize participants (2 max) to reduce blocking risk."

---

## 11. Summary

**Two-Phase Commit (2PC):**
- **Purpose:** Atomic transactions across multiple databases
- **Phase 1 (PREPARE):** All participants vote (YES/NO)
- **Phase 2 (COMMIT/ABORT):** Coordinator decides based on votes

**Protocol:**
```
Coordinator: PREPARE → Participants: YES/NO
If all YES: Coordinator: COMMIT → Participants: Apply
If any NO: Coordinator: ABORT → Participants: Rollback
```

**Limitations:**
- **Blocking:** Coordinator crash → Participants stuck in prepared state
- **Performance:** 2 network round-trips, locks held longer
- **Availability:** Any participant down → Transaction aborts

**Best Practices:**
- Set timeouts (`max_prepared_transactions`, monitor age)
- Minimize prepared transaction duration (prepare last)
- Replicate coordinator for high availability
- Consider alternatives (saga pattern) for long-running workflows

**Critical Pattern:**
```sql
-- Participant
BEGIN;
-- Perform work
PREPARE TRANSACTION 'tx_id';
-- WAIT for coordinator decision
COMMIT PREPARED 'tx_id';  -- or ROLLBACK PREPARED
```

**Monitoring:**
```sql
-- Check prepared transactions
SELECT * FROM pg_prepared_xacts WHERE NOW() - prepared > INTERVAL '5 minutes';
-- Alert on stale prepared transactions (coordinator likely crashed)
```

**Key Takeaway:** "2PC ensures atomic distributed transactions but sacrifices availability and performance. Use for critical consistency requirements (bank transfers). For high-throughput systems, prefer saga pattern (compensating transactions) or eliminate cross-DB transactions (microservices anti-pattern). In production: Always replicate coordinator and monitor prepared transaction age."

---

**Next:** [07_Optimistic_Locking.md](07_Optimistic_Locking.md) - Version columns, timestamp-based concurrency

