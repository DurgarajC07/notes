# 🏛️ Domain-Driven Design in Database Modeling

> Domain-Driven Design (DDD) bridges business logic and database structure, creating models that reflect real-world domains. This approach produces maintainable, scalable databases aligned with business needs.

---

## 📖 1. Concept Explanation

### What is Domain-Driven Design?

**Domain-Driven Design** is an approach where the database schema directly reflects the business domain, using concepts like **Entities**, **Value Objects**, **Aggregates**, and **Bounded Contexts**.

**Core DDD Concepts:**
```
DDD BUILDING BLOCKS
│
├── ENTITY
│   └── Object with unique identity (User, Order, Product)
│
├── VALUE OBJECT
│   └── Immutable object without identity (Address, Money, DateRange)
│
├── AGGREGATE
│   └── Cluster of entities with single root (Order + OrderItems)
│
├── AGGREGATE ROOT
│   └── Entry point to aggregate (Order is root for OrderItems)
│
├── REPOSITORY
│   └── Data access abstraction (OrderRepository)
│
└── BOUNDED CONTEXT
    └── Explicit boundary where model applies (Billing, Inventory, Shipping)
```

---

## 🧠 2. Why It Matters in Real Systems

### Without DDD - Anemic Domain Model

```sql
-- Generic, disconnected tables
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    status VARCHAR(20),
    total DECIMAL(10,2)
);

CREATE TABLE order_items (
    id BIGINT PRIMARY KEY,  -- Independent identity ❌
    order_id BIGINT,
    product_id BIGINT,
    quantity INT
);

-- Problems:
-- - OrderItems can exist without Order
-- - No business rules enforced
-- - Unclear relationships
```

### With DDD - Rich Domain Model

```sql
-- Order is AGGREGATE ROOT
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    status ENUM('draft', 'pending', 'paid', 'shipped') DEFAULT 'draft',
    total_amount DECIMAL(10,2) NOT NULL,
    
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    CHECK (total_amount >= 0)
);

-- OrderItems are ENTITY within Order aggregate
CREATE TABLE order_items (
    order_id BIGINT NOT NULL,          -- Part of composite key
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,
    
    PRIMARY KEY (order_id, product_id), -- No independent identity
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id)
);

-- Aggregate boundary enforced:
-- - Can't create OrderItems without Order
-- - Deleting Order cascades to OrderItems
-- - Total calculated from items
```

---

## ⚙️ 3. Internal Working

### Entity vs Value Object

**Entity (has identity):**
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,           -- Unique identity
    email VARCHAR(255) UNIQUE,
    name VARCHAR(100),
    created_at TIMESTAMP
);

-- Two users with same name are DIFFERENT users
-- Identity matters!
```

**Value Object (no identity, immutable):**
```sql
-- Address as value object (embedded in entity)
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100),
    
    -- Address fields (value object)
    address_street VARCHAR(255),
    address_city VARCHAR(100),
    address_state VARCHAR(50),
    address_zip VARCHAR(20),
    address_country CHAR(2)
);

-- Two identical addresses are the SAME
-- No separate identity needed
```

**Or separate table for reusability:**
```sql
CREATE TABLE addresses (
    street VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    zip VARCHAR(20),
    country CHAR(2),
    
    PRIMARY KEY (street, city, state, zip, country)  -- Composite PK (value)
);

CREATE TABLE customer_addresses (
    customer_id BIGINT,
    address_type ENUM('billing', 'shipping'),
    street VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    zip VARCHAR(20),
    country CHAR(2),
    
    PRIMARY KEY (customer_id, address_type),
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE,
    FOREIGN KEY (street, city, state, zip, country) 
        REFERENCES addresses(street, city, state, zip, country)
);
```

---

### Aggregate Pattern

```sql
-- Aggregate Root: Order
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    status ENUM('draft', 'submitted', 'paid', 'cancelled') DEFAULT 'draft',
    subtotal DECIMAL(12,2) NOT NULL DEFAULT 0,
    tax DECIMAL(12,2) NOT NULL DEFAULT 0,
    total DECIMAL(12,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    CHECK (total = subtotal + tax),
    CHECK (subtotal >= 0 AND tax >= 0)
);

-- Aggregate Member: OrderItem (cannot exist independently)
CREATE TABLE order_items (
    order_id BIGINT NOT NULL,
    line_number INT NOT NULL,
    product_id BIGINT NOT NULL,
    product_name VARCHAR(200) NOT NULL,  -- Denormalized for history
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(12,2) NOT NULL,
    
    PRIMARY KEY (order_id, line_number),
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id),
    CHECK (line_total = quantity * unit_price)
);

-- Aggregate Member: OrderShipment
CREATE TABLE order_shipments (
    order_id BIGINT PRIMARY KEY,  -- 1:1 with order
    tracking_number VARCHAR(100),
    carrier VARCHAR(50),
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP,
    
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    CHECK (delivered_at IS NULL OR delivered_at >= shipped_at)
);

-- Rule: Access OrderItems and OrderShipments ONLY through Order
```

---

## ✅ 4. Best Practices

### Design Aggregates with Clear Boundaries

```sql
-- ✅ Good: Order aggregate with clear boundary
-- Root: Order
-- Members: OrderItems, OrderPayment, OrderShipment

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    -- ... order fields
);

CREATE TABLE order_items (
    order_id BIGINT NOT NULL,  -- Part of Order aggregate
    -- ...
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);

-- ❌ Bad: OrderItem pointing to multiple aggregates
CREATE TABLE order_items (
    order_id BIGINT,      -- Part of Order aggregate
    invoice_id BIGINT,    -- Also part of Invoice aggregate? ❌ Ambiguous!
    shipment_id BIGINT    -- Crossing aggregate boundaries ❌
);
```

---

### Embed Value Objects When Appropriate

```sql
-- ✅ Good: Money as value object (embedded)
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(200),
    price_amount DECIMAL(10,2) NOT NULL,
    price_currency CHAR(3) NOT NULL DEFAULT 'USD',
    
    CHECK (price_amount >= 0)
);

-- ❌ Bad: Separate table for simple value object
CREATE TABLE money (
    id BIGINT PRIMARY KEY,  -- Unnecessary identity
    amount DECIMAL(10,2),
    currency CHAR(3)
);

CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(200),
    price_id BIGINT,  -- Overkill for value object
    FOREIGN KEY (price_id) REFERENCES money(id)
);
```

---

### Use Bounded Contexts for Large Domains

```sql
-- Bounded Context: E-Commerce (Sales)
CREATE TABLE ecom_orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    total DECIMAL(10,2),
    -- Focus: Sales process
);

-- Bounded Context: Fulfillment (Shipping)
CREATE TABLE fulfillment_shipments (
    id BIGINT PRIMARY KEY,
    order_id BIGINT,  -- References ecom_orders
    tracking_number VARCHAR(100),
    -- Focus: Physical shipping
);

-- Bounded Context: Accounting (Finance)
CREATE TABLE accounting_invoices (
    id BIGINT PRIMARY KEY,
    order_id BIGINT,  -- References ecom_orders
    amount DECIMAL(12,2),
    -- Focus: Financial records
);

-- Each context has own model of "Order"
-- Different concerns, different tables
```

---

### Enforce Invariants with Constraints

```sql
-- Domain rule: Order total must equal sum of items
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    total DECIMAL(12,2) NOT NULL,
    
    -- Validated by trigger or application
    CHECK (total >= 0)
);

-- Trigger to enforce invariant
CREATE TRIGGER calculate_order_total
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH ROW
BEGIN
    UPDATE orders
    SET total = (
        SELECT COALESCE(SUM(quantity * unit_price), 0)
        FROM order_items
        WHERE order_id = NEW.order_id
    )
    WHERE id = NEW.order_id;
END;
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Anemic Domain Model

```sql
-- ❌ Bad: Just data containers, no business rules
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    status VARCHAR(20),  -- Any string accepted
    total DECIMAL(10,2)   -- Can be negative
);

-- ✅ Good: Enforces business rules
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    status ENUM('draft', 'pending', 'paid', 'shipped', 'cancelled') DEFAULT 'draft',
    total DECIMAL(10,2) NOT NULL CHECK (total >= 0),
    
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

---

### Mistake 2: Violating Aggregate Boundaries

```sql
-- ❌ Bad: Direct access to aggregate members
SELECT * FROM order_items WHERE product_id = 123;  -- Bypasses Order

UPDATE order_items SET quantity = 10 WHERE id = 456;  -- No order validation

-- ✅ Good: Access through aggregate root
SELECT oi.* FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.id = 789 AND oi.product_id = 123;  -- Through Order

-- Application layer handles updates via Order aggregate
```

---

### Mistake 3: Everything is an Entity

```sql
-- ❌ Bad: Address as entity (unnecessary identity)
CREATE TABLE addresses (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,  -- Unnecessary
    street VARCHAR(255),
    city VARCHAR(100),
    zip VARCHAR(20)
);

-- ✅ Good: Address as value object (embedded or composite key)
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100),
    address_street VARCHAR(255),
    address_city VARCHAR(100),
    address_zip VARCHAR(20)
);
```

---

## 🚀 7. Performance Optimization Techniques

### Denormalize Within Aggregates

```sql
-- Denormalize product details in order items
-- (Preserve historical snapshot)
CREATE TABLE order_items (
    order_id BIGINT NOT  NULL,
    product_id BIGINT NOT NULL,
    product_name VARCHAR(200) NOT NULL,    -- Denormalized
    product_sku VARCHAR(50) NOT NULL,      -- Denormalized
    unit_price DECIMAL(10,2) NOT NULL,     -- Denormalized
    quantity INT NOT NULL,
    
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);

-- Benefits:
-- - Order history preserved even if product changes
-- - No joins needed for order details
-- - Aggregate self-contained
```

---

## 🧪 8. Examples

### E-Commerce DDD Model

```sql
-- AGGREGATE: Customer
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- VALUE OBJECT: Address (embedded in customer)
CREATE TABLE customer_addresses (
    customer_id BIGINT NOT NULL,
    address_type ENUM('billing', 'shipping') NOT NULL,
    street VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50),
    postal_code VARCHAR(20) NOT NULL,
    country CHAR(2) NOT NULL,
    
    PRIMARY KEY (customer_id, address_type),
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE
);

-- AGGREGATE ROOT: Order
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id BIGINT NOT NULL,
    status ENUM('cart', 'placed', 'paid', 'shipped', 'delivered', 'cancelled') DEFAULT 'cart',
    subtotal DECIMAL(12,2) NOT NULL DEFAULT 0,
    tax DECIMAL(12,2) NOT NULL DEFAULT 0,
    shipping_fee DECIMAL(10,2) NOT NULL DEFAULT 0,
    total DECIMAL(12,2) NOT NULL DEFAULT 0,
    placed_at TIMESTAMP,
    
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    CHECK (total = subtotal + tax + shipping_fee)
);

-- ENTITY: OrderItem (within Order aggregate)
CREATE TABLE order_items (
    order_id BIGINT NOT NULL,
    line_number INT NOT NULL,
    product_id BIGINT NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(12,2) NOT NULL,
    
    PRIMARY KEY (order_id, line_number),
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    CHECK (line_total = quantity * unit_price)
);

-- ENTITY: OrderPayment (within Order aggregate)
CREATE TABLE order_payments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    payment_method ENUM('credit_card', 'paypal', 'bank_transfer'),
    amount DECIMAL(12,2) NOT NULL,
    status ENUM('pending', 'completed', 'failed', 'refunded'),
    transaction_id VARCHAR(100),
    paid_at TIMESTAMP,
    
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE RESTRICT
);
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study: Uber - Ride Aggregate

```sql
-- AGGREGATE ROOT: Ride
CREATE TABLE rides (
    id BIGINT PRIMARY KEY,
    ride_number VARCHAR(50) UNIQUE NOT NULL,
    passenger_id BIGINT NOT NULL,
    driver_id BIGINT,
    status ENUM('requested', 'accepted', 'in_progress', 'completed', 'cancelled'),
    
    -- VALUE OBJECTS: Pickup/Dropoff locations (embedded)
    pickup_lat DECIMAL(10, 8) NOT NULL,
    pickup_lng DECIMAL(11, 8) NOT NULL,
    pickup_address VARCHAR(255),
    dropoff_lat DECIMAL(10, 8),
    dropoff_lng DECIMAL(11, 8),
    dropoff_address VARCHAR(255),
    
    -- VALUE OBJECT: Fare (embedded)
    fare_base DECIMAL(10,2),
    fare_distance DECIMAL(10,2),
    fare_time DECIMAL(10,2),
    fare_surge DECIMAL(10,2),
    fare_total DECIMAL(10,2),
    
    requested_at TIMESTAMP NOT NULL DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    FOREIGN KEY (passenger_id) REFERENCES users(id),
    FOREIGN KEY (driver_id) REFERENCES drivers(id)
);

-- ENTITY: RideLocation (within Ride aggregate)
CREATE TABLE ride_locations (
    ride_id BIGINT NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    lat DECIMAL(10, 8) NOT NULL,
    lng DECIMAL(11, 8) NOT NULL,
    
    PRIMARY KEY (ride_id, timestamp),
    FOREIGN KEY (ride_id) REFERENCES rides(id) ON DELETE CASCADE
);
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between Entity and Value Object?

**Answer:**

| Aspect | Entity | Value Object |
|--------|---------|--------------|
| **Identity** | Has unique ID | No identity |
| **Mutability** | Can change | Immutable |
| **Equality** | By ID | By value |
| **Lifecycle** | Independent | Part of entity |
| **Examples** | User, Order, Product | Address, Money, DateRange |

```sql
-- Entity: User (has identity)
CREATE TABLE users (
    id BIGINT PRIMARY KEY,  -- Identity
    name VARCHAR(100)       -- Can change
);

-- Value Object: Money (no identity, immutable)
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    price_amount DECIMAL(10,2),  -- Value object
    price_currency CHAR(3)       -- (embedded)
);
```

---

### Q2: What is an Aggregate and why use it?

**Answer:**

**Aggregate** = cluster of related entities/value objects treated as single unit, with one Aggregate Root.

**Benefits:**
- **Consistency boundary**: Changes atomic within aggregate
- **Encapsulation**: Hide internal structure
- **Concurrency control**: Lock at aggregate level
- **Clear ownership**: One root owns all members

**Example:**
```sql
-- Order Aggregate:
-- - Root: Order
-- - Members: OrderItems, OrderPayment, OrderShipment

-- All changes go through Order:
BEGIN TRANSACTION;
    UPDATE orders SET status = 'paid' WHERE id = 123;
    INSERT INTO order_payments VALUES (123, 'credit_card', 50.00);
    UPDATE order_items SET quantity = 2 WHERE order_id = 123;
COMMIT;
```

---

## 📚 Summary

### Key Takeaways

1. **Entity has identity**, Value Object doesn't
2. **Aggregates define consistency boundaries**
3. **Access aggregate members only through root**
4. **Embed value objects when possible**
5. **Use bounded contexts for large domains**
6. **Enforce invariants with constraints/triggers**
7. **Denormalize within aggregates for performance**
8. **Model reflects business domain, not just data**

---

*Last Updated: February 18, 2026*
