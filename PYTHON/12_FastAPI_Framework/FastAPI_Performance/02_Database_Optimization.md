# 🗄️ Database Query Optimization in FastAPI

## Overview

Database queries are often the primary performance bottleneck in web applications. Optimizing queries, using proper indexing, and implementing efficient data access patterns can dramatically improve application performance.

---

## Query Optimization Basics

### 1. **N+1 Query Problem**

```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship, Session, selectinload, joinedload
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Author(Base):
    __tablename__ = "authors"

    id = Column(Integer, primary_key=True)
    name = Column(String)
    books = relationship("Book", back_populates="author")

class Book(Base):
    __tablename__ = "books"

    id = Column(Integer, primary_key=True)
    title = Column(String)
    author_id = Column(Integer, ForeignKey("authors.id"))
    author = relationship("Author", back_populates="books")

# ❌ BAD: N+1 Query Problem
@app.get("/authors/bad")
async def get_authors_bad(db: Session = Depends(get_db)):
    """N+1 queries - very slow!"""
    authors = db.query(Author).all()  # 1 query

    result = []
    for author in authors:
        result.append({
            "name": author.name,
            "books": [book.title for book in author.books]  # N queries!
        })

    return result

# ✅ GOOD: Using selectinload
@app.get("/authors/good")
async def get_authors_good(db: Session = Depends(get_db)):
    """Single optimized query"""
    authors = db.query(Author).options(
        selectinload(Author.books)  # Load books in separate query
    ).all()

    result = []
    for author in authors:
        result.append({
            "name": author.name,
            "books": [book.title for book in author.books]
        })

    return result

# ✅ ALTERNATIVE: Using joinedload
@app.get("/authors/joined")
async def get_authors_joined(db: Session = Depends(get_db)):
    """Single query with JOIN"""
    authors = db.query(Author).options(
        joinedload(Author.books)  # Load books with LEFT OUTER JOIN
    ).all()

    result = []
    for author in authors:
        result.append({
            "name": author.name,
            "books": [book.title for book in author.books]
        })

    return result
```

### 2. **Lazy vs Eager Loading**

```python
from sqlalchemy.orm import lazyload, subqueryload

# Lazy loading (default) - load on access
@app.get("/books/lazy/{book_id}")
async def get_book_lazy(book_id: int, db: Session = Depends(get_db)):
    """Lazy loading - separate query when accessing author"""
    book = db.query(Book).filter(Book.id == book_id).first()
    author_name = book.author.name  # Triggers separate query here

    return {"title": book.title, "author": author_name}

# Eager loading with joinedload
@app.get("/books/eager/{book_id}")
async def get_book_eager(book_id: int, db: Session = Depends(get_db)):
    """Eager loading - single query with JOIN"""
    book = db.query(Book).options(
        joinedload(Book.author)
    ).filter(Book.id == book_id).first()

    author_name = book.author.name  # No additional query

    return {"title": book.title, "author": author_name}

# Subquery loading
@app.get("/authors/subquery")
async def get_authors_subquery(db: Session = Depends(get_db)):
    """Load relationships using subquery"""
    authors = db.query(Author).options(
        subqueryload(Author.books)  # Uses subquery instead of JOIN
    ).all()

    return [
        {
            "name": author.name,
            "book_count": len(author.books)
        }
        for author in authors
    ]
```

### 3. **Select Only Required Columns**

```python
from sqlalchemy import select

# ❌ BAD: Loading all columns
@app.get("/users/all-columns")
async def get_users_all(db: Session = Depends(get_db)):
    """Loads all columns including large text fields"""
    users = db.query(User).all()
    return [{"id": u.id, "name": u.name} for u in users]

# ✅ GOOD: Load only needed columns
@app.get("/users/specific-columns")
async def get_users_specific(db: Session = Depends(get_db)):
    """Load only required columns"""
    users = db.query(User.id, User.name, User.email).all()
    return [
        {"id": user.id, "name": user.name, "email": user.email}
        for user in users
    ]

# ✅ Using SQLAlchemy 2.0 style
@app.get("/users/select-columns")
async def get_users_select(db: Session = Depends(get_db)):
    """SQLAlchemy 2.0 style with select"""
    stmt = select(User.id, User.name, User.email)
    result = db.execute(stmt).all()

    return [
        {"id": row.id, "name": row.name, "email": row.email}
        for row in result
    ]
```

---

## Database Indexing

### 1. **Creating Indexes**

```python
from sqlalchemy import Index

class Product(Base):
    __tablename__ = "products"

    id = Column(Integer, primary_key=True)
    name = Column(String, index=True)  # Simple index
    category = Column(String)
    price = Column(Integer)
    created_at = Column(DateTime, index=True)

    # Composite index
    __table_args__ = (
        Index('idx_category_price', 'category', 'price'),
        Index('idx_name_category', 'name', 'category'),
    )

class Order(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), index=True)
    status = Column(String, index=True)
    created_at = Column(DateTime)

    # Partial index (PostgreSQL)
    __table_args__ = (
        Index(
            'idx_pending_orders',
            'user_id', 'created_at',
            postgresql_where=(status == 'pending')
        ),
    )
```

### 2. **Query Analysis**

```python
from sqlalchemy import text

@app.get("/debug/explain")
async def explain_query(db: Session = Depends(get_db)):
    """Analyze query execution plan"""
    # Get query
    query = db.query(Product).filter(
        Product.category == "Electronics",
        Product.price > 100
    )

    # Get SQL
    sql = str(query.statement.compile(compile_kwargs={"literal_binds": True}))

    # EXPLAIN ANALYZE (PostgreSQL)
    explain_query = f"EXPLAIN ANALYZE {sql}"
    result = db.execute(text(explain_query)).fetchall()

    return {
        "query": sql,
        "execution_plan": [row[0] for row in result]
    }

@app.get("/debug/query-stats")
async def query_stats(db: Session = Depends(get_db)):
    """Get query statistics"""
    # PostgreSQL specific
    stats_query = text("""
        SELECT
            schemaname,
            tablename,
            seq_scan,
            seq_tup_read,
            idx_scan,
            idx_tup_fetch
        FROM pg_stat_user_tables
        ORDER BY seq_scan DESC
        LIMIT 10
    """)

    result = db.execute(stats_query).fetchall()

    return {
        "tables": [
            {
                "schema": row[0],
                "table": row[1],
                "sequential_scans": row[2],
                "rows_seq_read": row[3],
                "index_scans": row[4],
                "rows_idx_fetched": row[5]
            }
            for row in result
        ]
    }
```

---

## Pagination and Limiting

### 1. **Efficient Pagination**

```python
from typing import Optional
from pydantic import BaseModel

class PaginationParams(BaseModel):
    page: int = 1
    page_size: int = 20

# ❌ BAD: Loading all then slicing
@app.get("/products/bad-pagination")
async def bad_pagination(db: Session = Depends(get_db)):
    """Loads all products into memory"""
    all_products = db.query(Product).all()
    return all_products[:20]  # Only returns 20 but loaded all

# ✅ GOOD: Database-level pagination
@app.get("/products/good-pagination")
async def good_pagination(
    page: int = 1,
    page_size: int = 20,
    db: Session = Depends(get_db)
):
    """Efficient pagination"""
    offset = (page - 1) * page_size

    products = db.query(Product)\
        .offset(offset)\
        .limit(page_size)\
        .all()

    # Get total count
    total = db.query(Product).count()

    return {
        "products": products,
        "total": total,
        "page": page,
        "page_size": page_size,
        "total_pages": (total + page_size - 1) // page_size
    }

# ✅ Cursor-based pagination (for large datasets)
@app.get("/products/cursor-pagination")
async def cursor_pagination(
    cursor: Optional[int] = None,
    limit: int = 20,
    db: Session = Depends(get_db)
):
    """Cursor-based pagination - better for large datasets"""
    query = db.query(Product)

    if cursor:
        query = query.filter(Product.id > cursor)

    products = query.order_by(Product.id).limit(limit).all()

    next_cursor = products[-1].id if products else None

    return {
        "products": products,
        "next_cursor": next_cursor,
        "has_more": len(products) == limit
    }
```

### 2. **Keyset Pagination**

```python
from datetime import datetime

@app.get("/orders/keyset-pagination")
async def keyset_pagination(
    last_id: Optional[int] = None,
    last_created_at: Optional[datetime] = None,
    limit: int = 20,
    db: Session = Depends(get_db)
):
    """Keyset pagination - most efficient for large datasets"""
    query = db.query(Order).order_by(
        Order.created_at.desc(),
        Order.id.desc()
    )

    # Apply keyset filtering
    if last_created_at and last_id:
        query = query.filter(
            (Order.created_at < last_created_at) |
            (
                (Order.created_at == last_created_at) &
                (Order.id < last_id)
            )
        )

    orders = query.limit(limit).all()

    return {
        "orders": orders,
        "next_keyset": {
            "last_id": orders[-1].id,
            "last_created_at": orders[-1].created_at.isoformat()
        } if orders else None
    }
```

---

## Bulk Operations

### 1. **Bulk Insert**

```python
# ❌ BAD: Individual inserts
@app.post("/products/slow-bulk")
async def slow_bulk_insert(
    products: List[dict],
    db: Session = Depends(get_db)
):
    """Slow - commits for each product"""
    for product_data in products:
        product = Product(**product_data)
        db.add(product)
        db.commit()  # Commits each time

    return {"count": len(products)}

# ✅ GOOD: Bulk insert with single commit
@app.post("/products/fast-bulk")
async def fast_bulk_insert(
    products: List[dict],
    db: Session = Depends(get_db)
):
    """Fast - single commit"""
    product_objects = [Product(**data) for data in products]
    db.bulk_save_objects(product_objects)
    db.commit()

    return {"count": len(products)}

# ✅ Even faster: bulk_insert_mappings
@app.post("/products/fastest-bulk")
async def fastest_bulk_insert(
    products: List[dict],
    db: Session = Depends(get_db)
):
    """Fastest - bypasses ORM overhead"""
    db.bulk_insert_mappings(Product, products)
    db.commit()

    return {"count": len(products)}
```

### 2. **Bulk Update**

```python
# ❌ BAD: Individual updates
@app.put("/products/slow-update")
async def slow_bulk_update(
    updates: List[dict],  # [{"id": 1, "price": 100}, ...]
    db: Session = Depends(get_db)
):
    """Slow - N queries"""
    for update_data in updates:
        product = db.query(Product).filter(
            Product.id == update_data["id"]
        ).first()
        product.price = update_data["price"]

    db.commit()
    return {"count": len(updates)}

# ✅ GOOD: Bulk update
@app.put("/products/fast-update")
async def fast_bulk_update(
    updates: List[dict],
    db: Session = Depends(get_db)
):
    """Fast - bulk update"""
    db.bulk_update_mappings(Product, updates)
    db.commit()

    return {"count": len(updates)}

# ✅ Single UPDATE query
@app.put("/products/increase-prices")
async def increase_prices(
    category: str,
    percentage: float,
    db: Session = Depends(get_db)
):
    """Update multiple rows with single query"""
    db.query(Product).filter(
        Product.category == category
    ).update({
        Product.price: Product.price * (1 + percentage / 100)
    })
    db.commit()

    return {"message": "Prices updated"}
```

---

## Connection Pooling

### 1. **Pool Configuration**

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

# Configure connection pool
engine = create_engine(
    "postgresql://user:password@localhost/dbname",
    poolclass=QueuePool,
    pool_size=5,              # Number of connections to maintain
    max_overflow=10,          # Additional connections when pool is full
    pool_timeout=30,          # Timeout waiting for connection
    pool_recycle=3600,        # Recycle connections after 1 hour
    pool_pre_ping=True,       # Verify connections before using
    echo=False                # Don't log SQL queries
)

# Monitor pool status
@app.get("/debug/pool-status")
async def pool_status():
    """Get connection pool statistics"""
    pool = engine.pool

    return {
        "size": pool.size(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "checked_in": pool.size() - pool.checkedout(),
        "timeout": pool.timeout(),
        "recycle": pool._recycle
    }
```

### 2. **Connection Management**

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_db_connection():
    """Properly manage database connections"""
    connection = engine.connect()
    try:
        yield connection
    finally:
        connection.close()

@app.get("/data/with-connection")
async def query_with_connection():
    """Use connection from pool"""
    async with get_db_connection() as conn:
        result = conn.execute(text("SELECT COUNT(*) FROM products"))
        count = result.scalar()

    return {"count": count}
```

---

## Query Caching

### 1. **SQLAlchemy Query Caching**

```python
from sqlalchemy.orm import Query
from typing import Any

class CachedQuery:
    """Wrapper for cached queries"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def execute(
        self,
        query: Query,
        cache_key: str,
        ttl: int = 300
    ) -> Any:
        """Execute query with caching"""
        # Try cache
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # Execute query
        result = query.all()

        # Serialize result
        serialized = [
            {c.name: getattr(row, c.name) for c in row.__table__.columns}
            for row in result
        ]

        # Cache result
        self.redis.setex(cache_key, ttl, json.dumps(serialized))

        return serialized

cached_query = CachedQuery(redis_client)

@app.get("/products/cached")
async def get_products_cached(
    category: str,
    db: Session = Depends(get_db)
):
    """Query with caching"""
    query = db.query(Product).filter(Product.category == category)
    cache_key = f"products:category:{category}"

    products = cached_query.execute(query, cache_key, ttl=600)

    return {"products": products}
```

### 2. **Query Result Cache**

```python
from functools import wraps

def cache_query_result(ttl: int = 300):
    """Decorator to cache query results"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key from function and args
            cache_key = f"query:{func.__name__}:{str(args)}:{str(kwargs)}"

            # Try cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Execute function
            result = await func(*args, **kwargs)

            # Cache result
            redis_client.setex(cache_key, ttl, json.dumps(result))

            return result

        return wrapper
    return decorator

@app.get("/stats/user-activity")
@cache_query_result(ttl=600)
async def get_user_activity(
    user_id: int,
    db: Session = Depends(get_db)
):
    """Expensive query with caching"""
    # Complex query
    result = db.execute(text("""
        SELECT
            DATE(created_at) as date,
            COUNT(*) as activity_count
        FROM user_activities
        WHERE user_id = :user_id
        GROUP BY DATE(created_at)
        ORDER BY date DESC
        LIMIT 30
    """), {"user_id": user_id})

    return [
        {"date": str(row.date), "count": row.activity_count}
        for row in result
    ]
```

---

## Read Replicas

### 1. **Multiple Database Connections**

```python
# Create engines for primary and replica
primary_engine = create_engine("postgresql://primary/db")
replica_engine = create_engine("postgresql://replica/db")

# Session makers
PrimarySession = sessionmaker(bind=primary_engine)
ReplicaSession = sessionmaker(bind=replica_engine)

# Dependencies
def get_primary_db():
    """Get primary database for writes"""
    db = PrimarySession()
    try:
        yield db
    finally:
        db.close()

def get_replica_db():
    """Get replica database for reads"""
    db = ReplicaSession()
    try:
        yield db
    finally:
        db.close()

# Use replica for reads
@app.get("/products/{product_id}")
async def get_product(
    product_id: int,
    db: Session = Depends(get_replica_db)  # Read from replica
):
    """Read operation - use replica"""
    product = db.query(Product).filter(Product.id == product_id).first()
    return product

# Use primary for writes
@app.post("/products")
async def create_product(
    product: ProductCreate,
    db: Session = Depends(get_primary_db)  # Write to primary
):
    """Write operation - use primary"""
    new_product = Product(**product.dict())
    db.add(new_product)
    db.commit()
    db.refresh(new_product)
    return new_product
```

---

## Best Practices

### ✅ Do's:

1. **Use eager loading** to avoid N+1 queries
2. **Create indexes** on frequently queried columns
3. **Use pagination** for large result sets
4. **Select only needed columns**
5. **Use bulk operations** for multiple inserts/updates
6. **Configure connection pooling** appropriately
7. **Cache expensive queries**
8. **Use read replicas** for scaling reads

### ❌ Don'ts:

1. **Don't use `SELECT *`** unnecessarily
2. **Don't load entire tables** into memory
3. **Don't ignore EXPLAIN plans**
4. **Don't commit after each insert** in loops
5. **Don't create too many indexes** (slows writes)
6. **Don't forget pool_pre_ping** for reliability

---

## Interview Questions

### Q1: What is the N+1 query problem?

**Answer**: N+1 occurs when loading a collection triggers N additional queries (one per item). For example, loading 100 authors then accessing books for each triggers 101 queries (1 + 100). Fix with `selectinload()` or `joinedload()`.

### Q2: When should you use joinedload vs selectinload?

**Answer**:

- **joinedload**: One-to-one or few related items. Uses LEFT OUTER JOIN.
- **selectinload**: One-to-many with many items. Uses separate SELECT IN query.
  selectinload is better for large collections to avoid cartesian products.

### Q3: How do you optimize a slow query?

**Answer**:

1. Run EXPLAIN ANALYZE to see execution plan
2. Add indexes on WHERE/JOIN/ORDER BY columns
3. Avoid SELECT \*, only fetch needed columns
4. Use pagination for large result sets
5. Consider query caching for expensive queries
6. Optimize joins and use proper join types

### Q4: What's the difference between offset and cursor pagination?

**Answer**:

- **Offset**: `LIMIT 20 OFFSET 100`. Simple but slow for large offsets.
- **Cursor**: Filter by ID/timestamp `WHERE id > cursor`. Constant performance regardless of page depth.

### Q5: How does connection pooling improve performance?

**Answer**: Connection pooling reuses database connections instead of creating new ones per request. Benefits:

- Reduces connection overhead
- Limits concurrent connections
- Improves response time
- Prevents connection exhaustion

---

## Summary

Database optimization techniques:

- **Eager loading** to eliminate N+1 queries
- **Proper indexing** for query performance
- **Efficient pagination** for large datasets
- **Bulk operations** for multiple records
- **Connection pooling** for resource management
- **Query caching** for expensive operations

Optimized queries are crucial for scalable FastAPI applications! 🗄️
