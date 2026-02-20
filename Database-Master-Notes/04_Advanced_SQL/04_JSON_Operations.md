# 🔗 JSON Operations - Working with Semi-Structured Data

> JSON operations enable storing, querying, and manipulating semi-structured data in relational databases. Master JSON functions for modern APIs, microservices, and flexible schemas.

---

## 📖 1. Concept Explanation

### What is JSON in Databases?

**JSON (JavaScript Object Notation)** is a lightweight data format stored in database columns for semi-structured data.

**JSON Types:**
```sql
-- PostgreSQL: json (text) vs jsonb (binary, optimized)
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    metadata JSON,        -- Stored as text, exact format preserved
    attributes JSONB      -- Binary format, faster queries, no whitespace
);

-- MySQL: JSON (single type, binary-like storage since 5.7)
CREATE TABLE products (
    id INT PRIMARY KEY,
    metadata JSON
);

-- SQL Server: NVARCHAR with JSON validation
CREATE TABLE products (
    id INT PRIMARY KEY,
    metadata NVARCHAR(MAX) CHECK (ISJSON(metadata) = 1)
);
```

**JSON vs Relational:**
```
RELATIONAL              JSON
│                       │
├── Fixed schema ✅     ├── Flexible schema ✅
├── Typed columns ✅    ├── Dynamic types ✅
├── Indexed easily ✅   ├── Indexed (GIN/virtual) ⚠️
├── Query fast ✅       ├── Query slower ⚠️
└── Joins efficient ✅  └── Nested data easy ✅
```

---

## 🧠 2. Why It Matters in Real Systems

### Without JSON: EAV Anti-Pattern

**❌ Bad: Entity-Attribute-Value (EAV) for dynamic attributes**
```sql
-- EAV table (anti-pattern for product attributes)
CREATE TABLE product_attributes (
    product_id INT,
    attribute_name VARCHAR(50),
    attribute_value VARCHAR(255)
);

-- Query: Get product with color and size
SELECT 
    p.name,
    MAX(CASE WHEN pa.attribute_name = 'color' THEN pa.attribute_value END) as color,
    MAX(CASE WHEN pa.attribute_name = 'size' THEN pa.attribute_value END) as size
FROM products p
LEFT JOIN product_attributes pa ON p.id = pa.product_id
WHERE p.id = 123
GROUP BY p.name;

-- Problems:
-- - Complex queries (multiple joins, CASE expressions) ❌
-- - No type safety (everything is VARCHAR) ❌  
-- - Difficult to index ❌
-- - Schema unclear ❌
```

**✅ Good: JSON column for variable attributes**
```sql
-- JSON column
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    attributes JSONB
);

INSERT INTO products VALUES (
    123, 
    'T-Shirt',
    '{"color": "blue", "size": "M", "material": "cotton"}'
);

-- Simple query
SELECT 
    name,
    attributes->>'color' as color,
    attributes->>'size' as size
FROM products
WHERE id = 123;

-- Benefits:
-- - Single row per product ✅
-- - Clear structure ✅
-- - Easy to add new attributes ✅
-- - Type validation in application ✅
```

---

## ⚙️ 3. Internal Working

### PostgreSQL: json vs jsonb

```sql
-- json: Text storage (exact input preserved)
-- - Stores raw text
-- - Preserves whitespace, key order
-- - Parses on every access (slower)
-- - Use for: Write-once, read-rarely data

-- jsonb: Binary storage (optimized)
-- - Decomposes into binary structure
-- - Removes whitespace, sorts keys
-- - Pre-parsed (faster queries)
-- - Supports indexing (GIN index)
-- - Use for: Frequent queries ✅

-- Benchmark (1M rows):
-- json parsing: ~5s per query
-- jsonb with GIN index: ~0.3s ✅ (16x faster)
```

### GIN Index for JSONB

```sql
-- Create GIN (Generalized Inverted Index)
CREATE INDEX idx_attributes_gin ON products USING GIN (attributes);

-- What it indexes:
-- - All keys and values in JSON
-- - Supports: @>, ?, ?&, ?| operators
-- - Fast for: Key existence, containment checks

-- Query using index:
SELECT * FROM products
WHERE attributes @> '{"color": "blue"}';  -- Uses GIN index ✅

-- Time: 0.3s (with index) vs 8s (sequential scan)
```

---

## ✅ 4. Best Practices

### Use JSONB Over JSON (PostgreSQL)

```sql
-- ✅ Use jsonb for queryable data
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_data JSONB  -- ✅ Fast queries
);

-- ❌ Avoid json for frequent queries
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_data JSON  -- Slower, no index support
);
```

### Validate JSON Structure

```sql
-- PostgreSQL: Use CHECK constraint
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    metadata JSONB,
    CONSTRAINT valid_metadata CHECK (
        jsonb_typeof(metadata->'price') = 'number' AND
        jsonb_typeof(metadata->'name') = 'string'
    )
);

-- MySQL: JSON_SCHEMA_VALID (8.0.17+)
ALTER TABLE products ADD CONSTRAINT check_metadata
CHECK (JSON_SCHEMA_VALID('{
    "type": "object",
    "properties": {
        "price": {"type": "number"},
        "name": {"type": "string"}
    },
    "required": ["name", "price"]
}', metadata));

-- SQL Server: Use CHECK with ISJSON
ALTER TABLE products ADD CONSTRAINT check_metadata
CHECK (ISJSON(metadata) = 1);
```

### Index Frequently Queried Keys

```sql
-- PostgreSQL: Expression index for specific key
CREATE INDEX idx_price ON products ((attributes->>'price'));

-- Or functional index for numeric operations
CREATE INDEX idx_price_numeric ON products (((attributes->>'price')::NUMERIC));

-- Now fast queries:
SELECT * FROM products
WHERE (attributes->>'price')::NUMERIC < 100;  -- Uses index ✅
```

### Avoid Over-Using JSON

```sql
-- ❌ Don't use JSON for frequently filtered fields
CREATE TABLE users (
    id INT PRIMARY KEY,
    data JSON  -- Contains email, name, age ❌
);

-- Slow query (must parse JSON):
SELECT * FROM users WHERE data->>'email' = 'user@example.com';

-- ✅ Use dedicated columns for common queries
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255),  -- ✅ Indexed, fast
    name VARCHAR(100),
    age INT,
    extra_data JSONB  -- ✅ Only for rarely queried fields
);

CREATE INDEX idx_email ON users(email);  -- Fast lookups ✅
```

---

## ❌ 5. Common Mistakes

### Mistake 1: json vs jsonb Confusion

```sql
-- ❌ Using json for queries (PostgreSQL)
CREATE TABLE products (metadata JSON);

SELECT * FROM products WHERE metadata->>'color' = 'blue';
-- Slow: Parses JSON on every access, can't use GIN index

-- ✅ Use jsonb for queries
CREATE TABLE products (metadata JSONB);
CREATE INDEX idx_metadata_gin ON products USING GIN (metadata);

SELECT * FROM products WHERE metadata @> '{"color": "blue"}';
-- Fast: Uses GIN index ✅
```

### Mistake 2: Wrong Operator (-> vs ->>)

```sql
-- PostgreSQL

-- -> returns JSON type
SELECT metadata->'price' FROM products;
-- Returns: "19.99" (JSON text, with quotes)

-- ->> returns TEXT type
SELECT metadata->>'price' FROM products;
-- Returns: 19.99 (plain text, no quotes)

-- ❌ Comparing JSON to text fails
WHERE metadata->'color' = 'blue'  -- ❌ Doesn't match

-- ✅ Use ->> for text comparison
WHERE metadata->>'color' = 'blue'  -- ✅ Works

-- ✅ Use -> for JSON comparison
WHERE metadata->'color' = '"blue"'  -- ✅ Works (note quotes)

-- ✅ Or use containment
WHERE metadata @> '{"color": "blue"}'  -- ✅ Best for JSONB
```

### Mistake 3: Not Casting Numeric Values

```sql
-- ❌ Comparing text to number
SELECT * FROM products
WHERE metadata->>'price' > 100;  -- ❌ Text comparison: "9" > "100"

-- ✅ Cast to numeric
SELECT * FROM products
WHERE (metadata->>'price')::NUMERIC > 100;  -- ✅ Numeric comparison
```

### Mistake 4: Deep Nesting Without Indexes

```sql
-- ❌ Deep path query without index
SELECT * FROM orders
WHERE metadata->'customer'->'address'->'city' = '"New York"';
-- Slow: Full table scan

-- ✅ Create expression index
CREATE INDEX idx_customer_city ON orders (
    (metadata->'customer'->'address'->>'city')
);

-- Now fast ✅
```

---

## 🔐 6. Security Considerations

### Prevent JSON Injection

```sql
-- ❌ Concatenating user input into JSON
DECLARE @userInput VARCHAR(50) = '", "role": "admin"}';
INSERT INTO users (metadata) VALUES ('{"name": "' + @userInput + '"}');
-- Result: {"name": "", "role": "admin"} -- Privilege escalation! ❌

-- ✅ Use parameterized JSON functions
DECLARE @userInput VARCHAR(50) = '", "role": "admin"}';
INSERT INTO users (metadata)
SELECT JSON_OBJECT('name', @userInput);  -- Escapes properly ✅
-- Result: {"name": "\", \"role\": \"admin\"}"} -- Safe ✅
```

### Validate Before Storing

```sql
-- PostgreSQL
CREATE FUNCTION validate_user_json(data JSONB)
RETURNS BOOLEAN AS $$
BEGIN
    -- Check required fields
    IF NOT (data ? 'name' AND data ? 'email') THEN
        RETURN FALSE;
    END IF;
    
    -- Validate email format
    IF NOT (data->>'email' ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$') THEN
        RETURN FALSE;
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

ALTER TABLE users ADD CONSTRAINT valid_metadata
CHECK (validate_user_json(metadata));
```

---

## 🚀 7. Performance Optimization

### GIN Index vs Expression Index

```sql
-- GIN index: All keys and values (large, general)
CREATE INDEX idx_metadata_gin ON products USING GIN (metadata);
-- Use for: Containment (@>), key existence (?)
-- Size: ~40% of table size
-- Query: WHERE metadata @> '{"color": "blue"}'

-- Expression index: Specific key (small, targeted)
CREATE INDEX idx_color ON products ((metadata->>'color'));
-- Use for: Equality on specific key
-- Size: ~5% of table size
-- Query: WHERE metadata->>'color' = 'blue'

-- Benchmark (1M rows, 10 keys per JSON):
-- GIN index query: 0.5s
-- Expression index query: 0.2s ✅ (2.5x faster for specific key)
-- No index: 12s
```

### Avoid SELECT *  with JSON

```sql
-- ❌ Large JSON in SELECT *
SELECT * FROM products;  -- Transfers MB of JSON data ❌

-- ✅ Extract only needed fields
SELECT 
    id,
    name,
    metadata->>'color' as color,
    metadata->>'price' as price
FROM products;  -- Transfers KB ✅
```

### Materialized Computed Columns

```sql
-- PostgreSQL: Generated column (12+)
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    metadata JSONB,
    price NUMERIC GENERATED ALWAYS AS ((metadata->>'price')::NUMERIC) STORED
);

CREATE INDEX idx_price ON products(price);  -- Fast queries ✅

-- Query using generated column
SELECT * FROM products WHERE price < 100;  -- Uses idx_price ✅

-- SQL Server: Persisted computed column
ALTER TABLE products ADD price AS JSON_VALUE(metadata, '$.price') PERSISTED;
CREATE INDEX idx_price ON products(price);
```

---

## 🧪 8. Examples

### PostgreSQL: Extracting JSON Data

```sql
-- Sample data
INSERT INTO products (name, attributes) VALUES
('T-Shirt', '{"color": "blue", "size": "M", "price": 19.99}'),
('Jeans', '{"color": "black", "size": "L", "price": 49.99}');

-- Extract scalar values
SELECT 
    name,
    attributes->>'color' as color,        -- Text extraction
    (attributes->>'price')::NUMERIC as price  -- Numeric extraction
FROM products;

-- Extract nested values
-- JSON: {"customer": {"name": "John", "address": {"city": "NYC"}}}
SELECT 
    order_id,
    metadata->'customer'->>'name' as customer_name,
    metadata->'customer'->'address'->>'city' as city
FROM orders;
```

### PostgreSQL: JSON Array Operations

```sql
-- Sample data with array
INSERT INTO products (name, attributes) VALUES
('Camera', '{"features": ["wifi", "4K", "waterproof"], "price": 299}');

-- Check if array contains value
SELECT name
FROM products
WHERE attributes->'features' ? 'wifi';  -- ✅ Has "wifi"

-- Get array length
SELECT 
    name,
    jsonb_array_length(attributes->'features') as feature_count
FROM products;

-- Expand array to rows
SELECT 
    name,
    jsonb_array_elements_text(attributes->'features') as feature
FROM products;

-- Output:
-- name   | feature
-- Camera | wifi
-- Camera | 4K
-- Camera | waterproof
```

### PostgreSQL: Updating JSON

```sql
-- Update specific key
UPDATE products
SET attributes = jsonb_set(attributes, '{price}', '29.99')
WHERE id = 1;

-- Add new key
UPDATE products
SET attributes = attributes || '{"discount": 10}'
WHERE id = 1;

-- Remove key
UPDATE products
SET attributes = attributes - 'discount'
WHERE id = 1;

-- Update nested value
UPDATE products
SET attributes = jsonb_set(
    attributes,
    '{customer, address, city}',  -- Path as array
    '"Los Angeles"'
)
WHERE id = 1;
```

### PostgreSQL: JSON Aggregation

```sql
-- Aggregate rows into JSON array
SELECT 
    category,
    jsonb_agg(
        jsonb_build_object(
            'name', name,
            'price', price
        )
    ) as products
FROM products
GROUP BY category;

-- Output:
-- category    | products
-- Electronics | [{"name": "Phone", "price": 599}, {"name": "Laptop", "price": 1299}]

-- Aggregate into JSON object
SELECT jsonb_object_agg(name, price) as price_map
FROM products;

-- Output: {"Phone": 599, "Laptop": 1299, "Tablet": 399}
```

### MySQL: JSON Functions

```sql
-- Extract values
SELECT 
    name,
    JSON_UNQUOTE(JSON_EXTRACT(metadata, '$.color')) as color,
    JSON_EXTRACT(metadata, '$.price') as price
FROM products;

-- Shorthand (MySQL 5.7.9+)
SELECT 
    name,
    metadata->>'$.color' as color,  -- ->> unquotes automatically
    metadata->'$.price' as price     -- -> returns JSON
FROM products;

-- Check if key exists
SELECT name
FROM products
WHERE JSON_CONTAINS_PATH(metadata, 'one', '$.color');

-- Array contains value
SELECT name
FROM products
WHERE JSON_CONTAINS(metadata->'$.features', '"wifi"');
```

### SQL Server: JSON Functions

```sql
-- Extract values
SELECT 
    name,
    JSON_VALUE(metadata, '$.color') as color,
    JSON_VALUE(metadata, '$.price') as price
FROM products;

-- Parse JSON array to table
SELECT *
FROM OPENJSON('["wifi", "4K", "waterproof"]')
WITH (feature VARCHAR(50) '$');

-- Output:
-- feature
-- wifi
-- 4K
-- waterproof

-- Convert JSON to table
SELECT *
FROM OPENJSON(N'[
    {"name": "Phone", "price": 599},
    {"name": "Laptop", "price": 1299}
]')
WITH (
    name VARCHAR(50) '$.name',
    price INT '$.price'
);
```

### SQL Server: Building JSON

```sql
-- Row to JSON
SELECT 
    id,
    name,
    price,
    (
        SELECT color, size, material
        FROM product_attributes pa
        WHERE pa.product_id = p.id
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
    ) as attributes
FROM products p
FOR JSON PATH;

-- Output:
-- [
--   {"id": 1, "name": "T-Shirt", "price": 19.99, 
--    "attributes": {"color": "blue", "size": "M", "material": "cotton"}}
-- ]
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: E-commerce - Product Catalog

```sql
-- Variable product attributes per category
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    category VARCHAR(50),
    base_price NUMERIC(10,2),
    attributes JSONB,  -- Variable by category
    created_at TIMESTAMP DEFAULT NOW()
);

-- Electronics: brand, warranty, specs
INSERT INTO products VALUES (
    1, 
    'iPhone 15 Pro',
    'Electronics',
    999.00,
    '{
        "brand": "Apple",
        "storage": "256GB",
        "color": "Titanium",
        "warranty_months": 12,
        "specs": {
            "screen": "6.1 inch",
            "processor": "A17 Pro",
            "camera": "48MP"
        }
    }'
);

-- Clothing: size, material, care
INSERT INTO products VALUES (
    2,
    'Cotton T-Shirt',
    'Clothing',
    19.99,
    '{
        "size": "M",
        "color": "Navy",
        "material": "100% Cotton",
        "care": ["Machine wash", "Tumble dry low"]
    }'
);

-- Query: Find electronics with 256GB storage
SELECT name, base_price
FROM products
WHERE category = 'Electronics'
  AND attributes @> '{"storage": "256GB"}';

-- Index for performance
CREATE INDEX idx_attributes_gin ON products USING GIN (attributes);
```

### Case Study 2: Logging/Analytics - Event Tracking

```sql
-- Store variable event data
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR(50),
    user_id INT,
    event_data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Different events have different data
INSERT INTO events VALUES
(1, 'page_view', 123, '{"page": "/products", "referrer": "google.com"}', NOW()),
(2, 'purchase', 123, '{"order_id": 456, "amount": 99.99, "items": ["SKU123", "SKU456"]}', NOW()),
(3, 'signup', 789, '{"email": "user@example.com", "source": "facebook"}', NOW());

-- Query: Users who purchased > $50
SELECT DISTINCT user_id
FROM events
WHERE event_type = 'purchase'
  AND (event_data->>'amount')::NUMERIC > 50;

-- Query: Top referrers for page views
SELECT 
    event_data->>'referrer' as referrer,
    COUNT(*) as views
FROM events
WHERE event_type = 'page_view'
GROUP BY event_data->>'referrer'
ORDER BY views DESC
LIMIT 10;
```

### Case Study 3: SaaS - User Preferences/Settings

```sql
-- Store user-specific settings
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    preferences JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Different users have different preferences
INSERT INTO users VALUES
(1, 'alice@example.com', '{
    "theme": "dark",
    "notifications": {
        "email": true,
        "push": false,
        "sms": false
    },
    "language": "en",
    "timezone": "America/New_York"
}'),
(2, 'bob@example.com', '{
    "theme": "light",
    "notifications": {
        "email": false,
        "push": true
    },
    "language": "es"
}');

-- Query: Users with email notifications enabled
SELECT email
FROM users
WHERE preferences->'notifications'->>'email' = 'true';

-- Update preferences (add new key)
UPDATE users
SET preferences = preferences || '{"beta_features": true}'
WHERE id = 1;

-- Update nested preference
UPDATE users
SET preferences = jsonb_set(
    preferences,
    '{notifications, push}',
    'true'
)
WHERE id = 2;
```

### Case Study 4: Microservices - API Response Caching

```sql
-- Cache external API responses
CREATE TABLE api_cache (
    id SERIAL PRIMARY KEY,
    api_endpoint VARCHAR(255),
    request_params JSONB,
    response_data JSONB,
    cached_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP
);

-- Cache weather API response
INSERT INTO api_cache VALUES (
    1,
    '/weather',
    '{"city": "New York", "units": "metric"}',
    '{
        "temp": 22,
        "humidity": 65,
        "conditions": "Partly Cloudy",
        "forecast": [
            {"day": "Mon", "high": 24, "low": 18},
            {"day": "Tue", "high": 26, "low": 20}
        ]
    }',
    NOW(),
    NOW() + INTERVAL '1 hour'
);

-- Query cache
SELECT response_data
FROM api_cache
WHERE api_endpoint = '/weather'
  AND request_params @> '{"city": "New York"}'
  AND expires_at > NOW();

-- Index for fast cache lookups
CREATE INDEX idx_cache_lookup ON api_cache (api_endpoint, request_params);
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: When should you use JSON columns vs normalized tables?

**Answer:**

**Use JSON when:**
- ✅ Schema is **variable/dynamic** (product attributes differ by category)
- ✅ Data is **rarely queried** individually (logging, audit trails)
- ✅ Schema changes **frequently** (feature flags, user preferences)
- ✅ Data is **opaque** to database (API responses, third-party data)

**Use normalized tables when:**
- ✅ Data is **frequently filtered/joined** (user emails, order IDs)
- ✅ Schema is **stable** (customers, orders, products)
- ✅ **Referential integrity** needed (foreign keys)
- ✅ **Complex aggregations** required (SUM, AVG across columns)

**Comparison:**
```sql
-- ❌ Frequently queried fields in JSON
CREATE TABLE users (
    id INT PRIMARY KEY,
    data JSON  -- {"email": "...", "name": "..."}
);
SELECT * FROM users WHERE data->>'email' = 'user@example.com';  -- Slow

-- ✅ Dedicated columns + JSON for extras
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255),  -- Indexed, fast ✅
    name VARCHAR(100),
    preferences JSON     -- Variable, rarely queried ✅
);
SELECT * FROM users WHERE email = 'user@example.com';  -- Fast ✅
```

---

### Q2: Explain the difference between json and jsonb in PostgreSQL.

**Answer:**

| Feature | json | jsonb |
|---------|------|-------|
| Storage | Text (raw JSON string) | Binary (decomposed structure) |
| Whitespace | Preserved | Removed |
| Key Order | Preserved | Sorted alphabetically |
| Duplicate Keys | Kept | Last value wins |
| Processing | Parses on every access | Pre-parsed |
| Indexing | Not supported | GIN index supported ✅ |
| Query Speed | Slower | Faster ✅ |
| Write Speed | Slightly faster | Slightly slower (decomposition overhead) |
| Use Case | Write-once, rarely read | Frequent queries ✅ |

**Example:**
```sql
-- json: Exact input preserved
INSERT INTO table1 (data_json) VALUES ('  {"b": 2,  "a": 1}  ');
SELECT data_json FROM table1;
-- Output: '  {"b": 2,  "a": 1}  ' (whitespace, key order preserved)

-- jsonb: Normalized
INSERT INTO table2 (data_jsonb) VALUES ('  {"b": 2,  "a": 1}  ');
SELECT data_jsonb FROM table2;
-- Output: '{"a": 1, "b": 2}' (no whitespace, keys sorted)

-- Duplicate keys
INSERT INTO table1 (data_json) VALUES ('{"a": 1, "a": 2}');
SELECT data_json FROM table1;  -- {"a": 1, "a": 2} (both kept)

INSERT INTO table2 (data_jsonb) VALUES ('{"a": 1, "a": 2}');
SELECT data_jsonb FROM table2;  -- {"a": 2} (last value wins)
```

**Recommendation:** Use **jsonb** for 99% of cases (query performance matters).

---

### Q3: How do you index JSON data for fast queries?

**Answer:**

**1. GIN Index (PostgreSQL jsonb):**
```sql
-- Index all keys and values
CREATE INDEX idx_metadata_gin ON products USING GIN (metadata);

-- Supports operators: @>, ?, ?&, ?|
SELECT * FROM products WHERE metadata @> '{"color": "blue"}';  -- Fast ✅
SELECT * FROM products WHERE metadata ? 'color';  -- Fast (key exists) ✅
```

**2. Expression Index (specific key):**
```sql
-- Index specific JSON key
CREATE INDEX idx_color ON products ((metadata->>'color'));

-- Fast queries on that key
SELECT * FROM products WHERE metadata->>'color' = 'blue';  -- Fast ✅
```

**3. Computed/Generated Column:**
```sql
-- PostgreSQL 12+
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    metadata JSONB,
    price NUMERIC GENERATED ALWAYS AS ((metadata->>'price')::NUMERIC) STORED
);

CREATE INDEX idx_price ON products(price);

SELECT * FROM products WHERE price < 100;  -- Fast ✅

-- SQL Server
ALTER TABLE products ADD price AS JSON_VALUE(metadata, '$.price') PERSISTED;
CREATE INDEX idx_price ON products(price);
```

**4. Virtual Column + Index (MySQL):**
```sql
-- Generated column
ALTER TABLE products ADD color VARCHAR(50) 
    AS (metadata->>'$.color') STORED;

CREATE INDEX idx_color ON products(color);

SELECT * FROM products WHERE color = 'blue';  -- Fast ✅
```

---

### Q4: What are the performance implications of using JSON?

**Answer:**

**Slower than normalized columns:**

| Operation | Normalized Column | JSON (no index) | JSON (GIN index) |
|-----------|-------------------|-----------------|------------------|
| Simple SELECT | 0.1s | 2.5s (25x slower) | 0.5s (5x slower) |
| WHERE equality | 0.05s | 3.0s (60x slower) | 0.3s (6x slower) |
| Numeric comparison | 0.08s | 5.0s (62x slower) | 1.0s (12x slower) |
| JOIN | 0.2s | Not practical | Not practical |

**Why slower:**
1. **Parsing overhead**: JSON must be parsed on access
2. **Type conversion**: `->>'key'` returns text, needs casting for numbers
3. **No foreign keys**: Can't use efficient join algorithms
4. **Large data**: JSON columns can be MB per row
5. **Indexing limitations**: GIN indexes are larger and slower than B-tree

**Performance tips:**
- ✅ Use jsonb (not json) in PostgreSQL
- ✅ Index frequently queried keys (GIN or expression index)
- ✅ Extract hot columns to dedicated fields
- ✅ Avoid SELECT * (transfer less data)
- ✅ Use computed columns for numeric operations

---

### Q5: How do you validate JSON structure before inserting?

**Answer:**

**PostgreSQL:**
```sql
-- CHECK constraint with custom function
CREATE FUNCTION validate_product_json(data JSONB)
RETURNS BOOLEAN AS $$
BEGIN
    -- Check required keys
    IF NOT (data ? 'name' AND data ? 'price') THEN
        RAISE EXCEPTION 'Missing required fields';
    END IF;
    
    -- Validate types
    IF jsonb_typeof(data->'price') != 'number' THEN
        RAISE EXCEPTION 'Price must be a number';
    END IF;
    
    -- Validate value ranges
    IF (data->>'price')::NUMERIC < 0 THEN
        RAISE EXCEPTION 'Price cannot be negative';
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

ALTER TABLE products ADD CONSTRAINT valid_json
CHECK (validate_product_json(metadata));
```

**MySQL (8.0.17+):**
```sql
-- JSON Schema validation
ALTER TABLE products ADD CONSTRAINT check_schema
CHECK (JSON_SCHEMA_VALID('{
    "type": "object",
    "properties": {
        "name": {"type": "string", "minLength": 1},
        "price": {"type": "number", "minimum": 0}
    },
    "required": ["name", "price"]
}', metadata));
```

**SQL Server:**
```sql
-- Basic ISJSON check
ALTER TABLE products ADD CONSTRAINT valid_json
CHECK (ISJSON(metadata) = 1);

-- Advanced validation with function
CREATE FUNCTION dbo.ValidateProductJSON(@json NVARCHAR(MAX))
RETURNS BIT
AS
BEGIN
    IF JSON_VALUE(@json, '$.name') IS NULL
        OR JSON_VALUE(@json, '$.price') IS NULL
        RETURN 0;
    
    IF CAST(JSON_VALUE(@json, '$.price') AS DECIMAL) < 0
        RETURN 0;
    
    RETURN 1;
END;

ALTER TABLE products ADD CONSTRAINT valid_product
CHECK (dbo.ValidateProductJSON(metadata) = 1);
```

---

## 🧩 11. Design Patterns

### Pattern 1: Hybrid Schema (Critical + JSON)

```sql
-- Extract frequently queried fields, keep rest in JSON
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,  -- Frequently queried ✅
    category VARCHAR(50) NOT NULL,  -- Indexed ✅
    price NUMERIC(10,2) NOT NULL,  -- Numeric operations ✅
    attributes JSONB,  -- Variable attributes (color, size, material)
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_category_price ON products(category, price);
CREATE INDEX idx_attributes_gin ON products USING GIN (attributes);

-- Fast queries on dedicated columns
SELECT * FROM products WHERE category = 'Electronics' AND price < 1000;

-- Flexible queries on JSON
SELECT * FROM products WHERE attributes @> '{"color": "blue"}';
```

### Pattern 2: JSON Schema Evolution

```sql
-- Version field in JSON for schema migration
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR(50),
    data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- V1 schema
INSERT INTO events VALUES (
    1, 
    'user_login',
    '{"version": 1, "user_id": 123, "ip": "1.2.3.4"}',
    NOW()
);

-- V2 schema (added location)
INSERT INTO events VALUES (
    2,
    'user_login',
    '{"version": 2, "user_id": 456, "ip": "5.6.7.8", "location": {"city": "NYC", "country": "US"}}',
    NOW()
);

-- Query handles both versions
SELECT 
    data->>'user_id' as user_id,
    CASE 
        WHEN (data->>'version')::INT >= 2 
        THEN data->'location'->>'city'
        ELSE NULL
    END as city
FROM events
WHERE event_type = 'user_login';
```

### Pattern 3: JSON Aggregation for APIs

```sql
-- Single query returns nested JSON for API response
SELECT jsonb_build_object(
    'user', jsonb_build_object(
        'id', u.id,
        'name', u.name,
        'email', u.email
    ),
    'orders', (
        SELECT jsonb_agg(jsonb_build_object(
            'order_id', o.id,
            'total', o.total,
            'items', (
                SELECT jsonb_agg(jsonb_build_object(
                    'product_name', p.name,
                    'quantity', oi.quantity,
                    'price', oi.price
                ))
                FROM order_items oi
                JOIN products p ON oi.product_id = p.id
                WHERE oi.order_id = o.id
            )
        ))
        FROM orders o
        WHERE o.user_id = u.id
    )
) as response
FROM users u
WHERE u.id = 123;

-- Output: Complete nested JSON ready for API
```

---

## 📚 Summary

### Key Takeaways

1. **JSON Types**: PostgreSQL (json/jsonb), MySQL (JSON), SQL Server (NVARCHAR + validation)
2. **Use jsonb** in PostgreSQL (faster queries, indexing support)
3. **Operators**: `->` (JSON), `->>` (text), `@>` (contains), `?` (has key)
4. **Indexing**: GIN index (all keys), expression index (specific key), computed columns
5. **Performance**: JSON slower than dedicated columns, but flexible
6. **Use JSON for**: Variable schemas, rarely queried data, opaque data
7. **Use dedicated columns for**: Frequently filtered fields, foreign keys, aggregations
8. **Common mistakes**: Using json (not jsonb), wrong operators (-> vs ->>), no indexing
9. **Validation**: CHECK constraints, JSON schema, custom functions
10. **Real-world**: Product attributes, event logging, user preferences, API caching

**Quick Reference:**

```sql
-- PostgreSQL
metadata->>'key'                       -- Text extraction
(metadata->>'price')::NUMERIC          -- Type casting
metadata @> '{"color": "blue"}'        -- Containment check
metadata ? 'key'                       -- Key exists
jsonb_set(metadata, '{key}', 'value')  -- Update
CREATE INDEX ON table USING GIN (col); -- GIN index

-- MySQL
metadata->>'$.key'                     -- Text extraction
JSON_EXTRACT(metadata, '$.key')        -- Value extraction
JSON_CONTAINS(metadata, '"value"', '$.key')  -- Check contains

-- SQL Server
JSON_VALUE(metadata, '$.key')          -- Extract scalar
JSON_QUERY(metadata, '$.object')       -- Extract object/array
ISJSON(metadata)                       -- Validate JSON
FOR JSON PATH                          -- Convert to JSON
```

---

**Next:** [05_XML_Operations.md](05_XML_Operations.md) - XML data operations  
**Related:** [../02_Data_Modeling/04_Schema_Design_Patterns.md](../02_Data_Modeling/04_Schema_Design_Patterns.md) - Schema patterns

---

*Last Updated: February 20, 2026*
