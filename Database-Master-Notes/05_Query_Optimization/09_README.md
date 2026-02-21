# Query Optimization - Getting the Best Execution Plans

This folder contains comprehensive guides on query optimization techniques, from understanding execution plans to preventing common performance pitfalls like the N+1 problem.

---

## 📚 Contents

### Core Concepts

1. **[Execution Plans](01_Execution_Plans.md)** (~30,000 words)
   - Reading and interpreting EXPLAIN output
   - Scan types: Sequential, Index, Bitmap, Index Only
   - Join algorithms: Nested Loop, Hash Join, Merge Join
   - Cost model internals (seq_page_cost, random_page_cost)
   - Comparing estimated vs actual rows
   - Real-world debugging workflows

2. **[Query Rewriting](02_Query_Rewriting.md)** (~28,000 words)
   - 7 rewriting patterns for optimization
   - Correlated subquery → JOIN conversion (O(N²) → O(N))
   - OR conditions → UNION ALL (enabling index usage)
   - EXISTS vs IN vs JOIN decision matrix
   - NOT IN NULL safety (critical gotcha)
   - Predicate pushdown strategies

3. **[Cost Estimation](03_Cost_Estimation.md)** (~29,000 words)
   - How optimizer calculates query costs
   - Cardinality estimation formulas
   - Selectivity calculations for equality, ranges, AND/OR
   - Hardware tuning: random_page_cost for SSD vs HDD
   - Histogram-based estimation for skewed data
   - Extended statistics for correlated columns

4. **[Join Algorithms](04_Join_Algorithms.md)** (~32,000 words)
   - Nested Loop Join: O(N × log M) with index, best for small outer
   - Hash Join: O(N + M) linear, best for equi-joins
   - Merge Join: O(N + M) for sorted data, best for range joins
   - Hash table internals: Collision handling, Grace Hash Join
   - work_mem tuning to prevent disk spills
   - Real-world 5000× speedup examples

### Optimization Techniques

5. **[Index Selection](05_Index_Selection.md)** (~29,000 words)
   - How optimizer chooses which index to use
   - Composite index column ordering (selectivity vs query patterns)
   - Covering indexes to eliminate heap access
   - Partial indexes for skewed data
   - Expression indexes for function-based queries
   - Detecting and dropping redundant indexes

6. **[Query Hints](06_Query_Hints.md)** (~35,000 words)
   - When and how to override optimizer decisions
   - Index hints, join algorithm hints, join order hints
   - Syntax across databases (PostgreSQL, MySQL, SQL Server, Oracle)
   - Emergency production fixes with hints
   - Hint interaction with plan caching
   - When to avoid hints (fix root cause instead)

7. **[Statistics Management](07_Statistics_Management.md)** (~38,000 words)
   - How optimizer statistics work (n_distinct, MCV, histograms)
   - Running ANALYZE after bulk operations
   - Tuning statistics target for skewed columns (100 → 1000 buckets)
   - Auto-ANALYZE configuration (scale_factor tuning)
   - Extended statistics for correlated columns (PostgreSQL 10+)
   - Detecting and fixing stale statistics

### Application-Level Optimization

8. **[N+1 Query Problem](08_N_Plus_One_Problem.md)** (~40,000 words)
   - Understanding the N+1 pattern (1 query + N queries)
   - ORM eager loading: select_related vs prefetch_related
   - Aggregation in database vs iteration in Python
   - DataLoader pattern for GraphQL
   - Detecting N+1 with APM, query counters, slow query logs
   - Real-world examples: 1001 queries → 3 queries

---

## 🎯 Learning Paths

### Path 1: Junior → Mid-Level (Debugging Slow Queries)

**Goal:** Diagnose and fix slow queries using execution plans

**Recommended Order:**
1. Start: **[01_Execution_Plans](01_Execution_Plans.md)**
   - Learn to read EXPLAIN ANALYZE output
   - Understand scan types (when index vs seq scan)
   - Practice: Run EXPLAIN on 10 queries, identify bottlenecks

2. Next: **[02_Query_Rewriting](02_Query_Rewriting.md)**
   - Learn common patterns: subquery → JOIN, OR → UNION ALL
   - Practice: Rewrite 5 slow queries from production

3. Then: **[05_Index_Selection](05_Index_Selection.md)**
   - Understand why optimizer chooses (or ignores) indexes
   - Practice: Create composite index for multi-column query

**Milestone Project:**
- Take 3 slow queries from your codebase
- Run EXPLAIN ANALYZE, identify problem (seq scan, nested loop, missing index)
- Apply fix (add index, rewrite query)
- Measure improvement (before/after timing)

**Success Criteria:**
- Can read execution plans and identify: scan type, join algorithm, estimated vs actual rows
- Can create appropriate indexes (single-column, composite, partial)
- Can rewrite subqueries efficiently

---

### Path 2: Mid-Level → Senior (Cost-Based Optimization)

**Goal:** Understand how optimizer makes decisions, tune for your workload

**Recommended Order:**
1. Start: **[03_Cost_Estimation](03_Cost_Estimation.md)**
   - Learn cost formulas: seq_page_cost, random_page_cost
   - Understand selectivity calculations
   - Practice: Tune random_page_cost for your hardware (SSD vs HDD)

2. Next: **[04_Join_Algorithms](04_Join_Algorithms.md)**
   - Master when to use nested loop vs hash vs merge join
   - Understand work_mem impact on hash join performance
   - Practice: Optimize 3-table join with proper algorithm selection

3. Then: **[07_Statistics_Management](07_Statistics_Management.md)**
   - Learn to keep optimizer statistics accurate
   - Tune auto-ANALYZE for fast-growing tables
   - Create extended statistics for correlated columns

**Milestone Project:**
- Audit a production database:
  - Check for stale statistics (last_autoanalyze > 1 day)
  - Identify queries with 10× row estimate misses
  - Tune statistics target for skewed columns
  - Configure auto-ANALYZE scale_factor
  - Measure improvement in query plan quality

**Success Criteria:**
- Can explain why optimizer chose specific plan (cost calculations)
- Can tune database parameters (random_page_cost, work_mem, statistics_target)
- Can identify and fix cardinality estimation errors

---

### Path 3: Senior → Staff (Production Optimization)

**Goal:** Handle emergency incidents, prevent issues proactively

**Recommended Order:**
1. Start: **[06_Query_Hints](06_Query_Hints.md)**
   - Learn emergency mitigation techniques
   - Understand when hints are appropriate (temporary fix)
   - Practice: Add hint to force better plan, document why

2. Next: **[08_N_Plus_One_Problem](08_N_Plus_One_Problem.md)**
   - Master ORM eager loading techniques
   - Implement query count monitoring
   - Practice: Fix N+1 in GraphQL API (DataLoader pattern)

3. Then: Review all chapters with focus on:
   - Real-world case studies
   - Interview questions (principal-level depth)
   - System design implications

**Milestone Project:**
- Implement comprehensive query monitoring:
  - Query count per request (alert if >50)
  - Slow query log analysis (detect N+1 patterns)
  - APM integration (track query time % of request time)
  - CI test: Assert max query count for critical endpoints
  - Dashboard: Track P50, P95, P99 query latency

**Success Criteria:**
- Can diagnose production incidents under pressure (Black Friday scenario)
- Can implement automated detection for N+1, missing indexes, stale stats
- Can make architecture decisions (when to denormalize, when to cache)

---

## 🔑 Key Concepts by Topic

### Execution Plan Analysis

**When to Use:**
- Every slow query investigation starts with EXPLAIN ANALYZE
- Debugging why query is slow (identify bottleneck node)
- Verifying optimization worked (compare before/after)

**Key Metrics:**
- **Estimated vs Actual Rows:** >10× difference = statistics problem
- **Node Time:** Which node takes most time? (focus optimization there)
- **Buffers:** High read count = I/O bottleneck

**Quick Wins:**
- Seq Scan on <1% of rows → Add index
- Nested Loop with 100K+ outer rows → Force Hash Join (or add index)
- Index Scan with low correlation → CLUSTER table by index

---

### Query Rewriting

**When to Use:**
- Cardinality estimates are right, but query structure is inefficient
- Complex subqueries (optimizer can't optimize)
- OR conditions preventing index usage

**Common Patterns:**
1. **Correlated subquery → JOIN:** 90× speedup (O(N²) → O(N))
2. **OR → UNION ALL:** Enable index on each branch (83× speedup)
3. **IN with large list → JOIN:** Avoid plan caching issues
4. **NOT IN → NOT EXISTS:** Handle NULLs correctly (critical!)

**Red Flags:**
- Subquery executed per row (EXPLAIN shows "SubPlanfilter")
- OR conditions in WHERE (prevents index usage)
- Function calls on indexed columns (LOWER(email) = ?)

---

### Cost Estimation & Statistics

**When to Use:**
- Execution plan shows large row estimate miss (100 estimated, 10K actual)
- Optimizer choosing wrong plan despite correct indexes
- After bulk data changes (ANALYZE immediately)

**Key Parameters:**
- **random_page_cost:** Tune for your hardware
  - SSD: 1.1-1.5
  - HDD: 4.0 (default)
  - NVMe: 1.05-1.1
- **default_statistics_target:** Histogram detail
  - Default: 100
  - Skewed data: 1000
  - Very skewed: 10,000
- **autovacuum_analyze_scale_factor:** How often to re-ANALYZE
  - Default: 0.1 (10% change triggers)
  - Fast-growing tables: 0.01 (1% change triggers)

**Fixes:**
- Stale stats → `ANALYZE table`
- Skewed column → `ALTER COLUMN SET STATISTICS 1000`
- Correlated columns → `CREATE STATISTICS ... (dependencies)`

---

### Index Strategy

**When to Use:**
- Query filters on column(s) without index
- Composite index exists but wrong column order
- Query slow despite having index (optimizer ignores it)

**Decision Matrix:**

| Query Pattern | Index Type |
|---------------|-----------|
| `WHERE col = ?` | Single-column B-Tree |
| `WHERE col1 = ? AND col2 = ?` | Composite (most selective first) |
| `WHERE col LIKE 'prefix%'` | B-Tree on col |
| `WHERE col > ? AND col < ?` | B-Tree (range) |
| `WHERE deleted_at IS NULL` (99%) | Partial index |
| `WHERE LOWER(col) = ?` | Expression index on LOWER(col) |
| `SELECT col1, col2 WHERE col3 = ?` | Covering index (col3) INCLUDE (col1, coltwo) |

**Anti-Patterns:**
- 10+ indexes on one table (write amplification)
- Index on low-selectivity column (status with 3 values)
- Redundant indexes (single-column + composite starting with same column)

---

### Join Optimization

**Algorithm Selection:**

| Scenario | Best Algorithm | Why |
|----------|---------------|-----|
| Small outer (<1K rows), indexed inner | Nested Loop | O(N × log M), fast with index |
| Both tables large (>10K), equi-join | Hash Join | O(N + M), linear |
| Pre-sorted data, range join | Merge Join | O(N + M), no hash overhead |
| Outer very small (<100), LIMIT | Nested Loop | Stops early |
| join inequality (>, <) | Nested Loop or Merge | Hash only works for = |

**Performance Tips:**
- **work_mem:** Increase if hash join shows Batches > 1 (disk spill)
- **Join order:** Filter small first, join with filtered result
- **Indexes:** Nested loop requires index on inner table's join column

---

### N+1 Prevention

**ORM Best Practices:**

```python
# ❌ N+1: 1 + N queries
for user in User.objects.all():
    print(user.profile.bio)  # N queries

# ✅ Eager load: 1 query
for user in User.objects.select_related('profile'):
    print(user.profile.bio)  # No additional queries
```

**Quick Reference:**

| Relationship | Eager Load Method | Queries |
|--------------|-------------------|---------|
| ForeignKey (author) | `.select_related('author')` | 1 (JOIN) |
| Reverse FK (posts) | `.prefetch_related('posts')` | 2 (IN query) |
| ManyToMany (tags) | `.prefetch_related('tags')` | 2 (JOIN + IN) |
| Nested (posts__comments) | `.prefetch_related('posts__comments')` | 3 |
| Aggregation (count) | `.annotate(count=Count('posts'))` | 1 (GROUP BY) |

**Detection:**
- Django Debug Toolbar shows duplicate queries
- APM alerts on high query count per request
- CI test: `assertNumQueries(expected_count)`

---

## 🚀 Quick Wins (Low-Hanging Fruit)

### 1. Add Missing Indexes (30 minutes)
```sql
-- Find seq scans on large tables:
SELECT
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / seq_scan AS avg_seq_read
FROM pg_stat_user_tables
WHERE seq_scan > 0
  AND seq_tup_read / seq_scan > 10000  -- Scans >10K rows per scan
ORDER BY seq_tup_read DESC;

-- For each table: Check common WHERE clauses
-- CREATE INDEX idx_tablename_column ON tablename(column);
```

**Expected Impact:** 10-100× speedup for filtered queries

---

### 2. Fix N+1 in Top Endpoints (1 hour)
```python
# Add query counter middleware (see 08_N_Plus_One_Problem.md)
# Check response headers: X-Query-Count

# For endpoints with >50 queries:
# - Add select_related() for ForeignKey
# - Add prefetch_related() for reverse FK/ManyToMany
# - Add .annotate() for counts/sums
```

**Expected Impact:** 50-200× reduction in query count

---

### 3. Tune random_page_cost for SSDs (5 minutes)
```sql
-- Default assumes HDD (4.0)
-- If you have SSDs:
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();

-- Test queries before/after, verify index usage improved
```

**Expected Impact:** Optimizer switches seq scan → index scan for 5-15% selectivity range

---

### 4. ANALYZE After Bulk Ops (Script it)
```sql
-- Add to ETL scripts:
BEGIN;
  COPY orders FROM '/data/orders.csv';  -- Load 10M rows
  ANALYZE orders;  -- Update statistics immediately
COMMIT;
```

**Expected Impact:** Prevents hours of wrong plans after data loads

---

### 5. Drop Unused Indexes (15 minutes)
```sql
-- Find indexes never used:
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (
      SELECT indexrelid FROM pg_constraint WHERE contype IN ('p', 'u')  -- Exclude PK/unique
  )
ORDER BY pg_relation_size(indexrelid) DESC;

-- DROP INDEX idx_never_used;
```

**Expected Impact:** Faster writes (less index maintenance), disk space savings

---

## 💡 Performance Debugging Workflow

```
1. Query slow (>1s)?
   ↓
2. Run: EXPLAIN (ANALYZE, BUFFERS) [query];
   ↓
3. Check estimated vs actual rows on each node
   ├─ >10× difference? → Statistics problem
   │   └─ Fix: ANALYZE table, increase statistics_target, extended stats
   │
   ├─ Seq Scan on <5% of table? → Missing index
   │   └─ Fix: CREATE INDEX
   │
   ├─ Nested Loop with large outer? → Wrong join algorithm
   │   └─ Fix: Increase work_mem, add index, or hint Hash Join
   │
   └─ Index Scan taking 80% of time? → Low correlation or too many rows
       └─ Fix: CLUSTER table by index, or use seq scan
   ↓
4. Apply fix, re-test with EXPLAIN (ANALYZE, BUFFERS)
   ↓
5. Verify:
   ├─ Execution time improved (1s → 0.05s = 20× faster)
   ├─ Estimated rows now match actual
   └─ Correct scan/join type chosen
   ↓
6. Deploy, monitor in production (APM, slow query log)
```

---

## 📊 Benchmarking Results (from Real-World Cases)

| Optimization | Before | After | Speedup | Notes |
|--------------|--------|-------|---------|-------|
| Add composite index | 45s | 0.86s | **53×** | E-commerce dashboard, 3-table join |
| Fix N+1 (ORM eager loading) | 5s | 0.05s | **100×** | REST API, 1001 queries → 3 queries |
| Correlated subquery → JOIN | 45s | 0.5s | **90×** | User dashboard with order counts |
| OR → UNION ALL | 25s | 0.3s | **83×** | Product search with category OR name |
| Hash join hint (stale stats) | 45s | 0.8s | **56×** | Black Friday incident, emergency fix |
| Increase work_mem (hash join) | 3.5s | 0.45s | **8×** | Prevent hash join disk spill (Batches: 16 → 1) |
| Tune random_page_cost (SSD) | 25s | 0.5s | **50×** | Force index usage on SSD |
| Extended statistics (correlation) | 45m | 8.6s | **314×** | Multi-tenant query with skewed tenant sizes |
| ANALYZE after bulk load | Timeout | 2.3s | **>25×** | Data warehouse ETL, 50M row load |
| Drop 7 redundant indexes | 50ms | 35ms | **1.4×** | INSERT performance, write amplification |

**Key Takeaway:** Top 3 optimizations (index, N+1, query rewrite) each provide 50-100× speedup. Compound effect: 100× × 100× = 10,000× total potential improvement.

---

## 🎓 Interview Preparation

### Junior/Mid-Level Questions

1. **Explain the difference between seq scan and index scan. When does optimizer choose each?**
   - See: [01_Execution_Plans.md](01_Execution_Plans.md) "Why Optimizer Chooses Seq Scan"

2. **How would you debug a slow query?**
   - Workflow: EXPLAIN ANALYZE → Identify bottleneck → Fix → Verify
   - See: [01_Execution_Plans.md](01_Execution_Plans.md) "Real-World Debugging Workflow"

3. **What is the N+1 query problem? How do you fix it?**
   - See: [08_N_Plus_One_Problem.md](08_N_Plus_One_Problem.md) entire chapter

4. **When should you create a composite index vs two single-column indexes?**
   - See: [05_Index_Selection.md](05_Index_Selection.md) "Column Ordering in Composite Indexes"

### Senior/Staff-Level Questions

5. **Explain how the query optimizer estimates costs. What parameters affect it?**
   - See: [03_Cost_Estimation.md](03_Cost_Estimation.md) "Cost Model Internals"

6. **How do statistics become stale, and how does it impact query performance?**
   - See: [07_Statistics_Management.md](07_Statistics_Management.md) "Why Statistics Matter"

7. **When would you use query hints, and when would you avoid them?**
   - See: [06_Query_Hints.md](06_Query_Hints.md) Q1 in Interview Questions

8. **Explain nested loop, hash join, and merge join. When is each optimal?**
   - See: [04_Join_Algorithms.md](04_Join_Algorithms.md) "Algorithm Selection Criteria"

### Principal-Level Questions

9. **Your API suddenly becomes 100× slower during a traffic spike. How do you diagnose and fix it?**
   - Scenario in: [06_Query_Hints.md](06_Query_Hints.md) "Black Friday Emergency Fix"

10. **How would you design a query monitoring system to prevent N+1 and missing index issues?**
    - See: [08_N_Plus_One_Problem.md](08_N_Plus_One_Problem.md) "Detection in Production"

11. **Explain extended statistics and when you'd create them.**
    - See: [07_Statistics_Management.md](07_Statistics_Management.md) Interview Q2

12. **How do query hints interact with query plan caching? What are the risks?**
    - See: [06_Query_Hints.md](06_Query_Hints.md) Interview Q2

---

## 🛠️ Tools & Commands

### PostgreSQL

**Query Analysis:**
```sql
-- Explain with all details
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS, TIMING) [query];

-- Find slow queries
SELECT query, calls, mean_exec_time, max_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- >1s
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Check statistics freshness
SELECT schemaname, tablename, last_analyze, last_autoanalyze, n_mod_since_analyze
FROM pg_stat_user_tables
WHERE n_mod_since_analyze > 0.2 * n_live_tup;  -- >20% stale
```

**Optimization:**
```sql
-- Update statistics
ANALYZE table_name;
ANALYZE;  -- All tables

-- Tune parameters
SET work_mem = '256MB';  -- Session-level
ALTER SYSTEM SET random_page_cost = 1.1;  -- System-wide (needs reload)

-- Increase statistics detail
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;

-- Extended statistics
CREATE STATISTICS orders_stats (dependencies, ndistinct, mcv)
ON tenant_id, created_at FROM orders;
```

### MySQL

**Query Analysis:**
```sql
-- Explain
EXPLAIN [query];
EXPLAIN FORMAT=JSON [query];

-- Profile query
SET profiling = 1;
[query];
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;
```

**Optimization:**
```sql
-- Update statistics
ANALYZE TABLE table_name;

-- Optimizer hints
SELECT /*+ HASH_JOIN(t1, t2) INDEX(t1 idx_name) */ ...;
```

### SQL Server

**Query Analysis:**
```sql
-- Execution plan
SET SHOWPLAN_ALL ON;
[query];
SET SHOWPLAN_ALL OFF;

-- Actual execution plan
SET STATISTICS TIME ON;
SET STATISTICS_IO ON;
[query];

-- Find expensive queries
SELECT TOP 20
    qs.execution_count,
    qs.total_elapsed_time / 1000000 AS total_elapsed_time_sec,
    qt.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_elapsed_time DESC;
```

**Optimization:**
```sql
-- Update statistics
UPDATE STATISTICS table_name;

-- Query hints
SELECT * FROM orders WITH (INDEX(idx_orders_created))
OPTION (HASH JOIN, MAXDOP 4);
```

---

## 📖 Additional Resources

### Books
- *SQL Performance Explained* by Markus Winand (index design, execution plans)
- *High Performance MySQL* by Baron Schwartz (query optimization deep dive)
- *PostgreSQL Query Optimization* by Henrietta Dombrovskaya (PostgreSQL-specific)

### Online Tools
- **pgMustard** (https://www.pgmustard.com/) - Execution plan visualizer
- **explain.depesz.com** - PostgreSQL EXPLAIN analyzer
- **EverSQL** - Automatic query optimization suggestions

### Extensions/Plugins
- **pg_stat_statements** (PostgreSQL) - Query performance tracking
- **pg_hint_plan** (PostgreSQL) - Query hint support
- **Django Debug Toolbar** - ORM query visualization
- **New Relic / Datadog APM** - Production query monitoring

---

## ✅ Self-Assessment Checklist

Mark your proficiency level for each topic:

### Execution Plan Analysis
- [ ] Can read EXPLAIN output (scan types, join algorithms)
- [ ] Can identify cost bottlenecks (which node is slow)
- [ ] Can spot cardinality mismatches (estimated vs actual rows)
- [ ] Can compare multiple execution plans (which is better)

### Query Optimization
- [ ] Can add appropriate indexes (single, composite, partial)
- [ ] Can rewrite subqueries efficiently (correlated → JOIN)
- [ ] Can choose optimal join algorithm (nested loop vs hash vs merge)
- [ ] Can tune database parameters (work_mem, random_page_cost)

### Statistics Management
- [ ] Can run ANALYZE at appropriate times (after bulk ops)
- [ ] Can tune statistics target for skewed data
- [ ] Can create extended statistics for correlated columns
- [ ] Can diagnose and fix stale statistics issues

### N+1 Prevention
- [ ] Can detect N+1 in ORM code (loops with relationship access)
- [ ] Can apply eager loading (select_related / prefetch_related)
- [ ] Can use aggregations instead of iteration (annotate + Count/Sum)
- [ ] Can implement query count monitoring (CI tests, APM)

### Production Debugging
- [ ] Can debug slow query under pressure (incident response)
- [ ] Can use query hints for emergency mitigation
- [ ] Can set up monitoring/alerting (slow queries, high query count)
- [ ] Can perform performance audits (find lowest-hanging fruit)

**Scoring:**
- **0-5 checked:** Start with Learning Path 1 (Junior → Mid-Level)
- **6-12 checked:** Continue with Learning Path 2 (Mid-Level → Senior)
- **13-18 checked:** Focus on Learning Path 3 (Senior → Staff)
- **19-20 checked:** You're ready to mentor others! Consider writing your own case studies.

---

## 🎯 Next Steps

After mastering this folder, proceed to:

1. **[06_Indexing_Performance](../06_Indexing_Performance/)** - Deep dive into index internals (B-Tree, hash, bitmap)
2. **[07_Transactions_Concurrency](../07_Transactions_Concurrency/)** - How transactions impact query performance
3. **[33_System_Design_Frontend](../33_System_Design_Frontend/)** - Apply query optimization to system design interviews

---

## 💬 Contributing

Found an error or have a suggestion? This is a personal knowledge base, but feedback welcome:
- Raise issue in repo
- Comment on specific errors/improvements
- Share your own query optimization war stories!

---

**Last Updated:** 2024-02-21  
**Total Word Count:** ~261,000 words across 9 files  
**Estimated Reading Time:** 20-25 hours for complete folder

*"Premature optimization is the root of all evil, but knowing how to optimize is the root of all senior/staff promotions."* — Adapted from Donald Knuth

---
