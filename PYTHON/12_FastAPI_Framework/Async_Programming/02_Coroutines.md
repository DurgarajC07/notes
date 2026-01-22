# FastAPI Async: Coroutines & Async/Await

## 📖 Introduction

**Coroutines** are the building blocks of async Python. They are functions defined with `async def` and called with `await`. Understanding coroutines is essential for writing efficient FastAPI applications.

## 🔑 Coroutine Basics

### Defining Coroutines

```python
import asyncio

# Regular function (synchronous)
def sync_function():
    """Synchronous function - blocks"""
    return "Sync result"

# Coroutine (asynchronous)
async def async_function():
    """
    Coroutine - async function

    - Defined with 'async def'
    - Returns coroutine object when called
    - Must be awaited to execute
    """
    return "Async result"

# Calling coroutines
def example():
    # Sync function - executes immediately
    result = sync_function()
    print(result)  # "Sync result"

    # Async function - returns coroutine object
    coro = async_function()
    print(type(coro))  # <class 'coroutine'>

    # Must await or run in event loop
    result = asyncio.run(coro)
    print(result)  # "Async result"
```

### await Keyword

```python
async def fetch_data(source: str) -> dict:
    """
    Simulate async data fetch

    await yields control to event loop
    """
    print(f"Fetching from {source}...")
    await asyncio.sleep(1)  # Simulate I/O - yields control
    print(f"Fetch from {source} complete")
    return {"source": source, "data": f"Data from {source}"}

async def main():
    """
    await executes coroutine and waits for result

    - Pauses current coroutine
    - Allows event loop to run other tasks
    - Resumes when awaited coroutine completes
    """
    # Sequential execution
    result1 = await fetch_data("API-1")
    result2 = await fetch_data("API-2")

    print(f"Results: {result1}, {result2}")

asyncio.run(main())
```

## 🔄 Sequential vs Concurrent

### Sequential Execution

```python
import time

async def task1():
    """Task 1 - takes 2 seconds"""
    print("Task 1 start")
    await asyncio.sleep(2)
    print("Task 1 done")
    return "Result 1"

async def task2():
    """Task 2 - takes 1 second"""
    print("Task 2 start")
    await asyncio.sleep(1)
    print("Task 2 done")
    return "Result 2"

async def sequential():
    """
    Sequential execution - SLOW

    Total time: 3 seconds (2 + 1)
    """
    start = time.time()

    result1 = await task1()  # Wait 2 seconds
    result2 = await task2()  # Wait 1 second

    elapsed = time.time() - start
    print(f"Sequential time: {elapsed:.2f}s")

    return result1, result2

asyncio.run(sequential())
```

### Concurrent Execution

```python
async def concurrent():
    """
    Concurrent execution - FAST

    Total time: 2 seconds (max of 2 and 1)
    """
    start = time.time()

    # Create tasks (start immediately)
    t1 = asyncio.create_task(task1())
    t2 = asyncio.create_task(task2())

    # Wait for both
    result1 = await t1
    result2 = await t2

    elapsed = time.time() - start
    print(f"Concurrent time: {elapsed:.2f}s")

    return result1, result2

asyncio.run(concurrent())
```

### Using asyncio.gather

```python
async def concurrent_gather():
    """
    Concurrent with gather - CLEANEST

    Total time: 2 seconds
    """
    start = time.time()

    # Run concurrently
    results = await asyncio.gather(
        task1(),
        task2()
    )

    elapsed = time.time() - start
    print(f"Gather time: {elapsed:.2f}s")

    return results

asyncio.run(concurrent_gather())
```

## 🌐 Async HTTP Requests

### Using httpx

```python
import httpx

async def fetch_url(url: str) -> dict:
    """
    Async HTTP request

    - Uses httpx for async HTTP
    - Non-blocking I/O
    """
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return {
            "url": url,
            "status": response.status_code,
            "length": len(response.text)
        }

async def fetch_multiple_urls():
    """Fetch multiple URLs concurrently"""
    urls = [
        "https://api.github.com/users/github",
        "https://api.github.com/users/python",
        "https://api.github.com/users/microsoft"
    ]

    # Concurrent requests
    results = await asyncio.gather(
        *[fetch_url(url) for url in urls]
    )

    return results

# FastAPI endpoint
from fastapi import FastAPI

app = FastAPI()

@app.get("/fetch-urls")
async def fetch_urls_endpoint():
    """
    FastAPI endpoint with concurrent HTTP requests

    - Much faster than sequential requests
    - Non-blocking for other requests
    """
    results = await fetch_multiple_urls()
    return {"results": results}
```

## 💾 Async Database Operations

### Using databases Library

```python
from databases import Database

database = Database("postgresql://user:password@localhost/dbname")

async def get_user(user_id: int) -> dict:
    """
    Async database query

    - Non-blocking query
    - Connection pool managed automatically
    """
    query = "SELECT * FROM users WHERE id = :user_id"
    user = await database.fetch_one(query=query, values={"user_id": user_id})

    return dict(user) if user else None

async def get_user_with_posts(user_id: int) -> dict:
    """
    Multiple concurrent queries

    - Fetch user and posts concurrently
    - Faster than sequential
    """
    user_query = "SELECT * FROM users WHERE id = :user_id"
    posts_query = "SELECT * FROM posts WHERE user_id = :user_id"

    # Concurrent queries
    user, posts = await asyncio.gather(
        database.fetch_one(query=user_query, values={"user_id": user_id}),
        database.fetch_all(query=posts_query, values={"user_id": user_id})
    )

    return {
        "user": dict(user) if user else None,
        "posts": [dict(post) for post in posts]
    }

@app.on_event("startup")
async def startup():
    await database.connect()

@app.on_event("shutdown")
async def shutdown():
    await database.disconnect()

@app.get("/users/{user_id}")
async def get_user_endpoint(user_id: int):
    """FastAPI endpoint with async database"""
    data = await get_user_with_posts(user_id)
    return data
```

### Using SQLAlchemy Async

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.ext.asyncio import async_sessionmaker
from sqlalchemy import select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

# Async engine
engine = create_async_engine(
    "postgresql+asyncpg://user:password@localhost/dbname",
    echo=True
)

# Async session factory
async_session = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str]
    email: Mapped[str]

async def get_user_sqlalchemy(user_id: int) -> User:
    """
    Async SQLAlchemy query

    - Uses async session
    - Non-blocking query execution
    """
    async with async_session() as session:
        stmt = select(User).where(User.id == user_id)
        result = await session.execute(stmt)
        user = result.scalar_one_or_none()
        return user

async def create_user_sqlalchemy(username: str, email: str) -> User:
    """
    Async insert with SQLAlchemy

    - Transaction committed asynchronously
    """
    async with async_session() as session:
        user = User(username=username, email=email)
        session.add(user)
        await session.commit()
        await session.refresh(user)
        return user

@app.get("/users-sqlalchemy/{user_id}")
async def get_user_sqlalchemy_endpoint(user_id: int):
    """FastAPI with async SQLAlchemy"""
    user = await get_user_sqlalchemy(user_id)

    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return {
        "id": user.id,
        "username": user.username,
        "email": user.email
    }
```

## 🔧 Async Context Managers

### Basic Async Context Manager

```python
class AsyncResource:
    """
    Async context manager

    - __aenter__ and __aexit__ are async
    - Use 'async with' syntax
    """

    async def __aenter__(self):
        """Setup (async)"""
        print("Acquiring resource...")
        await asyncio.sleep(0.5)
        self.resource = "Resource acquired"
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Cleanup (async)"""
        print("Releasing resource...")
        await asyncio.sleep(0.5)
        self.resource = None
        return False

async def use_async_context_manager():
    """Use async context manager"""
    async with AsyncResource() as resource:
        print(f"Using {resource.resource}")
        await asyncio.sleep(1)

asyncio.run(use_async_context_manager())
```

### Database Connection Manager

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_db_connection():
    """
    Async context manager for database connection

    - Ensures connection is closed
    - Handles exceptions properly
    """
    conn = await database.connect()
    try:
        yield conn
    finally:
        await conn.disconnect()

async def query_with_connection():
    """Use database connection manager"""
    async with get_db_connection() as conn:
        result = await conn.fetch_one("SELECT NOW()")
        return result
```

## 🔄 Async Generators

### Basic Async Generator

```python
async def async_range(n: int):
    """
    Async generator

    - Yields values asynchronously
    - Can await in generator
    - Use 'async for' to iterate
    """
    for i in range(n):
        await asyncio.sleep(0.1)
        yield i

async def consume_async_generator():
    """Consume async generator"""
    async for value in async_range(10):
        print(f"Received: {value}")

asyncio.run(consume_async_generator())
```

### Streaming Response with Async Generator

```python
from fastapi.responses import StreamingResponse

async def generate_csv_data():
    """
    Generate CSV data asynchronously

    - Yields chunks as ready
    - Low memory usage
    """
    # Header
    yield "id,name,price\n"

    # Rows
    for i in range(1000):
        await asyncio.sleep(0.001)  # Simulate DB query
        yield f"{i},Item {i},{i * 10.0}\n"

@app.get("/export-csv")
async def export_csv():
    """
    Stream CSV file

    - Uses async generator
    - Data generated on-the-fly
    """
    return StreamingResponse(
        generate_csv_data(),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=data.csv"}
    )
```

## 🎯 Async Comprehensions

### Async List Comprehension

```python
async def fetch_item(item_id: int) -> dict:
    """Fetch single item"""
    await asyncio.sleep(0.1)
    return {"id": item_id, "name": f"Item {item_id}"}

async def fetch_all_items():
    """
    Async list comprehension

    - Creates list of awaited results
    - Still sequential (awaits each one)
    """
    items = [await fetch_item(i) for i in range(10)]
    return items

async def fetch_all_items_concurrent():
    """
    Concurrent with gather

    - Much faster than list comprehension
    - All requests concurrent
    """
    items = await asyncio.gather(
        *[fetch_item(i) for i in range(10)]
    )
    return items
```

### Async Dict Comprehension

```python
async def fetch_all_items_dict():
    """
    Async dict comprehension

    - Creates dict of awaited results
    """
    item_dict = {
        i: await fetch_item(i)
        for i in range(10)
    }
    return item_dict
```

## 🚀 FastAPI Integration

### Mixing Sync and Async

```python
from fastapi import FastAPI

app = FastAPI()

# Async endpoint
@app.get("/async-endpoint")
async def async_endpoint():
    """
    Async endpoint

    - Runs in event loop
    - Can use await
    - Efficient for I/O
    """
    result = await fetch_data("API")
    return result

# Sync endpoint
@app.get("/sync-endpoint")
def sync_endpoint():
    """
    Sync endpoint

    - Runs in thread pool
    - Cannot use await
    - OK for CPU-bound or blocking code
    """
    import time
    time.sleep(1)  # Blocking call OK here
    return {"message": "Sync response"}

# Async endpoint calling sync code
@app.get("/async-calling-sync")
async def async_calling_sync():
    """
    Async endpoint with blocking code

    - Use run_in_executor for blocking code
    - Keeps event loop free
    """
    loop = asyncio.get_event_loop()

    result = await loop.run_in_executor(
        None,
        sync_blocking_function
    )

    return {"result": result}

def sync_blocking_function():
    """Blocking function"""
    import time
    time.sleep(2)
    return "Blocking result"
```

## ❓ Interview Questions

### Q1: What is a coroutine?

**Answer**:

- Function defined with **async def**
- Returns **coroutine object** when called
- Must be **awaited** to execute
- Yields control during I/O operations

### Q2: What's the difference between await and create_task?

**Answer**:

- **await**: Waits for coroutine to complete (sequential)
- **create_task**: Schedules coroutine (concurrent)
- **await** blocks current coroutine
- **create_task** returns immediately

### Q3: Can you use await in regular functions?

**Answer**:

- **No** - await only works in async functions
- Regular functions are synchronous
- Use **asyncio.run()** to run coroutine from sync code
- Or define function as async

### Q4: How to make multiple async calls concurrently?

**Answer**:

1. **asyncio.gather()**: Simple, returns results in order
2. **asyncio.create_task()**: Manual task management
3. **asyncio.wait()**: More control, returns done/pending
4. All run concurrently in event loop

## 📚 Summary

**Key Takeaways**:

1. **async def** defines coroutines
2. **await** executes and waits for coroutine
3. **Sequential**: await one after another
4. **Concurrent**: create_task or gather
5. **Async context managers**: async with
6. **Async generators**: async for
7. **FastAPI**: Use async for I/O-bound, def for CPU-bound
8. **httpx** for async HTTP requests
9. **databases/SQLAlchemy** for async database
10. **Coroutines don't run** until awaited

Coroutines enable FastAPI's efficient async I/O handling!
