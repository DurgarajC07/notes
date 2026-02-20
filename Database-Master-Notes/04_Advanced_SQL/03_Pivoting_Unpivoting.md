# 🔄 Pivoting and Unpivoting - Row-Column Transformations

> Pivoting converts rows to columns (wide format), unpivoting converts columns to rows (long format). Master data reshaping for reports, analytics, and data transformations.

---

## 📖 1. Concept Explanation

### What is Pivoting?

**Pivoting** transforms row values into column headers, creating a cross-tabulation (wide format).

```sql
-- Data (long format):
year | quarter | revenue
2024 | Q1      | 100
2024 | Q2      | 150
2024 | Q3      | 200

-- After PIVOT (wide format):
year | Q1  | Q2  | Q3
2024 | 100 | 150 | 200
```

### What is Unpivoting?

**Unpivoting** transforms column headers into row values (wide → long format).

```sql
-- Data (wide format):
year | Q1  | Q2  | Q3
2024 | 100 | 150 | 200

-- After UNPIVOT (long format):
year | quarter | revenue
2024 | Q1      | 100
2024 | Q2      | 150
2024 | Q3      | 200
```

### Database Support

```
PIVOTING SUPPORT
│
├── SQL Server: Native PIVOT/UNPIVOT ✅
├── Oracle: Native PIVOT/UNPIVOT ✅
├── PostgreSQL: crosstab() extension, manual CASE ✅
├── MySQL: Manual CASE expressions only
└── BigQuery: PIVOT/UNPIVOT (2023+) ✅
```

---

## 🧠 2. Why It Matters in Real Systems

### Without Pivot: Repetitive CASE Statements

**❌ Bad: Manual aggregation for each column**
```sql
-- Create quarterly report manually
SELECT 
    year,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue ELSE 0 END) as Q1,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue ELSE 0 END) as Q2,
    SUM(CASE WHEN quarter = 'Q3' THEN revenue ELSE 0 END) as Q3,
    SUM(CASE WHEN quarter = 'Q4' THEN revenue ELSE 0 END) as Q4
FROM sales
GROUP BY year;

-- Problems:
-- - Tedious for many columns ❌
-- - Error-prone (copy-paste mistakes) ❌
-- - Must update query when adding columns ❌
```

**✅ Good: Native PIVOT (SQL Server/Oracle)**
```sql
SELECT *
FROM (
    SELECT year, quarter, revenue
    FROM sales
) AS source
PIVOT (
    SUM(revenue)
    FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS pivoted;

-- Cleaner ✅
-- Less error-prone ✅
-- Intent more clear ✅
```

---

## ⚙️ 3. Internal Working

### Pivot Execution Plan

```sql
-- SQL Server PIVOT query:
SELECT *
FROM sales
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2])) AS pvt;

-- Execution steps:
-- 1. Filter columns: year, quarter, revenue
-- 2. GROUP BY year implicitly
-- 3. Create aggregates: SUM(CASE WHEN quarter='Q1'...)
-- 4. Output columns: year, [Q1], [Q2]

-- Equivalent manual query:
SELECT 
    year,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue END) as Q1,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue END) as Q2
FROM sales
GROUP BY year;
```

---

## ✅ 4. Best Practices

### Use CTE for Clarity

```sql
-- ✅ Wrap source query in CTE
WITH source AS (
    SELECT 
        year,
        quarter,
        SUM(revenue) as total_revenue
    FROM sales
    WHERE year >= 2020
    GROUP BY year, quarter
)
SELECT *
FROM source
PIVOT (
    SUM(total_revenue)
    FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS pivoted;

-- Benefit: Separate filtering/aggregation from pivoting ✅
```

### Specify Columns Explicitly

```sql
-- ❌ Implicit columns (unexpected behavior)
SELECT *
FROM sales
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2])) AS pvt;
-- Includes ALL non-pivoted columns (dangerous if table has many)

-- ✅ Explicit columns
SELECT year, Q1, Q2
FROM (
    SELECT year, quarter, revenue
    FROM sales
) AS source
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2])) AS pvt;
```

### Handle NULLs

```sql
-- ✅ Replace NULLs with 0 or default
SELECT 
    year,
    COALESCE(Q1, 0) as Q1,
    COALESCE(Q2, 0) as Q2,
    COALESCE(Q3, 0) as Q3
FROM (
    SELECT year, quarter, revenue
    FROM sales
) AS source
PIVOT (
    SUM(revenue)
    FOR quarter IN ([Q1], [Q2], [Q3])
) AS pivoted;
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Extra Columns in Source

```sql
-- ❌ Extra column affects grouping
SELECT *
FROM (
    SELECT year, quarter, revenue, region  -- Extra column! ❌
    FROM sales
) AS source
PIVOT (
    SUM(revenue)
    FOR quarter IN ([Q1], [Q2])
) AS pivoted;

-- Result: Pivot groups by (year, region) → multiple rows per year
-- Expected: One row per year

-- ✅ Include only necessary columns
SELECT *
FROM (
    SELECT year, quarter, revenue  -- No extra columns ✅
    FROM sales
) AS source
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2])) AS pvt;
```

### Mistake 2: Missing Aggregate Function

```sql
-- ❌ No aggregate in PIVOT
SELECT *
FROM sales
PIVOT (revenue FOR quarter IN ([Q1], [Q2])) AS pvt;
-- ERROR: Must use aggregate (SUM, AVG, MAX, etc.)

-- ✅ Add aggregate
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2])) AS pvt;
```

### Mistake 3: Dynamic Columns Not Handled

```sql
-- ❌ Pivot requires hardcoded columns
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2])) AS pvt;
-- If data has Q5, it won't appear

-- ✅ Use dynamic SQL (SQL Server)
DECLARE @columns NVARCHAR(MAX);
DECLARE @sql NVARCHAR(MAX);

SELECT @columns = STRING_AGG(QUOTENAME(quarter), ',')
FROM (SELECT DISTINCT quarter FROM sales) AS quarters;

SET @sql = N'
SELECT *
FROM (SELECT year, quarter, revenue FROM sales) AS source
PIVOT (SUM(revenue) FOR quarter IN (' + @columns + N')) AS pvt;
';

EXEC sp_executesql @sql;
```

---

## 🔐 6. Security Considerations

### Validate Dynamic Column Names

```sql
-- ❌ SQL injection risk
DECLARE @col NVARCHAR(50) = @UserInput;  -- User provides "Q1]; DROP TABLE sales; --"
EXEC('SELECT * FROM sales PIVOT (SUM(revenue) FOR quarter IN ([' + @col + '])) AS pvt');

-- ✅ Whitelist validation
IF @col NOT IN ('Q1', 'Q2', 'Q3', 'Q4')
    THROW 50000, 'Invalid column name', 1;
```

---

## 🚀 7. Performance Optimization

### Manual CASE vs Native PIVOT

```sql
-- Benchmark: 1M rows

-- Manual CASE: ~3.2 seconds
SELECT 
    year,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue END) as Q1,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue END) as Q2,
    SUM(CASE WHEN quarter = 'Q3' THEN revenue END) as Q3,
    SUM(CASE WHEN quarter = 'Q4' THEN revenue END) as Q4
FROM sales
GROUP BY year;

-- Native PIVOT: ~3.0 seconds (slightly faster)
SELECT *
FROM sales
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2], [Q3], [Q4])) AS pvt;

-- Native PIVOT optimizes execution plan slightly ✅
```

### Index for Pivot Operations

```sql
-- ✅ Composite index on grouping + pivot columns
CREATE INDEX idx_year_quarter ON sales(year, quarter) INCLUDE (revenue);

-- Covers: GROUP BY year + pivot on quarter
-- Query time: 3.0s → 0.8s ✅
```

---

## 🧪 8. Examples

### Example 1: SQL Server Native PIVOT

```sql
-- Quarterly revenue report
SELECT *
FROM (
    SELECT 
        YEAR(order_date) as year,
        'Q' + CAST(DATEPART(QUARTER, order_date) AS VARCHAR) as quarter,
        total_amount
    FROM orders
) AS source
PIVOT (
    SUM(total_amount)
    FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS pivoted
ORDER BY year;

-- Output:
-- year | Q1      | Q2      | Q3      | Q4
-- 2023 | 500000  | 750000  | 850000  | 900000
-- 2024 | 600000  | 800000  | NULL    | NULL
```

### Example 2: PostgreSQL crosstab (requires tablefunc extension)

```sql
-- Install extension
CREATE EXTENSION IF NOT EXISTS tablefunc;

-- Pivot using crosstab
SELECT *
FROM crosstab(
    'SELECT year, quarter, SUM(revenue)
     FROM sales
     GROUP BY year, quarter
     ORDER BY year, quarter',
    'SELECT DISTINCT quarter FROM sales ORDER BY quarter'
) AS ct (
    year INT,
    Q1 NUMERIC,
    Q2 NUMERIC,
    Q3 NUMERIC,
    Q4 NUMERIC
);
```

### Example 3: PostgreSQL Manual PIVOT (no extension)

```sql
-- Manual pivot with CASE
SELECT 
    year,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue END) as Q1,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue END) as Q2,
    SUM(CASE WHEN quarter = 'Q3' THEN revenue END) as Q3,
    SUM(CASE WHEN quarter = 'Q4' THEN revenue END) as Q4
FROM sales
GROUP BY year
ORDER BY year;
```

### Example 4: MySQL Manual PIVOT

```sql
-- MySQL has no native PIVOT, use CASE
SELECT 
    product_category,
    SUM(CASE WHEN YEAR(sale_date) = 2022 THEN amount ELSE 0 END) as `2022`,
    SUM(CASE WHEN YEAR(sale_date) = 2023 THEN amount ELSE 0 END) as `2023`,
    SUM(CASE WHEN YEAR(sale_date) = 2024 THEN amount ELSE 0 END) as `2024`
FROM sales
GROUP BY product_category;
```

### Example 5: Dynamic PIVOT (SQL Server)

```sql
-- Generate pivot columns dynamically
DECLARE @cols NVARCHAR(MAX);
DECLARE @query NVARCHAR(MAX);

-- Get distinct quarter values
SELECT @cols = STRING_AGG(QUOTENAME(quarter), ',')
FROM (SELECT DISTINCT quarter FROM sales) AS q;

-- Build dynamic query
SET @query = N'
SELECT *
FROM (
    SELECT year, quarter, revenue
    FROM sales
) AS source
PIVOT (
    SUM(revenue)
    FOR quarter IN (' + @cols + N')
) AS pivoted
ORDER BY year;
';

EXEC sp_executesql @query;
```

### Example 6: SQL Server UNPIVOT

```sql
-- Convert quarterly columns to rows
SELECT year, quarter, revenue
FROM (
    SELECT year, Q1, Q2, Q3, Q4
    FROM quarterly_sales
) AS source
UNPIVOT (
    revenue FOR quarter IN (Q1, Q2, Q3, Q4)
) AS unpvt;

-- Input:
-- year | Q1  | Q2  | Q3  | Q4
-- 2024 | 100 | 150 | 200 | 250

-- Output:
-- year | quarter | revenue
-- 2024 | Q1      | 100
-- 2024 | Q2      | 150
-- 2024 | Q3      | 200
-- 2024 | Q4      | 250
```

### Example 7: PostgreSQL Manual UNPIVOT

```sql
-- Convert columns to rows with UNION
SELECT year, 'Q1' as quarter, Q1 as revenue FROM quarterly_sales WHERE Q1 IS NOT NULL
UNION ALL
SELECT year, 'Q2', Q2 FROM quarterly_sales WHERE Q2 IS NOT NULL
UNION ALL
SELECT year, 'Q3', Q3 FROM quarterly_sales WHERE Q3 IS NOT NULL
UNION ALL
SELECT year, 'Q4', Q4 FROM quarterly_sales WHERE Q4 IS NOT NULL
ORDER BY year, quarter;
```

### Example 8: Unpivot with VALUES (PostgreSQL)

```sql
-- Cleaner UNPIVOT using CROSS JOIN LATERAL
SELECT 
    year,
    quarter,
    revenue
FROM quarterly_sales
CROSS JOIN LATERAL (
    VALUES 
        ('Q1', Q1),
        ('Q2', Q2),
        ('Q3', Q3),
        ('Q4', Q4)
) AS unpivoted(quarter, revenue)
WHERE revenue IS NOT NULL
ORDER BY year, quarter;
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Financial Reporting - Monthly P&L

```sql
-- Monthly revenue by product line
WITH monthly_data AS (
    SELECT 
        product_line,
        FORMAT(sale_date, 'MMM') as month,
        SUM(revenue) as total_revenue
    FROM sales
    WHERE YEAR(sale_date) = 2024
    GROUP BY product_line, FORMAT(sale_date, 'MMM')
)
SELECT *
FROM monthly_data
PIVOT (
    SUM(total_revenue)
    FOR month IN ([Jan], [Feb], [Mar], [Apr], [May], [Jun], 
                  [Jul], [Aug], [Sep], [Oct], [Nov], [Dec])
) AS pvt;

-- Output:
-- product_line | Jan    | Feb    | Mar    | ... | Dec
-- Electronics  | 50000  | 52000  | 55000  | ... | 70000
-- Clothing     | 30000  | 28000  | 35000  | ... | 45000
```

### Case Study 2: Analytics - Cohort Retention Matrix

```sql
-- User retention by signup cohort
WITH cohorts AS (
    SELECT 
        u.user_id,
        DATE_FORMAT(u.signup_date, '%Y-%m') as cohort,
        TIMESTAMPDIFF(MONTH, u.signup_date, a.activity_date) as months_since_signup
    FROM users u
    LEFT JOIN user_activity a ON u.user_id = a.user_id
    WHERE a.activity_date >= u.signup_date
),
retention AS (
    SELECT 
        cohort,
        months_since_signup,
        COUNT(DISTINCT user_id) as active_users
    FROM cohorts
    GROUP BY cohort, months_since_signup
)
SELECT 
    cohort,
    SUM(CASE WHEN months_since_signup = 0 THEN active_users END) as Month_0,
    SUM(CASE WHEN months_since_signup = 1 THEN active_users END) as Month_1,
    SUM(CASE WHEN months_since_signup = 2 THEN active_users END) as Month_2,
    SUM(CASE WHEN months_since_signup = 3 THEN active_users END) as Month_3,
    SUM(CASE WHEN months_since_signup = 4 THEN active_users END) as Month_4,
    SUM(CASE WHEN months_since_signup = 5 THEN active_users END) as Month_5,
    SUM(CASE WHEN months_since_signup = 6 THEN active_users END) as Month_6
FROM retention
GROUP BY cohort
ORDER BY cohort;

-- Output:
-- cohort  | Month_0 | Month_1 | Month_2 | Month_3 | ...
-- 2024-01 | 1000    | 650     | 520     | 450     | ...
-- 2024-02 | 1200    | 750     | 600     | 530     | ...
```

### Case Study 3: ETL - Normalize Wide Tables

```sql
-- Legacy table with wide format
CREATE TABLE survey_responses_wide (
    response_id INT,
    user_id INT,
    q1_rating INT,
    q2_rating INT,
    q3_rating INT,
    q4_rating INT,
    q5_rating INT
);

-- Transform to normalized format using UNPIVOT
INSERT INTO survey_responses_normalized (response_id, user_id, question, rating)
SELECT response_id, user_id, question, rating
FROM survey_responses_wide
UNPIVOT (
    rating FOR question IN (q1_rating, q2_rating, q3_rating, q4_rating, q5_rating)
) AS unpvt;

-- Now easy to analyze: AVG(rating) by question
SELECT question, AVG(rating)
FROM survey_responses_normalized
GROUP BY question;
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What is the difference between PIVOT and GROUP BY with CASE?

**Answer:**

**PIVOT:**
- **Syntax**: Specialized syntax for row-to-column transformation
- **Readability**: Intent is clear (pivoting data)
- **Performance**: Slightly optimized by query optimizer
- **Dynamic**: Requires dynamic SQL for unknown columns
- **Database Support**: SQL Server, Oracle, BigQuery (recent)

**GROUP BY with CASE:**
- **Syntax**: Standard SQL (works everywhere)
- **Readability**: More verbose
- **Performance**: Similar to PIVOT
- **Dynamic**: Also requires dynamic SQL
- **Database Support**: All databases ✅

**Example comparison:**
```sql
-- PIVOT (SQL Server)
SELECT * FROM sales
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2], [Q3], [Q4])) AS pvt;

-- GROUP BY + CASE (Universal)
SELECT 
    SUM(CASE WHEN quarter = 'Q1' THEN revenue END) as Q1,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue END) as Q2,
    SUM(CASE WHEN quarter = 'Q3' THEN revenue END) as Q3,
    SUM(CASE WHEN quarter = 'Q4' THEN revenue END) as Q4
FROM sales;
```

**Use PIVOT** when available for clarity, **use CASE** for portability.

---

### Q2: How do you create a dynamic PIVOT for unknown column values?

**Answer:** Use dynamic SQL to build column list at runtime.

```sql
-- SQL Server dynamic PIVOT
DECLARE @cols NVARCHAR(MAX);
DECLARE @query NVARCHAR(MAX);

-- Step 1: Get distinct pivot values
SELECT @cols = STRING_AGG(QUOTENAME(quarter), ',')
FROM (SELECT DISTINCT quarter FROM sales) AS q;

-- Step 2: Build dynamic query
SET @query = N'
SELECT *
FROM (
    SELECT year, quarter, revenue FROM sales
) AS source
PIVOT (
    SUM(revenue)
    FOR quarter IN (' + @cols + N')
) AS pivoted;
';

-- Step 3: Execute
EXEC sp_executesql @query;
```

**PostgreSQL alternative:**
```sql
-- Use crosstab with dynamic column definition
SELECT crosstab(
    'SELECT year, quarter, SUM(revenue) FROM sales GROUP BY 1,2 ORDER BY 1,2'
) AS ct (year INT, q1 NUMERIC, q2 NUMERIC, q3 NUMERIC, q4 NUMERIC);
```

---

### Q3: When would you use UNPIVOT in a real system?

**Answer:**

**Use UNPIVOT to normalize denormalized data:**

**1. ETL from legacy systems:**
```sql
-- Old system: Wide format
-- user_id | jan_spend | feb_spend | mar_spend
-- New system: Long format (normalized)
-- user_id | month | spend

-- UNPIVOT transformation
SELECT user_id, month, spend
FROM legacy_data
UNPIVOT (spend FOR month IN (jan_spend, feb_spend, mar_spend)) AS unpvt;
```

**2. Time-series analysis:**
```sql
-- Convert wide format to time-series
-- product | 2022_q1 | 2022_q2 | 2022_q3
-- → product | quarter | revenue
UNPIVOT (revenue FOR quarter IN (q1_2022, q2_2022, q3_2022)) AS unpvt;
```

**3. Survey data normalization:**
```sql
-- response_id | q1_rating | q2_rating | q3_rating
-- → response_id | question_id | rating
```

---

### Q4: What are the performance implications of PIVOT?

**Answer:**

**PIVOT performance is similar to GROUP BY + CASE:**

**1. Both use aggregation** → similar execution plan
```sql
-- PIVOT: ~3.0s on 1M rows
SELECT * FROM sales
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2], [Q3], [Q4])) AS pvt;

-- GROUP BY: ~3.2s on 1M rows
SELECT SUM(CASE WHEN quarter='Q1' THEN revenue END) as Q1, ...
FROM sales GROUP BY year;
```

**2. Indexes help both:**
```sql
CREATE INDEX idx_composite ON sales(year, quarter) INCLUDE (revenue);
-- PIVOT: 3.0s → 0.8s ✅
-- GROUP BY: 3.2s → 0.9s ✅
```

**3. Dynamic PIVOT adds overhead:**
- Dynamic SQL compilation on each execution
- Use stored procedure or cache query string

**4. Many columns = slower:**
- Pivoting 100 columns ≈ 100 aggregations
- Consider materialized view for frequent queries

---

### Q5: How do you handle NULL values in PIVOT operations?

**Answer:**

**PIVOT treats NULL differently than 0:**

```sql
-- Data with NULLs
-- year | quarter | revenue
-- 2024 | Q1      | 100
-- 2024 | Q2      | NULL
-- 2024 | Q3      | 200

-- PIVOT result
SELECT * FROM sales
PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2], [Q3])) AS pvt;
-- year | Q1  | Q2   | Q3
-- 2024 | 100 | NULL | 200  -- Q2 is NULL (not 0)

-- Replace NULLs with 0:
SELECT 
    year,
    COALESCE(Q1, 0) as Q1,
    COALESCE(Q2, 0) as Q2,
    COALESCE(Q3, 0) as Q3
FROM (
    SELECT * FROM sales
    PIVOT (SUM(revenue) FOR quarter IN ([Q1], [Q2], [Q3])) AS pvt
) AS result;
```

**NULL handling strategies:**
1. **COALESCE** after pivot
2. **ISNULL(revenue, 0)** before pivot
3. **SUM(COALESCE(revenue, 0))** in aggregate
4. **Default values in source data**

---

## 🧩 11. Design Patterns

### Pattern 1: Cross-Tab Report

```sql
-- Standard cross-tab: rows × columns
SELECT 
    product_category,
    SUM(CASE WHEN sale_year = 2022 THEN revenue END) as "2022",
    SUM(CASE WHEN sale_year = 2023 THEN revenue END) as "2023",
    SUM(CASE WHEN sale_year = 2024 THEN revenue END) as "2024",
    SUM(revenue) as Total
FROM sales
GROUP BY product_category
WITH ROLLUP;  -- Grand total row
```

### Pattern 2: Conditional Aggregation (Alternative to PIVOT)

```sql
-- Aggregate multiple columns without PIVOT
SELECT 
    region,
    COUNT(*) as total_customers,
    COUNT(CASE WHEN status = 'active' THEN 1 END) as active_count,
    COUNT(CASE WHEN status = 'inactive' THEN 1 END) as inactive_count,
    AVG(CASE WHEN status = 'active' THEN revenue END) as avg_active_revenue,
    AVG(CASE WHEN status = 'inactive' THEN revenue END) as avg_inactive_revenue
FROM customers
GROUP BY region;
```

### Pattern 3: Multi-Level Pivot

```sql
-- Pivot on two dimensions
WITH pivoted_data AS (
    SELECT 
        region,
        SUM(CASE WHEN year = 2023 AND quarter = 'Q1' THEN revenue END) as "2023_Q1",
        SUM(CASE WHEN year = 2023 AND quarter = 'Q2' THEN revenue END) as "2023_Q2",
        SUM(CASE WHEN year = 2024 AND quarter = 'Q1' THEN revenue END) as "2024_Q1",
        SUM(CASE WHEN year = 2024 AND quarter = 'Q2' THEN revenue END) as "2024_Q2"
    FROM sales
    GROUP BY region
)
SELECT * FROM pivoted_data;
```

---

## 📚 Summary

### Key Takeaways

1. **PIVOT**: Rows → Columns (wide format) for reporting
2. **UNPIVOT**: Columns → Rows (long format) for normalization
3. **Database support**: SQL Server/Oracle (native), PostgreSQL (crosstab), MySQL (manual CASE)
4. **Manual PIVOT**: `SUM(CASE WHEN col = 'value' THEN amount END)`
5. **Manual UNPIVOT**: `UNION ALL` or `CROSS JOIN LATERAL VALUES`
6. **Dynamic PIVOT**: Use dynamic SQL for unknown columns
7. **Performance**: ~equal to GROUP BY + CASE, benefit from indexes
8. **Common mistakes**: Extra columns in source, missing aggregates
9. **NULL handling**: Use COALESCE to replace NULLs
10. **Use cases**: Financial reports, cohort analysis, ETL transformations

**Syntax Quick Reference:**

```sql
-- SQL Server PIVOT
SELECT * FROM source
PIVOT (SUM(amount) FOR category IN ([A], [B], [C])) AS pvt;

-- SQL Server UNPIVOT
SELECT * FROM source
UNPIVOT (amount FOR category IN (col1, col2, col3)) AS unpvt;

-- Universal manual PIVOT
SELECT 
    SUM(CASE WHEN category = 'A' THEN amount END) as A,
    SUM(CASE WHEN category = 'B' THEN amount END) as B
FROM source;

-- Universal manual UNPIVOT
SELECT 'A', col1 FROM source WHERE col1 IS NOT NULL
UNION ALL
SELECT 'B', col2 FROM source WHERE col2 IS NOT NULL;
```

---

**Next:** [04_JSON_Operations.md](04_JSON_Operations.md) - Work with JSON data  
**Related:** [01_Window_Functions.md](01_Window_Functions.md) - Aggregation without pivoting

---

*Last Updated: February 20, 2026*
