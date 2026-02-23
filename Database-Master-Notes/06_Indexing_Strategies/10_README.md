# Indexing Strategies - Section Overview

## 📚 Module Overview

This section covers **index internals, strategies, and performance optimization**—essential knowledge for building production databases that scale. You'll learn when to use each index type, how write amplification affects performance, and strategies used by companies like GitHub, Stripe, and Uber.

**Target Audience:** Mid-level → Senior Database Engineers, Backend Engineers, SREs

**Estimated Learning Time:** 15-20 hours (deep dive)

---

## 📖 Files in This Section

### 1. [01_BTree_Indexes.md](01_BTree_Indexes.md)
**B-Tree Index Internals** (~50,000 words)

The most common index structure in databases. Understand how B+Trees work, page splits, fill factor, and latch coupling for concurrent access.

**Key Topics:**
- B+Tree structure (leaf pages linked for range scans)
- Insert algorithm and page splits (O(log N))
- Range queries vs point lookups
- Composite indexes and index-only scans
- Real-world: E-commerce product search (5000× speedup)

**When to read first:** Start here if you're new to indexing

---

### 2. [02_Hash_Indexes.md](02_Hash_Indexes.md)
**Hash Index Performance** (~45,000 words)

O(1) equality lookups, but no range scans. Learn when hash indexes beat B-Trees and collision resolution strategies.

**Key Topics:**
- Hash functions (MurmurHash, FNV-1a)
- Collision resolution (chaining vs open addressing)
- Load factor and dynamic resizing
- When hash beats B-Tree (equality-only queries)
- Real-world: Discord message ID lookups, Memcached internals

**Best for:** High-frequency point lookups (no range queries)

---

### 3. [03_Bitmap_Indexes.md](03_Bitmap_Indexes.md)
**Bitmap Indexes for Low-Cardinality** (~45,000 words)

Efficient for columns with few distinct values (e.g., status, gender, region). Uses bitwise operations for fast AND/OR queries.

**Key Topics:**
- Low-cardinality optimization (2-20 distinct values)
- Bitwise AND/OR/NOT operations (1 million rows in microseconds)
- RLE compression (90% space savings)
- Why bitmap indexes fail in OLTP (locking issues)
- Real-world: Netflix content recommendations, Uber trip analytics

**Best for:** Data warehouse queries, OLAP systems

---

### 4. [04_Covering_Indexes.md](04_Covering_Indexes.md)
**Covering Indexes for Index-Only Scans**

Eliminate table lookups by including all query columns in the index. Dramatic speedups for SELECT-heavy workloads.

**Key Topics:**
- Index-only scans (no heap lookup)
- INCLUDE columns (PostgreSQL 11+, SQL Server)
- When covering indexes are worth it
- Balancing write cost vs read benefit

**Best for:** Queries with 3-10 columns that always appear together

---

### 5. [05_Partial_Indexes.md](05_Partial_Indexes.md)
**Partial Indexes with WHERE Clauses**

Index only relevant rows (e.g., WHERE status = 'pending'). Reduces index size 90%+ and write amplification.

**Key Topics:**
- Filtered indexes (PostgreSQL, SQL Server)
- Reducing index size and write overhead
- Functional indexes (indexes on expressions)
- Real-world: E-commerce (index only active orders)

**Best for:** Queries that filter on specific values (pending, active, unprocessed)

---

### 6. [06_Composite_Indexes.md](06_Composite_Indexes.md)
**Multi-Column Composite Indexes**

Order matters! Learn how LEFT-to-RIGHT matching affects query performance.

**Key Topics:**
- Column ordering strategies (most selective first)
- Prefix matching rules (can use first 2 of 3 columns)
- Skip-scan optimization (Oracle, SQL Server)
- Redundant index detection

**Best for:** Queries with multiple WHERE conditions

---

### 7. [07_Full_Text_Indexes.md](07_Full_Text_Indexes.md)
**Full-Text Search Indexes**

GIN (Generalized Inverted Index) for text search. Powers features like "Search products containing 'wireless bluetooth'".

**Key Topics:**
- Inverted indexes (word → document IDs)
- Tokenization and stemming ("running" → "run")
- Ranking algorithms (TF-IDF, BM25)
- PostgreSQL GIN vs GiST indexes
- Real-world: E-commerce product search, documentation sites

**Best for:** Text search, LIKE '%pattern%' replacement

---

### 8. [08_Index_Maintenance.md](08_Index_Maintenance.md)
**Index Maintenance and Health Monitoring** (~40,000 words)

Indexes degrade over time via fragmentation and bloat. Learn GitHub's and Stripe's maintenance strategies.

**Key Topics:**
- Fragmentation measurement (logical vs physical order)
- REINDEX vs REBUILD vs REORGANIZE
- Maintenance schedules (daily/weekly/monthly)
- Zero-downtime strategies (REINDEX CONCURRENTLY)
- Real-world: GitHub (300+ tables, 50TB, 7-day staggered maintenance)

**Best for:** Production DBAs, Senior Engineers maintaining large databases

---

### 9. [09_Write_Amplification.md](09_Write_Amplification.md)
**Understanding Write Amplification Costs** (~38,000 words)

Every index multiplies write cost. 20 indexes = 21× write amplification! Learn when to avoid indexes.

**Key Topics:**
- Write amplification formula (WAF = physical writes / logical writes)
- B-Tree vs LSM-Tree trade-offs
- SSD endurance and lifetime impact
- Strategies: partial indexes, batching, bulk load
- Real-world: Twitter timeline ingestion, Uber trip state machine

**Best for:** Write-heavy workloads (IoT, logging, high-frequency trading)

---

### Decision Matrix: Which Index Type?

Use this table to quickly decide which index strategy fits your query pattern:

| Query Pattern | Cardinality | Workload | Best Index Type | WAF (Write Cost) | Example Use Case |
|---------------|-------------|----------|-----------------|------------------|------------------|
| **Equality (=)** | Any | OLTP | **B-Tree** | 5-10× | `WHERE user_id = 12345` |
| **Equality (=)** | Any | In-memory | **Hash** | 2-5× | Redis/Memcached key lookup |
| **Range (<, >, BETWEEN)** | Any | OLTP | **B-Tree** | 5-10× | `WHERE created_at > '2024-01-01'` |
| **Multiple =** | Low (2-20) | OLAP | **Bitmap** | 3-8× | `WHERE gender='F' AND region='US'` |
| **Text search (LIKE '%x%')** | High | Mixed | **Full-Text (GIN)** | 10-20× | `WHERE description ILIKE '%wireless%'` |
| **Specific subset** | Any | OLTP | **Partial** | 2-5× (subset only) | `WHERE status='pending'` (5% of data) |
| **Multi-column** | Any | OLTP | **Composite** | 5-10× | `WHERE (user_id, created_at)` |
| **Cover all cols** | Low | Read-heavy | **Covering** | 8-15× (wider index) | `SELECT name, email WHERE user_id=x` |

**WAF (Write Amplification Factor):** Lower = better for write-heavy workloads

---

## 🎯 Learning Paths

### Path 1: Junior → Mid-Level (Fundamentals)
**Goal:** Understand B-Tree basics and when to add indexes

1. **Start:** [01_BTree_Indexes.md](01_BTree_Indexes.md) - Learn B+Tree structure
2. **Then:** [06_Composite_Indexes.md](06_Composite_Indexes.md) - Multi-column indexes
3. **Then:** [08_Index_Maintenance.md](08_Index_Maintenance.md) - Fragmentation basics
4. **Exercise:** Create indexes for a sample e-commerce schema

**Time commitment:** 6-8 hours

**Outcome:** Can design basic index strategies and explain index-only scans

---

### Path 2: Mid-Level → Senior (Performance Optimization)
**Goal:** Optimize queries and reduce write amplification

1. **Start:** [04_Covering_Indexes.md](04_Covering_Indexes.md) - Index-only scans
2. **Then:** [05_Partial_Indexes.md](05_Partial_Indexes.md) - Filtered indexes
3. **Then:** [09_Write_Amplification.md](09_Write_Amplification.md) - Write costs
4. **Then:** [08_Index_Maintenance.md](08_Index_Maintenance.md) - Production maintenance
5. **Exercise:** Audit indexes on production table, drop 50% of unused indexes

**Time commitment:** 10-12 hours

**Outcome:** Can reduce query latency 10× and write amplification 50%

---

### Path 3: Senior → Staff (Architecture & Scale)
**Goal:** Design indexing strategies for 100M+ row tables

1. **Start:** [09_Write_Amplification.md](09_Write_Amplification.md) - LSM vs B-Tree
2. **Then:** [03_Bitmap_Indexes.md](03_Bitmap_Indexes.md) - OLAP optimization
3. **Then:** [08_Index_Maintenance.md](08_Index_Maintenance.md) - Zero-downtime at scale
4. **Then:** [02_Hash_Indexes.md](02_Hash_Indexes.md) - When to use hash
5. **Study:** Real-world cases (GitHub 50TB, Stripe 99.99% uptime, Twitter fanout)

**Time commitment:** 15-20 hours

**Outcome:** Can architect indexing for petabyte-scale databases

---

## ⚡ Quick Wins: Optimize in 1 Hour

**1. Drop Unused Indexes (30 minutes)**
```sql
-- PostgreSQL: Find indexes with 0 scans
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE '%pkey%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Drop unused indexes
DROP INDEX idx_users_last_name;  -- 0 scans, 500MB size
DROP INDEX idx_orders_notes;     -- 0 scans, 1.2GB size
-- Result: -50% write amplification, +20% INSERT throughput
```

**2. Create Partial Index (15 minutes)**
```sql
-- Before: Full index on status (100M rows)
CREATE INDEX idx_orders_status ON orders(status);

-- After: Partial index (only 5M pending/cancelled rows)
DROP INDEX idx_orders_status;
CREATE INDEX idx_orders_pending ON orders(status)
WHERE status IN ('pending', 'cancelled');

-- Result: -95% index size, -90% maintenance time, query still fast
```

**3. Replace Single Indexes with Composite (15 minutes)**
```sql
-- Before: Two separate indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created ON users(created_at);

-- After: One composite index (covers both queries)
DROP INDEX idx_users_email;
DROP INDEX idx_users_created;
CREATE INDEX idx_users_email_created ON users(email, created_at);

-- Queries still work:
-- WHERE email = 'x'               → Uses idx_users_email_created (leftmost prefix)
-- WHERE email = 'x' AND created_at > 'y'  → Uses idx_users_email_created (both columns)

-- Result: -50% index writes, -50% storage
```

---

## 🔧 Index Health Checklist

Run this monthly on production databases:

### 1. Unused Indexes (Drop candidates)
```sql
-- Find indexes with < 100 scans in 30 days
SELECT * FROM pg_stat_user_indexes
WHERE idx_scan < 100
ORDER BY pg_relation_size(indexrelid) DESC;
```

### 2. Duplicate/Redundant Indexes
```sql
-- Find indexes where one covers another
-- (idx_users_email_created covers idx_users_email)
SELECT * FROM pg_indexes
WHERE tablename = 'users'
ORDER BY indexname;
```

### 3. Fragmentation (Rebuild candidates)
```sql
-- PostgreSQL: Check bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_size,
    ROUND(100 * (pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) /
        NULLIF(pg_total_relation_size(schemaname||'.'||tablename), 0), 2) AS index_ratio
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- If index_ratio > 80%: Indexes are 4× larger than table (likely fragmented)
```

### 4. Write Amplification (Performance risk)
```sql
-- Count indexes per table
SELECT
    tablename,
    COUNT(*) AS num_indexes
FROM pg_indexes
WHERE schemaname = 'public'
GROUP BY tablename
HAVING COUNT(*) > 10
ORDER BY COUNT(*) DESC;

-- Red flag: > 15 indexes = high write amplification
```

---

## 🏆 Real-World Success Metrics

### E-Commerce Platform (150M products)
- **Before:** 25 indexes per table, query time 2s, INSERT 50/sec
- **After:** 8 indexes (dropped 17 unused), partial indexes on status
- **Result:** Query time 200ms (10× faster), INSERT 200/sec (4× faster)

### IoT Sensor Platform (500K writes/sec)
- **Before:** PostgreSQL B-Tree, WAF 40×, SSD saturated
- **After:** Migrated to Cassandra LSM-Tree, WAF 5×
- **Result:** Write throughput 2.5M/sec (5× faster), SSD lifespan 10× longer

### SaaS Multi-Tenant Database
- **Before:** Composite indexes wrong column order, 95% queries seq scan
- **After:** Reordered composite indexes (tenant_id first), added covering indexes
- **Result:** Query latency 5s → 50ms (100× faster), -80% CPU usage

---

## 🔗 Related Sections

**Previous:** [05_Query_Optimization](../05_Query_Optimization/) - Query planning and planner statistics

**Next:** [07_Transactions_Concurrency](../07_Transactions_Concurrency/) - How transactions affect index updates

**See Also:**
- [04_Advanced_SQL/06_Indexing_Strategies.md](../04_Advanced_SQL/06_Indexing_Strategies.md) - SQL syntax for creating indexes
- [05_Query_Optimization/04_Index_Selection.md](../05_Query_Optimization/04_Index_Selection.md) - How the planner chooses indexes

---

## 🎓 Interview Preparation

**Common Questions from This Section:**

1. **"Explain when a B-Tree index causes a page split and how it affects write performance."**
   - Read: [01_BTree_Indexes.md](01_BTree_Indexes.md) § Internal Working

2. **"Why would you use a hash index over a B-Tree?"**
   - Read: [02_Hash_Indexes.md](02_Hash_Indexes.md) § Best Practices

3. **"How do you detect and fix index fragmentation in production?"**
   - Read: [08_Index_Maintenance.md](08_Index_Maintenance.md) § Interview Questions

4. **"Explain write amplification and how to reduce it."**
   - Read: [09_Write_Amplification.md](09_Write_Amplification.md) § Common Mistakes

5. **"Design indexes for a table with 100M rows, 10K writes/sec, and 50K reads/sec."**
   - Read: [09_Write_Amplification.md](09_Write_Amplification.md) + [04_Covering_Indexes.md](04_Covering_Indexes.md)

**Depth by level:**
- **Junior:** Explain B-Tree structure, when to add index
- **Mid-Level:** Design composite indexes, explain covering indexes
- **Senior:** Reduce write amplification 50%, zero-downtime maintenance
- **Staff:** Design indexing for 1B+ rows, trade-offs between B-Tree and LSM

---

## 📊 Indexing Strategy Decision Tree

```
START: Need to optimize query?
│
├─ Equality (=) only?
│  ├─ YES → In-memory database?
│  │        ├─ YES → Hash Index (O(1))
│  │        └─ NO → B-Tree Index (most databases)
│  └─ NO → Continue
│
├─ Range queries (>, <, BETWEEN)?
│  └─ YES → B-Tree Index (only option)
│
├─ Text search (LIKE '%pattern%')?
│  └─ YES → Full-Text Index (GIN/GiST)
│
├─ Low cardinality (2-20 values)?
│  ├─ YES → OLAP/Data Warehouse?
│  │        ├─ YES → Bitmap Index
│  │        └─ NO → B-Tree (OLTP doesn't support bitmap)
│  └─ NO → Continue
│
├─ Filter only subset (5-20% of rows)?
│  └─ YES → Partial Index (WHERE clause)
│
├─ Multiple columns in WHERE?
│  └─ YES → Composite Index (leftmost columns most selective)
│
├─ SELECT includes non-indexed columns (heap lookup)?
│  └─ YES → Consider Covering Index (include all SELECT columns)
│
└─ Default: Single-column B-Tree Index
```

---

## 🛠️ Tools & Commands

### PostgreSQL
```sql
-- List all indexes with size and usage
SELECT * FROM pg_stat_user_indexes;

-- Check index fragmentation (requires pgstattuple extension)
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstatindex('idx_name');

-- Rebuild index without blocking writes (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_name;
```

### SQL Server
```sql
-- Index fragmentation
SELECT
    OBJECT_NAME(ips.object_id) AS table_name,
    i.name AS index_name,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') AS ips
JOIN sys.indexes AS i
    ON ips.object_id = i.object_id
    AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- Rebuild index online (Enterprise Edition)
ALTER INDEX idx_name ON table_name REBUILD WITH (ONLINE = ON);
```

### MySQL
```sql
-- Index cardinality (estimate of distinct values)
SHOW INDEX FROM table_name;

-- Rebuild all indexes on table
OPTIMIZE TABLE table_name;
```

---

## 📚 Further Reading

**Books:**
- *Database Internals* by Alex Petrov - Chapter 4-5 (B-Tree and LSM)
- *Designing Data-Intensive Applications* by Martin Kleppmann - Chapter 3 (Storage and Retrieval)

**Papers:**
- "The Log-Structured Merge-Tree (LSM-Tree)" - O'Neil et al. (1996)
- "Modern B-Tree Techniques" - Graefe (2011)

**Blogs:**
- Use The Index, Luke! - https://use-the-index-luke.com/
- PostgreSQL Documentation - Indexes: https://www.postgresql.org/docs/current/indexes.html

**Videos:**
- "How Does a Database Index Work?" - Hussein Nasser (YouTube)
- "B-Trees and Database Indexes" - MIT OpenCourseWare

---

## ✅ Section Completion Checklist

- [ ] Understand B-Tree page splits and fill factor
- [ ] Know when hash index beats B-Tree
- [ ] Can explain bitmap index AND/OR operations
- [ ] Can design composite indexes with optimal column order
- [ ] Can calculate write amplification factor (WAF)
- [ ] Can detect and fix index fragmentation
- [ ] Have audited one production database and dropped 30% of indexes
- [ ] Understand partial indexes for filtered queries
- [ ] Know trade-offs between B-Tree and LSM-Tree
- [ ] Can explain covering indexes and INCLUDE columns
- [ ] Understand full-text search with GIN indexes
- [ ] Can implement zero-downtime index maintenance

**When you can check 10/12:** You're ready for Senior Database Engineer interviews
**When you can check 12/12:** You're ready for Staff+ level architectural decisions

---

## 🎯 Key Takeaways

1. **B-Tree is default** - Handles 95% of use cases (range + equality)
2. **Measure before adding** - Every index has a write cost
3. **Drop unused indexes** - idx_scan = 0 → DELETE
4. **Write amplification = N indexes → N× writes** - Keep index count low for write-heavy workloads
5. **Partial indexes** - Index only 5% of rows if queries filter on specific values
6. **Composite index order matters** - Most selective column first, leftmost-prefix matching
7. **Covering indexes eliminate heap lookups** - 10× speedup for SELECT with few columns
8. **Maintain regularly** - REINDEX monthly, drop unused quarterly
9. **LSM for writes, B-Tree for reads** - Choose based on workload (80/20 rule)
10. **Monitor fragmentation** - > 30% fragmentation → REBUILD

**Golden Rule:** "Index for your queries, not your schema. Measure read benefit vs write cost. When in doubt, drop the index—you can always add it back."

---

**Next Steps:**
1. Complete learning path matching your level (6-20 hours)
2. Run index health checklist on your production database
3. Implement one quick win (drop unused indexes)
4. Move to [07_Transactions_Concurrency](../07_Transactions_Concurrency/) - Learn how transactions affect index updates

**Congratulations on completing Indexing Strategies!** You now have staff-level understanding of index internals and production optimization strategies. 🚀
