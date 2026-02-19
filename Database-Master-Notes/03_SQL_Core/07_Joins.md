# 🔗 Joins - Combining Data from Multiple Tables

> Joins combine rows from multiple tables based on related columns. Master INNER, LEFT, RIGHT, FULL, and CROSS joins to query relational data effectively.

---

## 📖 1. Concept Explanation

### What is a JOIN?

A **JOIN** clause combines rows from two or more tables based on a related column between them.

**Types of Joins:**
```
JOINS
│
├── INNER JOIN - Only matching rows from both tables
│
├── LEFT JOIN (LEFT OUTER JOIN) - All from left + matching from right
│
├── RIGHT JOIN (RIGHT OUTER JOIN) - All from right + matching from left
│
├── FULL JOIN (FULL OUTER JOIN) - All rows from both tables
│
├── CROSS JOIN - Cartesian product (all combinations)
│
└── SELF JOIN - Table joined with itself
```

**Visual Representation:**
```
INNER JOIN          LEFT JOIN           RIGHT JOIN          FULL OUTER JOIN
   A ∩ B              A ∪ (A ∩ B)         (A ∩ B) ∪ B        A ∪ B

   ┌─┬─┐             ┌───┬─┐             ┌─┬───┐             ┌───┬───┐
   │ │█│             │███│█│             │█│███│             │███│███│
   └─┴─┘             └───┴─┘             └─┴───┘             └───┴───┘
  Left Right        Left Right          Left Right          Left Right
```

---

## 🧠 2. Why It Matters in Real Systems

### Inefficient Data Retrieval Without Joins

**❌ Bad: Multiple queries (N+1 problem)**
```python
# Get users
users = db.query("SELECT * FROM users")  # 1 query

# Get orders for each user
for user in users:
    orders = db.query(f"SELECT * FROM orders WHERE user_id = {user.id}")  # N queries
    user.orders = orders

# Total: 1 + N queries (if 1000 users = 1001 queries) ❌
# Time: 10 seconds
```

**✅ Good: Single JOIN query**
```sql
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
ORDER BY u.id;

-- Total: 1 query ✅
-- Time: 0.1 seconds (100x faster)
```

---

## ⚙️ 3. Internal Working

### Join Algorithm Types

**1. Nested Loop Join (Small Tables)**
```sql
SELECT * FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- Algorithm:
-- For each row in users (outer loop):
--     For each row in orders (inner loop):
--         If u.id = o.user_id: Output row
-- 
-- Time: O(N × M)
-- Best for: Small tables or when index exists
```

**2. Hash Join (Large Tables)**
```sql
-- Algorithm:
-- 1. Build hash table from smaller table (users)
--    hash_table[user.id] = user
-- 2. Probe larger table (orders)
--    For each order: lookup user in hash_table
-- 
-- Time: O(N + M)
-- Memory: O(N) for hash table
-- Best for: Large tables without indexes
```

**3. Merge Join (Sorted Data)**
```sql
-- Algorithm:
-- 1. Sort both tables by join key (if not already sorted)
-- 2. Merge like mergesort
-- 
-- Time: O(N log N + M log M) for sorting + O(N + M) for merge
-- Best for: Pre-sorted data or indexed columns
```

---

## ✅ 4. Best Practices

### Use Appropriate Join Type

```sql
-- ✅ INNER JOIN: Only customers with orders
SELECT c.name, o.total
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

-- ✅ LEFT JOIN: All customers, even without orders
SELECT c.name, COUNT(o.id) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- ✅ Find customers with NO orders
SELECT c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;
```

### Always Specify Join Conditions

```sql
-- ❌ Implicit join (old style, avoid)
SELECT *
FROM users u, orders o
WHERE u.id = o.user_id;

-- ✅ Explicit JOIN (modern, clear)
SELECT *
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```

### Index Join Columns

```sql
-- ✅ Add indexes on join columns
CREATE INDEX idx_user_id ON orders(user_id);
CREATE INDEX idx_customer_id ON addresses(customer_id);

-- Joins now use index lookups instead of full table scans ✅
```

### Qualify Column Names

```sql
-- ❌ Ambiguous
SELECT id, name
FROM users
JOIN orders ON users.id = orders.user_id;
-- ERROR: Column 'id' in field list is ambiguous

-- ✅ Clear with table aliases
SELECT u.id, u.name, o.id as order_id
FROM users u
JOIN orders o ON u.id = o.user_id;
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Cartesian Product (Missing JOIN Condition)

```sql
-- ❌ Forgot ON clause: Returns ALL combinations!
SELECT *
FROM users u
CROSS JOIN orders o;
-- If 1000 users × 10,000 orders = 10,000,000 rows! ❌

-- ✅ Correct with JOIN condition
SELECT *
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```

### Mistake 2: Wrong Join Type

```sql
-- ❌ INNER JOIN: Excludes users without orders
SELECT u.name, COUNT(o.id) as order_count
FROM users u
INNER JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
-- Users with 0 orders not shown ❌

-- ✅ LEFT JOIN: Includes all users
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
-- Users with 0 orders shown as 0 ✅
```

### Mistake 3: Not Handling NULLs in LEFT JOIN

```sql
-- ❌ Filter in WHERE converts LEFT JOIN to INNER JOIN
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.status = 'completed';
-- Excludes users without orders ❌

-- ✅ Filter in ON clause to preserve LEFT JOIN
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.status = 'completed';
-- Includes all users ✅
```

### Mistake 4: Duplicate Rows from Multiple JOINs

```sql
-- ❌ Cartesian product between addresses and orders
SELECT u.name, COUNT(*) as count
FROM users u
LEFT JOIN addresses a ON u.id = a.user_id
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
-- If user has 2 addresses and 3 orders: count = 6! ❌

-- ✅ Join to aggregated data
SELECT u.name, addr_count, order_count
FROM users u
LEFT JOIN (
    SELECT user_id, COUNT(*) as addr_count
    FROM addresses
    GROUP BY user_id
) a ON u.id = a.user_id
LEFT JOIN (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    GROUP BY user_id
) o ON u.id = o.user_id;
```

---

## 🔐 6. Security Considerations

### SQL Injection in Dynamic Joins

```sql
-- ❌ Vulnerable
$table = $_GET['table'];
$sql = "SELECT * FROM users u JOIN $table t ON u.id = t.user_id";
// Input: "orders; DROP TABLE users; --"
// Disaster! ❌

-- ✅ Whitelist allowed tables
$allowed = ['orders', 'addresses', 'payments'];
if (!in_array($table, $allowed)) {
    throw new Exception('Invalid table');
}
$sql = "SELECT * FROM users u JOIN $table t ON u.id = t.user_id";
```

---

## 🚀 7. Performance Optimization

### Join Order Matters

```sql
-- ❌ Slower: Large table first
SELECT *
FROM huge_orders_table o
JOIN small_users_table u ON o.user_id = u.id
WHERE u.country = 'US';

-- ✅ Faster: Filter small table first
SELECT *
FROM small_users_table u
JOIN huge_orders_table o ON u.id = o.user_id
WHERE u.country = 'US';

-- Or let optimizer handle it with proper indexes
CREATE INDEX idx_users_country ON users(country);
CREATE INDEX idx_orders_user ON orders(user_id);
```

### Use Covering Indexes

```sql
-- Query
SELECT u.name, o.total, o. FROM users u
JOIN orders o ON u.id = o.user_id;

-- ✅ Covering indexes (contain all needed columns)
CREATE INDEX idx_users_covering ON users(id, name);
CREATE INDEX idx_orders_covering ON orders(user_id, total, status);
-- Query can be satisfied from indexes alone (no table access) ✅
```

### Reduce Join Complexity

```sql
-- ❌ Slow: Chained joins on large tables
SELECT *
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
JOIN categories c ON p.category_id = c.id
JOIN manufacturers m ON p.manufacturer_id = m.id;
-- 6-way join can be very slow ❌

-- ✅ Better: Pre-aggregate or denormalize hot paths
SELECT u.*, order_summary.*
FROM users u
JOIN order_summary_cache order_summary ON u.id = order_summary.user_id;
-- Use materialized view or cache table ✅
```

---

## 🧪 8. Examples

### INNER JOIN - Only Matching Rows

```sql
-- Customers who have placed orders
SELECT 
    c.name,
    c.email,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.email
ORDER BY total_spent DESC;
```

### LEFT JOIN - All from Left Table

```sql
-- All customers, with order count (including 0)
SELECT 
    c.name,
    c.email,
    COUNT(o.id) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.email;

-- Find customers with NO orders
SELECT c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;
```

### RIGHT JOIN - All from Right Table

```sql
-- All orders, even if customer deleted
SELECT 
    o.id,
    o.total,
    c.name as customer_name
FROM customers c
RIGHT JOIN orders o ON c.id = o.customer_id;

-- Note: Can rewrite as LEFT JOIN by swapping tables
SELECT 
    o.id,
    o.total,
    c.name as customer_name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id;
```

### FULL OUTER JOIN - All from Both Tables

```sql
-- All users and all orders (show orphans from both sides)
SELECT 
    u.name as user_name,
    o.id as order_id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- MySQL doesn't support FULL OUTER JOIN, emulate with UNION:
SELECT u.name, o.id
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
UNION
SELECT u.name, o.id
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;
```

### CROSS JOIN - Cartesian Product

```sql
-- All combinations of products and sizes
SELECT 
    p.name as product_name,
    s.size_name
FROM products p
CROSS JOIN sizes s
WHERE p.category = 'clothing';

-- Generate all date × category combinations for reporting
SELECT d.date, c.category_name
FROM date_series d
CROSS JOIN categories c;
```

### SELF JOIN - Table Joined to Itself

```sql
-- Find employees and their managers
SELECT 
    e.name as employee_name,
    m.name as manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Find users in same city
SELECT 
    u1.name as user1,
    u2.name as user2,
    u1.city
FROM users u1
JOIN users u2 ON u1.city = u2.city AND u1.id < u2.id;
```

### Multiple Joins

```sql
-- Order details with customer, product, and category info
SELECT 
    o.id as order_id,
    c.name as customer_name,
    p.name as product_name,
    cat.name as category_name,
    oi.quantity,
    oi.price
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
JOIN categories cat ON p.category_id = cat.id
WHERE o.status = 'completed'
  AND o.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Amazon - Product Recommendations

```sql
-- Users who bought product A also bought product B
SELECT 
    p2.name as also_bought,
    COUNT(*) as frequency
FROM order_items oi1
JOIN order_items oi2 ON oi1.order_id = oi2.order_id 
    AND oi1.product_id != oi2.product_id
JOIN products p2 ON oi2.product_id = p2.id
WHERE oi1.product_id = 12345  -- Product A
GROUP BY p2.id, p2.name
ORDER BY frequency DESC
LIMIT 10;
```

### Case Study 2: Stripe - Reconciliation Report

```sql
-- Match payments with payouts
SELECT 
    p.id as payment_id,
    p.amount as payment_amount,
    p.created_at as payment_date,
    po.id as payout_id,
    po.amount as payout_amount,
    po.arrival_date,
    CASE 
        WHEN po.id IS NULL THEN 'pending'
        ELSE 'paid'
    END as status
FROM payments p
LEFT JOIN payout_items pi ON p.id = pi.payment_id
LEFT JOIN payouts po ON pi.payout_id = po.id
WHERE p.created_at >= '2024-01-01'
ORDER BY p.created_at DESC;
```

### Case Study 3: Uber - Driver-Rider Matching

```sql
-- Available drivers near rider location
SELECT 
    d.id as driver_id,
    d.name,
    d.vehicle_type,
    d.rating,
    ST_Distance_Sphere(
        POINT(d.longitude, d.latitude),
        POINT(-122.4194, 37.7749)  -- Rider location
    ) / 1609.34 as distance_miles
FROM drivers d
LEFT JOIN rides r ON d.id = r.driver_id 
    AND r.status IN ('in_progress', 'accepted')
WHERE d.is_online = TRUE
  AND d.status = 'available'
  AND r.id IS NULL  -- Not currently on a ride
HAVING distance_miles < 5
ORDER BY distance_miles ASC
LIMIT 10;
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between INNER JOIN and LEFT JOIN?

**Answer:**

| Aspect | INNER JOIN | LEFT JOIN |
|--------|------------|-----------|
| **Returns** | Only matching rows | All left rows + matching right |
| **Use case** | Related data exists | Include rows without relation |
| **NULL** | No NULL from joins | NULL for non-matching right rows |

```sql
-- INNER JOIN: Only customers with orders
SELECT c.name, o.total
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;
-- Result: 100 rows (only customers with orders)

-- LEFT JOIN: All customers
SELECT c.name, o.total
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;
-- Result: 150 rows (all customers, NULL for those without orders)
```

---

### Q2: How do you find rows that DON'T have a match in another table?

**Answer:** Use **LEFT JOIN** with **IS NULL** check.

```sql
-- Find customers with NO orders
SELECT c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;

-- Or use NOT EXISTS (sometimes faster)
SELECT c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.id
);
```

---

### Q3: What's a CROSS JOIN? When would you use it?

**Answer:** A **CROSS JOIN** returns the **Cartesian product** (all possible combinations) of two tables.

**Use cases:**
- Generate combinations (e.g., all product × size)
- Create date series × categories for reporting
- Generate test data

```sql
-- Create all product-size combinations
SELECT p.name, s.size_name
FROM products p
CROSS JOIN sizes s;

-- If 100 products × 4 sizes = 400 rows
```

---

### Q4: What's a SELF JOIN?

**Answer:** A table joined **with itself** using aliases.

**Use cases:**
- Employee-Manager relationships
- Finding duplicates
- Comparing rows within same table

```sql
-- Employees with their managers
SELECT 
    e.name as employee,
    m.name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

### Q5: How do you optimize slow JOIN queries?

**Answer:**

**1. Add indexes on join columns:**
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_addresses_user_id ON addresses(user_id);
```

**2. Use covering indexes:**
```sql
CREATE INDEX idx_covering ON orders(user_id, total, status, created_at);
-- Query can be satisfied from index alone
```

**3. Filter early:**
```sql
-- ✅ Filter before join when possible
SELECT u.name, o.total
FROM users u
JOIN (
    SELECT user_id, total 
    FROM orders 
    WHERE status = 'completed'  -- Filter here
) o ON u.id = o.user_id;
```

**4. Analyze query plan:**
```sql
EXPLAIN SELECT ...;
-- Look for "Using index", "Using where", avoid "Using filesort"
```

**5. Denormalize hot paths:**
```sql
-- Pre-compute frequent joins
CREATE TABLE user_order_summary (
    user_id INT,
    order_count INT,
    total_spent DECIMAL(10,2),
    INDEX(user_id)
);
-- Refresh periodically
```

---

## 🧩 11. Design Patterns

### Pattern 1: Join to Aggregated Data

```sql
-- Avoid cartesian explosion
SELECT 
    u.name,
    order_stats.order_count,
    payment_stats.payment_count
FROM users u
LEFT JOIN (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    GROUP BY user_id
) order_stats ON u.id = order_stats.user_id
LEFT JOIN (
    SELECT user_id, COUNT(*) as payment_count
    FROM payments
    GROUP BY user_id
) payment_stats ON u.id = payment_stats.user_id;
```

### Pattern 2: Conditional JOIN

```sql
-- Different join logic based on condition
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id 
    AND o.status = 'completed'  -- Join condition, not WHERE
    AND o.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);
```

### Pattern 3: Link Table (Many-to-Many)

```sql
-- Students and courses (many-to-many)
SELECT 
    s.name as student_name,
    c.name as course_name,
    e.grade
FROM students s
JOIN enrollments e ON s.id = e.student_id
JOIN courses c ON e.course_id = c.id
WHERE e.semester = 'Fall 2024';
```

---

## 📚 Summary

### Key Takeaways

1. **INNER JOIN**: Only matching rows from both tables
2. **LEFT JOIN**: All from left + matching from right (NULL for non-matching)
3. **RIGHT JOIN**: All from right + matching from left
4. **FULL OUTER JOIN**: All rows from both tables
5. **CROSS JOIN**: Cartesian product (all combinations)
6. **SELF JOIN**: Table joined with itself
7. **Always index join columns**: Massive performance improvement
8. **Left join + IS NULL**: Find non-matching rows
9. **Explicit JOIN syntax**: More readable than implicit joins
10. **Join order**: Let optimizer decide, but filter small tables first

**Join Selection Guide:**
- Need **all** left rows → **LEFT JOIN**
- Need **only matching** → **INNER JOIN**
- Need **all combinations** → **CROSS JOIN**
- Find **non-matching** → **LEFT JOIN ... WHERE right.id IS NULL**

---

**Next:** [README.md](README.md) - Section overview  
**Related:** [../05_Query_Optimization/](../05_Query_Optimization/) - Join optimization

---

*Last Updated: February 19, 2026*
