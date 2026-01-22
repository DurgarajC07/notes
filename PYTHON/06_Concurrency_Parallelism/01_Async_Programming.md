# ⚡ Async Programming in Python

## Overview

Asynchronous programming allows handling multiple I/O operations concurrently without multithreading, making it ideal for I/O-bound applications.

---

## asyncio Fundamentals

### Coroutines

```python
import asyncio

# Define coroutine with async def
async def fetch_data():
    """Coroutine function"""
    print("Start fetching")
    await asyncio.sleep(1)  # Simulates I/O operation
    print("Done fetching")
    return {"data": "result"}

# Running coroutines
async def main():
    """Main coroutine"""
    result = await fetch_data()
    print(result)

# Run the event loop
if __name__ == "__main__":
    asyncio.run(main())

# Python 3.6 compatibility
# loop = asyncio.get_event_loop()
# loop.run_until_complete(main())
# loop.close()
```

### Concurrent Execution

```python
async def task1():
    """First task"""
    print("Task 1 starting")
    await asyncio.sleep(2)
    print("Task 1 done")
    return "Result 1"

async def task2():
    """Second task"""
    print("Task 2 starting")
    await asyncio.sleep(1)
    print("Task 2 done")
    return "Result 2"

async def task3():
    """Third task"""
    print("Task 3 starting")
    await asyncio.sleep(1.5)
    print("Task 3 done")
    return "Result 3"

# Run concurrently with gather
async def run_concurrent():
    """Run tasks concurrently"""
    results = await asyncio.gather(
        task1(),
        task2(),
        task3()
    )
    print(f"All results: {results}")

# Total time: ~2 seconds (not 4.5 seconds sequentially)

# Run concurrently as tasks
async def run_as_tasks():
    """Create and await tasks"""
    t1 = asyncio.create_task(task1())
    t2 = asyncio.create_task(task2())
    t3 = asyncio.create_task(task3())

    result1 = await t1
    result2 = await t2
    result3 = await t3

    return [result1, result2, result3]
```

---

## Event Loop

### Understanding the Event Loop

```python
import asyncio

# Get event loop
loop = asyncio.get_event_loop()

# Schedule coroutine
async def hello():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

# Run until complete
loop.run_until_complete(hello())

# Schedule multiple coroutines
tasks = [hello() for _ in range(3)]
loop.run_until_complete(asyncio.gather(*tasks))

# Run forever (for servers)
async def server_loop():
    while True:
        await handle_request()
        await asyncio.sleep(0.1)

# loop.run_forever()  # Runs indefinitely
```

### Event Loop Policies

```python
# Get current event loop policy
policy = asyncio.get_event_loop_policy()

# Create new event loop
loop = policy.new_event_loop()
asyncio.set_event_loop(loop)

# Windows specific
if sys.platform == 'win32':
    asyncio.set_event_loop_policy(
        asyncio.WindowsProactorEventLoopPolicy()
    )

# Use uvloop for better performance (Unix only)
import uvloop
asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
```

---

## Async Patterns

### 1. **gather() - Run Multiple Coroutines**

```python
async def fetch_user(user_id: int):
    """Fetch user data"""
    await asyncio.sleep(0.1)  # Simulate API call
    return {"id": user_id, "name": f"User{user_id}"}

async def fetch_multiple_users():
    """Fetch multiple users concurrently"""
    user_ids = [1, 2, 3, 4, 5]

    # Gather all results
    users = await asyncio.gather(
        *[fetch_user(uid) for uid in user_ids]
    )

    return users

# Handle exceptions
async def fetch_with_error_handling():
    """Gather with exception handling"""
    results = await asyncio.gather(
        fetch_user(1),
        fetch_user(2),
        fetch_user(999),  # Might fail
        return_exceptions=True  # Don't propagate exceptions
    )

    for result in results:
        if isinstance(result, Exception):
            print(f"Error: {result}")
        else:
            print(f"Success: {result}")
```

### 2. **wait() - More Control Over Completion**

```python
async def wait_example():
    """Wait with different strategies"""
    tasks = [
        asyncio.create_task(task1()),
        asyncio.create_task(task2()),
        asyncio.create_task(task3())
    ]

    # Wait for all to complete
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.ALL_COMPLETED
    )

    # Wait for first to complete
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )

    # Cancel pending tasks
    for task in pending:
        task.cancel()

    # Wait with timeout
    done, pending = await asyncio.wait(
        tasks,
        timeout=5.0
    )
```

### 3. **as_completed() - Process Results as They Complete**

```python
async def process_as_completed():
    """Process results as they come in"""
    tasks = [fetch_user(i) for i in range(10)]

    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(f"Got result: {result}")
        # Process immediately, don't wait for all
```

### 4. **Semaphore - Limit Concurrency**

```python
async def rate_limited_fetch(sem: asyncio.Semaphore, url: str):
    """Fetch with concurrency limit"""
    async with sem:
        # Only N concurrent requests
        response = await fetch(url)
        return response

async def fetch_many_with_limit():
    """Limit concurrent operations"""
    sem = asyncio.Semaphore(5)  # Max 5 concurrent

    urls = [f"https://api.example.com/items/{i}" for i in range(100)]

    tasks = [
        rate_limited_fetch(sem, url)
        for url in urls
    ]

    results = await asyncio.gather(*tasks)
    return results
```

### 5. **Lock - Mutual Exclusion**

```python
class AsyncCounter:
    """Thread-safe async counter"""

    def __init__(self):
        self.value = 0
        self.lock = asyncio.Lock()

    async def increment(self):
        """Increment counter safely"""
        async with self.lock:
            # Only one coroutine can execute this
            current = self.value
            await asyncio.sleep(0.01)  # Simulate work
            self.value = current + 1

async def test_counter():
    """Test counter with concurrent access"""
    counter = AsyncCounter()

    # Run 100 concurrent increments
    await asyncio.gather(*[
        counter.increment()
        for _ in range(100)
    ])

    print(f"Final value: {counter.value}")  # Always 100
```

---

## Async with HTTP Clients

### httpx - Async HTTP Client

```python
import httpx

async def fetch_url(url: str):
    """Fetch single URL"""
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()

async def fetch_multiple_urls(urls: list[str]):
    """Fetch multiple URLs concurrently"""
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return [r.json() for r in responses]

# With timeout and retries
async def fetch_with_retry(url: str, max_retries: int = 3):
    """Fetch with retry logic"""
    async with httpx.AsyncClient(timeout=10.0) as client:
        for attempt in range(max_retries):
            try:
                response = await client.get(url)
                response.raise_for_status()
                return response.json()
            except httpx.HTTPError as e:
                if attempt == max_retries - 1:
                    raise
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
```

---

## Async Database Operations

### SQLAlchemy Async

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Create async engine
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    echo=True,
    pool_size=20
)

# Create async session factory
AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_user(user_id: int):
    """Get user asynchronously"""
    async with AsyncSessionLocal() as session:
        result = await session.execute(
            select(User).where(User.id == user_id)
        )
        user = result.scalar_one_or_none()
        return user

async def create_user(username: str, email: str):
    """Create user asynchronously"""
    async with AsyncSessionLocal() as session:
        user = User(username=username, email=email)
        session.add(user)
        await session.commit()
        await session.refresh(user)
        return user

async def bulk_create_users(users_data: list[dict]):
    """Create multiple users"""
    async with AsyncSessionLocal() as session:
        users = [User(**data) for data in users_data]
        session.add_all(users)
        await session.commit()
```

---

## Async Context Managers

```python
class AsyncDatabaseConnection:
    """Async context manager"""

    async def __aenter__(self):
        """Enter async context"""
        print("Opening connection")
        self.conn = await asyncpg.connect(
            host='localhost',
            user='user',
            password='password',
            database='db'
        )
        return self.conn

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Exit async context"""
        print("Closing connection")
        await self.conn.close()

# Usage
async def use_connection():
    async with AsyncDatabaseConnection() as conn:
        result = await conn.fetch("SELECT * FROM users")
        return result

# Using contextlib
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_connection():
    """Async context manager using decorator"""
    conn = await asyncpg.connect(DATABASE_URL)
    try:
        yield conn
    finally:
        await conn.close()

async def use_decorated_cm():
    async with get_connection() as conn:
        await conn.execute("INSERT INTO logs (message) VALUES ($1)", "test")
```

---

## Async Generators

```python
async def async_range(count: int):
    """Async generator"""
    for i in range(count):
        await asyncio.sleep(0.1)
        yield i

async def consume_async_generator():
    """Consume async generator"""
    async for value in async_range(10):
        print(value)

# Async generator with cleanup
async def fetch_paginated_data(url: str):
    """Fetch data page by page"""
    page = 1
    async with httpx.AsyncClient() as client:
        while True:
            response = await client.get(f"{url}?page={page}")
            data = response.json()

            if not data["items"]:
                break

            for item in data["items"]:
                yield item

            page += 1

async def process_all_data():
    """Process data as it arrives"""
    async for item in fetch_paginated_data("https://api.example.com/items"):
        await process_item(item)
```

---

## Async Queue

```python
async def producer(queue: asyncio.Queue, n: int):
    """Produce items"""
    for i in range(n):
        await asyncio.sleep(0.1)
        await queue.put(f"item-{i}")
        print(f"Produced: item-{i}")

    await queue.put(None)  # Sentinel value

async def consumer(queue: asyncio.Queue, name: str):
    """Consume items"""
    while True:
        item = await queue.get()

        if item is None:
            queue.task_done()
            break

        print(f"Consumer {name} processing: {item}")
        await asyncio.sleep(0.2)  # Simulate work
        queue.task_done()

async def producer_consumer_pattern():
    """Producer-consumer with queue"""
    queue = asyncio.Queue(maxsize=10)

    # Start producer and consumers
    await asyncio.gather(
        producer(queue, 20),
        consumer(queue, "A"),
        consumer(queue, "B"),
        consumer(queue, "C")
    )
```

---

## Common Pitfalls

### 1. **Blocking Code in Async**

```python
# ❌ BAD: Blocking operations
async def bad_async():
    time.sleep(1)  # Blocks entire event loop!
    requests.get(url)  # Synchronous HTTP - blocks!

# ✅ GOOD: Use async versions
async def good_async():
    await asyncio.sleep(1)  # Non-blocking sleep
    async with httpx.AsyncClient() as client:
        await client.get(url)  # Async HTTP

# Run blocking code in executor
import concurrent.futures

async def run_blocking():
    loop = asyncio.get_running_loop()

    # Run in thread pool
    result = await loop.run_in_executor(
        None,  # Default executor
        blocking_function,
        arg1, arg2
    )

    return result
```

### 2. **Not Awaiting Coroutines**

```python
# ❌ BAD: Forgot await
async def bad():
    result = fetch_data()  # Returns coroutine, not result!
    print(result)  # <coroutine object>

# ✅ GOOD: Await coroutine
async def good():
    result = await fetch_data()
    print(result)  # Actual result
```

### 3. **Creating Tasks but Not Awaiting**

```python
# ❌ BAD: Task never completes
async def bad():
    asyncio.create_task(background_work())
    # Function ends, task might not finish

# ✅ GOOD: Track and await
async def good():
    task = asyncio.create_task(background_work())
    # Do other work
    await task  # Ensure completion
```

---

## Best Practices

### ✅ Do's:

1. **Use async for I/O-bound** operations
2. **Avoid blocking calls** in async code
3. **Use connection pooling**
4. **Set timeouts** on operations
5. **Use semaphores** to limit concurrency
6. **Handle exceptions** in tasks
7. **Use uvloop** for better performance

### ❌ Don'ts:

1. **Don't use for CPU-bound** work (use multiprocessing)
2. **Don't mix sync and async** database drivers
3. **Don't forget to await** coroutines
4. **Don't create unlimited** concurrent tasks
5. **Don't use time.sleep()** - use asyncio.sleep()
6. **Don't ignore task exceptions**

---

## Interview Questions

### Q1: What is async/await and how does it work?

**Answer**: async/await is Python's syntax for asynchronous programming:

- `async def` defines coroutine
- `await` yields control back to event loop
- Event loop manages multiple coroutines
- Only one coroutine executes at a time (single-threaded)
- Great for I/O-bound tasks, not CPU-bound

### Q2: Difference between asyncio.gather() and asyncio.wait()?

**Answer**:

- **gather()**: Returns results in order, simpler API
- **wait()**: More control (FIRST_COMPLETED, timeout), returns sets
  Use gather() for simple cases, wait() when need fine control.

### Q3: When should you use async vs threading vs multiprocessing?

**Answer**:

- **Async**: I/O-bound (network, file), single-threaded
- **Threading**: I/O-bound with blocking libraries, true parallelism limited by GIL
- **Multiprocessing**: CPU-bound, bypasses GIL, separate memory

### Q4: What is the event loop?

**Answer**: Core of async programming:

- Schedules and executes coroutines
- Handles I/O operations
- Manages callbacks and tasks
- Single-threaded but concurrent
- Can be customized (uvloop for performance)

### Q5: How do you handle blocking operations in async code?

**Answer**:

- Use async versions of libraries (httpx, asyncpg)
- Run blocking code in executor: `run_in_executor()`
- Use thread pool for blocking I/O
- Avoid blocking at all costs in hot path

---

## Summary

Async programming essentials:

- **Coroutines** with async/await
- **Event loop** manages execution
- **gather()** for concurrent execution
- **Semaphore** for rate limiting
- **httpx** for async HTTP
- **SQLAlchemy async** for databases
- **Async for I/O-bound**, not CPU-bound

Master async for scalable I/O operations! ⚡
