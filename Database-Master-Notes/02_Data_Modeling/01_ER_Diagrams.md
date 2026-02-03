# Entity-Relationship (ER) Diagrams

## 1. Concept Explanation

### What are ER Diagrams?

Entity-Relationship (ER) diagrams are visual representations of database structures that model the relationships between entities (tables) in a system. Created by Peter Chen in 1976, ER diagrams serve as the blueprint for relational database design, bridging the gap between business requirements and physical database implementation.

**Core Components:**

```
┌─────────────────────────────────────────────────────────────┐
│                    ER DIAGRAM COMPONENTS                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ENTITIES (Rectangles)                                       │
│  ┌──────────┐                                               │
│  │  User    │  ← Represents a table/entity                  │
│  └──────────┘                                               │
│                                                               │
│  ATTRIBUTES (Ovals/Ellipses)                                │
│      ○ user_id                                              │
│      ○ email    ← Properties of entities                    │
│      ○ name                                                 │
│                                                               │
│  RELATIONSHIPS (Diamonds)                                    │
│        ◇ places                                             │
│     ← Connection between entities                           │
│                                                               │
│  PRIMARY KEYS (Underlined)                                  │
│      user_id  ← Unique identifier                           │
│     ─────────                                               │
│                                                               │
│  CARDINALITY (1, N, M notation)                             │
│  User ──1──◇──N── Order                                     │
│        One to Many relationship                              │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Three Levels of ER Modeling:**

1. **Conceptual ERD**: High-level, business-focused view (no technical details)
2. **Logical ERD**: Detailed entities, attributes, relationships (still database-agnostic)
3. **Physical ERD**: Implementation-specific with data types, indexes, constraints

### Cardinality Types

**One-to-One (1:1):** Each entity instance relates to exactly one instance of another

```
User ──1────1── UserProfile
One user has exactly one profile
```

**One-to-Many (1:N):** One entity instance relates to many instances of another

```
Customer ──1────N── Order
One customer has many orders
```

**Many-to-Many (M:N):** Multiple instances on both sides (requires junction table)

```
Student ──M────N── Course
Many students enroll in many courses
```

---

## 2. Why It Matters

### Business Impact

**Communication Tool:**

- **Stakeholder Alignment**: ER diagrams create a shared language between developers, business analysts, and domain experts
- **Requirement Validation**: Visual models reveal missing entities and relationships early in design
- **Documentation**: Living documentation that survives developer turnover

**Cost Savings:**

- **Early Error Detection**: Finding a missing relationship in an ER diagram costs $100; finding it in production costs $100,000
- **Faster Onboarding**: New developers understand the domain model in hours instead of weeks
- **Change Impact Analysis**: Quickly assess how schema changes affect the entire system

**Real Cost of Poor Data Modeling:**

```
┌─────────────────────────────────────────────────────────────┐
│              COST OF FIXING DATA MODEL ERRORS                │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Design Phase:        $100 - $1,000                         │
│  Development Phase:   $1,000 - $10,000                      │
│  Testing Phase:       $10,000 - $50,000                     │
│  Production Phase:    $50,000 - $1,000,000+                 │
│                                                               │
│  Migration Cost:      10x - 100x development cost           │
│  Data Loss Risk:      High if relationships are wrong       │
│  Downtime Impact:     Hours to days for major changes       │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Technical Impact

**Query Performance:** Proper entity modeling directly impacts query efficiency

- **Normalized structures** reduce data duplication but may require more JOINs
- **Relationship design** affects index usage and query plan generation
- **Cardinality understanding** enables accurate query optimization

**Data Integrity:** ER diagrams enforce business rules at the database level

- **Foreign key constraints** prevent orphaned records
- **Cardinality constraints** enforce business logic (e.g., one user = one profile)
- **Entity identification** ensures uniqueness and referential integrity

**Scalability:** Well-designed ER models scale better

- **Clear boundaries** enable horizontal partitioning
- **Relationship clarity** simplifies distributed database design
- **Entity independence** allows microservice decomposition

---

## 3. Internal Working & Implementation

### From ER Diagram to Physical Schema

**Step 1: Entity to Table Mapping**

```
ER Diagram Entity:
┌──────────┐
│   User   │
│──────────│
│ user_id  │ ← Primary Key
│ email    │
│ name     │
│ created  │
└──────────┘

↓ TRANSFORMS TO ↓

SQL Table:
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**Step 2: Relationship to Foreign Key Mapping**

```
ER Diagram Relationship:
User ──1────N── Order
(One user places many orders)

↓ TRANSFORMS TO ↓

SQL Implementation:
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,

    -- Foreign key implements the relationship
    CONSTRAINT fk_user
        FOREIGN KEY (user_id)
        REFERENCES users(user_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);

-- Index for relationship queries
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

**Step 3: Many-to-Many with Junction Tables**

```
ER Diagram:
Student ──M────N── Course
(Many students enroll in many courses)

↓ TRANSFORMS TO ↓

CREATE TABLE students (
    student_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE
);

CREATE TABLE courses (
    course_id UUID PRIMARY KEY,
    course_name VARCHAR(100) NOT NULL,
    credits INTEGER NOT NULL
);

-- Junction table implements M:N relationship
CREATE TABLE enrollments (
    enrollment_id UUID PRIMARY KEY,
    student_id UUID NOT NULL,
    course_id UUID NOT NULL,
    enrolled_date DATE NOT NULL,
    grade CHAR(2),

    CONSTRAINT fk_student
        FOREIGN KEY (student_id)
        REFERENCES students(student_id)
        ON DELETE CASCADE,

    CONSTRAINT fk_course
        FOREIGN KEY (course_id)
        REFERENCES courses(course_id)
        ON DELETE CASCADE,

    -- Prevent duplicate enrollments
    CONSTRAINT uk_student_course
        UNIQUE (student_id, course_id)
);

CREATE INDEX idx_enrollments_student ON enrollments(student_id);
CREATE INDEX idx_enrollments_course ON enrollments(course_id);
```

### Advanced ER Concepts

**Weak Entities:** Entities that cannot exist without a parent entity

```
ER Diagram:
Order ══1────N══ OrderItem
(Double line indicates weak entity)

Implementation:
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    order_date TIMESTAMP NOT NULL
);

CREATE TABLE order_items (
    -- Composite primary key includes parent's key
    order_id UUID NOT NULL,
    item_number INTEGER NOT NULL,
    product_id UUID NOT NULL,
    quantity INTEGER NOT NULL,
    price DECIMAL(10,2) NOT NULL,

    PRIMARY KEY (order_id, item_number),

    CONSTRAINT fk_order
        FOREIGN KEY (order_id)
        REFERENCES orders(order_id)
        ON DELETE CASCADE  -- Weak entity dies with parent
);
```

**Supertype/Subtype Relationships (Inheritance):**

```
ER Diagram:
        ┌──────────┐
        │  Person  │ ← Supertype
        └────┬─────┘
             │ IS-A
        ┌────┴────┬────────┐
        │         │        │
    ┌───┴───┐ ┌──┴───┐ ┌──┴─────┐
    │Employee│ │Customer│ │Vendor │ ← Subtypes
    └────────┘ └────────┘ └────────┘

Implementation Strategy 1: Table Per Type (TPT)
-- Normalized, no duplication
CREATE TABLE persons (
    person_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE
);

CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    person_id UUID NOT NULL UNIQUE,
    hire_date DATE NOT NULL,
    salary DECIMAL(10,2),
    FOREIGN KEY (person_id) REFERENCES persons(person_id)
);

CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    person_id UUID NOT NULL UNIQUE,
    account_number VARCHAR(50),
    credit_limit DECIMAL(10,2),
    FOREIGN KEY (person_id) REFERENCES persons(person_id)
);

-- Pros: Normalized, easy to add subtypes
-- Cons: Requires JOINs for queries

Implementation Strategy 2: Single Table Inheritance (STI)
-- Denormalized, one table for all types
CREATE TABLE persons (
    person_id UUID PRIMARY KEY,
    person_type VARCHAR(20) NOT NULL, -- 'employee', 'customer', 'vendor'
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,

    -- Employee-specific columns
    hire_date DATE,
    salary DECIMAL(10,2),

    -- Customer-specific columns
    account_number VARCHAR(50),
    credit_limit DECIMAL(10,2),

    -- Vendor-specific columns
    tax_id VARCHAR(50),
    payment_terms VARCHAR(50)
);

CREATE INDEX idx_person_type ON persons(person_type);

-- Pros: No JOINs, simpler queries
-- Cons: Sparse columns, harder to enforce constraints

Implementation Strategy 3: Concrete Table Inheritance
-- Separate tables, duplicated columns
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,  -- Duplicated
    email VARCHAR(255) UNIQUE,   -- Duplicated
    hire_date DATE NOT NULL,
    salary DECIMAL(10,2)
);

CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,  -- Duplicated
    email VARCHAR(255) UNIQUE,   -- Duplicated
    account_number VARCHAR(50),
    credit_limit DECIMAL(10,2)
);

-- Pros: No JOINs, type-specific constraints
-- Cons: Data duplication, hard to query across types
```

---

## 4. Best Practices

### 1. Entity Naming Conventions

**✅ GOOD:**

```
-- Use singular nouns
CREATE TABLE user (...);
CREATE TABLE product (...);
CREATE TABLE order_item (...);

-- Clear, descriptive names
CREATE TABLE customer_subscription (...);
CREATE TABLE payment_method (...);
```

**❌ BAD:**

```
-- Plural names (avoid, though some teams prefer this)
CREATE TABLE users (...);  -- Can be confusing in code

-- Abbreviations
CREATE TABLE usr (...);
CREATE TABLE prod (...);

-- Ambiguous names
CREATE TABLE data (...);
CREATE TABLE info (...);
```

### 2. Relationship Design Rules

**Rule 1: Every M:N relationship needs a junction table**

**❌ BAD (Trying to avoid junction table):**

```sql
-- Storing multiple values in one column
CREATE TABLE students (
    student_id UUID PRIMARY KEY,
    course_ids TEXT  -- 'uuid1,uuid2,uuid3' ← ANTIPATTERN
);
```

**✅ GOOD:**

```sql
CREATE TABLE student_courses (
    student_id UUID,
    course_id UUID,
    enrollment_date DATE,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

**Rule 2: Junction tables should have business meaning**

**❌ BAD (Generic names):**

```sql
CREATE TABLE student_course (...);  -- Just combines names
CREATE TABLE user_product (...);
```

**✅ GOOD:**

```sql
CREATE TABLE enrollments (...);     -- Business concept
CREATE TABLE purchases (...);       -- Represents action
CREATE TABLE memberships (...);     -- Domain term
```

**Rule 3: Consider the relationship's attributes**

**❌ BAD (Missing relationship data):**

```sql
-- M:N without attributes loses context
CREATE TABLE project_assignments (
    employee_id UUID,
    project_id UUID,
    PRIMARY KEY (employee_id, project_id)
);
-- When did they start? What's their role? How many hours?
```

**✅ GOOD:**

```sql
CREATE TABLE project_assignments (
    assignment_id UUID PRIMARY KEY,
    employee_id UUID NOT NULL,
    project_id UUID NOT NULL,
    assigned_date DATE NOT NULL,
    role VARCHAR(50) NOT NULL,  -- 'Lead', 'Developer', 'QA'
    hours_allocated INTEGER,
    billable_rate DECIMAL(10,2),

    UNIQUE (employee_id, project_id),
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    FOREIGN KEY (project_id) REFERENCES projects(project_id)
);
```

### 3. Primary Key Selection

**✅ GOOD Strategies:**

```sql
-- 1. Surrogate keys (UUID) - Best for distributed systems
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL
);

-- 2. Auto-increment - Good for single-server systems
CREATE TABLE orders (
    order_id BIGSERIAL PRIMARY KEY,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL
);

-- 3. Natural composite keys - When truly unique
CREATE TABLE inventory_locations (
    warehouse_id INTEGER,
    aisle CHAR(3),
    shelf INTEGER,
    bin CHAR(2),
    PRIMARY KEY (warehouse_id, aisle, shelf, bin)
);
```

**❌ BAD:**

```sql
-- Using business data as primary key
CREATE TABLE users (
    email VARCHAR(255) PRIMARY KEY  -- Email can change!
);

-- Overly complex composite keys
CREATE TABLE transactions (
    PRIMARY KEY (date, time, user_id, product_id, store_id)
    -- 5-column PK = maintenance nightmare
);
```

### 4. Cardinality Validation

**Always validate assumed cardinality with business experts:**

```
Developer Assumption: User ──1────1── ShippingAddress
Reality: User ──1────N── ShippingAddress (home, work, vacation)

Developer Assumption: Order ──1────N── Payment
Reality: Order ──N────N── Payment (split payments, refunds)

Developer Assumption: Product ──1────N── Category
Reality: Product ──M────N── Category (multi-category products)
```

### 5. Documentation Standards

**✅ GOOD ER Diagram Documentation:**

```
┌─────────────────────────────────────────────────────────────┐
│                  E-COMMERCE ER DIAGRAM                       │
│                                                               │
│  Created: 2026-02-03                                         │
│  Author: Data Architecture Team                              │
│  Version: 2.1                                                │
│  Last Review: 2026-01-15                                     │
│                                                               │
│  BUSINESS RULES:                                             │
│  1. One customer can have multiple addresses                 │
│  2. Orders support split payments (M:N with payments)        │
│  3. Products can belong to multiple categories               │
│  4. Each order must have at least one item                   │
│  5. Customer accounts are soft-deleted (archived)            │
│                                                               │
│  CARDINALITY DECISIONS:                                      │
│  - User:Address = 1:N (validated with business 2025-12-10)  │
│  - Order:Payment = M:N (supports payment splitting)          │
│  - Product:Category = M:N (multi-category products)          │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Common Mistakes & Antipatterns

### Mistake 1: Mixing Conceptual and Physical Design

**❌ BAD (Physical details in conceptual ERD):**

```
Conceptual ERD should not show:
- Data types (VARCHAR(255), INTEGER)
- Indexes
- Partition keys
- Database-specific features

┌─────────────┐
│    User     │
│─────────────│
│ UUID        │ ← Too detailed for conceptual
│ VARCHAR(255)│
│ TIMESTAMP   │
└─────────────┘
```

**✅ GOOD (Separate concerns):**

```
Conceptual ERD:
┌──────────┐
│   User   │
│──────────│
│ ID       │
│ Email    │
│ Name     │
│ Created  │
└──────────┘

Physical ERD:
┌─────────────────────────────────┐
│   users                          │
│─────────────────────────────────│
│ user_id: UUID PK                │
│ email: VARCHAR(255) UNIQUE      │
│ name: VARCHAR(100) NOT NULL     │
│ created_at: TIMESTAMPTZ         │
│ INDEX: idx_email                │
│ PARTITION BY: RANGE(created_at) │
└─────────────────────────────────┘
```

### Mistake 2: Failing to Identify Hidden Entities

**❌ BAD (Missing junction table attributes as entity):**

```sql
-- Treating M:N too simply
CREATE TABLE student_courses (
    student_id UUID,
    course_id UUID,
    grade CHAR(2),           -- What semester?
    attendance_rate DECIMAL, -- Which professor?
    final_score INTEGER,     -- Multiple enrollments possible?
    PRIMARY KEY (student_id, course_id)
);
-- This is actually an ENROLLMENT entity!
```

**✅ GOOD (Recognizing enrollment as a first-class entity):**

```sql
CREATE TABLE enrollments (
    enrollment_id UUID PRIMARY KEY,
    student_id UUID NOT NULL,
    course_id UUID NOT NULL,
    semester_id UUID NOT NULL,
    instructor_id UUID NOT NULL,
    enrollment_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL, -- 'active', 'dropped', 'completed'
    grade CHAR(2),
    attendance_rate DECIMAL(5,2),
    final_score INTEGER,

    -- Student can enroll in same course multiple semesters
    UNIQUE (student_id, course_id, semester_id),

    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id),
    FOREIGN KEY (semester_id) REFERENCES semesters(semester_id),
    FOREIGN KEY (instructor_id) REFERENCES instructors(instructor_id)
);
```

### Mistake 3: Incorrect Cardinality

**Real-World Disaster: Healthcare System (2019)**

```
Assumed ER Model:
Patient ──1────N── MedicalRecord
(One patient has many medical records)

Patient ──1────1── Insurance
(One patient has one insurance)

Reality After 6 Months in Production:
1. Patients can have multiple active insurances (primary, secondary)
2. Family members can share insurance policies
3. Insurance changes over time (historical records needed)

Cost of Fixing:
- 3 months of migration work
- Data integrity issues for 50,000+ records
- $2M in engineering costs
- Regulatory compliance violations
```

**✅ CORRECT Model:**

```sql
CREATE TABLE patients (
    patient_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE insurance_policies (
    policy_id UUID PRIMARY KEY,
    policy_number VARCHAR(50) UNIQUE,
    provider VARCHAR(100) NOT NULL
);

-- M:N relationship with temporal data
CREATE TABLE patient_insurance_coverage (
    coverage_id UUID PRIMARY KEY,
    patient_id UUID NOT NULL,
    policy_id UUID NOT NULL,
    coverage_type VARCHAR(20), -- 'primary', 'secondary'
    start_date DATE NOT NULL,
    end_date DATE,

    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (policy_id) REFERENCES insurance_policies(policy_id),

    -- Prevent overlapping primary coverage
    EXCLUDE USING gist (
        patient_id WITH =,
        daterange(start_date, end_date, '[]') WITH &&
    ) WHERE (coverage_type = 'primary')
);
```

### Mistake 4: Overusing Inheritance

**❌ BAD (Inheritance where composition is better):**

```
ER Model:
        ┌──────────┐
        │ Document │
        └────┬─────┘
             │ IS-A
    ┌────────┼────────┐
    │        │        │
┌───┴───┐ ┌─┴──┐ ┌───┴────┐
│Invoice│ │Quote│ │Contract│
└───────┘ └────┘ └─────────┘

Problems:
- Adding new document type requires schema change
- Shared attributes scattered across tables
- Complex queries for "all documents"
```

**✅ GOOD (Composition approach):**

```sql
CREATE TABLE documents (
    document_id UUID PRIMARY KEY,
    document_type VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    created_by UUID NOT NULL,

    -- Common attributes for all documents
    status VARCHAR(20) NOT NULL,
    version INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE document_metadata (
    metadata_id UUID PRIMARY KEY,
    document_id UUID NOT NULL,
    key VARCHAR(100) NOT NULL,
    value TEXT NOT NULL,

    UNIQUE (document_id, key),
    FOREIGN KEY (document_id) REFERENCES documents(document_id)
);

-- Type-specific tables only for complex type-specific data
CREATE TABLE invoices (
    invoice_id UUID PRIMARY KEY,
    document_id UUID NOT NULL UNIQUE,
    invoice_number VARCHAR(50) UNIQUE,
    due_date DATE NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,

    FOREIGN KEY (document_id) REFERENCES documents(document_id)
);
```

### Mistake 5: Ignoring Temporal Relationships

**❌ BAD (No history tracking):**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    department_id UUID NOT NULL,  -- Current department only
    salary DECIMAL(10,2) NOT NULL, -- Current salary only
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
-- Can't answer: "What was John's salary in 2023?"
```

**✅ GOOD (Temporal modeling):**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    hire_date DATE NOT NULL
);

CREATE TABLE employee_department_history (
    history_id UUID PRIMARY KEY,
    employee_id UUID NOT NULL,
    department_id UUID NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE,  -- NULL = current

    FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
    FOREIGN KEY (department_id) REFERENCES departments(department_id),

    -- Prevent overlapping assignments
    EXCLUDE USING gist (
        employee_id WITH =,
        daterange(start_date, end_date, '[]') WITH &&
    )
);

CREATE TABLE employee_salary_history (
    history_id UUID PRIMARY KEY,
    employee_id UUID NOT NULL,
    salary DECIMAL(10,2) NOT NULL,
    currency CHAR(3) NOT NULL,
    effective_date DATE NOT NULL,
    end_date DATE,

    FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
);
```

---

## 6. Security Considerations

### 1. Sensitive Data Entity Separation

**✅ GOOD (Separate sensitive data):**

```sql
-- Public/semi-public data
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    created_at TIMESTAMP NOT NULL
);

-- Sensitive authentication data (restricted access)
CREATE TABLE user_credentials (
    credential_id UUID PRIMARY KEY,
    user_id UUID NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    password_salt VARCHAR(255) NOT NULL,
    failed_login_attempts INTEGER DEFAULT 0,
    locked_until TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Highly sensitive PII (separate database/encrypted)
CREATE TABLE user_pii (
    pii_id UUID PRIMARY KEY,
    user_id UUID NOT NULL UNIQUE,
    ssn_encrypted BYTEA,  -- Encrypted at application level
    dob_encrypted BYTEA,
    address_encrypted BYTEA,
    encryption_key_version INTEGER NOT NULL,

    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Row-level security
ALTER TABLE user_credentials ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_pii ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_credentials_isolation ON user_credentials
    USING (user_id = current_setting('app.current_user_id')::UUID);
```

### 2. Audit Trail Entities

**✅ GOOD (Comprehensive audit logging):**

```sql
CREATE TABLE audit_log (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name VARCHAR(100) NOT NULL,
    operation VARCHAR(10) NOT NULL, -- INSERT, UPDATE, DELETE
    record_id UUID NOT NULL,
    user_id UUID,
    changed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    ip_address INET,
    user_agent TEXT,
    old_values JSONB,  -- Previous state
    new_values JSONB,  -- New state

    -- Immutable audit log
    CHECK (changed_at <= NOW())
);

-- Prevent modifications to audit log
CREATE POLICY no_update_audit ON audit_log FOR UPDATE USING (false);
CREATE POLICY no_delete_audit ON audit_log FOR DELETE USING (false);

-- Trigger for automatic audit logging
CREATE OR REPLACE FUNCTION log_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_values)
        VALUES (TG_TABLE_NAME, TG_OP, OLD.id, row_to_json(OLD));
        RETURN OLD;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_values, new_values)
        VALUES (TG_TABLE_NAME, TG_OP, NEW.id, row_to_json(OLD), row_to_json(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, record_id, new_values)
        VALUES (TG_TABLE_NAME, TG_OP, NEW.id, row_to_json(NEW));
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### 3. Multi-Tenancy Modeling

**✅ GOOD (Secure tenant isolation):**

```sql
CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY,
    tenant_slug VARCHAR(50) UNIQUE NOT NULL,
    subscription_tier VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL
);

-- Every entity has tenant_id
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,

    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE,

    -- Email unique per tenant
    UNIQUE (tenant_id, email)
);

-- Row-level security enforces tenant isolation
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON customers
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

-- Indexes include tenant_id for query efficiency
CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_tenant_email ON customers(tenant_id, email);
```

---

## 7. Performance Optimization

### 1. Relationship Index Strategy

**Query Pattern Analysis:**

```sql
-- Analyze common join patterns
SELECT
    schemaname,
    tablename,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE tablename IN ('orders', 'order_items')
ORDER BY idx_scan DESC;
```

**✅ GOOD (Strategic index placement):**

```sql
-- 1:N relationship - index foreign key on "many" side
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL,

    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Critical: Index FK for JOIN performance
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Covering index for common query patterns
CREATE INDEX idx_orders_customer_date
    ON orders(customer_id, order_date DESC)
    INCLUDE (order_id, total_amount);

-- M:N junction table - compound index on both FKs
CREATE TABLE order_products (
    order_id UUID NOT NULL,
    product_id UUID NOT NULL,
    quantity INTEGER NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

-- Optimize both direction queries
CREATE INDEX idx_order_products_product ON order_products(product_id);
```

### 2. Denormalization for Read Performance

**When to Denormalize:**

```
┌─────────────────────────────────────────────────────────────┐
│              DENORMALIZATION DECISION MATRIX                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Denormalize When:                                           │
│  ✅ Read:Write ratio > 100:1                                │
│  ✅ JOIN involves 5+ tables                                 │
│  ✅ Query performance critical (< 50ms SLA)                 │
│  ✅ Data rarely changes                                     │
│  ✅ Acceptable staleness (eventual consistency)             │
│                                                               │
│  Keep Normalized When:                                       │
│  ❌ Data changes frequently                                 │
│  ❌ Strong consistency required                             │
│  ❌ Low read volume                                         │
│  ❌ Complex update patterns                                 │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**✅ GOOD (Strategic denormalization):**

```sql
-- Normalized schema
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    category_id UUID NOT NULL,
    price DECIMAL(10,2) NOT NULL
);

CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- Denormalized for read-heavy product listing
CREATE MATERIALIZED VIEW product_listing AS
SELECT
    p.product_id,
    p.name AS product_name,
    p.price,
    c.name AS category_name,
    c.category_id,
    COUNT(r.review_id) AS review_count,
    AVG(r.rating) AS avg_rating
FROM products p
JOIN categories c ON p.category_id = c.category_id
LEFT JOIN reviews r ON p.product_id = r.product_id
GROUP BY p.product_id, p.name, p.price, c.name, c.category_id;

-- Refresh strategy
CREATE INDEX idx_product_listing_category ON product_listing(category_id);
CREATE INDEX idx_product_listing_price ON product_listing(price);

-- Refresh incrementally
REFRESH MATERIALIZED VIEW CONCURRENTLY product_listing;
```

### 3. Partitioning Strategy Based on Relationships

**✅ GOOD (Partition by parent entity):**

```sql
-- Orders partitioned by customer
CREATE TABLE orders (
    order_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (customer_id, order_id)
) PARTITION BY HASH (customer_id);

CREATE TABLE orders_p0 PARTITION OF orders
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_p1 PARTITION OF orders
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_p2 PARTITION OF orders
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_p3 PARTITION OF orders
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Query benefits from partition pruning
SELECT * FROM orders
WHERE customer_id = '123e4567-e89b-12d3-a456-426614174000'
-- Only scans one partition
```

---

## 8. Real-World Use Cases

### Use Case 1: E-Commerce Platform

**Complete ER Model:**

```
┌─────────────────────────────────────────────────────────────┐
│              E-COMMERCE ENTITY RELATIONSHIPS                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Customer ──1────N── Order ──1────N── OrderItem              │
│      │                │                   │                   │
│      │                │                   │                   │
│     1│               N│                  N│                   │
│      │                │                   │                   │
│  Address         Payment              Product                │
│     N│               M│                  N│                   │
│      │                │                   │                   │
│      └────────────────┴───────────────────┴──N── Category   │
│                                                               │
│  Business Rules:                                             │
│  - Customer can have multiple shipping addresses            │
│  - Order can have split payments (M:N)                      │
│  - Products can belong to multiple categories               │
│  - Order items track inventory at time of order             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Implementation:**

```sql
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW(),
    last_login TIMESTAMP
);

CREATE TABLE addresses (
    address_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL,
    address_type VARCHAR(20) NOT NULL, -- 'shipping', 'billing'
    street_address VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50),
    postal_code VARCHAR(20) NOT NULL,
    country CHAR(2) NOT NULL,
    is_default BOOLEAN DEFAULT false,

    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE
);

CREATE TABLE products (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    cost DECIMAL(10,2) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    reorder_level INTEGER NOT NULL DEFAULT 10,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE categories (
    category_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    parent_category_id UUID,

    FOREIGN KEY (parent_category_id) REFERENCES categories(category_id)
);

-- M:N relationship
CREATE TABLE product_categories (
    product_id UUID,
    category_id UUID,
    display_order INTEGER,

    PRIMARY KEY (product_id, category_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE CASCADE
);

CREATE TABLE orders (
    order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    order_date TIMESTAMP DEFAULT NOW(),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    subtotal DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) NOT NULL,
    shipping_amount DECIMAL(10,2) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    shipping_address_id UUID NOT NULL,
    billing_address_id UUID NOT NULL,

    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(address_id),
    FOREIGN KEY (billing_address_id) REFERENCES addresses(address_id)
);

CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    product_id UUID NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,  -- Price at time of order
    subtotal DECIMAL(10,2) NOT NULL,

    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

CREATE TABLE payments (
    payment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_method VARCHAR(20) NOT NULL, -- 'credit_card', 'paypal', 'gift_card'
    amount DECIMAL(10,2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    transaction_id VARCHAR(255),
    processed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- M:N relationship (supports split payments)
CREATE TABLE order_payments (
    order_id UUID,
    payment_id UUID,
    amount DECIMAL(10,2) NOT NULL,

    PRIMARY KEY (order_id, payment_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (payment_id) REFERENCES payments(payment_id)
);

-- Indexes for common queries
CREATE INDEX idx_orders_customer ON orders(customer_id, order_date DESC);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);
CREATE INDEX idx_addresses_customer ON addresses(customer_id);
CREATE INDEX idx_product_categories_category ON product_categories(category_id);
```

### Use Case 2: Social Media Platform

**Complex Relationship Model:**

```sql
-- Self-referencing relationship (followers)
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    bio TEXT,
    avatar_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW()
);

-- M:N self-referencing (followers/following)
CREATE TABLE user_relationships (
    follower_id UUID NOT NULL,
    following_id UUID NOT NULL,
    followed_at TIMESTAMP DEFAULT NOW(),
    notification_enabled BOOLEAN DEFAULT true,

    PRIMARY KEY (follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(user_id) ON DELETE CASCADE,
    CHECK (follower_id != following_id)  -- Can't follow yourself
);

CREATE TABLE posts (
    post_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    content TEXT NOT NULL,
    media_urls TEXT[],
    post_type VARCHAR(20) NOT NULL, -- 'text', 'image', 'video', 'poll'
    visibility VARCHAR(20) DEFAULT 'public', -- 'public', 'followers', 'private'
    created_at TIMESTAMP DEFAULT NOW(),
    edited_at TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Self-referencing (post replies)
CREATE TABLE post_relationships (
    post_id UUID NOT NULL,
    parent_post_id UUID,  -- NULL for top-level posts
    relationship_type VARCHAR(20), -- 'reply', 'repost', 'quote'

    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
    FOREIGN KEY (parent_post_id) REFERENCES posts(post_id) ON DELETE CASCADE
);

CREATE TABLE reactions (
    reaction_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    post_id UUID NOT NULL,
    reaction_type VARCHAR(20) NOT NULL, -- 'like', 'love', 'laugh', 'angry'
    created_at TIMESTAMP DEFAULT NOW(),

    UNIQUE (user_id, post_id),  -- One reaction per user per post
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE
);

-- Indexes for social graph queries
CREATE INDEX idx_relationships_follower ON user_relationships(follower_id);
CREATE INDEX idx_relationships_following ON user_relationships(following_id);
CREATE INDEX idx_posts_user ON posts(user_id, created_at DESC);
CREATE INDEX idx_post_relationships_parent ON post_relationships(parent_post_id);
CREATE INDEX idx_reactions_post ON reactions(post_id, reaction_type);
```

### Use Case 3: Healthcare System (Temporal Relationships)

```sql
CREATE TABLE patients (
    patient_id UUID PRIMARY KEY,
    mrn VARCHAR(50) UNIQUE NOT NULL,  -- Medical Record Number
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    date_of_birth DATE NOT NULL,
    gender CHAR(1)
);

CREATE TABLE healthcare_providers (
    provider_id UUID PRIMARY KEY,
    npi VARCHAR(10) UNIQUE NOT NULL,  -- National Provider Identifier
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    specialty VARCHAR(100) NOT NULL
);

-- Temporal relationship: patient-provider assignment over time
CREATE TABLE patient_provider_assignments (
    assignment_id UUID PRIMARY KEY,
    patient_id UUID NOT NULL,
    provider_id UUID NOT NULL,
    assignment_type VARCHAR(50) NOT NULL, -- 'primary_care', 'specialist'
    start_date DATE NOT NULL,
    end_date DATE,  -- NULL = current

    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (provider_id) REFERENCES healthcare_providers(provider_id),

    -- Prevent overlapping primary care assignments
    EXCLUDE USING gist (
        patient_id WITH =,
        daterange(start_date, end_date, '[]') WITH &&
    ) WHERE (assignment_type = 'primary_care')
);

CREATE TABLE medications (
    medication_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    generic_name VARCHAR(255),
    dosage_forms VARCHAR(100)[]
);

-- Temporal: medication prescriptions over time
CREATE TABLE prescriptions (
    prescription_id UUID PRIMARY KEY,
    patient_id UUID NOT NULL,
    provider_id UUID NOT NULL,
    medication_id UUID NOT NULL,
    dosage VARCHAR(100) NOT NULL,
    frequency VARCHAR(100) NOT NULL,
    prescribed_date DATE NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE,  -- NULL = ongoing
    status VARCHAR(20) NOT NULL, -- 'active', 'completed', 'discontinued'

    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (provider_id) REFERENCES healthcare_providers(provider_id),
    FOREIGN KEY (medication_id) REFERENCES medications(medication_id)
);

-- Indexes for temporal queries
CREATE INDEX idx_assignments_patient_dates
    ON patient_provider_assignments(patient_id, start_date, end_date);
CREATE INDEX idx_prescriptions_patient_active
    ON prescriptions(patient_id, status, start_date)
    WHERE status = 'active';
```

---

## 9. Advanced Patterns & Techniques

### Pattern 1: Polymorphic Associations

**Scenario:** Comments can belong to posts, photos, or videos

**❌ BAD (Multiple nullable foreign keys):**

```sql
CREATE TABLE comments (
    comment_id UUID PRIMARY KEY,
    content TEXT NOT NULL,
    post_id UUID,
    photo_id UUID,
    video_id UUID,
    CHECK (
        (post_id IS NOT NULL AND photo_id IS NULL AND video_id IS NULL) OR
        (photo_id IS NOT NULL AND post_id IS NULL AND video_id IS NULL) OR
        (video_id IS NOT NULL AND post_id IS NULL AND photo_id IS NULL)
    )
);
-- Messy, hard to maintain, no referential integrity
```

**✅ GOOD (Generic foreign key with type discrimination):**

```sql
CREATE TABLE comments (
    comment_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    commentable_type VARCHAR(50) NOT NULL,  -- 'post', 'photo', 'video'
    commentable_id UUID NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),

    FOREIGN KEY (user_id) REFERENCES users(user_id),

    -- Partial unique index per type
    UNIQUE (commentable_type, commentable_id, comment_id)
);

-- Create views for type-specific queries
CREATE VIEW post_comments AS
SELECT c.*
FROM comments c
WHERE c.commentable_type = 'post';

-- Enforce referential integrity with triggers
CREATE FUNCTION check_commentable_exists()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.commentable_type = 'post' THEN
        IF NOT EXISTS (SELECT 1 FROM posts WHERE post_id = NEW.commentable_id) THEN
            RAISE EXCEPTION 'Referenced post does not exist';
        END IF;
    ELSIF NEW.commentable_type = 'photo' THEN
        IF NOT EXISTS (SELECT 1 FROM photos WHERE photo_id = NEW.commentable_id) THEN
            RAISE EXCEPTION 'Referenced photo does not exist';
        END IF;
    ELSIF NEW.commentable_type = 'video' THEN
        IF NOT EXISTS (SELECT 1 FROM videos WHERE video_id = NEW.commentable_id) THEN
            RAISE EXCEPTION 'Referenced video does not exist';
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_commentable_integrity
    BEFORE INSERT OR UPDATE ON comments
    FOR EACH ROW EXECUTE FUNCTION check_commentable_exists();
```

### Pattern 2: Recursive Relationships (Hierarchies)

**✅ GOOD (Adjacency List with Closure Table):**

```sql
-- Adjacency list for simple queries
CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id UUID,
    level INTEGER NOT NULL,

    FOREIGN KEY (parent_id) REFERENCES categories(category_id)
);

-- Closure table for efficient hierarchy queries
CREATE TABLE category_paths (
    ancestor_id UUID NOT NULL,
    descendant_id UUID NOT NULL,
    path_length INTEGER NOT NULL,

    PRIMARY KEY (ancestor_id, descendant_id),
    FOREIGN KEY (ancestor_id) REFERENCES categories(category_id) ON DELETE CASCADE,
    FOREIGN KEY (descendant_id) REFERENCES categories(category_id) ON DELETE CASCADE
);

-- Trigger to maintain closure table
CREATE FUNCTION maintain_category_paths()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        -- Insert self-reference
        INSERT INTO category_paths (ancestor_id, descendant_id, path_length)
        VALUES (NEW.category_id, NEW.category_id, 0);

        -- Insert paths from ancestors
        IF NEW.parent_id IS NOT NULL THEN
            INSERT INTO category_paths (ancestor_id, descendant_id, path_length)
            SELECT ancestor_id, NEW.category_id, path_length + 1
            FROM category_paths
            WHERE descendant_id = NEW.parent_id;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER category_paths_trigger
    AFTER INSERT ON categories
    FOR EACH ROW EXECUTE FUNCTION maintain_category_paths();

-- Efficient queries
-- Get all descendants (subtree)
SELECT c.*
FROM categories c
JOIN category_paths cp ON c.category_id = cp.descendant_id
WHERE cp.ancestor_id = '...' AND cp.path_length > 0;

-- Get all ancestors (path to root)
SELECT c.*
FROM categories c
JOIN category_paths cp ON c.category_id = cp.ancestor_id
WHERE cp.descendant_id = '...' AND cp.path_length > 0
ORDER BY cp.path_length DESC;
```

### Pattern 3: Versioned Entities

**✅ GOOD (Temporal versioning with history):**

```sql
CREATE TABLE documents (
    document_id UUID PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    created_by UUID NOT NULL
);

CREATE TABLE document_versions (
    version_id UUID PRIMARY KEY,
    document_id UUID NOT NULL,
    version_number INTEGER NOT NULL,
    content TEXT NOT NULL,
    changed_by UUID NOT NULL,
    changed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    change_summary VARCHAR(500),
    is_current BOOLEAN NOT NULL DEFAULT false,

    FOREIGN KEY (document_id) REFERENCES documents(document_id) ON DELETE CASCADE,
    UNIQUE (document_id, version_number),

    -- Only one current version per document
    EXCLUDE USING gist (document_id WITH =, is_current WITH AND)
        WHERE (is_current = true)
);

-- View for current versions only
CREATE VIEW current_documents AS
SELECT
    d.document_id,
    d.title,
    d.created_at,
    dv.version_number,
    dv.content,
    dv.changed_at AS last_modified,
    dv.changed_by AS last_modified_by
FROM documents d
JOIN document_versions dv ON d.document_id = dv.document_id
WHERE dv.is_current = true;
```

---

## 10. Interview Questions

### Junior Level (0-2 years)

**Q1: What is the difference between 1:1, 1:N, and M:N relationships?**

**Answer:**

- **1:1 (One-to-One)**: Each entity instance relates to exactly one instance of another entity. Example: User ↔ UserProfile. Implemented with a unique foreign key.
- **1:N (One-to-Many)**: One entity instance relates to multiple instances of another entity. Example: Customer → Orders. Implemented with a foreign key on the "many" side.
- **M:N (Many-to-Many)**: Multiple instances on both sides. Example: Students ↔ Courses. Requires a junction table with foreign keys to both entities.

**Q2: How do you convert an ER diagram to a relational schema?**

**Answer:**

1. **Entities → Tables**: Each entity becomes a table
2. **Attributes → Columns**: Entity attributes become table columns
3. **Primary Keys → Primary Keys**: Underlined attributes become PRIMARY KEY constraints
4. **Relationships → Foreign Keys**:
   - 1:N: Foreign key on the "many" side
   - M:N: Create junction table with foreign keys to both entities
   - 1:1: Foreign key on either side (usually the dependent entity)
5. **Add Constraints**: NOT NULL, UNIQUE, CHECK constraints based on business rules

**Q3: What is a weak entity and how is it implemented?**

**Answer:**
A weak entity cannot exist without its parent entity. It doesn't have a primary key of its own; instead, it uses a composite key that includes the parent's primary key.

Example: Order → OrderItems (items can't exist without an order)

```sql
CREATE TABLE order_items (
    order_id UUID NOT NULL,
    item_number INTEGER NOT NULL,
    product_id UUID NOT NULL,
    quantity INTEGER NOT NULL,

    PRIMARY KEY (order_id, item_number),  -- Composite key
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE
);
```

### Mid-Level (2-5 years)

**Q4: How would you model a social network's follower system?**

**Answer:**
This is a self-referencing many-to-many relationship:

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE user_follows (
    follower_id UUID NOT NULL,
    following_id UUID NOT NULL,
    followed_at TIMESTAMP DEFAULT NOW(),

    PRIMARY KEY (follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(user_id) ON DELETE CASCADE,
    CHECK (follower_id != following_id)  -- Can't follow yourself
);

-- Query: Get all followers of a user
SELECT u.*
FROM users u
JOIN user_follows f ON u.user_id = f.follower_id
WHERE f.following_id = ?;

-- Query: Get all users a user is following
SELECT u.*
FROM users u
JOIN user_follows f ON u.user_id = f.following_id
WHERE f.follower_id = ?;
```

**Q5: When should you use a junction table with additional attributes vs. a simple M:N table?**

**Answer:**
Use a junction table with attributes when the relationship itself has properties:

**Simple M:N (no relationship attributes):**

```sql
CREATE TABLE user_roles (
    user_id UUID,
    role_id UUID,
    PRIMARY KEY (user_id, role_id)
);
```

**Junction table with attributes:**

```sql
CREATE TABLE enrollments (
    enrollment_id UUID PRIMARY KEY,
    student_id UUID NOT NULL,
    course_id UUID NOT NULL,
    enrolled_date DATE NOT NULL,       -- Relationship attribute
    grade CHAR(2),                      -- Relationship attribute
    attendance_percentage DECIMAL(5,2), -- Relationship attribute

    UNIQUE (student_id, course_id)
);
```

Use junction table with attributes when you need to:

- Track when the relationship was created
- Store relationship-specific data (grade, role assignment date)
- Support temporal queries (active vs historical relationships)
- Have a unique identifier for the relationship itself

**Q6: How do you model a product hierarchy (category tree)?**

**Answer:**
Use an adjacency list with self-referencing foreign key:

```sql
CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id UUID,
    level INTEGER NOT NULL,
    sort_order INTEGER,

    FOREIGN KEY (parent_id) REFERENCES categories(category_id) ON DELETE CASCADE
);

-- For efficient hierarchy queries, add a closure table:
CREATE TABLE category_closure (
    ancestor_id UUID NOT NULL,
    descendant_id UUID NOT NULL,
    depth INTEGER NOT NULL,

    PRIMARY KEY (ancestor_id, descendant_id),
    FOREIGN KEY (ancestor_id) REFERENCES categories(category_id),
    FOREIGN KEY (descendant_id) REFERENCES categories(category_id)
);

-- Query all descendants (subtree)
SELECT c.*
FROM categories c
JOIN category_closure cc ON c.category_id = cc.descendant_id
WHERE cc.ancestor_id = ? AND cc.depth > 0;
```

### Senior Level (5-8 years)

**Q7: Design a data model for a multi-tenant SaaS application with tenant isolation.**

**Answer:**
Implement tenant isolation at the data model level:

```sql
CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY,
    tenant_slug VARCHAR(50) UNIQUE NOT NULL,
    subscription_tier VARCHAR(20) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Every entity includes tenant_id for isolation
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    email VARCHAR(255) NOT NULL,

    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    UNIQUE (tenant_id, email)  -- Email unique per tenant
);

-- Row-level security enforces tenant isolation
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Indexes include tenant_id
CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_tenant_email ON users(tenant_id, email);

-- Partition by tenant for large-scale systems
CREATE TABLE orders (
    order_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP NOT NULL,

    PRIMARY KEY (tenant_id, order_id)
) PARTITION BY HASH (tenant_id);
```

**Q8: How would you model a versioned document system with branching (like Git)?**

**Answer:**
Model documents with version trees supporting branches and merges:

```sql
CREATE TABLE documents (
    document_id UUID PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE document_versions (
    version_id UUID PRIMARY KEY,
    document_id UUID NOT NULL,
    version_hash VARCHAR(64) UNIQUE NOT NULL,  -- SHA-256 of content
    parent_version_id UUID,  -- NULL for initial version
    branch_name VARCHAR(100) NOT NULL DEFAULT 'main',
    content TEXT NOT NULL,
    author_id UUID NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    commit_message VARCHAR(500),

    FOREIGN KEY (document_id) REFERENCES documents(document_id),
    FOREIGN KEY (parent_version_id) REFERENCES document_versions(version_id)
);

-- Track merge commits (versions with multiple parents)
CREATE TABLE document_version_merges (
    version_id UUID NOT NULL,
    parent_version_id UUID NOT NULL,
    merge_order INTEGER NOT NULL,

    PRIMARY KEY (version_id, parent_version_id),
    FOREIGN KEY (version_id) REFERENCES document_versions(version_id),
    FOREIGN KEY (parent_version_id) REFERENCES document_versions(version_id)
);

-- Track branch heads
CREATE TABLE document_branches (
    branch_id UUID PRIMARY KEY,
    document_id UUID NOT NULL,
    branch_name VARCHAR(100) NOT NULL,
    head_version_id UUID NOT NULL,

    UNIQUE (document_id, branch_name),
    FOREIGN KEY (document_id) REFERENCES documents(document_id),
    FOREIGN KEY (head_version_id) REFERENCES document_versions(version_id)
);

-- Query version history (like git log)
WITH RECURSIVE version_history AS (
    -- Start with latest version
    SELECT version_id, parent_version_id, version_hash, created_at, 0 AS depth
    FROM document_versions
    WHERE version_id = ?

    UNION ALL

    -- Recursively get parent versions
    SELECT v.version_id, v.parent_version_id, v.version_hash, v.created_at, vh.depth + 1
    FROM document_versions v
    JOIN version_history vh ON v.version_id = vh.parent_version_id
)
SELECT * FROM version_history ORDER BY depth;
```

### Staff/Principal Level (8+ years)

**Q9: Design a data model for a financial trading platform handling millions of trades per second with full audit compliance.**

**Answer:**
Hybrid append-only event sourcing with CQRS:

```sql
-- Write model: Append-only event log (optimized for writes)
CREATE TABLE trade_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id UUID NOT NULL,  -- Trade ID
    event_type VARCHAR(50) NOT NULL,
    event_version INTEGER NOT NULL,
    event_data JSONB NOT NULL,
    user_id UUID NOT NULL,
    occurred_at TIMESTAMP NOT NULL DEFAULT NOW(),
    sequence_number BIGSERIAL,

    -- Immutable audit log
    CHECK (occurred_at <= NOW())
) PARTITION BY RANGE (occurred_at);

-- Partition by month for compliance retention
CREATE TABLE trade_events_2026_02 PARTITION OF trade_events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Read model: Materialized current state (optimized for queries)
CREATE TABLE trades (
    trade_id UUID PRIMARY KEY,
    trader_id UUID NOT NULL,
    symbol VARCHAR(10) NOT NULL,
    side VARCHAR(10) NOT NULL, -- 'BUY', 'SELL'
    quantity DECIMAL(20,8) NOT NULL,
    price DECIMAL(20,8) NOT NULL,
    status VARCHAR(20) NOT NULL, -- 'pending', 'filled', 'cancelled'
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    version INTEGER NOT NULL,  -- Optimistic locking

    -- Compliance indexes
    CHECK (quantity > 0 AND price > 0)
);

-- Position snapshots (aggregate state)
CREATE TABLE trader_positions (
    position_id UUID PRIMARY KEY,
    trader_id UUID NOT NULL,
    symbol VARCHAR(10) NOT NULL,
    quantity DECIMAL(20,8) NOT NULL,
    average_cost DECIMAL(20,8) NOT NULL,
    realized_pnl DECIMAL(20,8) NOT NULL,
    unrealized_pnl DECIMAL(20,8) NOT NULL,
    last_updated TIMESTAMP NOT NULL,

    UNIQUE (trader_id, symbol)
);

-- Idempotency table (prevent duplicate processing)
CREATE TABLE processed_events (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Performance indexes
CREATE INDEX idx_trade_events_aggregate ON trade_events(aggregate_id, sequence_number);
CREATE INDEX idx_trades_trader_symbol ON trades(trader_id, symbol, status);
CREATE INDEX idx_positions_trader ON trader_positions(trader_id);
```

**Q10: How would you model a global e-commerce platform supporting multiple currencies, tax jurisdictions, and regulatory compliance?**

**Answer:**
Multi-dimensional modeling with temporal and regulatory aspects:

```sql
-- Core entities with regulatory tracking
CREATE TABLE legal_entities (
    entity_id UUID PRIMARY KEY,
    entity_name VARCHAR(255) NOT NULL,
    jurisdiction VARCHAR(50) NOT NULL,  -- 'US-CA', 'EU-DE', 'UK'
    tax_id VARCHAR(100) NOT NULL,
    regulatory_status VARCHAR(50) NOT NULL,
    compliance_tier VARCHAR(20) NOT NULL,  -- 'tier1', 'tier2', 'tier3'

    UNIQUE (jurisdiction, tax_id)
);

CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    is_regulated BOOLEAN NOT NULL DEFAULT false,
    restricted_jurisdictions VARCHAR(50)[]
);

-- Multi-currency pricing with temporal validity
CREATE TABLE product_prices (
    price_id UUID PRIMARY KEY,
    product_id UUID NOT NULL,
    entity_id UUID NOT NULL,
    currency CHAR(3) NOT NULL,
    amount DECIMAL(20,8) NOT NULL,
    valid_from TIMESTAMP NOT NULL,
    valid_to TIMESTAMP,  -- NULL = current

    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (entity_id) REFERENCES legal_entities(entity_id),

    -- Prevent overlapping prices
    EXCLUDE USING gist (
        product_id WITH =,
        entity_id WITH =,
        currency WITH =,
        tstzrange(valid_from, valid_to, '[)') WITH &&
    )
);

-- Tax rules by jurisdiction (complex regulatory logic)
CREATE TABLE tax_rules (
    rule_id UUID PRIMARY KEY,
    jurisdiction VARCHAR(50) NOT NULL,
    product_category VARCHAR(100),
    tax_type VARCHAR(50) NOT NULL,  -- 'VAT', 'GST', 'sales_tax'
    tax_rate DECIMAL(10,6) NOT NULL,
    valid_from DATE NOT NULL,
    valid_to DATE,

    CHECK (tax_rate >= 0 AND tax_rate <= 1)
);

-- Orders with regulatory compliance
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    entity_id UUID NOT NULL,  -- Selling entity
    order_number VARCHAR(50) UNIQUE NOT NULL,
    order_date TIMESTAMP NOT NULL,
    billing_currency CHAR(3) NOT NULL,
    subtotal_amount DECIMAL(20,8) NOT NULL,
    tax_amount DECIMAL(20,8) NOT NULL,
    total_amount DECIMAL(20,8) NOT NULL,
    exchange_rate DECIMAL(20,8) NOT NULL,
    base_currency_amount DECIMAL(20,8) NOT NULL,  -- Converted to base currency

    -- Compliance fields
    regulatory_review_status VARCHAR(20),
    aml_check_status VARCHAR(20),  -- Anti-money laundering
    export_control_status VARCHAR(20),

    FOREIGN KEY (entity_id) REFERENCES legal_entities(entity_id)
);

-- Line items with tax breakdown
CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY,
    order_id UUID NOT NULL,
    product_id UUID NOT NULL,
    quantity DECIMAL(20,8) NOT NULL,
    unit_price DECIMAL(20,8) NOT NULL,
    currency CHAR(3) NOT NULL,
    subtotal DECIMAL(20,8) NOT NULL,

    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Tax breakdown per line item (audit trail)
CREATE TABLE order_item_taxes (
    tax_detail_id UUID PRIMARY KEY,
    order_item_id UUID NOT NULL,
    tax_rule_id UUID NOT NULL,
    jurisdiction VARCHAR(50) NOT NULL,
    tax_type VARCHAR(50) NOT NULL,
    taxable_amount DECIMAL(20,8) NOT NULL,
    tax_rate DECIMAL(10,6) NOT NULL,
    tax_amount DECIMAL(20,8) NOT NULL,

    FOREIGN KEY (order_item_id) REFERENCES order_items(order_item_id),
    FOREIGN KEY (tax_rule_id) REFERENCES tax_rules(rule_id)
);

-- Compliance audit log
CREATE TABLE compliance_events (
    event_id UUID PRIMARY KEY,
    order_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data JSONB NOT NULL,
    checked_at TIMESTAMP NOT NULL DEFAULT NOW(),
    checked_by VARCHAR(100) NOT NULL,
    result VARCHAR(20) NOT NULL,  -- 'passed', 'failed', 'review_required'

    FOREIGN KEY (order_id) REFERENCES orders(order_id)
) PARTITION BY RANGE (checked_at);

-- Indexes for regulatory queries
CREATE INDEX idx_orders_entity_date ON orders(entity_id, order_date);
CREATE INDEX idx_orders_regulatory_status ON orders(regulatory_review_status)
    WHERE regulatory_review_status = 'pending';
CREATE INDEX idx_compliance_events_date ON compliance_events(checked_at);
```

---

## 11. Tools & Resources

### ER Diagram Tools

**Free/Open Source:**

- **draw.io (diagrams.net)**: Free web-based diagramming
- **Lucidchart** (limited free): Collaborative diagramming
- **dbdiagram.io**: Text-to-diagram tool for quick schemas
- **PlantUML**: Code-based diagrams

**Commercial:**

- **ER/Studio**: Enterprise data modeling
- **Oracle SQL Developer Data Modeler**: Oracle-focused
- **MySQL Workbench**: MySQL ERD and forward/reverse engineering
- **pgModeler**: PostgreSQL-specific modeling

### Schema Design Validation Tools

```bash
# PostgreSQL: Analyze schema design
psql -d mydb -c "\d+ table_name"  # Detailed table info
psql -d mydb -c "\di+ table_name" # Index details

# Check for missing indexes on foreign keys
SELECT
    tc.table_name,
    kcu.column_name,
    tc.constraint_name
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
  ON tc.constraint_name = kcu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND NOT EXISTS (
    SELECT 1 FROM pg_indexes
    WHERE tablename = tc.table_name
      AND indexdef LIKE '%' || kcu.column_name || '%'
  );
```

### Documentation Standards

**Maintain ER diagram documentation:**

- **Version control**: Store diagrams in git alongside code
- **Change log**: Document why relationships changed
- **Business rules**: Annotate cardinality decisions with business justification
- **Review schedule**: Quarterly schema review meetings
- **Tool**: Use Markdown + ASCII diagrams for version-controllable docs

**Example Documentation Template:**

```markdown
# Database Schema: E-Commerce Platform

Version: 2.1.0
Last Updated: 2026-02-03
Reviewed By: Data Architecture Team

## Entity Relationship Overview

[ASCII Diagram or Link to draw.io]

## Business Rules

1. Customer:Address = 1:N (multiple shipping addresses)
   - Validated with Product Team on 2025-12-10
   - Supports business requirement for separate home/office delivery

2. Order:Payment = M:N (split payment support)
   - Added 2026-01-15 for enterprise customers
   - Allows paying with multiple payment methods

## Cardinality Decisions

| Relationship     | Cardinality | Rationale                  | Validated Date |
| ---------------- | ----------- | -------------------------- | -------------- |
| Customer-Address | 1:N         | Multiple ship-to locations | 2025-12-10     |
| Order-Payment    | M:N         | Split payment feature      | 2026-01-15     |

## Migration History

- 2026-01-15: Changed Order-Payment from 1:N to M:N
- 2025-12-01: Added product_categories junction table
```

---

## Summary

ER diagrams are the foundation of database design, translating business requirements into structured data models. Key takeaways:

1. **Three Levels**: Conceptual (business view), Logical (detailed entities), Physical (implementation)
2. **Cardinality Matters**: Validate 1:1, 1:N, M:N assumptions with business experts—mistakes here cost millions
3. **Junction Tables**: Every M:N relationship needs a junction table; give it business meaning
4. **Temporal Relationships**: Model historical relationships with temporal tables (valid_from/valid_to)
5. **Documentation**: Maintain living documentation with business rules and change history
6. **Security by Design**: Separate sensitive entities, implement row-level security, audit trails
7. **Performance**: Index foreign keys, consider denormalization for read-heavy patterns
8. **Avoid Antipatterns**: No multiple nullable FKs, no comma-separated values, no missing junction tables

**Critical Success Factors:**

- Start with conceptual ERD, refine to physical schema
- Validate cardinality with domain experts early and often
- Document business rules alongside technical implementation
- Review and update diagrams as system evolves
- Use ER diagrams as communication tool across teams

Well-designed ER diagrams prevent costly production issues, accelerate development, and create maintainable systems that scale.
