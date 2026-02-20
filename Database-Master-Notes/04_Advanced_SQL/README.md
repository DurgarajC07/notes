# 🚀 Advanced SQL - Beyond the Basics

> Master advanced SQL techniques that distinguish senior engineers from beginners: window functions, recursive queries, pivoting, JSON/XML operations, full-text search, and spatial queries.

---

## 📚 Section Overview

This section covers **advanced SQL capabilities** that are critical for:
- **Data Analytics**: Window functions, pivoting, cohort analysis
- **Hierarchical Data**: Recursive queries for org charts, bill of materials
- **Semi-Structured Data**: JSON and XML operations for flexible schemas
- **Content Search**: Full-text search for search engines and content platforms
- **Location Services**: Spatial queries for mapping and GIS applications

**Target Audience**: Mid-level → Senior engineers working on data-intensive applications

---

## 📖 Files in This Section

### [01_Window_Functions.md](01_Window_Functions.md) 🪟
**OLAP functions for analytical queries without GROUP BY**

**What You'll Learn:**
- ROW_NUMBER, RANK, DENSE_RANK, NTILE for ranking
- LAG, LEAD, FIRST_VALUE, LAST_VALUE for row comparisons
- Running totals, moving averages, window frames
- Performance: 150x faster than self-joins
- Real-world: E-commerce rankings, financial running balances, cohort analysis

**Key Concepts:**
```sql
-- Running total without self-join
SUM(revenue) OVER (ORDER BY date) as running_total

-- Top N per category
ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) as rank

-- Month-over-month comparison
LAG(revenue, 1) OVER (ORDER BY month) as prev_month_revenue
```

**When to Use:**
- ✅ Ranking (top products, leaderboards)
- ✅ Running calculations (cumulative sales, moving averages)
- ✅ Row comparisons (previous/next values, gaps)
- ✅ Pagination with row numbers

---

### [02_Recursive_Queries.md](02_Recursive_Queries.md) 🔄
**WITH RECURSIVE for hierarchical and graph-like data**

**What You'll Learn:**
- Recursive CTE structure (anchor + recursive + termination)
- Org charts, category trees, bill of materials
- Cycle detection and infinite loop prevention
- Graph traversal (social networks, shortest paths)
- Performance: Closure tables for complex queries

**Key Concepts:**
```sql
-- Org chart traversal
WITH RECURSIVE hierarchy AS (
    SELECT id, manager_id, 0 as level FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.manager_id, h.level + 1
    FROM employees e JOIN hierarchy h ON e.manager_id = h.id
    WHERE h.level < 100  -- Termination condition
)
SELECT * FROM hierarchy;
```

**When to Use:**
- ✅ Organizational hierarchies (manager-employee chains)
- ✅ Category/taxonomy trees (parent-child relationships)
- ✅ Graph traversal (social networks, friends-of-friends)
- ✅ Bill of materials (product components)

---

### [03_Pivoting_Unpivoting.md](03_Pivoting_Unpivoting.md) 🔄
**Row-to-column and column-to-row transformations**

**What You'll Learn:**
- PIVOT (rows → columns) for reports
- UNPIVOT (columns → rows) for normalization
- Dynamic pivot with unknown columns
- Manual pivot using CASE (portable)
- Performance: Similar to GROUP BY + CASE

**Key Concepts:**
```sql
-- Quarterly revenue report (pivot)
SELECT * FROM (
    SELECT year, quarter, revenue FROM sales
) PIVOT (
    SUM(revenue) FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
);

-- Normalize wide format (unpivot)
SELECT year, quarter, revenue FROM quarterly_sales
UNPIVOT (revenue FOR quarter IN (Q1, Q2, Q3, Q4));
```

**When to Use:**
- ✅ Cross-tab reports (quarterly revenues, cohort matrices)
- ✅ ETL transformations (normalize legacy data)
- ✅ Dynamic dashboards (pivot on user-selected dimensions)
- ✅ Financial reporting (monthly P&L)

---

### [04_JSON_Operations.md](04_JSON_Operations.md) 🔗
**Store, query, and manipulate semi-structured JSON data**

**What You'll Learn:**
- JSON vs JSONB (PostgreSQL): binary format 16x faster
- Extraction operators: `->` (JSON), `->>` (text), `@>` (contains)
- GIN indexes for JSON queries (10-30x speedup)
- When to use JSON vs normalized tables
- Security: Prevent JSON injection

**Key Concepts:**
```sql
-- PostgreSQL: Extract JSON fields
SELECT 
    metadata->>'name' as name,
    (metadata->>'price')::NUMERIC as price
FROM products
WHERE metadata @> '{"category": "Electronics"}';

-- JSON aggregation for APIs
SELECT jsonb_build_object(
    'user_id', u.id,
    'orders', jsonb_agg(jsonb_build_object('order_id', o.id, 'total', o.total))
) FROM users u JOIN orders o ON u.id = o.user_id;
```

**When to Use:**
- ✅ Variable schemas (product attributes differ by category)
- ✅ API response caching (opaque external data)
- ✅ User preferences/settings (flexible key-value)
- ✅ Event logging (different event types, different fields)

---

### [05_XML_Operations.md](05_XML_Operations.md) 📄
**XML data operations for legacy integrations**

**What You'll Learn:**
- XML types (SQL Server native, PostgreSQL basic)
- XPath queries for extraction
- FOR XML (SQL to XML) and OPENXML (XML to SQL)
- XML indexes (PRIMARY, PATH, VALUE)
- When XML is required (healthcare HL7, finance SWIFT)

**Key Concepts:**
```sql
-- SQL Server: Extract XML values
SELECT 
    data.value('(/product/name)[1]', 'VARCHAR(100)') as name,
    data.value('(/product/price)[1]', 'DECIMAL(10,2)') as price
FROM products;

-- Convert table to XML
SELECT * FROM orders
FOR XML PATH('order'), ROOT('orders');
```

**When to Use:**
- ✅ Legacy system integration (SOAP APIs)
- ✅ Healthcare (HL7 messages)
- ✅ Finance (SWIFT wire transfers)
- ✅ Document-centric data (contracts, invoices)
- ❌ Avoid for new projects (use JSON instead)

---

### [06_Full_Text_Search.md](06_Full_Text_Search.md) 🔍
**Linguistic search with ranking and stemming**

**What You'll Learn:**
- FTS vs LIKE: 150x faster, linguistic matching
- tsvector/tsquery (PostgreSQL), FULLTEXT (MySQL), CONTAINS (SQL Server)
- GIN indexes for full-text search
- Ranking (ts_rank), highlighting (ts_headline)
- Autocomplete with trigrams (pg_trgm)

**Key Concepts:**
```sql
-- PostgreSQL: Full-text search with ranking
CREATE INDEX idx_content ON articles USING GIN (to_tsvector('english', content));

SELECT 
    title,
    ts_rank(to_tsvector('english', content), query) as relevance
FROM articles, to_tsquery('english', 'database & performance') query
WHERE to_tsvector('english', content) @@ query
ORDER BY relevance DESC;
```

**When to Use:**
- ✅ Search engines (product search, documentation)
- ✅ Content platforms (blogs, forums, knowledge bases)
- ✅ Autocomplete/typeahead features
- ✅ Natural language queries (vs exact LIKE patterns)

---

### [07_Spatial_Queries.md](07_Spatial_Queries.md) 🗺️
**Geographic data for location-based services**

**What You'll Learn:**
- PostGIS (best spatial support), geometry vs geography types
- GIST indexes for spatial queries (40-100x speedup)
- ST_Distance (distance calculations), ST_DWithin (radius queries)
- K-nearest-neighbor with `<->` operator
- Point-in-polygon, polygon intersections

**Key Concepts:**
```sql
-- PostGIS: Find stores within 5km
CREATE INDEX idx_location ON stores USING GIST (location);

SELECT name FROM stores
WHERE ST_DWithin(
    location,
    ST_MakePoint(-122.4194, 37.7749)::geography,
    5000  -- meters
);

-- Nearest 5 stores (K-NN)
SELECT name FROM stores
ORDER BY location <-> ST_MakePoint(-122.4, 37.7)::geography
LIMIT 5;
```

**When to Use:**
- ✅ Ride-sharing (driver matching, route optimization)
- ✅ Real estate (property search by location, school districts)
- ✅ Logistics (delivery zones, warehouse assignment)
- ✅ Store locators (find nearest locations)

---

## 🎯 Learning Path

### Path 1: Data Analyst → Senior Analyst
```
1. Window Functions ← Start here
   └── Ranking, running totals, cohort analysis

2. Pivoting/Unpivoting
   └── Cross-tab reports, dynamic dashboards

3. Full-Text Search
   └── Search features for data exploration
```

### Path 2: Backend Engineer → Senior Engineer
```
1. JSON Operations ← Start here
   └── Flexible schemas, API integrations

2. Recursive Queries
   └── Hierarchical data (org charts, comments)

3. Full-Text Search
   └── Search functionality in applications

4. Spatial Queries (if building location features)
   └── Maps, geolocation, proximity search
```

### Path 3: Database Specialist → Staff Engineer
```
Complete all 7 files in order:
1. Window Functions (analytics foundation)
2. Recursive Queries (hierarchical mastery)
3. Pivoting/Unpivoting (reporting flexibility)
4. JSON Operations (semi-structured data)
5. XML Operations (legacy integrations)
6. Full-Text Search (content discovery)
7. Spatial Queries (geographic data)
```

---

## 💡 When to Use Each Technique

| Technique | Use When | Avoid When |
|-----------|----------|------------|
| **Window Functions** | Analytics, rankings, running totals | Simple aggregations (use GROUP BY) |
| **Recursive Queries** | Hierarchical data (org charts, categories) | Fixed depth (use multiple JOINs) |
| **Pivot** | Cross-tab reports, quarterly summaries | Many columns (slow, consider app logic) |
| **JSON** | Variable schema, API data, user prefs | Frequently queried fields (normalize) |
| **XML** | Legacy integrations, SOAP APIs | New projects (use JSON) |
| **Full-Text Search** | Natural language search, content platforms | Exact patterns (use LIKE), IDs (use =) |
| **Spatial** | Location services, maps, proximity | Non-geographic data |

---

## ⚡ Performance Quick Wins

**Window Functions:**
```sql
-- ✅ Index PARTITION BY and ORDER BY columns
CREATE INDEX idx_category_date ON sales(category, sale_date);

-- ✅ Use named WINDOW clause to avoid recomputation
WINDOW w AS (PARTITION BY category ORDER BY sale_date);
```

**Recursive Queries:**
```sql
-- ✅ Index parent-child relationship
CREATE INDEX idx_parent ON nodes(parent_id);

-- ✅ Add depth limit to prevent infinite loops
WHERE depth < 100
```

**JSON:**
```sql
-- ✅ Use JSONB (not JSON) in PostgreSQL
-- ✅ Create GIN index for containment queries
CREATE INDEX idx_metadata ON products USING GIN (metadata);

-- ✅ Extract hot fields to dedicated columns
ALTER TABLE products ADD price NUMERIC 
    GENERATED ALWAYS AS ((metadata->>'price')::NUMERIC) STORED;
CREATE INDEX idx_price ON products(price);
```

**Full-Text Search:**
```sql
-- ✅ Store tsvector in dedicated column
ALTER TABLE articles ADD search_tsv tsvector 
    GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;
CREATE INDEX idx_search ON articles USING GIN (search_tsv);
```

**Spatial:**
```sql
-- ✅ Use geography for accuracy, geometry for performance
-- ✅ Create GIST index
CREATE INDEX idx_location ON stores USING GIST (location);

-- ✅ Use ST_DWithin (index-optimized) not ST_Distance < N
WHERE ST_DWithin(location, point, 5000);
```

---

## 🛠️ Practice Exercises

### Exercise 1: Window Functions
**Task:** Calculate month-over-month revenue growth percentage
```sql
-- Schema: sales(id, sale_date, amount)
-- Expected output: month, total_revenue, prev_month_revenue, growth_pct
```

<details>
<summary>Solution</summary>

```sql
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', sale_date) as month,
        SUM(amount) as total_revenue
    FROM sales
    GROUP BY DATE_TRUNC('month', sale_date)
)
SELECT 
    month,
    total_revenue,
    LAG(total_revenue) OVER (ORDER BY month) as prev_month_revenue,
    ROUND(
        100.0 * (total_revenue - LAG(total_revenue) OVER (ORDER BY month)) / 
        LAG(total_revenue) OVER (ORDER BY month),
        2
    ) as growth_pct
FROM monthly_revenue
ORDER BY month;
```
</details>

### Exercise 2: Recursive Queries
**Task:** Find all subordinates of a manager (any depth)
```sql
-- Schema: employees(id, name, manager_id)
-- Input: manager_id = 5
-- Expected: All employees reporting to manager 5 (direct or indirect)
```

<details>
<summary>Solution</summary>

```sql
WITH RECURSIVE subordinates AS (
    -- Anchor: The manager themselves
    SELECT id, name, manager_id, 0 as level
    FROM employees
    WHERE id = 5
    
    UNION ALL
    
    -- Recursive: All subordinates
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
    WHERE s.level < 50  -- Max depth
)
SELECT id, name, level
FROM subordinates
WHERE id != 5  -- Exclude the manager themselves
ORDER BY level, name;
```
</details>

### Exercise 3: JSON Operations
**Task:** Find products with specific attribute, return as JSON
```sql
-- Schema: products(id, name, attributes JSONB)
-- Task: Find all electronics with price < 1000, return JSON array
```

<details>
<summary>Solution</summary>

```sql
SELECT jsonb_agg(
    jsonb_build_object(
        'id', id,
        'name', name,
        'price', (attributes->>'price')::NUMERIC
    )
) as products
FROM products
WHERE attributes->>'category' = 'Electronics'
  AND (attributes->>'price')::NUMERIC < 1000;
```
</details>

---

## 🔗 Related Sections

**Prerequisites:**
- [03_SQL_Core](../03_SQL_Core/) - Subqueries, CTEs, JOINs foundation
- [02_Data_Modeling](../02_Data_Modeling/) - Schema design patterns

**Next Steps:**
- [05_Query_Optimization](../05_Query_Optimization/) - Execution plans, query tuning
- [06_Indexing_Performance](../06_Indexing_Performance/) - Index strategies for advanced queries
- [07_Transactions_Concurrency](../07_Transactions_Concurrency/) - ACID, isolation levels

**Related Topics:**
- [10_Stored_Procedures_Functions](../10_Stored_Procedures_Functions/) - Encapsulate complex logic
- [19_Data_Warehousing](../19_Data_Warehousing/) - OLAP and analytical queries at scale

---

## 📊 Complexity Comparison

| Technique | Learning Curve | Performance Gain | Use Frequency |
|-----------|----------------|------------------|---------------|
| Window Functions | ⭐⭐⭐ Medium | ⚡⚡⚡ High (150x) | ⭐⭐⭐⭐⭐ Very High |
| Recursive Queries | ⭐⭐⭐⭐ Hard | ⚡⚡ Medium | ⭐⭐⭐ Medium |
| Pivoting | ⭐⭐ Easy | ⚡⚡ Medium | ⭐⭐⭐ Medium |
| JSON | ⭐⭐⭐ Medium | ⚡⚡ Medium | ⭐⭐⭐⭐ High |
| XML | ⭐⭐⭐⭐ Hard | ⚡ Low | ⭐ Low |
| Full-Text Search | ⭐⭐⭐ Medium | ⚡⚡⚡⚡ Very High (150x) | ⭐⭐⭐⭐ High |
| Spatial | ⭐⭐⭐⭐ Hard | ⚡⚡⚡⚡ Very High (75x) | ⭐⭐ Low-Medium |

---

## 🎓 Interview Focus Areas

**Most Asked Topics:**
1. **Window Functions** (90% of senior+ interviews)
   - Difference from GROUP BY
   - RANK vs DENSE_RANK vs ROW_NUMBER
   - Running totals, moving averages

2. **Recursive Queries** (60% of senior+ interviews)
   - WITH RECURSIVE structure
   - Preventing infinite loops
   - Use cases (org charts, hierarchies)

3. **JSON Operations** (70% of modern stack interviews)
   - When to use JSON vs normalized tables
   - Indexing strategies (GIN, computed columns)
   - Extraction operators

4. **Full-Text Search** (40% of product-focused interviews)
   - LIKE vs FTS performance
   - Ranking algorithms
   - Index types (GIN vs GiST)

5. **Spatial Queries** (20%, location-based companies 80%)
   - geometry vs geography
   - K-nearest-neighbor
   - Spatial index performance

---

## 📚 Summary

**You've Mastered:**
- ✅ **Window Functions**: Analytics without GROUP BY, 150x faster than self-joins
- ✅ **Recursive Queries**: Hierarchical data traversal, infinite loop prevention
- ✅ **Pivoting**: Row-column transformations for reporting
- ✅ **JSON/XML**: Semi-structured data for flexible schemas
- ✅ **Full-Text Search**: Linguistic search, 150x faster than LIKE
- ✅ **Spatial Queries**: Geographic data for location-based services

**Next Steps:**
1. Practice exercises in this README
2. Review execution plans: [05_Query_Optimization](../05_Query_Optimization/)
3. Optimize indexes: [06_Indexing_Performance](../06_Indexing_Performance/)
4. Apply to real-world projects: [12_Industry_Projects](../12_Industry_Projects/)

**Key Principle:**
> _"Advanced SQL distinguishes senior engineers. Master these techniques to write performant, elegant queries that solve complex problems with simple code."_

---

**Section Progress:** 7/7 files complete ✅  
**Estimated Study Time:** 18-24 hours (3-4 hours per file)  
**Difficulty:** ⭐⭐⭐⭐ (Senior Level)

---

*Last Updated: February 20, 2026*
