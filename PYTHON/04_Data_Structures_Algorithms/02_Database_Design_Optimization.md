# 🗄️ Database Design and Optimization

## Overview

Proper database design and optimization are crucial for application performance and scalability. Understanding indexing, normalization, and query optimization is essential.

---

## Database Normalization

### Normal Forms

```python
"""
1NF (First Normal Form):
- Atomic values (no repeating groups)
- Each column contains single value
- Unique rows

2NF (Second Normal Form):
- Must be in 1NF
- No partial dependencies
- All non-key attributes depend on entire primary key

3NF (Third Normal Form):
- Must be in 2NF
- No transitive dependencies
- Non-key attributes depend only on primary key

BCNF (Boyce-Codd Normal Form):
- Must be in 3NF
- Every determinant is a candidate key
"""

# Unnormalized (bad)
class OrderUnnormalized(Base):
    """Violates 1NF - repeating groups"""
    __tablename__ = "orders_bad"

    order_id = Column(Integer, primary_key=True)
    customer_name = Column(String)
    customer_email = Column(String)
    product_names = Column(String)  # "Product1,Product2,Product3"
    product_prices = Column(String)  # "10.00,20.00,30.00"

# Normalized (good)
class Customer(Base):
    """Customer table"""
    __tablename__ = "customers"

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    email = Column(String, unique=True, nullable=False)

    orders = relationship("Order", back_populates="customer")

class Order(Base):
    """Order table"""
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True)
    customer_id = Column(Integer, ForeignKey("customers.id"))
    order_date = Column(DateTime, default=datetime.utcnow)

    customer = relationship("Customer", back_populates="orders")
    items = relationship("OrderItem", back_populates="order")

class Product(Base):
    """Product table"""
    __tablename__ = "products"

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    price = Column(Numeric(10, 2), nullable=False)

    order_items = relationship("OrderItem", back_populates="product")

class OrderItem(Base):
    """Order items (many-to-many)"""
    __tablename__ = "order_items"

    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey("orders.id"))
    product_id = Column(Integer, ForeignKey("products.id"))
    quantity = Column(Integer, nullable=False)
    price_at_time = Column(Numeric(10, 2))  # Historical price

    order = relationship("Order", back_populates="items")
    product = relationship("Product", back_populates="order_items")
```

---

## Indexing Strategies

### Index Types

```python
from sqlalchemy import Index, UniqueConstraint

class User(Base):
    """User with various indexes"""
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)  # Automatic index
    email = Column(String, unique=True)     # Unique index
    username = Column(String, index=True)   # Simple index
    first_name = Column(String)
    last_name = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)
    status = Column(String)

    # Composite index
    __table_args__ = (
        Index('idx_name', 'first_name', 'last_name'),
        Index('idx_status_created', 'status', 'created_at'),
    )

class Post(Base):
    """Post with covering index"""
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), index=True)
    title = Column(String)
    content = Column(Text)
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    status = Column(String)

    # Covering index - includes all columns needed for query
    __table_args__ = (
        Index('idx_user_status_created',
              'user_id', 'status', 'created_at',
              postgresql_include=['title']),  # PostgreSQL only
    )

# When to create indexes
"""
✅ CREATE INDEX:
- Columns in WHERE clauses
- Columns in JOIN conditions
- Columns in ORDER BY
- Foreign keys
- Columns used frequently in queries

❌ DON'T INDEX:
- Small tables (<1000 rows)
- Columns with low cardinality (few distinct values)
- Columns frequently updated
- Large text columns
"""
```

### Partial Indexes

```python
from sqlalchemy import text

class Order(Base):
    """Order with partial index"""
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True)
    status = Column(String)
    created_at = Column(DateTime)

    # Partial index - only for active orders
    __table_args__ = (
        Index('idx_active_orders', 'created_at',
              postgresql_where=text("status = 'active'")),
    )

# Creating partial index in migration
"""
CREATE INDEX idx_active_orders
ON orders(created_at)
WHERE status = 'active';
"""
```

---

## Query Optimization

### N+1 Query Problem

```python
# ❌ BAD: N+1 queries
def get_users_with_posts_bad():
    """1 query for users + N queries for posts"""
    users = db.query(User).all()  # 1 query

    for user in users:
        # N queries (one per user)
        posts = user.posts  # Lazy loading triggers query
        print(f"{user.name}: {len(posts)} posts")

# ✅ GOOD: Eager loading with selectinload
from sqlalchemy.orm import selectinload

def get_users_with_posts_good():
    """2 queries total"""
    users = db.query(User)\
        .options(selectinload(User.posts))\
        .all()

    for user in users:
        posts = user.posts  # Already loaded
        print(f"{user.name}: {len(posts)} posts")

# ✅ GOOD: Join with joinedload
from sqlalchemy.orm import joinedload

def get_users_with_posts_joined():
    """1 query with JOIN"""
    users = db.query(User)\
        .options(joinedload(User.posts))\
        .all()

    for user in users:
        posts = user.posts
        print(f"{user.name}: {len(posts)} posts")
```

### Query Only Needed Columns

```python
from sqlalchemy import select

# ❌ BAD: Select all columns
def get_user_names_bad():
    """Loads all user data"""
    users = db.query(User).all()
    return [user.name for user in users]

# ✅ GOOD: Select only needed columns
def get_user_names_good():
    """Only loads name column"""
    result = db.query(User.name).all()
    return [name for (name,) in result]

# ✅ GOOD: Multiple specific columns
def get_user_summary():
    """Load only summary fields"""
    users = db.query(
        User.id,
        User.name,
        User.email
    ).all()

    return [
        {"id": id, "name": name, "email": email}
        for id, name, email in users
    ]
```

### Query Analysis

```python
# PostgreSQL EXPLAIN
def analyze_query(query):
    """Analyze query execution plan"""
    explain_query = f"EXPLAIN ANALYZE {query}"
    result = db.execute(text(explain_query))

    for row in result:
        print(row[0])

# Usage
query = """
SELECT u.*, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id
"""

analyze_query(query)

# Look for:
# - Sequential Scan (should be Index Scan)
# - High cost numbers
# - Large row estimates
# - Nested loops vs Hash joins
```

---

## Database Transactions

### ACID Properties

```python
from sqlalchemy.orm import Session
from sqlalchemy.exc import IntegrityError

def transfer_money(from_user_id: int, to_user_id: int, amount: float):
    """Transfer money between users (ACID transaction)"""
    session = Session()

    try:
        # Start transaction
        from_user = session.query(User).filter(
            User.id == from_user_id
        ).with_for_update().first()  # Lock row

        to_user = session.query(User).filter(
            User.id == to_user_id
        ).with_for_update().first()

        # Validate
        if from_user.balance < amount:
            raise ValueError("Insufficient funds")

        # Update balances
        from_user.balance -= amount
        to_user.balance += amount

        # Commit transaction
        session.commit()

        return True

    except Exception as e:
        # Rollback on error
        session.rollback()
        raise e

    finally:
        session.close()

# Context manager for transactions
from contextlib import contextmanager

@contextmanager
def transaction_scope():
    """Transaction context manager"""
    session = Session()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Usage
def create_order_with_items(order_data, items_data):
    """Create order with items atomically"""
    with transaction_scope() as session:
        # Create order
        order = Order(**order_data)
        session.add(order)
        session.flush()  # Get order.id

        # Create items
        for item_data in items_data:
            item = OrderItem(order_id=order.id, **item_data)
            session.add(item)

        # Commits automatically if no exception
```

### Isolation Levels

```python
from sqlalchemy import create_engine

# Set isolation level
engine = create_engine(
    DATABASE_URL,
    isolation_level="REPEATABLE READ"
)

"""
Isolation Levels (PostgreSQL):
1. READ UNCOMMITTED - Dirty reads possible
2. READ COMMITTED - Default, no dirty reads
3. REPEATABLE READ - Consistent snapshots
4. SERIALIZABLE - Full isolation, may retry

Trade-off: Higher isolation = Lower concurrency
"""

# Handle serialization errors
def safe_transaction():
    """Retry on serialization failure"""
    max_retries = 3

    for attempt in range(max_retries):
        try:
            with transaction_scope() as session:
                # Perform operations
                pass
            break
        except SerializationFailure:
            if attempt == max_retries - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
```

---

## Connection Pooling

### SQLAlchemy Connection Pool

```python
from sqlalchemy.pool import QueuePool

engine = create_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=20,           # Max connections to keep
    max_overflow=10,        # Extra connections when needed
    pool_timeout=30,        # Timeout waiting for connection
    pool_recycle=3600,      # Recycle connections after 1 hour
    pool_pre_ping=True,     # Verify connection before use
    echo_pool=True          # Log pool activity
)

# Monitor pool usage
def get_pool_status():
    """Check connection pool status"""
    pool = engine.pool
    return {
        "size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow()
    }
```

---

## Caching Strategies

### Query Result Caching

```python
import redis
import json
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379)

def cache_query(expire=300):
    """Cache database query results"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generate cache key
            cache_key = f"query:{func.__name__}:{args}:{kwargs}"

            # Try cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Execute query
            result = func(*args, **kwargs)

            # Cache result
            redis_client.setex(
                cache_key,
                expire,
                json.dumps(result)
            )

            return result

        return wrapper
    return decorator

@cache_query(expire=600)
def get_popular_products():
    """Get popular products (cached)"""
    products = db.query(Product)\
        .filter(Product.sales > 1000)\
        .order_by(Product.sales.desc())\
        .limit(10)\
        .all()

    return [{"id": p.id, "name": p.name} for p in products]
```

### Query Caching at DB Level

```python
# PostgreSQL - Materialized Views
"""
-- Create materialized view
CREATE MATERIALIZED VIEW user_stats AS
SELECT
    u.id,
    u.name,
    COUNT(p.id) as post_count,
    COUNT(c.id) as comment_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
LEFT JOIN comments c ON u.id = c.user_id
GROUP BY u.id, u.name;

-- Create index on materialized view
CREATE INDEX idx_user_stats_id ON user_stats(id);

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats;
"""

# Query materialized view
def get_user_stats(user_id):
    """Fast query from materialized view"""
    result = db.execute(text(
        "SELECT * FROM user_stats WHERE id = :user_id"
    ), {"user_id": user_id})

    return result.fetchone()
```

---

## Best Practices

### ✅ Do's:

1. **Normalize data** appropriately (3NF usually)
2. **Index foreign keys** and WHERE columns
3. **Use eager loading** to avoid N+1
4. **Select only needed** columns
5. **Use connection pooling**
6. **Set proper indexes** before large data
7. **Use transactions** for data consistency
8. **Monitor slow queries**
9. **Use EXPLAIN** to analyze queries
10. **Cache frequently accessed** data

### ❌ Don'ts:

1. **Don't over-normalize** (sometimes denormalize for performance)
2. **Don't index everything** (slows writes)
3. **Don't use SELECT \*** in production
4. **Don't forget to close** connections
5. **Don't run long transactions**
6. **Don't ignore query plans**
7. **Don't store large blobs** in database

---

## Interview Questions

### Q1: What is the N+1 query problem?

**Answer**: Loading parent records, then loading children one-by-one:

- Results in 1 + N queries (inefficient)
- Fix with eager loading (selectinload, joinedload)
- Can also use batch loading
  Example: Loading users then their posts in loop

### Q2: Explain database normalization.

**Answer**: Organizing data to reduce redundancy:

- **1NF**: Atomic values, unique rows
- **2NF**: No partial dependencies
- **3NF**: No transitive dependencies
- **BCNF**: Every determinant is candidate key
  Trade-off: Normalization vs performance

### Q3: What's the difference between selectinload and joinedload?

**Answer**:

- **selectinload**: Separate query with IN clause (2 queries)
- **joinedload**: Single query with JOIN
  Use selectinload for one-to-many (avoids cartesian), joinedload for many-to-one

### Q4: What is a covering index?

**Answer**: Index that includes all columns needed for query:

- Query can be satisfied from index alone
- No table access needed
- Much faster for read-heavy queries
  Example: Index on (user_id, created_at) including title

### Q5: Explain ACID properties.

**Answer**:

- **Atomicity**: All or nothing
- **Consistency**: Valid state transitions
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed data persists
  Essential for data integrity in transactions

---

## Summary

Database optimization essentials:

- **Normalization** for data integrity
- **Indexing** for query performance
- **Eager loading** to avoid N+1
- **Transactions** for consistency
- **Connection pooling** for efficiency
- **Caching** for frequently accessed data
- **Query analysis** with EXPLAIN

Optimize queries before adding cache! 🗄️
