# 🔄 CTEs - Common Table Expressions

> CTEs (WITH clause) create temporary named result sets. Master CTEs to write readable, maintainable SQL and solve recursive problems efficiently.

---

## 📖 1. Concept Explanation

### What is a CTE?

A **Common Table Expression (CTE)** is a temporary named result set defined using the `WITH` clause. Think of it as a "named subquery" that exists for the duration of a query.

**Basic Syntax:**
```sql
WITH cte_name AS (
    SELECT column1, column2
    FROM table
    WHERE condition
)
SELECT *
FROM cte_name;
```

**Types of CTEs:**
```
CTEs
│
├── Simple CTE (non-recursive)
│   ├── Single CTE
│   ├── Multiple CTEs
│   └── Nested CTEs
│
└── Recursive CTE
    ├── Anchor member (base case)
    ├── Recursive member (iterative step)
    └── Termination condition
```

**CTE vs Subquery:**
```sql
-- ❌ Subquery: Hard to read
SELECT u.name, subq.order_count
FROM users u
JOIN (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    GROUP BY user_id
) AS subq ON u.id = subq.user_id;

-- ✅ CTE: Much more readable
WITH order_counts AS (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    GROUP BY user_id
)
SELECT u.name, oc.order_count
FROM users u
JOIN order_counts oc ON u.id = oc.user_id;
```

---

## 🧠 2. Why It Matters in Real Systems

### Complex Queries Become Maintainable

**❌ Bad: Nested subqueries (hard to understand)**
```sql
SELECT 
    name,
    (SELECT COUNT(*) 
     FROM orders 
     WHERE user_id IN (
         SELECT id 
         FROM users 
         WHERE created_at > (
             SELECT DATE_SUB(NOW(), INTERVAL 30 DAY)
         )
     )
    ) as recent_user_orders
FROM users;
-- What does this do? Hard to tell! ❌
```

**✅ Good: CTEs (self-documenting)**
```sql
WITH recent_users AS (
    SELECT id
    FROM users
    WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
),
recent_orders AS (
    SELECT COUNT(*) as order_count
    FROM orders
    WHERE user_id IN (SELECT id FROM recent_users)
)
SELECT name, (SELECT order_count FROM recent_orders) as recent_user_orders
FROM users;
-- Clear step-by-step logic ✅
```

---

## ⚙️ 3. Internal Working

### CTE Execution

```sql
WITH high_value_orders AS (
    SELECT user_id, total
    FROM orders
    WHERE total > 1000
)
SELECT u.name, h.total
FROM users u
JOIN high_value_orders h ON u.id = h.user_id;

-- Execution:
-- 1. Execute CTE query: SELECT ... FROM orders WHERE total > 1000
-- 2. Store result temporarily (optimizer may materialize or inline)
-- 3. Execute main query using CTE
-- 4. Discard CTE after query completes
```

### Recursive CTE Execution

```sql
WITH RECURSIVE numbers AS (
    SELECT 1 as n          -- Anchor: Starting point
    UNION ALL
    SELECT n + 1           -- Recursive: Next value
    FROM numbers
    WHERE n < 10           -- Termination: Stop condition
)
SELECT * FROM numbers;

-- Execution:
-- Iteration 0: n = 1 (anchor)
-- Iteration 1: n = 2 (1 + 1)
-- Iteration 2: n = 3 (2 + 1)
-- ...
-- Iteration 9: n = 10 (9 + 1)
-- Iteration 10: WHERE n < 10 fails, stop
```

---

## ✅ 4. Best Practices

### Use CTEs for Readability

```sql
-- ✅ Breaking down complex logic with multiple CTEs
WITH 
active_users AS (
    SELECT id, name
    FROM users
    WHERE is_active = TRUE
),
recent_orders AS (
    SELECT user_id, SUM(total) as total_spent
    FROM orders
    WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
    GROUP BY user_id
),
high_value_customers AS (
    SELECT au.id, au.name, ro.total_spent
    FROM active_users au
    JOIN recent_orders ro ON au.id = ro.user_id
    WHERE ro.total_spent > 1000
)
SELECT *
FROM high_value_customers
ORDER BY total_spent DESC;

-- Each CTE has a clear purpose ✅
```

### Reuse CTEs Multiple Times

```sql
-- ✅ CTE defined once, used multiple times
WITH monthly_revenue AS (
    SELECT 
        DATE_FORMAT(created_at, '%Y-%m') as month,
        SUM(total) as revenue
    FROM orders
    GROUP BY month
)
SELECT 
    current.month,
    current.revenue as current_revenue,
    previous.revenue as previous_revenue,
    (current.revenue - previous.revenue) / previous.revenue * 100 as growth_pct
FROM monthly_revenue current
LEFT JOIN monthly_revenue previous 
    ON previous.month = DATE_FORMAT(DATE_SUB(STR_TO_DATE(CONCAT(current.month, '-01'), '%Y-%m-%d'), INTERVAL 1 MONTH), '%Y-%m');
```

### Limit Recursive CTE Depth

```sql
-- ✅ Always include max depth protection
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 as depth
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
    WHERE ct.depth < 10  -- ✅ Prevent infinite recursion
)
SELECT * FROM category_tree;
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Forgetting RECURSIVE Keyword

```sql
-- ❌ Error: Missing RECURSIVE
WITH numbers AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM numbers WHERE n < 10
)
SELECT * FROM numbers;
-- ERROR: Recursive reference not allowed

-- ✅ Correct
WITH RECURSIVE numbers AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM numbers WHERE n < 10
)
SELECT * FROM numbers;
```

### Mistake 2: No Termination Condition

```sql
-- ❌ Infinite recursion: No WHERE to stop
WITH RECURSIVE infinite AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM infinite  -- Never stops! ❌
)
SELECT * FROM infinite;
-- Runs forever or hits max recursion depth

-- ✅ Always add termination
WITH RECURSIVE finite AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM finite WHERE n < 100  -- Stops at 100 ✅
)
SELECT * FROM finite;
```

### Mistake 3: Overusing CTEs for Performance

```sql
-- ❌ CTE materialized multiple times (inefficient)
WITH expensive_calc AS (
    SELECT user_id, SUM(COMPLEX_CALCULATION(value)) as result
    FROM huge_table
    GROUP BY user_id
)
SELECT * FROM expensive_calc WHERE result > 100
UNION ALL
SELECT * FROM expensive_calc WHERE result <= 100;
-- expensive_calc executed twice! ❌

-- ✅ Better: Temporary table
CREATE TEMPORARY TABLE expensive_calc AS
SELECT user_id, SUM(COMPLEX_CALCULATION(value)) as result
FROM huge_table
GROUP BY user_id;

SELECT * FROM expensive_calc WHERE result > 100
UNION ALL
SELECT * FROM expensive_calc WHERE result <= 100;
-- Computed once, reused ✅
```

### Mistake 4: CTE Column Count Mismatch

```sql
-- ❌ Error: Column count doesn't match
WITH RECURSIVE numbers(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1, 'extra'  -- 2 columns, but CTE defined with 1 ❌
    FROM numbers
    WHERE n < 10
)
SELECT * FROM numbers;

-- ✅ Correct: Matching columns
WITH RECURSIVE numbers(n, label) AS (
    SELECT 1, 'one'
    UNION ALL
    SELECT n + 1, 'num'
    FROM numbers
    WHERE n < 10
)
SELECT * FROM numbers;
```

---

## 🔐 6. Security Considerations

### SQL Injection

```sql
-- ❌ Vulnerable: String concatenation
$category = $user_input;
$sql = "WITH cat AS (SELECT * FROM categories WHERE name = '$category')
        SELECT * FROM products WHERE category_id IN (SELECT id FROM cat)";

-- ✅ Safe: Parameterized
$stmt = $pdo->prepare("
    WITH cat AS (SELECT * FROM categories WHERE name = ?)
    SELECT * FROM products WHERE category_id IN (SELECT id FROM cat)
");
$stmt->execute([$category]);
```

---

## 🚀 7. Performance Optimization

### CTE Materialization

```sql
-- PostgreSQL: CTEs are materialized by default (pre-v12)
WITH large_cte AS (
    SELECT * FROM huge_table WHERE condition
)
SELECT * FROM large_cte LIMIT 10;
-- CTE fully executed, then LIMIT applied ❌

-- PostgreSQL 12+: Add NOT MATERIALIZED for inline
WITH large_cte AS NOT MATERIALIZED (
    SELECT * FROM huge_table WHERE condition
)
SELECT * FROM large_cte LIMIT 10;
-- LIMIT pushed down into CTE ✅

-- MySQL: CTEs are inlined by optimizer
```

### Index CTE Results

```sql
-- If CTE result is large and used multiple times
CREATE TEMPORARY TABLE temp_cte (
    INDEX(user_id),
    INDEX(created_at)
) AS
SELECT user_id, created_at, total
FROM orders
WHERE status = 'completed';

-- Use indexed temp table instead of CTE
SELECT * FROM temp_cte WHERE user_id = 123;  -- Uses index ✅
```

---

## 🧪 8. Examples

### Simple CTE

```sql
-- Get top 10 users by spending
WITH user_spending AS (
    SELECT 
        user_id,
        SUM(total) as total_spent,
        COUNT(*) as order_count
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
)
SELECT 
    u.name,
    u.email,
    us.total_spent,
    us.order_count
FROM users u
JOIN user_spending us ON u.id = us.user_id
ORDER BY us.total_spent DESC
LIMIT 10;
```

### Multiple CTEs

```sql
WITH 
active_users AS (
    SELECT id
    FROM users
    WHERE last_login_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
),
recent_orders AS (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
    GROUP BY user_id
),
engaged_users AS (
    SELECT au.id
    FROM active_users au
    JOIN recent_orders ro ON au.id = ro.user_id
    WHERE ro.order_count > 2
)
SELECT u.name, u.email
FROM users u
JOIN engaged_users eu ON u.id = eu.id;
```

### Recursive CTE - Employee Hierarchy

```sql
-- Get all reports under manager (both direct and indirect)
WITH RECURSIVE employee_hierarchy AS (
    -- Anchor: Start with CEO
    SELECT id, name, manager_id, 0 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: Get reports
    SELECT e.id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.id
    WHERE eh.level < 10  -- Max 10 levels
)
SELECT 
    REPEAT('  ', level) || name as hierarchy,
    level
FROM employee_hierarchy
ORDER BY level, name;

-- Output:
-- CEO (level 0)
--   VP Engineering (level 1)
--     Senior Engineer (level 2)
--       Engineer (level 3)
--   VP Sales (level 1)
--     Sales Rep (level 2)
```

### Recursive CTE - Date Series

```sql
-- Generate dates for last 30 days
WITH RECURSIVE date_series AS (
    SELECT CURRENT_DATE as date
    UNION ALL
    SELECT date - INTERVAL 1 DAY
    FROM date_series
    WHERE date > CURRENT_DATE - INTERVAL 30 DAY
)
SELECT date, DAYNAME(date) as day_name
FROM date_series
ORDER BY date;
```

### Recursive CTE - Category Path

```sql
-- Build full category path (e.g., "Electronics > Phones > iPhone")
WITH RECURSIVE category_path AS (
    SELECT 
        id,
        name,
        parent_id,
        CAST(name AS CHAR(500)) as path
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT 
        c.id,
        c.name,
        c.parent_id,
        CONCAT(cp.path, ' > ', c.name)
    FROM categories c
    JOIN category_path cp ON c.parent_id = cp.id
)
SELECT id, path
FROM category_path
ORDER BY path;
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Reddit - Comment Thread Traversal

```sql
-- Get entire comment thread with nested replies
WITH RECURSIVE comment_thread AS (
    -- Anchor: Top-level comment
    SELECT 
        id,
        parent_id,
        user_id,
        content,
        0 as depth,
        CAST(id AS CHAR(1000)) as path
    FROM comments
    WHERE post_id = 12345 AND parent_id IS NULL
    
    UNION ALL
    
    -- Recursive: Nested replies
    SELECT 
        c.id,
        c.parent_id,
        c.user_id,
        c.content,
        ct.depth + 1,
        CONCAT(ct.path, '-', c.id)
    FROM comments c
    JOIN comment_thread ct ON c.parent_id = ct.id
    WHERE ct.depth < 50  -- Max 50 levels of nesting
)
SELECT 
    REPEAT('  ', depth) || content as indented_comment,
    depth,
    user_id
FROM comment_thread
ORDER BY path;
```

### Case Study 2: LinkedIn - Connection Degrees

```sql
-- Find all 2nd-degree connections
WITH RECURSIVE connections AS (
    -- Anchor: 1st degree (direct connections)
    SELECT 
        friend_id as user_id,
        1 as degree,
        ARRAY[12345, friend_id] as path
    FROM friendships
    WHERE user_id = 12345
    
    UNION
    
    -- Recursive: 2nd degree
    SELECT 
        f.friend_id,
        c.degree + 1,
        c.path || f.friend_id
    FROM connections c
    JOIN friendships f ON c.user_id = f.user_id
    WHERE c.degree < 2
      AND f.friend_id != ALL(c.path)  -- Avoid cycles
)
SELECT 
    u.name,
    c.degree,
    CASE 
        WHEN c.degree = 1 THEN 'Direct connection'
        WHEN c.degree = 2 THEN '2nd degree connection'
    END as connection_type
FROM connections c
JOIN users u ON c.user_id = u.id
ORDER BY c.degree, u.name;
```

### Case Study 3: GitHub - PR Review Chain

```sql
-- Track approval chain for PR merges
WITH review_chain AS (
    SELECT 
        pr_id,
        reviewer_id,
        approved_at,
        LAG(approved_at) OVER (PARTITION BY pr_id ORDER BY approved_at) as previous_approval
    FROM pull_request_reviews
    WHERE status = 'approved'
),
approval_metrics AS (
    SELECT 
        pr_id,
        COUNT(*) as approval_count,
        MIN(approved_at) as first_approval,
        MAX(approved_at) as final_approval,
        TIMESTAMPDIFF(MINUTE, MIN(approved_at), MAX(approved_at)) as approval_duration_minutes
    FROM review_chain
    GROUP BY pr_id
)
SELECT 
    pr.title,
    am.approval_count,
    am.approval_duration_minutes,
    pr.merged_at
FROM pull_requests pr
JOIN approval_metrics am ON pr.id = am.pr_id
WHERE pr.merged_at IS NOT NULL;
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What is a CTE and why use it?

**Answer:** A **Common Table Expression (CTE)** is a temporary named result set defined with the `WITH` clause.

**Benefits:**
- **Readability**: Break complex queries into logical steps
- **Reusability**: Reference CTE multiple times
- **Recursion**: Solve hierarchical problems
- **Maintainability**: Easier to debug and modify

```sql
WITH high_value_users AS (
    SELECT user_id, SUM(total) as total_spent
    FROM orders
    GROUP BY user_id
    HAVING SUM(total) > 10000
)
SELECT u.name, hvu.total_spent
FROM users u
JOIN high_value_users hvu ON u.id = hvu.user_id;
```

---

### Q2: What's the difference between a CTE and a subquery?

**Answer:**

| Aspect | CTE | Subquery |
|--------|-----|----------|
| **Syntax** | WITH cte AS (...) | (...) AS subq |
| **Readability** | More readable | Can be nested/complex |
| **Reusability** | Can reference multiple times | Must repeat |
| **Recursion** | Supports recursive | No recursion |
| **Performance** | Similar (optimizer dependent) | Similar |

```sql
-- CTE: Named, reusable
WITH totals AS (SELECT user_id, SUM(total) FROM orders GROUP BY user_id)
SELECT * FROM totals WHERE SUM > 1000
UNION ALL
SELECT * FROM totals WHERE SUM <= 1000;

-- Subquery: Must repeat
SELECT * FROM (SELECT user_id, SUM(total) FROM orders GROUP BY user_id) t WHERE SUM > 1000
UNION ALL
SELECT * FROM (SELECT user_id, SUM(total) FROM orders GROUP BY user_id) t WHERE SUM <= 1000;
```

---

### Q3: How does a recursive CTE work?

**Answer:** A recursive CTE has two parts:
1. **Anchor member**: Starting point (non-recursive)
2. **Recursive member**: References the CTE itself

**Structure:**
```sql
WITH RECURSIVE cte AS (
    -- Anchor member (runs first)
    SELECT ...
    
    UNION ALL
    
    -- Recursive member (runs repeatedly)
    SELECT ... FROM cte WHERE condition
)
SELECT * FROM cte;
```

**Example: Generate 1 to 10**
```sql
WITH RECURSIVE numbers AS (
    SELECT 1 as n              -- Anchor: Start at 1
    UNION ALL
    SELECT n + 1 FROM numbers  -- Recursive: Add 1
    WHERE n < 10               -- Termination: Stop at 10
)
SELECT * FROM numbers;
-- Result: 1, 2, 3, ..., 10
```

---

### Q4: What are common use cases for recursive CTEs?

**Answer:**

1. **Hierarchies**: Organization charts, category trees
2. **Graphs**: Social networks, recommendations
3. **Date series**: Generate date ranges
4. **Bill of materials**: Product components
5. **File systems**: Directory traversal

```sql
-- Example: Get all subcategories
WITH RECURSIVE subcategories AS (
    SELECT id, name, parent_id
    FROM categories
    WHERE id = 5  -- Starting category
    
    UNION ALL
    
    SELECT c.id, c.name, c.parent_id
    FROM categories c
    JOIN subcategories s ON c.parent_id = s.id
)
SELECT * FROM subcategories;
```

---

### Q5: What's the difference between UNION and UNION ALL in recursive CTEs?

**Answer:**
- **UNION ALL**: Keeps duplicates, **required** for recursion
- **UNION**: Removes duplicates, **terminates** recursion early

```sql
-- ✅ UNION ALL: Standard for recursion
WITH RECURSIVE cte AS (
    SELECT 1 as n
    UNION ALL  -- Keeps duplicates
    SELECT n + 1 FROM cte WHERE n < 10
)
SELECT * FROM cte;

-- ❌ UNION: May terminate early
WITH RECURSIVE cte AS (
    SELECT 1 as n
    UNION  -- Removes duplicates
    SELECT n + 1 FROM cte WHERE n < 10
)
SELECT * FROM cte;
-- If duplicate rows, recursion stops
```

---

## 🧩 11. Design Patterns

### Pattern 1: Data Pipeline (ETL)

```sql
WITH 
raw_data AS (
    SELECT * FROM source_table WHERE date >= '2024-01-01'
),
cleaned_data AS (
    SELECT 
        id,
        TRIM(UPPER(name)) as name,
        COALESCE(price, 0) as price
    FROM raw_data
    WHERE name IS NOT NULL
),
enriched_data AS (
    SELECT 
        cd.*,
        c.category_name
    FROM cleaned_data cd
    LEFT JOIN categories c ON cd.category_id = c.id
)
SELECT * FROM enriched_data;
```

### Pattern 2: Running Totals

```sql
WITH daily_sales AS (
    SELECT 
        DATE(created_at) as sale_date,
        SUM(total) as daily_total
    FROM orders
    GROUP BY DATE(created_at)
)
SELECT 
    sale_date,
    daily_total,
    SUM(daily_total) OVER (ORDER BY sale_date) as running_total
FROM daily_sales;
```

### Pattern 3: Graph Traversal  

```sql
-- Find shortest path between nodes
WITH RECURSIVE path AS (
    SELECT 
        id as node_id,
        target_id,
        1 as distance,
        CAST(id AS CHAR(1000)) as path
    FROM edges
    WHERE id = 1  -- Start node
    
    UNION ALL
    
    SELECT 
        e.target_id,
        e.target_id,
        p.distance + 1,
        CONCAT(p.path, '->', e.target_id)
    FROM edges e
    JOIN path p ON e.id = p.target_id
    WHERE p.distance < 10
      AND FIND_IN_SET(e.target_id, p.path) = 0  -- Avoid cycles
)
SELECT * FROM path
WHERE target_id = 10  -- End node
ORDER BY distance
LIMIT 1;
```

---

## 📚 Summary

### Key Takeaways

1. **CTE**: Temporary named result set using WITH clause
2. **Readability**: Break complex queries into logical steps
3. **Reusability**: Reference CTE multiple times
4. **Recursive CTE**: Solve hierarchical/graph problems
5. **Structure**: Anchor member + Recursive member + Termination
6. **Termination**: Always include WHERE to stop recursion
7. **UNION ALL**: Required for recursive CTEs
8. **Performance**: Similar to subqueries (optimizer dependent)
9. **Materialization**: PostgreSQL pre-v12 materializes, v12+ can inline
10. **Use cases**: Hierarchies, graphs, date series, data pipelines

**CTE Template:**
```sql
WITH 
cte1 AS (SELECT ...),
cte2 AS (SELECT ... FROM cte1),
cte3 AS (SELECT ... FROM cte2)
SELECT * FROM cte3;
```

---

**Next:** [07_Joins.md](07_Joins.md) - Table joins  
**Related:** [../05_Query_Optimization/](../05_Query_Optimization/) - CTE performance

---

*Last Updated: February 19, 2026*
