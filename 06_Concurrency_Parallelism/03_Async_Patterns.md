# ⚡ Async/Await Patterns

## Overview

Advanced patterns for Python's asyncio, focusing on concurrent programming, task management, and performance optimization for backend applications.

---

## Understanding Asyncio

### Event Loop Fundamentals

```python
import asyncio
from typing import List

"""
Event Loop:
- Single-threaded concurrency
- Cooperative multitasking
- Tasks yield control with await
- Non-blocking I/O operations

When to use async:
✅ I/O-bound operations (API calls, database queries, file I/O)
✅ Many concurrent operations
✅ WebSocket/long-lived connections

When NOT to use:
❌ CPU-bound operations (use multiprocessing)
❌ Blocking third-party libraries
❌ Simple sequential code
"""

# Basic async function
async def fetch_data(url: str) -> dict:
    """Async function - can use await inside"""
    print(f"Fetching {url}")
    await asyncio.sleep(1)  # Simulate I/O
    return {"url": url, "data": "..."}

# Running async code
async def main():
    """Main async entry point"""
    result = await fetch_data("https://api.example.com")
    print(result)

# Run event loop
if __name__ == "__main__":
    asyncio.run(main())
```

---

## Concurrent Execution Patterns

### 1. gather() - Run Concurrently

```python
async def fetch_user(user_id: int) -> dict:
    """Fetch user data"""
    await asyncio.sleep(0.5)  # Simulate API call
    return {"id": user_id, "name": f"User{user_id}"}

async def fetch_posts(user_id: int) -> List[dict]:
    """Fetch user posts"""
    await asyncio.sleep(0.3)
    return [{"id": i, "title": f"Post {i}"} for i in range(3)]

# ✅ GOOD: Fetch concurrently with gather
async def get_user_profile_concurrent(user_id: int):
    """
    gather() - Run tasks concurrently

    - All tasks start immediately
    - Returns results in order
    - If one fails, all cancelled
    - Total time: max(task_times)
    """
    user, posts = await asyncio.gather(
        fetch_user(user_id),
        fetch_posts(user_id)
    )

    return {"user": user, "posts": posts}

# ❌ BAD: Sequential execution
async def get_user_profile_sequential(user_id: int):
    """
    Sequential - waits for each

    Total time: sum(task_times) = 0.5 + 0.3 = 0.8s
    """
    user = await fetch_user(user_id)
    posts = await fetch_posts(user_id)  # Waits for user first

    return {"user": user, "posts": posts}

# Timing comparison
import time

async def compare_performance():
    """Compare sequential vs concurrent"""

    # Sequential
    start = time.time()
    await get_user_profile_sequential(1)
    sequential_time = time.time() - start
    print(f"Sequential: {sequential_time:.2f}s")

    # Concurrent
    start = time.time()
    await get_user_profile_concurrent(1)
    concurrent_time = time.time() - start
    print(f"Concurrent: {concurrent_time:.2f}s")

    # Speedup
    print(f"Speedup: {sequential_time / concurrent_time:.1f}x")

# asyncio.run(compare_performance())
# Sequential: 0.80s
# Concurrent: 0.50s
# Speedup: 1.6x
```

### 2. as_completed() - Process as They Complete

```python
async def fetch_with_timeout(url: str, timeout: float) -> dict:
    """Fetch with varying completion times"""
    await asyncio.sleep(timeout)
    return {"url": url, "time": timeout}

async def process_urls_as_completed():
    """
    as_completed() - Process results as they finish

    Use when:
    - Want to process results immediately
    - Don't care about order
    - Show progress to user
    """
    urls = [
        ("https://api1.com", 2.0),
        ("https://api2.com", 0.5),
        ("https://api3.com", 1.0),
    ]

    # Create tasks
    tasks = [fetch_with_timeout(url, timeout) for url, timeout in urls]

    # Process as completed
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(f"Completed: {result['url']} in {result['time']}s")

    # Output order: api2 (0.5s), api3 (1.0s), api1 (2.0s)

# asyncio.run(process_urls_as_completed())
```

### 3. wait() - Advanced Control

```python
async def process_with_timeout():
    """
    wait() - Fine-grained control over task completion

    Options:
    - FIRST_COMPLETED: Return when any task completes
    - FIRST_EXCEPTION: Return when any task raises exception
    - ALL_COMPLETED: Wait for all (default)
    """
    tasks = [
        asyncio.create_task(fetch_with_timeout("url1", 1.0)),
        asyncio.create_task(fetch_with_timeout("url2", 2.0)),
        asyncio.create_task(fetch_with_timeout("url3", 3.0)),
    ]

    # Wait for first completion
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )

    print(f"First completed: {len(done)} task(s)")
    print(f"Still pending: {len(pending)} task(s)")

    # Cancel remaining tasks
    for task in pending:
        task.cancel()

    # Get result from completed task
    result = done.pop().result()
    print(f"Result: {result}")

# Wait with timeout
async def wait_with_timeout():
    """Wait with timeout"""
    tasks = [
        asyncio.create_task(fetch_with_timeout("url1", 5.0)),
        asyncio.create_task(fetch_with_timeout("url2", 5.0)),
    ]

    # Wait max 2 seconds
    done, pending = await asyncio.wait(
        tasks,
        timeout=2.0
    )

    print(f"Completed in time: {len(done)}")
    print(f"Timed out: {len(pending)}")

    # Cancel timed-out tasks
    for task in pending:
        task.cancel()
```

### 4. create_task() - Background Tasks

```python
async def background_task(name: str, duration: float):
    """Simulated background task"""
    print(f"{name} started")
    await asyncio.sleep(duration)
    print(f"{name} completed")
    return f"{name} result"

async def main_with_background_tasks():
    """
    create_task() - Run tasks in background

    Tasks start immediately, don't block
    """
    # Start background tasks
    task1 = asyncio.create_task(background_task("Task 1", 2.0))
    task2 = asyncio.create_task(background_task("Task 2", 1.0))

    # Do other work while tasks run
    print("Doing other work...")
    await asyncio.sleep(0.5)
    print("Still doing work...")

    # Wait for tasks when needed
    result1 = await task1
    result2 = await task2

    print(f"Results: {result1}, {result2}")
```

---

## Error Handling Patterns

### Handling Exceptions in Concurrent Tasks

```python
class APIError(Exception):
    """API call failed"""
    pass

async def fetch_with_error(url: str, should_fail: bool = False):
    """Fetch that may fail"""
    await asyncio.sleep(0.5)

    if should_fail:
        raise APIError(f"Failed to fetch {url}")

    return {"url": url, "status": "success"}

# ❌ BAD: gather() fails if any task fails
async def fetch_all_fail_fast():
    """One failure cancels all"""
    try:
        results = await asyncio.gather(
            fetch_with_error("url1", False),
            fetch_with_error("url2", True),  # This fails
            fetch_with_error("url3", False),
        )
    except APIError as e:
        print(f"Failed: {e}")
        # url3 might not have completed

# ✅ GOOD: gather() with return_exceptions
async def fetch_all_continue_on_error():
    """
    return_exceptions=True

    - Exceptions returned as values
    - All tasks complete
    - Handle errors individually
    """
    results = await asyncio.gather(
        fetch_with_error("url1", False),
        fetch_with_error("url2", True),
        fetch_with_error("url3", False),
        return_exceptions=True  # Don't raise, return exception
    )

    # Process results
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Task {i} failed: {result}")
        else:
            print(f"Task {i} succeeded: {result}")

# ✅ GOOD: Try/except per task
async def fetch_all_with_retry():
    """Handle each task's errors separately"""

    async def safe_fetch(url: str) -> dict:
        """Wrap fetch with error handling"""
        try:
            return await fetch_with_error(url, url == "url2")
        except APIError as e:
            print(f"Error fetching {url}: {e}")
            # Could retry here
            return {"url": url, "status": "error", "error": str(e)}

    results = await asyncio.gather(
        safe_fetch("url1"),
        safe_fetch("url2"),
        safe_fetch("url3"),
    )

    return results
```

---

## Rate Limiting and Semaphores

### Limit Concurrent Operations

```python
async def rate_limited_fetch():
    """
    Semaphore - Limit concurrent operations

    Use when:
    - API has rate limits
    - Database connection pool
    - Don't overwhelm external service
    """
    # Limit to 3 concurrent requests
    semaphore = asyncio.Semaphore(3)

    async def fetch_with_limit(url: str, sem: asyncio.Semaphore):
        """Fetch with semaphore"""
        async with sem:  # Acquire semaphore
            print(f"Fetching {url}")
            await asyncio.sleep(1)
            print(f"Completed {url}")
            return {"url": url}

    # Create 10 tasks
    urls = [f"url{i}" for i in range(10)]

    # Only 3 run at a time
    results = await asyncio.gather(
        *[fetch_with_limit(url, semaphore) for url in urls]
    )

    return results

# With rate limiter class
class RateLimiter:
    """Rate limiter with time window"""

    def __init__(self, max_calls: int, period: float):
        self.max_calls = max_calls
        self.period = period
        self.semaphore = asyncio.Semaphore(max_calls)
        self.calls = []

    async def __aenter__(self):
        """Acquire rate limit slot"""
        await self.semaphore.acquire()

        # Wait if needed
        now = asyncio.get_event_loop().time()

        # Remove old calls
        self.calls = [t for t in self.calls if now - t < self.period]

        if len(self.calls) >= self.max_calls:
            sleep_time = self.period - (now - self.calls[0])
            await asyncio.sleep(sleep_time)
            self.calls.pop(0)

        self.calls.append(now)

        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Release semaphore"""
        self.semaphore.release()

# Usage
async def api_with_rate_limit():
    """Use rate limiter"""
    limiter = RateLimiter(max_calls=5, period=1.0)  # 5 calls per second

    async def fetch(url: str):
        async with limiter:
            print(f"Fetching {url}")
            await asyncio.sleep(0.1)
            return {"url": url}

    urls = [f"url{i}" for i in range(20)]
    results = await asyncio.gather(*[fetch(url) for url in urls])

    return results
```

---

## Async Context Managers

### Database Connection Pool

```python
from typing import AsyncIterator
import aiosqlite

class AsyncDatabasePool:
    """Async database connection pool"""

    def __init__(self, db_path: str, pool_size: int = 5):
        self.db_path = db_path
        self.pool_size = pool_size
        self.semaphore = asyncio.Semaphore(pool_size)
        self.connections = []

    async def __aenter__(self):
        """Initialize pool"""
        for _ in range(self.pool_size):
            conn = await aiosqlite.connect(self.db_path)
            self.connections.append(conn)

        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Close all connections"""
        for conn in self.connections:
            await conn.close()

    async def execute(self, query: str, params: tuple = ()):
        """Execute query with connection from pool"""
        async with self.semaphore:
            # Get available connection
            conn = self.connections[0]  # Simplified

            async with conn.execute(query, params) as cursor:
                return await cursor.fetchall()

# Usage
async def use_db_pool():
    """Use database pool"""
    async with AsyncDatabasePool("app.db", pool_size=3) as db:
        # Multiple concurrent queries
        results = await asyncio.gather(
            db.execute("SELECT * FROM users WHERE id = ?", (1,)),
            db.execute("SELECT * FROM posts WHERE user_id = ?", (1,)),
            db.execute("SELECT * FROM comments WHERE user_id = ?", (1,)),
        )

        return results
```

---

## Real-World Patterns

### Batch Processing with Progress

```python
from typing import Callable, TypeVar, List
from tqdm.asyncio import tqdm

T = TypeVar('T')
R = TypeVar('R')

async def process_batch(
    items: List[T],
    process_func: Callable[[T], R],
    batch_size: int = 10,
    max_concurrent: int = 5
) -> List[R]:
    """
    Process items in batches with concurrency limit

    Use for:
    - API bulk operations
    - Data migration
    - Report generation
    """
    semaphore = asyncio.Semaphore(max_concurrent)
    results = []

    async def process_with_limit(item: T) -> R:
        """Process single item with semaphore"""
        async with semaphore:
            return await process_func(item)

    # Process in batches
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]

        # Process batch concurrently
        batch_results = await tqdm.gather(
            *[process_with_limit(item) for item in batch],
            desc=f"Batch {i//batch_size + 1}"
        )

        results.extend(batch_results)

        # Optional: delay between batches
        if i + batch_size < len(items):
            await asyncio.sleep(0.1)

    return results

# Example: Process 1000 users
async def update_users():
    """Update users in batches"""
    user_ids = list(range(1, 1001))

    async def update_user(user_id: int):
        """Update single user"""
        await asyncio.sleep(0.1)  # Simulate API call
        return f"Updated user {user_id}"

    results = await process_batch(
        user_ids,
        update_user,
        batch_size=50,
        max_concurrent=10
    )

    return results
```

### Timeout Pattern

```python
async def fetch_with_timeout(url: str, timeout: float = 5.0):
    """
    Fetch with timeout

    Prevents hanging on slow requests
    """
    try:
        async with asyncio.timeout(timeout):  # Python 3.11+
            return await fetch_data(url)

    except asyncio.TimeoutError:
        print(f"Request to {url} timed out after {timeout}s")
        return None

# Or for older Python versions
async def fetch_with_timeout_old(url: str, timeout: float = 5.0):
    """Fetch with timeout (Python < 3.11)"""
    try:
        return await asyncio.wait_for(fetch_data(url), timeout=timeout)

    except asyncio.TimeoutError:
        print(f"Request to {url} timed out")
        return None
```

---

## Best Practices

### ✅ Do's:

1. **Use gather()** for concurrent operations
2. **Add timeouts** to prevent hanging
3. **Limit concurrency** with Semaphore
4. **Handle exceptions** in each task
5. **Use create_task()** for background work
6. **Close resources** in finally blocks
7. **Profile async code** to find bottlenecks

### ❌ Don'ts:

1. **Don't mix** sync and async code
2. **Don't block** event loop (use run_in_executor for blocking code)
3. **Don't forget** to await coroutines
4. **Don't create** too many tasks (memory overhead)
5. **Don't ignore** task exceptions
6. **Don't use** async for CPU-bound work

---

## Interview Questions

### Q1: When to use async vs threads?

**Answer**:

- **Async**: I/O-bound (API calls, DB queries, file I/O), single-threaded
- **Threads**: I/O-bound with blocking libraries, parallel I/O
- **Multiprocessing**: CPU-bound (calculations, image processing)
- **Async pros**: Low overhead, many concurrent operations
- **Threads pros**: Works with blocking code
  Use async for modern I/O-heavy applications.

### Q2: What is event loop?

**Answer**:

- **Event loop**: Manages async task execution
- **Single-threaded**: One task at a time
- **Cooperative**: Tasks yield control with await
- **Non-blocking**: I/O happens concurrently
- **Scheduler**: Decides which task runs next
  Heart of asyncio, coordinates all async operations.

### Q3: gather() vs as_completed()?

**Answer**:

- **gather()**: Returns results in order, waits for all
- **as_completed()**: Process as they finish, any order
- **gather() use**: Need all results together
- **as_completed() use**: Show progress, process immediately
- **Performance**: Same, just different result handling
  Choose based on how you want to process results.

### Q4: How to handle errors in concurrent tasks?

**Answer**:

- **gather(return_exceptions=True)**: Returns exceptions as values
- **Try/except per task**: Wrap each task in error handler
- **wait()**: Check done/pending, handle individually
- **Don't**: Let one failure cancel all tasks
  Handle errors to ensure partial success possible.

### Q5: How to limit concurrent requests?

**Answer**:

- **Semaphore**: asyncio.Semaphore(max_concurrent)
- **Queue**: Task queue with workers
- **Rate limiter**: Time-based limiting
- **Why**: Prevent overwhelming service, respect rate limits
  Essential for production async code.

---

## Summary

Async/await patterns essentials:

- **gather()**: Concurrent execution, results in order
- **create_task()**: Background tasks
- **Semaphore**: Limit concurrent operations
- **Timeouts**: Prevent hanging requests
- **Error handling**: return_exceptions or per-task try/except
- **Use cases**: API calls, DB queries, I/O-bound work
- **Not for**: CPU-bound operations

Master async for high-performance I/O! ⚡
