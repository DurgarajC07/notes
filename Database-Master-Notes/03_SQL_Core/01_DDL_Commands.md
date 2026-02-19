# 🏗️ DDL Commands - Data Definition Language

> DDL commands define and modify database structure. Master CREATE, ALTER, DROP, and TRUNCATE to manage schemas effectively.

---

## 📖 1. Concept Explanation

### What is DDL?

**Data Definition Language (DDL)** commands define the structure and schema of databases, including tables, indexes, constraints, and relationships.

**Core DDL Commands:**
```
DDL COMMANDS
│
├── CREATE - Create database objects
│   ├── CREATE DATABASE
│   ├── CREATE TABLE
│   ├── CREATE INDEX
│   └── CREATE VIEW
│
├── ALTER - Modify existing objects
│   ├── ALTER TABLE ADD COLUMN
│   ├── ALTER TABLE DROP COLUMN
│   ├── ALTER TABLE MODIFY COLUMN
│   └── ALTER TABLE ADD CONSTRAINT
│
├── DROP - Delete objects permanently
│   ├── DROP TABLE
│   ├── DROP DATABASE
│   └── DROP INDEX
│
└── TRUNCATE - Remove all rows (keep structure)
    └── TRUNCATE TABLE
```

**Key Characteristics:**
- **Auto-commit**: DDL commands commit immediately (no rollback in most databases)
- **Schema changes**: Modify database structure
- **Implicit locks**: May lock tables during execution
- **Metadata changes**: Update system catalogs

---

## 🧠 2. Why It Matters in Real Systems

### Without Proper DDL Knowledge

```sql
-- ❌ Bad: Creating table without constraints
CREATE TABLE users (
    id INT,
    email VARCHAR(255),
    created_at VARCHAR(50)
);

-- Problems:
-- - No primary key (can't reference)
-- - No constraints (accepts duplicate emails)
-- - Wrong data type for date (VARCHAR instead of TIMESTAMP)
-- - No indexes (slow queries)
```

### With Proper DDL

```sql
-- ✅ Good: Well-designed table
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash CHAR(60) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
    
    INDEX idx_email (email),
    INDEX idx_active (is_active) WHERE is_active = TRUE,
    CHECK (email LIKE '%@%')
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## ⚙️ 3. Internal Working

### CREATE TABLE Internals

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    total DECIMAL(10,2)
);

-- What happens internally:
-- 1. Database creates entry in system catalog (information_schema)
-- 2. Allocates storage space (initial extent)
-- 3. Creates primary key index (B-tree)
-- 4. Sets up transaction log entries
-- 5. Updates metadata (pg_class, mysql.tables, etc.)
```

### ALTER TABLE Internals

```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Two approaches:
-- 1. ALGORITHM=COPY (old, slow):
--    - Creates new table with new structure
--    - Copies all data row by row
--    - Swaps tables
--    - Drops old table
--    - 10M rows = 5-10 minutes downtime ❌

-- 2. ALGORITHM=INPLACE (modern, fast):
--    - Modifies table metadata only
--    - No data copy
--    - MySQL 5.6+, PostgreSQL native
--    - 10M rows = < 1 second ✅
```

**PostgreSQL ALTER TABLE:**
```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- PostgreSQL: Near-instant (adds to system catalog only)
-- New rows write to new column
-- Old rows return NULL (no rewrite needed)
```

**MySQL ALTER TABLE:**
```sql
-- Force specific algorithm
ALTER TABLE users 
ADD COLUMN phone VARCHAR(20),
ALGORITHM=INPLACE, LOCK=NONE;

-- ALGORITHM options:
-- - INPLACE: Modify without copying
-- - COPY: Create new table (slow)
-- - DEFAULT: Let MySQL choose

-- LOCK options:
-- - NONE: Allow reads and writes
-- - SHARED: Allow reads, block writes
-- - EXCLUSIVE: Block all access
```

---

## ✅ 4. Best Practices

### CREATE TABLE Best Practices

```sql
-- ✅ Comprehensive table creation
CREATE TABLE IF NOT EXISTS products (
    -- Primary key
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    
    -- Business columns with constraints
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    
    -- Properly typed columns
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    stock_quantity INT NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Foreign keys
    category_id INT NOT NULL,
    brand_id INT,
    
    -- Audit columns
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
    created_by BIGINT,
    
    -- Indexes for common queries
    INDEX idx_sku (sku),
    INDEX idx_category (category_id, is_active),
    INDEX idx_brand (brand_id),
    INDEX idx_created (created_at),
    
    -- Foreign key constraints
    CONSTRAINT fk_category FOREIGN KEY (category_id) 
        REFERENCES categories(id) ON DELETE RESTRICT,
    CONSTRAINT fk_brand FOREIGN KEY (brand_id) 
        REFERENCES brands(id) ON DELETE SET NULL,
    CONSTRAINT fk_created_by FOREIGN KEY (created_by) 
        REFERENCES users(id) ON DELETE SET NULL
        
) ENGINE=InnoDB 
  DEFAULT CHARSET=utf8mb4 
  COLLATE=utf8mb4_unicode_ci
  COMMENT='Product catalog';
```

### ALTER TABLE Best Practices

```sql
-- ✅ Add column with default (online operation in modern databases)
ALTER TABLE users 
ADD COLUMN phone VARCHAR(20) DEFAULT NULL;

-- ✅ Add NOT NULL column safely (PostgreSQL)
ALTER TABLE users ADD COLUMN status VARCHAR(20);
UPDATE users SET status = 'active' WHERE status IS NULL;
ALTER TABLE users ALTER COLUMN status SET NOT NULL;

-- ✅ Add index concurrently (PostgreSQL - no blocking)
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);

-- ✅ Multiple changes in one statement (fewer locks)
ALTER TABLE products
    ADD COLUMN weight DECIMAL(8,2),
    ADD COLUMN dimensions VARCHAR(50),
    ADD INDEX idx_weight (weight);
```

### DROP Best Practices

```sql
-- ✅ Always use IF EXISTS (idempotent scripts)
DROP TABLE IF EXISTS temp_data;

-- ✅ Check dependencies before dropping
SELECT 
    TABLE_NAME, 
    CONSTRAINT_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_NAME = 'users';

-- ✅ Drop in correct order (foreign keys first)
DROP TABLE IF EXISTS order_items;     -- Child table
DROP TABLE IF EXISTS orders;          -- Parent table

-- ✅ Backup before destructive operations
CREATE TABLE users_backup AS SELECT * FROM users;
DROP TABLE users;
```

---

## ❌ 5. Common Mistakes

### Mistake 1: No IF NOT EXISTS/IF EXISTS

```sql
-- ❌ Bad: Fails if table exists
CREATE TABLE users (...);
-- Error if run twice

-- ✅ Good: Idempotent
CREATE TABLE IF NOT EXISTS users (...);
```

### Mistake 2: Adding NOT NULL Without Default

```sql
-- ❌ Bad: Fails if table has data
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
-- ERROR: Cannot add NOT NULL column to table with existing rows

-- ✅ Good: Add with default, then update
ALTER TABLE users ADD COLUMN phone VARCHAR(20) DEFAULT '';
UPDATE users SET phone = '+1-000-000-0000' WHERE phone = '';
-- Or in two steps:
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
UPDATE users SET phone = COALESCE(phone, '');
ALTER TABLE users MODIFY COLUMN phone VARCHAR(20) NOT NULL;
```

### Mistake 3: Dropping Column With Data

```sql
-- ❌ Bad: Permanent data loss
ALTER TABLE users DROP COLUMN email;
-- No way to recover!

-- ✅ Good: Rename first, verify, then drop
ALTER TABLE users RENAME COLUMN email TO email_deprecated;
-- Monitor for a few days/weeks
-- If safe:
ALTER TABLE users DROP COLUMN email_deprecated;
```

### Mistake 4: Large ALTER Without ALGORITHM

```sql
-- ❌ Bad: Might cause hours of downtime
ALTER TABLE large_table ADD COLUMN status VARCHAR(20);
-- Could lock table for hours on billions of rows

-- ✅ Good: Specify algorithm (MySQL)
ALTER TABLE large_table 
ADD COLUMN status VARCHAR(20),
ALGORITHM=INPLACE, LOCK=NONE;

-- ✅ Good: Use online DDL (PostgreSQL)
ALTER TABLE large_table ADD COLUMN status VARCHAR(20) DEFAULT NULL;
-- Instant in PostgreSQL
```

### Mistake 5: TRUNCATE vs DELETE Confusion

```sql
-- TRUNCATE: Fast, resets auto_increment, no rollback, no WHERE clause
TRUNCATE TABLE logs;  -- Removes all rows instantly

-- DELETE: Slow, keeps auto_increment, rollbackable, supports WHERE
DELETE FROM logs WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);

-- ❌ Bad: Using DELETE to empty table
DELETE FROM large_table;  -- Scans and deletes each row (slow!)

-- ✅ Good: TRUNCATE for full delete
TRUNCATE TABLE large_table;  -- Drops and recreates (instant)
```

---

## 🔐 6. Security Considerations

### Limit DDL Permissions

```sql
-- Create role with limited permissions
CREATE ROLE app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON database.* TO app_user;
-- NO DDL permissions (CREATE, ALTER, DROP)

-- Create DBA role with DDL permissions
CREATE ROLE dba_user;
GRANT ALL PRIVILEGES ON database.* TO dba_user;
```

### Audit DDL Changes

```sql
-- Enable general log (MySQL)
SET GLOBAL general_log = 'ON';

-- PostgreSQL: Enable DDL logging
ALTER SYSTEM SET log_statement = 'ddl';

-- Create audit trigger (SQL Server)
CREATE TRIGGER audit_ddl
ON DATABASE
FOR CREATE_TABLE, ALTER_TABLE, DROP_TABLE
AS
BEGIN
    INSERT INTO ddl_audit_log (event_type, object_name, sql_text, user_name)
    SELECT 
        EVENTDATA().value('(/EVENT_INSTANCE/EventType)[1]', 'VARCHAR(50)'),
        EVENTDATA().value('(/EVENT_INSTANCE/ObjectName)[1]', 'VARCHAR(255)'),
        EVENTDATA().value('(/EVENT_INSTANCE/TSQLCommand)[1]', 'VARCHAR(MAX)'),
        SYSTEM_USER;
END;
```

---

## 🚀 7. Performance Optimization Techniques

### Online DDL (MySQL 8.0+)

```sql
-- Add index without blocking writes
ALTER TABLE users 
ADD INDEX idx_email (email),
ALGORITHM=INPLACE, LOCK=NONE;

-- Add column instantly
ALTER TABLE users 
ADD COLUMN status VARCHAR(20) DEFAULT 'active',
ALGORITHM=INSTANT;  -- MySQL 8.0.12+
```

### Concurrent Index Creation (PostgreSQL)

```sql
-- Normal index creation (locks table)
CREATE INDEX idx_users_email ON users (email);

-- Concurrent index (no locks, longer but allows writes)
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);
```

### Partitioned Tables for Large Data

```sql
-- Create partitioned table
CREATE TABLE logs (
    id BIGINT AUTO_INCREMENT,
    created_at TIMESTAMP NOT NULL,
    message TEXT,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p_2023 VALUES LESS THAN (2024),
    PARTITION p_2024 VALUES LESS THAN (2025),
    PARTITION p_2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Add partition (fast, no data movement)
ALTER TABLE logs 
ADD PARTITION (PARTITION p_2026 VALUES LESS THAN (2027));

-- Drop old partition (instant delete of old data)
ALTER TABLE logs DROP PARTITION p_2023;
```

---

## 🧪 8. Examples

### Complete E-Commerce Schema

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS ecommerce
    DEFAULT CHARACTER SET utf8mb4
    DEFAULT COLLATE utf8mb4_unicode_ci;

USE ecommerce;

-- Customers table
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    phone VARCHAR(20),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW() ON UPDATE NOW(),
    
    INDEX idx_email (email),
    INDEX idx_name (last_name, first_name),
    CHECK (email LIKE '%@%.%')
) ENGINE=InnoDB;

-- Products table
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    cost DECIMAL(10,2),
    stock_quantity INT NOT NULL DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    category_id INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_sku (sku),
    INDEX idx_category_active (category_id, is_active),
    INDEX idx_price (price),
    CHECK (price >= 0),
    CHECK (stock_quantity >= 0)
) ENGINE=InnoDB;

-- Orders table
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id BIGINT NOT NULL,
    status ENUM('pending', 'paid', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    subtotal DECIMAL(12,2) NOT NULL,
    tax DECIMAL(12,2) NOT NULL,
    total DECIMAL(12,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_customer (customer_id, created_at),
    INDEX idx_status (status) WHERE status IN ('pending', 'paid'),
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    CHECK (total = subtotal + tax)
) ENGINE=InnoDB;

-- Order items table
CREATE TABLE order_items (
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(12,2) NOT NULL,
    
    PRIMARY KEY (order_id, product_id),
    INDEX idx_product (product_id),
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id),
    CHECK (quantity > 0),
    CHECK (line_total = quantity * unit_price)
) ENGINE=InnoDB;
```

### Schema Evolution Example

```sql
-- Version 1: Initial schema
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Version 2: Add authentication
ALTER TABLE users 
    ADD COLUMN password_hash CHAR(60) NOT NULL DEFAULT '',
    ADD COLUMN last_login_at TIMESTAMP NULL;

-- Version 3: Add user status
ALTER TABLE users 
    ADD COLUMN status ENUM('active', 'suspended', 'deleted') DEFAULT 'active',
    ADD INDEX idx_status (status);

-- Version 4: Add soft delete
ALTER TABLE users 
    ADD COLUMN deleted_at TIMESTAMP NULL,
    ADD INDEX idx_active (id) WHERE deleted_at IS NULL;

-- Version 5: Add profile fields
ALTER TABLE users 
    ADD COLUMN first_name VARCHAR(50),
    ADD COLUMN last_name VARCHAR(50),
    ADD COLUMN phone VARCHAR(20),
    ADD INDEX idx_name (last_name, first_name);
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: GitHub - Schema Migrations

```sql
-- Migration 001: Create repositories table
CREATE TABLE repositories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    owner_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    is_private BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE KEY uk_owner_name (owner_id, name),
    INDEX idx_owner (owner_id),
    FOREIGN KEY (owner_id) REFERENCES users(id)
);

-- Migration 002: Add stars tracking
ALTER TABLE repositories 
    ADD COLUMN stars_count INT NOT NULL DEFAULT 0,
    ADD INDEX idx_stars (stars_count DESC);

-- Migration 003: Add language detection
ALTER TABLE repositories 
    ADD COLUMN primary_language VARCHAR(50),
    ADD INDEX idx_language (primary_language);
```

### Case Study 2: Stripe - Zero-Downtime Schema Changes

```sql
-- Step 1: Add new column (nullable)
ALTER TABLE subscriptions 
ADD COLUMN billing_cycle_anchor TIMESTAMP NULL;

-- Step 2: Backfill data (in batches, application-level)
-- UPDATE subscriptions SET billing_cycle_anchor = created_at WHERE id BETWEEN ? AND ?;

-- Step 3: Make column required
ALTER TABLE subscriptions 
MODIFY COLUMN billing_cycle_anchor TIMESTAMP NOT NULL;

-- Step 4: Add index
CREATE INDEX CONCURRENTLY idx_billing_anchor 
ON subscriptions (billing_cycle_anchor);
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between DROP, TRUNCATE, and DELETE?

**Answer:**

| Aspect | DROP | TRUNCATE | DELETE |
|--------|------|----------|--------|
| **Removes** | Table structure | All rows | Specific rows |
| **Rollback** | No | No (usually) | Yes |
| **Speed** | Instant | Instant | Slow (row by row) |
| **WHERE clause** | N/A | No | Yes |
| **Triggers** | N/A | No | Yes |
| **Auto-increment** | Removed | Reset | Kept |
| **Use case** | Remove table | Empty table | Remove specific rows |

```sql
-- DROP: Removes table completely
DROP TABLE logs;  -- Table gone, can't query

-- TRUNCATE: Empties table, keeps structure
TRUNCATE TABLE logs;  -- All rows gone, table exists
INSERT INTO logs VALUES (...);  -- Works, id starts at 1

-- DELETE: Removes rows (can be selective)
DELETE FROM logs WHERE created_at < '2024-01-01';  -- Keep recent logs
```

---

### Q2: How do you add a NOT NULL column to a table with existing data?

**Answer:**

```sql
-- Method 1: Add with default value
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL DEFAULT 'N/A';

-- Method 2: Three-step process (safer for large tables)
-- Step 1: Add nullable column
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;

-- Step 2: Populate data (in batches if large table)
UPDATE users SET phone = 'N/A' WHERE phone IS NULL;

-- Step 3: Make NOT NULL
ALTER TABLE users MODIFY COLUMN phone VARCHAR(20) NOT NULL;

-- Method 3: PostgreSQL (best)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users ALTER COLUMN phone SET DEFAULT 'N/A';
UPDATE users SET phone = 'N/A' WHERE phone IS NULL;
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
```

---

### Q3: What is online DDL and why does it matter?

**Answer:**

**Online DDL** allows schema changes without blocking database access.

**Traditional (offline) DDL:**
```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- Table locked for 10 minutes on 100M rows
-- All queries blocked ❌
```

**Online DDL (MySQL 8.0+):**
```sql
ALTER TABLE users 
ADD COLUMN phone VARCHAR(20),
ALGORITHM=INSTANT;  -- or INPLACE
-- Completes in < 1 second
-- No blocking ✅
```

**PostgreSQL** (online by default):
```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- Instant, no locks ✅
```

**Matters because:**
- No downtime for 24/7 services
- No blocked user requests
- Continuous deployment possible

---

## 🧩 11. Design Patterns

### Pattern 1: Idempotent DDL Scripts

```sql
-- Always use IF EXISTS/IF NOT EXISTS
CREATE TABLE IF NOT EXISTS users (...);
DROP TABLE IF EXISTS temp_table;
CREATE INDEX IF NOT EXISTS idx_email ON users(email);

-- For ALTER: Check first
SET @column_exists = (
    SELECT COUNT(*) 
    FROM information_schema.COLUMNS 
    WHERE TABLE_NAME = 'users' AND COLUMN_NAME = 'phone'
);

SET @sql = IF(@column_exists > 0, 
    'SELECT "Column already exists"',
    'ALTER TABLE users ADD COLUMN phone VARCHAR(20)'
);
PREPARE stmt FROM @sql;
EXECUTE stmt;
```

### Pattern 2: Versioned Migrations

```sql
-- Migration table
CREATE TABLE schema_migrations (
    version VARCHAR(20) PRIMARY KEY,
    applied_at TIMESTAMP NOT NULL DEFAULT NOW(),
    description VARCHAR(255)
);

-- Migration script: V2024_02_19__add_user_phone.sql
BEGIN;
    ALTER TABLE users ADD COLUMN phone VARCHAR(20);
    INSERT INTO schema_migrations (version, description) 
    VALUES ('2024_02_19_001', 'Add phone column to users');
COMMIT;
```

---

## 📚 Summary

### Key Takeaways

1. **CREATE TABLE**: Define structure with constraints, indexes, foreign keys
2. **ALTER TABLE**: Modify schema; use ALGORITHM=INPLACE for large tables
3. **DROP**: Permanent deletion; always check dependencies first
4. **TRUNCATE**: Fast way to empty table; prefer over DELETE for full clear
5. **IF EXISTS/IF NOT EXISTS**: Make scripts idempotent
6. **Online DDL**: Use for zero-downtime deployments
7. **Always backup**: Before destructive operations
8. **Version migrations**: Track all schema changes
9. **Proper data types**: Choose correct types from the start
10. **Index strategically**: Add indexes for common queries during CREATE

---

**Next:** [02_DML_Commands.md](02_DML_Commands.md) - Data manipulation  
**Related:** [../02_Data_Modeling/](../02_Data_Modeling/) - Schema design

---

*Last Updated: February 19, 2026*
