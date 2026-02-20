# 📊 Window Functions - Advanced Analytical Queries

> Window functions perform calculations across rows related to the current row without collapsing them. Master ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD, and aggregates for powerful analytics.

---

## 📖 1. Concept Explanation

### What are Window Functions?

**Window functions** (also called **analytic functions**) compute values across a set of rows related to the current row, without grouping them into a single output row.

**Key Difference from GROUP BY:**
```sql
-- ❌ GROUP BY: Collapses rows
SELECT category, COUNT(*) as count
FROM products
GROUP BY category;
-- Result: 3 rows (one per category)

-- ✅ Window Function: Keeps all rows
SELECT 
    product_name,
    category,
    COUNT(*) OVER (PARTITION BY category) as category_count
FROM products;
-- Result: All product rows, each with category count
```

**Window Function Structure:**
```
FUNCTION() OVER (
    [PARTITION BY columns]  -- Group rows
    [ORDER BY columns]      -- Define order within partition
    [ROWS/RANGE clause]     -- Define frame (window)
)
```

**Categories of Window Functions:**
```
WINDOW FUNCTIONS
│
├── Ranking Functions
│   ├── ROW_NUMBER() - Unique sequential number
│   ├── RANK() - Rank with gaps for ties
│   ├── DENSE_RANK() - Rank without gaps
│   └── NTILE(n) - Divide into n buckets
│
├── Value Functions
│   ├── LAG(col, n) - Previous row value
│   ├── LEAD(col, n) - Next row value
│   ├── FIRST_VALUE() - First in window
│   └── LAST_VALUE() - Last in window
│
└── Aggregate Functions
    ├── SUM() OVER - Running total
    ├── AVG() OVER - Moving average
    ├── COUNT() OVER - Running count
    ├── MIN() OVER - Minimum in window
    └── MAX() OVER - Maximum in window
```

---

## 🧠 2. Why It Matters in Real Systems

### Analytics Without Window Functions (Inefficient)

**❌ Bad: Self-join for row numbering**
```sql
-- Get rank of each product by price (without window functions)
SELECT 
    p1.name,
    p1.price,
    COUNT(p2.id) as rank
FROM products p1
LEFT JOIN products p2 ON p1.price < p2.price OR (p1.price = p2.price AND p1.id >= p2.id)
GROUP BY p1.id, p1.name, p1.price
ORDER BY rank;
-- Slow: O(N²) complexity ❌
-- Time on 100k rows: 30+ seconds
```

**✅ Good: Window function (efficient)**
```sql
SELECT 
    name,
    price,
    ROW_NUMBER() OVER (ORDER BY price DESC) as rank
FROM products
ORDER BY rank;
-- Fast: O(N log N) complexity ✅
-- Time on 100k rows: 0.2 seconds (150x faster)
```

---

## ⚙️ 3. Internal Working

### Window Function Execution Order

```sql
SELECT 
    category,
    product_name,
    price,
    AVG(price) OVER (PARTITION BY category) as avg_price
FROM products
WHERE stock > 0
ORDER BY category, price DESC;

-- SQL Execution Order:
-- 1. FROM products
-- 2. WHERE stock > 0
-- 3. Window function: PARTITION BY, ORDER BY, compute AVG
-- 4. SELECT final columns
-- 5. ORDER BY category, price DESC
```

### Window Frame Concepts

```sql
-- Window frame: Rows used for calculation
SELECT 
    date,
    revenue,
    SUM(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as rolling_3day_sum
FROM daily_sales;

-- Frame for each row:
-- Row 1: [Row1]               → Sum(Row1)
-- Row 2: [Row1, Row2]         → Sum(Row1, Row2)
-- Row 3: [Row1, Row2, Row3]   → Sum(Row1, Row2, Row3)
-- Row 4: [Row2, Row3, Row4]   → Sum(Row2, Row3, Row4)
-- ...
```

---

## ✅ 4. Best Practices

### PARTITION BY for Group Analysis

```sql
-- ✅ Rank products within each category
SELECT 
    category,
    product_name,
    price,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) as price_rank_in_category
FROM products;

-- Without PARTITION BY: Ranks across all products
-- With PARTITION BY: Separate ranking per category ✅
```

### Use Appropriate Ranking Function

```sql
-- Sample data: scores with ties
-- Student | Score
-- Alice   | 95
-- Bob     | 90
-- Charlie | 90
-- David   | 85

-- ROW_NUMBER: Unique numbers (arbitrary for ties)
SELECT name, score,
       ROW_NUMBER() OVER (ORDER BY score DESC) as row_num
FROM students;
-- Alice: 1, Bob: 2, Charlie: 3, David: 4

-- RANK: Gaps after ties
SELECT name, score,
       RANK() OVER (ORDER BY score DESC) as rank
FROM students;
-- Alice: 1, Bob: 2, Charlie: 2, David: 4 (gap at 3)

-- DENSE_RANK: No gaps
SELECT name, score,
       DENSE_RANK() OVER (ORDER BY score DESC) as dense_rank
FROM students;
-- Alice: 1, Bob: 2, Charlie: 2, David: 3 (no gap) ✅
```

### Optimize Window Function Performance

```sql
-- ✅ Add index on PARTITION BY and ORDER BY columns
CREATE INDEX idx_category_price ON products(category, price DESC);

-- Query uses index for partitioning and ordering
SELECT 
    category,
    product_name,
    price,
    RANK() OVER (PARTITION BY category ORDER BY price DESC) as rank
FROM products;
-- Index scan instead of full table scan + sort ✅
```

### Use CTEs for Readability

```sql
-- ✅ Break complex window logic into steps
WITH ranked_products AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) as rn
    FROM products
)
SELECT *
FROM ranked_products
WHERE rn <= 3;  -- Top 3 per category
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Window Function in WHERE Clause

```sql
-- ❌ Error: Can't use window function in WHERE
SELECT product_name, price
FROM products
WHERE ROW_NUMBER() OVER (ORDER BY price) <= 10;
-- ERROR: window functions not allowed in WHERE

-- ✅ Correct: Use CTE or subquery
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY price) as rn
    FROM products
)
SELECT product_name, price
FROM ranked
WHERE rn <= 10;
```

### Mistake 2: Forgetting ORDER BY in Ranking

```sql
-- ❌ Meaningless: No ORDER BY in ranking function
SELECT 
    product_name,
    ROW_NUMBER() OVER (PARTITION BY category) as rn
FROM products;
-- Random numbering ❌

-- ✅ Correct: Always ORDER BY for ranking
SELECT 
    product_name,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) as rn
FROM products;
```

### Mistake 3: LAST_VALUE Frame Issue

```sql
-- ❌ Unexpected: Default frame is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
SELECT 
    date,
    revenue,
    LAST_VALUE(revenue) OVER (ORDER BY date) as last_revenue
FROM daily_sales;
-- Returns current row revenue, not last! ❌

-- ✅ Correct: Specify full frame
SELECT 
    date,
    revenue,
    LAST_VALUE(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as last_revenue
FROM daily_sales;
```

### Mistake 4: Performance on Large Datasets

```sql
-- ❌ Slow: Multiple window functions without index
SELECT 
    product_name,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY price),
    RANK() OVER (PARTITION BY category ORDER BY sales),
    AVG(price) OVER (PARTITION BY brand)
FROM massive_products_table;  -- 10 million rows
-- Multiple sorts without indexes: 60+ seconds ❌

-- ✅ Fast: Add indexes, use WINDOW clause
CREATE INDEX idx_cat_price ON massive_products_table(category, price);
CREATE INDEX idx_cat_sales ON massive_products_table(category, sales);
CREATE INDEX idx_brand ON massive_products_table(brand);

SELECT 
    product_name,
    ROW_NUMBER() OVER w1,
    RANK() OVER w2,
    AVG(price) OVER w3
FROM massive_products_table
WINDOW 
    w1 AS (PARTITION BY category ORDER BY price),
    w2 AS (PARTITION BY category ORDER BY sales),
    w3 AS (PARTITION BY brand);
-- Uses indexes: 2 seconds ✅
```

---

## 🔐 6. Security Considerations

### Row-Level Security with Window Functions

```sql
-- ✅ Window functions respect row-level security
CREATE POLICY tenant_isolation ON orders
    FOR ALL TO app_user
    USING (tenant_id = current_setting('app.tenant_id')::int);

ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- User only sees their tenant's data in window function
SET app.tenant_id = 123;
SELECT 
    order_id,
    ROW_NUMBER() OVER (ORDER BY created_at) as order_sequence
FROM orders;
-- Only tenant 123's orders ✅
```

---

## 🚀 7. Performance Optimization

### Named WINDOW Clause

```sql
-- ❌ Repeated window definition (computed multiple times)
SELECT 
    product_name,
    AVG(price) OVER (PARTITION BY category ORDER BY created_at),
    SUM(sales) OVER (PARTITION BY category ORDER BY created_at),
    COUNT(*) OVER (PARTITION BY category ORDER BY created_at)
FROM products;

-- ✅ Named window (computed once, reused)
SELECT 
    product_name,
    AVG(price) OVER w,
    SUM(sales) OVER w,
    COUNT(*) OVER w
FROM products
WINDOW w AS (PARTITION BY category ORDER BY created_at);
-- 3x faster ✅
```

### Materialized Results for Frequent Queries

```sql
-- ✅ Pre-compute window functions for dashboards
CREATE TABLE product_rankings AS
SELECT 
    product_id,
    product_name,
    category,
    price,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) as price_rank,
    PERCENT_RANK() OVER (PARTITION BY category ORDER BY sales DESC) as sales_percentile
FROM products;

CREATE INDEX idx_category_rank ON product_rankings(category, price_rank);

-- Dashboard queries now instant ✅
SELECT * FROM product_rankings WHERE category = 'Electronics' AND price_rank <= 10;
```

---

## 🧪 8. Examples

### ROW_NUMBER - Unique Sequential Numbers

```sql
-- Assign unique order numbers to each customer's orders
SELECT 
    customer_id,
    order_id,
    order_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as customer_order_number
FROM orders;

-- Result:
-- customer_id | order_id | order_date | customer_order_number
-- 1           | 101      | 2024-01-01 | 1
-- 1           | 102      | 2024-01-05 | 2
-- 1           | 103      | 2024-01-10 | 3
-- 2           | 104      | 2024-01-02 | 1
-- 2           | 105      | 2024-01-07 | 2
```

### RANK vs DENSE_RANK vs ROW_NUMBER

```sql
-- Compare ranking functions with ties
SELECT 
    student_name,
    score,
    ROW_NUMBER() OVER (ORDER BY score DESC) as row_num,
    RANK() OVER (ORDER BY score DESC) as rank,
    DENSE_RANK() OVER (ORDER BY score DESC) as dense_rank
FROM exam_scores
ORDER BY score DESC;

-- Result with ties:
-- Name    | Score | ROW_NUMBER | RANK | DENSE_RANK
-- Alice   | 95    | 1          | 1    | 1
-- Bob     | 90    | 2          | 2    | 2
-- Charlie | 90    | 3          | 2    | 2  ← Same rank
-- David   | 85    | 4          | 4    | 3  ← RANK skips 3, DENSE_RANK doesn't
-- Eve     | 80    | 5          | 5    | 4
```

### LAG and LEAD - Access Previous/Next Rows

```sql
-- Compare each day's revenue to previous day
SELECT 
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date) as prev_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY date) as day_over_day_change,
    LEAD(revenue, 1) OVER (ORDER BY date) as next_day_revenue
FROM daily_sales
ORDER BY date;

-- Result:
-- date       | revenue | prev_day | change | next_day
-- 2024-01-01 | 1000    | NULL     | NULL   | 1200
-- 2024-01-02 | 1200    | 1000     | +200   | 950
-- 2024-01-03 | 950     | 1200     | -250   | 1100
```

### Running Totals and Moving Averages

```sql
-- Running total and 7-day moving average
SELECT 
    date,
    revenue,
    SUM(revenue) OVER (ORDER BY date) as running_total,
    AVG(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7day
FROM daily_sales
ORDER BY date;

-- Result:
-- date       | revenue | running_total | moving_avg_7day
-- 2024-01-01 | 1000    | 1000          | 1000
-- 2024-01-02 | 1200    | 2200          | 1100
-- ...
-- 2024-01-07 | 1100    | 7500          | 1071  ← Average of last 7 days
-- 2024-01-08 | 1050    | 8550          | 1079  ← Window slides
```

### NTILE - Divide into Buckets

```sql
-- Divide customers into quartiles by spending
SELECT 
    customer_id,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent DESC) as spending_quartile
FROM customer_totals;

-- Result:
-- customer_id | total_spent | spending_quartile
-- 123         | 10000       | 1  ← Top 25%
-- 456         | 8500        | 1
-- 789         | 5000        | 2  ← 25-50%
-- 111         | 3000        | 3  ← 50-75%
-- 222         | 1000        | 4  ← Bottom 25%
```

### FIRST_VALUE and LAST_VALUE

```sql
-- Compare each product price to highest and lowest in category
SELECT 
    category,
    product_name,
    price,
    FIRST_VALUE(price) OVER (
        PARTITION BY category 
        ORDER BY price DESC
    ) as highest_price,
    LAST_VALUE(price) OVER (
        PARTITION BY category 
        ORDER BY price DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as lowest_price,
    price - FIRST_VALUE(price) OVER (
        PARTITION BY category 
        ORDER BY price DESC
    ) as diff_from_highest
FROM products;
```

### Percent Rank and Cumulative Distribution

```sql
-- Calculate percentile rank of each employee salary
SELECT 
    employee_name,
    department,
    salary,
    PERCENT_RANK() OVER (PARTITION BY department ORDER BY salary) as salary_percentile,
    CUME_DIST() OVER (PARTITION BY department ORDER BY salary) as cumulative_dist
FROM employees;

-- PERCENT_RANK: (rank - 1) / (total rows - 1)
-- CUME_DIST: (rows <= current) / (total rows)
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: E-Commerce - Top N Products per Category

```sql
-- Get top 3 products by sales in each category
WITH ranked_products AS (
    SELECT 
        category_name,
        product_name,
        units_sold,
        revenue,
        ROW_NUMBER() OVER (
            PARTITION BY category_name 
            ORDER BY revenue DESC
        ) as revenue_rank
    FROM products
    WHERE status = 'active'
)
SELECT 
    category_name,
    product_name,
    units_sold,
    revenue,
    revenue_rank
FROM ranked_products
WHERE revenue_rank <= 3
ORDER BY category_name, revenue_rank;
```

### Case Study 2: Finance - Running Balance

```sql
-- Calculate account balance after each transaction
SELECT 
    account_id,
    transaction_date,
    transaction_id,
    amount,
    transaction_type,
    SUM(
        CASE 
            WHEN transaction_type = 'credit' THEN amount
            WHEN transaction_type = 'debit' THEN -amount
        END
    ) OVER (
        PARTITION BY account_id 
        ORDER BY transaction_date, transaction_id
    ) as running_balance
FROM transactions
ORDER BY account_id, transaction_date;
```

### Case Study 3: Analytics - Cohort Retention

```sql
-- Calculate user retention by signup cohort
WITH user_cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', signup_date) as cohort_month
    FROM users
),
user_activity AS (
    SELECT 
        uc.user_id,
        uc.cohort_month,
        DATE_TRUNC('month', o.order_date) as activity_month,
        EXTRACT(MONTH FROM AGE(o.order_date, uc.cohort_month)) as months_since_signup
    FROM user_cohorts uc
    JOIN orders o ON uc.user_id = o.user_id
)
SELECT 
    cohort_month,
    months_since_signup,
    COUNT(DISTINCT user_id) as active_users,
    FIRST_VALUE(COUNT(DISTINCT user_id)) OVER (
        PARTITION BY cohort_month 
        ORDER BY months_since_signup
    ) as cohort_size,
    ROUND(
        100.0 * COUNT(DISTINCT user_id) / 
        FIRST_VALUE(COUNT(DISTINCT user_id)) OVER (
            PARTITION BY cohort_month 
            ORDER BY months_since_signup
        ),
        2
    ) as retention_rate
FROM user_activity
GROUP BY cohort_month, months_since_signup
ORDER BY cohort_month, months_since_signup;
```

### Case Study 4: SaaS - MRR Growth Analysis

```sql
-- Calculate Monthly Recurring Revenue (MRR) changes
WITH monthly_mrr AS (
    SELECT 
        DATE_TRUNC('month', date) as month,
        SUM(mrr) as total_mrr
    FROM subscription_snapshots
    GROUP BY DATE_TRUNC('month', date)
)
SELECT 
    month,
    total_mrr,
    LAG(total_mrr) OVER (ORDER BY month) as prev_month_mrr,
    total_mrr - LAG(total_mrr) OVER (ORDER BY month) as mrr_change,
    ROUND(
        100.0 * (total_mrr - LAG(total_mrr) OVER (ORDER BY month)) / 
        LAG(total_mrr) OVER (ORDER BY month),
        2
    ) as mrr_growth_rate
FROM monthly_mrr
ORDER BY month;
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between GROUP BY and window functions?

**Answer:**

| Aspect | GROUP BY | Window Functions |
|--------|----------|------------------|
| **Output rows** | Collapsed (one per group) | All input rows preserved |
| **Aggregates** | Returns aggregate only | Returns aggregate + detail |
| **Use case** | Summary reports | Running totals, rankings |

```sql
-- GROUP BY: 3 rows (one per category)
SELECT category, COUNT(*) as count
FROM products
GROUP BY category;

-- Window Function: All rows (with count per category)
SELECT product_name, category,
       COUNT(*) OVER (PARTITION BY category) as count
FROM products;
```

---

### Q2: What's the difference between RANK, DENSE_RANK, and ROW_NUMBER?

**Answer:**

- **ROW_NUMBER()**: Sequential unique numbers (1, 2, 3, 4, 5...)
- **RANK()**: Same rank for ties, gaps after (1, 2, 2, 4, 5...)
- **DENSE_RANK()**: Same rank for ties, no gaps (1, 2, 2, 3, 4...)

```sql
-- Data: scores 95, 90, 90, 85

ROW_NUMBER(): 1, 2, 3, 4  ← Always unique
RANK():       1, 2, 2, 4  ← Gap at 3
DENSE_RANK(): 1, 2, 2, 3  ← No gap
```

**Use DENSE_RANK when:** You want continuous ranking (leaderboards)  
**Use RANK when:** You want to show "true" position with gaps (competitions)  
**Use ROW_NUMBER when:** You need unique IDs (pagination)

---

### Q3: How do LAG and LEAD work?

**Answer:** 
- **LAG(col, n)**: Access value from **n rows before** current row
- **LEAD(col, n)**: Access value from **n rows after** current row

```sql
SELECT 
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date) as yesterday,
    LEAD(revenue, 1) OVER (ORDER BY date) as tomorrow
FROM daily_sales;

-- Row for 2024-01-15:
-- yesterday = revenue from 2024-01-14
-- tomorrow = revenue from 2024-01-16
```

**Use cases:**
- Compare today vs yesterday (growth rates)
- Detect changes (price changes, status changes)
- Fill gaps (COALESCE with LAG for missing data)

---

### Q4: What are window frames (ROWS vs RANGE)?

**Answer:**

**ROWS**: Physical offset (count rows)  
**RANGE**: Logical offset (value-based)

```sql
-- ROWS: Count rows
SELECT 
    date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg_3day
FROM daily_sales;
-- Always uses exactly 3 rows (if available)

-- RANGE: Value-based
SELECT 
    date,
    revenue,
    SUM(revenue) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
    ) as sum_last_7days
FROM daily_sales;
-- Uses all rows within 7 days of current date
```

**Default frame:**
- With ORDER BY: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
- Without ORDER BY: `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`

---

### Q5: How do you optimize slow window function queries?

**Answer:**

**1. Add indexes on PARTITION BY and ORDER BY columns:**
```sql
CREATE INDEX idx_cat_price ON products(category, price DESC);
```

**2. Use named WINDOW clause to avoid recomputation:**
```sql
SELECT AVG(price) OVER w, SUM(sales) OVER w
FROM products
WINDOW w AS (PARTITION BY category ORDER BY created_at);
```

**3. Materialize for frequent queries:**
```sql
CREATE TABLE product_rankings AS
SELECT *, ROW_NUMBER() OVER (...) as rank
FROM products;
```

**4. Reduce window frame size:**
```sql
-- ❌ Slow: Full partition scan
LAST_VALUE(x) OVER (PARTITION BY cat ORDER BY date)

-- ✅ Fast: Limited frame
LAST_VALUE(x) OVER (
    PARTITION BY cat ORDER BY date
    ROWS BETWEEN 100 PRECEDING AND CURRENT ROW
)
```

**5. Use EXPLAIN ANALYZE to identify bottlenecks:**
```sql
EXPLAIN ANALYZE
SELECT ROW_NUMBER() OVER (PARTITION BY category ORDER BY price)
FROM products;
-- Look for "Sort" and "WindowAgg" nodes
```

---

## 🧩 11. Design Patterns

### Pattern 1: Pagination with ROW_NUMBER

```sql
-- Efficient pagination (better than OFFSET)
WITH numbered_results AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (ORDER BY created_at DESC) as rn
    FROM products
    WHERE category = 'Electronics'
)
SELECT *
FROM numbered_results
WHERE rn BETWEEN 21 AND 30;  -- Page 3 (10 per page)
```

### Pattern 2: Gap and Island Detection

```sql
-- Find consecutive sequences (e.g., consecutive login days)
WITH login_numbered AS (
    SELECT 
        user_id,
        login_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) as rn,
        login_date - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) * INTERVAL '1 day' as island_id
    FROM user_logins
)
SELECT 
    user_id,
    MIN(login_date) as streak_start,
    MAX(login_date) as streak_end,
    COUNT(*) as streak_length
FROM login_numbered
GROUP BY user_id, island_id
HAVING COUNT(*) >= 7  -- Streaks of 7+ days
ORDER BY streak_length DESC;
```

### Pattern 3: Running Difference

```sql
-- Track inventory changes
SELECT 
    product_id,
    transaction_date,
    quantity_change,
    SUM(quantity_change) OVER (
        PARTITION BY product_id 
        ORDER BY transaction_date
    ) as current_stock
FROM inventory_transactions
ORDER BY product_id, transaction_date;
```

---

## 📚 Summary

### Key Takeaways

1. **Window functions preserve rows** unlike GROUP BY
2. **PARTITION BY** creates groups, **ORDER BY** defines order
3. **ROW_NUMBER** = unique, **RANK** = gaps, **DENSE_RANK** = no gaps
4. **LAG/LEAD** access previous/next rows (compare periods)
5. **Running totals**: `SUM() OVER (ORDER BY date)`
6. **Moving averages**: `AVG() OVER (ORDER BY date ROWS BETWEEN n PRECEDING AND CURRENT ROW)`
7. **Frame default**: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
8. **Optimize**: Index PARTITION BY and ORDER BY columns
9. **Named WINDOW**: Reuse window definitions
10. **Can't use in WHERE**: Use CTE or subquery instead

**Common Patterns:**
- Rankings: ROW_NUMBER, RANK, DENSE_RANK
- Comparisons: LAG, LEAD
- Aggregates: SUM, AVG, COUNT with OVER
- Distributions: NTILE, PERCENT_RANK

---

**Next:** [02_Recursive_Queries.md](02_Recursive_Queries.md) - Hierarchical data traversal  
**Related:** [../03_SQL_Core/06_CTEs.md](../03_SQL_Core/06_CTEs.md) - Common Table Expressions

---

*Last Updated: February 20, 2026*
