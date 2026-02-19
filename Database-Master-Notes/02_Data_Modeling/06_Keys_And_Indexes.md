# 🔑 Keys and Indexes - Database Performance & Integrity

> Keys identify records uniquely while indexes accelerate queries. Understanding the difference and choosing the right strategy is critical for database performance and data integrity.

---

## 📖 1. Concept Explanation

### What are Keys?

**Keys** are columns (or combinations of columns) that uniquely identify rows in a table and establish relationships between tables.

### What are Indexes?

**Indexes** are data structures that improve query performance by creating fast lookup paths to data, similar to a book's index.

**Key vs Index:**
- **Key**: Logical concept (uniqueness, relationships)
- **Index**: Physical structure (performance optimization)
- **Relationship**: Keys create indexes automatically

---

### Types of Keys

```
DATABASE KEYS
│
├── PRIMARY KEY
│   └── Uniquely identifies each row (one per table)
│
├── FOREIGN KEY
│   └── References primary key in another table
│
├── CANDIDATE KEY
│   └── Column(s) that could be primary key
│
├── ALTERNATE KEY
│   └── Candidate keys not chosen as primary
│
├── COMPOSITE KEY
│   └── Key composed of multiple columns
│
├── SURROGATE KEY
│   └── Artificial key (auto-increment, UUID)
│
└── NATURAL KEY
    └── Real-world unique identifier (SSN, email)
```

---

### Types of Indexes

```
DATABASE INDEXES
│
├── B-TREE INDEX (default)
│   ├── Range queries
│   ├── Sorting
│   └── Equality lookups
│
├── HASH INDEX
│   ├── Exact equality only
│   └── Very fast lookups
│
├── BITMAP INDEX
│   ├── Low cardinality columns
│   └── Data warehouses
│
├── FULL-TEXT INDEX
│   ├── Text search
│   └── GIN/GiST (PostgreSQL)
│
├── SPATIAL INDEX
│   ├── Geographic data
│   └── R-tree structure
│
├── COVERING INDEX
│   ├── Includes non-key columns
│   └── Index-only scans
│
└── PARTIAL INDEX
    ├── Filtered subset
    └── Conditional indexing
```

---

## 🧠 2. Why It Matters in Real Systems

### Without Proper Keys - Data Corruption

```sql
-- No primary key
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    total DECIMAL(10,2)
);

-- Duplicate orders possible!
INSERT INTO orders VALUES (1, 100, 50.00);
INSERT INTO orders VALUES (1, 100, 50.00);  -- ❌ Duplicate accepted

-- Can't identify specific order
UPDATE orders SET total = 60.00 WHERE order_id = 1;  -- Updates both! ❌
```

### With Primary Key - Data Integrity

```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    total DECIMAL(10,2)
);

-- Duplicates rejected
INSERT INTO orders VALUES (1, 100, 50.00);  -- ✅
INSERT INTO orders VALUES (1, 100, 50.00);  -- ❌ ERROR: duplicate key
```

---

### Without Indexes - Slow Queries

```sql
-- No index on customer_id
SELECT * FROM orders WHERE customer_id = 1000;

-- Execution plan: FULL TABLE SCAN
-- Scans all 10 million rows! ❌
-- Query time: 5 seconds
```

### With Index - Fast Queries

```sql
CREATE INDEX idx_customer_id ON orders (customer_id);

SELECT * FROM orders WHERE customer_id = 1000;

-- Execution plan: INDEX SEEK
-- Reads ~10 rows from index ✅
-- Query time: 0.01 seconds (500x faster!)
```

---

### Real-World Impact

| Scenario | Without Keys/Indexes | With Keys/Indexes |
|----------|---------------------|-------------------|
| **User lookup by email** | 3s (full scan) | 0.003s (index seek) |
| **Duplicate data** | Accepted | Rejected by key |
| **Orphaned records** | Possible | Prevented by FK |
| **Join performance** | O(n×m) | O(log n + m) |
| **Sort operations** | Slow heap sort | Fast index scan |

---

## ⚙️ 3. Internal Working

### Primary Key Internals

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY
);

-- What database creates:
-- 1. UNIQUE constraint on id
-- 2. NOT NULL constraint on id  
-- 3. Clustered or non-clustered B-tree index
-- 4. Special metadata marking it as PK
```

**Clustered vs Non-Clustered:**

```
CLUSTERED INDEX (InnoDB, SQL Server default for PK):
└─ Table data physically sorted by PK
└─ Fast range scans on PK
└─ Only ONE per table
└─ Leaf nodes contain actual rows

NON-CLUSTERED INDEX (PostgreSQL):
└─ Separate structure from table data
└─ Multiple allowed per table
└─ Leaf nodes contain pointers to rows
```

---

### B-Tree Index Structure

```
       [50]              ← Root node
      /    \
   [25]    [75]          ← Internal nodes
   /  \    /   \
[10][40][60][90]         ← Leaf nodes (contain data or pointers)
```

**Search for value 60:**
1. Start at root: 60 > 50, go right
2. Internal node: 60 < 75, go left
3. Leaf node: Found 60 ✅
4. Total comparisons: 3 (vs 10 million without index!)

**Time Complexity:**
- Search: O(log n)
- Insert: O(log n)
- Delete: O(log n)
- Range scan: O(log n + k) where k = rows returned

---

### Composite Index Structure

```sql
CREATE INDEX idx_name_age ON users (last_name, first_name);

-- Index structure:
(Smith, Alice)  → Row pointer
(Smith, Bob)    → Row pointer
(Smith, Charlie)→ Row pointer
(Wilson, Alice) → Row pointer
(Wilson, Zoe)   → Row pointer
```

**Left-prefix rule:**
```sql
-- Uses index (left-most column):
SELECT * FROM users WHERE last_name = 'Smith';  -- ✅

-- Uses index (both columns):
SELECT * FROM users WHERE last_name = 'Smith' AND first_name = 'Alice';  -- ✅

-- Cannot use index (skips left-most column):
SELECT * FROM users WHERE first_name = 'Alice';  -- ❌ Full scan
```

---

### Foreign Key Internals

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- ON INSERT INTO orders:
--   1. Check if customer_id exists in customers.id
--   2. If not found → ERROR
--   3. If found → Insert order

-- ON DELETE FROM customers WHERE id = X:
--   1. Check if any orders reference customer X
--   2. If ON DELETE RESTRICT → ERROR
--   3. If ON DELETE CASCADE → Delete orders too
--   4. If ON DELETE SET NULL → Set customer_id to NULL
```

**Performance consideration:**
```sql
-- Without index on FK column:
DELETE FROM customers WHERE id = 1000;
-- Database scans ALL orders to check references! ❌ Slow

-- With index on FK column:
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
-- Fast lookup of referencing orders ✅
```

---

## ✅ 4. Best Practices

### Always Index Foreign Keys

```sql
-- ✅ Good: Index on FK column
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE INDEX idx_orders_customer_id ON orders (customer_id);

-- Fast joins:
SELECT * FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.email = 'user@example.com';
```

---

### Choose Surrogate vs Natural Keys Carefully

**Surrogate Key (Auto-increment):**
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,  -- Surrogate key
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP
);

-- Pros:
-- - Stable (never changes)
-- - Small footprint (8 bytes)
-- - Fast joins
-- - No business logic dependency

-- Cons:
-- - Not meaningful to humans
-- - Need to expose ID or UUID publicly
```

**Natural Key:**
```sql
CREATE TABLE countries (
    country_code CHAR(2) PRIMARY KEY,  -- Natural key (ISO 3166)
    name VARCHAR(100)
);

-- Pros:
-- - Meaningful (US, CA, GB)
-- - No need for additional unique constraint

-- Cons:
-- - Can change (rare but happens)
-- - Larger in child tables
```

**Recommendation:** Use surrogates for most tables, natural keys for static reference data.

---

### Composite Indexes - Order Matters

```sql
-- Index on (status, created_at)
CREATE INDEX idx_status_date ON orders (status, created_at);

-- ✅ Uses index (left-prefix rule):
SELECT * FROM orders WHERE status = 'pending';
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-01-01';

-- ❌ Cannot use index (skips first column):
SELECT * FROM orders WHERE created_at > '2024-01-01';

-- Solution: Create separate index or reverse order
CREATE INDEX idx_date_status ON orders (created_at, status);
```

**Rule of thumb:** Put most selective column first (column with highest cardinality).

---

### Use Covering Indexes for Frequent Queries

```sql
-- Frequent query:
SELECT id, status, created_at FROM orders WHERE customer_id = 1000;

-- ❌ Non-covering index:
CREATE INDEX idx_customer ON orders (customer_id);
-- Execution: Index seek + table lookup for status, created_at

-- ✅ Covering index (includes all needed columns):
CREATE INDEX idx_customer_covering ON orders (customer_id) 
INCLUDE (status, created_at);
-- Or: CREATE INDEX idx_customer_covering ON orders (customer_id, status, created_at);

-- Execution: Index-only scan (no table access needed!) ⚡
```

---

### Partial Indexes for Filtered Queries

```sql
-- Only 5% of orders are 'pending'
-- 95% are 'completed'

-- ❌ Bad: Index all rows
CREATE INDEX idx_status ON orders (status);
-- Index size: 500 MB

-- ✅ Good: Index only pending orders
CREATE INDEX idx_pending_orders ON orders (status)
WHERE status = 'pending';
-- Index size: 25 MB (20x smaller!)

-- Query:
SELECT * FROM orders WHERE status = 'pending';
-- Uses partial index ✅
```

---

### UUID vs Auto-Increment Trade-offs

**Auto-Increment PRIMARY KEY:**
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT
);

-- Pros:
-- - Sequential (B-tree friendly, good insert performance)
-- - Small (8 bytes)
-- - Easy to debug (1, 2, 3...)

-- Cons:
-- - Enumeration attacks (guessable IDs)
-- - Coordination needed in distributed systems
-- - Exposes system scale (ID 1,000,000 = 1M users)
```

**UUID PRIMARY KEY:**
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid()
);

-- Pros:
-- - Globally unique (no coordination)
-- - Non-guessable (security)
-- - Distributed-friendly

-- Cons:
-- - Large (16 bytes)
-- - Random (poor B-tree locality, slower inserts)
-- - Hard to debug
```

**Best of both worlds:**
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,        -- Internal, fast joins
    public_id UUID UNIQUE DEFAULT gen_random_uuid()  -- External, secure
);

-- Use id for joins, expose public_id to users
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Over-Indexing

```sql
-- ❌ Bad: Too many indexes
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(200),
    category VARCHAR(50),
    price DECIMAL(10,2),
    created_at TIMESTAMP
);

CREATE INDEX idx_name ON products(name);
CREATE INDEX idx_category ON products(category);
CREATE INDEX idx_price ON products(price);
CREATE INDEX idx_created ON products(created_at);
CREATE INDEX idx_name_category ON products(name, category);
CREATE INDEX idx_category_price ON products(category, price);
CREATE INDEX idx_all ON products(name, category, price, created_at);

-- Problems:
-- - Every INSERT/UPDATE must maintain 8 indexes! ❌
-- - Slower writes
-- - More disk space
-- - Query optimizer confusion (which index to use?)
```

**Fix:**
```sql
-- ✅ Good: Strategic indexes only
CREATE INDEX idx_category ON products(category);  -- Most common filter
CREATE INDEX idx_category_price ON products(category, price);  -- Covers most queries
-- Only 2-3 indexes needed for most use cases
```

---

### Mistake 2: Not Indexing Foreign Keys

```sql
-- ❌ Bad: No index on FK
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- DELETE customer:
DELETE FROM customers WHERE id = 1000;
-- Scans ALL orders to check references! ❌
-- With 10M orders = 5-10 seconds

-- JOIN query:
SELECT * FROM customers c JOIN orders o ON c.id = o.customer_id;
-- Nested loop join, very slow! ❌
```

**Fix:**
```sql
-- ✅ Good: Always index FKs
CREATE INDEX idx_orders_customer ON orders(customer_id);
-- Now deletes and joins are fast ✅
```

---

### Mistake 3: Redundant Indexes

```sql
-- ❌ Bad: Redundant indexes
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_email_name ON users(email, name);

-- idx_email is redundant! idx_email_name can be used for:
-- - WHERE email = ?
-- - WHERE email = ? AND name = ?
-- (Left-prefix rule)

-- Only need idx_email_name ✅
```

---

### Mistake 4: Wrong Composite Index Order

```sql
-- Query pattern: Most users filter by status, some add date range
SELECT * FROM orders WHERE status = 'pending';
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-01-01';

-- ❌ Bad: Wrong order
CREATE INDEX idx_date_status ON orders(created_at, status);
-- First query cannot use index! ❌

-- ✅ Good: Status first (left-prefix rule)
CREATE INDEX idx_status_date ON orders(status, created_at);
-- Both queries use index ✅
```

---

### Mistake 5: Using Functions in Indexed Columns

```sql
CREATE INDEX idx_email ON users(email);

-- ❌ Bad: Function prevents index usage
SELECT * FROM users WHERE LOWER(email) = 'user@test.com';
-- Full table scan! ❌

-- ✅ Good: Functional index
CREATE INDEX idx_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'user@test.com';
-- Uses index ✅

-- Or normalize data:
UPDATE users SET email = LOWER(email);
SELECT * FROM users WHERE email = 'user@test.com';
```

---

## 🔐 6. Security Considerations

### Prevent Enumeration Attacks

```sql
-- ❌ Sequential IDs expose information
CREATE TABLE users (
    id SERIAL PRIMARY KEY  -- 1, 2, 3, 4...
);
-- Attacker can enumerate: /api/users/1, /api/users/2, etc.

-- ✅ Use UUIDs for public-facing IDs
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    public_id UUID UNIQUE DEFAULT gen_random_uuid()
);
-- Public URL: /api/users/550e8400-e29b-41d4-a716-446655440000
```

---

### Index Security-Sensitive Columns Carefully

```sql
-- Sensitive column with unique index
CREATE TABLE users (
    ssn CHAR(11) UNIQUE  -- Social Security Number
);

-- Index can leak information via timing attacks!
-- Query time varies based on whether SSN exists (index hit vs miss)

-- Solution: Hash + salt before indexing
CREATE TABLE users (
    ssn_hash CHAR(64) UNIQUE  -- SHA-256 hash
);
```

---

### Prevent SQL Injection via Keys

```sql
-- ❌ Vulnerable to SQL injection
sql = "SELECT * FROM users WHERE id = " + user_input;

-- ✅ Use parameterized queries
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,));

-- Primary key type enforcement provides additional protection:
CREATE TABLE users (
    id BIGINT PRIMARY KEY  -- Only accepts integers
);
```

---

## 🚀 7. Performance Optimization Techniques

### Index Hints for Query Optimizer

```sql
-- Force specific index usage (MySQL)
SELECT * FROM orders USE INDEX (idx_customer_id)
WHERE customer_id = 1000;

-- Ignore problematic index
SELECT * FROM orders IGNORE INDEX (idx_slow)
WHERE created_at > '2024-01-01';

-- PostgreSQL:
SET enable_seqscan = OFF;  -- Force index usage
```

---

### Clustered Index Optimization

```sql
-- InnoDB/SQL Server: Clustered PK
CREATE TABLE logs (
    id BIGINT PRIMARY KEY,  -- Sequential writes, fast!
    timestamp TIMESTAMP,
    message TEXT
);

-- ❌ Bad: UUID primary key (random writes)
CREATE TABLE logs (
    id UUID PRIMARY KEY,  -- Random page splits, slow! ❌
    timestamp TIMESTAMP,
    message TEXT
);

-- ✅ Better: Sequential ID + UUID
CREATE TABLE logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    public_id UUID UNIQUE,
    timestamp TIMESTAMP,
    message TEXT
);
```

---

### Index Maintenance

```sql
-- Check index usage (PostgreSQL)
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Never used!
ORDER BY pg_relation_size(indexrelid) DESC;

-- Drop unused indexes:
DROP INDEX idx_unused;

-- Rebuild fragmented indexes (MySQL):
OPTIMIZE TABLE large_table;

-- PostgreSQL:
REINDEX INDEX idx_name;
```

---

### Partial Index for Active Records

```sql
-- Only index active users (common query pattern)
CREATE INDEX idx_active_users ON users (email)
WHERE deleted_at IS NULL;

-- Much smaller index, faster queries!
-- 10M users, 9M active = 90% space savings
```

---

## 🧪 8. Examples

### E-Commerce Database Keys & Indexes

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_email (email),  -- Login queries
    INDEX idx_created (created_at)  -- Analytics
);

CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    category_id INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    
    INDEX idx_sku (sku),
    INDEX idx_category (category_id, is_active),  -- Listing pages
    INDEX idx_active_price (is_active, price) WHERE is_active = TRUE,  -- Partial index
    FOREIGN KEY (category_id) REFERENCES categories(id)
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    status ENUM('pending', 'paid', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(12,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_customer (customer_id, created_at),  -- Customer order history
    INDEX idx_status_date (status, created_at),    -- Admin dashboard
    INDEX idx_created (created_at DESC),           -- Recent orders
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE TABLE order_items (
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    
    PRIMARY KEY (order_id, product_id),  -- Composite PK
    INDEX idx_product (product_id),       -- Product sales analytics
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

---

### SaaS Multi-Tenant System

```sql
CREATE TABLE tenants (
    id INT PRIMARY KEY AUTO_INCREMENT,
    slug VARCHAR(50) UNIQUE NOT NULL,  -- subdomain
    plan ENUM('free', 'pro', 'enterprise') DEFAULT 'free',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_slug (slug)  -- Routing by subdomain
);

CREATE TABLE tenant_users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    email VARCHAR(255) NOT NULL,
    role ENUM('owner', 'admin', 'member') DEFAULT 'member',
    is_active BOOLEAN DEFAULT TRUE,
    
    UNIQUE KEY uk_tenant_email (tenant_id, email),  -- Email unique per tenant
    INDEX idx_tenant_active (tenant_id, is_active),  -- Most queries filter by tenant
    FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE
);

CREATE TABLE documents (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    user_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- All queries include tenant_id (row isolation)
    INDEX idx_tenant_created (tenant_id, created_at DESC),
    INDEX idx_tenant_user (tenant_id, user_id),
    FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES tenant_users(id)
);
```

---

### Time-Series Logging System

```sql
CREATE TABLE logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
    level ENUM('DEBUG', 'INFO', 'WARN', 'ERROR', 'FATAL') NOT NULL,
    service VARCHAR(50) NOT NULL,
    message TEXT NOT NULL,
    
    -- Time-based queries (most recent logs)
    INDEX idx_timestamp_level (timestamp DESC, level),
    INDEX idx_service_timestamp (service, timestamp DESC),
    
    -- Partial index for errors only
    INDEX idx_errors (timestamp DESC, service) WHERE level IN ('ERROR', 'FATAL')
) PARTITION BY RANGE (YEAR(timestamp) * 100 + MONTH(timestamp)) (
    PARTITION p_2024_01 VALUES LESS THAN (202402),
    PARTITION p_2024_02 VALUES LESS THAN (202403),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Twitter - Snowflake IDs

**Problem:** Need globally unique IDs at massive scale (thousands per second).

**Solution:** 64-bit Snowflake IDs as PRIMARY KEY

```sql
-- Twitter's Snowflake ID structure:
-- 41 bits: Timestamp (milliseconds)
-- 10 bits: Machine ID
-- 12 bits: Sequence number

CREATE TABLE tweets (
    id BIGINT PRIMARY KEY,  -- Snowflake ID, not AUTO_INCREMENT
    user_id BIGINT NOT NULL,
    text VARCHAR(280),
    created_at TIMESTAMP NOT NULL,
    
    INDEX idx_user_created (user_id, created_at DESC),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Benefits:
-- - Roughly time-ordered (good for B-tree)
-- - Generated without database coordination
-- - Unique across all datacenters
-- - Fast lookups and joins
```

---

### Case Study 2: GitHub - Composite Keys for Repositories

**Problem:** Repository names unique per user/org, but not globally.

**Solution:** Composite primary key

```sql
CREATE TABLE repositories (
    owner_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    is_private BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (owner_id, name),  -- Composite PK
    INDEX idx_owner_created (owner_id, created_at DESC),
    INDEX idx_public (is_private, created_at DESC) WHERE is_private = FALSE,
    FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE CASCADE
);

-- URLs: github.com/owner/repo-name
-- Both parts needed for uniqueness
```

---

### Case Study 3: Slack - Covering Indexes for Message Search

**Problem:** Message search queries need ID, timestamp, and channel without full table access.

**Solution:** Covering index

```sql
CREATE TABLE messages (
    id BIGINT PRIMARY KEY,
    channel_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Covering index for most queries
    INDEX idx_channel_covering (channel_id, created_at DESC) 
        INCLUDE (user_id, id),
    
    FOREIGN KEY (channel_id) REFERENCES channels(id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Query uses index-only scan:
SELECT id, user_id, created_at 
FROM messages 
WHERE channel_id = 123 
ORDER BY created_at DESC 
LIMIT 50;
```

---

### Case Study 4: Stripe - UUID for Idempotency

**Problem:** Prevent duplicate charges from network retries.

**Solution:** UUID as unique constraint, BIGINT as PK

```sql
CREATE TABLE payments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    idempotency_key UUID UNIQUE NOT NULL,  -- Client-provided
    customer_id BIGINT NOT NULL,
    amount DECIMAL(19,4) NOT NULL,
    currency CHAR(3) NOT NULL,
    status ENUM('pending', 'succeeded', 'failed'),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_idempotency (idempotency_key),  -- Deduplication
    INDEX idx_customer (customer_id, created_at DESC),
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- Client retries with same UUID:
-- First call: INSERT succeeds
-- Retries: UNIQUE violation, return original payment
```

---

### Case Study 5: Instagram - Partitioned by User ID

**Problem:** Billions of photos, need fast user-specific queries.

**Solution:** Composite PRIMARY KEY with user_id first, range partitioning

```sql
CREATE TABLE photos (
    user_id BIGINT NOT NULL,
    photo_id BIGINT NOT NULL,
    url VARCHAR(255) NOT NULL,
    caption TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (user_id, photo_id),  -- Composite PK
    INDEX idx_created (user_id, created_at DESC)
) PARTITION BY HASH(user_id) PARTITIONS 1000;

-- Query for user's photos hits only one partition:
SELECT * FROM photos 
WHERE user_id = 12345 
ORDER BY created_at DESC 
LIMIT 20;
-- Partition pruning: Only scans 1/1000 of table ✅
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between PRIMARY KEY and UNIQUE KEY?

**Answer:**

| Aspect | PRIMARY KEY | UNIQUE KEY |
|--------|------------|------------|
| **NULL allowed** | No | Yes |
| **Per table** | One only | Multiple allowed |
| **Clustered** | Often (InnoDB, SQL Server) | No |
| **Foreign key target** | Default | Possible but rare |
| **Purpose** | Row identification | Alternate keys |

**Example:**
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,       -- One PK, NOT NULL
    email VARCHAR(255) UNIQUE,   -- Multiple UNIQUEs ok, NULL allowed
    username VARCHAR(50) UNIQUE
);
```

---

### Q2: Should every table have a PRIMARY KEY?

**Answer:** **Yes, always.**

**Reasons:**
1. **Row identification**: Need way to reference specific rows
2. **Performance**: Most databases optimize for PK lookups
3. **Foreign keys**: Other tables need something to reference
4. **Replication**: Many tools require PK for conflict resolution
5. **ORM requirements**: Most ORMs expect PK

**Even for junction tables:**
```sql
-- ✅ Good: Composite PK
CREATE TABLE student_courses (
    student_id BIGINT,
    course_id BIGINT,
    enrolled_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (student_id, course_id)
);
```

---

### Q3: When should you use composite indexes?

**Answer:** Use composite indexes when queries filter/sort by multiple columns together.

**Example:**
```sql
-- Query pattern:
SELECT * FROM orders 
WHERE customer_id = 1000 
ORDER BY created_at DESC;

-- ✅ Composite index handles both:
CREATE INDEX idx_customer_date ON orders (customer_id, created_at DESC);

-- Single indexes would be less efficient:
CREATE INDEX idx_customer ON orders(customer_id);  -- Doesn't help with sort
CREATE INDEX idx_date ON orders(created_at);       -- Doesn't help with filter
```

**Rule:** Put most selective (filtered) columns first, then sort columns.

---

### Q4: How do you decide between AUTO_INCREMENT and UUID?

**Answer:**

**Choose AUTO_INCREMENT when:**
- Single database server
- Sequential access patterns
- Need small keys (8 bytes vs 16 bytes)
- Fast inserts critical
- IDs not exposed publicly

**Choose UUID when:**
- Distributed systems (multiple writers)
- Security (non-guessable IDs)
- Offline ID generation needed
- Merging data from multiple sources

**Best of both:**
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,  -- Internal use
    public_id UUID UNIQUE DEFAULT gen_random_uuid()  -- External use
);
```

---

### Q5: What is a covering index and when should you use it?

**Answer:**

**Covering index** contains all columns needed by a query, enabling index-only scans (no table access).

```sql
-- Query:
SELECT order_id, status, created_at 
FROM orders 
WHERE customer_id = 1000;

-- ❌ Non-covering: Needs table lookup
CREATE INDEX idx_customer ON orders(customer_id);

-- ✅ Covering: All data in index
CREATE INDEX idx_customer_covering 
ON orders(customer_id) INCLUDE (status, created_at);

-- Or (older syntax):
CREATE INDEX idx_customer_covering 
ON orders(customer_id, status, created_at);
```

**Use when:**
- Query pattern is very common
- Includes only a few columns (not SELECT *)
- Read-heavy workload (covering indexes slow writes)

---

### Q6: How does index cardinality affect performance?

**Answer:**

**Cardinality** = number of distinct values.

**High cardinality** (good for indexing):
- email, username, order_id
- Almost all values unique
- Index very selective

**Low cardinality** (poor for indexing):
- gender (M/F), boolean flags, status with few values
- Many duplicate values
- Index not selective

**Example:**
```sql
-- ❌ Bad: Low cardinality index
CREATE INDEX idx_gender ON users(gender);  -- Only 2-3 values
SELECT * FROM users WHERE gender = 'F';    -- Returns 50% of rows!

-- ✅ Good: High cardinality index
CREATE INDEX idx_email ON users(email);    -- Unique values
SELECT * FROM users WHERE email = 'user@test.com';  -- Returns 1 row

-- ✅ Better: Composite index with high cardinality first
CREATE INDEX idx_gender_created ON users(created_at, gender);
SELECT * FROM users WHERE created_at > '2024-01-01' AND gender = 'F';
```

---

### Q7: What's the difference between clustered and non-clustered indexes?

**Answer:**

**Clustered Index:**
- Table data physically sorted by index key
- Only ONE per table
- Leaf nodes contain actual data rows
- Example: InnoDB PRIMARY KEY

**Non-Clustered Index:**
- Separate structure from table data
- Multiple allowed per table
- Leaf nodes contain pointers to rows
- Example: All indexes in PostgreSQL

**Visual:**
```
CLUSTERED (InnoDB):
Table IS the index
[1][Row 1 data]
[2][Row 2 data]
[3][Row 3 data]

NON-CLUSTERED (PostgreSQL):
Table separate from index
Index: [1]→pointer, [2]→pointer, [3]→pointer
Table: [Row 1 data][Row 2 data][Row 3 data]
```

**Performance impact:**
- Clustered: Range scans on PK very fast (data contiguous)
- Non-Clustered: Every index access requires table lookup (2 IOs)

---

### Q8: When should you use partial indexes?

**Answer:**

**Partial indexes** index only rows matching a condition.

**Use when:**
- Querying only a small subset of rows
- Condition is frequently used
- Want smaller, faster index

**Examples:**
```sql
-- Only active users (5% of total)
CREATE INDEX idx_active ON users(email) WHERE is_active = TRUE;

-- Only pending orders (2% of total)
CREATE INDEX idx_pending ON orders(created_at DESC) 
WHERE status = 'pending';

-- Only non-deleted records
CREATE INDEX idx_active_users ON users(username) 
WHERE deleted_at IS NULL;

-- Benefits:
-- - 95% smaller index
-- - Faster queries on subset
-- - Less write overhead
```

---

## 🧩 11. Design Patterns

### Pattern 1: Surrogate + Natural Key

```sql
CREATE TABLE countries (
    id INT PRIMARY KEY AUTO_INCREMENT,    -- Surrogate (internal)
    code CHAR(2) UNIQUE NOT NULL,         -- Natural (ISO 3166)
    name VARCHAR(100) NOT NULL
);

-- Foreign keys use small surrogate:
CREATE TABLE addresses (
    id BIGINT PRIMARY KEY,
    country_id INT NOT NULL,  -- 4 bytes vs replicating 2-char code everywhere
    FOREIGN KEY (country_id) REFERENCES countries(id)
);
```

---

### Pattern 2: Composite Key for Many-to-Many

```sql
CREATE TABLE student_courses (
    student_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,
    enrolled_at TIMESTAMP DEFAULT NOW(),
    grade VARCHAR(2),
    
    PRIMARY KEY (student_id, course_id),  -- Composite PK
    INDEX idx_course (course_id),         -- Reverse lookup
    FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE,
    FOREIGN KEY (course_id) REFERENCES courses(id) ON DELETE CASCADE
);
```

---

### Pattern 3: Soft Delete with Partial Index

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    deleted_at TIMESTAMP,
    
    -- Email unique only among active users
    UNIQUE (email) WHERE deleted_at IS NULL
);

-- Partial index for active users only
CREATE INDEX idx_active_users ON users(id) WHERE deleted_at IS NULL;
```

---

### Pattern 4: Time-Series with Composite Key + Partitioning

```sql
CREATE TABLE metrics (
    timestamp TIMESTAMP NOT NULL,
    metric_name VARCHAR(50) NOT NULL,
    value FLOAT NOT NULL,
    
    PRIMARY KEY (metric_name, timestamp)  -- Composite PK
) PARTITION BY RANGE (UNIX_TIMESTAMP(timestamp)) (
    PARTITION p_2024_01 VALUES LESS THAN (UNIX_TIMESTAMP('2024-02-01')),
    PARTITION p_2024_02 VALUES LESS THAN (UNIX_TIMESTAMP('2024-03-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

---

### Pattern 5: Hierarchical Data with Self-Referencing FK

```sql
CREATE TABLE categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    parent_id BIGINT,
    name VARCHAR(100) NOT NULL,
    level INT NOT NULL,
    
    INDEX idx_parent (parent_id),  -- Important for hierarchy traversal
    FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE CASCADE,
    UNIQUE (parent_id, name)  -- Name unique per level
);
```

---

## 📚 Summary

### Key Takeaways

1. **Every table needs a PRIMARY KEY** - for identification and performance
2. **Always index FOREIGN KEY columns** - for joins and cascade performance
3. **Composite index order matters** - left-prefix rule, selectivity first
4. **Use covering indexes for common queries** - index-only scans
5. **Partial indexes save space** - index only what you query
6. **Surrogate keys are safer** - stable, small, no business logic dependency
7. **Don't over-index** - each index slows writes
8. **High cardinality columns  index better** - more selective
9. **Foreign keys enforce referential integrity** - prevent orphaned data
10. **Monitor index usage** - drop unused indexes

### Index Decision Tree

```
Need to optimize...

└─ Lookup by column?
   ├─ High cardinality? → Single column index
   ├─ Low cardinality? → Consider composite or skip
   └─ Always used together? → Composite index

└─ JOIN performance?
   └─ → Index all FK columns

└─ Sort operations?
   └─ → Index sort column (with DESC if needed)

└─ Range queries?
   └─ → B-tree index (default)

└─ Exact equality only?
   └─ → Hash index (if supported)

└─ Full-text search?
   └─ → Full-text index (GIN/GiST)

└─ Querying subset frequently?
   └─ → Partial index with WHERE clause

└─ SELECT specific columns only?
   └─ → Covering index (INCLUDE columns)

└─ Geographic queries?
   └─ → Spatial index (R-tree)
```

### Production Checklist

- [ ] Every table has PRIMARY KEY
- [ ] All FOREIGN KEY columns are indexed
- [ ] Composite indexes follow left-prefix rule
- [ ] High-frequency queries have covering indexes
- [ ] Partial indexes for common filtered queries
- [ ] No redundant indexes (check index usage stats)
- [ ] Indexes maintained regularly (ANALYZE, REINDEX)
- [ ] Surrogate keys used for most tables
- [ ] Natural keys only for static reference data
- [ ] Public-facing IDs use UUID or Snowflake ID

---

**Next Steps:**
- Review [../../05_Query_Optimization/05_Index_Selection.md](../../05_Query_Optimization/05_Index_Selection.md) for choosing indexes based on queries
- Study [../../06_Indexing_Strategies/](../../06_Indexing_Strategies/) for deep dive into index types
- Explore [../../30_Performance_Monitoring/](../../30_Performance_Monitoring/) for monitoring index usage

---

*Last Updated: February 18, 2026*
