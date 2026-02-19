# 03 SQL Core

> Master fundamental SQL commands for database manipulation and querying.

---

## 📋 Overview

This section covers the **core SQL command categories** that every database professional must master. These commands form the foundation of database interaction—creating structures, manipulating data, controlling access, managing transactions, and combining data from multiple sources.

**What You'll Learn:**
- DDL commands for database object management
- DML commands for data manipulation
- DCL commands for access control
- TCL commands for transaction management
- Subqueries for nested data retrieval
- CTEs for readable complex queries
- Joins for multi-table operations

---

## 📂 Section Contents

### [01_DDL_Commands.md](01_DDL_Commands.md)
**Data Definition Language** - Schema management

- CREATE, ALTER, DROP, TRUNCATE
- Online DDL operations (zero-downtime schema changes)
- Best practices for production deployments
- Performance implications of schema changes

**Key Concepts:**
- IF EXISTS/IF NOT EXISTS for idempotent scripts
- ALGORITHM options (INPLACE, INSTANT, COPY)
- Concurrent index creation
- Safe column additions with NOT NULL

---

### [02_DML_Commands.md](02_DML_Commands.md)
**Data Manipulation Language** - Data operations

- SELECT, INSERT, UPDATE, DELETE
- Batch operations for performance
- Upserts (INSERT ... ON CONFLICT / ON DUPLICATE KEY UPDATE)
- N+1 query problem and solutions

**Key Concepts:**
- Avoid SELECT * in production
- Always use WHERE with UPDATE/DELETE
- Batch inserts (10-100x faster)
- Optimistic locking with version columns

---

### [03_DCL_Commands.md](03_DCL_Commands.md)
**Data Control Language** - Access control

- GRANT, REVOKE permissions
- User and role management
- Principle of least privilege
- Row-level security (PostgreSQL)

**Key Concepts:**
- Never use root in production
- Restrict by IP address (`'user'@'10.0.%'`)
- Use roles instead of individual grants
- Audit and rotate credentials regularly

---

### [04_TCL_Commands.md](04_TCL_Commands.md)
**Transaction Control Language** - Transaction management

- BEGIN, COMMIT, ROLLBACK, SAVEPOINT
- ACID properties
- Deadlock handling
- Optimistic locking patterns

**Key Concepts:**
- Always use transactions for multi-statement operations
- Keep transactions short to minimize lock duration
- Handle deadlocks with retry + exponential backoff
- Understanding MVCC (Multi-Version Concurrency Control)

---

### [05_Subqueries.md](05_Subqueries.md)
**Nested queries** - Queries within queries

- Scalar, column, row, table subqueries
- Correlated vs non-correlated
- EXISTS vs IN performance
- Converting subqueries to JOINs

**Key Concepts:**
- Use EXISTS for large datasets (short-circuits)
- Avoid correlated subqueries in hot paths
- NOT EXISTS handles NULL better than NOT IN
- JOINs usually faster than correlated subqueries

---

### [06_CTEs.md](06_CTEs.md)
**Common Table Expressions** - WITH clause

- Named temporary result sets
- Recursive CTEs for hierarchical data
- Multiple CTEs for readable queries
- CTE vs subquery trade-offs

**Key Concepts:**
- Break complex queries into logical steps
- Recursive CTEs: anchor + recursive + termination
- Always include max depth limit (prevent infinite recursion)
- Materialized vs inline CTEs (PostgreSQL)

---

### [07_Joins.md](07_Joins.md)
**Combining tables** - Multi-table queries

- INNER, LEFT, RIGHT, FULL OUTER, CROSS joins
- SELF joins
- Join algorithms (nested loop, hash, merge)
- Join optimization

**Key Concepts:**
- LEFT JOIN + IS NULL to find non-matching rows
- Always index join columns
- Covering indexes for join-heavy queries
- Avoid cartesian products (missing ON clause)

---

## 🎯 Learning Path

### Prerequisites
- Basic SQL syntax
- Understanding of relational database concepts
- Completion of **01_Database_Fundamentals** and **02_Data_Modeling**

### Recommended Order

**Foundations (Must Learn First):**
1. **01_DDL_Commands** - Create database structures
2. **02_DML_Commands** - Manipulate data
3. **07_Joins** - Combine tables

**Intermediate:**
4. **03_DCL_Commands** - Secure access
5. **04_TCL_Commands** - Manage transactions

**Advanced:**
6. **05_Subqueries** - Nested queries
7. **06_CTEs** - Complex query patterns

---

## 💡 Key Concepts Summary

### Command Categories

```
SQL COMMANDS
│
├── DDL (Data Definition Language)
│   └── Structure management: CREATE, ALTER, DROP
│
├── DML (Data Manipulation Language)
│   └── Data operations: SELECT, INSERT, UPDATE, DELETE
│
├── DCL (Data Control Language)
│   └── Access control: GRANT, REVOKE
│
└── TCL (Transaction Control Language)
    └── Transactions: BEGIN, COMMIT, ROLLBACK
```

### Performance Hierarchy

**Fastest → Slowest:**
1. Indexed JOIN
2. Non-correlated subquery with index
3. CTE (depends on materialization)
4. Subquery in FROM clause
5. Correlated subquery with index
6. Correlated subquery without index

---

## 🧪 Practice Exercises

### Exercise 1: Multi-Table Query
```sql
-- Find top 10 customers by total spending (last 30 days)
-- Include: customer name, order count, total spent
-- Use JOINs and aggregations
```

<details>
<summary>Solution</summary>

```sql
SELECT 
    c.name,
    c.email,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'completed'
  AND o.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY c.id, c.name, c.email
ORDER BY total_spent DESC
LIMIT 10;
```
</details>

---

### Exercise 2: Find Gaps
```sql
-- Find customers who have NOT placed an order in the last 90 days
-- Include: customer name, last order date
```

<details>
<summary>Solution</summary>

```sql
-- Method 1: LEFT JOIN with IS NULL
SELECT 
    c.name,
    MAX(o.created_at) as last_order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id 
    AND o.created_at > DATE_SUB(NOW(), INTERVAL 90 DAY)
WHERE o.id IS NULL
GROUP BY c.id, c.name;

-- Method 2: NOT EXISTS (often faster)
SELECT 
    c.name,
    (SELECT MAX(created_at) FROM orders WHERE customer_id = c.id) as last_order_date
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 
    FROM orders 
    WHERE customer_id = c.id 
      AND created_at > DATE_SUB(NOW(), INTERVAL 90 DAY)
);
```
</details>

---

### Exercise 3: Recursive CTE
```sql
-- Build org chart showing employee hierarchy
-- Include: employee name, manager name, level
-- Use recursive CTE
```

<details>
<summary>Solution</summary>

```sql
WITH RECURSIVE org_chart AS (
    -- Anchor: Top-level (CEO, no manager)
    SELECT 
        id,
        name,
        manager_id,
        CAST(NULL AS CHAR(100)) as manager_name,
        0 as level,
        CAST(name AS CHAR(500)) as path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: Direct reports
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        oc.name as manager_name,
        oc.level + 1,
        CONCAT(oc.path, ' → ', e.name)
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
    WHERE oc.level < 10  -- Max 10 levels
)
SELECT 
    REPEAT('  ', level) || name as hierarchy,
    manager_name,
    level
FROM org_chart
ORDER BY path;
```
</details>

---

### Exercise 4: Transaction Safety
```sql
-- Implement a bank transfer with proper transaction handling
-- Requirements:
-- - Debit from account A
-- - Credit to account B
-- - Ensure atomicity (both or neither)
-- - Check for sufficient funds
```

<details>
<summary>Solution</summary>

```sql
BEGIN;
    -- Debit sender (with row lock)
    UPDATE accounts 
    SET balance = balance - 100 
    WHERE id = 1 AND balance >= 100;
    
    -- Check if update succeeded
    IF ROW_COUNT() = 0 THEN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' 
            SET MESSAGE_TEXT = 'Insufficient funds or account not found';
    END IF;
    
    -- Credit receiver
    UPDATE accounts 
    SET balance = balance + 100 
    WHERE id = 2;
    
    -- Log transaction
    INSERT INTO transaction_log (from_account, to_account, amount, timestamp)
    VALUES (1, 2, 100, NOW());
    
COMMIT;
```
</details>

---

## 🎓 Interview Focus Areas

### Essential Questions

**DDL:**
- DROP vs TRUNCATE vs DELETE comparison
- How to add NOT NULL column without downtime
- Online DDL vs offline DDL trade-offs

**DML:**
- N+1 query problem and solutions
- Batch operations for performance
- Upserts (INSERT ... ON CONFLICT)

**DCL:**
- Principle of least privilege
- Role-based access control
- Row-level security implementation

**TCL:**
- ACID properties
- Deadlock causes and handling
- Isolation levels

**Joins:**
- INNER vs LEFT JOIN
- Finding non-matching rows (LEFT JOIN + IS NULL)
- Join algorithm types (nested loop, hash, merge)

**Subqueries/CTEs:**
- Correlated vs non-correlated subqueries
- EXISTS vs IN performance
- Recursive CTE structure and use cases

---

## 🔗 Related Sections

**Prerequisites:**
- [01_Database_Fundamentals](../01_Database_Fundamentals/) - ACID, normalization
- [02_Data_Modeling](../02_Data_Modeling/) - Keys, constraints, relationships

**Next Steps:**
- [04_Advanced_SQL](../04_Advanced_SQL/) - Window functions, pivoting, JSON
- [05_Query_Optimization](../05_Query_Optimization/) - Performance tuning
- [07_Transactions_And_Concurrency](../07_Transactions_And_Concurrency/) - Advanced transaction patterns

**Related:**
- [06_Indexing_Strategies](../06_Indexing_Strategies/) - Index design for joins
- [11_PostgreSQL](../11_PostgreSQL/) - PostgreSQL-specific SQL features
- [12_MySQL](../12_MySQL/) - MySQL-specific SQL features

---

## 📚 Additional Resources

### Books
- **SQL Antipatterns** by Bill Karwin - Common mistakes
- **SQL Performance Explained** by Markus Winand - Query optimization
- **High Performance MySQL** by Baron Schwartz - MySQL-specific

### Articles
- [Use The Index, Luke!](https://use-the-index-luke.com/) - SQL indexing and tuning
- [Modern SQL](https://modern-sql.com/) - SQL standard features
- [PostgreSQL Docs](https://www.postgresql.org/docs/) - Official documentation

### Practice
- [LeetCode Database](https://leetcode.com/problemset/database/) - SQL problems
- [HackerRank SQL](https://www.hackerrank.com/domains/sql) - Practice exercises
- [SQLZoo](https://sqlzoo.net/) - Interactive tutorials

---

## ✅ Section Checklist

**Foundations:**
- [ ] Create tables with proper constraints
- [ ] Insert, update, delete data safely
- [ ] Write INNER and LEFT JOINs
- [ ] Use transactions for consistency

**Intermediate:**
- [ ] Implement role-based access control
- [ ] Handle deadlocks with retry logic
- [ ] Write subqueries in WHERE and FROM clauses
- [ ] Use EXISTS for performance

**Advanced:**
- [ ] Optimize slow multi-table joins
- [ ] Write recursive CTEs for hierarchies
- [ ] Implement row-level security
- [ ] Use online DDL for zero-downtime schema changes

---

**Estimated Time to Master:** 2-3 weeks of daily practice  
**Difficulty:** ★★☆☆☆ (Beginner to Intermediate)

---

*Last Updated: February 19, 2026*
