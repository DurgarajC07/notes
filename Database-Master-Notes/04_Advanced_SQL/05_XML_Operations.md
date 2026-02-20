# 📄 XML Operations - Working with XML Data in Databases

> XML operations enable storing, querying, and manipulating hierarchical XML data in relational databases. Master XPath, XQuery, and XML transformation for legacy integrations and document processing.

---

## 📖 1. Concept Explanation

### What is XML in Databases?

**XML (eXtensible Markup Language)** is a hierarchical markup language stored in database columns for semi-structured document data.

**XML vs JSON:**
```
XML                            JSON
│                              │
├── Verbose (tags) ⚠️          ├── Concise ✅
├── Schema validation (XSD) ✅ ├── Schema optional
├── Attributes + elements ✅   ├── Key-value only
├── Order matters ✅           ├── Order irrelevant (objects)
├── Comments supported ✅      ├── No comments
├── Namespaces ✅              ├── No namespaces
└── Legacy systems ✅          └── Modern APIs ✅
```

**Database Support:**
```
XML SUPPORT
│
├── SQL Server: Native XML type ✅ (best support)
├── Oracle: XMLType ✅ (comprehensive)
├── PostgreSQL: XML type ✅ (basic)
├── MySQL: No native XML type ❌ (text only)
└── BigQuery: No XML support ❌
```

---

## 🧠 2. Why It Matters in Real Systems

### When XML is Required

**Use XML when:**
- ✅ **Legacy system integration** (SOAP APIs, old enterprise systems)
- ✅ **Document-centric data** (invoices, contracts, reports)
- ✅ **Schema validation critical** (healthcare HL7, financial SWIFT)
- ✅ **Order matters** (narrative documents, workflows)
- ✅ **Mixed content** (text with embedded markup like HTML)

**Use JSON instead when:**
- ✅ **Modern APIs** (REST, GraphQL)
- ✅ **Simple key-value data**
- ✅ **Performance matters** (JSON parsing faster)
- ✅ **Developer preference** (JSON easier to work with)

---

## ⚙️ 3. Internal Working

### SQL Server XML Storage

```sql
-- Typed XML (validated against XSD schema)
CREATE XML SCHEMA COLLECTION ProductSchema AS
'<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">
    <xs:element name="product">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="name" type="xs:string"/>
                <xs:element name="price" type="xs:decimal"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
</xs:schema>';

CREATE TABLE products (
    id INT PRIMARY KEY,
    data XML(ProductSchema)  -- Validated on insert
);

-- Untyped XML (no validation)
CREATE TABLE products (
    id INT PRIMARY KEY,
    data XML  -- Any XML accepted
);
```

### XML Index Types (SQL Server)

```sql
-- Primary XML index (required for selective indexes)
CREATE PRIMARY XML INDEX idx_xml_primary
ON products(data);

-- Secondary indexes (require primary):
-- 1. PATH index: Optimizes path expressions
CREATE XML INDEX idx_xml_path ON products(data)
USING XML INDEX idx_xml_primary FOR PATH;

-- 2. VALUE index: Optimizes value-based searches
CREATE XML INDEX idx_xml_value ON products(data)
USING XML INDEX idx_xml_primary FOR VALUE;

-- 3. PROPERTY index: Optimizes property retrieval
CREATE XML INDEX idx_xml_property ON products(data)
USING XML INDEX idx_xml_primary FOR PROPERTY;
```

---

## ✅ 4. Best Practices

### Validate XML with Schemas

```sql
-- SQL Server: Create schema collection
CREATE XML SCHEMA COLLECTION OrderSchema AS
'<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">
    <xs:element name="order">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="customerId" type="xs:int"/>
                <xs:element name="total" type="xs:decimal"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
</xs:schema>';

-- Use typed XML column
CREATE TABLE orders (
    id INT PRIMARY KEY,
    order_data XML(OrderSchema)  -- ✅ Validated
);

-- Invalid XML rejected:
INSERT INTO orders VALUES (1, '<order><customerId>abc</customerId></order>');
-- ERROR: xs:int expected, got 'abc'
```

### Index Frequently Queried Paths

```sql
-- ✅ Create primary + PATH index
CREATE PRIMARY XML INDEX idx_xml_primary ON products(data);
CREATE XML INDEX idx_xml_path ON products(data)
USING XML INDEX idx_xml_primary FOR PATH;

-- Now fast queries:
SELECT * FROM products
WHERE data.exist('/product[@category="electronics"]') = 1;
-- Uses idx_xml_path ✅
```

### Extract to Computed Columns for Performance

```sql
-- ❌ Slow: Extract XML in every query
SELECT 
    id,
    data.value('(/product/name)[1]', 'VARCHAR(100)') as name,
    data.value('(/product/price)[1]', 'DECIMAL(10,2)') as price
FROM products;

-- ✅ Fast: Computed columns + regular indexes
ALTER TABLE products ADD 
    name AS data.value('(/product/name)[1]', 'VARCHAR(100)') PERSISTED,
    price AS data.value('(/product/price)[1]', 'DECIMAL(10,2)') PERSISTED;

CREATE INDEX idx_name ON products(name);
CREATE INDEX idx_price ON products(price);

-- Now regular queries work:
SELECT * FROM products WHERE price < 100;  -- Fast ✅
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Not Using Indexes

```sql
-- ❌ No index: Sequential scan
SELECT * FROM products
WHERE data.exist('/product[@category="electronics"]') = 1;
-- Time: 15s on 1M rows ❌

-- ✅ With XML indexes:
CREATE PRIMARY XML INDEX idx_xml_primary ON products(data);
CREATE XML INDEX idx_xml_path ON products(data)
USING XML INDEX idx_xml_primary FOR PATH;
-- Time: 0.5s ✅ (30x faster)
```

### Mistake 2: Wrong XPath Syntax

```sql
-- ❌ Missing [1] for singleton
SELECT data.value('/product/name', 'VARCHAR(100)') FROM products;
-- ERROR: Singleton value required, XPath expression returns multiple nodes

-- ✅ Use [1] to get first match
SELECT data.value('(/product/name)[1]', 'VARCHAR(100)') FROM products;

-- ✅ Or use query() for multiple
SELECT data.query('/product/name') FROM products;
```

### Mistake 3: Namespace Handling

```sql
-- XML with namespace:
-- <prod:product xmlns:prod="http://example.com/products">
--   <prod:name>Phone</prod:name>
-- </prod:product>

-- ❌ Forgetting namespace
SELECT data.value('(/product/name)[1]', 'VARCHAR(100)') FROM products;
-- Returns NULL ❌

-- ✅ Declare namespace
SELECT data.value(
    'declare namespace prod="http://example.com/products";
     (/prod:product/prod:name)[1]', 
    'VARCHAR(100)'
) FROM products;
```

---

## 🔐 6. Security Considerations

### Prevent XML Injection

```sql
-- ❌ Concatenating user input
DECLARE @userInput VARCHAR(100) = '</name><admin>true</admin><name>';
DECLARE @xml VARCHAR(MAX) = '<product><name>' + @userInput + '</name></product>';
-- Result: <product><name></name><admin>true</admin><name></name></product>
-- Privilege escalation! ❌

-- ✅ Use parameterized construction
DECLARE @userInput VARCHAR(100) = '</name><admin>true</admin><name>';
DECLARE @xml XML;
SET @xml = (
    SELECT @userInput as 'name'
    FOR XML PATH('product'), TYPE
);
-- Result: <product><name>&lt;/name&gt;&lt;admin&gt;true&lt;/admin&gt;&lt;name&gt;</name></product>
-- Input escaped properly ✅
```

### Validate Against Schema

```sql
-- ✅ Schema prevents malicious XML structure
CREATE XML SCHEMA COLLECTION UserSchema AS
'<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">
    <xs:element name="user">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="name" type="xs:string"/>
                <xs:element name="email" type="xs:string"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
</xs:schema>';

CREATE TABLE users (data XML(UserSchema));

-- Rejects unexpected elements:
INSERT INTO users VALUES ('<user><name>Alice</name><admin>true</admin></user>');
-- ERROR: Element 'admin' not defined in schema
```

---

## 🚀 7. Performance Optimization

### XML vs JSON Performance

```sql
-- Benchmark (1M rows):

-- XML parsing (SQL Server):
SELECT AVG(CAST(data.value('(/product/price)[1]', 'VARCHAR(20)') AS DECIMAL))
FROM products_xml;
-- Time: 25s ❌

-- JSON parsing (SQL Server):
SELECT AVG(CAST(JSON_VALUE(data, '$.price') AS DECIMAL))
FROM products_json;
-- Time: 8s ✅ (3x faster)

-- Normalized column:
SELECT AVG(price) FROM products;
-- Time: 0.5s ✅✅ (50x faster than XML)
```

**Recommendation:** Use JSON for new projects, XML only when required.

---

## 🧪 8. Examples

### SQL Server: Basic XML Operations

```sql
-- Insert XML
INSERT INTO products VALUES (
    1,
    '<product>
        <name>Laptop</name>
        <price>1299.99</price>
        <category>Electronics</category>
    </product>'
);

-- Extract single value
SELECT 
    id,
    data.value('(/product/name)[1]', 'VARCHAR(100)') as name,
    data.value('(/product/price)[1]', 'DECIMAL(10,2)') as price
FROM products;

-- Check existence
SELECT id
FROM products
WHERE data.exist('/product[category="Electronics"]') = 1;

-- Query (returns XML fragment)
SELECT data.query('/product/name') FROM products;
-- Output: <name>Laptop</name>
```

### SQL Server: XQuery with FLWOR

```sql
-- FLWOR: For-Let-Where-Order-Return
SELECT data.query('
    for $p in /catalog/product
    let $discount := $p/price * 0.1
    where $p/price > 100
    order by $p/price descending
    return <item>
        <name>{data($p/name)}</name>
        <original>{data($p/price)}</original>
        <discounted>{data($p/price) - $discount}</discounted>
    </item>
') as discounted_products
FROM catalogs;
```

### SQL Server: Modify XML

```sql
-- Insert new element
UPDATE products
SET data.modify('
    insert <stock>50</stock>
    as last into (/product)[1]
')
WHERE id = 1;

-- Update element value
UPDATE products
SET data.modify('
    replace value of (/product/price/text())[1]
    with 999.99
')
WHERE id = 1;

-- Delete element
UPDATE products
SET data.modify('
    delete /product/discontinued
')
WHERE id = 1;
```

### SQL Server: XML to Table (Shredding)

```sql
-- XML data
DECLARE @xml XML = '
<orders>
    <order id="1">
        <customer>Alice</customer>
        <total>299.99</total>
    </order>
    <order id="2">
        <customer>Bob</customer>
        <total>149.99</total>
    </order>
</orders>';

-- Shred to relational rows
SELECT
    T.c.value('@id', 'INT') as order_id,
    T.c.value('(customer)[1]', 'VARCHAR(100)') as customer,
    T.c.value('(total)[1]', 'DECIMAL(10,2)') as total
FROM @xml.nodes('/orders/order') AS T(c);

-- Output:
-- order_id | customer | total
-- 1        | Alice    | 299.99
-- 2        | Bob      | 149.99
```

### SQL Server: Table to XML (FOR XML)

```sql
-- RAW mode: Generic row/column structure
SELECT id, name, price
FROM products
FOR XML RAW;
-- Output: <row id="1" name="Laptop" price="1299.99"/>

-- AUTO mode: Infers structure from table names
SELECT p.name, p.price, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.id
FOR XML AUTO;
-- Output: <p name="Laptop" price="1299.99"><c category_name="Electronics"/></p>

-- PATH mode: Custom structure
SELECT 
    name as 'product/name',
    price as 'product/price',
    category as 'product/@category'
FROM products
FOR XML PATH('item'), ROOT('catalog');
-- Output:
-- <catalog>
--   <item>
--     <product category="Electronics">
--       <name>Laptop</name>
--       <price>1299.99</price>
--     </product>
--   </item>
-- </catalog>
```

### PostgreSQL: Basic XML

```sql
-- PostgreSQL has limited XML support
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    data XML
);

INSERT INTO products VALUES (
    1,
    '<product><name>Laptop</name><price>1299</price></product>'
);

-- Extract using xpath()
SELECT 
    id,
    (xpath('/product/name/text()', data))[1]::text as name,
    (xpath('/product/price/text()', data))[1]::text::numeric as price
FROM products;

-- xpath returns array, use [1] for first match
```

### Oracle: XMLType

```sql
-- Create table with XMLType
CREATE TABLE products (
    id NUMBER PRIMARY KEY,
    data XMLType
);

-- Insert XML
INSERT INTO products VALUES (
    1,
    XMLType('<product><name>Laptop</name><price>1299</price></product>')
);

-- Extract using XMLQuery
SELECT 
    id,
    XMLQuery('/product/name/text()' PASSING data RETURNING CONTENT).getStringVal() as name,
    XMLQuery('/product/price/text()' PASSING data RETURNING CONTENT).getNumberVal() as price
FROM products;

-- Or use XMLTable for shredding
SELECT x.*
FROM products p,
     XMLTable('/product'
              PASSING p.data
              COLUMNS 
                  name VARCHAR2(100) PATH 'name',
                  price NUMBER PATH 'price'
     ) x
WHERE p.id = 1;
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Healthcare - HL7 Message Processing

```sql
-- HL7 messages stored as XML
CREATE TABLE hl7_messages (
    id INT PRIMARY KEY,
    message_xml XML,
    received_at DATETIME DEFAULT GETDATE()
);

-- Sample HL7 ADT (Admit/Discharge/Transfer) message
INSERT INTO hl7_messages VALUES (
    1,
    '<HL7Message>
        <MSH>
            <SendingFacility>Hospital A</SendingFacility>
            <ReceivingFacility>Lab B</ReceivingFacility>
        </MSH>
        <PID>
            <PatientId>12345</PatientId>
            <PatientName>John Doe</PatientName>
            <DateOfBirth>1980-05-15</DateOfBirth>
        </PID>
        <PV1>
            <AdmitDateTime>2024-02-20T10:30:00</AdmitDateTime>
            <AttendingDoctor>Dr. Smith</AttendingDoctor>
        </PV1>
    </HL7Message>',
    GETDATE()
);

-- Extract patient data
SELECT 
    message_xml.value('(/HL7Message/PID/PatientId)[1]', 'VARCHAR(50)') as patient_id,
    message_xml.value('(/HL7Message/PID/PatientName)[1]', 'VARCHAR(100)') as patient_name,
    message_xml.value('(/HL7Message/PV1/AdmitDateTime)[1]', 'DATETIME') as admit_time
FROM hl7_messages;

-- Index for fast patient lookup
CREATE PRIMARY XML INDEX idx_hl7_primary ON hl7_messages(message_xml);
CREATE XML INDEX idx_hl7_path ON hl7_messages(message_xml)
USING XML INDEX idx_hl7_primary FOR PATH;
```

### Case Study 2: Finance - SWIFT Message Storage

```sql
-- SWIFT MT messages (wire transfers)
CREATE TABLE swift_messages (
    id INT PRIMARY KEY,
    message_xml XML,
    processed BIT DEFAULT 0
);

-- MT103 (Single Customer Credit Transfer)
INSERT INTO swift_messages VALUES (
    1,
    '<MT103>
        <BasicHeader>
            <SendingBank>CITIUS33XXX</SendingBank>
        </BasicHeader>
        <Field20>REF20240220001</Field20>
        <Field32A>
            <Date>240220</Date>
            <Currency>USD</Currency>
            <Amount>50000.00</Amount>
        </Field32A>
        <Field50K>
            <OrderingCustomer>John Smith</OrderingCustomer>
        </Field50K>
        <Field59>
            <BeneficiaryName>Jane Doe</BeneficiaryName>
            <BeneficiaryAccount>1234567890</BeneficiaryAccount>
        </Field59>
    </MT103>',
    0
);

-- Extract for processing
SELECT 
    id,
    message_xml.value('(/MT103/Field20)[1]', 'VARCHAR(50)') as reference,
    message_xml.value('(/MT103/Field32A/Currency)[1]', 'VARCHAR(3)') as currency,
    message_xml.value('(/MT103/Field32A/Amount)[1]', 'DECIMAL(18,2)') as amount,
    message_xml.value('(/MT103/Field59/BeneficiaryAccount)[1]', 'VARCHAR(50)') as beneficiary_account
FROM swift_messages
WHERE processed = 0;
```

### Case Study 3: E-commerce - Order Export/Import

```sql
-- Export orders to XML for partner integration
SELECT 
    o.order_id as '@id',
    o.order_date as '@date',
    c.email as 'customer/email',
    c.name as 'customer/name',
    (
        SELECT 
            p.name as 'product',
            oi.quantity as '@quantity',
            oi.price as '@price'
        FROM order_items oi
        JOIN products p ON oi.product_id = p.id
        WHERE oi.order_id = o.order_id
        FOR XML PATH('item'), TYPE
    ) as 'items'
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.order_date >= '2024-02-01'
FOR XML PATH('order'), ROOT('orders');

-- Output:
-- <orders>
--   <order id="1" date="2024-02-15">
--     <customer>
--       <email>alice@example.com</email>
--       <name>Alice Smith</name>
--     </customer>
--     <items>
--       <item quantity="2" price="19.99">
--         <product>T-Shirt</product>
--       </item>
--       <item quantity="1" price="49.99">
--         <product>Jeans</product>
--       </item>
--     </items>
--   </order>
-- </orders>
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: When would you use XML over JSON in a database?

**Answer:**

**Use XML when:**
1. **Legacy system integration** - SOAP APIs, older enterprise systems require XML
2. **Strict schema validation** - XSD schemas for healthcare (HL7), finance (SWIFT)
3. **Document-centric** - Contracts, invoices, reports with mixed content
4. **Order matters** - Sequential processing, narrative documents
5. **Attributes + elements** - Rich metadata (e.g., `<price currency="USD">100</price>`)
6. **Namespaces required** - Avoiding naming conflicts in complex schemas

**Use JSON instead when:**
1. **Modern APIs** - REST, GraphQL, microservices
2. **Performance critical** - JSON parsing 3-5x faster
3. **Developer experience** - JSON easier to work with
4. **Simple data** - Key-value structures

**Example comparison:**
```sql
-- XML (verbose)
<product>
    <name>Laptop</name>
    <price currency="USD">1299</99>
</product>

-- JSON (concise)
{"name": "Laptop", "price": 1299.99, "currency": "USD"}
```

---

### Q2: How do you query XML data efficiently in SQL Server?

**Answer:**

**1. Use .value() for scalar extraction:**
```sql
SELECT data.value('(/product/name)[1]', 'VARCHAR(100)') as name
FROM products;
```

**2. Use .exist() for filtering:**
```sql
SELECT * FROM products
WHERE data.exist('/product[@category="Electronics"]') = 1;
```

**3. Use .query() for XML fragments:**
```sql
SELECT data.query('/product/name') FROM products;
```

**4. Use .nodes() for shredding to rows:**
```sql
SELECT T.c.value('(name)[1]', 'VARCHAR(100)') as name
FROM products
CROSS APPLY data.nodes('/product') AS T(c);
```

**5. Create XML indexes:**
```sql
-- Primary index (required)
CREATE PRIMARY XML INDEX idx_xml_primary ON products(data);

-- Secondary PATH index (for path expressions)
CREATE XML INDEX idx_xml_path ON products(data)
USING XML INDEX idx_xml_primary FOR PATH;

-- Now 10-30x faster queries ✅
```

---

### Q3: What are XML indexes and when should you use them?

**Answer:**

**SQL Server XML indexes:**

**1. Primary XML index** (required first):
- Shreds XML into internal table
- Maps nodes, paths, values
- ~3x size of XML data
- Required for secondary indexes

**2. Secondary indexes:**

**PATH index** - Optimizes path expressions:
```sql
CREATE XML INDEX idx_path ON products(data)
USING XML INDEX idx_primary FOR PATH;

-- Fast queries:
WHERE data.exist('/product/name') = 1
```

**VALUE index** - Optimizes value searches:
```sql
CREATE XML INDEX idx_value ON products(data)
USING XML INDEX idx_primary FOR VALUE;

-- Fast queries:
WHERE data.exist('/product[price > 100]') = 1
```

**PROPERTY index** - Optimizes property bag retrieval:
```sql
CREATE XML INDEX idx_property ON products(data)
USING XML INDEX idx_primary FOR PROPERTY;

-- Fast queries extracting multiple values:
SELECT data.value('(/product/name)[1]', '...'),
       data.value('(/product/price)[1]', '...')
```

**When to use:**
- ✅ Large XML columns (>1KB per row)
- ✅ Frequent WHERE clauses on XML paths
- ✅ Tables with >10K rows
- ❌ Small tables (index overhead not worth it)
- ❌ Rarely queried XML

---

### Q4: How do you convert relational data to XML in SQL Server?

**Answer:**

**Use FOR XML with different modes:**

**1. RAW mode** - Generic structure:
```sql
SELECT id, name, price FROM products FOR XML RAW;
-- <row id="1" name="Laptop" price="1299"/>
```

**2. AUTO mode** - Infers from table structure:
```sql
SELECT p.name, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.id
FOR XML AUTO;
-- <p name="Laptop"><c category_name="Electronics"/></p>
```

**3. PATH mode** - Custom structure (most flexible):
```sql
SELECT 
    name as 'product/name',
    price as 'product/price',
    category as 'product/@category'  -- @ for attribute
FROM products
FOR XML PATH('item'), ROOT('catalog');

-- Output:
-- <catalog>
--   <item>
--     <product category="Electronics">
--       <name>Laptop</name>
--       <price>1299</price>
--     </product>
--   </item>
-- </catalog>
```

**4. EXPLICIT mode** - Full control (complex, rarely used)

**Recommendation:** Use **PATH mode** for maximum flexibility.

---

### Q5: How do XML namespaces work in SQL Server XQuery?

**Answer:**

**XML with namespace:**
```xml
<prod:product xmlns:prod="http://example.com/products">
    <prod:name>Laptop</prod:name>
    <prod:price>1299</prod:price>
</prod:product>
```

**Query requires namespace declaration:**
```sql
-- ❌ Without namespace (returns NULL)
SELECT data.value('(/product/name)[1]', 'VARCHAR(100)') FROM products;

-- ✅ With namespace declaration
SELECT data.value('
    declare namespace prod="http://example.com/products";
    (/prod:product/prod:name)[1]
', 'VARCHAR(100)') FROM products;

-- ✅ Use WITH XMLNAMESPACES for reuse
WITH XMLNAMESPACES('http://example.com/products' as prod)
SELECT 
    data.value('(/prod:product/prod:name)[1]', 'VARCHAR(100)') as name,
    data.value('(/prod:product/prod:price)[1]', 'DECIMAL(10,2)') as price
FROM products;
```

**Default namespace:**
```sql
-- XML with default namespace (no prefix)
-- <product xmlns="http://example.com/products">...</product>

-- Declare as default
WITH XMLNAMESPACES(DEFAULT 'http://example.com/products')
SELECT data.value('(/product/name)[1]', 'VARCHAR(100)') FROM products;
```

---

## 🧩 11. Design Patterns

### Pattern 1: Computed Columns for Performance

```sql
-- Extract frequently queried fields to computed columns
CREATE TABLE products (
    id INT PRIMARY KEY,
    data XML,
    name AS data.value('(/product/name)[1]', 'VARCHAR(100)') PERSISTED,
    price AS data.value('(/product/price)[1]', 'DECIMAL(10,2)') PERSISTED,
    category AS data.value('(/product/@category)[1]', 'VARCHAR(50)') PERSISTED
);

-- Create regular indexes
CREATE INDEX idx_category_price ON products(category, price);

-- Now fast queries without XML parsing:
SELECT * FROM products WHERE category = 'Electronics' AND price < 1000;
-- Uses B-tree index, not XML parsing ✅
```

### Pattern 2: XML Schema Evolution

```sql
-- Version attribute for schema changes
CREATE TABLE documents (
    id INT PRIMARY KEY,
    data XML
);

-- V1 schema
INSERT INTO documents VALUES (
    1,
    '<document version="1">
        <title>Report</title>
        <content>Data</content>
    </document>'
);

-- V2 schema (added author)
INSERT INTO documents VALUES (
    2,
    '<document version="2">
        <title>Report 2</title>
        <author>Alice</author>
        <content>Data</content>
    </document>'
);

-- Query handles both versions
SELECT 
    id,
    data.value('(/document/@version)[1]', 'INT') as version,
    data.value('(/document/title)[1]', 'VARCHAR(100)') as title,
    CASE 
        WHEN data.exist('/document/author') = 1 
        THEN data.value('(/document/author)[1]', 'VARCHAR(100)')
        ELSE 'Unknown'
    END as author
FROM documents;
```

### Pattern 3: Hybrid Storage (Normalized + XML)

```sql
-- Core fields normalized, extras in XML
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,  -- Frequently queried ✅
    category VARCHAR(50) NOT NULL,
    base_price DECIMAL(10,2) NOT NULL,
    specs XML,  -- Variable specs per category
    created_at DATETIME DEFAULT GETDATE()
);

-- Indexes on normalized fields
CREATE INDEX idx_category_price ON products(category, base_price);

-- XML for category-specific attributes
INSERT INTO products VALUES (
    1,
    'iPhone 15 Pro',
    'Electronics',
    999.00,
    '<specs>
        <storage>256GB</storage>
        <color>Titanium</color>
        <warranty>12 months</warranty>
    </specs>',
    GETDATE()
);

-- Fast queries on indexed fields + flexible XML
SELECT * FROM products
WHERE category = 'Electronics' AND base_price < 1000
  AND specs.exist('/specs[storage="256GB"]') = 1;
```

---

## 📚 Summary

### Key Takeaways

1. **XML in databases**: SQL Server (best), Oracle (good), PostgreSQL (basic), MySQL (no native support)
2. **Use XML for**: Legacy integrations, schema validation, document-centric data
3. **Prefer JSON** for modern applications (3-5x faster parsing)
4. **XPath/XQuery**: Query language for XML (like SQL for relational data)
5. **XML indexes**: Primary + secondary (PATH, VALUE, PROPERTY)
6. **Extraction**: .value() (scalar), .query() (fragment), .nodes() (shred to rows)
7. **Performance**: Extract hot fields to computed columns + B-tree indexes
8. **Common mistakes**: Missing [1], forgetting namespaces, no indexes
9. **Security**: Prevent XML injection with parameterized construction
10. **Real-world**: Healthcare (HL7), finance (SWIFT), legacy enterprise systems

**SQL Server Quick Reference:**

```sql
-- Extract scalar
data.value('(/product/name)[1]', 'VARCHAR(100)')

-- Check existence
data.exist('/product[@category="Electronics"]') = 1

-- Query fragment
data.query('/product/name')

-- Shred to rows
CROSS APPLY data.nodes('/products/product') AS T(c)

-- Modify
data.modify('replace value of (/product/price/text())[1] with 999')

-- Table to XML
SELECT * FROM products FOR XML PATH('product'), ROOT('catalog')

-- Indexes
CREATE PRIMARY XML INDEX idx_primary ON products(data);
CREATE XML INDEX idx_path ON products(data) USING XML INDEX idx_primary FOR PATH;
```

---

**Next:** [06_Full_Text_Search.md](06_Full_Text_Search.md) - Full-text search operations  
**Related:** [04_JSON_Operations.md](04_JSON_Operations.md) - JSON vs XML comparison

---

*Last Updated: February 20, 2026*
