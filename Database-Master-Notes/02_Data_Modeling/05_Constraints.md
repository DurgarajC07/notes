# 🔒 Database Constraints - Enforcing Data Integrity

> Constraints are the rules that ensure data quality and consistency. They prevent invalid data from entering the database and maintain referential integrity across tables.

---

## 📖 1. Concept Explanation

### What are Database Constraints?

**Constraints** are rules enforced by the database to ensure data validity, consistency, and integrity. They act as gatekeepers, rejecting invalid data before it corrupts your database.

**Why Constraints Matter:**
- **Data quality**: Prevent invalid/corrupt data
- **Business rules**: Enforce domain logic at database level
- **Referential integrity**: Maintain relationships between tables
- **Security**: Additional layer of validation
- **Performance**: Enable query optimizations

---

### Core Constraint Types

```
DATABASE CONSTRAINTS
│
├── PRIMARY KEY
│   └── Uniquely identifies each row (NOT NULL + UNIQUE)
│
├── FOREIGN KEY
│   └── Enforces relationships between tables
│
├── UNIQUE
│   └── Ensures no duplicate values in column(s)
│
├── NOT NULL
│   └── Requires a value (no NULLs allowed)
│
├── CHECK
│   └── Validates data against custom conditions
│
├── DEFAULT
│   └── Provides default value when none specified
│
└── EXCLUSION (PostgreSQL)
    └── Ensures no overlapping values (time ranges, etc.)
```

---

## 🧠 2. Why It Matters in Real Systems

### Without Constraints - Data Chaos

```sql
-- No constraints
CREATE TABLE users (
    id INT,
    email VARCHAR(255),
    age INT,
    balance DECIMAL(10,2)
);

-- Disasters waiting to happen:
INSERT INTO users VALUES (1, 'user@example.com', 150, 100.00);  -- Age 150? ❌
INSERT INTO users VALUES (2, 'user@example.com', 25, 100.00);   -- Duplicate email ❌
INSERT INTO users VALUES (3, NULL, 30, 100.00);                 -- No email ❌
INSERT INTO users VALUES (4, 'user4@test.com', 25, -50.00);     -- Negative balance ❌
INSERT INTO users VALUES (1, 'user5@test.com', 25, 100.00);     -- Duplicate ID ❌

-- All accepted! Database is now corrupt.
```

---

### With Constraints - Data Integrity

```sql
-- Properly constrained table
CREATE TABLE users (
    id BIGINT PRIMARY KEY,                    -- Must be unique, non-null
    email VARCHAR(255) UNIQUE NOT NULL,       -- Must be unique and present
    age INT CHECK (age BETWEEN 0 AND 120),    -- Valid age range
    balance DECIMAL(10,2) CHECK (balance >= 0), -- No negative balance
    created_at TIMESTAMP DEFAULT NOW()        -- Auto-set if not provided
);

-- Invalid data rejected:
INSERT INTO users VALUES (1, 'user@example.com', 150, 100.00);
-- ERROR: Check constraint "age" violated

INSERT INTO users VALUES (2, 'user@example.com', 25, 100.00);
-- ERROR: Unique constraint "email" violated

INSERT INTO users VALUES (3, NULL, 30, 100.00);
-- ERROR: NOT NULL constraint violated

-- Only valid data accepted ✅
```

---

### Real-World Impact

| Scenario | Without Constraints | With Constraints |
|----------|---------------------|------------------|
| **Duplicate users** | Multiple accounts per email | One account per email |
| **Orphaned records** | Orders without customers | Orders must reference valid customers |
| **Invalid data** | Age = 999, price = -100 | Values within valid ranges |
| **Production bugs** | Crashes from NULL errors | NULL handling enforced at DB level |
| **Data quality** | Garbage in, garbage out | Clean, trustworthy data |

---

## ⚙️ 3. Internal Working

### PRIMARY KEY Internals

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY
);

-- Internally creates:
-- 1. NOT NULL constraint
-- 2. UNIQUE constraint
-- 3. B-Tree index on id (for fast lookups)

-- What happens on INSERT:
INSERT INTO orders VALUES (1);
-- ✅ Accepted

INSERT INTO orders VALUES (1);
-- ❌ ERROR: duplicate key violates unique constraint "orders_pkey"

INSERT INTO orders VALUES (NULL);
-- ❌ ERROR: null value violates not-null constraint
```

**Composite Primary Key:**
```sql
CREATE TABLE order_items (
    order_id BIGINT,
    product_id BIGINT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)  -- Combination must be unique
);

-- Valid: Different combinations
INSERT INTO order_items VALUES (1, 100, 5);
INSERT INTO order_items VALUES (1, 200, 3);  -- ✅ Different product
INSERT INTO order_items VALUES (2, 100, 2);  -- ✅ Different order

-- Invalid: Duplicate combination
INSERT INTO order_items VALUES (1, 100, 10);  -- ❌ Duplicate (1, 100)
```

---

### FOREIGN KEY Internals

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- What happens on INSERT:
INSERT INTO orders VALUES (1, 999);
-- ❌ ERROR: foreign key violation - customer 999 doesn't exist

-- Must insert customer first:
INSERT INTO customers VALUES (999, 'John Doe');  -- ✅
INSERT INTO orders VALUES (1, 999);              -- ✅ Now valid
```

**Cascading Actions:**
```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
        ON DELETE CASCADE      -- Delete orders when customer deleted
        ON UPDATE CASCADE      -- Update orders when customer ID changes
);

-- Delete customer:
DELETE FROM customers WHERE id = 999;
-- Automatically deletes all orders for customer 999 ✅
```

**Foreign Key Options:**
- `ON DELETE CASCADE`: Delete dependent rows
- `ON DELETE SET NULL`: Set FK to NULL
- `ON DELETE SET DEFAULT`: Set FK to default value
- `ON DELETE RESTRICT`: Prevent deletion (default)
- `ON DELETE NO ACTION`: Same as RESTRICT

---

### CHECK Constraint Internals

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10,2),
    stock INT,
    
    -- Simple CHECK
    CHECK (price > 0),
    CHECK (stock >= 0),
    
    -- Named CHECK (better error messages)
    CONSTRAINT valid_price CHECK (price BETWEEN 0.01 AND 999999.99),
    
    -- Multi-column CHECK
    CONSTRAINT discount_valid CHECK (
        discount_percent IS NULL OR 
        (discount_percent BETWEEN 0 AND 100 AND price > 0)
    )
);

-- Database evaluates CHECK on every INSERT/UPDATE:
INSERT INTO products VALUES (1, 'Widget', -10.00, 100);
-- ❌ ERROR: new row violates check constraint "valid_price"

INSERT INTO products VALUES (2, 'Gadget', 50.00, -5);
-- ❌ ERROR: new row violates check constraint "stock >= 0"
```

---

### UNIQUE Constraint Internals

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(20) UNIQUE
);

-- Internally creates UNIQUE INDEX:
-- CREATE UNIQUE INDEX users_email_key ON users (email);

-- Handles NULL specially:
INSERT INTO users VALUES (1, 'a@test.com', '111-1111');  -- ✅
INSERT INTO users VALUES (2, 'b@test.com', '222-2222');  -- ✅
INSERT INTO users VALUES (3, NULL, NULL);                -- ✅ NULLs allowed
INSERT INTO users VALUES (4, NULL, NULL);                -- ✅ Multiple NULLs ok
INSERT INTO users VALUES (5, 'a@test.com', '333-3333');  -- ❌ Duplicate email
```

**Composite UNIQUE:**
```sql
CREATE TABLE team_members (
    team_id BIGINT,
    user_id BIGINT,
    role VARCHAR(50),
    UNIQUE (team_id, user_id)  -- User can be in each team once
);

-- Valid: Different combinations
INSERT INTO team_members VALUES (1, 100, 'member');
INSERT INTO team_members VALUES (1, 200, 'admin');   -- ✅ Different user
INSERT INTO team_members VALUES (2, 100, 'owner');   -- ✅ Different team

-- Invalid: Duplicate combination
INSERT INTO team_members VALUES (1, 100, 'owner');   -- ❌ Duplicate
```

---

## ✅ 4. Best Practices

### Always Use PRIMARY KEY

```sql
-- ❌ Bad: No primary key
CREATE TABLE logs (
    timestamp TIMESTAMP,
    message TEXT
);
-- Problems: Can't reference rows, no guaranteed uniqueness

-- ✅ Good: Always have a primary key
CREATE TABLE logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    timestamp TIMESTAMP NOT NULL,
    message TEXT
);
```

---

### Use Foreign Keys for Referential Integrity

```sql
-- ✅ Enforces data integrity
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);

-- Benefits:
-- - Cannot create orders for non-existent customers
-- - Cannot accidentally delete customers with active orders
-- - Database enforces referential integrity
```

**Choose the right cascading behavior:**
```sql
-- User account deletion should remove all user data
CREATE TABLE user_sessions (
    user_id BIGINT,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Order deletion should NOT delete customer
CREATE TABLE orders (
    customer_id BIGINT,
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT
);
```

---

### Name Your Constraints

```sql
-- ❌ Bad: Auto-generated names
CREATE TABLE products (
    price DECIMAL(10,2) CHECK (price > 0)
);
-- Error message: "new row violates check constraint products_check"

-- ✅ Good: Descriptive names
CREATE TABLE products (
    price DECIMAL(10,2),
    CONSTRAINT price_must_be_positive CHECK (price > 0),
    CONSTRAINT price_within_reasonable_range CHECK (price < 1000000)
);
-- Error message: "violates check constraint price_must_be_positive"
```

---

### Use CHECK for Business Rules

```sql
CREATE TABLE employees (
    id BIGINT PRIMARY KEY,
    birth_date DATE NOT NULL,
    hire_date DATE NOT NULL,
    termination_date DATE,
    salary DECIMAL(12,2) NOT NULL,
    
    -- Business rules enforced at DB level
    CONSTRAINT valid_age CHECK (
        birth_date <= CURRENT_DATE - INTERVAL '18 years'
    ),
    CONSTRAINT hire_after_birth CHECK (hire_date > birth_date),
    CONSTRAINT termination_after_hire CHECK (
        termination_date IS NULL OR termination_date >= hire_date
    ),
    CONSTRAINT reasonable_salary CHECK (
        salary BETWEEN 0.01 AND 10000000.00
    )
);
```

---

### NOT NULL by Default (When Appropriate)

```sql
-- ✅ Good: Be explicit about NULLability
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,           -- Required
    first_name VARCHAR(50) NOT NULL,       -- Required
    last_name VARCHAR(50) NOT NULL,        -- Required
    middle_name VARCHAR(50),               -- Optional (NULLable)
    phone VARCHAR(20),                     -- Optional
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Prevents NULL-related bugs:
SELECT * FROM users WHERE email = ?;  -- Won't miss rows with NULL email
```

---

### Partial UNIQUE Constraints

```sql
-- Multiple rows can be 'inactive', but only one can be 'primary'
CREATE TABLE user_emails (
    user_id BIGINT,
    email VARCHAR(255) NOT NULL,
    is_primary BOOLEAN DEFAULT FALSE,
    
    -- Only one primary email per user
    UNIQUE (user_id, is_primary) WHERE is_primary = TRUE
);

-- Or in PostgreSQL:
CREATE UNIQUE INDEX idx_one_primary_email 
ON user_emails (user_id) 
WHERE is_primary = TRUE;
```

---

## ❌ 5. Common Mistakes

### Mistake 1: No Foreign Keys

```sql
-- ❌ Bad: No referential integrity
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT  -- No FK constraint
);

-- Problems:
INSERT INTO orders VALUES (1, 999);  -- Accepts non-existent customer ❌
DELETE FROM customers WHERE id = 1;  -- Orphans orders ❌

-- ✅ Good: Use foreign keys
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT
);
```

---

### Mistake 2: Over-Relying on Application Logic

```sql
-- ❌ Bad: Validation only in application
CREATE TABLE products (
    price DECIMAL(10,2)  -- No constraints
);

-- Application code:
if (price <= 0) {
    throw new Error("Invalid price");
}
// Problem: Database accepts invalid data from other sources
// (admin tools, scripts, third-party integrations)

-- ✅ Good: Enforce in database
CREATE TABLE products (
    price DECIMAL(10,2) CHECK (price > 0)
);
-- Now invalid data rejected regardless of source ✅
```

---

### Mistake 3: Using CHECK Instead of Foreign Key

```sql
-- ❌ Bad: CHECK constraint for referential integrity
CREATE TABLE orders (
    customer_id BIGINT CHECK (
        customer_id IN (SELECT id FROM customers)
    )  -- Slow subquery on every insert!
);

-- ✅ Good: Use foreign key
CREATE TABLE orders (
    customer_id BIGINT,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

---

### Mistake 4: Ignoring NULL Behavior

```sql
-- ❌ Bad: Misunderstanding NULLs
CREATE TABLE products (
    name VARCHAR(100),
    category VARCHAR(50)
);

-- This finds nothing!
SELECT * FROM products WHERE category != 'electronics';
-- Doesn't match rows where category IS NULL ❌

-- ✅ Good: Handle NULLs explicitly
SELECT * FROM products 
WHERE category != 'electronics' OR category IS NULL;

-- Better: Use NOT NULL where appropriate
CREATE TABLE products (
    name VARCHAR(100) NOT NULL,
    category VARCHAR(50) NOT NULL DEFAULT 'uncategorized'
);
```

---

### Mistake 5: Complex CHECK Constraints

```sql
-- ❌ Bad: Complex logic in CHECK
CREATE TABLE orders (
    status VARCHAR(20),
    paid_amount DECIMAL(10,2),
    total_amount DECIMAL(10,2),
    CHECK (
        (status = 'pending' AND paid_amount = 0) OR
        (status = 'partial' AND paid_amount > 0 AND paid_amount < total_amount) OR
        (status = 'paid' AND paid_amount = total_amount)
    )  -- Hard to maintain, poor performance
);

-- ✅ Good: Simpler constraints + application logic
CREATE TABLE orders (
    status VARCHAR(20) CHECK (status IN ('pending', 'partial', 'paid')),
    paid_amount DECIMAL(10,2) CHECK (paid_amount >= 0),
    total_amount DECIMAL(10,2) CHECK (total_amount > 0),
    CHECK (paid_amount <= total_amount)
);
-- Complex state machine logic in application ✅
```

---

## 🔐 6. Security Considerations

### Prevent SQL Injection via Constraints

```sql
-- Constraints provide additional validation layer
CREATE TABLE users (
    username VARCHAR(50) NOT NULL CHECK (
        username ~ '^[a-zA-Z0-9_-]{3,50}$'  -- Regex validation
    ),
    email VARCHAR(255) NOT NULL CHECK (
        email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'
    )
);

-- Even if application validation is bypassed, database rejects:
INSERT INTO users VALUES ('admin'' OR ''1''=''1', 'x@x.com');
-- ❌ ERROR: violates check constraint (rejects SQL injection attempt)
```

---

### Row-Level Security with CHECK

```sql
-- Multi-tenant security
CREATE TABLE documents (
    id BIGINT PRIMARY KEY,
    tenant_id INT NOT NULL,
    content TEXT,
    
    -- Ensure tenant_id matches session
    CHECK (tenant_id = current_setting('app.tenant_id')::INT)
);

-- Before queries, set tenant:
SET app.tenant_id = 1;

-- Cannot insert data for other tenants:
INSERT INTO documents VALUES (1, 2, 'Secret data');
-- ❌ ERROR: violates check constraint
```

---

### Audit Trail with NOT NULL

```sql
-- Enforce audit columns
CREATE TABLE sensitive_data (
    id BIGINT PRIMARY KEY,
    data TEXT,
    
    -- Audit fields (required)
    created_by BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_by BIGINT NOT NULL,
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id)
);

-- Cannot create records without accountability ✅
```

---

## 🚀 7. Performance Optimization Techniques

### Deferred Constraint Checking

```sql
-- Problem: Large batch inserts with FK constraints are slow
CREATE TABLE orders (
    customer_id BIGINT,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- Solution: Defer constraint checking
BEGIN;
SET CONSTRAINTS ALL DEFERRED;  -- Check at commit, not per row

INSERT INTO orders VALUES (1, 1000);
INSERT INTO orders VALUES (2, 1001);
-- ... 10,000 more inserts

COMMIT;  -- All constraints checked once at the end ✅
```

---

### Partial Indexes for Constraints

```sql
-- Only index active records for UNIQUE constraint
CREATE UNIQUE INDEX idx_active_usernames 
ON users (username) 
WHERE deleted_at IS NULL;

-- Benefits:
-- - Smaller index (only active users)
-- - Faster constraint checking
-- - Can reuse usernames after soft delete
```

---

### Avoid Expensive CHECK Constraints

```sql
-- ❌ Bad: Subquery in CHECK (slow!)
CREATE TABLE orders (
    customer_id BIGINT CHECK (
        customer_id IN (SELECT id FROM customers)
    )
);

-- ✅ Good: Use foreign key
CREATE TABLE orders (
    customer_id BIGINT,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- ✅ Good: Simple CHECK expressions
CREATE TABLE orders (
    quantity INT CHECK (quantity > 0 AND quantity < 10000)
);
```

---

### Materialized Constraints for Performance

```sql
-- Problem: CHECK constraint with expensive calculation
CREATE TABLE accounts (
    balance DECIMAL(10,2),
    total_orders DECIMAL(10,2),
    CHECK (balance >= -1000 OR total_orders > 5000)  -- Complex logic
);

-- Solution: Denormalize to simpler constraint
CREATE TABLE accounts (
    balance DECIMAL(10,2),
    total_orders DECIMAL(10,2),
    credit_level VARCHAR(20) AS (
        CASE 
            WHEN balance >= 0 THEN 'good'
            WHEN total_orders > 5000 THEN 'premium'
            ELSE 'limited'
        END
    ) STORED,
    CHECK (credit_level IN ('good', 'premium', 'limited'))
);
```

---

## 🧪 8. Examples

### E-Commerce Order System

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL CHECK (email LIKE '%@%.%'),
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock_quantity INT NOT NULL CHECK (stock_quantity >= 0),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(12,2) NOT NULL CHECK (total_amount >= 0),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    shipped_at TIMESTAMP,
    
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT,
    CHECK (shipped_at IS NULL OR shipped_at >= created_at)
);

CREATE TABLE order_items (
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price >= 0),
    
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT
);
```

---

### SaaS Multi-Tenant System

```sql
CREATE TABLE tenants (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) UNIQUE NOT NULL,
    plan ENUM('free', 'pro', 'enterprise') DEFAULT 'free',
    max_users INT NOT NULL CHECK (max_users > 0),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE tenant_users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    email VARCHAR(255) NOT NULL,
    role ENUM('owner', 'admin', 'member', 'viewer') DEFAULT 'member',
    is_active BOOLEAN DEFAULT TRUE,
    
    FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE,
    UNIQUE (tenant_id, email),  -- Email unique per tenant
    
    -- At least one owner per tenant (enforced at application level)
    CHECK (is_active = TRUE OR role != 'owner')  -- Can't deactivate owner
);

-- Ensure tenant doesn't exceed user limit
CREATE TRIGGER check_user_limit BEFORE INSERT ON tenant_users
FOR EACH ROW
BEGIN
    DECLARE user_count INT;
    DECLARE user_limit INT;
    
    SELECT COUNT(*), t.max_users INTO user_count, user_limit
    FROM tenant_users tu
    JOIN tenants t ON t.id = tu.tenant_id
    WHERE tu.tenant_id = NEW.tenant_id AND tu.is_active = TRUE;
    
    IF user_count >= user_limit THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Tenant user limit exceeded';
    END IF;
END;
```

---

### Financial Transaction System

```sql
CREATE TABLE accounts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    account_number VARCHAR(20) UNIQUE NOT NULL,
    balance DECIMAL(19,4) NOT NULL DEFAULT 0 CHECK (balance >= -10000),  -- Allow $10K overdraft
    currency CHAR(3) NOT NULL CHECK (currency ~ '^[A-Z]{3}$'),
    status ENUM('active', 'frozen', 'closed') DEFAULT 'active',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE transactions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    from_account_id BIGINT,
    to_account_id BIGINT,
    amount DECIMAL(19,4) NOT NULL CHECK (amount > 0),
    currency CHAR(3) NOT NULL,
    type ENUM('transfer', 'deposit', 'withdrawal', 'fee', 'refund'),
    status ENUM('pending', 'completed', 'failed', 'reversed') DEFAULT 'pending',
    reference_id VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP,
    
    FOREIGN KEY (from_account_id) REFERENCES accounts(id),
    FOREIGN KEY (to_account_id) REFERENCES accounts(id),
    
    -- Business rules
    CHECK (from_account_id != to_account_id),  -- Can't transfer to self
    CHECK (from_account_id IS NOT NULL OR to_account_id IS NOT NULL),  -- At least one account
    CHECK (completed_at IS NULL OR completed_at >= created_at),
    
    -- Currency consistency
    CHECK (
        (from_account_id IS NULL) OR 
        (currency = (SELECT currency FROM accounts WHERE id = from_account_id))
    )
);
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Airbnb - Booking Conflicts

**Problem:** Prevent double-booking of properties.

**Solution:** EXCLUSION constraint with date ranges

```sql
-- PostgreSQL EXCLUSION constraint
CREATE TABLE bookings (
    id BIGINT PRIMARY KEY,
    property_id BIGINT NOT NULL,
    check_in DATE NOT NULL,
    check_out DATE NOT NULL,
    
    CHECK (check_out > check_in),
    
    -- Prevent overlapping bookings
    EXCLUDE USING GIST (
        property_id WITH =,
        daterange(check_in, check_out) WITH &&
    )
);

-- Attempts to double-book:
INSERT INTO bookings VALUES (1, 100, '2024-03-01', '2024-03-05');  -- ✅
INSERT INTO bookings VALUES (2, 100, '2024-03-03', '2024-03-07');  -- ❌ Overlaps!
INSERT INTO bookings VALUES (3, 100, '2024-03-06', '2024-03-10');  -- ✅ No overlap
```

---

### Case Study 2: Stripe - Idempotent Payments

**Problem:** Prevent duplicate charges from retries.

**Solution:** UNIQUE constraint on idempotency key

```sql
CREATE TABLE payments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    idempotency_key UUID UNIQUE NOT NULL,  -- Client-generated
    customer_id BIGINT NOT NULL,
    amount DECIMAL(19,4) NOT NULL CHECK (amount > 0),
    currency CHAR(3) NOT NULL,
    status ENUM('pending', 'succeeded', 'failed'),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- Client retries with same key:
INSERT INTO payments (idempotency_key, customer_id, amount, currency)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 123, 100.00, 'USD');  -- ✅

-- Retry (network issue):
INSERT INTO payments (idempotency_key, customer_id, amount, currency)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 123, 100.00, 'USD');  
-- ❌ UNIQUE violation - no duplicate charge!

-- Application catches error and returns original payment ✅
```

---

### Case Study 3: GitHub - Username Changes

**Problem:** Allow username changes but prevent username recycling too quickly.

**Solution:** Soft delete with time-based UNIQUE constraint

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    deleted_at TIMESTAMP,
    
    -- Username unique among active users
    UNIQUE (username) WHERE deleted_at IS NULL
);

CREATE TABLE username_history (
    username VARCHAR(50) NOT NULL,
    user_id BIGINT NOT NULL,
    released_at TIMESTAMP NOT NULL DEFAULT NOW(),
    available_at TIMESTAMP NOT NULL DEFAULT NOW() + INTERVAL '90 days',
    
    -- Can't reuse username for 90 days
    EXCLUDE USING GIST (
        username WITH =,
        tsrange(released_at, available_at) WITH &&
    )
);

-- User changes username:
UPDATE users SET username = 'newname' WHERE id = 1;
INSERT INTO username_history (username, user_id) VALUES ('oldname', 1);

-- Someone tries to take 'oldname' immediately:
-- Application checks username_history.available_at ❌
```

---

### Case Study 4: Uber - Driver Availability

**Problem:** Drivers can only be on one ride at a time.

**Solution:** CHECK constraint + application logic

```sql
CREATE TABLE rides (
    id BIGINT PRIMARY KEY,
    driver_id BIGINT NOT NULL,
    passenger_id BIGINT NOT NULL,
    status ENUM('requested', 'accepted', 'in_progress', 'completed', 'cancelled'),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    FOREIGN KEY (driver_id) REFERENCES drivers(id),
    FOREIGN KEY (passenger_id) REFERENCES passengers(id),
    
    CHECK (completed_at IS NULL OR completed_at >= started_at),
    CHECK (passenger_id != driver_id),  -- Can't ride own car
    
    -- Only one active ride per driver
    UNIQUE (driver_id) WHERE status IN ('accepted', 'in_progress')
);
```

---

### Case Study 5: Slack - Message Threading

**Problem:** Messages must be in same channel as parent thread.

**Solution:** CHECK constraint with subquery (or FK to composite key)

```sql
CREATE TABLE messages (
    id BIGINT PRIMARY KEY,
    channel_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    parent_id BIGINT,  -- NULL = top-level message
    content TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY (channel_id) REFERENCES channels(id),
    FOREIGN KEY (user_id) REFERENCES users(id),
    
    -- Thread reply must be in same channel as parent
    FOREIGN KEY (channel_id, parent_id) 
        REFERENCES messages(channel_id, id) ON DELETE CASCADE
);

-- Better approach: Composite FK
ALTER TABLE messages 
ADD CONSTRAINT fk_parent_same_channel 
FOREIGN KEY (channel_id, parent_id) 
REFERENCES messages(channel_id, id);
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between PRIMARY KEY and UNIQUE?

**Answer:**

| Feature | PRIMARY KEY | UNIQUE |
|---------|------------|--------|
| **NULL allowed** | No (implicitly NOT NULL) | Yes (can have NULLs) |
| **Per table** | Only one | Multiple allowed |
| **Index created** | Yes (clustered in some DBs) | Yes (non-clustered) |
| **Foreign key target** | Yes (default) | Yes (but rare) |

**Example:**
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,        -- NOT NULL, unique, one per table
    email VARCHAR(255) UNIQUE,    -- Can be NULL, multiple unique columns ok
    username VARCHAR(50) UNIQUE
);

INSERT INTO users VALUES (1, 'a@test.com', 'alice');  -- ✅
INSERT INTO users VALUES (2, NULL, 'bob');            -- ✅ NULL email ok
INSERT INTO users VALUES (3, NULL, 'charlie');        -- ✅ Multiple NULLs ok
INSERT INTO users VALUES (NULL, 'd@test.com', 'dave');-- ❌ NULL id not allowed
```

---

### Q2: When should you use ON DELETE CASCADE vs RESTRICT?

**Answer:**

**Use CASCADE when:**
- Deleting parent should automatically delete children
- Children have no meaning without parent
- Examples: User → Sessions, Order → Order Items

**Use RESTRICT when:**
- Prevent accidental data loss
- Children should be handled explicitly
- Examples: Customer → Orders, Category → Products

**Example:**
```sql
-- CASCADE: Delete user deletes all sessions
CREATE TABLE sessions (
    user_id BIGINT,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- RESTRICT: Can't delete customer with orders
CREATE TABLE orders (
    customer_id BIGINT,
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT
);

-- SET NULL: Delete category, products become uncategorized
CREATE TABLE products (
    category_id BIGINT,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL
);
```

---

### Q3: Why might you disable foreign key constraints temporarily?

**Answer:**

**Valid reasons:**
1. **Bulk data loading**: FK checks slow down large imports
2. **Database migration**: Circular dependencies during schema changes
3. **Data restoration**: Restore order might violate FKs temporarily

**Example:**
```sql
-- Disable FK checks (MySQL)
SET FOREIGN_KEY_CHECKS = 0;

-- Bulk load data
LOAD DATA INFILE '/data/customers.csv' INTO TABLE customers;
LOAD DATA INFILE '/data/orders.csv' INTO TABLE orders;

-- Re-enable and validate
SET FOREIGN_KEY_CHECKS = 1;

-- Verify integrity
SELECT o.id FROM orders o 
LEFT JOIN customers c ON o.customer_id = c.id 
WHERE c.id IS NULL;  -- Find orphaned orders
```

**Warning:** Always re-enable and validate! Leaving FK checks disabled is dangerous.

---

### Q4: How do you handle CHECK constraints with multiple columns?

**Answer:**

```sql
CREATE TABLE promotions (
    id BIGINT PRIMARY KEY,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    discount_percent DECIMAL(5,2),
    discount_amount DECIMAL(10,2),
    
    -- Multi-column CHECKs
    CHECK (end_date >= start_date),
    
    -- Exactly one discount type
    CHECK (
        (discount_percent IS NOT NULL AND discount_amount IS NULL) OR
        (discount_percent IS NULL AND discount_amount IS NOT NULL)
    ),
    
    -- Discount ranges
    CHECK (discount_percent IS NULL OR discount_percent BETWEEN 0 AND 100),
    CHECK (discount_amount IS NULL OR discount_amount > 0)
);
```

---

### Q5: What happens to indexes when you create constraints?

**Answer:**

**Automatically created indexes:**
- **PRIMARY KEY**: Creates unique index (often clustered)
- **UNIQUE**: Creates unique index
- **FOREIGN KEY**: May create index (database-dependent)

**Example:**
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,           -- Creates: idx_users_pkey
    email VARCHAR(255) UNIQUE        -- Creates: idx_users_email_key
);

-- Check indexes:
SHOW INDEXES FROM users;

-- Output:
-- Table  Column  Key_name        Index_type
-- users  id      PRIMARY         BTREE
-- users  email   users_email_key BTREE
```

**Foreign keys:** Some databases (PostgreSQL) don't auto-create FK indexes:
```sql
CREATE TABLE orders (
    customer_id BIGINT,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- Manually create index for FK (important!)
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

---

### Q6: How do you enforce business rules that span multiple tables?

**Answer:**

**Options:**
1. **Triggers**: Most flexible, but complex
2. **CHECK with subquery**: Limited support, performance issues
3. **Application logic**: Less reliable, can be bypassed
4. **Materialized views**: For read queries
5. **Stored procedures**: Encapsulate logic

**Example with trigger:**
```sql
-- Business rule: Account balance must equal sum of transactions
CREATE TRIGGER check_balance_consistency 
BEFORE UPDATE ON accounts
FOR EACH ROW
BEGIN
    DECLARE calculated_balance DECIMAL(19,4);
    
    SELECT SUM(amount) INTO calculated_balance
    FROM transactions
    WHERE account_id = NEW.id AND status = 'completed';
    
    IF ABS(NEW.balance - calculated_balance) > 0.01 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Balance does not match transactions';
    END IF;
END;
```

**Best practice:** Use constraints where possible, triggers when necessary, and application logic as last resort.

---

## 🧩 11. Design Patterns

### Pattern 1: Soft Delete with Constraints

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    deleted_at TIMESTAMP,
    
    -- Email unique only among active users
    UNIQUE (email) WHERE deleted_at IS NULL
);

-- Can reuse email after soft delete:
INSERT INTO users VALUES (1, 'user@test.com', NULL);           -- ✅
UPDATE users SET deleted_at = NOW() WHERE id = 1;             -- Soft delete
INSERT INTO users VALUES (2, 'user@test.com', NULL);           -- ✅ Reuse email
```

---

### Pattern 2: Immutable Records with Constraints

```sql
CREATE TABLE audit_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    table_name VARCHAR(50) NOT NULL,
    record_id BIGINT NOT NULL,
    action ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by BIGINT NOT NULL,
    
    -- Prevent updates/deletes (immutable)
    CONSTRAINT immutable CHECK (FALSE)  -- Blocks all updates
);

-- Or use trigger:
CREATE TRIGGER prevent_modifications
BEFORE UPDATE OR DELETE ON audit_log
FOR EACH ROW
BEGIN
    SIGNAL SQLSTATE '45000' 
    SET MESSAGE_TEXT = 'Audit log records are immutable';
END;
```

---

### Pattern 3: State Machine with Constraints

```sql
CREATE TYPE order_status AS ENUM (
    'draft', 'pending', 'paid', 'processing', 
    'shipped', 'delivered', 'cancelled', 'refunded'
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    status order_status DEFAULT 'draft',
    previous_status order_status,
    
    -- Enforce valid transitions
    CHECK (
        (status = 'pending' AND (previous_status IS NULL OR previous_status = 'draft')) OR
        (status = 'paid' AND previous_status = 'pending') OR
        (status = 'processing' AND previous_status = 'paid') OR
        (status = 'shipped' AND previous_status = 'processing') OR
        (status = 'delivered' AND previous_status = 'shipped') OR
        (status = 'cancelled' AND previous_status IN ('draft', 'pending')) OR
        (status = 'refunded' AND previous_status = 'delivered')
    )
);
```

---

### Pattern 4: Hierarchical Data with Self-Referencing FK

```sql
CREATE TABLE categories (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id BIGINT,
    level INT NOT NULL CHECK (level >= 0 AND level <= 5),  -- Max 5 levels
    
    -- Self-referencing FK
    FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE CASCADE,
    
    -- Root categories have NULL parent
    CHECK ((parent_id IS NULL AND level = 0) OR (parent_id IS NOT NULL AND level > 0))
);

-- Prevent circular references (application/trigger logic):
CREATE TRIGGER prevent_cycle
BEFORE INSERT OR UPDATE ON categories
FOR EACH ROW
BEGIN
    IF EXISTS (
        WITH RECURSIVE ancestors AS (
            SELECT id, parent_id FROM categories WHERE id = NEW.parent_id
            UNION ALL
            SELECT c.id, c.parent_id FROM categories c
            JOIN ancestors a ON c.id = a.parent_id
        )
        SELECT 1 FROM ancestors WHERE id = NEW.id
    ) THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Circular reference detected';
    END IF;
END;
```

---

### Pattern 5: Time-Based Constraints

```sql
CREATE TABLE subscriptions (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    plan ENUM('monthly', 'annual') NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Time-based business rules
    CHECK (end_date > start_date),
    CHECK (
        (plan = 'monthly' AND end_date <= start_date + INTERVAL '31 days') OR
        (plan = 'annual' AND end_date <= start_date + INTERVAL '366 days')
    ),
    CHECK (
        (is_active = TRUE AND end_date >= CURRENT_DATE) OR
        (is_active = FALSE)
    ),
    
    -- No overlapping subscriptions
    EXCLUDE USING GIST (
        user_id WITH =,
        daterange(start_date, end_date) WITH &&
    )
);
```

---

## 📚 Summary

### Key Takeaways

1. **Always use PRIMARY KEY** - every table needs one
2. **Enforce referential integrity with FOREIGN KEY** - prevent orphaned data
3. **Use NOT NULL by default** - be explicit about optional fields
4. **CHECK constraints enforce business rules** - validate at database level
5. **Name your constraints** - better error messages for debugging
6. **Choose appropriate CASCADE behavior** - RESTRICT by default, CASCADE when appropriate
7. **Use UNIQUE for alternate keys** - emails, usernames, etc.
8. **DEFAULT values improve data quality** - timestamps, statuses, flags
9. **Constraints are last line of defense** - application validation can be bypassed
10. **Consider performance impact** - deferred checking for bulk operations

### Constraint Decision Tree

```
Need to ensure...

└─ Uniqueness?
   ├─ Primary identifier? → PRIMARY KEY
   ├─ Alternate key? → UNIQUE
   └─ Conditional uniqueness? → Partial UNIQUE index

└─ Relationships?
   ├─ Parent-child? → FOREIGN KEY
   ├─ Delete parent? 
   │  ├─ Delete children? → ON DELETE CASCADE
   │  ├─ Prevent deletion? → ON DELETE RESTRICT
   │  └─ Nullify children? → ON DELETE SET NULL
   └─ Self-referencing? → FK to same table

└─ Value validity?
   ├─ Required field? → NOT NULL
   ├─ Fixed set? → ENUM or CHECK IN (...)
   ├─ Range? → CHECK (column BETWEEN x AND y)
   └─ Complex rule? → CHECK with expression

└─ Default value needed?
   └─ → DEFAULT value or expression

└─ No overlapping ranges?
   └─ → EXCLUDE (PostgreSQL)

└─ Multiple column rules?
   └─ → Multi-column CHECK
```

### Production Checklist

- [ ] All tables have PRIMARY KEY
- [ ] All relationships have FOREIGN KEY constraints
- [ ] Required fields marked NOT NULL
- [ ] Appropriate CASCADE/RESTRICT behavior configured
- [ ] Business rules validated with CHECK constraints
- [ ] Constraints have descriptive names
- [ ] UNIQUE constraints on alternate keys (email, username)
- [ ] DEFAULT values for timestamps, status fields
- [ ] Foreign key indexes created manually (if needed)
- [ ] Consider performance impact of complex CHECKs

---

**Next Steps:**
- Study [06_Keys_And_Indexes.md](06_Keys_And_Indexes.md) for indexing strategies
- Review [../../07_Transactions_And_Concurrency/03_Locking_Mechanisms.md](../../07_Transactions_And_Concurrency/03_Locking_Mechanisms.md) for lock behavior
- Explore [../../08_Isolation_Levels_And_Locking/](../../08_Isolation_Levels_And_Locking/) for transaction isolation

---

*Last Updated: February 18, 2026*
