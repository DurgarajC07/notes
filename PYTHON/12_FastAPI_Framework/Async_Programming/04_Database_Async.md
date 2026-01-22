# 🗄️ Async Database Operations

## Overview

Async database operations enable FastAPI applications to handle multiple database queries concurrently without blocking the event loop. This significantly improves throughput for I/O-bound database operations.

---

## Async Database Drivers

### 1. **SQLAlchemy 2.0 Async**

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker
)
from sqlalchemy import select, update, delete
from sqlalchemy.orm import declarative_base, selectinload
from typing import List, Optional

# Create async engine
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/dbname",
    echo=True,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,  # Verify connections before use
    pool_recycle=3600    # Recycle connections after 1 hour
)

# Create async session factory
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

Base = declarative_base()

# Model Definition
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True, index=True)
    name = Column(String)
    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True)
    title = Column(String)
    content = Column(Text)
    author_id = Column(Integer, ForeignKey("users.id"))
    author = relationship("User", back_populates="posts")

# CRUD Operations
class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(self, email: str, name: str) -> User:
        user = User(email=email, name=name)
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user

    async def get_by_id(self, user_id: int) -> Optional[User]:
        result = await self.session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> Optional[User]:
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def get_all(
        self,
        skip: int = 0,
        limit: int = 100
    ) -> List[User]:
        result = await self.session.execute(
            select(User).offset(skip).limit(limit)
        )
        return result.scalars().all()

    async def update(
        self,
        user_id: int,
        **kwargs
    ) -> Optional[User]:
        await self.session.execute(
            update(User)
            .where(User.id == user_id)
            .values(**kwargs)
        )
        await self.session.commit()
        return await self.get_by_id(user_id)

    async def delete(self, user_id: int) -> bool:
        result = await self.session.execute(
            delete(User).where(User.id == user_id)
        )
        await self.session.commit()
        return result.rowcount > 0
```

### 2. **Eager Loading Relationships**

```python
class UserRepository:
    async def get_with_posts(self, user_id: int) -> Optional[User]:
        """Load user with all posts in single query"""
        result = await self.session.execute(
            select(User)
            .options(selectinload(User.posts))
            .where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_users_with_post_count(self) -> List[dict]:
        """Get users with post count using join"""
        from sqlalchemy import func

        result = await self.session.execute(
            select(
                User.id,
                User.email,
                User.name,
                func.count(Post.id).label("post_count")
            )
            .outerjoin(Post)
            .group_by(User.id)
        )
        return [
            {
                "id": row.id,
                "email": row.email,
                "name": row.name,
                "post_count": row.post_count
            }
            for row in result
        ]
```

---

## FastAPI Integration

### 1. **Dependency Injection Pattern**

```python
from fastapi import FastAPI, Depends, HTTPException
from typing import AsyncGenerator

app = FastAPI()

# Database dependency
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()

# Repository dependency
def get_user_repository(
    session: AsyncSession = Depends(get_db)
) -> UserRepository:
    return UserRepository(session)

# API Endpoints
@app.post("/users/", response_model=UserResponse)
async def create_user(
    user: UserCreate,
    repo: UserRepository = Depends(get_user_repository)
):
    # Check if user exists
    existing = await repo.get_by_email(user.email)
    if existing:
        raise HTTPException(400, "Email already registered")

    return await repo.create(user.email, user.name)

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    repo: UserRepository = Depends(get_user_repository)
):
    user = await repo.get_by_id(user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return user

@app.get("/users/{user_id}/posts", response_model=UserWithPosts)
async def get_user_with_posts(
    user_id: int,
    repo: UserRepository = Depends(get_user_repository)
):
    user = await repo.get_with_posts(user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return user
```

### 2. **Startup and Shutdown Events**

```python
@app.on_event("startup")
async def startup():
    # Create tables
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    print("Database connected and tables created")

@app.on_event("shutdown")
async def shutdown():
    await engine.dispose()
    print("Database connections closed")
```

---

## Transaction Management

### 1. **Manual Transaction Control**

```python
async def transfer_money(
    session: AsyncSession,
    from_account_id: int,
    to_account_id: int,
    amount: float
):
    """Transfer money between accounts with transaction"""
    async with session.begin():
        # Lock rows for update
        from_account = await session.execute(
            select(Account)
            .where(Account.id == from_account_id)
            .with_for_update()  # SELECT FOR UPDATE
        )
        from_account = from_account.scalar_one()

        to_account = await session.execute(
            select(Account)
            .where(Account.id == to_account_id)
            .with_for_update()
        )
        to_account = to_account.scalar_one()

        # Validate balance
        if from_account.balance < amount:
            raise ValueError("Insufficient funds")

        # Perform transfer
        from_account.balance -= amount
        to_account.balance += amount

        # Commit happens automatically at end of context
        # If exception occurs, rollback happens automatically

@app.post("/transfer")
async def transfer(
    transfer: TransferRequest,
    session: AsyncSession = Depends(get_db)
):
    try:
        await transfer_money(
            session,
            transfer.from_account,
            transfer.to_account,
            transfer.amount
        )
        return {"status": "success"}
    except ValueError as e:
        raise HTTPException(400, str(e))
```

### 2. **Nested Transactions (Savepoints)**

```python
async def create_order_with_items(
    session: AsyncSession,
    order_data: dict,
    items: List[dict]
):
    """Create order with items using savepoints"""
    async with session.begin():
        # Create order
        order = Order(**order_data)
        session.add(order)
        await session.flush()  # Get order.id

        for item_data in items:
            # Use savepoint for each item
            async with session.begin_nested():
                try:
                    # Check inventory
                    product = await session.execute(
                        select(Product)
                        .where(Product.id == item_data["product_id"])
                        .with_for_update()
                    )
                    product = product.scalar_one()

                    if product.stock < item_data["quantity"]:
                        raise ValueError(
                            f"Insufficient stock for {product.name}"
                        )

                    # Create order item
                    order_item = OrderItem(
                        order_id=order.id,
                        **item_data
                    )
                    session.add(order_item)

                    # Update inventory
                    product.stock -= item_data["quantity"]

                except Exception:
                    # This savepoint will rollback
                    # but outer transaction continues
                    raise

        # All items processed, commit order
        return order
```

---

## Connection Pooling

### 1. **Pool Configuration**

```python
from sqlalchemy.pool import NullPool, QueuePool

# Production: Use QueuePool
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/dbname",
    poolclass=QueuePool,
    pool_size=20,           # Number of connections to maintain
    max_overflow=10,        # Extra connections when pool exhausted
    pool_timeout=30,        # Timeout waiting for connection
    pool_recycle=3600,      # Recycle connections after 1 hour
    pool_pre_ping=True,     # Test connections before use
    echo_pool=True          # Log pool events
)

# Testing: Use NullPool (no connection pooling)
test_engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/testdb",
    poolclass=NullPool
)
```

### 2. **Pool Monitoring**

```python
@app.get("/health/database")
async def database_health():
    """Check database connection pool health"""
    pool = engine.pool

    return {
        "status": "healthy",
        "pool_size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "total_connections": pool.size() + pool.overflow()
    }
```

---

## Bulk Operations

### 1. **Bulk Insert**

```python
async def bulk_create_users(
    session: AsyncSession,
    users_data: List[dict]
) -> List[User]:
    """Efficiently create multiple users"""
    # Method 1: Using bulk_insert_mappings (faster)
    await session.execute(
        User.__table__.insert(),
        users_data
    )
    await session.commit()

    # Method 2: Using ORM objects (slower but with validation)
    users = [User(**data) for data in users_data]
    session.add_all(users)
    await session.commit()

    return users

# Usage
@app.post("/users/bulk")
async def bulk_create(
    users: List[UserCreate],
    session: AsyncSession = Depends(get_db)
):
    users_data = [user.dict() for user in users]
    created = await bulk_create_users(session, users_data)
    return {"created_count": len(created)}
```

### 2. **Bulk Update**

```python
from sqlalchemy.dialects.postgresql import insert

async def upsert_users(
    session: AsyncSession,
    users_data: List[dict]
):
    """Insert or update users (PostgreSQL)"""
    stmt = insert(User).values(users_data)

    # On conflict, update
    stmt = stmt.on_conflict_do_update(
        index_elements=["email"],
        set_={
            "name": stmt.excluded.name,
            "updated_at": func.now()
        }
    )

    await session.execute(stmt)
    await session.commit()
```

---

## Raw SQL Queries

### 1. **Execute Raw SQL**

```python
from sqlalchemy import text

async def get_user_statistics(session: AsyncSession):
    """Execute raw SQL with parameters"""
    query = text("""
        SELECT
            DATE_TRUNC('day', created_at) as date,
            COUNT(*) as user_count,
            COUNT(DISTINCT email) as unique_emails
        FROM users
        WHERE created_at >= :start_date
        GROUP BY DATE_TRUNC('day', created_at)
        ORDER BY date DESC
    """)

    result = await session.execute(
        query,
        {"start_date": datetime.now() - timedelta(days=30)}
    )

    return [
        {
            "date": row.date,
            "user_count": row.user_count,
            "unique_emails": row.unique_emails
        }
        for row in result
    ]
```

---

## Performance Optimization

### 1. **Batching and Chunking**

```python
async def process_users_in_batches(
    session: AsyncSession,
    batch_size: int = 1000
):
    """Process large dataset in batches"""
    offset = 0

    while True:
        # Fetch batch
        result = await session.execute(
            select(User)
            .offset(offset)
            .limit(batch_size)
        )
        users = result.scalars().all()

        if not users:
            break

        # Process batch
        for user in users:
            # Do something with user
            await process_user(user)

        offset += batch_size

        # Clear session to free memory
        session.expire_all()
```

### 2. **Query Optimization**

```python
# Bad: N+1 Query Problem
async def get_users_bad(session: AsyncSession):
    users = await session.execute(select(User))
    users = users.scalars().all()

    for user in users:
        # This triggers N additional queries!
        posts = await session.execute(
            select(Post).where(Post.author_id == user.id)
        )
        user.posts = posts.scalars().all()

# Good: Eager Loading
async def get_users_good(session: AsyncSession):
    result = await session.execute(
        select(User)
        .options(selectinload(User.posts))
    )
    return result.scalars().all()

# Better: Select only needed columns
async def get_user_summaries(session: AsyncSession):
    result = await session.execute(
        select(User.id, User.email, User.name)
    )
    return result.all()
```

---

## Error Handling

### 1. **Database-Specific Errors**

```python
from sqlalchemy.exc import (
    IntegrityError,
    DBAPIError,
    OperationalError
)
from asyncpg.exceptions import UniqueViolationError

@app.post("/users/")
async def create_user(
    user: UserCreate,
    session: AsyncSession = Depends(get_db)
):
    try:
        new_user = User(**user.dict())
        session.add(new_user)
        await session.commit()
        await session.refresh(new_user)
        return new_user

    except IntegrityError as e:
        await session.rollback()
        if "unique constraint" in str(e).lower():
            raise HTTPException(400, "Email already exists")
        raise HTTPException(400, "Database integrity error")

    except OperationalError as e:
        await session.rollback()
        # Database connection issue
        raise HTTPException(503, "Database unavailable")

    except Exception as e:
        await session.rollback()
        raise HTTPException(500, "Internal server error")
```

### 2. **Retry Logic**

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type
)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type(OperationalError)
)
async def create_user_with_retry(
    session: AsyncSession,
    user_data: dict
):
    """Retry on database connection errors"""
    user = User(**user_data)
    session.add(user)
    await session.commit()
    await session.refresh(user)
    return user
```

---

## Testing Async Database

### 1. **Test Setup with In-Memory Database**

```python
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.ext.asyncio import async_sessionmaker

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

@pytest.fixture
async def test_db():
    """Create test database"""
    engine = create_async_engine(TEST_DATABASE_URL, echo=False)

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    TestSessionLocal = async_sessionmaker(
        engine,
        class_=AsyncSession,
        expire_on_commit=False
    )

    async with TestSessionLocal() as session:
        yield session

    await engine.dispose()

@pytest.mark.asyncio
async def test_create_user(test_db):
    """Test user creation"""
    repo = UserRepository(test_db)

    user = await repo.create(
        email="test@example.com",
        name="Test User"
    )

    assert user.id is not None
    assert user.email == "test@example.com"

    # Verify in database
    fetched = await repo.get_by_email("test@example.com")
    assert fetched.id == user.id
```

---

## Comparison: Sync vs Async Database

| Aspect                 | Sync (Django ORM)      | Async (SQLAlchemy)   |
| ---------------------- | ---------------------- | -------------------- |
| Query Execution        | Blocking               | Non-blocking         |
| Concurrent Requests    | Limited by threads     | High concurrency     |
| Connection Pooling     | Thread-local           | Async-aware          |
| Transaction Management | Auto-commit by default | Explicit control     |
| ORM Features           | Full-featured          | Full-featured (2.0+) |
| Learning Curve         | Lower                  | Higher               |
| Performance            | Good for CPU-bound     | Better for I/O-bound |

```python
# Django Sync
def get_users():
    return User.objects.all()  # Blocks thread

# SQLAlchemy Async
async def get_users(session: AsyncSession):
    result = await session.execute(select(User))
    return result.scalars().all()  # Non-blocking
```

---

## Best Practices

### ✅ Do's:

1. **Use connection pooling** for production
2. **Close sessions** properly in finally blocks
3. **Use transactions** for multi-step operations
4. **Eager load** relationships to avoid N+1 queries
5. **Index frequently** queried columns
6. **Use prepared statements** (automatically done)
7. **Monitor connection pool** health
8. **Set appropriate timeouts**

### ❌ Don'ts:

1. **Don't forget** to await database calls
2. **Don't keep sessions** open longer than needed
3. **Don't execute queries** in loops (N+1 problem)
4. **Don't use blocking** sync code in async context
5. **Don't share sessions** between requests
6. **Don't ignore** database errors

---

## Interview Questions

### Q1: What's the difference between `flush()` and `commit()` in SQLAlchemy?

**Answer**:

- `flush()`: Sends pending changes to the database but doesn't commit the transaction. Useful when you need the database to generate IDs before committing.
- `commit()`: Commits the transaction, making all changes permanent. Automatically flushes first.

```python
# Need ID before commit
session.add(user)
await session.flush()  # user.id is now available
print(user.id)  # Works

await session.commit()  # Make permanent
```

### Q2: How do you prevent N+1 query problems in async SQLAlchemy?

**Answer**: Use eager loading with `selectinload()` or `joinedload()`:

```python
# N+1 Problem (BAD)
users = await session.execute(select(User))
for user in users.scalars():
    posts = await session.execute(
        select(Post).where(Post.author_id == user.id)
    )  # N additional queries!

# Solution (GOOD)
users = await session.execute(
    select(User).options(selectinload(User.posts))
)  # Only 2 queries total
```

### Q3: How do you handle database transactions across multiple operations?

**Answer**: Use `async with session.begin()` for automatic transaction management:

```python
async with session.begin():
    # All operations in one transaction
    user = User(email="test@example.com")
    session.add(user)
    await session.flush()

    profile = Profile(user_id=user.id)
    session.add(profile)

    # Auto-commit on success
    # Auto-rollback on exception
```

### Q4: What's the purpose of `pool_pre_ping` in SQLAlchemy?

**Answer**: It tests connections from the pool before using them, preventing "MySQL has gone away" errors:

```python
engine = create_async_engine(
    url,
    pool_pre_ping=True  # Test before use
)
# Ensures stale connections are recycled
```

### Q5: How do you handle concurrent updates to the same row?

**Answer**: Use `SELECT FOR UPDATE` to lock rows:

```python
async with session.begin():
    result = await session.execute(
        select(Account)
        .where(Account.id == account_id)
        .with_for_update()  # Lock row
    )
    account = result.scalar_one()

    # Safe to update - row is locked
    account.balance += amount
```

---

## Summary

Async database operations in FastAPI provide:

- **High Concurrency**: Handle many database operations simultaneously
- **Non-blocking I/O**: Don't block the event loop during queries
- **Connection Pooling**: Efficient connection reuse
- **Transaction Control**: Proper ACID guarantees
- **ORM Features**: Full SQLAlchemy 2.0 capabilities

Use async database operations when building high-throughput APIs that need to handle many concurrent database queries efficiently. 🚀
