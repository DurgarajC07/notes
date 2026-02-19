# 🔍 Subqueries - Nested SQL Queries

> Subqueries are queries within queries. Master nested SELECT statements to write powerful, expressive SQL for complex data retrieval.

---

## 📖 1. Concept Explanation

### What is a Subquery?

A **subquery** (or **nested query**) is a SELECT statement embedded within another SQL statement.

**Types of Subqueries:**
```
SUBQUERIES
│
├── By Position
│   ├── SELECT clause (scalar subquery)
│   ├── FROM clause (derived table)
│   ├── WHERE clause (filter)
│   └── HAVING clause (aggregate filter)
│
├── By Return Type
│   ├── Scalar (single value)
│   ├── Column (single column, multiple rows)
│   ├── Row (single row, multiple columns)
│   └── Table (multiple rows and columns)
│
└── By Execution
    ├── Non-correlated (runs once)
    └── Correlated (runs per outer row)
```

**Basic Syntax:**
```sql
-- Subquery in WHERE clause
SELECT name 
FROM users 
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- Scalar subquery in SELECT
SELECT 
    name,
    (SELECT COUNT(*) FROM orders WHERE user_id = users.id) as order_count
FROM users;
```

---

## 🧠 2. Why It Matters in Real Systems

### Find Users with Above-Average Orders

**❌ Bad: Two separate queries (application joins)**
```sql
-- Query 1: Get average
SELECT AVG(total) FROM orders;  -- Returns 150.00

-- Query 2: Find users above average (in application)
SELECT user_id FROM orders WHERE total > 150.00;

-- Problems:
-- - Two round trips to database ❌
-- - Race condition (average changes between queries) ❌
-- - More complex application code ❌
```

**✅ Good: Single query with subquery**
```sql
SELECT user_id, total
FROM orders
WHERE total > (SELECT AVG(total) FROM orders);
-- One query, consistent snapshot ✅
```

---

## ⚙️ 3. Internal Working

### Non-Correlated Subquery Execution

```sql
SELECT name
FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- Execution:
-- 1. Run subquery once: SELECT user_id FROM orders WHERE total > 1000
--    Result: [1, 5, 9]
-- 2. Rewrite outer query: SELECT name FROM users WHERE id IN (1, 5, 9)
-- 3. Execute outer query
```

### Correlated Subquery Execution

```sql
SELECT name,
       (SELECT COUNT(*) FROM orders WHERE user_id = users.id) as order_count
FROM users;

-- Execution:
-- 1. Fetch first user (id=1)
-- 2. Run subquery: SELECT COUNT(*) FROM orders WHERE user_id = 1
-- 3. Fetch second user (id=2)
-- 4. Run subquery: SELECT COUNT(*) FROM orders WHERE user_id = 2
-- ...
-- N users = N subquery executions (can be slow!)
```

---

## ✅ 4. Best Practices

### Use JOINs Instead of Correlated Subqueries

```sql
-- ❌ Slow: Correlated subquery (runs N times)
SELECT u.name,
       (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) as order_count
FROM users u;
-- If 10,000 users: 10,000 subquery executions ❌

-- ✅ Fast: JOIN with GROUP BY
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
-- Single query execution ✅
```

### EXISTS vs IN for Large Datasets

```sql
-- ❌ Slower: IN with large subquery result
SELECT name 
FROM users 
WHERE id IN (SELECT user_id FROM huge_orders_table);
-- IN loads all IDs into memory ❌

-- ✅ Faster: EXISTS (short-circuits)
SELECT name
FROM users u
WHERE EXISTS (SELECT 1 FROM huge_orders_table o WHERE o.user_id = u.id);
-- Stops at first match ✅
```

### Avoid SELECT * in Subqueries

```sql
-- ❌ Wasteful
SELECT user_id
FROM (
    SELECT * FROM orders  -- Fetches all columns ❌
) AS subq
WHERE total > 100;

-- ✅ Efficient
SELECT user_id
FROM (
    SELECT user_id, total FROM orders  -- Only needed columns ✅
) AS subq
WHERE total > 100;
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Forgetting Subquery Alias

```sql
-- ❌ Error: Every derived table must have an alias
SELECT * FROM (SELECT id, name FROM users);
-- ERROR 1248: Every derived table must have its own alias

-- ✅ Correct
SELECT * FROM (SELECT id, name FROM users) AS u;
```

### Mistake 2: Returning Multiple Columns for = Comparison

```sql
-- ❌ Error: Subquery returns more than 1 column
SELECT name
FROM users
WHERE id = (SELECT id, email FROM users WHERE name = 'John');
-- ERROR 1241: Operand should contain 1 column(s)

-- ✅ Correct: Return single column
SELECT name
FROM users
WHERE id = (SELECT id FROM users WHERE name = 'John' LIMIT 1);
```

### Mistake 3: Correlated Subquery Performance

```sql
-- ❌ Very slow: Correlated subquery in large table
SELECT p.name,
       (SELECT AVG(price) FROM products WHERE category = p.category) as avg_price
FROM products p;
-- Runs AVG calculation for EVERY product ❌

-- ✅ Fast: JOIN with pre-aggregated data
SELECT p.name, c.avg_price
FROM products p
JOIN (
    SELECT category, AVG(price) as avg_price
    FROM products
    GROUP BY category
) c ON p.category = c.category;
-- AVG calculated once per category ✅
```

### Mistake 4: Using = Instead of IN

```sql
-- ❌ Error if subquery returns multiple rows
SELECT name
FROM users
WHERE id = (SELECT user_id FROM orders);
-- ERROR 1242: Subquery returns more than 1 row

-- ✅ Use IN for multiple values
SELECT name
FROM users
WHERE id IN (SELECT user_id FROM orders);
```

---

## 🔐 6. Security Considerations

### SQL Injection in Dynamic Subqueries

```sql
-- ❌ Vulnerable: String concatenation
$sql = "SELECT * FROM users WHERE id IN (
    SELECT user_id FROM orders WHERE status = '" . $user_input . "'
)";
-- Input: ' OR '1'='1
-- Becomes: ... WHERE status = '' OR '1'='1'  ❌

-- ✅ Safe: Parameterized
$stmt = $pdo->prepare("
    SELECT * FROM users WHERE id IN (
        SELECT user_id FROM orders WHERE status = ?
    )
");
$stmt->execute([$user_input]);
```

---

## 🚀 7. Performance Optimization

### Materialize Subquery to Temporary Table

```sql
-- ❌ Slow: Complex subquery used multiple times
SELECT *
FROM orders o1
WHERE total > (SELECT AVG(total) FROM orders)
  AND user_id IN (SELECT user_id FROM orders WHERE total > (SELECT AVG(total) FROM orders));

-- ✅ Fast: Calculate once, reuse
CREATE TEMPORARY TABLE avg_order AS
SELECT AVG(total) as avg_total FROM orders;

SELECT *
FROM orders o
WHERE total > (SELECT avg_total FROM avg_order);
```

### Index Columns Used in Subqueries

```sql
-- Subquery
SELECT name 
FROM users 
WHERE id IN (SELECT user_id FROM orders WHERE status = 'completed');

-- ✅ Add indexes
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_user_id ON orders(user_id);
-- Subquery now uses index scan instead of full table scan ✅
```

---

## 🧪 8. Examples

### Scalar Subquery - Single Value

```sql
-- Get users with above-average order total
SELECT name, 
       (SELECT AVG(total) FROM orders) as avg_order,
       total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE total > (SELECT AVG(total) FROM orders);
```

### Column Subquery - Multiple Rows, One Column

```sql
-- Find products in categories that have sales
SELECT name, category
FROM products
WHERE category IN (
    SELECT DISTINCT category 
    FROM orders 
    WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
);
```

### Row Subquery - One Row, Multiple Columns

```sql
-- Find user with same (city, country) as user 123
SELECT name
FROM users
WHERE (city, country) = (
    SELECT city, country 
    FROM users 
    WHERE id = 123
);
```

### Table Subquery (Derived Table)

```sql
-- Top 5 users by total spending
SELECT u.name, subq.total_spent
FROM users u
JOIN (
    SELECT user_id, SUM(total) as total_spent
    FROM orders
    GROUP BY user_id
    ORDER BY total_spent DESC
    LIMIT 5
) subq ON u.id = subq.user_id;
```

### EXISTS - Check for Existence

```sql
-- Users who have placed at least one order
SELECT name
FROM users u
WHERE EXISTS (
    SELECT 1 
    FROM orders o 
    WHERE o.user_id = u.id
);

-- Users with NO orders
SELECT name
FROM users u
WHERE NOT EXISTS (
    SELECT 1 
    FROM orders o 
    WHERE o.user_id = u.id
);
```

### NOT IN vs NOT EXISTS

```sql
-- ❌ NOT IN: NULL handling issues
SELECT name
FROM users
WHERE id NOT IN (SELECT user_id FROM orders);
-- If user_id has NULL: returns 0 rows! ❌

-- ✅ NOT EXISTS: Handles NULL correctly
SELECT name
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: E-Commerce - Find Repeat Customers

```sql
-- Customers with more than 3 orders
SELECT u.name, u.email, subq.order_count
FROM users u
JOIN (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
    HAVING COUNT(*) > 3
) subq ON u.id = subq.user_id
ORDER BY subq.order_count DESC;
```

### Case Study 2: Social Media - Mutual Friends

```sql
-- Find mutual friends between user 1 and user 2
SELECT u.name
FROM users u
WHERE u.id IN (
    -- Friends of user 1
    SELECT friend_id FROM friendships WHERE user_id = 1
)
AND u.id IN (
    -- Friends of user 2
    SELECT friend_id FROM friendships WHERE user_id = 2
);
```

### Case Study 3: Analytics - Cohort Analysis

```sql
-- Users who made their first purchase in January 2024
-- and have purchased again
SELECT u.name, u.email
FROM users u
WHERE u.id IN (
    -- First purchase in Jan 2024
    SELECT user_id
    FROM orders
    WHERE DATE_FORMAT(created_at, '%Y-%m') = '2024-01'
    GROUP BY user_id
    HAVING MIN(created_at) BETWEEN '2024-01-01' AND '2024-01-31'
)
AND EXISTS (
    -- Has repeat purchase
    SELECT 1
    FROM orders o2
    WHERE o2.user_id = u.id
      AND o2.created_at > '2024-01-31'
);
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between a subquery and a JOIN?

**Answer:**
- **Subquery**: Query within a query, returns data used by outer query
- **JOIN**: Combines rows from multiple tables based on a condition

**Subquery:**
```sql
SELECT name
FROM users
WHERE id IN (SELECT user_id FROM orders);
```

**JOIN:**
```sql
SELECT DISTINCT u.name
FROM users u
JOIN orders o ON u.id = o.user_id;
```

**When to use each:**
- **Subquery**: Filtering, aggregations, readability
- **JOIN**: Performance (especially LEFT JOIN), accessing columns from multiple tables

---

### Q2: What's a correlated subquery?

**Answer:** A subquery that **references columns from the outer query**. It runs **once per outer row** (can be slow).

```sql
-- Correlated: References u.id from outer query
SELECT u.name,
       (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) as order_count
FROM users u;

-- Non-correlated: Runs once independently
SELECT name
FROM users
WHERE id IN (SELECT user_id FROM orders);
```

---

### Q3: What's the difference between IN and EXISTS?

**Answer:**

| Feature | IN | EXISTS |
|---------|-----|---------|
| **Returns** | List of values | Boolean (TRUE/FALSE) |
| **NULL handling** | Issues with NULL | Handles NULL correctly |
| **Performance** | Loads all results | Short-circuits at first match |
| **Best for** | Small result sets | Large result sets |

```sql
-- IN: Returns list of IDs
SELECT name FROM users WHERE id IN (1, 2, 3);

-- EXISTS: Returns TRUE/FALSE
SELECT name FROM users u 
WHERE EXISTS (SELECT 1 FROM orders WHERE user_id = u.id);
-- Stops at first match ✅
```

---

### Q4: Can you use ORDER BY in a subquery?

**Answer:** **Generally no**, unless:
- Using LIMIT (then ORDER BY is meaningful)
- In derived table (FROM clause)

```sql
-- ❌ Ignored: ORDER BY without LIMIT
SELECT id FROM (
    SELECT id FROM users ORDER BY created_at DESC
) subq;

-- ✅ Meaningful: ORDER BY with LIMIT
SELECT id FROM (
    SELECT id FROM users ORDER BY created_at DESC LIMIT 10
) subq;
```

---

### Q5: How do you optimize a slow subquery?

**Answer:**

1. **Convert correlated to JOIN:**
```sql
-- ❌ Slow
SELECT (SELECT COUNT(*) FROM orders WHERE user_id = u.id)
FROM users u;

-- ✅ Fast
SELECT u.id, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;
```

2. **Use EXISTS instead of IN:**
```sql
-- ❌ Slower
WHERE id IN (SELECT user_id FROM huge_table);

-- ✅ Faster
WHERE EXISTS (SELECT 1 FROM huge_table WHERE user_id = id);
```

3. **Add indexes:**
```sql
CREATE INDEX idx_user_id ON orders(user_id);
```

4. **Materialize complex subqueries:**
```sql
CREATE TEMPORARY TABLE temp_results AS
SELECT user_id, COUNT(*) as cnt FROM orders GROUP BY user_id;

SELECT * FROM users u
JOIN temp_results t ON u.id = t.user_id;
```

---

## 🧩 11. Design Patterns

### Pattern 1: Filtering with Subquery

```sql
-- Find products in best-selling categories
SELECT name, price
FROM products
WHERE category IN (
    SELECT category
    FROM order_items oi
    JOIN products p ON oi.product_id = p.id
    GROUP BY category
    ORDER BY SUM(oi.quantity) DESC
    LIMIT 5
);
```

### Pattern 2: Comparing to Aggregates

```sql
-- Employees earning more than department average
SELECT e.name, e.salary, e.department
FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department = e.department
);
```

### Pattern 3: Derived Table for Complex Calculations

```sql
-- Customer lifetime value (CLV)
SELECT u.name, clv.total_revenue, clv.order_count
FROM users u
JOIN (
    SELECT 
        user_id,
        SUM(total) as total_revenue,
        COUNT(*) as order_count,
        AVG(total) as avg_order_value
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
) clv ON u.id = clv.user_id
WHERE clv.total_revenue > 1000
ORDER BY clv.total_revenue DESC;
```

### Pattern 4: Self-Referencing with NOT EXISTS

```sql
-- Find gaps in sequential IDs
SELECT t1.id + 1 AS gap_start
FROM table_name t1
WHERE NOT EXISTS (
    SELECT 1 FROM table_name t2 
    WHERE t2.id = t1.id + 1
)
AND t1.id < (SELECT MAX(id) FROM table_name);
```

---

## 📚 Summary

### Key Takeaways

1. **Subquery**: Query within a query, used for filtering and calculations
2. **Types**: Scalar, column, row, table subqueries
3. **Correlated**: References outer query (slow), runs per row
4. **Non-correlated**: Independent (fast), runs once
5. **EXISTS**: Faster than IN for large datasets
6. **NOT EXISTS**: Handles NULL better than NOT IN
7. **JOINs**: Usually faster than correlated subqueries
8. **Derived tables**: Must have alias
9. **Optimization**: Add indexes, convert to JOINs, materialize if needed
10. **Use cases**: Filtering, comparisons to aggregates, complex calculations

**Performance Hierarchy (Fast → Slow):**
```
1. Non-correlated subquery with indexes
2. JOIN
3. EXISTS
4. IN
5. Correlated subquery without indexes
```

---

**Next:** [06_CTEs.md](06_CTEs.md) - Common Table Expressions  
**Related:** [../05_Query_Optimization/](../05_Query_Optimization/) - Query performance

---

*Last Updated: February 19, 2026*
