# Schema Design Principles

## 1. Concept Explanation

### What is Schema Design?

Schema design is the process of defining the structure, organization, and constraints of a database to efficiently store, retrieve, and maintain data while enforcing business rules. It's the architectural blueprint that determines how data elements relate to each other and how they're accessed by applications.

**Core Principles:**

```
┌─────────────────────────────────────────────────────────────┐
│              SCHEMA DESIGN PRINCIPLES HIERARCHY              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. CORRECTNESS (Foundation)                                 │
│     └─ Data integrity, accuracy, business rules              │
│                                                               │
│  2. PERFORMANCE (Essential)                                  │
│     └─ Query efficiency, indexing strategy, access patterns  │
│                                                               │
│  3. MAINTAINABILITY (Long-term)                              │
│     └─ Clear structure, documentation, evolution support     │
│                                                               │
│  4. SCALABILITY (Growth)                                     │
│     └─ Horizontal/vertical scaling, partitioning, sharding   │
│                                                               │
│  5. SECURITY (Critical)                                      │
│     └─ Access control, encryption, audit trails              │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### The Schema Design Spectrum

```
NORMALIZATION ←──────────────────────────→ DENORMALIZATION
(3NF, BCNF)                                  (Star Schema)

                   SWEET SPOT
                   ↓
            Hybrid Approach
            (Normalize by default,
             denormalize strategically)

Trade-offs:
┌──────────────────┬────────────────┬─────────────────┐
│                  │  NORMALIZED    │  DENORMALIZED   │
├──────────────────┼────────────────┼─────────────────┤
│ Data Redundancy  │  Minimal       │  High           │
│ Write Performance│  Fast          │  Slower         │
│ Read Performance │  May need JOINs│  Fast (no JOINs)│
│ Data Integrity   │  High          │  Lower          │
│ Storage Cost     │  Lower         │  Higher         │
│ Maintenance      │  Easier        │  Complex        │
└──────────────────┴────────────────┴─────────────────┘
```

### Key Design Principles

**1. Start with Requirements, Not Technology**

**❌ BAD (Technology-first):**

```
"We'll use MongoDB because it's trendy"
"Let's use microservices architecture"
"We need GraphQL for everything"
```

**✅ GOOD (Requirements-first):**

```
1. Analyze access patterns (read:write ratio, query types)
2. Understand data relationships (transactional vs analytical)
3. Identify consistency requirements (strong vs eventual)
4. Determine scale requirements (users, data volume, growth)
5. Assess team expertise and operational capabilities
6. THEN choose technology that fits
```

**2. Design for Query Patterns**

**Schema should optimize for how data is accessed:**

```sql
-- If 90% of queries are "get customer with orders":
-- ❌ BAD: Highly normalized (requires JOIN)
SELECT c.*, o.*
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id = ?;
-- 2 table scans, JOIN overhead

-- ✅ GOOD: Denormalize frequently accessed data
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    total_orders INTEGER,        -- Denormalized count
    last_order_date TIMESTAMP,   -- Denormalized
    total_spent DECIMAL(10,2),   -- Denormalized
    lifetime_value DECIMAL(10,2) -- Denormalized
);
-- Single table lookup, updated via triggers/events
```

**3. Explicit Over Implicit**

**❌ BAD (Implicit relationships in data):**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    permissions TEXT  -- 'admin,editor,viewer' ← Implicit
);
-- Can't enforce referential integrity
-- Can't query "all admins" efficiently
-- Parsing required in application layer
```

**✅ GOOD (Explicit relationships):**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE roles (
    role_id UUID PRIMARY KEY,
    role_name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE user_roles (
    user_id UUID,
    role_id UUID,
    assigned_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (role_id) REFERENCES roles(role_id)
);
-- Explicit, queryable, maintainable
```

**4. Constrain at the Database Level**

**✅ GOOD (Database-enforced constraints):**

```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL DEFAULT NOW(),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount DECIMAL(10,2) NOT NULL,

    -- Business rules enforced at DB level
    CHECK (total_amount >= 0),
    CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled')),
    CHECK (order_date <= NOW()),  -- Can't order in future

    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Prevent orders without items
CREATE FUNCTION check_order_has_items()
RETURNS TRIGGER AS $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM order_items WHERE order_id = NEW.order_id) THEN
        RAISE EXCEPTION 'Order must have at least one item';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_order_items
    BEFORE UPDATE OF status ON orders
    FOR EACH ROW
    WHEN (NEW.status != 'pending')
    EXECUTE FUNCTION check_order_has_items();
```

---

## 2. Why It Matters

### Business Impact

**Cost of Poor Schema Design:**

```
┌─────────────────────────────────────────────────────────────┐
│         REAL-WORLD COSTS OF SCHEMA DESIGN MISTAKES          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  TWITTER (2008-2010):                                        │
│  - Poor sharding strategy caused "Fail Whale" outages        │
│  - $15M+ in lost revenue during peak outages                 │
│  - 2-year re-architecture project                            │
│                                                               │
│  HEALTHCARE SYSTEM (2019):                                   │
│  - Incorrect cardinality (Patient:Insurance = 1:1)           │
│  - 6-month data migration, $2M engineering cost              │
│  - Regulatory compliance violations                          │
│                                                               │
│  E-COMMERCE PLATFORM (2020):                                 │
│  - No partitioning strategy for orders table                 │
│  - Query timeouts during Black Friday (100M+ rows)           │
│  - $500K in lost sales, 12-hour emergency migration          │
│                                                               │
│  FINTECH STARTUP (2021):                                     │
│  - Weak data integrity (no foreign keys)                     │
│  - Orphaned records caused incorrect account balances        │
│  - $1.2M in refunds, loss of customer trust                  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Benefits of Good Schema Design:**

1. **Performance**: Queries execute in milliseconds instead of seconds
2. **Reliability**: Database enforces business rules, preventing data corruption
3. **Maintainability**: New features integrate cleanly without schema rewrites
4. **Scalability**: Designed for growth from day one (partitioning, sharding)
5. **Cost Efficiency**: Reduced storage, compute, and operational overhead

### Technical Impact

**Query Performance:**

```
Bad Schema (Normalized, No Indexes):
SELECT * FROM orders WHERE customer_id = ?;
→ Full table scan: 500ms on 10M rows

Good Schema (Strategic Indexes):
CREATE INDEX idx_orders_customer ON orders(customer_id);
→ Index scan: 5ms on 10M rows

100x performance improvement
```

**Data Integrity:**

```
Bad Schema (Application-enforced):
if (order.total_amount < 0) { return error; }
→ Race conditions, bugs bypass checks

Good Schema (Database-enforced):
CHECK (total_amount >= 0)
→ Impossible to violate, enforced at DB level
```

**Storage Efficiency:**

```
Bad Schema (Text for everything):
status TEXT  -- Stores 'pending' as 7 bytes × 10M rows = 70MB

Good Schema (Enum/Integer):
status SMALLINT  -- 2 bytes × 10M rows = 20MB
→ 3.5x storage savings, faster comparisons
```

---

## 3. Internal Working & Implementation

### Principle 1: Atomic Data Types

**❌ BAD (Composite values in single column):**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    name VARCHAR(100),  -- "John Doe" ← First and last name mixed
    address TEXT  -- "123 Main St, City, State 12345" ← Composite
);

-- Problems:
-- 1. Can't sort by last name
-- 2. Can't filter by city
-- 3. String parsing required in application
-- 4. No validation of individual components
```

**✅ GOOD (Atomic columns):**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20)
);

CREATE TABLE employee_addresses (
    address_id UUID PRIMARY KEY,
    employee_id UUID NOT NULL,
    address_type VARCHAR(20) NOT NULL, -- 'home', 'office'
    street_line1 VARCHAR(255) NOT NULL,
    street_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50),
    postal_code VARCHAR(20) NOT NULL,
    country CHAR(2) NOT NULL,

    FOREIGN KEY (employee_id) REFERENCES employees(employee_id) ON DELETE CASCADE
);

-- Now we can:
-- 1. Sort by last name: ORDER BY last_name
-- 2. Filter by city: WHERE city = 'San Francisco'
-- 3. Validate postal codes by country
-- 4. Support multiple addresses per employee
```

### Principle 2: Single Source of Truth

**❌ BAD (Duplicate data without synchronization):**

```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_name VARCHAR(100),  -- Duplicated from customers
    customer_email VARCHAR(255), -- Duplicated from customers
    customer_phone VARCHAR(20),  -- Duplicated from customers
    order_date TIMESTAMP
);
-- If customer changes email, orders have stale data
-- Update anomalies, data inconsistency
```

**✅ GOOD (Normalize, reference source of truth):**

```sql
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20)
);

CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL,

    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Customer data lives in one place
-- Orders reference customers by ID
-- Updates propagate automatically through foreign key
```

**Exception: Strategic Denormalization**

When to duplicate data:

```sql
CREATE TABLE order_snapshots (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,

    -- Snapshot customer data at time of order (historical record)
    customer_name_at_order VARCHAR(100) NOT NULL,
    customer_email_at_order VARCHAR(255) NOT NULL,
    shipping_address_at_order TEXT NOT NULL,

    -- Current reference for queries
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
-- Rationale: Historical accuracy (customer name may change)
-- Trade-off: More storage, but preserves audit trail
```

### Principle 3: Appropriate Data Types

**❌ BAD (VARCHAR for everything):**

```sql
CREATE TABLE products (
    product_id VARCHAR(255),      -- Should be UUID or BIGINT
    price VARCHAR(50),            -- Should be DECIMAL
    stock_quantity VARCHAR(20),   -- Should be INTEGER
    is_active VARCHAR(10),        -- Should be BOOLEAN
    created_at VARCHAR(100)       -- Should be TIMESTAMP
);
-- Wastes storage, no type safety, slow comparisons
```

**✅ GOOD (Specific data types):**

```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    cost DECIMAL(10,2) NOT NULL CHECK (cost >= 0 AND cost <= price),
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    is_active BOOLEAN NOT NULL DEFAULT true,
    weight_kg DECIMAL(8,3),
    dimensions_cm INTEGER[3],  -- [length, width, height]
    tags TEXT[],
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Benefits:
-- 1. Type safety (can't insert 'abc' for price)
-- 2. Efficient storage (BOOLEAN = 1 byte vs VARCHAR = variable)
-- 3. Proper comparisons (numeric vs string)
-- 4. Database can optimize queries based on type
```

**Data Type Selection Guide:**

```sql
-- IDs: Use UUID for distributed systems, BIGSERIAL for single server
product_id UUID PRIMARY KEY
product_id BIGSERIAL PRIMARY KEY

-- Money: Use DECIMAL (never FLOAT/DOUBLE)
price DECIMAL(10,2)  -- $99,999,999.99 max

-- Timestamps: Use TIMESTAMP WITH TIME ZONE
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()

-- Booleans: Use BOOLEAN (not CHAR(1) or SMALLINT)
is_active BOOLEAN NOT NULL DEFAULT true

-- Enums: Use VARCHAR with CHECK constraint or native ENUM
status VARCHAR(20) CHECK (status IN ('active', 'inactive', 'suspended'))

-- JSON: Use JSONB (not JSON) for better performance
metadata JSONB

-- Arrays: Use native arrays for simple lists
tags TEXT[]

-- Large Text: Use TEXT (not VARCHAR with arbitrary limit)
description TEXT
```

### Principle 4: Referential Integrity

**✅ GOOD (Comprehensive foreign key constraints):**

```sql
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    shipping_address_id UUID NOT NULL,

    FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
        ON DELETE RESTRICT    -- Prevent deletion if orders exist
        ON UPDATE CASCADE,    -- Update orders if customer_id changes

    FOREIGN KEY (shipping_address_id)
        REFERENCES addresses(address_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);

CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY,
    order_id UUID NOT NULL,
    product_id UUID NOT NULL,

    FOREIGN KEY (order_id)
        REFERENCES orders(order_id)
        ON DELETE CASCADE,    -- Delete items when order is deleted

    FOREIGN KEY (product_id)
        REFERENCES products(product_id)
        ON DELETE RESTRICT   -- Prevent product deletion if referenced
);
```

**ON DELETE/ON UPDATE Options:**

```
CASCADE: Propagate change to child rows
  Use: When child can't exist without parent (weak entity)
  Example: Order → OrderItems

RESTRICT: Prevent change if child rows exist
  Use: When child independence matters
  Example: Product referenced in orders shouldn't be deleted

SET NULL: Set child foreign key to NULL
  Use: When child can exist without parent
  Example: Employee → Manager (if manager leaves)

SET DEFAULT: Set child foreign key to default value
  Use: When there's a sensible default
  Example: Product → Category (set to 'Uncategorized')

NO ACTION: Similar to RESTRICT (checked at end of statement)
  Use: When you need deferred constraint checking
```

### Principle 5: Indexing Strategy

**❌ BAD (No indexes or over-indexing):**

```sql
-- No indexes: Every query is a full table scan
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,  -- Only PK index exists
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL,
    status VARCHAR(20)
);
-- Query: WHERE customer_id = ? → Full table scan (slow)

-- Over-indexing: Write performance suffers
CREATE INDEX idx_1 ON orders(customer_id);
CREATE INDEX idx_2 ON orders(order_date);
CREATE INDEX idx_3 ON orders(status);
CREATE INDEX idx_4 ON orders(customer_id, order_date);
CREATE INDEX idx_5 ON orders(customer_id, status);
CREATE INDEX idx_6 ON orders(order_date, status);
CREATE INDEX idx_7 ON orders(customer_id, order_date, status);
-- 7 indexes × every INSERT/UPDATE = slow writes
```

**✅ GOOD (Strategic indexes based on query patterns):**

```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL,
    status VARCHAR(20) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL
);

-- Analyze query patterns first:
-- 1. "Get customer orders" (90% of queries)
-- 2. "Recent orders by status" (5%)
-- 3. "Orders by date range" (3%)
-- 4. "High-value orders" (2%)

-- Index 1: Foreign key for JOINs (critical)
CREATE INDEX idx_orders_customer
    ON orders(customer_id);

-- Index 2: Composite for common query pattern
CREATE INDEX idx_orders_customer_date
    ON orders(customer_id, order_date DESC)
    INCLUDE (order_id, total_amount, status);
-- Covers: WHERE customer_id = ? ORDER BY order_date DESC
-- Includes extra columns for index-only scans

-- Index 3: Partial index for active orders
CREATE INDEX idx_orders_active_status
    ON orders(status, order_date DESC)
    WHERE status IN ('pending', 'processing');
-- Smaller index, faster queries on active orders

-- Index 4: Functional index for date range queries
CREATE INDEX idx_orders_date_range
    ON orders USING BRIN (order_date);
-- BRIN index for time-series data (smaller, efficient for ranges)

-- Monitor index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE tablename = 'orders'
ORDER BY idx_scan DESC;
```

---

## 4. Best Practices

### 1. Naming Conventions

**✅ GOOD (Consistent, clear naming):**

```sql
-- Tables: Singular nouns, lowercase with underscores
customers
products
order_items

-- Columns: Descriptive, lowercase with underscores
customer_id
first_name
created_at
is_active

-- Primary Keys: {table_name}_id
customer_id (in customers table)
product_id (in products table)

-- Foreign Keys: {referenced_table}_id
customer_id (in orders table, references customers)
product_id (in order_items table, references products)

-- Indexes: idx_{table}_{columns}
idx_orders_customer
idx_orders_customer_date

-- Constraints: {type}_{table}_{columns}
ck_orders_amount_positive  -- CHECK
fk_orders_customer         -- FOREIGN KEY
uk_users_email             -- UNIQUE
```

**❌ BAD (Inconsistent, cryptic naming):**

```sql
-- Inconsistent capitalization
CREATE TABLE Users (...);
CREATE TABLE PRODUCTS (...);
CREATE TABLE order_Items (...);

-- Abbreviations
CREATE TABLE cust (...);
CREATE TABLE prod (...);
CREATE TABLE ord_itm (...);

-- Hungarian notation (encoding type in name)
CREATE TABLE tblCustomers (
    intCustomerID INTEGER,
    strEmail VARCHAR(255),
    dtCreated TIMESTAMP
);
```

### 2. Soft Deletes vs Hard Deletes

**When to use soft deletes:**

```sql
-- E-commerce: Need order history even if customer account deleted
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100),
    deleted_at TIMESTAMP,  -- NULL = active, timestamp = deleted

    -- Partial unique index (only active accounts)
    CONSTRAINT uk_active_email UNIQUE (email) WHERE deleted_at IS NULL
);

-- View for active customers only
CREATE VIEW active_customers AS
SELECT * FROM customers WHERE deleted_at IS NULL;

-- Restore functionality
UPDATE customers
SET deleted_at = NULL
WHERE customer_id = ? AND deleted_at IS NOT NULL;
```

**When to use hard deletes:**

```sql
-- Session tokens: No need to keep expired tokens
CREATE TABLE user_sessions (
    session_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Hard delete expired sessions (no recovery needed)
DELETE FROM user_sessions WHERE expires_at < NOW();

-- Temporary data: Cart items (no historical value)
DELETE FROM cart_items WHERE cart_id = ? AND product_id = ?;
```

**Hybrid Approach:**

```sql
-- Archive table for compliance
CREATE TABLE orders_archive (
    LIKE orders INCLUDING ALL
);

-- Move to archive before deletion
BEGIN;
    INSERT INTO orders_archive SELECT * FROM orders WHERE order_id = ?;
    DELETE FROM orders WHERE order_id = ?;
COMMIT;
```

### 3. Temporal Data Modeling

**✅ GOOD (Track historical changes):**

```sql
-- Current state table
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    current_price DECIMAL(10,2) NOT NULL,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Price history table (audit trail)
CREATE TABLE product_price_history (
    history_id UUID PRIMARY KEY,
    product_id UUID NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    effective_from TIMESTAMP NOT NULL,
    effective_to TIMESTAMP,  -- NULL = current
    changed_by UUID NOT NULL,
    change_reason VARCHAR(500),

    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (changed_by) REFERENCES users(user_id),

    -- Prevent overlapping time ranges
    EXCLUDE USING gist (
        product_id WITH =,
        tstzrange(effective_from, effective_to, '[)') WITH &&
    )
);

-- Trigger to update history on price change
CREATE FUNCTION record_price_change()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.current_price != NEW.current_price THEN
        -- Close previous price period
        UPDATE product_price_history
        SET effective_to = NOW()
        WHERE product_id = NEW.product_id AND effective_to IS NULL;

        -- Insert new price period
        INSERT INTO product_price_history
            (product_id, price, effective_from, changed_by)
        VALUES
            (NEW.product_id, NEW.current_price, NOW(), current_setting('app.user_id')::UUID);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER track_price_changes
    AFTER UPDATE OF current_price ON products
    FOR EACH ROW
    EXECUTE FUNCTION record_price_change();

-- Query: Get price at specific point in time
SELECT price
FROM product_price_history
WHERE product_id = ?
  AND effective_from <= '2025-12-25'
  AND (effective_to IS NULL OR effective_to > '2025-12-25');
```

### 4. Multi-Tenancy Patterns

**Pattern 1: Shared Schema (Row-Level Isolation)**

**✅ GOOD for: Large number of tenants, similar data structure**

```sql
CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY,
    tenant_slug VARCHAR(50) UNIQUE NOT NULL,
    subscription_tier VARCHAR(20) NOT NULL
);

CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,  -- Every table has tenant_id
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100),

    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    UNIQUE (tenant_id, email)  -- Email unique per tenant
);

-- Row-level security for automatic tenant isolation
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON customers
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

-- Every index includes tenant_id
CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_tenant_email ON customers(tenant_id, email);
```

**Pattern 2: Separate Schemas per Tenant**

**✅ GOOD for: Small number of tenants, custom per-tenant schema**

```sql
-- Create schema per tenant
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_globex;

-- Each tenant has own tables
CREATE TABLE tenant_acme.customers (...);
CREATE TABLE tenant_globex.customers (...);

-- Application sets search_path
SET search_path = tenant_acme, public;
SELECT * FROM customers;  -- Queries tenant_acme.customers

-- Pros: Full isolation, custom schema per tenant
-- Cons: Schema management overhead, limited to ~1000s of tenants
```

**Pattern 3: Separate Databases per Tenant**

**✅ GOOD for: Full isolation, compliance requirements**

```sql
-- Each tenant gets own database
CREATE DATABASE tenant_acme;
CREATE DATABASE tenant_globex;

-- Connection pooling per tenant
-- Pros: Complete isolation, independent backups/restores
-- Cons: Highest overhead, cross-tenant queries impossible
```

### 5. Audit Trail Design

**✅ GOOD (Comprehensive audit logging):**

```sql
CREATE TABLE audit_log (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name VARCHAR(100) NOT NULL,
    operation VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, DELETE
    record_id UUID NOT NULL,
    user_id UUID,
    occurred_at TIMESTAMP NOT NULL DEFAULT NOW(),
    ip_address INET,
    user_agent TEXT,
    old_values JSONB,
    new_values JSONB,

    -- Partition by time for efficient queries
    CHECK (occurred_at <= NOW())
) PARTITION BY RANGE (occurred_at);

CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_log_2026_02 PARTITION OF audit_log
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Immutable audit log (prevent tampering)
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;
CREATE POLICY no_update_audit ON audit_log FOR UPDATE USING (false);
CREATE POLICY no_delete_audit ON audit_log FOR DELETE USING (false);

-- Generic audit trigger
CREATE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
DECLARE
    audit_user UUID;
BEGIN
    BEGIN
        audit_user := current_setting('app.user_id')::UUID;
    EXCEPTION WHEN OTHERS THEN
        audit_user := NULL;
    END;

    IF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, user_id, old_values)
        VALUES (TG_TABLE_NAME, 'DELETE', OLD.id, audit_user, row_to_json(OLD));
        RETURN OLD;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, user_id, old_values, new_values)
        VALUES (TG_TABLE_NAME, 'UPDATE', NEW.id, audit_user, row_to_json(OLD), row_to_json(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, record_id, user_id, new_values)
        VALUES (TG_TABLE_NAME, 'INSERT', NEW.id, audit_user, row_to_json(NEW));
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Attach trigger to tables requiring audit
CREATE TRIGGER audit_customers
    AFTER INSERT OR UPDATE OR DELETE ON customers
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

---

## 5. Common Mistakes & Antipatterns

### Mistake 1: Entity-Attribute-Value (EAV) Antipattern

**❌ BAD (EAV model):**

```sql
-- Attempting to store flexible attributes
CREATE TABLE entities (
    entity_id UUID PRIMARY KEY,
    entity_type VARCHAR(50)
);

CREATE TABLE attributes (
    attribute_id UUID PRIMARY KEY,
    entity_id UUID,
    attribute_name VARCHAR(100),
    attribute_value TEXT,
    FOREIGN KEY (entity_id) REFERENCES entities(entity_id)
);

-- Querying is a nightmare:
SELECT
    e.entity_id,
    MAX(CASE WHEN a.attribute_name = 'name' THEN a.attribute_value END) AS name,
    MAX(CASE WHEN a.attribute_name = 'email' THEN a.attribute_value END) AS email,
    MAX(CASE WHEN a.attribute_name = 'age' THEN a.attribute_value END) AS age
FROM entities e
JOIN attributes a ON e.entity_id = a.entity_id
WHERE e.entity_type = 'user'
GROUP BY e.entity_id;

-- Problems:
-- 1. No type safety (age stored as TEXT)
-- 2. No constraints (can't enforce email format)
-- 3. Complex queries with multiple CASE statements
-- 4. Poor performance (many JOINs)
-- 5. No referential integrity
```

**✅ GOOD (Proper schema with JSONB for truly flexible data):**

```sql
-- Core attributes as columns
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    age INTEGER CHECK (age >= 0 AND age <= 150),
    created_at TIMESTAMP DEFAULT NOW(),

    -- Flexible metadata as JSONB
    metadata JSONB
);

-- Type-safe queries, indexes on JSONB
CREATE INDEX idx_users_metadata ON users USING GIN (metadata);

-- Query JSONB efficiently
SELECT * FROM users
WHERE metadata->>'subscription' = 'premium'
  AND (metadata->>'newsletter_consent')::boolean = true;

-- Still type-safe for core attributes
SELECT * FROM users WHERE age > 18;
```

### Mistake 2: Storing Lists as Comma-Separated Values

**❌ BAD (CSV in database):**

```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255),
    category_ids TEXT  -- '123,456,789' ← ANTIPATTERN
);

-- Problems:
-- 1. Can't JOIN with categories table
-- 2. Can't enforce foreign key constraints
-- 3. String parsing required
-- 4. Can't index efficiently
-- 5. Hard to query "all products in category X"
```

**✅ GOOD (Junction table):**

```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE product_categories (
    product_id UUID,
    category_id UUID,
    PRIMARY KEY (product_id, category_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE CASCADE
);

-- Proper queries with JOINs
SELECT p.*
FROM products p
JOIN product_categories pc ON p.product_id = pc.product_id
WHERE pc.category_id = ?;

-- Or use array type for simple cases (no referential integrity)
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255),
    tag_names TEXT[]  -- {'electronics', 'sale', 'featured'}
);

-- Array queries
SELECT * FROM products WHERE 'sale' = ANY(tag_names);
```

### Mistake 3: Using NULL for "Not Applicable"

**❌ BAD (Ambiguous NULL):**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    name VARCHAR(100),
    termination_date DATE  -- NULL = active OR unknown?
);

-- Is NULL "still employed" or "termination date not recorded"?
-- Three-valued logic causes bugs
```

**✅ GOOD (Explicit status column):**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    employment_status VARCHAR(20) NOT NULL DEFAULT 'active',
    hire_date DATE NOT NULL,
    termination_date DATE,  -- NULL only when status = 'active'

    CHECK (
        (employment_status = 'active' AND termination_date IS NULL) OR
        (employment_status = 'terminated' AND termination_date IS NOT NULL)
    )
);

-- Clear semantics: NULL means "not applicable"
-- Query active employees explicitly
SELECT * FROM employees WHERE employment_status = 'active';
```

### Mistake 4: God Table Antipattern

**❌ BAD (Mega-table with everything):**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    -- Basic info
    email VARCHAR(255),
    password_hash VARCHAR(255),
    name VARCHAR(100),
    -- Profile
    bio TEXT,
    avatar_url VARCHAR(500),
    date_of_birth DATE,
    -- Preferences
    theme VARCHAR(20),
    language CHAR(2),
    timezone VARCHAR(50),
    -- Subscription
    subscription_tier VARCHAR(20),
    subscription_expires DATE,
    payment_method VARCHAR(50),
    -- Analytics
    last_login TIMESTAMP,
    login_count INTEGER,
    page_views INTEGER,
    -- Support
    support_tier VARCHAR(20),
    assigned_agent_id UUID,
    -- ... 50 more columns
);
-- Problems: Wide rows, scattered data, poor cache locality
```

**✅ GOOD (Decompose into logical entities):**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE user_profiles (
    profile_id UUID PRIMARY KEY,
    user_id UUID NOT NULL UNIQUE,
    name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    date_of_birth DATE,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

CREATE TABLE user_preferences (
    preference_id UUID PRIMARY KEY,
    user_id UUID NOT NULL UNIQUE,
    theme VARCHAR(20) DEFAULT 'light',
    language CHAR(2) DEFAULT 'en',
    timezone VARCHAR(50) DEFAULT 'UTC',
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

CREATE TABLE user_subscriptions (
    subscription_id UUID PRIMARY KEY,
    user_id UUID NOT NULL UNIQUE,
    tier VARCHAR(20) NOT NULL,
    expires_at DATE NOT NULL,
    payment_method_id UUID,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

CREATE TABLE user_analytics (
    user_id UUID PRIMARY KEY,
    last_login TIMESTAMP,
    login_count INTEGER DEFAULT 0,
    page_views INTEGER DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Benefits:
-- 1. Narrow rows = better cache performance
-- 2. Logical separation = easier to understand
-- 3. Can query only needed tables
-- 4. Easier to add columns to specific entities
```

### Mistake 5: Premature Optimization

**❌ BAD (Over-engineering before knowing requirements):**

```sql
-- Day 1 of project, 0 users:
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255)
) PARTITION BY HASH (user_id);  -- Premature partitioning

-- 16 partitions for a table with 100 rows
CREATE TABLE users_p0 PARTITION OF users ...;
CREATE TABLE users_p1 PARTITION OF users ...;
-- ... 14 more partitions

-- Complex sharding logic before hitting 1000 users
-- Result: Unnecessary complexity, harder to develop
```

**✅ GOOD (Start simple, optimize when needed):**

```sql
-- Day 1: Simple schema
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Month 6: 1M users, queries slowing down
-- Add index for common query
CREATE INDEX idx_users_created ON users(created_at DESC);

-- Year 2: 50M users, table size causing issues
-- NOW add partitioning
ALTER TABLE users RENAME TO users_old;
CREATE TABLE users (...) PARTITION BY RANGE (created_at);
-- Migrate data in batches

-- Principle: "Make it work, make it right, make it fast"
-- Optimize based on actual data and usage patterns
```

---

## 6. Security Considerations

### 1. Sensitive Data Isolation

**✅ GOOD (Separate sensitive data):**

```sql
-- Public profile data
CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500)
);

-- Authentication data (restricted access)
CREATE TABLE user_credentials (
    user_id UUID PRIMARY KEY,
    password_hash VARCHAR(255) NOT NULL,
    mfa_secret VARCHAR(255),
    recovery_codes TEXT[],
    failed_login_attempts INTEGER DEFAULT 0,
    locked_until TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES user_profiles(user_id)
);

-- PII (separate database, encrypted)
CREATE TABLE user_pii (
    user_id UUID PRIMARY KEY,
    ssn_encrypted BYTEA,
    dob_encrypted BYTEA,
    address_encrypted BYTEA,
    phone_encrypted BYTEA,
    encryption_key_version INTEGER NOT NULL,
    FOREIGN KEY (user_id) REFERENCES user_profiles(user_id)
);

-- Row-level security
ALTER TABLE user_credentials ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_pii ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_own_credentials ON user_credentials
    USING (user_id = current_setting('app.user_id')::UUID);
```

### 2. SQL Injection Prevention

**Database-level protection:**

```sql
-- Parameterized queries (application layer)
-- ✅ GOOD
SELECT * FROM users WHERE email = $1;  -- Parameterized

-- ❌ BAD
SELECT * FROM users WHERE email = 'user@example.com';  -- String concatenation

-- Database constraints prevent malicious data
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0 AND price <= 1000000),
    stock INTEGER CHECK (stock >= 0)
);

-- Stored procedures with SECURITY DEFINER
CREATE FUNCTION get_user_orders(user_id_param UUID)
RETURNS TABLE (order_id UUID, order_date TIMESTAMP, total DECIMAL)
LANGUAGE plpgsql
SECURITY DEFINER  -- Runs with definer's privileges
AS $$
BEGIN
    -- Validate input
    IF user_id_param IS NULL THEN
        RAISE EXCEPTION 'user_id cannot be NULL';
    END IF;

    RETURN QUERY
    SELECT o.order_id, o.order_date, o.total_amount
    FROM orders o
    WHERE o.customer_id = user_id_param
    ORDER BY o.order_date DESC;
END;
$$;

-- Grant execute only (not direct table access)
REVOKE ALL ON orders FROM app_user;
GRANT EXECUTE ON FUNCTION get_user_orders TO app_user;
```

### 3. Least Privilege Access

**✅ GOOD (Role-based access):**

```sql
-- Read-only role for analytics
CREATE ROLE analytics_read;
GRANT CONNECT ON DATABASE mydb TO analytics_read;
GRANT USAGE ON SCHEMA public TO analytics_read;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analytics_read;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO analytics_read;

-- Application role with limited access
CREATE ROLE app_user;
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE ON customers, orders, products TO app_user;
-- NO DELETE permission (soft deletes only)

-- Admin role for migrations
CREATE ROLE db_admin;
GRANT ALL PRIVILEGES ON DATABASE mydb TO db_admin;

-- No one uses superuser in production
```

---

## 7. Performance Optimization

### 1. Denormalization Strategy

**When to denormalize:**

```
Decision Matrix:
┌──────────────────────────────────────────────────────────┐
│  Normalize When:           │  Denormalize When:          │
├────────────────────────────┼─────────────────────────────┤
│ • Data changes frequently  │ • Read:Write ratio > 100:1  │
│ • Strong consistency needed│ • JOIN involves 5+ tables   │
│ • Low read volume          │ • Query SLA < 50ms          │
│ • Complex update patterns  │ • Data rarely changes       │
│                            │ • Acceptable staleness      │
└────────────────────────────┴─────────────────────────────┘
```

**✅ GOOD (Strategic denormalization):**

```sql
-- Normalized base tables
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    base_price DECIMAL(10,2) NOT NULL
);

CREATE TABLE product_reviews (
    review_id UUID PRIMARY KEY,
    product_id UUID NOT NULL,
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Denormalized materialized view for product listing
CREATE MATERIALIZED VIEW product_catalog AS
SELECT
    p.product_id,
    p.name,
    p.base_price,
    COUNT(r.review_id) AS review_count,
    COALESCE(AVG(r.rating), 0) AS avg_rating,
    MIN(r.created_at) AS first_review_date,
    MAX(r.created_at) AS last_review_date
FROM products p
LEFT JOIN product_reviews r ON p.product_id = r.product_id
GROUP BY p.product_id, p.name, p.base_price;

-- Indexes on materialized view
CREATE INDEX idx_product_catalog_rating ON product_catalog(avg_rating DESC);
CREATE INDEX idx_product_catalog_reviews ON product_catalog(review_count DESC);

-- Refresh strategy (incremental)
REFRESH MATERIALIZED VIEW CONCURRENTLY product_catalog;

-- Or use triggers for real-time updates
CREATE FUNCTION refresh_product_catalog()
RETURNS TRIGGER AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY product_catalog;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_refresh_catalog
    AFTER INSERT OR UPDATE OR DELETE ON product_reviews
    FOR EACH STATEMENT
    EXECUTE FUNCTION refresh_product_catalog();
```

### 2. Partitioning Strategy

**✅ GOOD (Range partitioning by time):**

```sql
-- Orders table partitioned by month
CREATE TABLE orders (
    order_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_date, order_id)
) PARTITION BY RANGE (order_date);

-- Create partitions
CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE orders_2026_03 PARTITION OF orders
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

-- Indexes on each partition
CREATE INDEX idx_orders_2026_01_customer ON orders_2026_01(customer_id);
CREATE INDEX idx_orders_2026_02_customer ON orders_2026_02(customer_id);

-- Automatic partition pruning
EXPLAIN SELECT * FROM orders
WHERE order_date BETWEEN '2026-02-01' AND '2026-02-28';
-- Only scans orders_2026_02 partition

-- Automate partition creation
CREATE OR REPLACE FUNCTION create_next_month_partition()
RETURNS void AS $$
DECLARE
    next_month DATE;
    partition_name TEXT;
BEGIN
    next_month := date_trunc('month', NOW()) + INTERVAL '2 months';
    partition_name := 'orders_' || to_char(next_month, 'YYYY_MM');

    EXECUTE format('
        CREATE TABLE IF NOT EXISTS %I PARTITION OF orders
        FOR VALUES FROM (%L) TO (%L)',
        partition_name,
        next_month,
        next_month + INTERVAL '1 month'
    );
END;
$$ LANGUAGE plpgsql;

-- Schedule monthly
-- (Use pg_cron or external scheduler)
```

### 3. Caching Strategy in Schema

**✅ GOOD (Cache-friendly schema):**

```sql
-- Hot data (frequently accessed)
CREATE TABLE user_sessions (
    session_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,
    data JSONB
) WITH (fillfactor = 90);  -- Leave room for HOT updates

-- Separate hot and cold data
CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY,
    -- Hot columns (accessed every request)
    username VARCHAR(50) NOT NULL,
    display_name VARCHAR(100),
    avatar_url VARCHAR(500),
    -- Cold columns (rarely accessed) in separate table
);

CREATE TABLE user_profile_details (
    user_id UUID PRIMARY KEY,
    bio TEXT,
    interests TEXT[],
    education JSONB,
    work_history JSONB,
    FOREIGN KEY (user_id) REFERENCES user_profiles(user_id)
);

-- Cache invalidation hints
CREATE TABLE cache_invalidation (
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    invalidated_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (entity_type, entity_id)
);

-- Trigger to record cache invalidations
CREATE FUNCTION mark_cache_invalid()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO cache_invalidation (entity_type, entity_id)
    VALUES (TG_TABLE_NAME, NEW.id)
    ON CONFLICT (entity_type, entity_id) DO UPDATE
    SET invalidated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## 8. Real-World Use Cases

### Use Case 1: E-Commerce Schema (Production-Grade)

```sql
-- Complete e-commerce schema following all principles

-- Core entities
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    phone VARCHAR(20),
    email_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP,  -- Soft delete

    CONSTRAINT uk_active_email UNIQUE (email) WHERE deleted_at IS NULL
);

CREATE TABLE addresses (
    address_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL,
    address_type VARCHAR(20) NOT NULL CHECK (address_type IN ('shipping', 'billing')),
    is_default BOOLEAN DEFAULT false,
    recipient_name VARCHAR(100) NOT NULL,
    street_line1 VARCHAR(255) NOT NULL,
    street_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50),
    postal_code VARCHAR(20) NOT NULL,
    country CHAR(2) NOT NULL,
    phone VARCHAR(20),

    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE
);

CREATE TABLE products (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    base_price DECIMAL(10,2) NOT NULL CHECK (base_price >= 0),
    cost DECIMAL(10,2) NOT NULL CHECK (cost >= 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    reorder_level INTEGER DEFAULT 10,
    weight_kg DECIMAL(8,3),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP
);

CREATE TABLE categories (
    category_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    parent_category_id UUID,
    display_order INTEGER,
    FOREIGN KEY (parent_category_id) REFERENCES categories(category_id)
);

CREATE TABLE product_categories (
    product_id UUID,
    category_id UUID,
    is_primary BOOLEAN DEFAULT false,
    PRIMARY KEY (product_id, category_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE CASCADE
);

-- Orders (partitioned by date)
CREATE TABLE orders (
    order_id UUID NOT NULL DEFAULT gen_random_uuid(),
    order_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL DEFAULT NOW(),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    subtotal DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) NOT NULL,
    shipping_amount DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    total_amount DECIMAL(10,2) NOT NULL,
    shipping_address_id UUID NOT NULL,
    billing_address_id UUID NOT NULL,
    notes TEXT,

    PRIMARY KEY (order_date, order_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(address_id),
    FOREIGN KEY (billing_address_id) REFERENCES addresses(address_id),

    CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded')),
    CHECK (total_amount = subtotal + tax_amount + shipping_amount - discount_amount)
) PARTITION BY RANGE (order_date);

-- Create initial partitions
CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE orders_2026_03 PARTITION OF orders
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL,  -- For partitioning
    product_id UUID NOT NULL,
    sku VARCHAR(50) NOT NULL,  -- Snapshot at time of order
    product_name VARCHAR(255) NOT NULL,  -- Snapshot
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,  -- Price at time of order
    subtotal DECIMAL(10,2) NOT NULL,

    FOREIGN KEY (order_id, order_date) REFERENCES orders(order_id, order_date) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id),

    CHECK (subtotal = quantity * unit_price)
);

-- Payment handling
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_method VARCHAR(20) NOT NULL,
    amount DECIMAL(10,2) NOT NULL CHECK (amount > 0),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    transaction_id VARCHAR(255),
    processed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),

    CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'refunded')),
    CHECK (payment_method IN ('credit_card', 'debit_card', 'paypal', 'stripe', 'gift_card'))
);

CREATE TABLE order_payments (
    order_id UUID,
    order_date TIMESTAMP,
    payment_id UUID,
    amount DECIMAL(10,2) NOT NULL,

    PRIMARY KEY (order_id, payment_id),
    FOREIGN KEY (order_id, order_date) REFERENCES orders(order_id, order_date),
    FOREIGN KEY (payment_id) REFERENCES payments(payment_id)
);

-- Indexes for performance
CREATE INDEX idx_customers_email ON customers(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_addresses_customer ON addresses(customer_id);
CREATE INDEX idx_products_sku ON products(sku) WHERE deleted_at IS NULL;
CREATE INDEX idx_product_categories_category ON product_categories(category_id);
CREATE INDEX idx_orders_customer ON orders(customer_id, order_date DESC);
CREATE INDEX idx_orders_status ON orders(status, order_date DESC) WHERE status != 'delivered';
CREATE INDEX idx_order_items_product ON order_items(product_id);

-- Audit triggers
CREATE TRIGGER audit_customers AFTER INSERT OR UPDATE OR DELETE ON customers
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
CREATE TRIGGER audit_orders AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

### Use Case 2: SaaS Multi-Tenant Platform

```sql
-- Full multi-tenant schema with row-level security

CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_slug VARCHAR(50) UNIQUE NOT NULL,
    company_name VARCHAR(255) NOT NULL,
    subscription_tier VARCHAR(20) NOT NULL DEFAULT 'free',
    max_users INTEGER NOT NULL DEFAULT 5,
    max_storage_gb INTEGER NOT NULL DEFAULT 10,
    is_active BOOLEAN DEFAULT true,
    trial_ends_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),

    CHECK (subscription_tier IN ('free', 'starter', 'professional', 'enterprise'))
);

CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'member',
    is_active BOOLEAN DEFAULT true,
    last_login TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),

    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    UNIQUE (tenant_id, email),
    CHECK (role IN ('owner', 'admin', 'member', 'viewer'))
);

-- Row-level security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

CREATE POLICY tenant_admins_see_all ON users
    USING (
        tenant_id = current_setting('app.current_tenant')::UUID
        AND EXISTS (
            SELECT 1 FROM users u
            WHERE u.user_id = current_setting('app.user_id')::UUID
              AND u.role IN ('owner', 'admin')
        )
    );

-- Apply to all tables
CREATE TABLE projects (
    project_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_by UUID NOT NULL,
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    FOREIGN KEY (created_by) REFERENCES users(user_id)
);

ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON projects
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

-- Tenant usage tracking
CREATE TABLE tenant_usage (
    usage_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    metric_name VARCHAR(50) NOT NULL,
    metric_value BIGINT NOT NULL,
    recorded_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE
);
```

---

## 9. Design Patterns

### Pattern 1: Temporal Table Pattern

```sql
-- Bitemporal table (system time + business time)
CREATE TABLE product_prices (
    price_id UUID PRIMARY KEY,
    product_id UUID NOT NULL,
    price DECIMAL(10,2) NOT NULL,

    -- Business time (when price is/was valid in real world)
    valid_from DATE NOT NULL,
    valid_to DATE,  -- NULL = current

    -- System time (when record was created/modified in database)
    system_time_start TIMESTAMP NOT NULL DEFAULT NOW(),
    system_time_end TIMESTAMP,  -- NULL = current version

    FOREIGN KEY (product_id) REFERENCES products(product_id),

    EXCLUDE USING gist (
        product_id WITH =,
        daterange(valid_from, valid_to, '[)') WITH &&
    ) WHERE (system_time_end IS NULL)
);

-- Query: "What was the price on 2025-12-25?"
SELECT price FROM product_prices
WHERE product_id = ?
  AND valid_from <= '2025-12-25'
  AND (valid_to IS NULL OR valid_to > '2025-12-25')
  AND system_time_end IS NULL;

-- Query: "What did we think the price was on 2025-12-25, as of 2026-01-01?"
SELECT price FROM product_prices
WHERE product_id = ?
  AND valid_from <= '2025-12-25'
  AND (valid_to IS NULL OR valid_to > '2025-12-25')
  AND system_time_start <= '2026-01-01'
  AND (system_time_end IS NULL OR system_time_end > '2026-01-01');
```

### Pattern 2: Polymorphic Associations Pattern

```sql
-- Tagging system (tags apply to multiple entity types)
CREATE TABLE tags (
    tag_id UUID PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE taggings (
    tagging_id UUID PRIMARY KEY,
    tag_id UUID NOT NULL,
    taggable_type VARCHAR(50) NOT NULL,  -- 'product', 'post', 'user'
    taggable_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),

    FOREIGN KEY (tag_id) REFERENCES tags(tag_id) ON DELETE CASCADE,
    UNIQUE (tag_id, taggable_type, taggable_id)
);

-- Enforce referential integrity with triggers
CREATE FUNCTION check_taggable_exists()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.taggable_type = 'product' AND NOT EXISTS (
        SELECT 1 FROM products WHERE product_id = NEW.taggable_id
    ) THEN
        RAISE EXCEPTION 'Product % does not exist', NEW.taggable_id;
    ELSIF NEW.taggable_type = 'post' AND NOT EXISTS (
        SELECT 1 FROM posts WHERE post_id = NEW.taggable_id
    ) THEN
        RAISE EXCEPTION 'Post % does not exist', NEW.taggable_id;
    ELSIF NEW.taggable_type = 'user' AND NOT EXISTS (
        SELECT 1 FROM users WHERE user_id = NEW.taggable_id
    ) THEN
        RAISE EXCEPTION 'User % does not exist', NEW.taggable_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_taggable_integrity
    BEFORE INSERT OR UPDATE ON taggings
    FOR EACH ROW EXECUTE FUNCTION check_taggable_exists();

-- Query: Get all tags for a product
SELECT t.* FROM tags t
JOIN taggings tg ON t.tag_id = tg.tag_id
WHERE tg.taggable_type = 'product' AND tg.taggable_id = ?;
```

### Pattern 3: Event Sourcing Pattern

```sql
-- Event store (append-only log)
CREATE TABLE events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(50) NOT NULL,  -- 'Order', 'User', 'Product'
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_version INTEGER NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB,
    occurred_at TIMESTAMP NOT NULL DEFAULT NOW(),
    user_id UUID,
    sequence_number BIGSERIAL,

    -- Immutable
    CHECK (occurred_at <= NOW())
) PARTITION BY RANGE (occurred_at);

-- Prevent updates/deletes
ALTER TABLE events ENABLE ROW LEVEL SECURITY;
CREATE POLICY no_modify_events ON events FOR UPDATE USING (false);
CREATE POLICY no_delete_events ON events FOR DELETE USING (false);

-- Materialized view (current state)
CREATE TABLE order_snapshots (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    status VARCHAR(20) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    items JSONB NOT NULL,
    version INTEGER NOT NULL,
    last_event_id UUID NOT NULL,
    snapshot_at TIMESTAMP NOT NULL
);

-- Rebuild state from events
CREATE FUNCTION rebuild_order_snapshot(order_id_param UUID)
RETURNS void AS $$
DECLARE
    event_rec RECORD;
    current_state JSONB := '{}';
BEGIN
    FOR event_rec IN
        SELECT * FROM events
        WHERE aggregate_type = 'Order' AND aggregate_id = order_id_param
        ORDER BY sequence_number
    LOOP
        -- Apply event to state
        IF event_rec.event_type = 'OrderCreated' THEN
            current_state := event_rec.event_data;
        ELSIF event_rec.event_type = 'ItemAdded' THEN
            -- Merge item into state
            current_state := current_state || event_rec.event_data;
        -- ... handle other event types
        END IF;
    END LOOP;

    -- Upsert snapshot
    INSERT INTO order_snapshots (order_id, customer_id, status, ...)
    VALUES (...) -- Extract from current_state
    ON CONFLICT (order_id) DO UPDATE SET ...;
END;
$$ LANGUAGE plpgsql;
```

---

## 10. Interview Questions

### Junior Level (0-2 years)

**Q1: What are the key principles of good schema design?**

**Answer:**

1. **Correctness**: Enforce business rules at the database level (constraints, foreign keys)
2. **Appropriate Data Types**: Use specific types (DECIMAL for money, not FLOAT)
3. **Normalization**: Eliminate redundancy (start with 3NF)
4. **Referential Integrity**: Use foreign keys to maintain relationships
5. **Indexing Strategy**: Index foreign keys and frequently queried columns
6. **Clear Naming**: Consistent, descriptive table/column names

**Q2: When should you use soft deletes vs hard deletes?**

**Answer:**

- **Soft Deletes** (deleted_at timestamp): When you need audit trails, compliance requirements, or ability to restore (e.g., customer accounts, orders)
- **Hard Deletes** (actual DELETE): When data has no historical value (e.g., session tokens, temporary carts, expired tokens)

**Q3: Explain the difference between CHAR, VARCHAR, and TEXT data types.**

**Answer:**

- **CHAR(n)**: Fixed-length, pads with spaces, use for fixed-width data (e.g., country codes 'US', 'UK')
- **VARCHAR(n)**: Variable-length up to n characters, use when you know max length (e.g., email VARCHAR(255))
- **TEXT**: Variable-length, no limit, use for long content (e.g., blog posts, descriptions)

Performance: CHAR fastest for fixed-width, VARCHAR for most cases, TEXT for large content.

### Mid-Level (2-5 years)

**Q4: How do you handle temporal data (historical changes) in a schema?**

**Answer:**
Create separate history/audit tables with temporal columns:

```sql
-- Current state table
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    name VARCHAR(100),
    salary DECIMAL(10,2)
);

-- History table
CREATE TABLE employee_salary_history (
    history_id UUID PRIMARY KEY,
    employee_id UUID NOT NULL,
    salary DECIMAL(10,2) NOT NULL,
    effective_from TIMESTAMP NOT NULL,
    effective_to TIMESTAMP,  -- NULL = current
    changed_by UUID NOT NULL,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    EXCLUDE USING gist (
        employee_id WITH =,
        tstzrange(effective_from, effective_to, '[)') WITH &&
    )
);
```

Use triggers to automatically populate history tables on changes.

**Q5: How would you design a multi-tenant SaaS database schema?**

**Answer:**
Three approaches:

1. **Shared Schema** (best for large # of tenants):
   - Add `tenant_id` to every table
   - Use row-level security for isolation
   - Efficient, but requires careful query filtering

2. **Separate Schemas** (best for medium # of tenants):
   - One schema per tenant
   - Complete isolation
   - Limited to ~1000s of tenants

3. **Separate Databases** (best for compliance/full isolation):
   - One database per tenant
   - Highest overhead
   - Complete isolation, independent backups

Choice depends on: number of tenants, data volume per tenant, compliance requirements, operational complexity.

**Q6: What is the Entity-Attribute-Value (EAV) antipattern and why should it be avoided?**

**Answer:**
EAV stores all data as key-value pairs instead of proper columns:

```sql
-- ❌ EAV Antipattern
entities: (entity_id, entity_type)
attributes: (entity_id, attribute_name, attribute_value)

Problems:
1. No type safety (all values TEXT)
2. No constraints/foreign keys
3. Complex queries with multiple JOINs
4. Poor performance
5. Difficult to maintain
```

**Solution:** Use proper columns for known attributes, JSONB for truly flexible data.

### Senior Level (5-8 years)

**Q7: Design a schema for a financial ledger system with immutability and audit requirements.**

**Answer:**

```sql
-- Immutable append-only ledger
CREATE TABLE ledger_entries (
    entry_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL,
    transaction_id UUID NOT NULL,
    entry_type VARCHAR(10) NOT NULL CHECK (entry_type IN ('debit', 'credit')),
    amount DECIMAL(20,8) NOT NULL CHECK (amount > 0),
    currency CHAR(3) NOT NULL,
    balance_after DECIMAL(20,8) NOT NULL,
    occurred_at TIMESTAMP NOT NULL DEFAULT NOW(),
    sequence_number BIGSERIAL,
    metadata JSONB,

    -- Immutable: prevent updates/deletes
    CHECK (occurred_at <= NOW()),
    FOREIGN KEY (account_id) REFERENCES accounts(account_id)
) PARTITION BY RANGE (occurred_at);

-- Prevent modifications
ALTER TABLE ledger_entries ENABLE ROW LEVEL SECURITY;
CREATE POLICY no_update ON ledger_entries FOR UPDATE USING (false);
CREATE POLICY no_delete ON ledger_entries FOR DELETE USING (false);

-- Account balance snapshots
CREATE TABLE account_balances (
    account_id UUID PRIMARY KEY,
    current_balance DECIMAL(20,8) NOT NULL,
    last_entry_id UUID NOT NULL,
    last_updated TIMESTAMP NOT NULL,
    FOREIGN KEY (account_id) REFERENCES accounts(account_id),
    FOREIGN KEY (last_entry_id) REFERENCES ledger_entries(entry_id)
);

-- Double-entry bookkeeping constraint
CREATE FUNCTION check_double_entry()
RETURNS TRIGGER AS $$
BEGIN
    -- Verify debits = credits for transaction
    IF (SELECT SUM(CASE WHEN entry_type = 'debit' THEN amount ELSE -amount END)
        FROM ledger_entries
        WHERE transaction_id = NEW.transaction_id) != 0 THEN
        RAISE EXCEPTION 'Transaction must balance (debits = credits)';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE CONSTRAINT TRIGGER enforce_double_entry
    AFTER INSERT ON ledger_entries
    DEFERRABLE INITIALLY DEFERRED
    FOR EACH ROW EXECUTE FUNCTION check_double_entry();
```

**Q8: How do you handle schema migrations with zero downtime for a high-traffic application?**

**Answer:**
Use the expand-contract pattern:

1. **Expand Phase** (additive changes only):
   - Add new column (nullable)
   - Add new table
   - Deploy application version supporting both old and new schema

2. **Dual-Write Phase**:
   - Application writes to both old and new columns/tables
   - Backfill old data to new structure

3. **Migrate Reads**:
   - Switch reads to new structure
   - Monitor and validate

4. **Contract Phase**:
   - Remove old column/table
   - Deploy application using only new schema

Example:

```sql
-- Phase 1: Add new column (nullable)
ALTER TABLE users ADD COLUMN email_new VARCHAR(255);
CREATE INDEX CONCURRENTLY idx_users_email_new ON users(email_new);

-- Phase 2: Backfill (in batches)
UPDATE users SET email_new = LOWER(email) WHERE email_new IS NULL LIMIT 10000;

-- Phase 3: Make NOT NULL (after backfill complete)
ALTER TABLE users ALTER COLUMN email_new SET NOT NULL;

-- Phase 4: Switch application to use email_new

-- Phase 5: Drop old column (after validation)
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users RENAME COLUMN email_new TO email;
```

### Staff/Principal Level (8+ years)

**Q9: Design a globally distributed schema for a social network with 1B+ users, considering consistency, availability, and partition tolerance trade-offs.**

**Answer:**
Multi-region active-active architecture with eventual consistency:

```sql
-- User profile (CP system - strong consistency required)
-- Stored in user's home region, replicated synchronously
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    home_region VARCHAR(20) NOT NULL,  -- 'us-east', 'eu-west', 'ap-south'
    created_at TIMESTAMP NOT NULL,

    -- Conflict-free replicated data type (CRDT) metadata
    version_vector JSONB  -- {'us-east': 5, 'eu-west': 3}
);

-- Partition by user home region
CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY,
    bio TEXT,
    avatar_url VARCHAR(500),
    updated_at TIMESTAMP NOT NULL,
    updated_region VARCHAR(20) NOT NULL,
    vector_clock JSONB,  -- Lamport timestamp for conflict resolution

    FOREIGN KEY (user_id) REFERENCES users(user_id)
) PARTITION BY LIST (home_region);

-- Posts (AP system - high availability, eventual consistency)
-- Written to local region, async replicated globally
CREATE TABLE posts (
    post_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    origin_region VARCHAR(20) NOT NULL,
    tombstone BOOLEAN DEFAULT false,  -- For distributed deletes

    -- Global secondary index for user timeline
    FOREIGN KEY (user_id) REFERENCES users(user_id)
) PARTITION BY HASH (user_id);

-- Relationships (AP system with anti-entropy)
CREATE TABLE user_relationships (
    follower_id UUID,
    following_id UUID,
    created_at TIMESTAMP NOT NULL,
    origin_region VARCHAR(20) NOT NULL,
    PRIMARY KEY (follower_id, following_id)
) PARTITION BY HASH (follower_id);

-- Regional feed cache (eventually consistent, rebuilt periodically)
CREATE MATERIALIZED VIEW user_feed_cache AS
SELECT
    ur.follower_id AS user_id,
    p.post_id,
    p.content,
    p.created_at,
    p.user_id AS author_id
FROM user_relationships ur
JOIN posts p ON ur.following_id = p.user_id
WHERE p.created_at > NOW() - INTERVAL '7 days'
  AND p.tombstone = false;

-- Conflict resolution strategy
CREATE FUNCTION resolve_profile_conflict()
RETURNS TRIGGER AS $$
BEGIN
    -- Last-write-wins with vector clocks
    IF NEW.vector_clock->NEW.updated_region > OLD.vector_clock->NEW.updated_region THEN
        RETURN NEW;
    ELSE
        RETURN OLD;  -- Reject stale update
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Consistency guarantees:
-- - User profile writes: Synchronous multi-region (CP)
-- - Posts: Async replication (AP)
-- - Follows: Async with anti-entropy reconciliation
-- - Reads: Serve from nearest region (eventual consistency acceptable)
```

**Key Design Decisions:**

1. **Partitioning**: User-based sharding (co-locate user data)
2. **Replication**: Synchronous for critical data (profile), async for high-volume (posts)
3. **Conflict Resolution**: Vector clocks + last-write-wins
4. **Caching**: Regional feed caches rebuilt hourly
5. **Consistency**: CP for user data, AP for social graph

**Q10: How would you design a schema to support GDPR right to erasure across a complex system with 200+ tables and data in multiple systems?**

**Answer:**
Implement a data lineage tracking and orchestrated deletion system:

```sql
-- Data catalog: Track PII across system
CREATE TABLE data_catalog (
    catalog_id UUID PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    column_name VARCHAR(100) NOT NULL,
    data_classification VARCHAR(50) NOT NULL,  -- 'PII', 'SENSITIVE', 'PUBLIC'
    contains_user_data BOOLEAN NOT NULL,
    retention_days INTEGER,
    anonymization_function VARCHAR(255),  -- Function to anonymize data

    UNIQUE (table_name, column_name)
);

-- User data map: Track where user's data lives
CREATE TABLE user_data_locations (
    mapping_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    data_classification VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL,

    INDEX (user_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Deletion requests
CREATE TABLE deletion_requests (
    request_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    requested_at TIMESTAMP NOT NULL DEFAULT NOW(),
    verified_at TIMESTAMP,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    completed_at TIMESTAMP,

    CHECK (status IN ('pending', 'verified', 'in_progress', 'completed', 'failed')),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Deletion tasks (orchestrates deletion across tables)
CREATE TABLE deletion_tasks (
    task_id UUID PRIMARY KEY,
    request_id UUID NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    record_ids UUID[] NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT,

    FOREIGN KEY (request_id) REFERENCES deletion_requests(request_id)
);

-- Audit log (immutable record of deletion)
CREATE TABLE deletion_audit (
    audit_id UUID PRIMARY KEY,
    request_id UUID NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    deleted_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deletion_method VARCHAR(50) NOT NULL,  -- 'hard_delete', 'anonymize', 'archive'
    backup_location VARCHAR(500),  -- For compliance

    FOREIGN KEY (request_id) REFERENCES deletion_requests(request_id)
) PARTITION BY RANGE (deleted_at);

-- Orchestration function
CREATE FUNCTION execute_deletion_request(request_id_param UUID)
RETURNS void AS $$
DECLARE
    task_rec RECORD;
BEGIN
    -- Create deletion tasks for each table
    INSERT INTO deletion_tasks (request_id, table_name, record_ids)
    SELECT
        request_id_param,
        table_name,
        array_agg(record_id)
    FROM user_data_locations
    WHERE user_id = (SELECT user_id FROM deletion_requests WHERE request_id = request_id_param)
    GROUP BY table_name;

    -- Execute each task
    FOR task_rec IN SELECT * FROM deletion_tasks WHERE request_id = request_id_param LOOP
        -- Dynamic deletion based on classification
        IF (SELECT data_classification FROM data_catalog WHERE table_name = task_rec.table_name LIMIT 1) = 'PII' THEN
            EXECUTE format('DELETE FROM %I WHERE id = ANY($1)', task_rec.table_name) USING task_rec.record_ids;
        ELSE
            -- Anonymize instead of delete
            EXECUTE format('UPDATE %I SET ... = %I(...) WHERE id = ANY($1)',
                          task_rec.table_name,
                          (SELECT anonymization_function FROM data_catalog WHERE table_name = task_rec.table_name LIMIT 1),
                          task_rec.record_ids);
        END IF;

        -- Log audit trail
        INSERT INTO deletion_audit (request_id, table_name, record_id, deletion_method)
        SELECT request_id_param, task_rec.table_name, unnest(task_rec.record_ids), 'hard_delete';

        UPDATE deletion_tasks SET status = 'completed', completed_at = NOW() WHERE task_id = task_rec.task_id;
    END LOOP;

    UPDATE deletion_requests SET status = 'completed', completed_at = NOW() WHERE request_id = request_id_param;
END;
$$ LANGUAGE plpgsql;
```

**Implementation Strategy:**

1. **Data Discovery**: Catalog all PII across 200+ tables
2. **Tracking**: Automatically log user data locations on INSERT
3. **Request Handling**: 30-day verification period
4. **Orchestrated Deletion**: Async task-based deletion
5. **Audit Trail**: Immutable log of all deletions
6. **Cross-System**: API to trigger deletion in external systems (S3, analytics, etc.)
7. **Compliance**: Archive deleted data in encrypted storage for legal requirements

---

## 11. Tools & Resources

### Schema Design Tools

**Modeling:**

- **dbdiagram.io**: Text-to-diagram schema design
- **draw.io**: Visual ER diagrams
- **pgModeler**: PostgreSQL-specific modeling
- **MySQL Workbench**: MySQL schema design

**Migration Tools:**

- **Liquibase**: Database-agnostic migrations
- **Flyway**: Java-based migrations
- **Alembic**: Python migrations (SQLAlchemy)
- **migrate**: Go database migrations

**Schema Analysis:**

```bash
# PostgreSQL: Analyze schema
psql -d mydb -c "\d+ table_name"

# Find missing indexes on foreign keys
SELECT
    tc.table_name,
    kcu.column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu ON tc.constraint_name = kcu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND NOT EXISTS (
    SELECT 1 FROM pg_indexes
    WHERE tablename = tc.table_name
      AND indexdef LIKE '%' || kcu.column_name || '%'
  );

# Check table sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Best Practices Checklist

```markdown
Schema Design Checklist:
☐ Requirements documented (access patterns, consistency needs)
☐ Entities identified and normalized (3NF minimum)
☐ Primary keys defined (UUID for distributed, BIGSERIAL for single-server)
☐ Foreign keys with ON DELETE/UPDATE actions
☐ CHECK constraints for business rules
☐ Appropriate data types (DECIMAL for money, TIMESTAMP WITH TIME ZONE)
☐ Indexes on foreign keys and frequently queried columns
☐ Unique constraints where applicable
☐ NOT NULL constraints on required fields
☐ Soft delete columns if needed (deleted_at)
☐ Audit trail tables for sensitive data
☐ Partitioning strategy for large tables (>100M rows)
☐ Multi-tenancy isolation (if applicable)
☐ Consistent naming conventions
☐ Security: Row-level security, encrypted sensitive columns
☐ Documentation: ER diagrams, business rules, cardinality decisions
☐ Migration strategy: Zero-downtime deployments
☐ Monitoring: Query performance, index usage, table sizes
```

---

## Summary

Schema design principles form the foundation of reliable, performant, and maintainable database systems:

1. **Start with Requirements**: Analyze access patterns, consistency needs, and scale before choosing structure
2. **Normalize First**: Start with 3NF, denormalize strategically based on real usage data
3. **Enforce at Database Level**: Constraints, foreign keys, and triggers prevent data corruption
4. **Appropriate Types**: Use specific data types (DECIMAL for money, TIMESTAMP WITH TIME ZONE)
5. **Index Strategically**: Foreign keys + frequently queried columns, avoid over-indexing
6. **Design for Change**: Schema migrations should be zero-downtime (expand-contract pattern)
7. **Security by Design**: Separate sensitive data, row-level security, audit trails
8. **Document Decisions**: ER diagrams + business rules + change history

**Critical Success Factors:**

- Requirements-first, not technology-first
- Database constraints enforce business rules
- Strategic denormalization based on read:write ratios
- Temporal modeling for historical data
- Multi-tenancy isolation patterns
- Comprehensive audit trails
- Zero-downtime migration strategies

Well-designed schemas prevent costly production issues, enable system evolution, and scale gracefully from prototype to billions of rows.
