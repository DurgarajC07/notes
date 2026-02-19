# 🔢 Data Types - Choosing the Right Type for Your Data

> Selecting the correct data type is critical for storage efficiency, query performance, data integrity, and application correctness. Understanding data types prevents bugs and optimizes database performance.

---

## 📖 1. Concept Explanation

### What are Data Types?

**Data Types** define what kind of values can be stored in a column, how the data is stored internally, how much space it consumes, and what operations can be performed on it.

**Why Data Types Matter:**
- **Storage efficiency**: Wrong types waste disk space
- **Query performance**: Proper types enable index optimizations
- **Data integrity**: Types prevent invalid data
- **Application correctness**: Type mismatches cause bugs

---

### Core Data Type Categories

```
DATA TYPE HIERARCHY
│
├── NUMERIC TYPES
│   ├── Integers (TINYINT, INT, BIGINT)
│   ├── Decimals (DECIMAL, NUMERIC)
│   └── Floating Point (FLOAT, DOUBLE)
│
├── STRING TYPES
│   ├── Fixed Length (CHAR)
│   ├── Variable Length (VARCHAR, TEXT)
│   └── Binary (BLOB, BYTEA)
│
├── DATE/TIME TYPES
│   ├── DATE (year-month-day)
│   ├── TIME (hour:minute:second)
│   ├── DATETIME, TIMESTAMP (both date and time)
│   └── INTERVAL (time duration)
│
├── BOOLEAN TYPES
│   └── BOOLEAN, BOOL (true/false/null)
│
├── JSON TYPES
│   ├── JSON (text storage)
│   └── JSONB (binary storage - PostgreSQL)
│
├── SPATIAL TYPES
│   ├── POINT, LINESTRING, POLYGON
│   └── GEOMETRY, GEOGRAPHY
│
├── ARRAY TYPES
│   └── Arrays of any type (PostgreSQL, MongoDB)
│
├── UUID TYPES
│   └── UUID (universally unique identifier)
│
└── CUSTOM TYPES
    ├── ENUM (predefined set of values)
    └── COMPOSITE TYPES
```

---

## 🧠 2. Why It Matters in Real Systems

### Storage Cost Impact

**Bad: Using BIGINT for small values**
```sql
-- Storing age (0-120) as BIGINT
CREATE TABLE users (
    id BIGINT,
    age BIGINT  -- ❌ 8 bytes per row
);

-- 10 million users = 80 MB wasted
-- (Should be TINYINT = 1 byte = 10 MB)
```

**Good: Right-sized types**
```sql
CREATE TABLE users (
    id BIGINT,        -- 8 bytes (large ID space needed)
    age TINYINT,      -- 1 byte (0-255 range sufficient)
    status CHAR(1),   -- 1 byte (single character)
    email VARCHAR(255) -- Variable length
);

-- Savings: 7 bytes per row × 10M users = 70 MB saved
```

---

### Performance Impact

**Index size directly affects query speed:**

```sql
-- Bad: VARCHAR(255) for country codes
CREATE TABLE addresses (
    country VARCHAR(255)  -- ❌ Index 255 bytes per entry
);

-- Index size: 255 bytes × 10M rows = 2.55 GB

-- Good: CHAR(2) for ISO country codes
CREATE TABLE addresses (
    country CHAR(2)  -- ✅ Index 2 bytes per entry
);

-- Index size: 2 bytes × 10M rows = 20 MB
-- 127x smaller index = much faster queries!
```

---

### Data Integrity

```sql
-- Bad: Storing dates as strings
CREATE TABLE events (
    event_date VARCHAR(20)  -- ❌ Allows '2024-13-45' (invalid)
);

INSERT INTO events VALUES ('February 30, 2024');  -- Accepts invalid date!

-- Good: Use DATE type
CREATE TABLE events (
    event_date DATE  -- ✅ Only valid dates allowed
);

INSERT INTO events VALUES ('2024-02-30');  -- ERROR: invalid date
```

---

## ⚙️ 3. Internal Working

### Numeric Types - Storage & Range

| Type          | Storage | Signed Range                     | Unsigned Range     |
|---------------|---------|----------------------------------|--------------------|
| **TINYINT**   | 1 byte  | -128 to 127                      | 0 to 255           |
| **SMALLINT**  | 2 bytes | -32,768 to 32,767                | 0 to 65,535        |
| **MEDIUMINT** | 3 bytes | -8,388,608 to 8,388,607          | 0 to 16,777,215    |
| **INT**       | 4 bytes | -2,147,483,648 to 2,147,483,647  | 0 to 4,294,967,295 |
| **BIGINT**    | 8 bytes | -9.2×10¹⁸ to 9.2×10¹⁸            | 0 to 18.4×10¹⁸     |

**Decimal vs Float:**

```sql
-- DECIMAL: Exact precision (financial data)
DECIMAL(10, 2)  -- 10 total digits, 2 after decimal
-- Stored as: 12345678.90 (exact)

-- FLOAT/DOUBLE: Approximate (scientific data)
FLOAT  -- 4 bytes, ~7 decimal digits precision
DOUBLE -- 8 bytes, ~15 decimal digits precision
-- Stored as: 12345678.899999976 (approximate)
```

**Financial Example:**
```sql
-- Bad: Using FLOAT for money
CREATE TABLE transactions (
    amount FLOAT  -- ❌ Rounding errors in calculations
);

SELECT SUM(amount) FROM transactions;
-- Result: 999.9999999997 instead of 1000.00

-- Good: Use DECIMAL
CREATE TABLE transactions (
    amount DECIMAL(10, 2)  -- ✅ Exact precision
);

SELECT SUM(amount) FROM transactions;
-- Result: 1000.00 (exact)
```

---

### String Types - Storage Efficiency

**CHAR vs VARCHAR:**

```sql
-- CHAR(n): Fixed-length, space-padded
CREATE TABLE users (
    country_code CHAR(2)  -- Always 2 bytes
);

INSERT INTO users VALUES ('US');  -- Stored as 'US'
INSERT INTO users VALUES ('A');   -- Stored as 'A ' (space-padded)

-- VARCHAR(n): Variable-length, 1-2 byte length prefix
CREATE TABLE users (
    name VARCHAR(100)  -- 1-100 bytes + 1-2 byte overhead
);

INSERT INTO users VALUES ('Jo');      -- Stored as: [2][Jo] = 3 bytes
INSERT INTO users VALUES ('Alexander'); -- Stored as: [9][Alexander] = 10 bytes
```

**TEXT Types:**

| Type         | Max Length        | Use Case                     |
|--------------|-------------------|------------------------------|
| **TINYTEXT** | 255 bytes         | Short descriptions           |
| **TEXT**     | 65,535 bytes      | Articles, comments           |
| **MEDIUMTEXT** | 16 MB           | Large documents              |
| **LONGTEXT** | 4 GB              | Books, extensive content     |

---

### Date/Time Types

**DATETIME vs TIMESTAMP:**

```sql
-- DATETIME: Stores absolute date/time (no timezone)
DATETIME  -- Range: '1000-01-01' to '9999-12-31'
          -- Storage: 8 bytes
          -- Stored as: 2024-02-18 14:30:00

-- TIMESTAMP: Stores Unix timestamp (UTC conversion)
TIMESTAMP -- Range: '1970-01-01' to '2038-01-19' (32-bit)
          -- Storage: 4 bytes
          -- Stored as: seconds since 1970-01-01 00:00:00 UTC
          -- Auto-converts to session timezone
```

**Real-World Example:**
```sql
-- User in New York creates event at 3 PM EST
INSERT INTO events (event_time) VALUES ('2024-02-18 15:00:00');

-- DATETIME: Stores '2024-02-18 15:00:00' (no context)
-- User in Tokyo sees: '2024-02-18 15:00:00' ❌ Wrong!

-- TIMESTAMP: Stores as UTC (20:00:00 UTC)
-- User in Tokyo sees: '2024-02-19 05:00:00' ✅ Correct!
```

---

### JSON Types (PostgreSQL)

**JSON vs JSONB:**

```sql
-- JSON: Text storage, re-parsed on every read
CREATE TABLE products (
    metadata JSON  -- Stored as plain text
);

-- JSONB: Binary storage, pre-parsed, indexable
CREATE TABLE products (
    metadata JSONB  -- Stored as binary, decomposed
);

-- Performance comparison:
-- JSON:
--   - Faster to write (no parsing)
--   - Slower to query (must parse every time)
--   - No indexing support

-- JSONB:
--   - Slower to write (parsed once)
--   - Much faster to query
--   - Full GIN index support
```

**JSONB Example:**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    profile JSONB
);

-- Create GIN index for fast JSON queries
CREATE INDEX idx_profile ON users USING GIN (profile);

-- Fast queries:
SELECT * FROM users WHERE profile->>'city' = 'New York';
SELECT * FROM users WHERE profile @> '{"premium": true}';
```

---

## ✅ 4. Best Practices

### Choosing the Right Numeric Type

```sql
-- ✅ Use smallest type that fits your data
CREATE TABLE products (
    id BIGINT,              -- Large ID space for scalability
    category_id INT,        -- Up to 4 billion categories
    stock_quantity SMALLINT, -- -32K to 32K sufficient
    rating DECIMAL(3,2),    -- 0.00 to 9.99
    price DECIMAL(10,2),    -- Exact financial precision
    weight_kg FLOAT         -- Approximate weight ok
);
```

### String Type Selection

```sql
-- ✅ Use CHAR for fixed-length data
CREATE TABLE users (
    country_code CHAR(2),      -- Always 2 characters
    gender CHAR(1),            -- M/F/O
    status CHAR(1)             -- A/I/S
);

-- ✅ Use VARCHAR for variable-length data
CREATE TABLE users (
    email VARCHAR(255),        -- Email length varies
    name VARCHAR(100),         -- Name length varies
    bio TEXT                   -- Long, unbounded text
);

-- ❌ Don't over-allocate VARCHAR
CREATE TABLE users (
    email VARCHAR(1000)  -- ❌ Wastes memory, slows indexes
);
```

### Date/Time Best Practices

```sql
-- ✅ Use TIMESTAMP WITH TIME ZONE for user data
CREATE TABLE events (
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ✅ Use DATE for date-only data
CREATE TABLE employees (
    birth_date DATE,  -- No time component needed
    hire_date DATE
);

-- ✅ Use INTERVAL for durations
CREATE TABLE videos (
    duration INTERVAL  -- '02:30:15' (2 hours, 30 min, 15 sec)
);
```

### ENUM for Fixed Sets

```sql
-- ✅ Use ENUM for small, stable sets
CREATE TYPE order_status AS ENUM (
    'pending',
    'processing',
    'shipped',
    'delivered',
    'cancelled'
);

CREATE TABLE orders (
    status order_status DEFAULT 'pending'
);

-- Benefits:
-- - Type safety (only valid values allowed)
-- - Storage efficient (stored as integer internally)
-- - Self-documenting code
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Using VARCHAR for Everything

```sql
-- ❌ Bad: VARCHAR for all columns
CREATE TABLE users (
    id VARCHAR(50),          -- Should be INT or BIGINT
    age VARCHAR(3),          -- Should be TINYINT
    is_active VARCHAR(5),    -- Should be BOOLEAN
    balance VARCHAR(20),     -- Should be DECIMAL
    created_at VARCHAR(30)   -- Should be TIMESTAMP
);

-- Problems:
-- - No data validation (accepts '999' for age)
-- - Wastes storage (VARCHAR overhead)
-- - Breaks sorting ('10' < '2' alphabetically)
-- - Can't use date/time functions
```

**Fix:**
```sql
-- ✅ Good: Proper types
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    age TINYINT CHECK (age BETWEEN 0 AND 120),
    is_active BOOLEAN DEFAULT TRUE,
    balance DECIMAL(12,2),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

### Mistake 2: Using FLOAT for Money

```sql
-- ❌ Bad: FLOAT for financial data
CREATE TABLE transactions (
    amount FLOAT
);

INSERT INTO transactions VALUES (0.1), (0.2);
SELECT SUM(amount) FROM transactions;
-- Result: 0.30000000000000004 ❌

-- ✅ Good: DECIMAL for financial data
CREATE TABLE transactions (
    amount DECIMAL(12,2)
);

INSERT INTO transactions VALUES (0.1), (0.2);
SELECT SUM(amount) FROM transactions;
-- Result: 0.30 ✅
```

---

### Mistake 3: Over-Allocating VARCHAR

```sql
-- ❌ Bad: Excessive VARCHAR lengths
CREATE TABLE users (
    email VARCHAR(5000),  -- Only needs 255
    name VARCHAR(1000),   -- Only needs 100
    zip_code VARCHAR(500) -- Only needs 10
);

-- Problems:
-- - Huge memory allocations for sorts/joins
-- - Bloated indexes
-- - Wasted buffer pool space
```

**Fix:**
```sql
-- ✅ Good: Right-sized VARCHAR
CREATE TABLE users (
    email VARCHAR(255),    -- RFC 5321 limit is 254
    name VARCHAR(100),     -- Reasonable max name length
    zip_code VARCHAR(10)   -- Longest ZIP+4: 10 chars
);
```

---

### Mistake 4: Storing Dates as Strings

```sql
-- ❌ Bad: Dates as strings
CREATE TABLE events (
    event_date VARCHAR(20)
);

INSERT INTO events VALUES 
    ('2024-01-15'),
    ('01/15/2024'),
    ('15-Jan-2024'),
    ('January 15, 2024');

-- Problems:
-- - Inconsistent formats
-- - Can't sort chronologically
-- - Can't use date functions
-- - No validation (accepts '2024-13-45')

-- Can't do simple queries:
SELECT * FROM events 
WHERE event_date > '2024-01-01';  -- String comparison! ❌
```

**Fix:**
```sql
-- ✅ Good: Use DATE type
CREATE TABLE events (
    event_date DATE
);

INSERT INTO events VALUES ('2024-01-15');

-- Works correctly:
SELECT * FROM events 
WHERE event_date > '2024-01-01';  -- Date comparison ✅

-- Can use date functions:
SELECT 
    event_date,
    EXTRACT(YEAR FROM event_date) as year,
    event_date + INTERVAL '7 days' as next_week
FROM events;
```

---

### Mistake 5: Not Using BOOLEAN

```sql
-- ❌ Bad: Many representations of boolean
CREATE TABLE users (
    is_active VARCHAR(5),    -- 'true', 'false', 'yes', 'no'
    is_admin CHAR(1),        -- 'Y', 'N', 'T', 'F', '1', '0'
    is_verified INT          -- 0, 1, -1, 2
);

-- ✅ Good: Use BOOLEAN
CREATE TABLE users (
    is_active BOOLEAN DEFAULT TRUE,
    is_admin BOOLEAN DEFAULT FALSE,
    is_verified BOOLEAN
);

-- Clear queries:
SELECT * FROM users WHERE is_active = TRUE;
SELECT * FROM users WHERE NOT is_admin;
```

---

## 🔐 6. Security Considerations

### SQL Injection via Type Mismatch

```sql
-- Vulnerable: String concatenation
sql = "SELECT * FROM users WHERE id = " + user_input;
-- If user_input = "1 OR 1=1", returns all users! ❌

-- Safe: Parameterized queries with proper types
-- Application code:
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
-- Database enforces INT type, rejects malicious input ✅
```

### Information Disclosure

```sql
-- ❌ Exposing too much precision
CREATE TABLE salaries (
    amount DECIMAL(12,2)  -- Exact salary visible
);

-- ✅ Store exact, display rounded
CREATE TABLE salaries (
    amount DECIMAL(12,2)
);

-- In application/view:
SELECT ROUND(amount/1000)*1000 as salary_range FROM salaries;
-- Returns: $75,000 instead of $75,234.56
```

### UUID for Security

```sql
-- ❌ Bad: Sequential IDs expose information
CREATE TABLE users (
    id SERIAL PRIMARY KEY  -- Predictable: 1, 2, 3...
);
-- Attackers can enumerate: /api/users/1, /api/users/2, etc.

-- ✅ Good: UUID prevents enumeration
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid()
);
-- URLs like: /api/users/550e8400-e29b-41d4-a716-446655440000
-- Can't guess other user IDs
```

---

## 🚀 7. Performance Optimization Techniques

### Index Efficiency

```sql
-- ❌ Bad: Large VARCHAR in index
CREATE TABLE logs (
    id BIGINT,
    message VARCHAR(5000),
    INDEX idx_message (message)  -- Huge index!
);

-- ✅ Good: Index prefix or hash
CREATE TABLE logs (
    id BIGINT,
    message TEXT,
    message_hash CHAR(32),  -- MD5 hash
    INDEX idx_hash (message_hash)  -- Small, fast index
);

-- Or use prefix index:
CREATE INDEX idx_message ON logs (message(100));  -- First 100 chars
```

### Memory-Efficient Sorts

```sql
-- ❌ Bad: Large strings in ORDER BY
SELECT * FROM users 
ORDER BY biography;  -- TEXT column, huge sorts!

-- ✅ Good: Order by indexed smaller column
SELECT * FROM users 
ORDER BY created_at DESC;  -- TIMESTAMP, efficient sort
```

### Partitioning by Type

```sql
-- Partition large table by date for fast queries
CREATE TABLE events (
    id BIGINT,
    event_date DATE,
    data JSONB
) PARTITION BY RANGE (event_date);

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Query hits only relevant partition:
SELECT * FROM events 
WHERE event_date BETWEEN '2024-01-01' AND '2024-01-31';
-- Only scans events_2024_01 partition ✅
```

---

## 🧪 8. Examples

### E-commerce Product Catalog

```sql
CREATE TABLE products (
    -- Primary key
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    
    -- Product info
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    
    -- Pricing
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    cost DECIMAL(10,2) CHECK (cost >= 0),
    discount_percent DECIMAL(5,2) CHECK (discount_percent BETWEEN 0 AND 100),
    
    -- Inventory
    stock_quantity INT DEFAULT 0 CHECK (stock_quantity >= 0),
    weight_kg FLOAT CHECK (weight_kg > 0),
    
    -- Categorization
    category_id INT NOT NULL,
    brand_id INT,
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    is_featured BOOLEAN DEFAULT FALSE,
    
    -- Metadata
    metadata JSONB,  -- Flexible attributes
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_category (category_id),
    INDEX idx_active_featured (is_active, is_featured),
    INDEX idx_price (price),
    FULLTEXT INDEX idx_search (name, description)
);
```

### User Management System

```sql
CREATE TABLE users (
    -- Primary key (UUID for security)
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Authentication
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash CHAR(60) NOT NULL,  -- bcrypt hash
    
    -- Profile
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    date_of_birth DATE CHECK (date_of_birth <= CURRENT_DATE),
    
    -- Contact
    phone VARCHAR(20),
    country_code CHAR(2),  -- ISO 3166-1 alpha-2
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    is_verified BOOLEAN DEFAULT FALSE,
    is_premium BOOLEAN DEFAULT FALSE,
    
    -- Preferences (JSONB for flexibility)
    preferences JSONB DEFAULT '{}'::jsonb,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_login_at TIMESTAMP WITH TIME ZONE,
    
    -- Indexes
    INDEX idx_email (email),
    INDEX idx_active (is_active) WHERE is_active = TRUE,
    INDEX idx_preferences USING GIN (preferences)
);
```

### Time-Series Sensor Data

```sql
CREATE TABLE sensor_readings (
    -- Time and sensor
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    sensor_id INT NOT NULL,
    
    -- Readings (float for sensor data)
    temperature FLOAT,
    humidity FLOAT,
    pressure FLOAT,
    
    -- Status
    status SMALLINT,  -- Status codes 0-255
    
    -- Primary key (composite)
    PRIMARY KEY (sensor_id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Create partitions by month
CREATE TABLE sensor_readings_2024_02 PARTITION OF sensor_readings
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

### Financial Transactions

```sql
CREATE TABLE transactions (
    -- Primary key
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    transaction_uuid UUID UNIQUE DEFAULT gen_random_uuid(),
    
    -- Accounts
    from_account_id BIGINT NOT NULL,
    to_account_id BIGINT NOT NULL,
    
    -- Amount (exact precision!)
    amount DECIMAL(19,4) NOT NULL CHECK (amount > 0),
    currency CHAR(3) NOT NULL,  -- ISO 4217 (USD, EUR, etc.)
    
    -- Transaction type
    type ENUM('transfer', 'payment', 'refund', 'reversal'),
    status ENUM('pending', 'completed', 'failed', 'cancelled'),
    
    -- Metadata
    description VARCHAR(500),
    reference_id VARCHAR(100),
    
    -- Timestamps (immutable for audit trail)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    completed_at TIMESTAMP WITH TIME ZONE,
    
    -- Indexes
    INDEX idx_from_account (from_account_id, created_at),
    INDEX idx_to_account (to_account_id, created_at),
    INDEX idx_status (status) WHERE status = 'pending',
    INDEX idx_reference (reference_id)
);
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Shopify - Product Attributes

**Challenge:** E-commerce products have varying attributes (clothing has sizes, electronics have specs).

**Solution:** JSONB for flexible schema

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(200),
    base_price DECIMAL(10,2),
    attributes JSONB  -- Flexible product attributes
);

-- Clothing product
INSERT INTO products VALUES (
    1, 
    'T-Shirt', 
    19.99,
    '{
        "sizes": ["S", "M", "L", "XL"],
        "colors": ["red", "blue", "black"],
        "material": "100% Cotton"
    }'
);

-- Electronics product
INSERT INTO products VALUES (
    2,
    'Laptop',
    999.99,
    '{
        "cpu": "Intel i7",
        "ram": "16GB",
        "storage": "512GB SSD",
        "screen": "15.6 inch"
    }'
);

-- Query by JSON attributes
SELECT * FROM products 
WHERE attributes->>'material' = '100% Cotton';

SELECT * FROM products 
WHERE attributes @> '{"cpu": "Intel i7"}';
```

---

### Case Study 2: Stripe - Financial Precision

**Challenge:** Financial calculations must be exact (no rounding errors).

**Solution:** DECIMAL with proper precision

```sql
-- Store amounts in smallest currency unit (cents)
CREATE TABLE payments (
    id BIGINT PRIMARY KEY,
    amount_cents BIGINT NOT NULL,  -- 1999 = $19.99
    currency CHAR(3) NOT NULL,
    
    -- Or use DECIMAL for dollar amounts
    amount_decimal DECIMAL(12,2)  -- $19.99
);

-- Stripe's approach: Store integers (cents)
-- Benefits:
-- - No floating point errors
-- - Exact arithmetic
-- - Smaller storage than DECIMAL
-- - Faster integer operations

-- Calculate 2.5% processing fee (exact)
UPDATE payments 
SET fee_cents = (amount_cents * 25) / 1000  -- Integer division
WHERE id = 123;
```

---

### Case Study 3: Uber - Geo-Location

**Challenge:** Store and query millions of driver locations efficiently.

**Solution:** Spatial data types with geo-indexing

```sql
-- PostGIS extension
CREATE EXTENSION postgis;

CREATE TABLE driver_locations (
    driver_id BIGINT PRIMARY KEY,
    location GEOGRAPHY(POINT, 4326),  -- WGS 84 coordinate system
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Spatial index for fast queries
CREATE INDEX idx_location ON driver_locations USING GIST (location);

-- Find drivers within 5km of user
SELECT driver_id, 
       ST_Distance(location, ST_MakePoint(-73.935242, 40.730610)::geography) as distance_m
FROM driver_locations
WHERE ST_DWithin(
    location,
    ST_MakePoint(-73.935242, 40.730610)::geography,
    5000  -- 5km radius
)
ORDER BY distance_m
LIMIT 10;
```

---

### Case Study 4: Twitter - User IDs

**Challenge:** Need globally unique IDs that can be generated at scale.

**Solution:** Snowflake IDs (64-bit integers)

```sql
-- Twitter Snowflake ID format (BIGINT - 64 bits):
-- - 41 bits: Timestamp (milliseconds)
-- - 10 bits: Machine ID
-- - 12 bits: Sequence number

CREATE TABLE tweets (
    id BIGINT PRIMARY KEY,  -- Snowflake ID
    user_id BIGINT NOT NULL,
    text VARCHAR(280),  -- Twitter's character limit
    created_at TIMESTAMP WITH TIME ZONE
);

-- Benefits of Snowflake IDs:
-- - Time-ordered (roughly sortable by creation time)
-- - No database coordination needed
-- - Can generate millions per second
-- - Fits in BIGINT (efficient storage & indexing)
```

---

### Case Study 5: Slack - Message Search

**Challenge:** Fast full-text search across millions of messages.

**Solution:** Full-text indexes + denormalized search columns

```sql
CREATE TABLE messages (
    id BIGINT PRIMARY KEY,
    channel_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    content TEXT,
    content_search TSVECTOR,  -- Pre-computed search vector
    created_at TIMESTAMP WITH TIME ZONE
);

-- Full-text index
CREATE INDEX idx_search ON messages USING GIN (content_search);

-- Trigger to maintain search vector
CREATE TRIGGER messages_search_update
BEFORE INSERT OR UPDATE ON messages
FOR EACH ROW EXECUTE FUNCTION
tsvector_update_trigger(content_search, 'pg_catalog.english', content);

-- Fast search query
SELECT * FROM messages
WHERE content_search @@ to_tsquery('urgent & meeting');
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: When should you use DECIMAL vs FLOAT?

**Answer:**

**Use DECIMAL when:**
- Financial calculations (money, prices, taxes)
- Exact precision required
- No rounding errors acceptable
- Example: `DECIMAL(10,2)` for currency

**Use FLOAT when:**
- Scientific measurements
- Approximate values acceptable
- Performance critical (FLOAT operations faster)
- Example: Sensor readings, GPS coordinates

**Code Example:**
```sql
-- Financial: Use DECIMAL
CREATE TABLE invoices (
    total DECIMAL(12,2)  -- Exact: $12345.67
);

-- Scientific: Use FLOAT
CREATE TABLE sensor_data (
    temperature FLOAT  -- Approximate: 23.456789
);
```

---

### Q2: How do you choose between VARCHAR and TEXT?

**Answer:**

**VARCHAR(n):**
- When you know the maximum length
- Need to enforce length constraint
- Want indexed searches (can index full column)
- Examples: emails, names, SKUs

**TEXT:**
- Unknown or very large content
- No length limit needed
- Full-text search with FTS indexes
- Examples: articles, comments, descriptions

**Performance Note:**
```sql
-- VARCHAR(255): Can fully index
CREATE INDEX idx_email ON users (email);  -- Fast

-- TEXT: Index prefix only (some databases)
CREATE INDEX idx_bio ON users (bio(100));  -- First 100 chars

-- Or use full-text index
CREATE INDEX idx_bio_fts ON users USING GIN (to_tsvector('english', bio));
```

---

### Q3: Why use TIMESTAMP WITH TIME ZONE vs DATETIME?

**Answer:**

**TIMESTAMP WITH TIME ZONE (recommended):**
- Stores UTC, converts to user timezone
- Correct for global applications
- Handles daylight saving automatically

**DATETIME:**
- Stores literal date/time (no timezone)
- Ambiguous across timezones
- Simpler for single-timezone apps

**Example:**
```sql
-- User in NY schedules meeting at 3 PM
-- DATETIME: Stores '2024-02-18 15:00:00' (ambiguous)
-- TIMESTAMPTZ: Stores as UTC '2024-02-18 20:00:00+00'

-- User in Tokyo queries:
SELECT event_time AT TIME ZONE 'Asia/Tokyo' FROM events;
-- Returns: '2024-02-19 05:00:00' (correct conversion)
```

---

### Q4: When should you use UUID vs BIGINT for primary keys?

**Answer:**

**UUID pros:**
- Globally unique (no coordination needed)
- Security (non-guessable)
- Distributed systems (no central ID generation)

**UUID cons:**
- Larger storage (16 bytes vs 8 bytes)
- Slower indexes (random, no locality)
- Not sequential (poor index performance)

**BIGINT pros:**
- Smaller (8 bytes)
- Sequential (excellent index performance)
- Simpler (easy to debug)

**BIGINT cons:**
- Need coordination in distributed systems
- Predictable (security issue)
- Limited to single database

**Recommendation:**
```sql
-- Public-facing IDs: UUID
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid()
);

-- Internal IDs: BIGINT
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT
);

-- Hybrid approach: BIGINT + UUID
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,  -- Fast joins
    public_id UUID UNIQUE DEFAULT gen_random_uuid()  -- Expose to users
);
```

---

### Q5: How does JSONB differ from JSON in PostgreSQL?

**Answer:**

**JSON:**
- Stored as text
- Exact preservations (whitespace, order)
- Faster to insert
- Must parse on every read
- No indexing

**JSONB:**
- Stored as binary (decomposed)
- Minor overhead on insert (parsing)
- Much faster to query
- Full GIN index support
- No whitespace/order preservation

**Performance Example:**
```sql
-- JSON: Slow query (must parse)
SELECT * FROM products WHERE data->>'category' = 'electronics';

-- JSONB: Fast with GIN index
CREATE INDEX idx_data ON products USING GIN (data);
SELECT * FROM products WHERE data @> '{"category": "electronics"}';

-- JSONB: 10-100x faster for queries
```

**Recommendation:** Always use JSONB unless you need exact text preservation.

---

### Q6: Explain the difference between INT, BIGINT, and SERIAL.

**Answer:**

**INT:**
- 4 bytes, range: -2.1B to 2.1B
- Use for moderate-sized IDs

**BIGINT:**
- 8 bytes, range: -9.2×10¹⁸ to 9.2×10¹⁸
- Use for large tables (billions of rows)

**SERIAL / BIGSERIAL:**
- Auto-incrementing integer
- Shorthand for `INT/BIGINT AUTO_INCREMENT`

**Example:**
```sql
-- SERIAL expands to:
CREATE TABLE users (
    id SERIAL PRIMARY KEY
);

-- Equivalent to:
CREATE SEQUENCE users_id_seq;
CREATE TABLE users (
    id INT DEFAULT nextval('users_id_seq') PRIMARY KEY
);

-- When to use each:
-- SERIAL: Small-medium tables (< 2 billion rows)
-- BIGSERIAL: Large tables, future-proof
```

---

### Q7: What's the best way to store boolean values?

**Answer:**

**Use native BOOLEAN type:**
```sql
CREATE TABLE users (
    is_active BOOLEAN DEFAULT TRUE
);

-- Clear queries
SELECT * FROM users WHERE is_active = TRUE;
SELECT * FROM users WHERE NOT is_verified;
```

**Avoid these anti-patterns:**
```sql
-- ❌ Bad: String representation
is_active VARCHAR(5)  -- 'true', 'false', 'yes', 'no'

-- ❌ Bad: Integer representation
is_active TINYINT  -- 0, 1, -1, 2 (ambiguous)

-- ❌ Bad: Character
is_active CHAR(1)  -- 'Y', 'N', 'T', 'F', '1', '0' (confusing)
```

**Storage:** BOOLEAN = 1 byte (or 1 bit in some databases)

---

### Q8: How do you handle currency in a multi-currency system?

**Answer:**

**Store amount + currency separately:**
```sql
CREATE TABLE transactions (
    id BIGINT PRIMARY KEY,
    amount DECIMAL(19,4) NOT NULL,  -- Max 15 digits + 4 decimals
    currency CHAR(3) NOT NULL,      -- ISO 4217: USD, EUR, JPY
    
    CHECK (amount > 0)
);

-- Why 4 decimals?
-- - Most currencies: 2 decimals (USD: $1.23)
-- - Some currencies: 3 decimals (KWD: 1.234)
-- - Crypto: many decimals (BTC: 0.00001234)

-- Exchange rates: Store in separate table
CREATE TABLE exchange_rates (
    from_currency CHAR(3),
    to_currency CHAR(3),
    rate DECIMAL(18,8),  -- High precision for accuracy
    effective_date DATE,
    
    PRIMARY KEY (from_currency, to_currency, effective_date)
);
```

**Alternative: Store in smallest unit (cents)**
```sql
-- Stripe's approach
CREATE TABLE payments (
    amount_cents BIGINT NOT NULL,  -- 1234 = $12.34
    currency CHAR(3) NOT NULL
);

-- Benefits:
-- - Integer arithmetic (exact, fast)
-- - No decimal precision issues
-- - Smaller storage
```

---

## 🧩 11. Design Patterns

### Pattern 1: Type-Safe Enums

**Problem:** Need constrained set of values with type safety.

**Solution:**
```sql
-- Create custom ENUM type
CREATE TYPE user_role AS ENUM (
    'guest',
    'member',
    'moderator',
    'admin',
    'super_admin'
);

CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    role user_role DEFAULT 'member'
);

-- Benefits:
-- - Type safety (only valid values)
-- - Self-documenting
-- - Efficient storage (integer internally)
-- - Database-enforced constraints

-- Usage:
INSERT INTO users (role) VALUES ('admin');  -- ✅
INSERT INTO users (role) VALUES ('hacker'); -- ❌ ERROR
```

---

### Pattern 2: Polymorphic Associations with JSONB

**Problem:** Different entity types need different attributes.

**Solution:**
```sql
CREATE TABLE notifications (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    type VARCHAR(50) NOT NULL,
    data JSONB NOT NULL,  -- Type-specific data
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Email notification
INSERT INTO notifications VALUES (
    1, 100, 'email',
    '{"to": "user@example.com", "subject": "Welcome", "body": "..."}'
);

-- SMS notification
INSERT INTO notifications VALUES (
    2, 100, 'sms',
    '{"phone": "+1234567890", "message": "Verification code: 123456"}'
);

-- Push notification
INSERT INTO notifications VALUES (
    3, 100, 'push',
    '{"device_id": "abc123", "title": "New message", "badge": 1}'
);

-- Query by type
SELECT * FROM notifications 
WHERE type = 'email' 
  AND data->>'to' = 'user@example.com';
```

---

### Pattern 3: Audit Trail with Immutable Types

**Problem:** Need complete audit history that cannot be modified.

**Solution:**
```sql
CREATE TABLE audit_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    table_name VARCHAR(50) NOT NULL,
    record_id BIGINT NOT NULL,
    action ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    old_values JSONB,  -- Before state
    new_values JSONB,  -- After state
    changed_by BIGINT NOT NULL,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    
    -- Make immutable in application logic
    INDEX idx_record (table_name, record_id),
    INDEX idx_user (changed_by),
    INDEX idx_time (changed_at)
);

-- Trigger example (PostgreSQL)
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, record_id, action, old_values, new_values, changed_by)
        VALUES (TG_TABLE_NAME, OLD.id, 'UPDATE', row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb, current_user_id());
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

### Pattern 4: Soft Delete with Timestamps

**Problem:** Need to delete records but maintain history.

**Solution:**
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    deleted_at TIMESTAMP WITH TIME ZONE,  -- NULL = not deleted
    
    -- Unique constraint that ignores soft-deleted
    UNIQUE (email) WHERE deleted_at IS NULL
);

-- Soft delete
UPDATE users 
SET deleted_at = NOW(), is_active = FALSE 
WHERE id = 123;

-- Query active users
SELECT * FROM users WHERE deleted_at IS NULL;

-- Partial index for performance
CREATE INDEX idx_active_users ON users (id) WHERE deleted_at IS NULL;
```

---

### Pattern 5: Multi-Tenant Data with Composite Keys

**Problem:** SaaS application with multiple tenants sharing database.

**Solution:**
```sql
CREATE TABLE tenant_users (
    tenant_id INT NOT NULL,
    user_id BIGINT NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Composite primary key
    PRIMARY KEY (tenant_id, user_id),
    
    -- Unique per tenant
    UNIQUE (tenant_id, email)
);

-- All queries must include tenant_id
SELECT * FROM tenant_users 
WHERE tenant_id = 1 AND user_id = 100;

-- Partition by tenant for large tables
CREATE TABLE tenant_orders (
    tenant_id INT NOT NULL,
    order_id BIGINT NOT NULL,
    amount DECIMAL(10,2),
    PRIMARY KEY (tenant_id, order_id)
) PARTITION BY LIST (tenant_id);
```

---

## 📚 Summary

### Key Takeaways

1. **Choose the smallest type that fits your data** - saves storage, improves performance
2. **Use DECIMAL for money** - avoid floating-point rounding errors
3. **Use proper date/time types** - enables time functions and correct sorting
4. **Use BOOLEAN for binary states** - clear intent, efficient storage
5. **Use JSONB for flexible schemas** - when structure varies by record
6. **Use UUID for public IDs** - security and distributed systems
7. **Use ENUM for fixed sets** - type safety and self-documentation
8. **Right-size VARCHAR** - don't over-allocate, hurts indexes
9. **Use TIMESTAMP WITH TIME ZONE** - correct for global applications
10. **Consider index size** - smaller types = faster indexes

### Data Type Decision Tree

```
Need to store...

└─ Numbers?
   ├─ Money? → DECIMAL(19,4)
   ├─ Integer?
   │  ├─ 0-255? → TINYINT
   │  ├─ 0-65K? → SMALLINT
   │  ├─ 0-4B? → INT
   │  └─ Larger? → BIGINT
   └─ Approximate? → FLOAT or DOUBLE

└─ Text?
   ├─ Fixed length? → CHAR(n)
   ├─ Variable, known max? → VARCHAR(n)
   └─ Large/unknown? → TEXT

└─ Date/Time?
   ├─ Date only? → DATE
   ├─ Time only? → TIME
   ├─ Both?
   │  ├─ Global app? → TIMESTAMP WITH TIME ZONE
   │  └─ Single timezone? → DATETIME or TIMESTAMP
   └─ Duration? → INTERVAL

└─ True/False? → BOOLEAN

└─ Flexible structure? → JSONB

└─ Fixed set of values? → ENUM

└─ Binary data? → BLOB or BYTEA

└─ Location? → GEOGRAPHY/GEOMETRY (PostGIS)

└─ Unique ID?
   ├─ Public-facing? → UUID
   └─ Internal? → BIGINT SERIAL
```

### Production Checklist

- [ ] All money fields use DECIMAL, not FLOAT
- [ ] Date/time fields use proper types, not VARCHAR
- [ ] VARCHAR lengths are appropriate, not over-allocated
- [ ] Boolean fields use BOOLEAN type
- [ ] Primary keys have appropriate type (BIGINT for large tables)
- [ ] Foreign keys match the type of referenced column
- [ ] Indexes consider column size (smaller = faster)
- [ ] CHECK constraints validate data ranges
- [ ] JSONB used instead of JSON (PostgreSQL)
- [ ] Timestamp columns include timezone

---

**Next Steps:**
- Review [05_Constraints.md](05_Constraints.md) for enforcing data rules
- Study [06_Keys_And_Indexes.md](06_Keys_And_Indexes.md) for index strategies
- Explore [../../05_Query_Optimization/05_Index_Selection.md](../../05_Query_Optimization/05_Index_Selection.md) for choosing indexes based on types

---

*Last Updated: February 18, 2026*
