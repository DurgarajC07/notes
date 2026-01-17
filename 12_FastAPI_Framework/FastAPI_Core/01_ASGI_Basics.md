# FastAPI Core: ASGI Basics

## 📖 Introduction

**ASGI (Asynchronous Server Gateway Interface)** is the spiritual successor to WSGI, designed for asynchronous Python web applications. FastAPI is built on ASGI, enabling true async/await support and handling thousands of concurrent connections efficiently.

## 🔑 Key Concepts

### WSGI vs ASGI

```python
# WSGI (Django, Flask) - Synchronous
def wsgi_application(environ, start_response):
    """
    WSGI application - blocking I/O

    Problems:
    - Blocks on I/O operations
    - One request per thread/process
    - Limited concurrency
    """
    status = '200 OK'
    headers = [('Content-Type', 'text/plain')]
    start_response(status, headers)

    # Blocking database query
    result = database.query()  # Blocks entire thread

    return [b'Hello World']

# ASGI (FastAPI, Starlette) - Asynchronous
async def asgi_application(scope, receive, send):
    """
    ASGI application - non-blocking I/O

    Benefits:
    - Non-blocking I/O
    - Thousands of concurrent connections
    - WebSocket support
    - HTTP/2 support
    """
    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [(b'content-type', b'text/plain')],
    })

    # Non-blocking database query
    result = await database.query()  # Doesn't block event loop

    await send({
        'type': 'http.response.body',
        'body': b'Hello World',
    })
```

### ASGI Servers

```python
# Uvicorn - Lightning fast ASGI server
"""
Install:
    pip install uvicorn[standard]

Run:
    uvicorn main:app --reload
    uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
"""

# Hypercorn - ASGI server with HTTP/2 support
"""
Install:
    pip install hypercorn

Run:
    hypercorn main:app --bind 0.0.0.0:8000
"""

# Daphne - Django Channels ASGI server
"""
Install:
    pip install daphne

Run:
    daphne main:app
"""
```

## 🚀 FastAPI Application Structure

### Basic Application

```python
from fastapi import FastAPI
from typing import Optional

# Create FastAPI instance
app = FastAPI(
    title="My API",
    description="API with async/await support",
    version="1.0.0",
    docs_url="/docs",  # Swagger UI
    redoc_url="/redoc"  # ReDoc
)

@app.get("/")
async def root():
    """
    Root endpoint

    async def - declares asynchronous function
    FastAPI will await this function
    """
    return {"message": "Hello World"}

@app.get("/items/{item_id}")
async def read_item(item_id: int, q: Optional[str] = None):
    """
    Path parameter and query parameter

    Args:
        item_id: Path parameter (required)
        q: Query parameter (optional)
    """
    return {"item_id": item_id, "q": q}
```

### Sync vs Async Endpoints

```python
import time
import asyncio

@app.get("/sync")
def sync_endpoint():
    """
    Synchronous endpoint

    - Runs in thread pool
    - Blocking operations OK
    - Use for CPU-bound tasks
    """
    time.sleep(1)  # Blocks thread (not event loop)
    return {"message": "Sync response"}

@app.get("/async")
async def async_endpoint():
    """
    Asynchronous endpoint

    - Runs in event loop
    - Must use await for I/O
    - Use for I/O-bound tasks
    """
    await asyncio.sleep(1)  # Doesn't block event loop
    return {"message": "Async response"}

# WRONG - Don't block event loop
@app.get("/wrong")
async def wrong_endpoint():
    """
    BAD: Blocking call in async function

    This blocks the event loop, preventing other requests
    from being processed.
    """
    time.sleep(1)  # DON'T DO THIS
    return {"message": "Wrong"}

# RIGHT - Use await for async operations
@app.get("/right")
async def right_endpoint():
    """
    GOOD: Async call in async function

    Event loop can handle other requests while waiting
    """
    await asyncio.sleep(1)  # CORRECT
    return {"message": "Right"}
```

## 🔄 Event Loop Basics

### Understanding the Event Loop

```python
import asyncio
from typing import List

async def fetch_data(source_id: int) -> dict:
    """
    Simulate async data fetch

    Args:
        source_id: Data source ID

    Returns:
        Fetched data
    """
    print(f"Fetching from source {source_id}...")
    await asyncio.sleep(1)  # Simulate I/O
    return {"source_id": source_id, "data": f"Data from {source_id}"}

@app.get("/sequential")
async def sequential_requests():
    """
    Sequential async requests (SLOW)

    Time: 3 seconds (1 + 1 + 1)
    """
    result1 = await fetch_data(1)
    result2 = await fetch_data(2)
    result3 = await fetch_data(3)

    return {"results": [result1, result2, result3]}

@app.get("/concurrent")
async def concurrent_requests():
    """
    Concurrent async requests (FAST)

    Time: 1 second (all run concurrently)
    """
    # Create tasks
    task1 = asyncio.create_task(fetch_data(1))
    task2 = asyncio.create_task(fetch_data(2))
    task3 = asyncio.create_task(fetch_data(3))

    # Wait for all tasks
    results = await asyncio.gather(task1, task2, task3)

    return {"results": results}

@app.get("/concurrent-gather")
async def concurrent_with_gather():
    """
    Concurrent requests using gather (CLEANER)

    asyncio.gather() runs coroutines concurrently
    """
    results = await asyncio.gather(
        fetch_data(1),
        fetch_data(2),
        fetch_data(3)
    )

    return {"results": results}
```

## 💾 Async Database Operations

### Async Database with databases Library

```python
from databases import Database
from typing import List, Optional

# Async database connection
database = Database("postgresql://user:password@localhost/dbname")

@app.on_event("startup")
async def startup():
    """Connect to database on startup"""
    await database.connect()
    print("Database connected")

@app.on_event("shutdown")
async def shutdown():
    """Disconnect from database on shutdown"""
    await database.disconnect()
    print("Database disconnected")

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """
    Async database query

    - Non-blocking query
    - Event loop handles other requests
    """
    query = "SELECT * FROM users WHERE id = :user_id"
    user = await database.fetch_one(query=query, values={"user_id": user_id})

    if user is None:
        return {"error": "User not found"}

    return dict(user)

@app.get("/users")
async def list_users(limit: int = 10):
    """Fetch multiple users"""
    query = "SELECT * FROM users LIMIT :limit"
    users = await database.fetch_all(query=query, values={"limit": limit})

    return {"users": [dict(user) for user in users]}

@app.post("/users")
async def create_user(name: str, email: str):
    """Create user with async insert"""
    query = """
        INSERT INTO users (name, email, created_at)
        VALUES (:name, :email, NOW())
        RETURNING id, name, email, created_at
    """

    user = await database.fetch_one(
        query=query,
        values={"name": name, "email": email}
    )

    return dict(user)
```

### Async SQLAlchemy 2.0

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.ext.asyncio import async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import select

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
    name: Mapped[str]
    email: Mapped[str]

async def get_session():
    """Dependency for database session"""
    async with async_session() as session:
        yield session

@app.get("/users/{user_id}/sqlalchemy")
async def get_user_sqlalchemy(
    user_id: int,
    session: AsyncSession = Depends(get_session)
):
    """
    Async SQLAlchemy query

    Uses SQLAlchemy 2.0 async API
    """
    stmt = select(User).where(User.id == user_id)
    result = await session.execute(stmt)
    user = result.scalar_one_or_none()

    if user is None:
        raise HTTPException(status_code=404, detail="User not found")

    return {
        "id": user.id,
        "name": user.name,
        "email": user.email
    }
```

## 🌐 Async HTTP Requests

### Using httpx for Async HTTP

```python
import httpx
from typing import List

# Async HTTP client
http_client = httpx.AsyncClient(timeout=10.0)

@app.on_event("startup")
async def startup_http():
    """Create HTTP client"""
    global http_client
    http_client = httpx.AsyncClient(timeout=10.0)

@app.on_event("shutdown")
async def shutdown_http():
    """Close HTTP client"""
    await http_client.aclose()

@app.get("/external-api")
async def call_external_api():
    """
    Call external API asynchronously

    - Non-blocking HTTP request
    - Event loop handles other requests
    """
    response = await http_client.get("https://api.example.com/data")

    return response.json()

@app.get("/multiple-apis")
async def call_multiple_apis():
    """
    Call multiple APIs concurrently

    Time: ~1 second (concurrent) vs ~3 seconds (sequential)
    """
    # Concurrent requests
    responses = await asyncio.gather(
        http_client.get("https://api1.example.com/data"),
        http_client.get("https://api2.example.com/data"),
        http_client.get("https://api3.example.com/data")
    )

    return {
        "results": [r.json() for r in responses]
    }
```

## 🔧 Async Context Managers

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_db_connection():
    """
    Async context manager for database connection

    Ensures connection is properly closed
    """
    conn = await database.connect()
    try:
        yield conn
    finally:
        await conn.disconnect()

@app.get("/with-context")
async def with_context_manager():
    """Use async context manager"""
    async with get_db_connection() as conn:
        result = await conn.fetch_one("SELECT 1")
        return {"result": result}

# FastAPI lifespan context manager (FastAPI 0.93+)
@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Lifespan context manager

    Replaces @app.on_event("startup") and @app.on_event("shutdown")
    """
    # Startup
    await database.connect()
    print("Startup complete")

    yield  # Application runs

    # Shutdown
    await database.disconnect()
    print("Shutdown complete")

# Create app with lifespan
app = FastAPI(lifespan=lifespan)
```

## 🧪 Performance Comparison

### Benchmark: Sync vs Async

```python
import time

# Sync endpoint (blocking)
@app.get("/sync-sleep")
def sync_sleep():
    """
    Synchronous sleep - blocks thread

    Concurrency: Limited by thread pool size (default: 40)
    """
    time.sleep(0.1)
    return {"message": "sync"}

# Async endpoint (non-blocking)
@app.get("/async-sleep")
async def async_sleep():
    """
    Asynchronous sleep - doesn't block event loop

    Concurrency: Thousands of concurrent requests
    """
    await asyncio.sleep(0.1)
    return {"message": "async"}

"""
Benchmark Results (1000 requests, 100 concurrent):

Sync endpoint:
    Total time: 2.5 seconds
    Requests/sec: 400

Async endpoint:
    Total time: 0.2 seconds
    Requests/sec: 5000

Async is 12.5x faster for I/O-bound operations!
"""
```

## ⚡ Best Practices

### When to Use Async

```python
# ✅ USE ASYNC for:
# - Database queries
# - HTTP requests to external APIs
# - File I/O
# - Network operations
# - WebSocket connections

@app.get("/good-async")
async def good_async_usage():
    """Good use of async - I/O operations"""
    # Database query
    user = await database.fetch_one("SELECT * FROM users LIMIT 1")

    # HTTP request
    response = await http_client.get("https://api.example.com/data")

    # Multiple concurrent operations
    results = await asyncio.gather(
        database.fetch_one("SELECT * FROM posts LIMIT 1"),
        http_client.get("https://api2.example.com/data")
    )

    return {"user": dict(user), "data": response.json()}

# ❌ DON'T USE ASYNC for:
# - CPU-intensive computations
# - Blocking libraries (requests, time.sleep)
# - Synchronous file operations

@app.get("/bad-async")
async def bad_async_usage():
    """Bad use of async - CPU-bound work"""
    # CPU-intensive computation (blocks event loop)
    result = sum(i * i for i in range(10_000_000))  # BAD

    return {"result": result}

# ✅ For CPU-bound work, use sync endpoint
@app.get("/good-sync")
def good_sync_usage():
    """Good use of sync - CPU-bound work"""
    # Runs in thread pool, doesn't block event loop
    result = sum(i * i for i in range(10_000_000))

    return {"result": result}
```

## 🎯 Production Configuration

### Uvicorn with Multiple Workers

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello from worker"}

"""
Run with multiple workers:

    uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

Workers:
    - Each worker runs in separate process
    - Load balanced by Uvicorn
    - Workers = CPU cores (typically)
    - Each worker has own event loop

Production setup:
    - Use Gunicorn with Uvicorn workers
    - gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker
"""
```

### Gunicorn + Uvicorn

```bash
# Install
pip install gunicorn uvicorn[standard]

# Run with Gunicorn + Uvicorn workers
gunicorn main:app \
    --workers 4 \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000 \
    --access-logfile - \
    --error-logfile - \
    --log-level info
```

```python
# gunicorn.conf.py
import multiprocessing

# Server socket
bind = "0.0.0.0:8000"
backlog = 2048

# Worker processes
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn.workers.UvicornWorker"
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 50

# Logging
accesslog = "-"
errorlog = "-"
loglevel = "info"

# Process naming
proc_name = "fastapi-app"

# Server mechanics
daemon = False
pidfile = None
umask = 0
user = None
group = None
tmp_upload_dir = None

# SSL
keyfile = None
certfile = None
```

## ❓ Interview Questions

### Q1: What is ASGI and how is it different from WSGI?

**Answer**:

- **WSGI**: Synchronous, blocking I/O, one request per thread
- **ASGI**: Asynchronous, non-blocking I/O, thousands of concurrent connections
- **ASGI supports**: WebSocket, HTTP/2, background tasks
- **FastAPI uses ASGI** for async/await support

### Q2: When should you use `async def` vs `def` in FastAPI?

**Answer**:

- **Use `async def`** for I/O-bound operations (database, HTTP requests, file I/O)
- **Use `def`** for CPU-bound operations (heavy computation, data processing)
- **`async def`** runs in event loop, must use `await` for I/O
- **`def`** runs in thread pool, can use blocking operations

### Q3: How does FastAPI handle concurrent requests?

**Answer**:

- **Async endpoints**: Event loop handles thousands concurrently
- **Sync endpoints**: Thread pool (default 40 threads)
- **One worker process**: One event loop
- **Multiple workers**: Multiple event loops (one per process)

### Q4: What are the performance benefits of async in FastAPI?

**Answer**:

- **Higher concurrency**: Thousands vs hundreds of requests
- **Lower latency**: Non-blocking I/O
- **Better resource usage**: No thread overhead
- **Scalability**: More requests per CPU core
- **10-20x improvement** for I/O-bound workloads

## 📚 Summary

**Key Takeaways**:

1. **ASGI** enables async/await in Python web apps
2. **FastAPI** built on ASGI for high concurrency
3. **async def** for I/O-bound, **def** for CPU-bound
4. **Event loop** handles thousands of concurrent requests
5. **asyncio.gather()** for concurrent operations
6. **Async database** with databases or SQLAlchemy
7. **Uvicorn** is the recommended ASGI server
8. **Multiple workers** for production (CPU cores)
9. **Don't block event loop** with sync operations
10. **10-20x performance** improvement over WSGI

ASGI and async/await are the foundation of FastAPI's performance!
