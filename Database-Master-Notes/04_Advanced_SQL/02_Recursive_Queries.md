# 🔄 Recursive Queries - Hierarchical Data Traversal

> Recursive queries traverse hierarchical and graph-like data structures using recursive CTEs. Master self-referencing queries for org charts, category trees, and network analysis.

---

## 📖 1. Concept Explanation

### What are Recursive Queries?

**Recursive queries** use **recursive CTEs** (Common Table Expressions) to traverse hierarchical or graph-like data by repeatedly referencing themselves.

**Structure:**
```sql
WITH RECURSIVE cte_name AS (
    -- Anchor member (base case): Starting point
    SELECT ...
    WHERE condition
    
    UNION ALL
    
    -- Recursive member: References CTE itself
    SELECT ...
    FROM cte_name
    JOIN table ON condition
    WHERE termination_condition
)
SELECT * FROM cte_name;
```

**Execution Flow:**
```
1. Execute anchor member → Initial result set
2. Execute recursive member using result from step 1
3. Execute recursive member using result from step 2
4. Repeat until no new rows (or max depth reached)
5. UNION ALL combines all iterations
```

**Common Use Cases:**
```
RECURSIVE QUERIES
│
├── Organizational Hierarchies
│   └── Employee → Manager relationships
│
├── Category/Taxonomy Trees
│   └── Parent → Child categories
│
├── Bill of Materials (BOM)
│   └── Product → Component relationships
│
├── Social Networks
│   └── Friend → Friend connections (degrees of separation)
│
├── File System Traversal
│   └── Directory → Subdirectory paths
│
└── Graph Analysis
    ├── Shortest paths
    └── Reachability analysis
```

---

## 🧠 2. Why It Matters in Real Systems

### Non-Recursive Approach (Limited Depth)

**❌ Bad: Multiple self-joins for fixed depth**
```sql
-- Get 3 levels of employee hierarchy (hardcoded depth)
SELECT 
    e1.id as level1_id,
    e1.name as level1_name,
    e2.name as level2_name,
    e3.name as level3_name
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id
LEFT JOIN employees e3 ON e2.manager_id = e3.id;

-- Problems:
-- - Only works for exactly 3 levels ❌
-- - Need to rewrite for different depths ❌
-- - Can't handle variable depth hierarchies ❌
```

**✅ Good: Recursive CTE (unlimited depth)**
```sql
WITH RECURSIVE hierarchy AS (
    -- Anchor: Top level (no manager)
    SELECT id, name, manager_id, 0 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: All subordinates
    SELECT e.id, e.name, e.manager_id, h.level + 1
    FROM employees e
    JOIN hierarchy h ON e.manager_id = h.id
    WHERE h.level < 10  -- Max depth protection
)
SELECT * FROM hierarchy;
-- Handles any depth dynamically ✅
```

---

## ⚙️ 3. Internal Working

### Recursive CTE Execution

```sql
WITH RECURSIVE numbers AS (
    SELECT 1 as n, 1 as factorial
    UNION ALL
    SELECT n + 1, factorial * (n + 1)
    FROM numbers
    WHERE n < 5
)
SELECT * FROM numbers;

-- Execution steps:
-- Iteration 0 (Anchor): n=1, factorial=1
-- Iteration 1: n=2, factorial=2  (1 * 2)
-- Iteration 2: n=3, factorial=6  (2 * 3)
-- Iteration 3: n=4, factorial=24 (6 * 4)
-- Iteration 4: n=5, factorial=120 (24 * 5)
-- Iteration 5: WHERE n < 5 false, stop
-- 
-- Result: 1, 2, 6, 24, 120
```

### Memory Management

```sql
-- PostgreSQL stores intermediate results in working table
-- Each iteration reads from previous iteration's output
-- Memory usage: O(max_depth * rows_per_level)

-- For large hierarchies, consider iterative approach:
DO $$
DECLARE
    current_level INT := 0;
BEGIN
    CREATE TEMP TABLE hierarchy_temp (
        id INT,
        parent_id INT,
        level INT
    );
    
    -- Insert level 0
    INSERT INTO hierarchy_temp
    SELECT id, parent_id, 0
    FROM categories
    WHERE parent_id IS NULL;
    
    -- Iteratively add levels
    LOOP
        INSERT INTO hierarchy_temp
        SELECT c.id, c.parent_id, current_level + 1
        FROM categories c
        JOIN hierarchy_temp h ON c.parent_id = h.id
        WHERE h.level = current_level;
        
        EXIT WHEN NOT FOUND;
        current_level := current_level + 1;
    END LOOP;
END $$;
```

---

## ✅ 4. Best Practices

### Always Include Termination Condition

```sql
-- ✅ Max depth limit (prevent infinite loops)
WITH RECURSIVE hierarchy AS (
    SELECT id, parent_id, 0 as depth
    FROM nodes
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT n.id, n.parent_id, h.depth + 1
    FROM nodes n
    JOIN hierarchy h ON n.parent_id = h.id
    WHERE h.depth < 100  -- ✅ Safety limit
)
SELECT * FROM hierarchy;

-- ❌ No termination: Infinite loop if cycle exists
WITH RECURSIVE hierarchy AS (
    SELECT id, parent_id FROM nodes WHERE parent_id IS NULL
    UNION ALL
    SELECT n.id, n.parent_id FROM nodes n
    JOIN hierarchy h ON n.parent_id = h.id
)
SELECT * FROM hierarchy;  -- ❌ Can run forever
```

### Detect and Prevent Cycles

```sql
-- ✅ Track visited nodes to prevent cycles
WITH RECURSIVE path AS (
    SELECT 
        id,
        parent_id,
        ARRAY[id] as path_ids,
        0 as depth
    FROM nodes
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT 
        n.id,
        n.parent_id,
        p.path_ids || n.id,
        p.depth + 1
    FROM nodes n
    JOIN path p ON n.parent_id = p.id
    WHERE NOT (n.id = ANY(p.path_ids))  -- ✅ Prevent revisiting
      AND p.depth < 100
)
SELECT * FROM path;
```

### Use Breadth-First vs Depth-First

```sql
-- Breadth-first: Use UNION (not UNION ALL) + ORDER BY
-- Processes all nodes at level N before level N+1
WITH RECURSIVE breadth_first AS (
    SELECT id, parent_id, 0 as level
    FROM nodes
    WHERE parent_id IS NULL
    
    UNION  -- Not UNION ALL
    
    SELECT n.id, n.parent_id, bf.level + 1
    FROM nodes n
    JOIN breadth_first bf ON n.parent_id = bf.id
    WHERE bf.level < 10
)
SELECT * FROM breadth_first
ORDER BY level, id;  -- ✅ Processes level by level

-- Depth-first: Use UNION ALL + specific ordering
-- Standard recursive CTE with path tracking
```

### Index Parent-Child Relationships

```sql
-- ✅ Add index on foreign key for fast joins
CREATE INDEX idx_parent ON nodes(parent_id);

-- Recursive query performs join on each iteration
-- Without index: O(N²) full table scans ❌
-- With index: O(N log N) index lookups ✅
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Missing RECURSIVE Keyword

```sql
-- ❌ Error: Recursive reference without RECURSIVE
WITH hierarchy AS (
    SELECT id, parent_id FROM nodes WHERE parent_id IS NULL
    UNION ALL
    SELECT n.id, n.parent_id FROM nodes n
    JOIN hierarchy h ON n.parent_id = h.id
)
SELECT * FROM hierarchy;
-- ERROR: recursive reference to query "hierarchy"

-- ✅ Correct: Add RECURSIVE
WITH RECURSIVE hierarchy AS ( ... )
```

### Mistake 2: No Termination Condition

```sql
-- ❌ Infinite loop with circular reference
-- Data: node 1 → node 2 → node 1 (cycle)
WITH RECURSIVE bad_recursion AS (
    SELECT id, parent_id FROM nodes WHERE id = 1
    UNION ALL
    SELECT n.id, n.parent_id FROM nodes n
    JOIN bad_recursion br ON n.parent_id = br.id
)
SELECT * FROM bad_recursion;
-- Runs forever! ❌

-- ✅ Add depth limit
WHERE depth < 100  -- Stops at 100 levels
```

### Mistake 3: Incorrect Anchor Member

```sql
-- ❌ Anchor returns no rows
WITH RECURSIVE hierarchy AS (
    SELECT id, parent_id, 0 as level
    FROM employees
    WHERE id = 99999  -- Non-existent ID ❌
    
    UNION ALL
    
    SELECT e.id, e.parent_id, h.level + 1
    FROM employees e
    JOIN hierarchy h ON e.manager_id = h.id
)
SELECT * FROM hierarchy;
-- Returns empty (anchor had no rows) ❌

-- ✅ Verify anchor returns expected rows
WHERE manager_id IS NULL  -- Top level
```

### Mistake 4: Performance on Large Hierarchies

```sql
-- ❌ Slow: No index, large hierarchy
WITH RECURSIVE hierarchy AS (
    SELECT id, parent_id, 0 as depth
    FROM categories  -- 1 million rows
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.id, c.parent_id, h.depth + 1
    FROM categories c
    JOIN hierarchy h ON c.parent_id = h.id
    WHERE h.depth < 50
)
SELECT * FROM hierarchy;
-- Time: 60+ seconds ❌

-- ✅ Add index
CREATE INDEX idx_parent_id ON categories(parent_id);
-- Time: 2 seconds ✅
```

---

## 🔐 6. Security Considerations

### Limit Recursion Depth

```sql
-- ✅ Set session-level recursion limit (PostgreSQL)
SET max_recursion_depth = 100;

-- ✅ Application-level limit in query
WHERE depth < 50  -- Max 50 levels
```

### Prevent Resource Exhaustion

```sql
-- ✅ Statement timeout
SET statement_timeout = '30s';

WITH RECURSIVE hierarchy AS ( ... )
SELECT * FROM hierarchy;
-- Kills query after 30 seconds if still running
```

---

## 🚀 7. Performance Optimization

### Materialize Frequent Hierarchies

```sql
-- ✅ Pre-compute for read-heavy workloads
CREATE TABLE category_paths AS
WITH RECURSIVE paths AS (
    SELECT 
        id,
        name,
        parent_id,
        name::TEXT as path,
        0 as depth
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT 
        c.id,
        c.name,
        c.parent_id,
        p.path || ' > ' || c.name,
        p.depth + 1
    FROM categories c
    JOIN paths p ON c.parent_id = p.id
    WHERE p.depth < 20
)
SELECT * FROM paths;

CREATE INDEX idx_category_paths ON category_paths(id);

-- Queries now instant ✅
SELECT * FROM category_paths WHERE id = 123;
```

### Use Closure Tables for Complex Queries

```sql
-- Closure table: Pre-compute all ancestor-descendant pairs
CREATE TABLE category_closure (
    ancestor_id INT,
    descendant_id INT,
    depth INT,
    PRIMARY KEY (ancestor_id, descendant_id)
);

-- Populate using recursive query once
INSERT INTO category_closure
WITH RECURSIVE closure AS (
    -- Self-references
    SELECT id, id, 0 FROM categories
    
    UNION ALL
    
    -- Parent-child relationships
    SELECT p.ancestor_id, c.id, p.depth + 1
    FROM closure p
    JOIN categories c ON p.descendant_id = c.parent_id
)
SELECT * FROM closure;

CREATE INDEX idx_ancestor ON category_closure(ancestor_id);
CREATE INDEX idx_descendant ON category_closure(descendant_id);

-- Now queries are simple lookups:
-- Get all descendants: SELECT descendant_id FROM category_closure WHERE ancestor_id = 5
-- Get all ancestors: SELECT ancestor_id FROM category_closure WHERE descendant_id = 10
-- ✅ O(1) lookups instead of O(N) recursion
```

---

## 🧪 8. Examples

### Employee Hierarchy (Org Chart)

```sql
-- Build complete org chart from CEO down
WITH RECURSIVE org_chart AS (
    -- Anchor: CEO (no manager)
    SELECT 
        id,
        name,
        title,
        manager_id,
        0 as level,
        name::TEXT as path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: All subordinates
    SELECT 
        e.id,
        e.name,
        e.title,
        e.manager_id,
        oc.level + 1,
        oc.path || ' → ' || e.name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
    WHERE oc.level < 20  -- Max 20 levels
)
SELECT 
    REPEAT('  ', level) || name as hierarchy,
    title,
    level,
    path
FROM org_chart
ORDER BY path;

-- Output:
-- CEO
--   VP Engineering
--     Director of Backend
--       Senior Engineer
--       Junior Engineer
--   VP Sales
--     Sales Manager
```

### Category Tree with Product Counts

```sql
-- Get category tree with product counts at each level
WITH RECURSIVE category_tree AS (
    SELECT 
        c.id,
        c.name,
        c.parent_id,
        0 as level
    FROM categories c
    WHERE c.parent_id IS NULL
    
    UNION ALL
    
    SELECT 
        c.id,
        c.name,
        c.parent_id,
        ct.level + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
    WHERE ct.level < 10
)
SELECT 
    REPEAT('  ', level) || ct.name as category_hierarchy,
    ct.level,
    COUNT(p.id) as product_count,
    SUM(COUNT(p.id)) OVER (PARTITION BY ct.id) as total_products_in_subtree
FROM category_tree ct
LEFT JOIN products p ON p.category_id = ct.id
GROUP BY ct.id, ct.name, ct.level
ORDER BY ct.id;
```

### Bill of Materials (BOM) - Product Components

```sql
-- Calculate total cost of product including all components
WITH RECURSIVE bom AS (
    -- Anchor: Top-level product
    SELECT 
        product_id,
        component_id,
        quantity,
        1 as level,
        quantity as total_quantity
    FROM product_components
    WHERE product_id = 12345  -- iPhone
    
    UNION ALL
    
    -- Recursive: Sub-components
    SELECT 
        pc.product_id,
        pc.component_id,
        pc.quantity,
        b.level + 1,
        b.total_quantity * pc.quantity
    FROM product_components pc
    JOIN bom b ON pc.product_id = b.component_id
    WHERE b.level < 10
)
SELECT 
    c.name as component_name,
    b.level,
    b.total_quantity,
    c.unit_cost,
    b.total_quantity * c.unit_cost as total_cost
FROM bom b
JOIN components c ON b.component_id = c.id
ORDER BY b.level, c.name;
```

### Social Network - Degrees of Separation

```sql
-- Find all friends within 3 degrees of user
WITH RECURSIVE friend_network AS (
    -- Anchor: Direct friends (1st degree)
    SELECT 
        friend_id as user_id,
        1 as degree,
        ARRAY[12345, friend_id] as path
    FROM friendships
    WHERE user_id = 12345
    
    UNION
    
    -- Recursive: Friends of friends
    SELECT 
        f.friend_id,
        fn.degree + 1,
        fn.path || f.friend_id
    FROM friendships f
    JOIN friend_network fn ON f.user_id = fn.user_id
    WHERE fn.degree < 3
      AND NOT (f.friend_id = ANY(fn.path))  -- Prevent cycles
)
SELECT 
    u.name,
    fn.degree,
    CASE fn.degree
        WHEN 1 THEN 'Direct friend'
        WHEN 2 THEN 'Friend of friend'
        WHEN 3 THEN '3rd degree connection'
    END as relationship
FROM friend_network fn
JOIN users u ON fn.user_id = u.id
ORDER BY fn.degree, u.name;
```

### File System Traversal

```sql
-- Get full file paths in directory tree
WITH RECURSIVE file_paths AS (
    -- Anchor: Root directories
    SELECT 
        id,
        name,
        parent_id,
        '/' || name as full_path,
        0 as depth
    FROM files
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive: Subdirectories and files
    SELECT 
        f.id,
        f.name,
        f.parent_id,
        fp.full_path || '/' || f.name,
        fp.depth + 1
    FROM files f
    JOIN file_paths fp ON f.parent_id = fp.id
    WHERE fp.depth < 100
)
SELECT 
    full_path,
    depth
FROM file_paths
ORDER BY full_path;

-- Output:
-- /home
-- /home/user
-- /home/user/documents
-- /home/user/documents/report.pdf
```

### Date Series Generation

```sql
-- Generate all dates in a range
WITH RECURSIVE date_series AS (
    SELECT DATE '2024-01-01' as date
    
    UNION ALL
    
    SELECT date + INTERVAL '1 day'
    FROM date_series
    WHERE date < DATE '2024-12-31'
)
SELECT date, DAYNAME(date) as day_name
FROM date_series;

-- Useful for gap analysis, reports with zero values
```

### Shortest Path in Graph

```sql
-- Find shortest path between two nodes
WITH RECURSIVE paths AS (
    -- Anchor: Start node
    SELECT 
        node_id,
        target_node,
        1 as distance,
        ARRAY[node_id, target_node] as path
    FROM edges
    WHERE node_id = 1  -- Start
    
    UNION
    
    -- Recursive: Extend paths
    SELECT 
        e.node_id,
        e.target_node,
        p.distance + 1,
        p.path || e.target_node
    FROM edges e
    JOIN paths p ON e.node_id = p.target_node
    WHERE NOT (e.target_node = ANY(p.path))  -- No cycles
      AND p.distance < 10  -- Max path length
)
SELECT *
FROM paths
WHERE target_node = 10  -- End
ORDER BY distance
LIMIT 1;  -- Shortest path
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Amazon - Product Category Navigation

```sql
-- Build breadcrumb navigation for product
WITH RECURSIVE breadcrumb AS (
    SELECT 
        id,
        name,
        parent_id,
        name as path,
        0 as level
    FROM categories
    WHERE id = 456  -- Current category: "Smartphones"
    
    UNION ALL
    
    SELECT 
        c.id,
        c.name,
        c.parent_id,
        c.name || ' > ' || b.path,
        b.level + 1
    FROM categories c
    JOIN breadcrumb b ON c.id = b.parent_id
)
SELECT path
FROM breadcrumb
ORDER BY level DESC
LIMIT 1;

-- Output: "Electronics > Mobile > Smartphones"
```

### Case Study 2: LinkedIn - Connection Recommendations

```sql
-- Find 2nd-degree connections for recommendations
WITH RECURSIVE connections AS (
    -- 1st degree: Direct connections
    SELECT 
        connection_id as user_id,
        1 as degree
    FROM connections
    WHERE user_id = 12345
    
    UNION
    
    -- 2nd degree: Connections of connections
    SELECT 
        c.connection_id,
        co.degree + 1
    FROM connections c
    JOIN connections co ON c.user_id = co.user_id
    WHERE co.degree = 1
      AND c.connection_id != 12345  -- Not self
)
SELECT 
    u.name,
    u.title,
    u.company,
    COUNT(*) as mutual_connections
FROM connections c
JOIN users u ON c.user_id = u.id
WHERE c.degree = 2
GROUP BY u.id, u.name, u.title, u.company
HAVING COUNT(*) >= 3  -- At least 3 mutual connections
ORDER BY mutual_connections DESC
LIMIT 10;
```

### Case Study 3: Manufacturing - Supply Chain Analysis

```sql
-- Identify all suppliers needed for product
WITH RECURSIVE supply_chain AS (
    SELECT 
        component_id,
        supplier_id,
        quantity_needed,
        1 as level
    FROM product_bom
    WHERE product_id = 789  -- Final product
    
    UNION ALL
    
    SELECT 
        pb.component_id,
        pb.supplier_id,
        sc.quantity_needed * pb.quantity_needed,
        sc.level + 1
    FROM product_bom pb
    JOIN supply_chain sc ON pb.product_id = sc.component_id
    WHERE sc.level < 15
)
SELECT 
    s.supplier_name,
    c.component_name,
    SUM(sc.quantity_needed) as total_quantity_needed
FROM supply_chain sc
JOIN suppliers s ON sc.supplier_id = s.id
JOIN components c ON sc.component_id = c.id
GROUP BY s.supplier_name, c.component_name
ORDER BY s.supplier_name, total_quantity_needed DESC;
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What is a recursive CTE and when would you use it?

**Answer:** A recursive CTE is a CTE that references itself, used for traversing hierarchical or graph-like data.

**Structure:**
1. **Anchor member**: Starting point (base case)
2. **Recursive member**: Self-reference with join condition
3. **Termination**: WHERE clause to stop recursion

**Use cases:**
- Org charts (employee-manager hierarchies)
- Category trees (parent-child relationships)
- Bill of materials (product components)
- Social networks (friend connections)
- File systems (directory traversal)

```sql
WITH RECURSIVE cte AS (
    SELECT ... WHERE start_condition  -- Anchor
    UNION ALL
    SELECT ... FROM cte WHERE stop_condition  -- Recursive
)
SELECT * FROM cte;
```

---

### Q2: How do you prevent infinite loops in recursive queries?

**Answer:**

**1. Add depth limit:**
```sql
WHERE depth < 100  -- Stop at 100 levels
```

**2. Track visited nodes:**
```sql
WHERE NOT (node_id = ANY(path_array))  -- Prevent revisiting
```

**3. Use database limits:**
```sql
SET max_recursion_depth = 100;  -- Session limit
```

**4. Statement timeout:**
```sql
SET statement_timeout = '30s';  -- Kill after 30 seconds
```

---

### Q3: What's the difference between UNION and UNION ALL in recursive CTEs?

**Answer:**

- **UNION ALL**: Standard choice, keeps duplicates, faster
- **UNION**: Removes duplicates, can terminate recursion early (when no new rows)

```sql
-- UNION ALL: Continues until WHERE condition fails
WITH RECURSIVE cte AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM cte WHERE n < 10
)
SELECT * FROM cte;  -- Returns 1-10

-- UNION: May terminate earlier (no new unique rows)
WITH RECURSIVE cte AS (
    SELECT 1 as n
    UNION  -- Removes duplicates
    SELECT 1 FROM cte LIMIT 10
)
SELECT * FROM cte;  -- Returns only 1 (no new rows after first iteration)
```

**Use UNION ALL** unless you specifically need deduplication.

---

### Q4: How do you optimize recursive queries on large datasets?

**Answer:**

**1. Add index on parent-child relationship:**
```sql
CREATE INDEX idx_parent ON nodes(parent_id);
```

**2. Limit recursion depth:**
```sql
WHERE depth < 20  -- Don't traverse entire tree
```

**3. Materialize results:**
```sql
CREATE TABLE hierarchy_cache AS
WITH RECURSIVE ... ;
-- Query cache instead of re-computing
```

**4. Use closure tables for frequent queries:**
```sql
-- Pre-compute all ancestor-descendant pairs
CREATE TABLE category_closure (
    ancestor_id INT,
    descendant_id INT,
    depth INT
);
-- O(1) lookups instead of recursion
```

**5. Batch processing for updates:**
```sql
-- Instead of recursive query per request,
-- batch-rebuild hierarchy nightly
```

---

### Q5: Can you write a recursive query to find the Nth level manager?

**Answer:**

```sql
-- Find the manager 2 levels up from employee 123
WITH RECURSIVE management_chain AS (
    -- Anchor: Start with employee
    SELECT 
        id,
        name,
        manager_id,
        0 as levels_up
    FROM employees
    WHERE id = 123
    
    UNION ALL
    
    -- Recursive: Go up management chain
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        mc.levels_up + 1
    FROM employees e
    JOIN management_chain mc ON e.id = mc.manager_id
    WHERE mc.levels_up < 2  -- Stop at 2 levels
)
SELECT name, levels_up
FROM management_chain
WHERE levels_up = 2;  -- Get manager 2 levels up
```

---

## 🧩 11. Design Patterns

### Pattern 1: Hierarchical Rollup

```sql
-- Calculate totals at each level of hierarchy
WITH RECURSIVE category_totals AS (
    -- Leaf nodes
    SELECT 
        id,
        parent_id,
        name,
        (SELECT SUM(price) FROM products WHERE category_id = c.id) as total_value
    FROM categories c
    WHERE id NOT IN (SELECT DISTINCT parent_id FROM categories WHERE parent_id IS NOT NULL)
    
    UNION ALL
    
    -- Parent nodes (sum of children)
    SELECT 
        c.id,
        c.parent_id,
        c.name,
        (SELECT SUM(ct.total_value) 
         FROM category_totals ct 
         WHERE ct.parent_id = c.id)
    FROM categories c
    WHERE c.id IN (SELECT DISTINCT parent_id FROM categories WHERE parent_id IS NOT NULL)
)
SELECT * FROM category_totals;
```

### Pattern 2: Path Enumeration

```sql
-- Store full path for each node
WITH RECURSIVE paths AS (
    SELECT 
        id,
        name,
        parent_id,
        ARRAY[id] as path_ids,
        name::TEXT as path_names
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT 
        c.id,
        c.name,
        c.parent_id,
        p.path_ids || c.id,
        p.path_names || ' > ' || c.name
    FROM categories c
    JOIN paths p ON c.parent_id = p.id
)
SELECT id, path_names FROM paths;
```

### Pattern 3: Cycle Detection

```sql
-- Detect cycles in graph
WITH RECURSIVE cycle_check AS (
    SELECT 
        node_id,
        target_node,
        ARRAY[node_id] as visited,
        FALSE as has_cycle
    FROM edges
    WHERE node_id = 1
    
    UNION ALL
    
    SELECT 
        e.node_id,
        e.target_node,
        cc.visited || e.node_id,
        (e.node_id = ANY(cc.visited)) as has_cycle
    FROM edges e
    JOIN cycle_check cc ON e.node_id = cc.target_node
    WHERE NOT cc.has_cycle
      AND array_length(cc.visited, 1) < 100
)
SELECT DISTINCT has_cycle FROM cycle_check WHERE has_cycle = TRUE;
```

---

## 📚 Summary

### Key Takeaways

1. **Recursive CTEs** traverse hierarchical/graph data
2. **Structure**: Anchor (base) + Recursive (self-reference) + Termination
3. **Always use UNION ALL** (unless you need deduplication)
4. **Termination required**: Prevent infinite loops with depth limits
5. **Cycle detection**: Track visited nodes with arrays
6. **Index parent_id**: Critical for performance
7. **Materialize results**: For frequent queries, pre-compute hierarchies
8. **Closure tables**: For complex queries, store all ancestor-descendant pairs
9. **Max depth**: Always include `WHERE depth < N`
10. **Use cases**: Org charts, category trees, BOM, social networks, file systems

**Performance Tips:**
- Index: `CREATE INDEX idx_parent ON table(parent_id)`
- Limit depth: `WHERE level < 20`
- Materialize: `CREATE TABLE hierarchy_cache AS WITH RECURSIVE ...`
- Closure table: Pre-compute all relationships

---

**Next:** [03_Pivoting_Unpivoting.md](03_Pivoting_Unpivoting.md) - Transform rows to columns  
**Related:** [../03_SQL_Core/06_CTEs.md](../03_SQL_Core/06_CTEs.md) - Common Table Expressions basics

---

*Last Updated: February 20, 2026*
