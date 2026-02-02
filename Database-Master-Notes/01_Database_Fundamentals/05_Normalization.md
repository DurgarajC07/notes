# 📐 Database Normalization - Designing Efficient Schemas

> Normalization is the process of organizing data to reduce redundancy and improve data integrity. Understanding normalization forms (1NF through 5NF) is crucial for designing maintainable databases, though knowing when to denormalize is equally important.

---

## 📖 1. Concept Explanation

### What is Database Normalization?

**Normalization** is a systematic approach to decomposing tables to:

1. **Eliminate redundancy** (avoid storing same data multiple times)
2. **Ensure data integrity** (prevent update/delete/insert anomalies)
3. **Organize data logically** (group related data together)

```
Journey from Unnormalized to 5NF:

Unnormalized Table (Chaos)
├─> 1NF: Eliminate repeating groups
│   ├─> 2NF: Remove partial dependencies
│   │   ├─> 3NF: Remove transitive dependencies
│   │   │   ├─> BCNF: Stronger version of 3NF
│   │   │   │   ├─> 4NF: Remove multi-valued dependencies
│   │   │   │   │   └─> 5NF: Remove join dependencies

Each level reduces redundancy and improves integrity!
```

---

### The Three Anomalies

**Before understanding normal forms, understand what problems we're solving:**

**1. Update Anomaly:**

```
❌ Bad: Same data stored multiple times

| OrderID | CustomerName | CustomerEmail      |
|---------|--------------|---------------------|
| 1       | Alice        | alice@example.com   |
| 2       | Alice        | alice@example.com   |
| 3       | Alice        | alice@example.com   |

Problem: Alice changes email → Must update 3 rows!
Risk: Forget to update one → Inconsistent data 💥
```

**2. Deletion Anomaly:**

```
❌ Bad: Deleting one piece of information deletes unrelated information

| OrderID | CustomerName | CustomerEmail      |
|---------|--------------|---------------------|
| 1       | Alice        | alice@example.com   |

Problem: Alice cancels her only order → Delete row
Result: Lost Alice's contact info forever! 💥
```

**3. Insertion Anomaly:**

```
❌ Bad: Can't insert data without unrelated data

| OrderID | CustomerName | CustomerEmail      |
|---------|--------------|---------------------|
| ???     | Bob          | bob@example.com     |

Problem: Can't add Bob to database until he places an order! 💥
```

**Solution:** Normalize the database!

---

## 🧠 2. Why It Matters in Real Systems

### Real Disaster: Knight Capital (2012) - Denormalization Gone Wrong

**System:** Trading platform with denormalized data

**What Happened:**

```
Denormalized Schema (For "Performance"):
┌──────────────────────────────────────┐
│ Trades Table                          │
├──────────────────────────────────────┤
│ trade_id                              │
│ stock_symbol                          │
│ stock_name          ← Redundant       │
│ stock_exchange      ← Redundant       │
│ current_price       ← Redundant       │
│ company_sector      ← Redundant       │
│ ...                                   │
└──────────────────────────────────────┘

Batch job updated stock prices:
- Updated 1,000,000 trades for AAPL
- Missed 100 trades (race condition)
- Result: 100 trades had stale price! 💥

Impact:
- Executed trades at wrong prices
- Lost $440 million in 45 minutes
- Company went bankrupt
```

**Lesson:** Redundancy creates update anomalies. Normalization prevents this disaster!

---

### Real Success: Stripe - Normalized Payment Data

**System:** Payment processing (highly normalized)

**Design:**

```sql
-- Customers table (no redundancy)
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP
);

-- Payment methods table (separate entity)
CREATE TABLE payment_methods (
    method_id UUID PRIMARY KEY,
    customer_id UUID REFERENCES customers(customer_id),
    type VARCHAR(50),  -- 'card', 'bank_account'
    last4 VARCHAR(4),
    created_at TIMESTAMP
);

-- Charges table (references, no duplication)
CREATE TABLE charges (
    charge_id UUID PRIMARY KEY,
    customer_id UUID REFERENCES customers(customer_id),
    payment_method_id UUID REFERENCES payment_methods(method_id),
    amount INT,
    currency VARCHAR(3),
    status VARCHAR(50),
    created_at TIMESTAMP
);

Benefits:
✅ Customer email updated in ONE place
✅ Payment method updated in ONE place
✅ No update anomalies
✅ Handles billions of transactions/year reliably
```

**Lesson:** Normalization ensures data integrity at scale!

---

### Real Disaster: Healthcare System - Insertion Anomaly

**System:** Hospital patient records (poorly normalized)

**What Happened:**

```sql
-- Bad design: Patient info mixed with appointment info
CREATE TABLE patient_appointments (
    appointment_id INT PRIMARY KEY,
    patient_name VARCHAR(100),
    patient_ssn VARCHAR(11),
    patient_dob DATE,
    doctor_name VARCHAR(100),
    appointment_date DATETIME,
    diagnosis TEXT
);

Problem:
- New patient calls to register (no appointment yet)
- Can't insert patient without appointment_id! 💥
- Front desk creates fake appointment to register patient
- Database fills with fake/cancelled appointments
- Reporting becomes nightmare

Impact:
- 15% of records were fake appointments
- Compliance violations (HIPAA)
- $2M fine
```

**Lesson:** Insertion anomalies force workarounds that corrupt data!

---

## ⚙️ 3. Internal Working

### First Normal Form (1NF)

**Rule:** Eliminate repeating groups. Each cell must contain atomic (indivisible) values.

```sql
-- ❌ Not in 1NF: Repeating groups
CREATE TABLE orders_bad (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    products TEXT  -- "Laptop, Mouse, Keyboard" ← Not atomic!
);

-- ✅ 1NF: Atomic values
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100)
);

CREATE TABLE order_items (
    order_id INT,
    product_name VARCHAR(100),
    PRIMARY KEY (order_id, product_name)
);

Transformation:
Before (Not 1NF):
| order_id | customer_name | products                    |
|----------|---------------|------------------------------|
| 1        | Alice         | Laptop, Mouse, Keyboard      |

After (1NF):
| order_id | customer_name |
|----------|---------------|
| 1        | Alice         |

| order_id | product_name  |
|----------|---------------|
| 1        | Laptop        |
| 1        | Mouse         |
| 1        | Keyboard      |

Benefits:
✅ Can query individual products
✅ Can count products easily
✅ Can enforce constraints on products
```

---

### Second Normal Form (2NF)

**Rule:** Must be in 1NF + No partial dependencies (all non-key attributes must depend on the ENTIRE primary key).

```sql
-- ❌ Not in 2NF: Partial dependency
CREATE TABLE order_items_bad (
    order_id INT,
    product_id INT,
    customer_name VARCHAR(100),  -- Depends only on order_id (partial!)
    product_name VARCHAR(100),
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

Problem:
customer_name depends on order_id alone (not product_id)
└─> Partial dependency on composite key!

-- ✅ 2NF: Remove partial dependencies
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100)  -- Now depends on full PK
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100),
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

Transformation:
Before (1NF but not 2NF):
| order_id | product_id | customer_name | product_name | quantity |
|----------|------------|---------------|--------------|----------|
| 1        | 101        | Alice         | Laptop       | 1        |
| 1        | 102        | Alice         | Mouse        | 2        |
| 2        | 101        | Bob           | Laptop       | 1        |

After (2NF):
Orders:
| order_id | customer_name |
|----------|---------------|
| 1        | Alice         |
| 2        | Bob           |

Order_Items:
| order_id | product_id | product_name | quantity |
|----------|------------|--------------|----------|
| 1        | 101        | Laptop       | 1        |
| 1        | 102        | Mouse        | 2        |
| 2        | 101        | Laptop       | 1        |

Benefits:
✅ Update customer_name once per order
✅ No partial dependency anomalies
```

---

### Third Normal Form (3NF)

**Rule:** Must be in 2NF + No transitive dependencies (non-key attributes can't depend on other non-key attributes).

```sql
-- ❌ Not in 3NF: Transitive dependency
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),  -- Depends on department_id (transitive!)
    department_location VARCHAR(100)  -- Depends on department_id (transitive!)
);

Problem:
department_name depends on department_id (not directly on employee_id)
└─> Transitive dependency: employee_id → department_id → department_name

-- ✅ 3NF: Remove transitive dependencies
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100),
    department_location VARCHAR(100)
);

Transformation:
Before (2NF but not 3NF):
| employee_id | employee_name | department_id | department_name | department_location |
|-------------|---------------|---------------|-----------------|---------------------|
| 1           | Alice         | 10            | Engineering     | Building A          |
| 2           | Bob           | 10            | Engineering     | Building A          |
| 3           | Carol         | 20            | Marketing       | Building B          |

After (3NF):
Employees:
| employee_id | employee_name | department_id |
|-------------|---------------|---------------|
| 1           | Alice         | 10            |
| 2           | Bob           | 10            |
| 3           | Carol         | 20            |

Departments:
| department_id | department_name | department_location |
|---------------|-----------------|---------------------|
| 10            | Engineering     | Building A          |
| 20            | Marketing       | Building B          |

Benefits:
✅ Update department name/location in ONE place
✅ No transitive dependency anomalies
✅ Can add department without employee
```

---

### Boyce-Codd Normal Form (BCNF)

**Rule:** Stricter version of 3NF. Every determinant must be a candidate key.

```sql
-- ❌ 3NF but not BCNF
CREATE TABLE course_instructor_bad (
    student_id INT,
    course_id INT,
    instructor_id INT,
    PRIMARY KEY (student_id, course_id)
);

Constraint: Each course has only ONE instructor
Problem:
instructor_id determines course_id (course → instructor is 1:1)
But instructor_id is NOT a candidate key!

-- ✅ BCNF: Split the relationship
CREATE TABLE student_courses (
    student_id INT,
    course_id INT,
    PRIMARY KEY (student_id, course_id)
);

CREATE TABLE course_instructors (
    course_id INT PRIMARY KEY,
    instructor_id INT,
    UNIQUE (instructor_id, course_id)
);

Benefits:
✅ Course-instructor relationship defined once
✅ Can't assign multiple instructors to same course
✅ Update instructor for course in ONE place
```

---

### Fourth Normal Form (4NF)

**Rule:** Must be in BCNF + No multi-valued dependencies.

```sql
-- ❌ Not in 4NF: Multi-valued dependencies
CREATE TABLE employee_skills_bad (
    employee_id INT,
    skill VARCHAR(100),
    language VARCHAR(100),
    PRIMARY KEY (employee_id, skill, language)
);

Problem:
Skills and languages are independent:
- Alice knows Python, Java (skills)
- Alice speaks English, Spanish (languages)
- Must store all combinations! (2 skills × 2 languages = 4 rows)

| employee_id | skill  | language |
|-------------|--------|----------|
| 1           | Python | English  |
| 1           | Python | Spanish  |
| 1           | Java   | English  |
| 1           | Java   | Spanish  |

-- ✅ 4NF: Split multi-valued dependencies
CREATE TABLE employee_skills (
    employee_id INT,
    skill VARCHAR(100),
    PRIMARY KEY (employee_id, skill)
);

CREATE TABLE employee_languages (
    employee_id INT,
    language VARCHAR(100),
    PRIMARY KEY (employee_id, language)
);

After (4NF):
Employee_Skills:
| employee_id | skill  |
|-------------|--------|
| 1           | Python |
| 1           | Java   |

Employee_Languages:
| employee_id | language |
|-------------|----------|
| 1           | English  |
| 1           | Spanish  |

Benefits:
✅ Only 4 rows instead of 4 (2 + 2)
✅ Add skill without touching languages
✅ Add language without touching skills
```

---

### Fifth Normal Form (5NF)

**Rule:** Must be in 4NF + No join dependencies.

```sql
-- ❌ Not in 5NF: Join dependency
CREATE TABLE supplier_part_project_bad (
    supplier_id INT,
    part_id INT,
    project_id INT,
    PRIMARY KEY (supplier_id, part_id, project_id)
);

Constraints:
- Supplier S1 supplies part P1
- Project PR1 uses part P1
- Supplier S1 works on project PR1
- Therefore: S1 supplies P1 to PR1 (join dependency)

-- ✅ 5NF: Decompose join dependencies
CREATE TABLE supplier_parts (
    supplier_id INT,
    part_id INT,
    PRIMARY KEY (supplier_id, part_id)
);

CREATE TABLE project_parts (
    project_id INT,
    part_id INT,
    PRIMARY KEY (project_id, part_id)
);

CREATE TABLE supplier_projects (
    supplier_id INT,
    project_id INT,
    PRIMARY KEY (supplier_id, project_id)
);

Benefits:
✅ No redundant combinations
✅ Easier to maintain relationships
✅ Join to reconstruct original relationship
```

---

## ✅ 4. Best Practices

### 1. Normalize to 3NF by Default

```sql
-- ✅ 3NF is the sweet spot for most applications
-- Further normalization (BCNF, 4NF, 5NF) has diminishing returns

-- Example: E-commerce schema
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE addresses (
    address_id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    street VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    zip_code VARCHAR(20),
    country VARCHAR(50)
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    shipping_address_id INT REFERENCES addresses(address_id),
    billing_address_id INT REFERENCES addresses(address_id),
    order_date TIMESTAMP DEFAULT NOW(),
    status VARCHAR(50)
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(order_id),
    product_id INT REFERENCES products(product_id),
    quantity INT,
    unit_price DECIMAL(10, 2)
);

-- This is 3NF:
-- ✅ No repeating groups (1NF)
-- ✅ No partial dependencies (2NF)
-- ✅ No transitive dependencies (3NF)
-- ✅ Clean, maintainable, performant
```

---

### 2. Denormalize for Performance (When Necessary)

```sql
-- ❌ Don't denormalize without measuring first!

-- ✅ Denormalize only after proving performance bottleneck
-- Example: Read-heavy e-commerce product page

-- Normalized (Slow for product page):
SELECT
    p.product_id,
    p.name,
    c.category_name,  -- JOIN 1
    b.brand_name,     -- JOIN 2
    COUNT(r.review_id) as review_count,  -- JOIN 3
    AVG(r.rating) as avg_rating           -- Aggregation
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN brands b ON p.brand_id = b.brand_id
LEFT JOIN reviews r ON p.product_id = r.product_id
WHERE p.product_id = 12345
GROUP BY p.product_id, c.category_name, b.brand_name;

-- ✅ Denormalized (Fast for product page):
CREATE TABLE product_summary (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    category_name VARCHAR(100),  -- Denormalized
    brand_name VARCHAR(100),     -- Denormalized
    review_count INT,             -- Denormalized
    avg_rating DECIMAL(3, 2),    -- Denormalized
    last_updated TIMESTAMP
);

-- Maintain with trigger or batch job
CREATE TRIGGER update_product_summary
AFTER INSERT OR UPDATE OR DELETE ON reviews
FOR EACH ROW
EXECUTE FUNCTION refresh_product_summary();

-- Now product page query is simple:
SELECT * FROM product_summary WHERE product_id = 12345;

Trade-offs:
✅ 100x faster reads (no joins)
❌ More storage (redundancy)
❌ More complex writes (maintain summary)
❌ Risk of inconsistency (update lag)

When to denormalize:
- Read:Write ratio > 100:1
- Measured performance bottleneck
- Acceptable staleness (eventual consistency)
```

---

### 3. Use Materialized Views for Denormalization

```sql
-- ✅ Let database handle denormalization
-- Materialized view = Cached query result

CREATE MATERIALIZED VIEW product_details AS
SELECT
    p.product_id,
    p.name,
    c.category_name,
    b.brand_name,
    COUNT(r.review_id) as review_count,
    AVG(r.rating) as avg_rating
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN brands b ON p.brand_id = b.brand_id
LEFT JOIN reviews r ON p.product_id = r.product_id
GROUP BY p.product_id, p.name, c.category_name, b.brand_name;

-- Create index for fast lookups
CREATE INDEX idx_product_details_product_id
ON product_details(product_id);

-- Refresh periodically (or on-demand)
REFRESH MATERIALIZED VIEW product_details;

-- Query is fast (pre-computed):
SELECT * FROM product_details WHERE product_id = 12345;

Benefits:
✅ Fast reads (like denormalized table)
✅ No manual maintenance (database handles it)
✅ Can refresh concurrently (no locking)
✅ Clear separation (view vs base tables)
```

---

### 4. Document Denormalization Decisions

```sql
-- ✅ Always document WHY you denormalized
-- Help future developers understand trade-offs

CREATE TABLE product_cache (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    category_name VARCHAR(100),  -- DENORMALIZED: For performance
    brand_name VARCHAR(100),     -- DENORMALIZED: For performance

    -- DENORMALIZATION RATIONALE:
    -- Product page receives 10M requests/day
    -- Joining categories/brands caused 500ms latency
    -- Denormalized version: 10ms latency
    -- Trade-off: Category/brand name updated via nightly batch job
    -- Acceptable staleness: 24 hours
    -- Last reviewed: 2026-01-15

    last_sync_at TIMESTAMP DEFAULT NOW()
);
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Over-Normalization

```sql
-- ❌ Bad: Excessive normalization (diminishing returns)
-- Splitting address into separate tables for each component

CREATE TABLE customers (customer_id INT PRIMARY KEY, name VARCHAR(100));
CREATE TABLE addresses (address_id INT PRIMARY KEY, customer_id INT);
CREATE TABLE address_streets (address_id INT, street VARCHAR(255));
CREATE TABLE address_cities (city_id INT PRIMARY KEY, city_name VARCHAR(100));
CREATE TABLE address_city_mapping (address_id INT, city_id INT);
CREATE TABLE address_states (state_id INT PRIMARY KEY, state_name VARCHAR(50));
CREATE TABLE address_state_mapping (address_id INT, state_id INT);
-- ... this is madness! 😵

-- Querying address requires 6 JOINs:
SELECT
    c.name,
    s.street,
    city.city_name,
    state.state_name
FROM customers c
JOIN addresses a ON c.customer_id = a.customer_id
JOIN address_streets s ON a.address_id = s.address_id
JOIN address_city_mapping cm ON a.address_id = cm.address_id
JOIN address_cities city ON cm.city_id = city.city_id
JOIN address_state_mapping sm ON a.address_id = sm.address_id
JOIN address_states state ON sm.state_id = state.state_id;

-- ✅ Good: Practical normalization (3NF is enough)
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE addresses (
    address_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    street VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    zip_code VARCHAR(20)
);

-- Simple query (1 JOIN):
SELECT c.name, a.street, a.city, a.state
FROM customers c
JOIN addresses a ON c.customer_id = a.customer_id;

Lesson: 3NF is usually sufficient. BCNF/4NF/5NF rarely needed.
```

---

### Mistake 2: Premature Denormalization

```sql
-- ❌ Bad: Denormalizing without measuring
CREATE TABLE orders_denormalized (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100),     -- Denormalized "for performance"
    customer_email VARCHAR(255),    -- Denormalized "for performance"
    product_name VARCHAR(255),      -- Denormalized "for performance"
    product_price DECIMAL(10, 2),   -- Denormalized "for performance"
    -- ... without proving slow queries!
);

-- Problems:
-- ❌ Update anomalies (customer changes email → update all orders)
-- ❌ Storage waste (redundant data)
-- ❌ Maintenance nightmare (keep data in sync)
-- ❌ No actual performance benefit (joins are fast in modern DBs)

-- ✅ Good: Normalize first, optimize later
-- 1. Create normalized schema
CREATE TABLE customers (customer_id INT PRIMARY KEY, name VARCHAR(100), email VARCHAR(255));
CREATE TABLE products (product_id INT PRIMARY KEY, name VARCHAR(255), price DECIMAL(10, 2));
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    product_id INT REFERENCES products(product_id)
);

-- 2. Measure query performance
EXPLAIN ANALYZE
SELECT o.order_id, c.name, c.email, p.name, p.price
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN products p ON o.product_id = p.product_id;

-- 3. If slow (and indexes don't help), THEN denormalize
-- But measure first!
```

---

### Mistake 3: Not Using Foreign Keys

```sql
-- ❌ Bad: No foreign keys (referential integrity not enforced)
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT  -- No foreign key constraint!
);

-- Problems:
INSERT INTO orders (order_id, customer_id) VALUES (1, 999);
-- 999 doesn't exist in customers table!
-- ❌ Orphaned record (referential integrity violated)

DELETE FROM customers WHERE customer_id = 10;
-- Orders for customer 10 still exist!
-- ❌ Orphaned records

-- ✅ Good: Enforce foreign keys
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE CASCADE  -- Or RESTRICT, depending on requirements
        ON UPDATE CASCADE
);

-- Now database enforces integrity:
INSERT INTO orders (order_id, customer_id) VALUES (1, 999);
-- ERROR: Foreign key violation ✅

DELETE FROM customers WHERE customer_id = 10;
-- CASCADE: Orders deleted automatically ✅
-- RESTRICT: Deletion blocked if orders exist ✅
```

---

### Mistake 4: Storing Calculated Values

```sql
-- ❌ Bad: Storing calculated values (not normalized)
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    subtotal DECIMAL(10, 2),
    tax DECIMAL(10, 2),
    shipping DECIMAL(10, 2),
    total DECIMAL(10, 2)  -- Calculated! (subtotal + tax + shipping)
);

-- Problems:
UPDATE orders SET subtotal = 100.00 WHERE order_id = 1;
-- total still shows old value! ❌
-- Update anomaly

-- ✅ Good: Calculate on-the-fly
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    subtotal DECIMAL(10, 2),
    tax DECIMAL(10, 2),
    shipping DECIMAL(10, 2)
    -- No 'total' column
);

-- Calculate when needed:
SELECT
    order_id,
    subtotal,
    tax,
    shipping,
    (subtotal + tax + shipping) AS total
FROM orders;

-- OR: Use generated column (PostgreSQL 12+, MySQL 5.7+)
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    subtotal DECIMAL(10, 2),
    tax DECIMAL(10, 2),
    shipping DECIMAL(10, 2),
    total DECIMAL(10, 2) GENERATED ALWAYS AS (subtotal + tax + shipping) STORED
);

-- Database maintains total automatically ✅
```

---

## 🔐 6. Security Considerations

### 1. Normalized Schema Improves Access Control

```sql
-- ✅ Easier to control access with normalized tables

-- Normalized schema:
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100)
);

CREATE TABLE customer_payment_methods (
    payment_method_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    card_number_encrypted BYTEA,  -- Sensitive!
    card_expiry VARCHAR(7)
);

-- Grant different permissions:
-- Customer service can see customers but NOT payment info
GRANT SELECT ON customers TO customer_service_role;
GRANT SELECT, UPDATE ON customers TO customer_service_role;
-- No access to payment_methods ✅

-- Payment processor can see payment info
GRANT SELECT ON customer_payment_methods TO payment_processor_role;

-- vs Denormalized schema (harder to control access):
CREATE TABLE customers_denormalized (
    customer_id INT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100),
    card_number_encrypted BYTEA,  -- Mixed with non-sensitive data!
    card_expiry VARCHAR(7)
);

-- Customer service needs customer info but now has access to payment info ❌
```

---

### 2. Audit Logging Benefits from Normalization

```sql
-- ✅ Normalized schema makes audit logging precise

-- Log customer email changes:
CREATE TRIGGER audit_customer_email
AFTER UPDATE ON customers
FOR EACH ROW
WHEN (OLD.email IS DISTINCT FROM NEW.email)
EXECUTE FUNCTION log_email_change();

-- vs Denormalized schema:
-- Email stored in customers, orders, support_tickets, etc.
-- Must audit ALL tables ❌
-- Risk of missing audit log entry
```

---

### 3. PII Data Isolation

```sql
-- ✅ Separate PII into dedicated tables for compliance

CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    created_at TIMESTAMP
);

CREATE TABLE user_pii (
    user_id UUID PRIMARY KEY REFERENCES users(user_id),
    email VARCHAR(255),
    full_name VARCHAR(100),
    ssn_encrypted BYTEA,
    phone VARCHAR(20),
    -- All PII in one place for GDPR/CCPA compliance
    data_retention_until DATE
);

-- Benefits:
-- ✅ Easy to delete PII (GDPR "right to be forgotten")
DELETE FROM user_pii WHERE user_id = 'xxx';  -- User data remains

-- ✅ Easy to encrypt entire table
-- ✅ Easy to audit PII access
-- ✅ Easy to apply stricter access controls
REVOKE ALL ON user_pii FROM public;
GRANT SELECT ON user_pii TO compliance_officer;
```

---

## 🚀 7. Performance Optimization Techniques

### 1. Index Foreign Keys

```sql
-- ✅ Always index foreign keys (improve JOIN performance)

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- CREATE INDEX ON FOREIGN KEY:
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- Now JOIN is fast:
EXPLAIN ANALYZE
SELECT c.name, COUNT(o.order_id) as order_count
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id;

-- Index scan on orders(customer_id) ✅
-- Without index: Sequential scan (slow) ❌
```

---

### 2. Covering Indexes for Common Queries

```sql
-- ✅ Create covering indexes (include all columns needed by query)

-- Frequent query:
SELECT order_id, order_date, total
FROM orders
WHERE customer_id = 12345
AND order_date >= '2026-01-01';

-- Covering index (includes all columns):
CREATE INDEX idx_orders_customer_date_covering
ON orders(customer_id, order_date)
INCLUDE (order_id, total);

-- Query uses index-only scan (doesn't touch table) ✅
-- Faster than regular index + table lookup
```

---

### 3. Partition Large Tables

```sql
-- ✅ Partition normalized tables for performance

-- Orders table partitioned by year:
CREATE TABLE orders (
    order_id BIGSERIAL,
    customer_id INT,
    order_date DATE,
    total DECIMAL(10, 2)
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE orders_2026 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- Query automatically uses relevant partition:
SELECT * FROM orders WHERE order_date >= '2026-01-01';
-- Only scans orders_2026 partition ✅
-- Faster than scanning entire table

-- Index each partition separately:
CREATE INDEX idx_orders_2026_customer ON orders_2026(customer_id);
```

---

### 4. Use Batch Updates for Denormalized Data

```sql
-- ✅ If denormalized, update in batches (not per-row triggers)

-- Denormalized product summary:
CREATE TABLE product_summary (
    product_id INT PRIMARY KEY,
    review_count INT,
    avg_rating DECIMAL(3, 2)
);

-- ❌ Bad: Update after every review (slow)
CREATE TRIGGER update_product_summary_per_review
AFTER INSERT ON reviews
FOR EACH ROW
EXECUTE FUNCTION refresh_single_product_summary();

-- ✅ Good: Batch update every 10 minutes
CREATE FUNCTION batch_refresh_product_summary()
RETURNS void AS $$
BEGIN
    INSERT INTO product_summary (product_id, review_count, avg_rating)
    SELECT
        product_id,
        COUNT(*) as review_count,
        AVG(rating) as avg_rating
    FROM reviews
    WHERE created_at >= NOW() - INTERVAL '10 minutes'
    GROUP BY product_id
    ON CONFLICT (product_id) DO UPDATE SET
        review_count = EXCLUDED.review_count,
        avg_rating = EXCLUDED.avg_rating;
END;
$$ LANGUAGE plpgsql;

-- Schedule via cron or pg_cron:
SELECT cron.schedule('refresh-summary', '*/10 * * * *', 'SELECT batch_refresh_product_summary()');
```

---

## 🧪 8. Examples

### Example 1: Blog Platform (3NF Schema)

```sql
-- Users table
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Posts table
CREATE TABLE posts (
    post_id SERIAL PRIMARY KEY,
    author_id INT NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    published_at TIMESTAMP,
    FOREIGN KEY (author_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Comments table
CREATE TABLE comments (
    comment_id SERIAL PRIMARY KEY,
    post_id INT NOT NULL,
    author_id INT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
    FOREIGN KEY (author_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Tags table
CREATE TABLE tags (
    tag_id SERIAL PRIMARY KEY,
    tag_name VARCHAR(50) UNIQUE NOT NULL
);

-- Post-Tag relationship (Many-to-Many)
CREATE TABLE post_tags (
    post_id INT,
    tag_id INT,
    PRIMARY KEY (post_id, tag_id),
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id) REFERENCES tags(tag_id) ON DELETE CASCADE
);

-- Indexes for performance:
CREATE INDEX idx_posts_author ON posts(author_id);
CREATE INDEX idx_posts_published ON posts(published_at);
CREATE INDEX idx_comments_post ON comments(post_id);
CREATE INDEX idx_comments_author ON comments(author_id);

-- This schema is 3NF:
-- ✅ 1NF: All atomic values
-- ✅ 2NF: No partial dependencies
-- ✅ 3NF: No transitive dependencies
-- ✅ Foreign keys enforce referential integrity
-- ✅ Many-to-many relationship properly modeled
```

---

### Example 2: E-Commerce with Selective Denormalization

```sql
-- Normalized tables:
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    base_price DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE product_reviews (
    review_id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(product_id),
    user_id INT REFERENCES users(user_id),
    rating INT CHECK (rating BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Denormalized cache for product listings (read-heavy):
CREATE TABLE product_listing_cache (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    base_price DECIMAL(10, 2),
    review_count INT,          -- Denormalized
    avg_rating DECIMAL(3, 2),  -- Denormalized
    last_updated TIMESTAMP DEFAULT NOW()
);

-- Maintain cache with materialized view refresh:
CREATE MATERIALIZED VIEW product_listing_mv AS
SELECT
    p.product_id,
    p.name,
    p.base_price,
    COUNT(r.review_id) as review_count,
    COALESCE(AVG(r.rating), 0) as avg_rating
FROM products p
LEFT JOIN product_reviews r ON p.product_id = r.product_id
GROUP BY p.product_id, p.name, p.base_price;

-- Refresh every 5 minutes:
SELECT cron.schedule('refresh-product-listing', '*/5 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY product_listing_mv');

-- Product listing query (fast):
SELECT * FROM product_listing_mv WHERE product_id = 12345;

-- Product detail query (normalized, joins acceptable):
SELECT
    p.*,
    AVG(r.rating) as avg_rating,
    COUNT(r.review_id) as review_count
FROM products p
LEFT JOIN product_reviews r ON p.product_id = r.product_id
WHERE p.product_id = 12345
GROUP BY p.product_id;
```

---

### Example 3: Multi-Tenant SaaS (Normalized with Tenant Isolation)

```sql
-- Tenants table
CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_name VARCHAR(255) NOT NULL,
    plan VARCHAR(50),  -- 'free', 'pro', 'enterprise'
    created_at TIMESTAMP DEFAULT NOW()
);

-- Users table (normalized, references tenant)
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100),
    role VARCHAR(50),  -- 'admin', 'member', 'viewer'
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    UNIQUE (tenant_id, email)  -- Email unique per tenant
);

-- Projects table (tenant-scoped)
CREATE TABLE projects (
    project_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_by UUID,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    FOREIGN KEY (created_by) REFERENCES users(user_id)
);

-- Tasks table (tenant-scoped)
CREATE TABLE tasks (
    task_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL,
    tenant_id UUID NOT NULL,  -- Denormalized for RLS
    title VARCHAR(255),
    status VARCHAR(50),
    assigned_to UUID,
    FOREIGN KEY (project_id) REFERENCES projects(project_id) ON DELETE CASCADE,
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    FOREIGN KEY (assigned_to) REFERENCES users(user_id)
);

-- Row-Level Security (RLS) for tenant isolation:
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_users ON users
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

CREATE POLICY tenant_isolation_projects ON projects
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

CREATE POLICY tenant_isolation_tasks ON tasks
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

-- Application sets tenant context:
-- SET app.current_tenant = 'tenant-uuid';
-- Now all queries automatically filtered by tenant ✅
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: Stripe - Payment Processing

**Requirements:**

- ACID transactions (money involved)
- Strong data integrity
- Auditable (compliance)

**Schema Design (Highly Normalized):**

```sql
-- Customers (deduplicated)
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE,  -- Updated in ONE place
    metadata JSONB
);

-- Payment Methods (separate entity)
CREATE TABLE payment_methods (
    payment_method_id UUID PRIMARY KEY,
    customer_id UUID REFERENCES customers(customer_id),
    type VARCHAR(50),
    last4 VARCHAR(4),
    fingerprint VARCHAR(64) UNIQUE  -- Detect duplicates
);

-- Charges (references, no duplication)
CREATE TABLE charges (
    charge_id UUID PRIMARY KEY,
    customer_id UUID REFERENCES customers(customer_id),
    payment_method_id UUID REFERENCES payment_methods(payment_method_id),
    amount INT,
    currency VARCHAR(3),
    status VARCHAR(50),
    created_at TIMESTAMP
);

-- Events (immutable audit log)
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(100),
    related_object_type VARCHAR(50),
    related_object_id UUID,
    data JSONB,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

Benefits:
✅ Customer email updated once
✅ Payment method deduplicated by fingerprint
✅ Charges reference, don't duplicate
✅ Events immutable (audit trail)
✅ Strong consistency (ACID)
```

---

### Use Case 2: Twitter - Timeline Data (Denormalized for Read Performance)

**Requirements:**

- Billions of tweets/day
- Sub-second timeline loading
- Eventual consistency acceptable

**Schema Design (Selectively Denormalized):**

```sql
-- Tweets table (normalized)
CREATE TABLE tweets (
    tweet_id BIGINT PRIMARY KEY,
    author_id BIGINT,
    content TEXT,
    created_at TIMESTAMP
);

-- Timelines (denormalized for performance)
CREATE TABLE user_timelines (
    user_id BIGINT,
    tweet_id BIGINT,
    author_id BIGINT,        -- Denormalized
    author_name VARCHAR(100), -- Denormalized
    author_avatar_url TEXT,   -- Denormalized
    content TEXT,             -- Denormalized
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id)
) PARTITION BY HASH (user_id);

-- Fanout on write:
-- When tweet posted, denormalize into followers' timelines
-- Acceptable: Slight delay (eventual consistency)
-- Benefit: Timeline reads are single-partition queries ✅

Trade-offs:
✅ Timeline loads in <100ms (no joins)
❌ More storage (denormalized)
❌ Update lag (author changes name → timelines lag)
✅ Acceptable for Twitter's use case
```

---

### Use Case 3: Healthcare System - Patient Records (Strict Normalization)

**Requirements:**

- HIPAA compliance
- Audit trail
- Strong referential integrity

**Schema Design (Strictly Normalized to 3NF):**

```sql
-- Patients (no redundancy)
CREATE TABLE patients (
    patient_id UUID PRIMARY KEY,
    ssn_encrypted BYTEA UNIQUE,
    date_of_birth DATE,
    created_at TIMESTAMP
);

-- Patient Names (separate for history tracking)
CREATE TABLE patient_names (
    name_id UUID PRIMARY KEY,
    patient_id UUID REFERENCES patients(patient_id),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    valid_from TIMESTAMP,
    valid_to TIMESTAMP
);

-- Appointments (references only)
CREATE TABLE appointments (
    appointment_id UUID PRIMARY KEY,
    patient_id UUID REFERENCES patients(patient_id),
    doctor_id UUID REFERENCES doctors(doctor_id),
    scheduled_at TIMESTAMP,
    status VARCHAR(50)
);

-- Medical Records (references only)
CREATE TABLE medical_records (
    record_id UUID PRIMARY KEY,
    patient_id UUID REFERENCES patients(patient_id),
    appointment_id UUID REFERENCES appointments(appointment_id),
    diagnosis TEXT,
    created_at TIMESTAMP
);

-- Audit Log (every change tracked)
CREATE TABLE audit_log (
    audit_id UUID PRIMARY KEY,
    table_name VARCHAR(100),
    record_id UUID,
    user_id UUID,
    action VARCHAR(50),  -- INSERT, UPDATE, DELETE
    old_values JSONB,
    new_values JSONB,
    timestamp TIMESTAMP DEFAULT NOW()
) PARTITION BY RANGE (timestamp);

Benefits:
✅ Patient info updated in ONE place
✅ Complete audit trail (HIPAA)
✅ Referential integrity (no orphaned records)
✅ Historical tracking (valid_from/valid_to)
✅ Data accuracy (life-critical!)
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What is database normalization and why is it important?

**Answer:**

**Normalization is organizing data to reduce redundancy and improve data integrity.**

**Why it matters:**

1. **Prevents anomalies:**
   - **Update anomaly:** Changing data in one place doesn't require updating multiple rows
   - **Deletion anomaly:** Deleting one entity doesn't accidentally delete unrelated data
   - **Insertion anomaly:** Can add data without requiring unrelated data

2. **Saves storage:**
   - No duplicate data
   - Smaller database size

3. **Improves maintainability:**
   - Changes made in one place
   - Easier to understand schema

**Real-world example:**

```
Unnormalized (Bad):
| OrderID | CustomerName | CustomerEmail      |
|---------|--------------|---------------------|
| 1       | Alice        | alice@example.com   |
| 2       | Alice        | alice@example.com   |  ← Duplicate
| 3       | Alice        | alice@example.com   |  ← Duplicate

Problem: Alice changes email → Must update 3 rows! (Update anomaly)

Normalized (Good):
Customers:
| CustomerID | Name  | Email              |
|------------|-------|--------------------|
| 1          | Alice | alice@example.com  |  ← ONE place

Orders:
| OrderID | CustomerID |
|---------|------------|
| 1       | 1          |
| 2       | 1          |
| 3       | 1          |

Solution: Alice changes email → Update 1 row! ✅
```

---

### Q2: Explain 1NF, 2NF, and 3NF with examples.

**Answer:**

**1NF (First Normal Form): Atomic values only**

```
❌ Not 1NF: Repeating groups
| StudentID | Name  | Subjects                  |
|-----------|-------|---------------------------|
| 1         | Alice | Math, Physics, Chemistry  |  ← Not atomic!

✅ 1NF: Atomic values
| StudentID | Name  | Subject   |
|-----------|-------|-----------|
| 1         | Alice | Math      |
| 1         | Alice | Physics   |
| 1         | Alice | Chemistry |
```

**2NF (Second Normal Form): No partial dependencies**

```
❌ Not 2NF: Partial dependency on composite key
| StudentID | Subject | Teacher     | Room  |
|-----------|---------|-------------|-------|
| 1         | Math    | Dr. Smith   | 101   |
| 1         | Physics | Dr. Jones   | 102   |

Problem: Teacher and Room depend on Subject only (not StudentID)
        └─> Partial dependency on composite key (StudentID, Subject)

✅ 2NF: Split into separate tables
Students_Subjects:
| StudentID | Subject |
|-----------|---------|
| 1         | Math    |
| 1         | Physics |

Subjects:
| Subject | Teacher     | Room  |
|---------|-------------|-------|
| Math    | Dr. Smith   | 101   |
| Physics | Dr. Jones   | 102   |
```

**3NF (Third Normal Form): No transitive dependencies**

```
❌ Not 3NF: Transitive dependency
| EmployeeID | Name  | DeptID | DeptName    |
|------------|-------|--------|-------------|
| 1          | Alice | 10     | Engineering |
| 2          | Bob   | 10     | Engineering |

Problem: DeptName depends on DeptID (not directly on EmployeeID)
        └─> Transitive dependency: EmployeeID → DeptID → DeptName

✅ 3NF: Split into separate tables
Employees:
| EmployeeID | Name  | DeptID |
|------------|-------|--------|
| 1          | Alice | 10     |
| 2          | Bob   | 10     |

Departments:
| DeptID | DeptName    |
|--------|-------------|
| 10     | Engineering |
```

**Key takeaway:** Each level removes a specific type of redundancy!

---

### Q3: When should you denormalize a database?

**Answer:**

**Denormalize only after proving a performance bottleneck!**

**When to denormalize:**

1. **Read-heavy workloads (read:write ratio > 100:1)**
   - Example: Product catalog (millions of reads, few writes)
   - Solution: Denormalize for fast reads

2. **Complex joins causing latency**
   - Example: Dashboard queries joining 10+ tables
   - Solution: Materialized view or denormalized summary table

3. **Reporting/Analytics queries**
   - Example: Monthly sales reports with aggregations
   - Solution: Denormalized fact table (star schema)

4. **Acceptable staleness (eventual consistency OK)**
   - Example: Social media like counts
   - Solution: Denormalized counter (update async)

**When NOT to denormalize:**

1. **Write-heavy workloads**
   - Example: Financial transactions
   - Denormalization slows down writes (maintain multiple copies)

2. **Strong consistency required**
   - Example: Banking, inventory
   - Denormalization risks inconsistency

3. **Haven't measured performance**
   - **Premature optimization is the root of all evil!**
   - Modern databases handle joins well

**Process:**

```
1. Start normalized (3NF)
2. Measure query performance (EXPLAIN ANALYZE)
3. Try indexes first (cheaper than denormalization)
4. If still slow, consider materialized views
5. If still slow, denormalize strategically
6. Document WHY you denormalized
```

**Example:**

```sql
-- Normalized (Slow: 500ms)
SELECT p.name, c.category_name, b.brand_name, COUNT(r.review_id)
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN brands b ON p.brand_id = b.brand_id
LEFT JOIN reviews r ON p.product_id = r.product_id
WHERE p.product_id = 12345
GROUP BY p.product_id, c.category_name, b.brand_name;

-- Denormalized (Fast: 10ms)
SELECT * FROM product_cache WHERE product_id = 12345;

-- BUT: Must maintain product_cache (async job)
```

---

### Q4: What are the trade-offs between normalization and denormalization?

**Answer:**

| Aspect                | Normalization                 | Denormalization                          |
| --------------------- | ----------------------------- | ---------------------------------------- |
| **Data Integrity**    | ✅ Strong (no redundancy)     | ❌ Weak (redundancy risks inconsistency) |
| **Storage**           | ✅ Efficient (no duplication) | ❌ Wasteful (duplicate data)             |
| **Write Performance** | ✅ Fast (update one place)    | ❌ Slow (update multiple places)         |
| **Read Performance**  | ❌ Slower (joins required)    | ✅ Faster (no joins)                     |
| **Maintainability**   | ✅ Easy (clear structure)     | ❌ Hard (complex synchronization)        |
| **Consistency**       | ✅ Strong (ACID)              | ❌ Eventual (lag)                        |

**Real-world example:**

**Banking system (Normalized):**

```sql
-- Accounts table (normalized)
CREATE TABLE accounts (
    account_id INT PRIMARY KEY,
    customer_id INT,
    balance DECIMAL(10, 2)
);

-- Transactions reference account
CREATE TABLE transactions (
    transaction_id INT PRIMARY KEY,
    account_id INT REFERENCES accounts(account_id),
    amount DECIMAL(10, 2)
);

Trade-offs:
✅ Balance updated in ONE place (strong consistency)
❌ Must JOIN to see account + transactions (slightly slower reads)
Decision: Choose normalization (correctness > speed)
```

**Social media (Denormalized):**

```sql
-- Posts table (denormalized)
CREATE TABLE posts (
    post_id INT PRIMARY KEY,
    author_id INT,
    author_name VARCHAR(100),    -- Denormalized
    author_avatar_url TEXT,      -- Denormalized
    like_count INT,              -- Denormalized
    content TEXT
);

Trade-offs:
✅ Feed loads fast (no joins)
❌ Author changes name → must update all posts (slow)
❌ Like_count might be slightly off (eventual consistency)
Decision: Choose denormalization (speed > immediate consistency)
```

---

### Q5: How do you handle denormalization in a write-heavy system?

**Answer:**

**Strategies for denormalization in write-heavy systems:**

**1. Batch updates (reduce write frequency):**

```sql
-- Instead of updating after every write:
CREATE TRIGGER update_counter_per_write  -- ❌ Slow!
AFTER INSERT ON likes
FOR EACH ROW EXECUTE FUNCTION increment_like_count();

-- Batch update every 10 seconds:
CREATE FUNCTION batch_update_like_counts() RETURNS void AS $$
BEGIN
    UPDATE posts p
    SET like_count = (SELECT COUNT(*) FROM likes WHERE post_id = p.post_id)
    WHERE p.post_id IN (
        SELECT DISTINCT post_id FROM likes
        WHERE created_at >= NOW() - INTERVAL '10 seconds'
    );
END;
$$ LANGUAGE plpgsql;
```

**2. Async updates (don't block writes):**

```sql
-- Write to queue, update denormalized data async
-- Main write (fast):
INSERT INTO likes (user_id, post_id) VALUES (123, 456);

-- Publish message to queue (RabbitMQ, Kafka):
PUBLISH 'like_counts_queue', {post_id: 456};

-- Background worker updates denormalized count:
SUBSCRIBE 'like_counts_queue', (message) => {
    UPDATE posts SET like_count = like_count + 1
    WHERE post_id = message.post_id;
};
```

**3. Eventually consistent (accept lag):**

```sql
-- Accept that denormalized data lags by seconds/minutes
-- Main write (fast):
INSERT INTO transactions (user_id, amount) VALUES (123, 100);

-- Denormalized summary updated later:
-- (User balance might be slightly stale for few seconds)
UPDATE user_summary
SET total_spent = total_spent + 100
WHERE user_id = 123;
```

**4. Materialized views (let database handle it):**

```sql
-- Database maintains denormalized view:
CREATE MATERIALIZED VIEW post_stats AS
SELECT
    post_id,
    COUNT(DISTINCT likes.user_id) as like_count,
    COUNT(DISTINCT comments.comment_id) as comment_count
FROM posts
LEFT JOIN likes ON posts.post_id = likes.post_id
LEFT JOIN comments ON posts.post_id = comments.post_id
GROUP BY post_id;

-- Refresh periodically (not per-write):
REFRESH MATERIALIZED VIEW CONCURRENTLY post_stats;
```

**Key principle:** Decouple denormalized updates from main writes (don't block critical path!).

---

## 🧩 11. Design Patterns

### Pattern 1: Materialized View Pattern

**Problem:** Complex joins needed frequently, but denormalization too risky

**Solution:**

```sql
-- Create materialized view (pre-computed join)
CREATE MATERIALIZED VIEW order_summaries AS
SELECT
    o.order_id,
    o.order_date,
    c.customer_name,
    c.customer_email,
    SUM(oi.quantity * oi.unit_price) as total_amount,
    COUNT(oi.order_item_id) as item_count
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.order_date, c.customer_name, c.customer_email;

-- Create index for fast lookups:
CREATE INDEX idx_order_summaries_order_id ON order_summaries(order_id);

-- Refresh strategy:
-- Option 1: Periodic refresh
REFRESH MATERIALIZED VIEW CONCURRENTLY order_summaries;

-- Option 2: Incremental refresh (PostgreSQL 13+)
REFRESH MATERIALIZED VIEW order_summaries WITH DATA;

-- Query is fast (pre-computed):
SELECT * FROM order_summaries WHERE order_id = 12345;
```

**Pros:**

- ✅ Fast reads (no joins)
- ✅ Database manages consistency
- ✅ Can refresh concurrently (no locking)

**Cons:**

- ❌ Staleness (data lags behind base tables)
- ❌ Refresh overhead (resource-intensive)

---

### Pattern 2: Shadow/Cache Table Pattern

**Problem:** Need denormalized data for performance but want to maintain normalized source

**Solution:**

```sql
-- Normalized source (source of truth)
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10, 2)
);

CREATE TABLE reviews (
    review_id INT PRIMARY KEY,
    product_id INT REFERENCES products(product_id),
    rating INT CHECK (rating BETWEEN 1 AND 5)
);

-- Denormalized cache (for performance)
CREATE TABLE product_cache (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10, 2),
    review_count INT,         -- Denormalized
    avg_rating DECIMAL(3, 2), -- Denormalized
    cache_updated_at TIMESTAMP
);

-- Maintain cache with trigger or background job:
CREATE OR REPLACE FUNCTION refresh_product_cache(p_product_id INT)
RETURNS void AS $$
BEGIN
    INSERT INTO product_cache (
        product_id, name, price, review_count, avg_rating, cache_updated_at
    )
    SELECT
        p.product_id,
        p.name,
        p.price,
        COUNT(r.review_id),
        COALESCE(AVG(r.rating), 0),
        NOW()
    FROM products p
    LEFT JOIN reviews r ON p.product_id = r.product_id
    WHERE p.product_id = p_product_id
    GROUP BY p.product_id, p.name, p.price
    ON CONFLICT (product_id) DO UPDATE SET
        name = EXCLUDED.name,
        price = EXCLUDED.price,
        review_count = EXCLUDED.review_count,
        avg_rating = EXCLUDED.avg_rating,
        cache_updated_at = NOW();
END;
$$ LANGUAGE plpgsql;

-- Background job refreshes stale cache entries:
SELECT refresh_product_cache(product_id)
FROM product_cache
WHERE cache_updated_at < NOW() - INTERVAL '10 minutes';
```

---

### Pattern 3: Event Sourcing Pattern

**Problem:** Need full audit trail and ability to recompute denormalized data

**Solution:**

```sql
-- Event log (immutable, append-only)
CREATE TABLE events (
    event_id BIGSERIAL PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    occurred_at TIMESTAMP DEFAULT NOW()
) PARTITION BY RANGE (occurred_at);

-- Example events:
-- {event_type: 'OrderPlaced', data: {order_id: 123, customer_id: 456}}
-- {event_type: 'ItemAdded', data: {order_id: 123, product_id: 789}}
-- {event_type: 'OrderPaid', data: {order_id: 123, amount: 99.99}}

-- Projections (denormalized views computed from events)
CREATE TABLE order_summary_projection (
    order_id UUID PRIMARY KEY,
    customer_id UUID,
    total_items INT,
    total_amount DECIMAL(10, 2),
    status VARCHAR(50),
    last_updated TIMESTAMP
);

-- Rebuild projection from events:
CREATE OR REPLACE FUNCTION rebuild_order_summary()
RETURNS void AS $$
BEGIN
    TRUNCATE order_summary_projection;

    INSERT INTO order_summary_projection
    SELECT
        (event_data->>'order_id')::UUID as order_id,
        (event_data->>'customer_id')::UUID as customer_id,
        COUNT(*) FILTER (WHERE event_type = 'ItemAdded') as total_items,
        SUM((event_data->>'amount')::DECIMAL) FILTER (WHERE event_type = 'OrderPaid') as total_amount,
        MAX(event_data->>'status') as status,
        MAX(occurred_at) as last_updated
    FROM events
    WHERE event_type IN ('OrderPlaced', 'ItemAdded', 'OrderPaid')
    GROUP BY
        (event_data->>'order_id')::UUID,
        (event_data->>'customer_id')::UUID;
END;
$$ LANGUAGE plpgsql;

-- Benefits:
-- ✅ Can rebuild denormalized data from events
-- ✅ Full audit trail (every change recorded)
-- ✅ Time travel (query state at any point in time)
```

---

## 📚 Summary

**Normalization:**

- ✅ **1NF:** Atomic values (no repeating groups)
- ✅ **2NF:** No partial dependencies (no dependency on part of composite key)
- ✅ **3NF:** No transitive dependencies (no dependency on non-key attributes)
- ✅ **BCNF:** Every determinant is a candidate key
- ✅ **4NF:** No multi-valued dependencies
- ✅ **5NF:** No join dependencies

**When to normalize:**

- Strong consistency required (banking, inventory)
- Write-heavy workloads
- Data integrity critical

**When to denormalize:**

- Read-heavy workloads (read:write ratio > 100:1)
- Complex joins causing latency
- Eventual consistency acceptable

**Best practices:**

- ✅ Normalize to 3NF by default
- ✅ Measure before denormalizing
- ✅ Use materialized views for denormalization
- ✅ Document denormalization decisions
- ✅ Index foreign keys

**Key takeaway:** Start normalized, denormalize strategically based on measured performance bottlenecks!
