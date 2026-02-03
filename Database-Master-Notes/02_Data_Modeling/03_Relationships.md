# Database Relationships

## 1. Concept Explanation

### What are Database Relationships?

Database relationships define how entities (tables) are connected to each other, representing real-world associations between data. These relationships form the core of relational database design, enabling data integrity, efficient queries, and accurate representation of business logic.

**Core Relationship Types:**

```
┌─────────────────────────────────────────────────────────────┐
│              DATABASE RELATIONSHIP TYPES                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. ONE-TO-ONE (1:1)                                         │
│     User ──1────1── UserProfile                              │
│     Each user has exactly one profile                        │
│                                                               │
│  2. ONE-TO-MANY (1:N)                                        │
│     Customer ──1────N── Order                                │
│     One customer has many orders                             │
│                                                               │
│  3. MANY-TO-MANY (M:N)                                       │
│     Student ──M────N── Course                                │
│     Many students enroll in many courses                     │
│                                                               │
│  4. SELF-REFERENCING                                         │
│     Employee ──N────1── Employee (Manager)                   │
│     Employees report to other employees                      │
│                                                               │
│  5. HIERARCHICAL                                             │
│     Category ──1────N── Category (Parent-Child)              │
│     Categories contain subcategories                         │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Relationship Cardinality

**Cardinality** defines how many instances of one entity can be associated with instances of another entity:

```
Notation Systems:

Chen Notation (ER Diagrams):
  1      = Exactly one
  N, M   = Many
  0..1   = Zero or one (optional)
  1..N   = One or many

Crow's Foot Notation:
  ──|    = Exactly one
  ──○    = Zero or one
  ──<    = Many
  ──○<   = Zero or many

Example:
Customer ──|────○< Order
"One customer can have zero or many orders"
```

### Participation Constraints

**Total Participation (Mandatory):** Every instance must participate in the relationship

```
Employee ═══ WorksFor ═══ Department
(Double line = every employee MUST work for a department)
```

**Partial Participation (Optional):** Instances may or may not participate

```
Employee ─── Manages ─── Department
(Single line = not every employee manages a department)
```

---

## 2. Why It Matters

### Business Impact

**Data Integrity Enforcement:**

- **Referential Integrity**: Prevents orphaned records (orders without customers)
- **Business Rule Enforcement**: Ensures relationships match real-world constraints
- **Consistency**: Related data stays synchronized across tables

**Cost of Wrong Relationships:**

```
┌─────────────────────────────────────────────────────────────┐
│         REAL DISASTERS FROM RELATIONSHIP MISTAKES            │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  HEALTHCARE SYSTEM (2019):                                   │
│  ❌ Assumed: Patient ──1────1── Insurance                    │
│  ✅ Reality: Patient ──1────N── Insurance                    │
│  Result: 6-month migration, $2M cost, compliance violations  │
│                                                               │
│  E-COMMERCE PLATFORM (2020):                                 │
│  ❌ Assumed: Order ──1────1── Payment                        │
│  ✅ Reality: Order ──1────N── Payment (split payments)       │
│  Result: Complete payment system rewrite, 3-month delay      │
│                                                               │
│  SOCIAL NETWORK (2018):                                      │
│  ❌ Design: No relationship indexes on foreign keys          │
│  Result: 5-second query times, $500K infrastructure costs    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Technical Impact

**Query Performance:**

```
Bad: No relationship definition
SELECT * FROM orders WHERE customer_id = ?;
→ Full table scan: 800ms on 50M rows

Good: Foreign key + index
CREATE INDEX idx_orders_customer ON orders(customer_id);
→ Index scan: 3ms on 50M rows

266x performance improvement
```

**Data Consistency:**

```
Bad: No foreign key constraint
DELETE FROM customers WHERE customer_id = ?;
-- Orphaned orders remain in database

Good: Foreign key with ON DELETE CASCADE
FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE;
-- All related orders automatically deleted
```

---

## 3. Internal Working & Implementation

### One-to-One (1:1) Relationships

**Use Cases:**

- Separate sensitive data (User ↔ UserCredentials)
- Split large tables for performance (Product ↔ ProductDetails)
- Optional relationships (Employee ↔ ParkingSpace)

**Implementation:**

```sql
-- User profile (public data)
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Authentication credentials (sensitive, separate table)
CREATE TABLE user_credentials (
    user_id UUID PRIMARY KEY,  -- PK enforces 1:1
    password_hash VARCHAR(255) NOT NULL,
    mfa_secret VARCHAR(255),
    last_password_change TIMESTAMP,
    failed_login_attempts INTEGER DEFAULT 0,

    FOREIGN KEY (user_id)
        REFERENCES users(user_id)
        ON DELETE CASCADE  -- When user deleted, credentials deleted
);

-- Query: Get user with credentials (1:1 JOIN)
SELECT u.*, c.last_password_change
FROM users u
JOIN user_credentials c ON u.user_id = c.user_id
WHERE u.username = ?;
```

**1:1 Implementation Pattern:**

```sql
-- Option 1: Separate tables (most common)
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE employee_parking (
    employee_id UUID PRIMARY KEY,  -- PK + FK = 1:1
    parking_space_number VARCHAR(10) UNIQUE NOT NULL,
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
);

-- Option 2: Nullable FK in either table
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parking_space_id UUID UNIQUE,  -- UNIQUE enforces 1:1
    FOREIGN KEY (parking_space_id) REFERENCES parking_spaces(parking_space_id)
);
```

### One-to-Many (1:N) Relationships

**Most Common Relationship Type:**

- Customer → Orders
- Department → Employees
- Blog Post → Comments

**Implementation:**

```sql
-- "One" side (parent)
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL
);

-- "Many" side (child) - FK on this side
CREATE TABLE orders (
    order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL,  -- Foreign key on "many" side
    order_date TIMESTAMP DEFAULT NOW(),
    total_amount DECIMAL(10,2) NOT NULL,

    FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
        ON DELETE RESTRICT  -- Prevent customer deletion if orders exist
        ON UPDATE CASCADE   -- Update orders if customer_id changes
);

-- Critical: Index the foreign key for JOIN performance
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Query: Get customer with all orders
SELECT c.name, o.order_id, o.order_date, o.total_amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id = ?
ORDER BY o.order_date DESC;

-- Query: Get customers with more than 10 orders
SELECT c.customer_id, c.name, COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name
HAVING COUNT(o.order_id) > 10;
```

**1:N Pattern with Constraints:**

```sql
CREATE TABLE departments (
    department_id UUID PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    max_employees INTEGER DEFAULT 100
);

CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    department_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,

    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

-- Trigger: Enforce max employees per department
CREATE FUNCTION check_department_capacity()
RETURNS TRIGGER AS $$
DECLARE
    current_count INTEGER;
    max_capacity INTEGER;
BEGIN
    SELECT COUNT(*), d.max_employees
    INTO current_count, max_capacity
    FROM employees e
    JOIN departments d ON e.department_id = d.department_id
    WHERE e.department_id = NEW.department_id
    GROUP BY d.max_employees;

    IF current_count >= max_capacity THEN
        RAISE EXCEPTION 'Department at maximum capacity (% employees)', max_capacity;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_department_capacity
    BEFORE INSERT ON employees
    FOR EACH ROW
    EXECUTE FUNCTION check_department_capacity();
```

### Many-to-Many (M:N) Relationships

**Requires Junction Table:**

- Students ↔ Courses
- Products ↔ Categories
- Users ↔ Roles

**Implementation:**

```sql
-- First entity
CREATE TABLE students (
    student_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

-- Second entity
CREATE TABLE courses (
    course_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_code VARCHAR(20) UNIQUE NOT NULL,
    course_name VARCHAR(255) NOT NULL,
    credits INTEGER NOT NULL
);

-- Junction table (associative entity)
CREATE TABLE enrollments (
    enrollment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL,
    course_id UUID NOT NULL,
    enrolled_date DATE NOT NULL DEFAULT CURRENT_DATE,
    grade CHAR(2),  -- Relationship attributes
    status VARCHAR(20) DEFAULT 'active',

    -- Composite unique constraint (prevent duplicate enrollments)
    UNIQUE (student_id, course_id, enrolled_date),

    FOREIGN KEY (student_id)
        REFERENCES students(student_id)
        ON DELETE CASCADE,

    FOREIGN KEY (course_id)
        REFERENCES courses(course_id)
        ON DELETE CASCADE
);

-- Indexes on both foreign keys
CREATE INDEX idx_enrollments_student ON enrollments(student_id);
CREATE INDEX idx_enrollments_course ON enrollments(course_id);
CREATE INDEX idx_enrollments_status ON enrollments(status) WHERE status = 'active';

-- Query: Get all courses for a student
SELECT c.course_code, c.course_name, e.enrolled_date, e.grade
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id
WHERE s.student_id = ?
ORDER BY e.enrolled_date DESC;

-- Query: Get all students in a course
SELECT s.name, s.email, e.grade, e.status
FROM courses c
JOIN enrollments e ON c.course_id = e.course_id
JOIN students s ON e.student_id = s.student_id
WHERE c.course_id = ?
ORDER BY s.name;

-- Query: Popular courses (most enrollments)
SELECT c.course_name, COUNT(e.student_id) AS enrollment_count
FROM courses c
LEFT JOIN enrollments e ON c.course_id = e.course_id
WHERE e.status = 'active'
GROUP BY c.course_id, c.course_name
ORDER BY enrollment_count DESC
LIMIT 10;
```

**M:N with Business Logic:**

```sql
-- E-commerce: Products can belong to multiple categories
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    base_price DECIMAL(10,2) NOT NULL
);

CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL
);

-- Junction table with additional attributes
CREATE TABLE product_categories (
    product_id UUID,
    category_id UUID,
    is_primary BOOLEAN DEFAULT false,  -- Mark primary category
    display_order INTEGER,  -- Control display order in category
    added_date DATE DEFAULT CURRENT_DATE,

    PRIMARY KEY (product_id, category_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE CASCADE
);

-- Constraint: Only one primary category per product
CREATE UNIQUE INDEX idx_product_primary_category
    ON product_categories(product_id)
    WHERE is_primary = true;

-- Trigger: Ensure every product has exactly one primary category
CREATE FUNCTION ensure_primary_category()
RETURNS TRIGGER AS $$
BEGIN
    -- If this is the first category, make it primary
    IF NOT EXISTS (
        SELECT 1 FROM product_categories
        WHERE product_id = NEW.product_id
    ) THEN
        NEW.is_primary := true;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_primary_category
    BEFORE INSERT ON product_categories
    FOR EACH ROW
    EXECUTE FUNCTION ensure_primary_category();
```

### Self-Referencing Relationships

**Use Cases:**

- Organizational hierarchies (Employee → Manager)
- Social networks (User → Followers)
- Category trees (Category → Parent Category)

**Implementation: Employee-Manager Hierarchy**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    manager_id UUID,  -- Self-referencing foreign key

    FOREIGN KEY (manager_id)
        REFERENCES employees(employee_id)
        ON DELETE SET NULL  -- If manager deleted, set to NULL
);

-- Prevent circular references (employee can't be their own manager)
ALTER TABLE employees ADD CONSTRAINT check_not_self_manager
    CHECK (employee_id != manager_id);

-- Query: Get employee with their manager
SELECT
    e.name AS employee_name,
    m.name AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
WHERE e.employee_id = ?;

-- Query: Get all direct reports for a manager
SELECT e.*
FROM employees e
WHERE e.manager_id = ?;

-- Query: Get full reporting hierarchy (recursive CTE)
WITH RECURSIVE org_hierarchy AS (
    -- Anchor: Start with CEO (no manager)
    SELECT
        employee_id,
        name,
        manager_id,
        1 AS level,
        name::TEXT AS hierarchy_path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: Get direct reports
    SELECT
        e.employee_id,
        e.name,
        e.manager_id,
        oh.level + 1,
        oh.hierarchy_path || ' → ' || e.name
    FROM employees e
    JOIN org_hierarchy oh ON e.manager_id = oh.employee_id
)
SELECT * FROM org_hierarchy
ORDER BY level, name;

-- Query: Count levels in hierarchy
WITH RECURSIVE depth_calc AS (
    SELECT employee_id, manager_id, 0 AS depth
    FROM employees WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.manager_id, d.depth + 1
    FROM employees e
    JOIN depth_calc d ON e.manager_id = d.employee_id
)
SELECT MAX(depth) + 1 AS hierarchy_depth FROM depth_calc;
```

**Self-Referencing M:N (Social Network):**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL
);

-- Self-referencing many-to-many (followers/following)
CREATE TABLE user_relationships (
    follower_id UUID NOT NULL,
    following_id UUID NOT NULL,
    followed_at TIMESTAMP DEFAULT NOW(),

    PRIMARY KEY (follower_id, following_id),

    FOREIGN KEY (follower_id)
        REFERENCES users(user_id) ON DELETE CASCADE,

    FOREIGN KEY (following_id)
        REFERENCES users(user_id) ON DELETE CASCADE,

    -- Can't follow yourself
    CHECK (follower_id != following_id)
);

CREATE INDEX idx_relationships_follower ON user_relationships(follower_id);
CREATE INDEX idx_relationships_following ON user_relationships(following_id);

-- Query: Get all followers of a user
SELECT u.username, ur.followed_at
FROM user_relationships ur
JOIN users u ON ur.follower_id = u.user_id
WHERE ur.following_id = ?
ORDER BY ur.followed_at DESC;

-- Query: Get all users this user is following
SELECT u.username, ur.followed_at
FROM user_relationships ur
JOIN users u ON ur.following_id = u.user_id
WHERE ur.follower_id = ?
ORDER BY ur.followed_at DESC;

-- Query: Mutual followers (bidirectional relationship)
SELECT u.username
FROM user_relationships ur1
JOIN user_relationships ur2
    ON ur1.follower_id = ur2.following_id
    AND ur1.following_id = ur2.follower_id
JOIN users u ON u.user_id = ur1.following_id
WHERE ur1.follower_id = ?;

-- Query: Suggest users to follow (2nd degree connections)
WITH first_degree AS (
    SELECT following_id AS user_id
    FROM user_relationships
    WHERE follower_id = ?
)
SELECT DISTINCT u.username, COUNT(*) AS mutual_connections
FROM user_relationships ur
JOIN users u ON ur.following_id = u.user_id
WHERE ur.follower_id IN (SELECT user_id FROM first_degree)
  AND ur.following_id != ?  -- Not the current user
  AND ur.following_id NOT IN (SELECT user_id FROM first_degree)  -- Not already following
GROUP BY u.user_id, u.username
ORDER BY mutual_connections DESC
LIMIT 10;
```

### Hierarchical Relationships (Trees)

**Adjacency List Pattern:**

```sql
CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id UUID,
    level INTEGER NOT NULL DEFAULT 0,

    FOREIGN KEY (parent_id)
        REFERENCES categories(category_id)
        ON DELETE CASCADE
);

-- Query: Get immediate children
SELECT * FROM categories WHERE parent_id = ?;

-- Query: Get full subtree (recursive)
WITH RECURSIVE category_tree AS (
    SELECT category_id, name, parent_id, 0 AS depth
    FROM categories
    WHERE category_id = ?

    UNION ALL

    SELECT c.category_id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT * FROM category_tree ORDER BY depth, name;

-- Query: Get path to root
WITH RECURSIVE path_to_root AS (
    SELECT category_id, name, parent_id, name::TEXT AS path
    FROM categories
    WHERE category_id = ?

    UNION ALL

    SELECT c.category_id, c.name, c.parent_id, c.name || ' ← ' || p.path
    FROM categories c
    JOIN path_to_root p ON p.parent_id = c.category_id
)
SELECT path FROM path_to_root WHERE parent_id IS NULL;
```

**Closure Table Pattern (Efficient Queries):**

```sql
-- Main table
CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- Closure table (stores all ancestor-descendant pairs)
CREATE TABLE category_paths (
    ancestor_id UUID NOT NULL,
    descendant_id UUID NOT NULL,
    depth INTEGER NOT NULL,

    PRIMARY KEY (ancestor_id, descendant_id),
    FOREIGN KEY (ancestor_id) REFERENCES categories(category_id) ON DELETE CASCADE,
    FOREIGN KEY (descendant_id) REFERENCES categories(category_id) ON DELETE CASCADE
);

-- Trigger: Maintain closure table on INSERT
CREATE FUNCTION maintain_category_closure()
RETURNS TRIGGER AS $$
BEGIN
    -- Insert self-reference
    INSERT INTO category_paths (ancestor_id, descendant_id, depth)
    VALUES (NEW.category_id, NEW.category_id, 0);

    -- Insert paths from all ancestors
    INSERT INTO category_paths (ancestor_id, descendant_id, depth)
    SELECT ancestor_id, NEW.category_id, depth + 1
    FROM category_paths
    WHERE descendant_id = NEW.parent_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER category_closure_trigger
    AFTER INSERT ON categories
    FOR EACH ROW
    EXECUTE FUNCTION maintain_category_closure();

-- Query: Get all descendants (O(1) instead of recursive)
SELECT c.*
FROM categories c
JOIN category_paths cp ON c.category_id = cp.descendant_id
WHERE cp.ancestor_id = ?
  AND cp.depth > 0
ORDER BY cp.depth, c.name;

-- Query: Get all ancestors
SELECT c.*
FROM categories c
JOIN category_paths cp ON c.category_id = cp.ancestor_id
WHERE cp.descendant_id = ?
  AND cp.depth > 0
ORDER BY cp.depth DESC;

-- Query: Get immediate children only
SELECT c.*
FROM categories c
JOIN category_paths cp ON c.category_id = cp.descendant_id
WHERE cp.ancestor_id = ?
  AND cp.depth = 1;
```

---

## 4. Best Practices

### 1. Always Index Foreign Keys

**❌ BAD:**

```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
-- Missing index on customer_id!
```

**✅ GOOD:**

```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Always index foreign keys
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- For composite foreign keys, index both directions
CREATE TABLE order_items (
    order_id UUID,
    product_id UUID,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

CREATE INDEX idx_order_items_product ON order_items(product_id);
-- order_id already indexed by PRIMARY KEY
```

### 2. Choose Appropriate ON DELETE Actions

```sql
-- CASCADE: Delete children when parent deleted (weak entities)
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE CASCADE  -- Delete orders when customer deleted
);

CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY,
    order_id UUID NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
        ON DELETE CASCADE  -- Delete items when order deleted
);

-- RESTRICT: Prevent deletion if children exist
CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    category_id UUID NOT NULL,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
        ON DELETE RESTRICT  -- Can't delete category if products exist
);

-- SET NULL: Orphan children (make FK NULL)
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    manager_id UUID,
    FOREIGN KEY (manager_id) REFERENCES employees(employee_id)
        ON DELETE SET NULL  -- If manager deleted, set to NULL
);

-- SET DEFAULT: Set to default value
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    category_id UUID NOT NULL DEFAULT '...',  -- Default category
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
        ON DELETE SET DEFAULT  -- Set to 'Uncategorized' if category deleted
);
```

### 3. Name Junction Tables Meaningfully

**❌ BAD (Generic names):**

```sql
CREATE TABLE student_course (...);
CREATE TABLE user_role (...);
CREATE TABLE product_tag (...);
```

**✅ GOOD (Business meaning):**

```sql
CREATE TABLE enrollments (...);      -- Student-Course relationship
CREATE TABLE role_assignments (...); -- User-Role relationship
CREATE TABLE product_taggings (...); -- Product-Tag relationship
```

### 4. Add Relationship Metadata

**❌ BAD (Minimal junction table):**

```sql
CREATE TABLE user_roles (
    user_id UUID,
    role_id UUID,
    PRIMARY KEY (user_id, role_id)
);
-- When was role assigned? By whom? Still active?
```

**✅ GOOD (Rich junction table):**

```sql
CREATE TABLE role_assignments (
    assignment_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    role_id UUID NOT NULL,
    assigned_by UUID NOT NULL,
    assigned_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true,
    notes TEXT,

    UNIQUE (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (role_id) REFERENCES roles(role_id),
    FOREIGN KEY (assigned_by) REFERENCES users(user_id)
);
```

### 5. Validate Cardinality Assumptions

**Always confirm cardinality with domain experts:**

```sql
-- Developer assumption: One address per customer
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    -- ...
);

-- Reality after production: Customers need multiple addresses!
-- Result: Expensive migration to 1:N relationship

-- ✅ GOOD: Validate early, design for multiple addresses
CREATE TABLE customer_addresses (
    address_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    address_type VARCHAR(20) NOT NULL, -- 'billing', 'shipping', 'home'
    is_default BOOLEAN DEFAULT false,
    street_line1 VARCHAR(255) NOT NULL,
    -- ...
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

---

## 5. Common Mistakes & Antipatterns

### Mistake 1: Storing Relationships as Comma-Separated Values

**❌ BAD:**

```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255),
    category_ids TEXT  -- '123,456,789' ← ANTIPATTERN
);

-- Problems:
-- 1. Can't JOIN with categories
-- 2. No referential integrity
-- 3. String parsing required
-- 4. Can't index efficiently
-- 5. Can't query "all products in category X"
```

**✅ GOOD:**

```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE product_categories (
    product_id UUID,
    category_id UUID,
    PRIMARY KEY (product_id, category_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

-- Proper queries with JOINs
SELECT p.*
FROM products p
JOIN product_categories pc ON p.product_id = pc.product_id
WHERE pc.category_id = ?;
```

### Mistake 2: Missing Indexes on Foreign Keys

**Real-World Disaster: E-Commerce Platform (2020)**

```
Scenario: 50M orders, no index on orders.customer_id

Query: "Get customer orders"
SELECT * FROM orders WHERE customer_id = ?;
→ Full table scan: 5-10 seconds per query
→ Black Friday: 100,000 concurrent users
→ Database CPU: 100%, queries timing out
→ $500K in lost sales

Fix: CREATE INDEX idx_orders_customer ON orders(customer_id);
→ Queries: 5ms instead of 5 seconds
→ 1000x improvement
```

### Mistake 3: Incorrect ON DELETE Actions

**❌ BAD (CASCADE where RESTRICT needed):**

```sql
CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    category_id UUID NOT NULL,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
        ON DELETE CASCADE  -- ❌ Deleting category deletes all products!
);

-- Accidental deletion:
DELETE FROM categories WHERE name = 'Electronics';
-- Oops! Deleted 10,000 products
```

**✅ GOOD:**

```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    category_id UUID NOT NULL,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
        ON DELETE RESTRICT  -- ✅ Prevents accidental deletion
);

-- Now this fails safely:
DELETE FROM categories WHERE name = 'Electronics';
-- ERROR: Cannot delete category with existing products
```

### Mistake 4: Circular Dependencies

**❌ BAD:**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    department_id UUID NOT NULL,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

CREATE TABLE departments (
    department_id UUID PRIMARY KEY,
    manager_id UUID NOT NULL,  -- Department must have manager
    FOREIGN KEY (manager_id) REFERENCES employees(employee_id)
);

-- Chicken-and-egg problem:
-- Can't create employee without department
-- Can't create department without manager (who is an employee)
```

**✅ GOOD (Make one side optional):**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    department_id UUID NOT NULL,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

CREATE TABLE departments (
    department_id UUID PRIMARY KEY,
    manager_id UUID,  -- Optional initially
    FOREIGN KEY (manager_id) REFERENCES employees(employee_id)
);

-- Now we can:
-- 1. Create department without manager
-- 2. Create employee in that department
-- 3. Update department to set manager
```

### Mistake 5: Not Using Junction Tables for M:N

**❌ BAD (Attempting M:N without junction table):**

```sql
-- Trying to store multiple authors in one column
CREATE TABLE books (
    book_id UUID PRIMARY KEY,
    title VARCHAR(255),
    author_ids TEXT  -- 'uuid1,uuid2,uuid3' ← WRONG
);

-- Or even worse: Duplicate rows
INSERT INTO books VALUES (..., 'Book Title', 'Author1');
INSERT INTO books VALUES (..., 'Book Title', 'Author2');
-- Now we have duplicate book data!
```

**✅ GOOD:**

```sql
CREATE TABLE books (
    book_id UUID PRIMARY KEY,
    title VARCHAR(255) NOT NULL
);

CREATE TABLE authors (
    author_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE book_authors (
    book_id UUID,
    author_id UUID,
    author_order INTEGER,  -- First author, second author, etc.
    PRIMARY KEY (book_id, author_id),
    FOREIGN KEY (book_id) REFERENCES books(book_id),
    FOREIGN KEY (author_id) REFERENCES authors(author_id)
);
```

---

## 6. Security Considerations

### 1. Row-Level Security on Relationships

```sql
CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    email VARCHAR(255),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id)
);

CREATE TABLE documents (
    document_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    owner_id UUID NOT NULL,
    content TEXT,
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    FOREIGN KEY (owner_id) REFERENCES users(user_id)
);

-- Row-level security: Users only see their tenant's data
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON documents
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

-- Additional policy: Users only see documents they own or shared with them
CREATE POLICY document_access ON documents
    USING (
        owner_id = current_setting('app.user_id')::UUID
        OR document_id IN (
            SELECT document_id FROM document_shares
            WHERE user_id = current_setting('app.user_id')::UUID
        )
    );
```

### 2. Audit Trail for Relationship Changes

```sql
-- Track relationship changes
CREATE TABLE relationship_audit (
    audit_id UUID PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    parent_table VARCHAR(100) NOT NULL,
    child_table VARCHAR(100) NOT NULL,
    parent_id UUID NOT NULL,
    child_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL,  -- 'LINK', 'UNLINK'
    performed_by UUID NOT NULL,
    performed_at TIMESTAMP DEFAULT NOW(),
    reason TEXT
);

-- Trigger: Audit enrollment changes
CREATE FUNCTION audit_enrollment_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO relationship_audit (
            table_name, parent_table, child_table,
            parent_id, child_id, operation, performed_by
        ) VALUES (
            'enrollments', 'students', 'courses',
            NEW.student_id, NEW.course_id, 'LINK',
            current_setting('app.user_id')::UUID
        );
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO relationship_audit (
            table_name, parent_table, child_table,
            parent_id, child_id, operation, performed_by
        ) VALUES (
            'enrollments', 'students', 'courses',
            OLD.student_id, OLD.course_id, 'UNLINK',
            current_setting('app.user_id')::UUID
        );
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_enrollments
    AFTER INSERT OR DELETE ON enrollments
    FOR EACH ROW
    EXECUTE FUNCTION audit_enrollment_changes();
```

---

## 7. Performance Optimization

### 1. Optimize JOIN Queries

```sql
-- Query: Get orders with customer info (1:N relationship)
EXPLAIN ANALYZE
SELECT o.order_id, o.order_date, c.name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > '2026-01-01';

-- Optimization 1: Ensure indexes exist
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Optimization 2: Covering index (include commonly selected columns)
CREATE INDEX idx_orders_date_customer
    ON orders(order_date, customer_id)
    INCLUDE (order_id, total_amount);

-- Optimization 3: Partial index for recent orders only
CREATE INDEX idx_orders_recent
    ON orders(order_date, customer_id)
    WHERE order_date > NOW() - INTERVAL '1 year';
```

### 2. Denormalize for Read-Heavy Workloads

```sql
-- Normalized (many JOINs)
SELECT
    o.order_id,
    c.name AS customer_name,
    COUNT(oi.order_item_id) AS item_count,
    SUM(oi.subtotal) AS total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_id = ?
GROUP BY o.order_id, c.name;

-- Denormalized (faster reads, slower writes)
CREATE TABLE order_summaries (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    customer_name VARCHAR(100) NOT NULL,  -- Denormalized
    item_count INTEGER NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL,

    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Trigger: Maintain denormalized data
CREATE FUNCTION update_order_summary()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO order_summaries (order_id, customer_id, customer_name, item_count, total_amount, created_at)
    SELECT
        o.order_id,
        o.customer_id,
        c.name,
        COUNT(oi.order_item_id),
        SUM(oi.subtotal),
        o.created_at
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.order_id = NEW.order_id
    GROUP BY o.order_id, o.customer_id, c.name, o.created_at
    ON CONFLICT (order_id) DO UPDATE SET
        item_count = EXCLUDED.item_count,
        total_amount = EXCLUDED.total_amount;

    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER maintain_order_summary
    AFTER INSERT OR UPDATE OR DELETE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION update_order_summary();

-- Now queries are simple and fast
SELECT * FROM order_summaries WHERE order_id = ?;
```

### 3. Partition Large Relationship Tables

```sql
-- Large junction table (100M+ enrollments)
CREATE TABLE enrollments (
    enrollment_id UUID NOT NULL,
    student_id UUID NOT NULL,
    course_id UUID NOT NULL,
    enrolled_date DATE NOT NULL,
    semester VARCHAR(20) NOT NULL,

    PRIMARY KEY (enrolled_date, enrollment_id)
) PARTITION BY RANGE (enrolled_date);

-- Partition by academic year
CREATE TABLE enrollments_2024 PARTITION OF enrollments
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE enrollments_2025 PARTITION OF enrollments
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE enrollments_2026 PARTITION OF enrollments
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- Indexes on each partition
CREATE INDEX idx_enrollments_2026_student ON enrollments_2026(student_id);
CREATE INDEX idx_enrollments_2026_course ON enrollments_2026(course_id);

-- Query benefits from partition pruning
SELECT * FROM enrollments
WHERE enrolled_date BETWEEN '2026-01-01' AND '2026-12-31'
  AND student_id = ?;
-- Only scans enrollments_2026 partition
```

---

## 8. Real-World Use Cases

### Use Case 1: Social Network Relationships

```sql
-- Complex social graph with multiple relationship types
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

-- Following/Followers (M:N self-referencing)
CREATE TABLE user_follows (
    follower_id UUID,
    following_id UUID,
    followed_at TIMESTAMP DEFAULT NOW(),
    notification_enabled BOOLEAN DEFAULT true,
    PRIMARY KEY (follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(user_id) ON DELETE CASCADE,
    CHECK (follower_id != following_id)
);

-- Friendships (symmetric M:N)
CREATE TABLE friendships (
    friendship_id UUID PRIMARY KEY,
    user_id_1 UUID NOT NULL,
    user_id_2 UUID NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    requested_by UUID NOT NULL,
    requested_at TIMESTAMP DEFAULT NOW(),
    accepted_at TIMESTAMP,

    FOREIGN KEY (user_id_1) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id_2) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (requested_by) REFERENCES users(user_id),

    CHECK (user_id_1 < user_id_2),  -- Prevent duplicates
    CHECK (status IN ('pending', 'accepted', 'blocked'))
);

-- Blocks (asymmetric M:N)
CREATE TABLE user_blocks (
    blocker_id UUID,
    blocked_id UUID,
    blocked_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (blocker_id, blocked_id),
    FOREIGN KEY (blocker_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (blocked_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Query: Get friend recommendations (friends of friends)
WITH my_friends AS (
    SELECT
        CASE WHEN user_id_1 = ? THEN user_id_2 ELSE user_id_1 END AS friend_id
    FROM friendships
    WHERE (user_id_1 = ? OR user_id_2 = ?)
      AND status = 'accepted'
),
friends_of_friends AS (
    SELECT
        CASE WHEN f.user_id_1 IN (SELECT friend_id FROM my_friends)
             THEN f.user_id_2
             ELSE f.user_id_1
        END AS suggested_id,
        COUNT(*) AS mutual_friends
    FROM friendships f
    WHERE (f.user_id_1 IN (SELECT friend_id FROM my_friends)
           OR f.user_id_2 IN (SELECT friend_id FROM my_friends))
      AND f.status = 'accepted'
    GROUP BY suggested_id
)
SELECT u.username, fof.mutual_friends
FROM friends_of_friends fof
JOIN users u ON fof.suggested_id = u.user_id
WHERE fof.suggested_id != ?
  AND fof.suggested_id NOT IN (SELECT friend_id FROM my_friends)
  AND fof.suggested_id NOT IN (
      SELECT blocked_id FROM user_blocks WHERE blocker_id = ?
  )
ORDER BY fof.mutual_friends DESC
LIMIT 10;
```

### Use Case 2: Healthcare Patient-Provider Network

```sql
-- Complex temporal relationships
CREATE TABLE patients (
    patient_id UUID PRIMARY KEY,
    mrn VARCHAR(50) UNIQUE NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50)
);

CREATE TABLE providers (
    provider_id UUID PRIMARY KEY,
    npi VARCHAR(10) UNIQUE NOT NULL,
    specialty VARCHAR(100)
);

CREATE TABLE facilities (
    facility_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

-- Patient-Provider assignments (temporal 1:N)
CREATE TABLE patient_provider_assignments (
    assignment_id UUID PRIMARY KEY,
    patient_id UUID NOT NULL,
    provider_id UUID NOT NULL,
    assignment_type VARCHAR(50) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE,

    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (provider_id) REFERENCES providers(provider_id),

    -- Prevent overlapping primary care assignments
    EXCLUDE USING gist (
        patient_id WITH =,
        daterange(start_date, end_date, '[]') WITH &&
    ) WHERE (assignment_type = 'primary_care')
);

-- Provider-Facility relationships (M:N with schedule)
CREATE TABLE provider_schedules (
    schedule_id UUID PRIMARY KEY,
    provider_id UUID NOT NULL,
    facility_id UUID NOT NULL,
    day_of_week INTEGER NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    effective_from DATE NOT NULL,
    effective_to DATE,

    FOREIGN KEY (provider_id) REFERENCES providers(provider_id),
    FOREIGN KEY (facility_id) REFERENCES facilities(facility_id)
);

-- Appointments (complex M:N with time slots)
CREATE TABLE appointments (
    appointment_id UUID PRIMARY KEY,
    patient_id UUID NOT NULL,
    provider_id UUID NOT NULL,
    facility_id UUID NOT NULL,
    appointment_datetime TIMESTAMP NOT NULL,
    duration_minutes INTEGER NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled',

    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (provider_id) REFERENCES providers(provider_id),
    FOREIGN KEY (facility_id) REFERENCES facilities(facility_id),

    CHECK (status IN ('scheduled', 'confirmed', 'completed', 'cancelled', 'no_show'))
);

-- Prevent double-booking
CREATE UNIQUE INDEX idx_no_double_booking
    ON appointments(provider_id, appointment_datetime)
    WHERE status NOT IN ('cancelled', 'no_show');
```

---

## 9. Interview Questions

### Junior Level (0-2 years)

**Q1: What are the three main types of database relationships?**

**Answer:**

1. **One-to-One (1:1)**: Each record in Table A relates to exactly one record in Table B. Example: User ↔ UserProfile. Implemented with a unique foreign key.

2. **One-to-Many (1:N)**: One record in Table A relates to many records in Table B. Example: Customer → Orders. Foreign key goes on the "many" side.

3. **Many-to-Many (M:N)**: Multiple records on both sides. Example: Students ↔ Courses. Requires a junction table.

**Q2: How do you implement a one-to-many relationship in SQL?**

**Answer:**

```sql
-- "One" side (parent)
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY,
    name VARCHAR(100)
);

-- "Many" side (child) with foreign key
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP,

    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Index the foreign key for performance
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

**Q3: Why should you always index foreign keys?**

**Answer:**
Foreign keys are frequently used in JOIN queries. Without an index, the database must perform a full table scan to find matching rows, which is extremely slow for large tables. An index on the foreign key enables fast lookups using index scans instead of full table scans, improving query performance by 100-1000x.

### Mid-Level (2-5 years)

**Q4: How do you implement a many-to-many relationship, and why is a junction table necessary?**

**Answer:**
Many-to-many relationships require a junction (bridge/associative) table because relational databases can't directly store multiple values in a single foreign key column.

```sql
-- Two main entities
CREATE TABLE students (student_id UUID PRIMARY KEY, name VARCHAR(100));
CREATE TABLE courses (course_id UUID PRIMARY KEY, course_name VARCHAR(255));

-- Junction table
CREATE TABLE enrollments (
    enrollment_id UUID PRIMARY KEY,
    student_id UUID NOT NULL,
    course_id UUID NOT NULL,
    enrolled_date DATE,
    grade CHAR(2),

    UNIQUE (student_id, course_id),  -- Prevent duplicate enrollments
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);

-- Index both foreign keys
CREATE INDEX idx_enrollments_student ON enrollments(student_id);
CREATE INDEX idx_enrollments_course ON enrollments(course_id);
```

The junction table:

1. Stores the relationship itself as data
2. Prevents data duplication
3. Enables referential integrity
4. Allows relationship-specific attributes (enrollment_date, grade)

**Q5: What are the different ON DELETE options for foreign keys, and when should you use each?**

**Answer:**

- **CASCADE**: Automatically delete child rows when parent is deleted. Use for weak entities that can't exist without parent (Order → OrderItems).
- **RESTRICT**: Prevent parent deletion if children exist. Use when child independence matters (Category with Products).
- **SET NULL**: Set foreign key to NULL when parent is deleted. Use when child can exist without parent (Employee → Manager).
- **SET DEFAULT**: Set foreign key to default value. Use when there's a sensible default (Product → Category sets to 'Uncategorized').
- **NO ACTION**: Similar to RESTRICT but checked at end of transaction. Use for deferred constraint checking.

**Q6: How do you implement a self-referencing relationship for an organizational hierarchy?**

**Answer:**

```sql
CREATE TABLE employees (
    employee_id UUID PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    manager_id UUID,  -- Self-referencing FK

    FOREIGN KEY (manager_id)
        REFERENCES employees(employee_id)
        ON DELETE SET NULL,  -- If manager deleted, set to NULL

    CHECK (employee_id != manager_id)  -- Can't be own manager
);

-- Query full hierarchy with recursive CTE
WITH RECURSIVE org_chart AS (
    -- Anchor: top-level (CEO)
    SELECT employee_id, name, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: direct reports
    SELECT e.employee_id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart ORDER BY level, name;
```

### Senior Level (5-8 years)

**Q7: How would you optimize a many-to-many relationship query that's performing poorly due to large junction tables?**

**Answer:**
Multiple optimization strategies:

1. **Covering Indexes** (include commonly queried columns):

```sql
CREATE INDEX idx_enrollments_student_course
    ON enrollments(student_id, course_id)
    INCLUDE (enrolled_date, grade, status);
-- Index-only scan, no table access needed
```

2. **Partial Indexes** (index only active records):

```sql
CREATE INDEX idx_active_enrollments
    ON enrollments(student_id, course_id)
    WHERE status = 'active';
-- Smaller index, faster queries for active enrollments
```

3. **Partition Junction Table** (for very large tables):

```sql
CREATE TABLE enrollments (
    student_id UUID,
    course_id UUID,
    enrolled_date DATE,
    PRIMARY KEY (enrolled_date, student_id, course_id)
) PARTITION BY RANGE (enrolled_date);

-- Partition by semester/year
CREATE TABLE enrollments_2026_spring PARTITION OF enrollments
    FOR VALUES FROM ('2026-01-01') TO ('2026-06-01');
```

4. **Denormalize for Read-Heavy Workloads**:

```sql
-- Materialized view for frequent query
CREATE MATERIALIZED VIEW student_course_summary AS
SELECT
    s.student_id,
    s.name,
    COUNT(e.course_id) AS total_courses,
    AVG(CASE e.grade
        WHEN 'A' THEN 4.0 WHEN 'B' THEN 3.0
        WHEN 'C' THEN 2.0 WHEN 'D' THEN 1.0
        ELSE 0 END) AS gpa
FROM students s
LEFT JOIN enrollments e ON s.student_id = e.student_id
GROUP BY s.student_id, s.name;

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY student_course_summary;
```

5. **Use Bloom Filters for Existence Checks**:

```sql
-- PostgreSQL Bloom index for "is student enrolled in any course?"
CREATE INDEX idx_enrollments_bloom
    ON enrollments USING bloom(student_id, course_id);
```

**Q8: Design a relationship model for a multi-tenant SaaS application where customers can share resources across tenants.**

**Answer:**

```sql
-- Tenants
CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

-- Users belong to tenants
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    email VARCHAR(255) NOT NULL,
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    UNIQUE (tenant_id, email)
);

-- Documents belong to primary tenant
CREATE TABLE documents (
    document_id UUID PRIMARY KEY,
    owner_tenant_id UUID NOT NULL,
    created_by UUID NOT NULL,
    title VARCHAR(255) NOT NULL,

    FOREIGN KEY (owner_tenant_id) REFERENCES tenants(tenant_id),
    FOREIGN KEY (created_by) REFERENCES users(user_id)
);

-- Cross-tenant sharing (M:N)
CREATE TABLE document_shares (
    share_id UUID PRIMARY KEY,
    document_id UUID NOT NULL,
    shared_with_tenant_id UUID NOT NULL,
    permission_level VARCHAR(20) NOT NULL,
    shared_by UUID NOT NULL,
    shared_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,

    FOREIGN KEY (document_id) REFERENCES documents(document_id) ON DELETE CASCADE,
    FOREIGN KEY (shared_with_tenant_id) REFERENCES tenants(tenant_id),
    FOREIGN KEY (shared_by) REFERENCES users(user_id),

    UNIQUE (document_id, shared_with_tenant_id),
    CHECK (permission_level IN ('read', 'write', 'admin'))
);

-- Row-level security: Users see documents owned by their tenant OR shared with them
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_document_access ON documents
    USING (
        owner_tenant_id = current_setting('app.current_tenant')::UUID
        OR document_id IN (
            SELECT document_id FROM document_shares
            WHERE shared_with_tenant_id = current_setting('app.current_tenant')::UUID
              AND (expires_at IS NULL OR expires_at > NOW())
        )
    );
```

### Staff/Principal Level (8+ years)

**Q9: Design a relationship model for a global social network with 1B+ users, considering graph database characteristics in a relational database.**

**Answer:**
Hybrid approach combining relational and graph-like optimizations:

```sql
-- Users partitioned by region
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    home_region VARCHAR(20) NOT NULL,  -- 'us-east', 'eu-west', 'ap-south'
    created_at TIMESTAMP NOT NULL
) PARTITION BY LIST (home_region);

CREATE TABLE users_us_east PARTITION OF users FOR VALUES IN ('us-east');
CREATE TABLE users_eu_west PARTITION OF users FOR VALUES IN ('eu-west');
CREATE TABLE users_ap_south PARTITION OF users FOR VALUES IN ('ap-south');

-- Relationships (follower graph) - partitioned by follower
CREATE TABLE user_follows (
    follower_id UUID NOT NULL,
    following_id UUID NOT NULL,
    followed_at TIMESTAMP DEFAULT NOW(),

    PRIMARY KEY (follower_id, following_id)
) PARTITION BY HASH (follower_id);

-- 64 partitions for horizontal scaling
CREATE TABLE user_follows_p00 PARTITION OF user_follows
    FOR VALUES WITH (MODULUS 64, REMAINDER 0);
-- ... create partitions p01 through p63

-- Adjacency list materialization (pre-computed graph paths)
CREATE TABLE user_follow_counts (
    user_id UUID PRIMARY KEY,
    follower_count INTEGER NOT NULL DEFAULT 0,
    following_count INTEGER NOT NULL DEFAULT 0,
    last_updated TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 2-hop cache for friend recommendations (friends of friends)
CREATE TABLE user_2hop_network (
    user_id UUID NOT NULL,
    connected_user_id UUID NOT NULL,
    mutual_connections INTEGER NOT NULL,
    last_computed TIMESTAMP NOT NULL,

    PRIMARY KEY (user_id, connected_user_id)
) PARTITION BY HASH (user_id);

-- Background job: Compute 2-hop network
CREATE FUNCTION compute_2hop_network(target_user UUID)
RETURNS void AS $$
BEGIN
    DELETE FROM user_2hop_network WHERE user_id = target_user;

    INSERT INTO user_2hop_network (user_id, connected_user_id, mutual_connections, last_computed)
    WITH first_degree AS (
        SELECT following_id AS friend_id
        FROM user_follows
        WHERE follower_id = target_user
    ),
    second_degree AS (
        SELECT
            uf.following_id AS suggested_id,
            COUNT(*) AS mutual_count
        FROM user_follows uf
        WHERE uf.follower_id IN (SELECT friend_id FROM first_degree)
          AND uf.following_id != target_user
          AND uf.following_id NOT IN (SELECT friend_id FROM first_degree)
        GROUP BY uf.following_id
    )
    SELECT target_user, suggested_id, mutual_count, NOW()
    FROM second_degree
    WHERE mutual_count >= 3  -- Threshold for recommendations
    ORDER BY mutual_count DESC
    LIMIT 1000;  -- Cap per user
END;
$$ LANGUAGE plpgsql;

-- Graph analytics: Pagerank-style influence score
CREATE TABLE user_influence_scores (
    user_id UUID PRIMARY KEY,
    influence_score DECIMAL(10,8) NOT NULL,
    computed_at TIMESTAMP NOT NULL,

    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Trigger: Update follower counts on follow/unfollow
CREATE FUNCTION update_follow_counts()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE user_follow_counts
        SET following_count = following_count + 1
        WHERE user_id = NEW.follower_id;

        UPDATE user_follow_counts
        SET follower_count = follower_count + 1
        WHERE user_id = NEW.following_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE user_follow_counts
        SET following_count = following_count - 1
        WHERE user_id = OLD.follower_id;

        UPDATE user_follow_counts
        SET follower_count = follower_count - 1
        WHERE user_id = OLD.following_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER maintain_follow_counts
    AFTER INSERT OR DELETE ON user_follows
    FOR EACH ROW
    EXECUTE FUNCTION update_follow_counts();
```

**Key Design Decisions:**

1. **Partitioning Strategy**: Hash partition by follower_id (64 partitions) enables horizontal scaling
2. **Denormalization**: Pre-compute follower/following counts for profile pages
3. **2-Hop Cache**: Materialize friend recommendations asynchronously
4. **Regional Partitioning**: Users partitioned by home region for data locality
5. **Graph Algorithms**: Run PageRank-style influence scores in background jobs
6. **Write Path**: Optimistic - accept follows immediately, compute recommendations async
7. **Read Path**: Serve from denormalized caches, fall back to real-time queries

**Q10: How would you design relationships for a financial trading platform with complex hierarchical products, account structures, and regulatory reporting requirements?**

**Answer:**

```sql
-- Account hierarchy (self-referencing tree)
CREATE TABLE accounts (
    account_id UUID PRIMARY KEY,
    account_number VARCHAR(50) UNIQUE NOT NULL,
    account_type VARCHAR(50) NOT NULL,
    parent_account_id UUID,
    account_level INTEGER NOT NULL,
    is_leaf BOOLEAN NOT NULL DEFAULT true,

    FOREIGN KEY (parent_account_id) REFERENCES accounts(account_id),
    CHECK (account_type IN ('root', 'firm', 'branch', 'client', 'sub_account'))
);

-- Closure table for efficient hierarchy queries
CREATE TABLE account_hierarchy (
    ancestor_id UUID NOT NULL,
    descendant_id UUID NOT NULL,
    depth INTEGER NOT NULL,
    path TEXT NOT NULL,  -- For regulatory reporting

    PRIMARY KEY (ancestor_id, descendant_id),
    FOREIGN KEY (ancestor_id) REFERENCES accounts(account_id),
    FOREIGN KEY (descendant_id) REFERENCES accounts(account_id)
);

-- Product hierarchy (multi-level categorization)
CREATE TABLE product_categories (
    category_id UUID PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL,
    parent_category_id UUID,
    category_level INTEGER NOT NULL,
    regulatory_class VARCHAR(50),  -- For compliance

    FOREIGN KEY (parent_category_id) REFERENCES product_categories(category_id)
);

CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    product_code VARCHAR(50) UNIQUE NOT NULL,
    product_type VARCHAR(50) NOT NULL,
    category_id UUID NOT NULL,
    underlying_asset VARCHAR(50),

    FOREIGN KEY (category_id) REFERENCES product_categories(category_id),
    CHECK (product_type IN ('equity', 'option', 'future', 'forex', 'crypto'))
);

-- Complex relationship: Account positions (temporal, multi-currency)
CREATE TABLE positions (
    position_id UUID PRIMARY KEY,
    account_id UUID NOT NULL,
    product_id UUID NOT NULL,
    quantity DECIMAL(20,8) NOT NULL,
    average_cost DECIMAL(20,8) NOT NULL,
    currency CHAR(3) NOT NULL,
    position_date DATE NOT NULL,

    FOREIGN KEY (account_id) REFERENCES accounts(account_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id),

    -- Unique position per account-product-date
    UNIQUE (account_id, product_id, position_date)
) PARTITION BY RANGE (position_date);

-- Partition by month for regulatory archival
CREATE TABLE positions_2026_01 PARTITION OF positions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Relationship authorization (M:N with temporal validity)
CREATE TABLE account_permissions (
    permission_id UUID PRIMARY KEY,
    account_id UUID NOT NULL,
    user_id UUID NOT NULL,
    permission_type VARCHAR(50) NOT NULL,
    granted_by UUID NOT NULL,
    granted_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP,

    FOREIGN KEY (account_id) REFERENCES accounts(account_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (granted_by) REFERENCES users(user_id),

    CHECK (permission_type IN ('view', 'trade', 'admin', 'compliance_view')),

    -- Prevent expired permissions
    CHECK (expires_at IS NULL OR expires_at > granted_at)
);

-- Regulatory reporting: Hierarchical aggregation view
CREATE MATERIALIZED VIEW account_hierarchy_positions AS
WITH RECURSIVE account_tree AS (
    SELECT
        account_id,
        account_id AS root_account_id,
        1 AS level
    FROM accounts
    WHERE parent_account_id IS NULL

    UNION ALL

    SELECT
        a.account_id,
        at.root_account_id,
        at.level + 1
    FROM accounts a
    JOIN account_tree at ON a.parent_account_id = at.account_id
)
SELECT
    at.root_account_id,
    at.account_id,
    p.product_id,
    pc.regulatory_class,
    SUM(p.quantity) AS total_quantity,
    SUM(p.quantity * p.average_cost) AS total_value,
    p.currency
FROM account_tree at
JOIN positions p ON at.account_id = p.account_id
JOIN products pr ON p.product_id = pr.product_id
JOIN product_categories pc ON pr.category_id = pc.category_id
WHERE p.position_date = CURRENT_DATE
GROUP BY at.root_account_id, at.account_id, p.product_id,
         pc.regulatory_class, p.currency;

-- Refresh for daily regulatory reports
CREATE INDEX idx_hierarchy_positions_regulatory
    ON account_hierarchy_positions(root_account_id, regulatory_class);
```

---

## 10. Tools & Resources

### Relationship Visualization Tools

- **dbdiagram.io**: Quick ER diagrams with relationship notation
- **draw.io**: Flexible diagramming with Crow's Foot notation
- **pgAdmin**: PostgreSQL GUI with relationship browser
- **DBeaver**: Universal database tool with ER diagram generation

### Analysis Queries

```sql
-- Find all foreign keys in database
SELECT
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name,
    rc.update_rule,
    rc.delete_rule
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
    ON ccu.constraint_name = tc.constraint_name
JOIN information_schema.referential_constraints AS rc
    ON tc.constraint_name = rc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY';

-- Find missing indexes on foreign keys
SELECT
    tc.table_name,
    kcu.column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND NOT EXISTS (
    SELECT 1
    FROM pg_indexes
    WHERE tablename = tc.table_name
      AND indexdef LIKE '%' || kcu.column_name || '%'
  );

-- Analyze relationship query performance
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.email = 'user@example.com';
```

---

## Summary

Database relationships are the cornerstone of relational database design:

1. **Three Core Types**: 1:1 (rare), 1:N (most common), M:N (requires junction table)
2. **Always Index Foreign Keys**: 100-1000x performance improvement for JOINs
3. **Choose ON DELETE Carefully**: CASCADE for weak entities, RESTRICT for independence
4. **Junction Tables Need Business Meaning**: Name them for what they represent (enrollments, not student_course)
5. **Self-Referencing for Hierarchies**: Use recursive CTEs or closure tables
6. **Validate Cardinality**: Confirm assumptions with domain experts early
7. **Relationship Metadata**: Store when/why relationships were created
8. **Security**: Row-level security on relationships, audit trail for changes

**Critical Success Factors:**

- Model real-world relationships accurately
- Index all foreign keys without exception
- Document cardinality decisions and business rules
- Use appropriate ON DELETE/UPDATE actions
- Optimize large junction tables (partitioning, covering indexes)
- Validate relationship integrity with triggers/constraints

Well-designed relationships ensure data integrity, enable efficient queries, and accurately represent business logic across the entire application lifecycle.
