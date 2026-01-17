# FastAPI Async: Event Loop

## 📖 Introduction

The **event loop** is the core of Python's async/await functionality. It manages and executes asynchronous tasks, handling I/O operations efficiently without blocking. Understanding the event loop is essential for building high-performance FastAPI applications.

## 🔄 Event Loop Basics

### What is an Event Loop?

```python
import asyncio

"""
Event Loop:
- Single-threaded execution
- Manages coroutines (async functions)
- Switches between tasks during I/O waits
- Non-blocking I/O operations

Traditional Threading:
- Multiple threads
- OS handles context switching
- Overhead from thread creation
- Race conditions possible

Event Loop Advantages:
- Lower overhead (no thread switching)
- No race conditions (single-threaded)
- Efficient I/O handling
- Thousands of concurrent connections
"""

async def example_coroutine():
    """
    Coroutine - async function

    - Declared with 'async def'
    - Called with 'await'
    - Runs in event loop
    """
    print("Starting coroutine")
    await asyncio.sleep(1)  # Yields control to event loop
    print("Coroutine complete")
    return "Result"

# Running coroutine
async def main():
    result = await example_coroutine()
    print(f"Result: {result}")

# Start event loop
asyncio.run(main())
```

### Event Loop Lifecycle

```python
import asyncio
import time

async def task1():
    """Task 1"""
    print("Task 1 started")
    await asyncio.sleep(2)
    print("Task 1 completed")
    return "Result 1"

async def task2():
    """Task 2"""
    print("Task 2 started")
    await asyncio.sleep(1)
    print("Task 2 completed")
    return "Result 2"

async def main():
    """
    Event loop execution flow:

    1. Create tasks
    2. Schedule tasks in event loop
    3. Run tasks concurrently
    4. Switch between tasks during I/O waits
    5. Collect results when tasks complete
    """
    print("Starting main")
    start_time = time.time()

    # Create tasks (scheduled immediately)
    t1 = asyncio.create_task(task1())
    t2 = asyncio.create_task(task2())

    # Wait for both tasks
    result1 = await t1
    result2 = await t2

    elapsed = time.time() - start_time
    print(f"Total time: {elapsed:.2f}s")  # ~2s (concurrent, not 3s sequential)

    return result1, result2

asyncio.run(main())
```

## 🎯 Creating and Managing Tasks

### Create Task

```python
import asyncio
from typing import List

async def fetch_data(source_id: int, delay: float) -> dict:
    """
    Simulate async data fetch

    Args:
        source_id: Data source ID
        delay: Simulated delay in seconds

    Returns:
        Fetched data
    """
    print(f"Fetching from source {source_id}")
    await asyncio.sleep(delay)
    return {"source_id": source_id, "data": f"Data from source {source_id}"}

async def main():
    """Create and manage tasks"""

    # Create single task
    task = asyncio.create_task(fetch_data(1, 1.0))
    result = await task
    print(f"Result: {result}")

    # Create multiple tasks
    tasks = [
        asyncio.create_task(fetch_data(i, 0.5))
        for i in range(5)
    ]

    # Wait for all tasks
    results = await asyncio.gather(*tasks)
    print(f"All results: {results}")

asyncio.run(main())
```

### Task with Name

```python
async def named_task_example():
    """Task with name for debugging"""

    # Create task with name
    task = asyncio.create_task(
        fetch_data(1, 1.0),
        name="fetch-source-1"
    )

    print(f"Task name: {task.get_name()}")

    result = await task
    return result
```

### Task Cancellation

```python
async def long_running_task():
    """Long-running task that can be cancelled"""
    try:
        print("Long task started")
        await asyncio.sleep(10)
        print("Long task completed")
        return "Success"
    except asyncio.CancelledError:
        print("Task was cancelled")
        # Cleanup code here
        raise  # Re-raise to mark task as cancelled

async def cancel_example():
    """Cancel task example"""

    # Create task
    task = asyncio.create_task(long_running_task())

    # Let it run for 2 seconds
    await asyncio.sleep(2)

    # Cancel task
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Task cancellation confirmed")

asyncio.run(cancel_example())
```

## 🔀 Concurrent Execution Patterns

### asyncio.gather - Wait for All

```python
async def fetch_user(user_id: int):
    """Fetch user data"""
    await asyncio.sleep(0.5)
    return {"id": user_id, "name": f"User {user_id}"}

async def fetch_posts(user_id: int):
    """Fetch user posts"""
    await asyncio.sleep(0.3)
    return [{"id": i, "title": f"Post {i}"} for i in range(3)]

async def fetch_comments(user_id: int):
    """Fetch user comments"""
    await asyncio.sleep(0.4)
    return [{"id": i, "text": f"Comment {i}"} for i in range(5)]

async def gather_example():
    """
    asyncio.gather - concurrent execution

    - Runs all coroutines concurrently
    - Returns results in order
    - Fails if any coroutine raises exception (unless return_exceptions=True)
    """
    results = await asyncio.gather(
        fetch_user(1),
        fetch_posts(1),
        fetch_comments(1)
    )

    user, posts, comments = results

    return {
        "user": user,
        "posts": posts,
        "comments": comments
    }

# With exception handling
async def gather_with_exceptions():
    """Handle exceptions in gather"""
    results = await asyncio.gather(
        fetch_user(1),
        fetch_posts(1),
        fetch_comments(1),
        return_exceptions=True  # Don't fail on exception
    )

    # Check for exceptions
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Task {i} failed: {result}")
        else:
            print(f"Task {i} succeeded: {result}")
```

### asyncio.wait - More Control

```python
async def wait_example():
    """
    asyncio.wait - more control than gather

    - Can wait for first completion or all
    - Returns (done, pending) sets
    - More flexible than gather
    """
    tasks = [
        asyncio.create_task(fetch_data(i, i * 0.5))
        for i in range(1, 4)
    ]

    # Wait for all tasks
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.ALL_COMPLETED
    )

    results = [task.result() for task in done]
    print(f"All tasks completed: {results}")

async def wait_first_example():
    """Wait for first task to complete"""
    tasks = [
        asyncio.create_task(fetch_data(1, 2.0)),
        asyncio.create_task(fetch_data(2, 1.0)),
        asyncio.create_task(fetch_data(3, 3.0))
    ]

    # Wait for first completion
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )

    # Get first result
    first_result = list(done)[0].result()
    print(f"First result: {first_result}")

    # Cancel pending tasks
    for task in pending:
        task.cancel()

async def wait_with_timeout():
    """Wait with timeout"""
    tasks = [
        asyncio.create_task(fetch_data(i, 2.0))
        for i in range(3)
    ]

    # Wait with 1 second timeout
    done, pending = await asyncio.wait(
        tasks,
        timeout=1.0
    )

    print(f"Completed: {len(done)}, Pending: {len(pending)}")

    # Cancel pending
    for task in pending:
        task.cancel()
```

### asyncio.as_completed - Process Results as They Arrive

```python
async def as_completed_example():
    """
    Process results as they complete

    - Yields tasks as they finish
    - Order not guaranteed
    - Good for progressive results
    """
    tasks = [
        asyncio.create_task(fetch_data(i, (4 - i) * 0.5))
        for i in range(1, 5)
    ]

    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(f"Got result: {result}")
        # Process immediately without waiting for others
```

## ⏱️ Timeouts and Deadlines

### asyncio.wait_for - Timeout for Single Operation

```python
async def wait_for_example():
    """
    Timeout for single coroutine

    - Raises asyncio.TimeoutError if exceeds timeout
    - Cancels coroutine on timeout
    """
    try:
        result = await asyncio.wait_for(
            fetch_data(1, 2.0),
            timeout=1.0
        )
        print(f"Result: {result}")
    except asyncio.TimeoutError:
        print("Operation timed out")

# With fallback
async def wait_for_with_fallback():
    """Timeout with fallback value"""
    try:
        result = await asyncio.wait_for(
            fetch_data(1, 2.0),
            timeout=1.0
        )
    except asyncio.TimeoutError:
        result = {"source_id": 1, "data": "Fallback data"}

    return result
```

### asyncio.timeout (Python 3.11+)

```python
async def timeout_context_example():
    """
    Timeout context manager (Python 3.11+)

    - Cleaner syntax
    - Automatically cancels on timeout
    """
    try:
        async with asyncio.timeout(1.0):
            result = await fetch_data(1, 2.0)
            print(f"Result: {result}")
    except asyncio.TimeoutError:
        print("Operation timed out")
```

## 🔄 Event Loop in FastAPI

### Accessing Event Loop

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

@app.get("/concurrent")
async def concurrent_requests():
    """
    FastAPI handles event loop automatically

    - Each request runs in event loop
    - Can create tasks within request
    - Tasks complete before response sent
    """
    # Get current event loop
    loop = asyncio.get_event_loop()
    print(f"Running in loop: {loop}")

    # Create concurrent tasks
    results = await asyncio.gather(
        fetch_data(1, 0.5),
        fetch_data(2, 0.5),
        fetch_data(3, 0.5)
    )

    return {"results": results}

@app.get("/background-safe")
async def background_safe():
    """
    Background tasks use same event loop

    - BackgroundTasks run in same loop
    - Complete after response sent
    """
    from fastapi import BackgroundTasks

    async def bg_task():
        await asyncio.sleep(1)
        print("Background task done")

    background_tasks = BackgroundTasks()
    background_tasks.add_task(bg_task)

    return {"message": "Task scheduled"}
```

### Run Blocking Code in Executor

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

# Thread pool for blocking operations
executor = ThreadPoolExecutor(max_workers=10)

def blocking_operation(n: int) -> int:
    """
    CPU-intensive or blocking operation

    - Would block event loop if called directly
    - Must run in executor
    """
    import time
    time.sleep(2)  # Blocking call
    return n * n

@app.get("/blocking")
async def run_blocking():
    """
    Run blocking code without blocking event loop

    - Uses thread pool executor
    - Event loop free to handle other requests
    """
    loop = asyncio.get_event_loop()

    # Run in executor
    result = await loop.run_in_executor(
        executor,
        blocking_operation,
        10
    )

    return {"result": result}

@app.get("/multiple-blocking")
async def run_multiple_blocking():
    """Run multiple blocking operations concurrently"""
    loop = asyncio.get_event_loop()

    # Create tasks for executor
    tasks = [
        loop.run_in_executor(executor, blocking_operation, i)
        for i in range(5)
    ]

    # Wait for all
    results = await asyncio.gather(*tasks)

    return {"results": results}
```

## 🔧 Advanced Event Loop Patterns

### Semaphore for Rate Limiting

```python
async def rate_limited_fetch(semaphore: asyncio.Semaphore, source_id: int):
    """
    Fetch with rate limiting

    - Semaphore limits concurrent operations
    - Prevents overwhelming external services
    """
    async with semaphore:
        print(f"Fetching source {source_id}")
        result = await fetch_data(source_id, 0.5)
        return result

async def semaphore_example():
    """
    Rate limiting with semaphore

    - Limit to 3 concurrent requests
    - Others wait for slot
    """
    semaphore = asyncio.Semaphore(3)  # Max 3 concurrent

    tasks = [
        rate_limited_fetch(semaphore, i)
        for i in range(10)
    ]

    results = await asyncio.gather(*tasks)
    return results
```

### Queue for Producer-Consumer

```python
async def producer(queue: asyncio.Queue, n: int):
    """
    Producer - adds items to queue

    Args:
        queue: Async queue
        n: Number of items to produce
    """
    for i in range(n):
        await asyncio.sleep(0.1)
        await queue.put(i)
        print(f"Produced: {i}")

    await queue.put(None)  # Sentinel value

async def consumer(queue: asyncio.Queue, consumer_id: int):
    """
    Consumer - processes items from queue

    Args:
        queue: Async queue
        consumer_id: Consumer identifier
    """
    while True:
        item = await queue.get()

        if item is None:
            # Sentinel - stop consuming
            await queue.put(None)  # Re-queue for other consumers
            break

        # Process item
        await asyncio.sleep(0.2)
        print(f"Consumer {consumer_id} processed: {item}")

        queue.task_done()

async def queue_example():
    """
    Producer-consumer pattern

    - Queue coordinates async tasks
    - Multiple consumers process items
    """
    queue = asyncio.Queue(maxsize=10)

    # Start producer and consumers
    await asyncio.gather(
        producer(queue, 20),
        consumer(queue, 1),
        consumer(queue, 2),
        consumer(queue, 3)
    )

    # Wait for all items processed
    await queue.join()
```

### Lock for Shared Resources

```python
shared_resource = 0
lock = asyncio.Lock()

async def increment_with_lock():
    """
    Thread-safe increment with lock

    - Lock prevents race conditions
    - Only one coroutine holds lock at a time
    """
    global shared_resource

    async with lock:
        # Critical section
        temp = shared_resource
        await asyncio.sleep(0.01)  # Simulate work
        shared_resource = temp + 1

async def lock_example():
    """Demonstrate lock usage"""
    global shared_resource
    shared_resource = 0

    # 100 concurrent increments
    tasks = [increment_with_lock() for _ in range(100)]
    await asyncio.gather(*tasks)

    print(f"Final value: {shared_resource}")  # Should be 100
```

## ❓ Interview Questions

### Q1: What is the event loop and how does it work?

**Answer**:

- **Single-threaded** execution environment
- **Manages coroutines** and async operations
- **Switches between tasks** during I/O waits
- **Non-blocking I/O** for high concurrency

### Q2: What's the difference between asyncio.gather and asyncio.wait?

**Answer**:

- **gather**: Returns results in order, simpler API
- **wait**: Returns (done, pending) sets, more control
- **gather** raises exception if any task fails
- **wait** allows partial completion, timeouts

### Q3: How to run blocking code in async FastAPI?

**Answer**:

- Use **loop.run_in_executor()** with ThreadPoolExecutor
- Blocking code runs in thread pool
- Event loop free for other requests
- Or use **def** instead of **async def** (FastAPI runs in thread pool)

### Q4: What is a semaphore and when to use it?

**Answer**:

- **Limits concurrent operations** (e.g., max 10 concurrent)
- **Prevents resource exhaustion** (DB connections, API rate limits)
- Use **asyncio.Semaphore(n)** with async context manager
- Good for rate limiting external API calls

## 📚 Summary

**Key Takeaways**:

1. **Event loop** manages async tasks in single thread
2. **asyncio.create_task()** schedules coroutine
3. **asyncio.gather()** runs tasks concurrently
4. **asyncio.wait()** for more control
5. **asyncio.wait_for()** adds timeout
6. **Semaphore** limits concurrent operations
7. **Queue** for producer-consumer patterns
8. **Lock** for shared resource protection
9. **run_in_executor()** for blocking code
10. **FastAPI manages** event loop automatically

The event loop enables FastAPI's high-performance async capabilities!
