# 🗄️ Database Session Management

## Overview

Proper database session management is critical for application reliability, performance, and data integrity. FastAPI's dependency injection system provides an elegant way to manage database sessions.

---

## Session Lifecycle

### 1. **Basic Session Pattern**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.ext.declarative import declarative_base
from typing import Generator

# Database setup
SQLALCHEMY_DATABASE_URL = "postgresql://user:pass@localhost/dbname"

engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Dependency
def get_db() -> Generator[Session, None, None]:
    """
    Database session dependency
    - Creates session
    - Yields session to endpoint
    - Closes session after endpoint completes
    """
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Usage
from fastapi import Depends

@app.get("/users/")
def get_users(db: Session = Depends(get_db)):
    users = db.query(User).all()
    return users
# db.close() called automatically
```

### 2. **Async Session Pattern**

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker
)
from typing import AsyncGenerator

# Async database setup
ASYNC_DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/dbname"

async_engine = create_async_engine(
    ASYNC_DATABASE_URL,
    echo=True,
    pool_size=20,
    max_overflow=10
)

AsyncSessionLocal = async_sessionmaker(
    async_engine,
    class_=AsyncSession,
    expire_on_commit=False
)

# Async dependency
async def get_async_db() -> AsyncGenerator[AsyncSession, None]:
    """Async database session dependency"""
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()

# Usage
@app.get("/users/")
async def get_users(db: AsyncSession = Depends(get_async_db)):
    result = await db.execute(select(User))
    users = result.scalars().all()
    return users
```

---

## Transaction Management

### 1. **Automatic Transaction Control**

```python
async def get_db_with_auto_commit() -> AsyncGenerator[AsyncSession, None]:
    """Session with automatic commit on success"""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()  # Auto-commit if no exception
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()

@app.post("/users/")
async def create_user(
    user: UserCreate,
    db: AsyncSession = Depends(get_db_with_auto_commit)
):
    new_user = User(**user.dict())
    db.add(new_user)
    # Automatically committed after endpoint completes successfully
    return new_user
```

### 2. **Manual Transaction Control**

```python
async def get_db_manual() -> AsyncGenerator[AsyncSession, None]:
    """Session with manual commit control"""
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()

@app.post("/users/")
async def create_user(
    user: UserCreate,
    db: AsyncSession = Depends(get_db_manual)
):
    new_user = User(**user.dict())
    db.add(new_user)

    try:
        await db.commit()  # Explicit commit
        await db.refresh(new_user)
    except IntegrityError:
        await db.rollback()
        raise HTTPException(400, "User already exists")

    return new_user
```

### 3. **Nested Transactions (Savepoints)**

```python
async def create_order_with_items(
    db: AsyncSession,
    order_data: dict,
    items: List[dict]
):
    """Multi-step operation with savepoints"""
    async with db.begin():  # Start transaction
        # Create order
        order = Order(**order_data)
        db.add(order)
        await db.flush()  # Get order.id

        for item_data in items:
            # Use savepoint for each item
            async with db.begin_nested():
                try:
                    product = await db.get(Product, item_data["product_id"])

                    if product.stock < item_data["quantity"]:
                        raise ValueError("Insufficient stock")

                    # Create order item
                    order_item = OrderItem(
                        order_id=order.id,
                        **item_data
                    )
                    db.add(order_item)

                    # Update stock
                    product.stock -= item_data["quantity"]

                except ValueError:
                    # Rollback this item only
                    await db.rollback()
                    raise

        # Commit entire order
        return order

@app.post("/orders/")
async def create_order(
    order: OrderCreate,
    db: AsyncSession = Depends(get_db_manual)
):
    try:
        new_order = await create_order_with_items(
            db,
            order.dict(),
            order.items
        )
        await db.commit()
        return new_order
    except Exception as e:
        await db.rollback()
        raise HTTPException(400, str(e))
```

---

## Connection Pooling

### 1. **Pool Configuration**

```python
from sqlalchemy.pool import QueuePool, NullPool

# Production: Connection pooling
production_engine = create_async_engine(
    ASYNC_DATABASE_URL,
    poolclass=QueuePool,
    pool_size=20,           # Maintain 20 connections
    max_overflow=10,        # Allow 10 extra when pool exhausted
    pool_timeout=30,        # Wait 30s for available connection
    pool_recycle=3600,      # Recycle connections after 1 hour
    pool_pre_ping=True,     # Verify connections before use
    echo_pool=True          # Log pool events
)

# Testing: No connection pooling
test_engine = create_async_engine(
    TEST_DATABASE_URL,
    poolclass=NullPool,     # No pooling
    echo=False
)

# Development: Smaller pool
dev_engine = create_async_engine(
    DEV_DATABASE_URL,
    pool_size=5,
    max_overflow=5,
    echo=True
)
```

### 2. **Pool Monitoring**

```python
@app.get("/health/database")
async def database_health():
    """Monitor connection pool health"""
    pool = async_engine.pool

    return {
        "status": "healthy",
        "pool_size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "total_connections": pool.size() + pool.overflow()
    }

@app.get("/health/database/detailed")
async def database_detailed_health(db: AsyncSession = Depends(get_async_db)):
    """Detailed health check with query"""
    try:
        # Test query
        result = await db.execute(text("SELECT 1"))
        result.scalar_one()

        return {
            "status": "healthy",
            "database": "accessible",
            "pool": {
                "size": async_engine.pool.size(),
                "checked_out": async_engine.pool.checkedout()
            }
        }
    except Exception as e:
        return {
            "status": "unhealthy",
            "error": str(e)
        }
```

---

## Session Patterns

### 1. **Repository Pattern**

```python
class BaseRepository:
    """Base repository with session"""
    def __init__(self, db: AsyncSession):
        self.db = db

class UserRepository(BaseRepository):
    async def get_by_id(self, user_id: int) -> Optional[User]:
        result = await self.db.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> Optional[User]:
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def create(self, user_data: dict) -> User:
        user = User(**user_data)
        self.db.add(user)
        await self.db.flush()
        await self.db.refresh(user)
        return user

    async def update(self, user_id: int, user_data: dict) -> Optional[User]:
        user = await self.get_by_id(user_id)
        if not user:
            return None

        for key, value in user_data.items():
            setattr(user, key, value)

        await self.db.flush()
        await self.db.refresh(user)
        return user

    async def delete(self, user_id: int) -> bool:
        user = await self.get_by_id(user_id)
        if not user:
            return False

        await self.db.delete(user)
        await self.db.flush()
        return True

# Repository dependency
def get_user_repository(
    db: AsyncSession = Depends(get_async_db)
) -> UserRepository:
    return UserRepository(db)

# Usage
@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    repo: UserRepository = Depends(get_user_repository)
):
    user = await repo.get_by_id(user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return user

@app.post("/users/")
async def create_user(
    user: UserCreate,
    repo: UserRepository = Depends(get_user_repository),
    db: AsyncSession = Depends(get_async_db)
):
    # Check duplicate
    existing = await repo.get_by_email(user.email)
    if existing:
        raise HTTPException(400, "Email already registered")

    new_user = await repo.create(user.dict())
    await db.commit()  # Commit after repository operations
    return new_user
```

### 2. **Unit of Work Pattern**

```python
class UnitOfWork:
    """Unit of Work pattern for managing transactions"""
    def __init__(self, session: AsyncSession):
        self.session = session
        self.users = UserRepository(session)
        self.posts = PostRepository(session)
        self.comments = CommentRepository(session)

    async def __aenter__(self):
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            await self.commit()
        else:
            await self.rollback()

    async def commit(self):
        await self.session.commit()

    async def rollback(self):
        await self.session.rollback()

async def get_uow(
    db: AsyncSession = Depends(get_async_db)
) -> UnitOfWork:
    return UnitOfWork(db)

# Usage
@app.post("/posts/")
async def create_post_with_tags(
    post: PostCreate,
    uow: UnitOfWork = Depends(get_uow)
):
    async with uow:
        # Create post
        new_post = await uow.posts.create(post.dict())

        # Create tags
        for tag_name in post.tags:
            tag = await uow.tags.get_or_create(tag_name)
            await uow.post_tags.create(new_post.id, tag.id)

        # All committed or rolled back together
        return new_post
```

---

## Error Handling

### 1. **Database Errors**

```python
from sqlalchemy.exc import (
    IntegrityError,
    OperationalError,
    DataError
)

async def get_db_with_error_handling() -> AsyncGenerator[AsyncSession, None]:
    """Session with comprehensive error handling"""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()

        except IntegrityError as e:
            await session.rollback()
            logger.error(f"Integrity error: {e}")
            raise HTTPException(400, "Database constraint violation")

        except OperationalError as e:
            await session.rollback()
            logger.error(f"Operational error: {e}")
            raise HTTPException(503, "Database unavailable")

        except DataError as e:
            await session.rollback()
            logger.error(f"Data error: {e}")
            raise HTTPException(400, "Invalid data format")

        except Exception as e:
            await session.rollback()
            logger.error(f"Unexpected error: {e}")
            raise HTTPException(500, "Internal server error")

        finally:
            await session.close()
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
async def execute_with_retry(db: AsyncSession, query):
    """Execute query with automatic retry on connection errors"""
    return await db.execute(query)

async def get_db_with_retry() -> AsyncGenerator[AsyncSession, None]:
    """Session with retry on connection errors"""
    max_retries = 3
    retry_count = 0

    while retry_count < max_retries:
        try:
            async with AsyncSessionLocal() as session:
                yield session
                await session.commit()
                break

        except OperationalError as e:
            retry_count += 1
            if retry_count >= max_retries:
                raise HTTPException(503, "Database unavailable")

            logger.warning(f"DB connection error, retry {retry_count}/{max_retries}")
            await asyncio.sleep(2 ** retry_count)  # Exponential backoff

        finally:
            await session.close()
```

---

## Session Best Practices

### 1. **Read-Only Sessions**

```python
async def get_readonly_db() -> AsyncGenerator[AsyncSession, None]:
    """Read-only session for queries"""
    async with AsyncSessionLocal() as session:
        try:
            # Set read-only mode
            await session.execute(text("SET TRANSACTION READ ONLY"))
            yield session
        finally:
            await session.close()

@app.get("/users/")
async def list_users(db: AsyncSession = Depends(get_readonly_db)):
    """Use read-only session for read operations"""
    result = await db.execute(select(User))
    return result.scalars().all()
```

### 2. **Session Isolation Levels**

```python
from sqlalchemy import event

async def get_db_serializable() -> AsyncGenerator[AsyncSession, None]:
    """Session with SERIALIZABLE isolation level"""
    async with AsyncSessionLocal() as session:
        try:
            # Set isolation level
            await session.execute(
                text("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
            )
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()

@app.post("/transfer")
async def transfer_money(
    transfer: TransferRequest,
    db: AsyncSession = Depends(get_db_serializable)
):
    """Use high isolation level for critical transactions"""
    # Transfer logic with serializable isolation
    pass
```

### 3. **Session Timeout**

```python
import asyncio

async def get_db_with_timeout(
    timeout: int = 30
) -> AsyncGenerator[AsyncSession, None]:
    """Session with query timeout"""
    async with AsyncSessionLocal() as session:
        try:
            # Set statement timeout
            await session.execute(
                text(f"SET statement_timeout = '{timeout}s'")
            )
            yield session
            await session.commit()

        except asyncio.TimeoutError:
            await session.rollback()
            raise HTTPException(408, "Database query timeout")

        finally:
            await session.close()
```

---

## Testing Database Sessions

### 1. **Test Database Setup**

```python
import pytest
from sqlalchemy.ext.asyncio import create_async_engine

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

@pytest.fixture
async def test_engine():
    """Create test engine"""
    engine = create_async_engine(TEST_DATABASE_URL, echo=False)

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield engine

    await engine.dispose()

@pytest.fixture
async def test_db(test_engine):
    """Create test database session"""
    TestSessionLocal = async_sessionmaker(
        test_engine,
        class_=AsyncSession,
        expire_on_commit=False
    )

    async with TestSessionLocal() as session:
        yield session
        await session.rollback()

@pytest.mark.asyncio
async def test_create_user(test_db):
    """Test user creation with test database"""
    user = User(email="test@example.com", name="Test")
    test_db.add(user)
    await test_db.commit()

    result = await test_db.execute(
        select(User).where(User.email == "test@example.com")
    )
    fetched = result.scalar_one()

    assert fetched.email == "test@example.com"
```

### 2. **Override Session Dependency**

```python
@pytest.fixture
async def async_client(test_db):
    """Test client with overridden database"""
    async def override_get_db():
        yield test_db

    app.dependency_overrides[get_async_db] = override_get_db

    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

    app.dependency_overrides.clear()

@pytest.mark.asyncio
async def test_endpoint(async_client):
    """Test endpoint with test database"""
    response = await async_client.post(
        "/users/",
        json={"email": "test@example.com", "name": "Test"}
    )

    assert response.status_code == 201
```

---

## Comparison: Django vs FastAPI Session Management

| Aspect             | Django                 | FastAPI                 |
| ------------------ | ---------------------- | ----------------------- |
| Session Creation   | Automatic per request  | Manual with dependency  |
| Transaction        | Auto-commit by default | Explicit control        |
| Connection Pooling | Built-in               | Configure in SQLAlchemy |
| Async Support      | Limited                | Full support            |
| Testing            | Django test database   | Override dependencies   |
| Cleanup            | Automatic              | Yield pattern           |

```python
# Django (automatic)
def my_view(request):
    users = User.objects.all()  # Session automatic
    return JsonResponse({"users": list(users)})

# FastAPI (explicit)
@app.get("/users/")
async def get_users(db: AsyncSession = Depends(get_async_db)):
    result = await db.execute(select(User))
    users = result.scalars().all()
    return {"users": users}
```

---

## Best Practices

### ✅ Do's:

1. **Use dependency injection** for session management
2. **Always close sessions** with try/finally or context managers
3. **Handle database errors** gracefully
4. **Configure connection pooling** appropriately
5. **Use transactions** for multi-step operations
6. **Monitor pool health** in production
7. **Use read-only sessions** for queries
8. **Test with in-memory database**

### ❌ Don'ts:

1. **Don't share sessions** across requests
2. **Don't forget to commit** changes
3. **Don't hold sessions** longer than needed
4. **Don't ignore pool exhaustion**
5. **Don't use blocking** sync code with async sessions
6. **Don't skip error handling**
7. **Don't create sessions** manually in endpoints

---

## Interview Questions

### Q1: Why use dependency injection for database sessions?

**Answer**: Ensures proper session lifecycle management, automatic cleanup, easy testing with dependency overrides, and consistent error handling across endpoints.

### Q2: What's the difference between `flush()` and `commit()`?

**Answer**:

- `flush()`: Sends changes to database but doesn't commit transaction
- `commit()`: Commits transaction, making changes permanent

### Q3: How do you handle database connection pool exhaustion?

**Answer**: Configure appropriate `pool_size` and `max_overflow`, monitor pool health, implement timeouts, and use connection pooling efficiently with proper session cleanup.

### Q4: What's the purpose of `pool_pre_ping`?

**Answer**: Tests connections from pool before use to detect stale connections, preventing "MySQL has gone away" type errors.

### Q5: How do you test endpoints that use database sessions?

**Answer**: Use in-memory test database and override `get_db` dependency:

```python
app.dependency_overrides[get_db] = lambda: test_db
```

---

## Summary

Effective database session management in FastAPI requires:

- **Dependency injection** for session lifecycle
- **Connection pooling** for performance
- **Transaction control** for data integrity
- **Error handling** for reliability
- **Testing strategies** for quality

Proper session management ensures your FastAPI application is robust, performant, and maintainable! 🗄️
