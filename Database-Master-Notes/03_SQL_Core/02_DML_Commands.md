# 📝 DML Commands - Data Manipulation Language

> DML commands manipulate data within tables. Master SELECT, INSERT, UPDATE, and DELETE to query and modify data effectively.

---

## 📖 1. Concept Explanation

### What is DML?

**Data Manipulation Language (DML)** commands work with data stored in tables—retrieving, inserting, updating, and deleting rows.

**Core DML Commands:**
```
DML COMMANDS
│
├── SELECT - Retrieve data
│   ├── WHERE (filtering)
│   ├── ORDER BY (sorting)
│   ├── GROUP BY (aggregation)
│   ├── HAVING (filtered aggregation)
│   └── LIMIT/OFFSET (pagination)
│
├── INSERT - Add new rows
│   ├── INSERT INTO ... VALUES
│   ├── INSERT INTO ... SELECT
│   └── INSERT IGNORE / ON CONFLICT
│
├── UPDATE - Modify existing rows
│   ├── UPDATE ... SET
│   └── UPDATE with JOIN
│
└── DELETE - Remove rows
    ├── DELETE FROM ... WHERE
    └── DELETE with JOIN
```

**Key Characteristics:**
- **Transactional**: Can be rolled back (within transaction)
- **Row-level**: Operates on data, not structure
- **Logged**: All changes recorded in transaction log
- **Can use indexes**: For fast data access

---

## 🧠 2. Why It Matters in Real Systems

### Inefficient vs Efficient Queries

**❌ Bad: N+1 Query Problem**
```sql
-- Application makes 1001 queries for 1000 users!
SELECT * FROM users;  -- Returns 1000 users

-- Then for each user:
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
-- ... 1000 more queries ❌
```

**✅ Good: Single Efficient Query**
```sql
-- One query with JOIN
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.is_active = TRUE
ORDER BY u.created_at DESC;
-- Returns everything in 1 query ✅
```

---

## ⚙️ 3. Internal Working

### SELECT Query Execution Order

```sql
SELECT DISTINCT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.is_active = TRUE
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 5
ORDER BY order_count DESC
LIMIT 10;

-- Internal execution order:
-- 1. FROM users u
-- 2. LEFT JOIN orders o
-- 3. WHERE u.is_active = TRUE
-- 4. GROUP BY u.id, u.name
-- 5. COUNT(o.id)
-- 6. HAVING COUNT(o.id) > 5
-- 7. SELECT u.name, order_count
-- 8. DISTINCT
-- 9. ORDER BY order_count DESC
-- 10. LIMIT 10
```

### INSERT Internals

```sql
INSERT INTO users (email, name) VALUES ('user@test.com', 'John');

-- What happens:
-- 1. Allocate space in table
-- 2. Write data to buffer pool (memory)
-- 3. Write to transaction log (WAL)
-- 4. Update indexes
-- 5. Return auto-increment ID
-- 6. Eventually flush to disk
```

---

## ✅ 4. Best Practices

### SELECT Best Practices

```sql
-- ✅ Specify columns (not SELECT *)
SELECT id, email, name FROM users;  -- Better performance
-- vs
SELECT * FROM users;  -- Reads all columns, more I/O

-- ✅ Use WHERE to limit rows
SELECT * FROM orders WHERE created_at > '2024-01-01';

-- ✅ Use LIMIT for large result sets
SELECT * FROM logs ORDER BY created_at DESC LIMIT 100;

-- ✅ Index columns used in WHERE, ORDER BY, JOIN
CREATE INDEX idx_created ON orders(created_at);
```

### INSERT Best Practices

```sql
-- ✅ Batch inserts (much faster)
INSERT INTO users (email, name) VALUES
    ('user1@test.com', 'User 1'),
    ('user2@test.com', 'User 2'),
    ('user3@test.com', 'User 3');
-- vs 3 separate INSERTs

-- ✅ Use INSERT IGNORE to skip duplicates (MySQL)
INSERT IGNORE INTO users (email, name) 
VALUES ('existing@test.com', 'John');

-- ✅ Use ON CONFLICT for upsert (PostgreSQL)
INSERT INTO users (id, email, name) 
VALUES (1, 'user@test.com', 'John')
ON CONFLICT (id) DO UPDATE 
SET email = EXCLUDED.email, name = EXCLUDED.name;

-- ✅ Use RETURNING to get inserted data (PostgreSQL)
INSERT INTO users (email, name) 
VALUES ('user@test.com', 'John')
RETURNING id, created_at;
```

### UPDATE Best Practices

```sql
-- ✅ Always use WHERE to avoid updating all rows
UPDATE users SET status = 'inactive' WHERE id = 123;
-- NOT: UPDATE users SET status = 'inactive';  ❌ Updates ALL rows!

-- ✅ Update multiple columns in one statement
UPDATE users 
SET status = 'inactive', updated_at = NOW()
WHERE id = 123;

-- ✅ Use transactions for critical updates
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- ✅ Use LIMIT to update in batches
UPDATE users SET is_verified = TRUE 
WHERE email_verified_at IS NOT NULL 
LIMIT 1000;
```

### DELETE Best Practices

```sql
-- ✅ Always use WHERE
DELETE FROM logs WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);

-- ✅ Use soft deletes for important data
UPDATE users SET deleted_at = NOW() WHERE id = 123;
-- vs
DELETE FROM users WHERE id = 123;  -- Permanent!

-- ✅ Delete in batches for large operations
DELETE FROM logs 
WHERE created_at < '2024-01-01' 
LIMIT 10000;
-- Repeat until 0 rows affected

-- ✅ Check foreign key dependencies first
SELECT COUNT(*) FROM orders WHERE user_id = 123;
-- If > 0, decide: CASCADE or RESTRICT?
DELETE FROM users WHERE id = 123;
```

---

## ❌ 5. Common Mistakes

### Mistake 1: SELECT * in Production

```sql
-- ❌ Bad: Fetches all columns (waste bandwidth)
SELECT * FROM users WHERE id = 123;

-- ✅ Good: Only needed columns
SELECT id, email, name FROM users WHERE id = 123;

-- Impact: 
-- SELECT * on 50-column table = 10KB per row
-- SELECT 3 cols = 200 bytes per row
-- 50x more data transferred!
```

### Mistake 2: UPDATE/DELETE Without WHERE

```sql
-- ❌ DISASTER: Updates ALL rows
UPDATE users SET role = 'admin';
-- All users are now admins! ❌

-- ❌ DISASTER: Deletes ALL rows
DELETE FROM orders;
-- All orders gone! ❌

-- ✅ Always use WHERE
UPDATE users SET role = 'admin' WHERE id = 123;
DELETE FROM orders WHERE id = 456;
```

### Mistake 3: No LIMIT on Large Queries

```sql
-- ❌ Bad: Returns millions of rows
SELECT * FROM logs;
-- Application runs out of memory ❌

-- ✅ Good: Pagination
SELECT * FROM logs 
ORDER BY created_at DESC 
LIMIT 100 OFFSET 0;

-- ✅ Even better: Cursor-based pagination
SELECT * FROM logs 
WHERE id > 12345  -- Last seen ID
ORDER BY id 
LIMIT 100;
```

### Mistake 4: Inefficient Batch Updates

```sql
-- ❌ Bad: Loop in application (1000 queries)
for user in users:
    UPDATE users SET last_login = NOW() WHERE id = user.id;

-- ✅ Good: Single UPDATE with case/join
UPDATE users u
JOIN user_logins l ON u.id = l.user_id
SET u.last_login = l.login_time;

-- ✅ Better: Batch by IDs
UPDATE users 
SET last_login = NOW() 
WHERE id IN (1, 2, 3, ..., 1000);
```

### Mistake 5: Not Using Transactions

```sql
-- ❌ Bad: No transaction (money can be lost!)
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- [CRASH HERE] ❌
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- ✅ Good: Atomic transaction
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

---

## 🔐 6. Security Considerations

### SQL Injection Prevention

```sql
-- ❌ VULNERABLE: String concatenation
$sql = "SELECT * FROM users WHERE email = '" . $user_input . "'";
// Input: ' OR '1'='1
// Becomes: SELECT * FROM users WHERE email = '' OR '1'='1'
// Returns all users! ❌

-- ✅ Safe: Parameterized queries
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = ?");
$stmt->execute([$user_input]);
// Input escaped automatically ✅
```

### Limit Data Exposure

```sql
-- ❌ Bad: Returns sensitive data
SELECT * FROM users WHERE username = 'john';
-- Returns password hash, SSN, etc.

-- ✅ Good: Only public data
SELECT id, username, name, avatar_url 
FROM users 
WHERE username = 'john';
```

### Row-Level Security (PostgreSQL)

```sql
-- Enable row-level security
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Policy: Users see only their documents
CREATE POLICY user_documents ON documents
    FOR SELECT
    USING (owner_id = current_user_id());

-- Now SELECT automatically filters:
SELECT * FROM documents;  -- Only returns user's documents
```

---

## 🚀 7. Performance Optimization Techniques

### Indexed Column in WHERE

```sql
-- ❌ Slow: No index, full table scan
SELECT * FROM users WHERE LOWER(email) = 'user@test.com';

-- ✅ Fast: Index on email
CREATE INDEX idx_email ON users(email);
SELECT * FROM users WHERE email = 'user@test.com';

-- ✅ Even better: Case-insensitive index
CREATE INDEX idx_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'user@test.com';
```

### Batch Operations

```sql
-- ❌ Slow: 1000 individual INSERTs
INSERT INTO logs VALUES (...);  -- × 1000 times
-- Time: 10 seconds

-- ✅ Fast: Batch INSERT
INSERT INTO logs VALUES
    (...),
    (...),
    -- ... 1000 rows
;
-- Time: 0.5 seconds (20x faster)
```

### Covering Indexes

```sql
-- Query:
SELECT id, status, created_at FROM orders WHERE user_id = 123;

-- ✅ Covering index (contains all needed columns)
CREATE INDEX idx_user_covering ON orders(user_id, status, created_at);
-- Query uses index-only scan (no table access) ⚡
```

---

## 🧪 8. Examples

### E-Commerce Queries

```sql
-- Get customer with recent orders
SELECT 
    c.id,
    c.name,
    c.email,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent,
    MAX(o.created_at) as last_order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id 
    AND o.status = 'completed'
WHERE c.is_active = TRUE
GROUP BY c.id, c.name, c.email
HAVING COUNT(o.id) > 0
ORDER BY total_spent DESC
LIMIT 100;

-- Insert new order with items
BEGIN;
    INSERT INTO orders (customer_id, status, total)
    VALUES (123, 'pending', 150.00)
    RETURNING id INTO @order_id;
    
    INSERT INTO order_items (order_id, product_id, quantity, price) VALUES
        (@order_id, 1, 2, 50.00),
        (@order_id, 5, 1, 50.00);
COMMIT;

-- Update inventory after order
UPDATE products p
JOIN order_items oi ON p.id = oi.product_id
SET p.stock = p.stock - oi.quantity
WHERE oi.order_id = 12345;

--Soft delete inactive customers
UPDATE customers 
SET deleted_at = NOW() 
WHERE last_login_at < DATE_SUB(NOW(), INTERVAL 2 YEAR)
  AND deleted_at IS NULL;
```

### Analytics Queries

```sql
-- Daily revenue report
SELECT 
    DATE(created_at) as order_date,
    COUNT(*) as order_count,
    SUM(total) as revenue,
    AVG(total) as avg_order_value
FROM orders
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
  AND status = 'completed'
GROUP BY DATE(created_at)
ORDER BY order_date DESC;

-- Top selling products
SELECT 
    p.name,
    p.sku,
    SUM(oi.quantity) as units_sold,
    SUM(oi.quantity * oi.price) as revenue
FROM products p
JOIN order_items oi ON p.id = oi.product_id
JOIN orders o ON oi.order_id = o.id
WHERE o.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
  AND o.status = 'completed'
GROUP BY p.id, p.name, p.sku
ORDER BY units_sold DESC
LIMIT 10;
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Twitter - Feed Generation

```sql
-- Get personalized feed (following + own tweets)
SELECT 
    t.id,
    t.user_id,
    t.text,
    t.created_at,
    u.username,
    u.avatar_url,
    COUNT(l.id) as like_count,
    COUNT(r.id) as retweet_count
FROM tweets t
JOIN users u ON t.user_id = u.id
LEFT JOIN likes l ON t.id = l.tweet_id
LEFT JOIN retweets r ON t.id = r.tweet_id
WHERE t.user_id IN (
    SELECT following_id FROM follows WHERE follower_id = 12345
    UNION
    SELECT 12345  -- Include own tweets
)
  AND t.created_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY t.id, t.user_id, t.text, t.created_at, u.username, u.avatar_url
ORDER BY t.created_at DESC
LIMIT 50;
```

### Case Study 2: Stripe - Idempotent Payments

```sql
-- Upsert payment (prevent duplicates)
INSERT INTO payments (
    idempotency_key,
    customer_id,
    amount,
    currency,
    status
) VALUES (
    '550e8400-e29b-41d4-a716-446655440000',
    123,
    5000,
    'USD',
    'pending'
)
ON CONFLICT (idempotency_key) 
DO UPDATE SET
    updated_at = NOW()
RETURNING id, status, created_at;

-- If duplicate, returns existing payment ✅
```

### Case Study 3: Uber - Location Updates

```sql
-- Batch update driver locations (millions per minute)
INSERT INTO driver_locations (driver_id, lat, lng, updated_at) VALUES
    (1, 37.7749, -122.4194, NOW()),
    (2, 34.0522, -118.2437, NOW()),
    -- ... thousands more
ON DUPLICATE KEY UPDATE
    lat = VALUES(lat),
    lng = VALUES(lng),
    updated_at = VALUES(updated_at);
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between DELETE and TRUNCATE?

**Answer:**

| Aspect | DELETE | TRUNCATE |
|--------|--------|----------|
| **Speed** | Slow (row by row) | Fast (drops/recreates) |
| **WHERE clause** | Yes | No |
| **Rollback** | Yes (in transaction) | No (usually) |
| **Triggers** | Fires triggers | No triggers |
| **Auto-increment** | Preserved | Reset |
| **Logging** | Logged per row | Minimal logging |

```sql
-- DELETE: Selective, transactional
BEGIN;
DELETE FROM logs WHERE created_at < '2024-01-01';
ROLLBACK;  -- Can undo ✅

-- TRUNCATE: All rows, fast
TRUNCATE TABLE logs;  -- No rollback ❌
```

---

### Q2: How do you handle large UPDATE operations?

**Answer:** Update in batches to avoid long locks.

```sql
-- ❌ Bad: Updates all rows at once (long lock)
UPDATE users SET is_verified = TRUE WHERE email_verified_at IS NOT NULL;

-- ✅ Good: Batch updates
REPEAT
    UPDATE users 
    SET is_verified = TRUE 
    WHERE email_verified_at IS NOT NULL 
      AND is_verified = FALSE
    LIMIT 10000;
    
    -- Sleep to reduce load
    SELECT SLEEP(1);
UNTIL ROW_COUNT() = 0 END REPEAT;
```

---

### Q3: What's the N+1 query problem?

**Answer:** Making 1 query to get N items, then N more queries to get related data.

```sql
-- ❌ N+1 Problem:
SELECT * FROM users;  -- 1 query, returns 1000 users

-- Application then does:
for each user:
    SELECT * FROM orders WHERE user_id = ?;  -- 1000 queries
-- Total: 1001 queries ❌

-- ✅ Solution: JOIN
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- Total: 1 query ✅
```

---

## 🧩 11. Design Patterns

### Pattern 1: Upsert (Insert or Update)

```sql
-- PostgreSQL
INSERT INTO product_stats (product_id, views)
VALUES (123, 1)
ON CONFLICT (product_id) 
DO UPDATE SET views = product_stats.views + 1;

-- MySQL
INSERT INTO product_stats (product_id, views)
VALUES (123, 1)
ON DUPLICATE KEY UPDATE views = views + 1;
```

### Pattern 2: Soft Delete

```sql
-- Instead of DELETE
UPDATE users 
SET deleted_at = NOW() 
WHERE id = 123;

-- All queries filter out deleted
SELECT * FROM users 
WHERE deleted_at IS NULL;
```

### Pattern 3: Optimistic Locking

```sql
-- Add version column
ALTER TABLE accounts ADD COLUMN version INT DEFAULT 0;

-- Update with version check
UPDATE accounts 
SET balance = balance + 100, version = version + 1
WHERE id = 123 AND version = 5;

-- If 0 rows updated, someone else modified it (retry)
```

---

## 📚 Summary

### Key Takeaways

1. **SELECT**: Use specific columns, add WHERE, LIMIT results
2. **INSERT**: Batch for performance, use ON CONFLICT for upserts
3. **UPDATE**: Always use WHERE, batch large updates, use transactions
4. **DELETE**: Prefer soft deletes, always use WHERE, batch large deletes
5. **Avoid SELECT ***: Specify needed columns only
6. **Prevent SQL injection**: Use parameterized queries
7. **Use indexes**: WHERE, JOIN, ORDER BY columns
8. **Batch operations**: Much faster than individual queries
9. **Transactions**: For consistency across multiple statements
10. **Monitor queries**: Use slow query log, EXPLAIN plans

---

**Next:** [03_DCL_Commands.md](03_DCL_Commands.md) - Access control  
**Related:** [../05_Query_Optimization/](../05_Query_Optimization/) - Performance tuning

---

*Last Updated: February 19, 2026*
