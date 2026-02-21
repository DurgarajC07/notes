# Query Rewriting - SQL Optimization Patterns

## 1. Concept Explanation

**Query rewriting** is the art of transforming a SQL query into a semantically equivalent form that executes faster. It's about expressing the same logic different ways to help the optimizer (or bypass it entirely).

Think of it like refactoring code: the behavior stays the same, but the performance improves dramatically.

### Types of Query Rewriting

```
1. Predicate Pushdown → Move filters closer to data source
2. Subquery Elimination → Convert correlated subqueries to joins
3. Join Elimination → Remove unnecessary joins
4. Expression Simplification → Simplify complex expressions
5. Constant Folding → Pre-compute constant expressions
6. Disjunction to Union → OR conditions to UNION ALL
7. Common Subexpression Elimination → Reuse computed values
```

### Before vs After Example

**Before (Slow: 45 seconds)**
```sql
SELECT DISTINCT u.username
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
      AND o.total > (
          SELECT AVG(total) FROM orders
      )
);
```

**After (Fast: 0.5 seconds)**
```sql
WITH avg_order AS (
    SELECT AVG(total) as avg_total FROM orders
)
SELECT DISTINCT u.username
FROM users u
INNER JOIN orders o ON u.id = o.user_id
CROSS JOIN avg_order a
WHERE o.total > a.avg_total;
```

**Changes:**
1. Correlated subquery → JOIN
2. Scalar subquery → CTE (computed once)
3. EXISTS → INNER JOIN with DISTINCT

---

## 2. Why It Matters

### Production Impact

**Without Rewriting:**
```
❌ "Optimizer can't figure it out"
❌ "Database is slow, need more hardware"
❌ "Queries timeout during peak hours"
```

**With Rewriting:**
```
✅ "Converted subquery to join → 100× faster"
✅ "Rewrote OR to UNION ALL → uses indexes"
✅ "Eliminated self-join → reduces I/O by 50%"
```

### Real Scenario: E-commerce Product Search

**Original Query (25 seconds, full table scan)**
```sql
-- Find products matching ANY of 5 categories OR text search
SELECT * FROM products
WHERE category_id = 1
   OR category_id = 2
   OR category_id = 3
   OR category_id = 4
   OR category_id = 5
   OR name LIKE '%laptop%'
ORDER BY price
LIMIT 20;
```

**Problem:** OR conditions prevent index usage (index scan on category_id can't combine with LIKE)

**Rewritten Query (0.3 seconds, index scans)**
```sql
-- Split into separate index-friendly queries
(SELECT * FROM products WHERE category_id = 1 LIMIT 20)
UNION ALL
(SELECT * FROM products WHERE category_id = 2 LIMIT 20)
UNION ALL
(SELECT * FROM products WHERE category_id = 3 LIMIT 20)
UNION ALL
(SELECT * FROM products WHERE category_id = 4 LIMIT 20)
UNION ALL
(SELECT * FROM products WHERE category_id = 5 LIMIT 20)
UNION ALL
(SELECT * FROM products WHERE name LIKE '%laptop%' LIMIT 20)
ORDER BY price
LIMIT 20;
```

**Result:** Each branch uses its own index → 83× faster

---

## 3. Internal Working

### How Query Rewriting Works

```
Original Query Text
       ↓
┌──────────────────┐
│ Parser           │ → Abstract Syntax Tree (AST)
└─────────┬────────┘
          ↓
┌──────────────────┐
│ Rewriter         │ → Apply rewrite rules
│ - View expansion │   (automatic by database)
│ - Subquery flatten│
│ - Constant folding│
└─────────┬────────┘
          ↓
┌──────────────────┐
│ Your Rewrite     │ → Manual optimization
│ (This chapter)   │   (you control this)
└─────────┬────────┘
          ↓
┌──────────────────┐
│ Optimizer        │ → Choose execution plan
└──────────────────┘
```

### Optimizer's Automatic Rewrites

Modern databases already do some rewrites automatically:

**1. Predicate Simplification**
```sql
-- You write:
WHERE status = 'active' AND status = 'active'

-- Optimizer rewrites to:
WHERE status = 'active'
```

**2. Constant Folding**
```sql
-- You write:
WHERE price > 100 * 0.9

-- Optimizer rewrites to:
WHERE price > 90
```

**3. Transitivity**
```sql
-- You write:
WHERE a.id = b.id AND b.id = 5

-- Optimizer adds:
WHERE a.id = 5 AND b.id = 5 AND a.id = b.id
-- (Now can use indexes on both a.id and b.id)
```

**4. Subquery Flattening (Simple Cases)**
```sql
-- You write:
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 100);

-- Optimizer may rewrite to:
SELECT DISTINCT u.* FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.total > 100;
```

### But Optimizer Has Limits

**Optimizer CAN'T rewrite:**
- Complex correlated subqueries
- OR conditions spanning multiple columns
- Functions with side effects
- Queries with volatile functions (RANDOM(), NOW())

**That's where manual rewriting comes in.**

---

## 4. Best Practices

### Practice 1: Predicate Pushdown (Move Filters Early)

**❌ Bad: Filter Late**
```sql
-- Joins 1M users + 10M orders, THEN filters
SELECT u.username, o.order_id
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE u.country = 'US'  -- Filters after join
  AND o.created_at >= '2024-01-01';
```

**✅ Good: Filter Early with CTEs**
```sql
-- Filters first, then joins smaller datasets
WITH us_users AS (
    SELECT id, username
    FROM users
    WHERE country = 'US'  -- Filter: 1M → 200K
),
recent_orders AS (
    SELECT user_id, order_id
    FROM orders
    WHERE created_at >= '2024-01-01'  -- Filter: 10M → 500K
)
SELECT u.username, o.order_id
FROM us_users u
INNER JOIN recent_orders o ON u.id = o.user_id;

-- Performance: 10M joins → 100K joins (100× fewer comparisons)
```

### Practice 2: Eliminate Correlated Subqueries

**❌ Bad: Correlated Subquery (O(N²) Complexity)**
```sql
-- Runs subquery for EACH user (1M times)
SELECT u.username, u.total_spent,
       (SELECT COUNT(*)
        FROM orders o
        WHERE o.user_id = u.id) as order_count
FROM users u
WHERE u.total_spent > 1000;

-- Execution plan shows:
-- Seq Scan on users  (cost=0.00..5000000000.00 rows=10000)
--   SubPlan 1
--     ->  Aggregate  (cost=4500.00..4500.01 rows=1)
--           ->  Seq Scan on orders  (cost=0.00..4000.00 rows=200)
--                 Filter: (user_id = u.id)  ← Runs 1M times!
```

**✅ Good: JOIN + GROUP BY (O(N) Complexity)**
```sql
-- Aggregates once, then joins
SELECT u.username, u.total_spent, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.total_spent > 1000
GROUP BY u.id, u.username, u.total_spent;

-- Or use CTE for clarity:
WITH user_order_counts AS (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    GROUP BY user_id
)
SELECT u.username, u.total_spent, COALESCE(uoc.order_count, 0) as order_count
FROM users u
LEFT JOIN user_order_counts uoc ON u.id = uoc.user_id
WHERE u.total_spent > 1000;

-- Performance: 45s → 0.5s (90× faster)
```

### Practice 3: Convert OR to UNION ALL (Enable Index Usage)

**❌ Bad: OR Prevents Index Usage**
```sql
-- Sequential scan (can't use idx_orders_user_id OR idx_orders_status together)
SELECT * FROM orders
WHERE user_id = 12345
   OR status = 'pending';

-- Execution plan:
-- Seq Scan on orders  (cost=0.00..50000.00 rows=100000)
--   Filter: ((user_id = 12345) OR (status = 'pending'))
```

**✅ Good: UNION ALL Uses Both Indexes**
```sql
SELECT * FROM orders WHERE user_id = 12345
UNION ALL
SELECT * FROM orders WHERE status = 'pending' AND user_id <> 12345;
-- ^^^ Avoid duplicates with AND user_id <> 12345

-- Execution plan:
-- Append  (cost=0.43..1500.00 rows=10000)
--   ->  Index Scan using idx_orders_user_id on orders  (cost=0.43..50.00 rows=100)
--         Index Cond: (user_id = 12345)
--   ->  Bitmap Heap Scan on orders  (cost=500.00..1400.00 rows=9900)
--         Recheck Cond: (status = 'pending')
--         Filter: (user_id <> 12345)
--         ->  Bitmap Index Scan on idx_orders_status  (cost=0.00..497.50 rows=10000)
--               Index Cond: (status = 'pending')

-- Performance: 25s → 0.5s (50× faster)
```

**When to Use UNION vs UNION ALL:**
- **UNION** → Removes duplicates (adds overhead)
- **UNION ALL** → Keeps duplicates (faster, use when conditions are mutually exclusive)

### Practice 4: Remove DISTINCT When Not Needed

**❌ Bad: Unnecessary DISTINCT (Adds Sort Overhead)**
```sql
-- Primary key join already guarantees uniqueness
SELECT DISTINCT u.username, o.order_id
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- Execution plan shows:
-- HashAggregate  (cost=8000.00..8500.00 rows=50000)  ← Unnecessary!
--   Group Key: u.username, o.order_id
--   ->  Hash Join  (cost=3000.00..7500.00 rows=50000)
```

**✅ Good: Remove DISTINCT**
```sql
-- order_id is unique, no need for DISTINCT
SELECT u.username, o.order_id
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- Execution plan shows:
-- Hash Join  (cost=3000.00..7500.00 rows=50000)  ← Direct join, no aggregation

-- Performance: Eliminates grouping overhead (10-20% faster)
```

**Keep DISTINCT when:**
- LEFT JOIN with many-to-many relationships
- Denormalized data with actual duplicates

### Practice 5: Use EXISTS Instead of IN for Large Subqueries

**❌ Bad: IN with Large Subquery (Materializes Entire Subquery)**
```sql
-- Loads all 1M order IDs into memory
SELECT * FROM users
WHERE id IN (
    SELECT user_id FROM orders  -- 1M rows
);

-- Execution plan:
-- Hash Semi Join  (cost=50000.00..60000.00 rows=100000)
--   Hash Cond: (users.id = orders.user_id)
--   ->  Seq Scan on users  (cost=0.00..5000.00 rows=200000)
--   ->  Hash  (cost=20000.00..20000.00 rows=1000000)  ← Builds 1M row hash table
--         ->  Seq Scan on orders  (cost=0.00..20000.00 rows=1000000)
```

**✅ Good: EXISTS (Short-Circuits on First Match)**
```sql
-- Stops at first matching order for each user
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
    LIMIT 1  -- Explicit short-circuit
);

-- Execution plan:
-- Nested Loop Semi Join  (cost=0.43..8000.00 rows=100000)
--   ->  Seq Scan on users u  (cost=0.00..5000.00 rows=200000)
--   ->  Index Scan using idx_orders_user_id on orders o  (cost=0.43..0.50 rows=1)
--         Index Cond: (user_id = u.id)

-- Performance: Stops after finding first order (faster for most cases)
```

**When to use each:**
- **IN** → When subquery returns small result set (<1000 rows)
- **EXISTS** → When subquery returns large result set or you just need existence check
- **JOIN** → When you need columns from both tables

---

## 5. Common Mistakes

### Mistake 1: Using SELECT * in Subqueries

**Problem:**
```sql
-- Forces database to process all columns unnecessarily
SELECT u.username
FROM users u
WHERE EXISTS (
    SELECT * FROM orders o  -- ← Transfers all order columns (wasteful)
    WHERE o.user_id = u.id
);
```

**Solution:**
```sql
-- Only select what's needed (or SELECT 1 for existence checks)
SELECT u.username
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o  -- ← Minimal data transfer
    WHERE o.user_id = u.id
);
```

### Mistake 2: NOT IN with NULLs

**Problem:**
```sql
-- Fails silently if subquery contains NULL
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM deleted_users);

-- If deleted_users.user_id has any NULL:
-- NOT IN (1, 2, NULL) evaluates to UNKNOWN (not TRUE, not FALSE)
-- Result: Returns 0 rows (unexpected!)
```

**Solution:**
```sql
-- Use NOT EXISTS (NULL-safe)
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM deleted_users d
    WHERE d.user_id = u.id
);

-- Or filter NULLs explicitly:
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM deleted_users WHERE user_id IS NOT NULL);
```

### Mistake 3: Function Calls Preventing Index Usage

**Problem:**
```sql
-- Index on email exists, but function prevents its use
SELECT * FROM users
WHERE LOWER(email) = 'user@example.com';

-- Execution plan:
-- Seq Scan on users  (cost=0.00..25000.00 rows=1000)
--   Filter: (lower(email) = 'user@example.com')  ← Function disables index
```

**Solution:**
```sql
-- Option 1: Store data in normalized form (lowercase)
UPDATE users SET email = LOWER(email);
CREATE INDEX idx_users_email ON users(email);

SELECT * FROM users WHERE email = 'user@example.com';  -- Index used

-- Option 2: Functional index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';  -- Index used

-- Option 3: Rewrite query to avoid function
-- If you know data is already lowercase:
SELECT * FROM users WHERE email = LOWER('user@example.com');  -- 'user@example.com'
```

### Mistake 4: Rewriting Too Aggressively (Losing Readability)

**Problem:**
```sql
-- Optimized for performance but unreadable
SELECT u.id, u.name,
       (SELECT SUM(o.total) FROM orders o WHERE o.user_id = u.id AND o.status = 'completed') +
       (SELECT SUM(o.total) FROM orders o WHERE o.user_id = u.id AND o.status = 'pending') -
       (SELECT COALESCE(SUM(r.amount), 0) FROM refunds r WHERE r.user_id = u.id)
FROM users u
WHERE u.created_at >= '2024-01-01';

-- 3 separate subqueries → Same table scanned 3 times!
```

**Solution:**
```sql
-- Use CTE for readability AND performance
WITH user_financials AS (
    SELECT
        user_id,
        SUM(CASE WHEN status = 'completed' THEN total ELSE 0 END) as completed_total,
        SUM(CASE WHEN status = 'pending' THEN total ELSE 0 END) as pending_total
    FROM orders
    WHERE user_id IN (SELECT id FROM users WHERE created_at >= '2024-01-01')
    GROUP BY user_id
),
user_refunds AS (
    SELECT user_id, SUM(amount) as refund_total
    FROM refunds
    WHERE user_id IN (SELECT id FROM users WHERE created_at >= '2024-01-01')
    GROUP BY user_id
)
SELECT
    u.id,
    u.name,
    COALESCE(uf.completed_total, 0) + COALESCE(uf.pending_total, 0) - COALESCE(ur.refund_total, 0) as net_value
FROM users u
LEFT JOIN user_financials uf ON u.id = uf.user_id
LEFT JOIN user_refunds ur ON u.id = ur.user_id
WHERE u.created_at >= '2024-01-01';

-- Single-pass aggregation → 3× faster + readable
```

---

## 6. Security Considerations

### 1. Query Rewriting and SQL Injection

**Risk:**
```sql
-- Rewriting concatenated strings can expose injection vulnerabilities
-- BAD (vulnerable):
query = f"SELECT * FROM users WHERE id IN ({user_input})"
# user_input = "1 OR 1=1--"
# Resulting query: SELECT * FROM users WHERE id IN (1 OR 1=1--)
```

**Solution:**
```sql
-- Always use parameterized queries
query = "SELECT * FROM users WHERE id IN (?)"
execute(query, user_input_list)  -- Database escapes input safely
```

### 2. Information Disclosure via Query Timing

**Risk:**
```sql
-- Attacker infers data existence via execution time
-- Fast: User doesn't exist
-- Slow: User exists and has many orders
SELECT * FROM users u
WHERE username = ? AND EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

**Mitigation:**
```sql
-- Use constant-time operations for authentication-related queries
-- Or add artificialdela delay to mask timing differences
```

---

## 7. Performance Optimization

### Optimization Pattern 1: Batch Processing with Window Functions

**Before (N+1 Query Problem)**
```sql
-- Application code (pseudocode):
orders = query("SELECT * FROM orders WHERE status = 'pending'")
for order in orders:
    total_items = query(
        "SELECT COUNT(*) FROM order_items WHERE order_id = ?",
        order.id
    )  # ← Runs 10,000 times for 10K orders!
```

**After (Single Query with Window Function)**
```sql
-- Single query with all needed data
SELECT
    o.order_id,
    o.customer_id,
    o.total,
    COUNT(oi.id) OVER (PARTITION BY o.order_id) as item_count
FROM orders o
LEFT JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status = 'pending';

-- Performance: 10,000 queries → 1 query (1000× fewer round-trips)
```

### Optimization Pattern 2: Lateral Joins for Top-N Per Group

**Before (Correlated Subquery)**
```sql
-- Get top 3 orders per customer
SELECT c.customer_name, o.order_id, o.total
FROM customers c
CROSS JOIN LATERAL (
    SELECT order_id, total
    FROM orders
    WHERE customer_id = c.id
    ORDER BY total DESC
    LIMIT 3
) o;

-- Execution plan:
-- Nested Loop  (cost=0.43..500000.00 rows=300000)
--   ->  Seq Scan on customers c  (cost=0.00..500.00 rows=10000)
--   ->  Limit  (cost=0.43..50.00 rows=3)
--         ->  Index Scan using idx_orders_customer_total on orders
--               Index Cond: (customer_id = c.id)
--               Filter: (total DESC)

-- Fast with proper indexing!
```

**Ensure Index:**
```sql
CREATE INDEX idx_orders_customer_total ON orders(customer_id, total DESC);
```

### Optimization Pattern 3: Materialized CTEs (PostgreSQL 12+)

**Problem:** CTE executed multiple times

```sql
-- CTE scanned 3 times (once per join)
WITH order_stats AS (
    SELECT user_id, AVG(total) as avg_total, COUNT(*) as count
    FROM orders
    GROUP BY user_id
)
SELECT u.username, os1.avg_total, os2.count
FROM users u
JOIN order_stats os1 ON u.id = os1.user_id  -- Scan 1
JOIN order_stats os2 ON u.id = os2.user_id  -- Scan 2 (unnecessary!)
WHERE os1.avg_total > 100;
```

**Solution:**
```sql
-- Force materialization
WITH order_stats AS MATERIALIZED (  -- ← Keywords: MATERIALIZED
    SELECT user_id, AVG(total) as avg_total, COUNT(*) as count
    FROM orders
    GROUP BY user_id
)
SELECT u.username, os.avg_total, os.count
FROM users u
JOIN order_stats os ON u.id = os.user_id
WHERE os.avg_total > 100;

-- CTE computed once, reused from memory
```

---

## 8. Examples

### Example 1: E-commerce Dashboard Optimization

**Scenario:** Dashboard showing sales metrics (running very slow: 2 minutes)

**Original Query:**
```sql
SELECT
    p.product_name,
    (SELECT COUNT(*) FROM order_items oi WHERE oi.product_id = p.id) as times_ordered,
    (SELECT SUM(oi.quantity * oi.price)
     FROM order_items oi
     WHERE oi.product_id = p.id) as revenue,
    (SELECT AVG(r.rating)
     FROM reviews r
     WHERE r.product_id = p.id) as avg_rating,
    (SELECT COUNT(DISTINCT o.customer_id)
     FROM order_items oi
     JOIN orders o ON oi.order_id = o.id
     WHERE oi.product_id = p.id) as unique_customers
FROM products p
WHERE p.category_id = 5
ORDER BY revenue DESC
LIMIT 100;
```

**Problems:**
1. **4 correlated subqueries** → Each runs once per product (100 products × 4 = 400 subqueries)
2. **Same table scanned multiple times** → order_items scanned 3 times per product
3. **Expensive DISTINCT** → unique_customers subquery does full scan + group

**Rewritten Query (Optimized):**
```sql
WITH category_products AS (
    -- Filter products first
    SELECT id, product_name
    FROM products
    WHERE category_id = 5
),
product_metrics AS (
    -- Aggregate order_items once
    SELECT
        oi.product_id,
        COUNT(*) as times_ordered,
        SUM(oi.quantity * oi.price) as revenue,
        COUNT(DISTINCT o.customer_id) as unique_customers
    FROM order_items oi
    INNER JOIN orders o ON oi.order_id = o.id
    WHERE oi.product_id IN (SELECT id FROM category_products)
    GROUP BY oi.product_id
),
product_ratings AS (
    -- Aggregate reviews separately
    SELECT product_id, AVG(rating) as avg_rating
    FROM reviews
    WHERE product_id IN (SELECT id FROM category_products)
    GROUP BY product_id
)
SELECT
    cp.product_name,
    COALESCE(pm.times_ordered, 0) as times_ordered,
    COALESCE(pm.revenue, 0) as revenue,
    COALESCE(pr.avg_rating, 0) as avg_rating,
    COALESCE(pm.unique_customers, 0) as unique_customers
FROM category_products cp
LEFT JOIN product_metrics pm ON cp.id = pm.product_id
LEFT JOIN product_ratings pr ON cp.id = pr.product_id
ORDER BY revenue DESC
LIMIT 100;
```

**Performance:**
- **Before:** 120 seconds (400 subqueries, 1200 table scans)
- **After:** 1.5 seconds (3 CTEs, 1 scan per table)
- **Improvement:** 80× faster

---

### Example 2: Social Media Feed Query

**Scenario:** Show posts from friends and followed users, sorted by recency

**Original Query (Slow: 15 seconds)**
```sql
SELECT p.post_id, p.content, p.created_at, u.username
FROM posts p
JOIN users u ON p.user_id = u.id
WHERE p.user_id IN (
    -- Friends
    SELECT friend_id FROM friendships WHERE user_id = 12345
    UNION
    -- Followed users
    SELECT followed_id FROM follows WHERE follower_id = 12345
)
ORDER BY p.created_at DESC
LIMIT 50;
```

**Problems:**
1. **UNION in subquery** → Forces full evaluation before outer query
2. **Large IN list** → 10,000 friendships/follows → slow hash lookup
3. **Sort after filter** → Sorting all matching posts (1M posts)

**Rewritten Query (Fast: 0.5 seconds)**
```sql
-- Option 1: Split into two queries with UNION ALL (can use indexes separately)
(
    SELECT p.post_id, p.content, p.created_at, u.username, p.user_id
    FROM posts p
    JOIN users u ON p.user_id = u.id
    WHERE p.user_id IN (SELECT friend_id FROM friendships WHERE user_id = 12345)
    ORDER BY p.created_at DESC
    LIMIT 100  -- Fetch more than needed to ensure 50 after dedup
)
UNION ALL
(
    SELECT p.post_id, p.content, p.created_at, u.username, p.user_id
    FROM posts p
    JOIN users u ON p.user_id = u.id
    WHERE p.user_id IN (SELECT followed_id FROM follows WHERE follower_id = 12345)
      AND p.user_id NOT IN (SELECT friend_id FROM friendships WHERE user_id = 12345)  -- Avoid duplicates
    ORDER BY p.created_at DESC
    LIMIT 100
)
ORDER BY created_at DESC
LIMIT 50;

-- Option 2: Use temporary table for followed users (even faster for large lists)
CREATE TEMP TABLE user_feed_sources AS
SELECT friend_id as user_id FROM friendships WHERE user_id = 12345
UNION
SELECT followed_id FROM follows WHERE follower_id = 12345;

CREATE INDEX idx_temp_user_id ON user_feed_sources(user_id);

SELECT p.post_id, p.content, p.created_at, u.username
FROM posts p
JOIN users u ON p.user_id = u.id
WHERE p.user_id IN (SELECT user_id FROM user_feed_sources)
ORDER BY p.created_at DESC
LIMIT 50;
```

**Performance:**
- **Before:** 15 seconds (large IN list, full sort)
- **After (Option 1):** 0.8 seconds (index usage per branch)
- **After (Option 2):** 0.4 seconds (temp table indexed)
- **Improvement:** 30-40× faster

---

### Example 3: Report Generation with Recursive CTE

**Scenario:** Generate org chart with all employees under a manager

**Original Query (Slow: 30 seconds for deep hierarchies)**
```sql
-- Application does recursive fetching (N+1 problem)
-- Pseudocode:
function get_all_subordinates(manager_id):
    direct_reports = query("SELECT id FROM employees WHERE manager_id = ?", manager_id)
    all_subordinates = direct_reports
    for report in direct_reports:
        all_subordinates += get_all_subordinates(report.id)  # Recursive!
    return all_subordinates

subordinates = get_all_subordinates(100)  # Manager ID 100
# Makes 1+ queries per level of hierarchy (10 levels = 1000+ queries)
```

**Rewritten Query (Single Recursive Query: 0.5 seconds)**
```sql
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: Direct reports of manager 100
    SELECT id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id = 100
    
    UNION ALL
    
    -- Recursive case: Reports of reports
    SELECT e.id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
    WHERE eh.level < 10  -- Prevent infinite recursion
)
SELECT id, name, level
FROM employee_hierarchy
ORDER BY level, name;
```

**Performance:**
- **Before:** 30 seconds (1000+ round-trip queries)
- **After:** 0.5 seconds (single recursive query)
- **Improvement:** 60× faster

---

## 9. Real-World Use Cases

### Use Case 1: SaaS Multi-Tenant Query Isolation

**Challenge:** Tenant 12345 has 10M records, other tenants have 100 records each. Shared query slow for large tenant.

**Original:**
```sql
SELECT * FROM events
WHERE tenant_id = ?
  AND event_type = 'page_view'
ORDER BY created_at DESC
LIMIT 100;

-- Small tenant: 0.01s (uses index)
-- Large tenant: 45s (index + sort on 10M rows)
```

**Rewritten (Separate Execution Paths):**
```sql
-- Application logic:
tenant_size = get_tenant_size(tenant_id)

if tenant_size > 1000000:
    # Large tenant: Use partition + limit pushdown
    query = """
        SELECT * FROM events
        WHERE tenant_id = ? AND event_type = 'page_view'
        ORDER BY created_at DESC
        LIMIT 100
        OPTION (MAXDOP 4)  -- Parallel execution
    """
else:
    # Small tenant: Simple index scan
    query = """
        SELECT * FROM events
        WHERE tenant_id = ? AND event_type = 'page_view'
        ORDER BY created_at DESC
        LIMIT 100
    """
```

**Result:** Both code paths optimized for their use case

---

### Use Case 2: Analytics Query with Pre-Aggregation

**Challenge:** Dashboard shows daily active users (DAU) for last 90 days. Query scans billions of events.

**Original (Slow: 2 minutes):**
```sql
SELECT DATE(created_at) as day, COUNT(DISTINCT user_id) as dau
FROM events
WHERE created_at >= NOW() - INTERVAL '90 days'
GROUP BY DATE(created_at)
ORDER BY day;
```

**Rewritten (Materialized View):**
```sql
-- Create pre-aggregated table
CREATE TABLE daily_active_users (
    day DATE PRIMARY KEY,
    dau INTEGER,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Incremental refresh (runs every hour)
INSERT INTO daily_active_users (day, dau)
SELECT DATE(created_at), COUNT(DISTINCT user_id)
FROM events
WHERE created_at >= CURRENT_DATE - INTERVAL '1 day'
GROUP BY DATE(created_at)
ON CONFLICT (day) DO UPDATE
SET dau = EXCLUDED.dau, updated_at = NOW();

-- Dashboard query (Fast: 0.01 seconds)
SELECT day, dau FROM daily_active_users
WHERE day >= CURRENT_DATE - INTERVAL '90 days'
ORDER BY day;
```

**Result:** 12,000× faster (2 minutes → 0.01 seconds)

---

## 10. Interview Questions

### Q1: When would you rewrite an IN subquery to a JOIN?

**Senior Answer:**

```
Rewrite IN to JOIN when:

1. You need columns from both tables
   IN: SELECT * FROM users WHERE id IN (SELECT user_id FROM orders)
   JOIN: SELECT u.*, o.order_id FROM users u JOIN orders o ON u.id = o.user_id

2. Subquery returns large result set
   IN: Materializes entire subquery (memory intensive)
   JOIN: Streams data (memory efficient)

3. You need aggregations
   IN: Doesn't support COUNT, SUM, etc.
   JOIN: Can add COUNT(o.id) as order_count

Keep IN when:
- Subquery returns small, static list: WHERE status IN ('active', 'pending')
- Readability matters and performance is acceptable
- Optimizer automatically converts to semi-join anyway (check EXPLAIN)

Example:
```sql
-- IN (subquery executed once, result cached)
WHERE id IN (SELECT user_id FROM premium_users)  -- 1000 rows

-- JOIN (processes all combinations, needs DISTINCT)
SELECT DISTINCT u.* FROM users u
JOIN premium_users pu ON u.id = pu.user_id

-- For this case, IN is cleaner and equivalent in performance
```
```

### Q2: How do you optimize a query with multiple OR conditions?

**Staff Answer:**

```
Strategy depends on data distribution and indexes:

1. UNION ALL (when branches can use different indexes):
```sql
-- Before: OR prevents index usage
WHERE category_id = 5 OR name LIKE 'laptop%'

-- After: Each branch uses its index
(SELECT * FROM products WHERE category_id = 5)
UNION ALL
(SELECT * FROM products WHERE name LIKE 'laptop%' AND category_id <> 5)
```

2. Bitmap Index Scan (when single index supports all branches):
```sql
-- PostgreSQL combines indexes automatically
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_name ON products(name);

-- Query optimizer uses Bitmap OR:
WHERE category_id IN (1,2,3,4,5)  -- Better than OR chain
```

3. GIN Index (for complex OR conditions on same column):
```sql
-- For text search with multiple keywords
CREATE INDEX idx_products_name_gin ON products USING GIN (to_tsvector('english', name));

WHERE to_tsvector('english', name) @@ to_tsquery('laptop | tablet | phone');
-- Single GIN index scan instead of multiple OR conditions
```

4. Denormalization (for frequently ORed columns):
```sql
-- Add flags for common OR conditions
ALTER TABLE products ADD COLUMN is_electronics BOOLEAN;
UPDATE products SET is_electronics = (category_id IN (1,2,3,4,5));
CREATE INDEX idx_products_electronics ON products(is_electronics);

-- Now simple filter:
WHERE is_electronics = TRUE  -- Instead of: WHERE category_id IN (1,2,3,4,5)
```

Measure before/after with EXPLAIN ANALYZE to confirm improvement.
```

### Q3: Explain when NOT EXISTS is better than NOT IN.

**Mid-Level Answer:**

```
NOT EXISTS is safer and often faster than NOT IN:

1. NULL Handling (Critical Difference):
```sql
-- NOT IN fails silently with NULLs
WHERE id NOT IN (1, 2, NULL)  -- Returns 0 rows! (id <> 1 AND id <> 2 AND id <> NULL → UNKNOWN)

-- NOT EXISTS handles NULLs correctly
WHERE NOT EXISTS (SELECT 1 FROM table WHERE id = outer.id)  -- Works as expected
```

2. Short-Circuit Evaluation:
```sql
-- NOT EXISTS stops at first match
WHERE NOT EXISTS (
    SELECT 1 FROM orders WHERE user_id = users.id LIMIT 1
)  -- Stops immediately when order found

-- NOT IN materializes entire subquery
WHERE id NOT IN (SELECT user_id FROM orders)  -- Processes all orders first
```

3. Performance with Large Subqueries:
```sql
-- NOT IN: Hash anti-join (memory intensive for 10M rows)
WHERE id NOT IN (SELECT user_id FROM orders)  -- Builds hash table of 10M IDs

-- NOT EXISTS: Nested loop anti-join (can use index)
WHERE NOT EXISTS (
    SELECT 1 FROM orders WHERE user_id = users.id
)  -- Index seek per user
```

Use NOT IN only when:
- Subquery guaranteed to have NO NULLs
- Subquery result is small (<1000 rows)
- Subquery is a static list: WHERE status NOT IN ('deleted', 'banned')

Always prefer NOT EXISTS for production code (NULL-safe, generally faster).
```

### Q4: Design a query optimization strategy for a 10-table JOIN running slowly.

**Principal Answer:**

```
Step-by-step optimization framework:

1. Analyze Current Plan
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
[10-table query];

-- Look for:
-- - Cartesian products (missing join conditions)
-- - Nested loops on large tables (should be hash joins)
-- - Sequential scans on large tables (need indexes)
-- - Row count mismatches (estimated vs actual)
```

2. Identify Dominant Tables
```sql
-- Find tables contributing most rows
WITH table_sizes AS (
    SELECT schemaname, tablename, n_live_tup as row_count
    FROM pg_stat_user_tables
    WHERE tablename IN ('orders', 'users', 'products', ...)
)
SELECT * FROM table_sizes ORDER BY row_count DESC;

-- Strategy:
-- - Start joins with smallest filtered tables (reduce intermediate results)
-- - Join large tables last
```

3. Apply Filters Early (Predicate Pushdown)
```sql
-- Before: 10-way join, then filter
FROM orders o
JOIN users u ON ...
JOIN products p ON ...
...
WHERE o.created_at >= '2024-01-01'  -- Applied last

-- After: Filter first, then join
WITH recent_orders AS (
    SELECT * FROM orders WHERE created_at >= '2024-01-01'  -- Filter early
),
... [rest of CTEs with filters]
SELECT * FROM recent_orders ro
JOIN ... [smaller tables now]
```

4. Break Into Subqueries (Divide and Conquer)
```sql
-- Instead of 10-table monster query:
WITH order_aggregates AS (
    SELECT order_id, SUM(price) as total
    FROM order_items
    GROUP BY order_id
),
user_metrics AS (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    WHERE created_at >= '2024-01-01'
    GROUP BY user_id
),
...
SELECT * FROM users u
JOIN user_metrics um ON u.id = um.user_id
JOIN order_aggregates oa ON ...;
-- Each CTE optimized independently
```

5. Add Indexes Based on Plan
```sql
-- From EXPLAIN ANALYZE, identify sequential scans:
-- Seq Scan on orders  Filter: (created_at >= '2024-01-01')
CREATE INDEX idx_orders_created_date ON orders(created_at);

-- Nested loop joining on unindexed columns:
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

6. Consider Materialization for Complex Subqueries
```sql
-- If subquery is expensive and reused:
CREATE TEMP TABLE expensive_subquery AS
SELECT ... [complex calculation];

CREATE INDEX ON expensive_subquery(...);
ANALYZE expensive_subquery;

-- Now join with temp table (cached, indexed)
```

7. Verify and Monitor
```sql
-- Compare plans before/after
-- Check production metrics (pg_stat_statements)
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
WHERE query LIKE '%10_table_query%';
```

Rule of thumb:
- 2-3 joins: Usually fine with proper indexes
- 4-6 joins: Watch execution plan closely
- 7-10 joins: Consider breaking into CTEs or denormalization
- 10+ joins: Red flag - rethink data model

```

---

## 11. Summary

### Key Rewriting Patterns

| Pattern | When to Use | Performance Gain |
|---------|-------------|------------------|
| Correlated subquery → JOIN | Subquery runs per outer row | 10-100× |
| OR → UNION ALL | OR spans different columns/indexes | 10-50× |
| IN (large) → EXISTS | Subquery >1000 rows | 2-10× |
| SELECT * → Specific columns | Large rows, many columns | 2-5× |
| Function on column → Rewrite | Index exists on column | 10-100× |
| Multiple subqueries → CTE | Same data source | 5-50× |
| DISTINCT → Remove if redundant | Primary key joins | 10-20% |
| Predicate push-down | Large joins | 10-100× |

### Decision Framework

```
1. Profile query (EXPLAIN ANALYZE)
2. Identify bottleneck (scan type, join algorithm, row count)
3. Apply appropriate rewrite pattern
4. Verify improvement
5. Document why rewrite was needed
```

### Production Checklist

- [ ] Convert correlated subqueries to JOINs or CTEs
- [ ] Replace OR with UNION ALL when indexes can help
- [ ] Use EXISTS instead of IN for large subqueries
- [ ] Push filters as close to table scans as possible
- [ ] Eliminate unnecessary DISTINCT
- [ ] Avoid functions on indexed columns (use functional indexes)
- [ ] Extract common subexpressions into CTEs
- [ ] Consider temp tables for complex subqueries reused multiple times
- [ ] Verify NULL handling with NOT IN/NOT EXISTS
- [ ] Balance readability vs performance (document complex rewrites)

**Remember:** Query rewriting is an art. The goal isn't the shortest query—it's the fastest query that your team can maintain. Always measure before and after, and document your reasoning.

---

**Next Reading:**
- `03_Cost_Estimation.md` - Understanding how the optimizer calculates costs
- `04_Join_Algorithms.md` - Deep dive into nested loop, hash, and merge joins
- `05_Index_Selection.md` - Choosing the right indexes for your rewrites
