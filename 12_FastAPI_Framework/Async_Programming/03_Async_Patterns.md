# FastAPI Async: Patterns & Best Practices

## 📖 Introduction

Async programming requires different patterns than synchronous code. This guide covers common async patterns, best practices, and pitfalls to avoid when building FastAPI applications.

## 🎯 Concurrent Request Pattern

### Parallel External API Calls

```python
import httpx
import asyncio
from typing import List, Dict
from fastapi import FastAPI

app = FastAPI()

async def fetch_user(client: httpx.AsyncClient, user_id: int) -> dict:
    """Fetch user from external API"""
    response = await client.get(f"https://api.example.com/users/{user_id}")
    return response.json()

async def fetch_user_posts(client: httpx.AsyncClient, user_id: int) -> List[dict]:
    """Fetch user posts"""
    response = await client.get(f"https://api.example.com/users/{user_id}/posts")
    return response.json()

async def fetch_user_comments(client: httpx.AsyncClient, user_id: int) -> List[dict]:
    """Fetch user comments"""
    response = await client.get(f"https://api.example.com/users/{user_id}/comments")
    return response.json()

@app.get("/user-profile/{user_id}")
async def get_user_profile(user_id: int):
    """
    Fetch user profile with all data

    Pattern: Parallel API calls
    - All requests concurrent
    - Total time = slowest request
    - 3x faster than sequential
    """
    async with httpx.AsyncClient(timeout=10.0) as client:
        # Concurrent requests
        user, posts, comments = await asyncio.gather(
            fetch_user(client, user_id),
            fetch_user_posts(client, user_id),
            fetch_user_comments(client, user_id)
        )

    return {
        "user": user,
        "posts": posts,
        "comments": comments
    }
```

### With Error Handling

```python
from fastapi import HTTPException

@app.get("/user-profile-safe/{user_id}")
async def get_user_profile_safe(user_id: int):
    """
    Parallel requests with error handling

    Pattern: return_exceptions=True
    - Don't fail entire request if one API fails
    - Handle errors individually
    """
    async with httpx.AsyncClient(timeout=10.0) as client:
        results = await asyncio.gather(
            fetch_user(client, user_id),
            fetch_user_posts(client, user_id),
            fetch_user_comments(client, user_id),
            return_exceptions=True  # Don't raise on error
        )

    # Check for errors
    user, posts, comments = results

    if isinstance(user, Exception):
        raise HTTPException(status_code=503, detail="User service unavailable")

    # Use empty list if posts/comments failed
    if isinstance(posts, Exception):
        posts = []

    if isinstance(comments, Exception):
        comments = []

    return {
        "user": user,
        "posts": posts,
        "comments": comments
    }
```

## 🔄 Rate Limiting Pattern

### Semaphore for Concurrency Control

```python
async def rate_limited_request(
    semaphore: asyncio.Semaphore,
    client: httpx.AsyncClient,
    url: str
) -> dict:
    """
    Make request with rate limiting

    Pattern: Semaphore limits concurrent requests
    """
    async with semaphore:
        response = await client.get(url)
        return response.json()

@app.get("/batch-fetch")
async def batch_fetch(urls: List[str]):
    """
    Fetch multiple URLs with rate limiting

    Pattern: Limit concurrent requests
    - Prevents overwhelming external service
    - Max 5 concurrent requests
    """
    semaphore = asyncio.Semaphore(5)  # Max 5 concurrent

    async with httpx.AsyncClient() as client:
        tasks = [
            rate_limited_request(semaphore, client, url)
            for url in urls
        ]
        results = await asyncio.gather(*tasks)

    return {"results": results}
```

### Token Bucket Rate Limiter

```python
import time
from collections import deque

class AsyncTokenBucket:
    """
    Token bucket rate limiter

    Pattern: Rate limit by tokens per second
    """

    def __init__(self, rate: float, capacity: int):
        """
        Args:
            rate: Tokens per second
            capacity: Maximum tokens
        """
        self.rate = rate
        self.capacity = capacity
        self.tokens = capacity
        self.last_update = time.time()
        self.lock = asyncio.Lock()

    async def acquire(self, tokens: int = 1):
        """
        Acquire tokens

        Waits if insufficient tokens
        """
        async with self.lock:
            while True:
                now = time.time()
                elapsed = now - self.last_update

                # Refill tokens
                self.tokens = min(
                    self.capacity,
                    self.tokens + elapsed * self.rate
                )
                self.last_update = now

                if self.tokens >= tokens:
                    self.tokens -= tokens
                    return

                # Wait for tokens
                wait_time = (tokens - self.tokens) / self.rate
                await asyncio.sleep(wait_time)

# Global rate limiter
rate_limiter = AsyncTokenBucket(rate=10.0, capacity=100)

@app.get("/rate-limited-endpoint")
async def rate_limited_endpoint():
    """
    Endpoint with rate limiting

    Pattern: Token bucket ensures rate limits
    """
    await rate_limiter.acquire()

    # Process request
    return {"message": "Request processed"}
```

## 💾 Database Connection Pool Pattern

### Connection Pool Management

```python
from databases import Database

class DatabasePool:
    """
    Database connection pool manager

    Pattern: Reuse database connections
    """

    def __init__(self, url: str, min_size: int = 10, max_size: int = 20):
        self.database = Database(
            url,
            min_size=min_size,
            max_size=max_size
        )

    async def connect(self):
        """Connect to database"""
        await self.database.connect()

    async def disconnect(self):
        """Disconnect from database"""
        await self.database.disconnect()

    async def fetch_one(self, query: str, values: dict = None):
        """Execute query and fetch one row"""
        return await self.database.fetch_one(query=query, values=values)

    async def fetch_all(self, query: str, values: dict = None):
        """Execute query and fetch all rows"""
        return await self.database.fetch_all(query=query, values=values)

    async def execute(self, query: str, values: dict = None):
        """Execute query without returning results"""
        return await self.database.execute(query=query, values=values)

# Global pool
db_pool = DatabasePool("postgresql://user:password@localhost/dbname")

@app.on_event("startup")
async def startup():
    await db_pool.connect()

@app.on_event("shutdown")
async def shutdown():
    await db_pool.disconnect()

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """
    Use connection pool

    Pattern: Reuse connections for efficiency
    """
    user = await db_pool.fetch_one(
        "SELECT * FROM users WHERE id = :user_id",
        {"user_id": user_id}
    )

    return dict(user) if user else None
```

## 🔁 Retry Pattern

### Exponential Backoff Retry

```python
import random

async def retry_with_backoff(
    coro,
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0
):
    """
    Retry coroutine with exponential backoff

    Pattern: Retry failed operations
    - Exponential backoff: 1s, 2s, 4s, 8s...
    - Jitter to prevent thundering herd
    """
    for attempt in range(max_retries):
        try:
            return await coro()
        except Exception as e:
            if attempt == max_retries - 1:
                raise  # Last attempt, re-raise

            # Calculate delay with exponential backoff
            delay = min(base_delay * (2 ** attempt), max_delay)

            # Add jitter (random 0-25% of delay)
            jitter = random.uniform(0, delay * 0.25)
            total_delay = delay + jitter

            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {total_delay:.2f}s")
            await asyncio.sleep(total_delay)

@app.get("/external-api")
async def call_external_api():
    """
    Call external API with retries

    Pattern: Retry on failure with backoff
    """
    async def make_request():
        async with httpx.AsyncClient() as client:
            response = await client.get("https://api.example.com/data")
            response.raise_for_status()
            return response.json()

    try:
        data = await retry_with_backoff(make_request, max_retries=3)
        return {"data": data}
    except Exception as e:
        raise HTTPException(status_code=503, detail=f"External API failed: {e}")
```

## ⏱️ Timeout Pattern

### Request-Level Timeout

```python
@app.get("/with-timeout")
async def endpoint_with_timeout():
    """
    Endpoint with timeout

    Pattern: Prevent hanging requests
    """
    try:
        result = await asyncio.wait_for(
            slow_operation(),
            timeout=5.0  # 5 second timeout
        )
        return {"result": result}
    except asyncio.TimeoutError:
        raise HTTPException(status_code=504, detail="Request timed out")

async def slow_operation():
    """Simulated slow operation"""
    await asyncio.sleep(10)  # Takes 10 seconds
    return "Done"
```

### Multiple Operations with Individual Timeouts

```python
@app.get("/multi-timeout")
async def multi_timeout():
    """
    Multiple operations with individual timeouts

    Pattern: Different timeouts for different operations
    """
    async def fast_op():
        await asyncio.sleep(1)
        return "Fast"

    async def slow_op():
        await asyncio.sleep(5)
        return "Slow"

    results = await asyncio.gather(
        asyncio.wait_for(fast_op(), timeout=2.0),
        asyncio.wait_for(slow_op(), timeout=10.0),
        return_exceptions=True
    )

    # Handle timeouts
    processed = []
    for result in results:
        if isinstance(result, asyncio.TimeoutError):
            processed.append("Timed out")
        else:
            processed.append(result)

    return {"results": processed}
```

## 🎭 Circuit Breaker Pattern

### Simple Circuit Breaker

```python
from enum import Enum
from datetime import datetime, timedelta

class CircuitState(Enum):
    CLOSED = "closed"  # Normal operation
    OPEN = "open"      # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing if recovered

class CircuitBreaker:
    """
    Circuit breaker pattern

    Pattern: Fail fast when service is down
    - CLOSED: Normal operation
    - OPEN: Too many failures, reject immediately
    - HALF_OPEN: Test if service recovered
    """

    def __init__(
        self,
        failure_threshold: int = 5,
        timeout: int = 60,
        success_threshold: int = 2
    ):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.success_threshold = success_threshold

        self.state = CircuitState.CLOSED
        self.failures = 0
        self.successes = 0
        self.last_failure_time = None

    async def call(self, coro):
        """
        Execute coroutine through circuit breaker

        Raises exception if circuit is open
        """
        if self.state == CircuitState.OPEN:
            # Check if timeout passed
            if datetime.now() - self.last_failure_time > timedelta(seconds=self.timeout):
                self.state = CircuitState.HALF_OPEN
                self.successes = 0
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = await coro()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        """Handle successful call"""
        self.failures = 0

        if self.state == CircuitState.HALF_OPEN:
            self.successes += 1
            if self.successes >= self.success_threshold:
                self.state = CircuitState.CLOSED

    def _on_failure(self):
        """Handle failed call"""
        self.failures += 1
        self.last_failure_time = datetime.now()

        if self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Global circuit breaker
external_api_breaker = CircuitBreaker(failure_threshold=5, timeout=60)

@app.get("/protected-endpoint")
async def protected_endpoint():
    """
    Endpoint protected by circuit breaker

    Pattern: Fail fast when external service is down
    """
    async def call_external_api():
        async with httpx.AsyncClient() as client:
            response = await client.get("https://api.example.com/data")
            response.raise_for_status()
            return response.json()

    try:
        data = await external_api_breaker.call(call_external_api)
        return {"data": data}
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))
```

## 🚫 Anti-Patterns to Avoid

### ❌ DON'T: Blocking Event Loop

```python
# BAD: Blocking call in async function
@app.get("/bad-async")
async def bad_async():
    """
    DON'T DO THIS

    Blocks event loop, preventing other requests
    """
    import time
    time.sleep(5)  # BLOCKS EVENT LOOP
    return {"message": "Bad"}

# GOOD: Use run_in_executor
@app.get("/good-async")
async def good_async():
    """
    CORRECT: Run blocking code in executor

    Event loop free to handle other requests
    """
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, blocking_function)
    return {"result": result}

def blocking_function():
    import time
    time.sleep(5)
    return "Done"
```

### ❌ DON'T: Unnecessary Async

```python
# BAD: Async with no I/O
@app.get("/bad-async-cpu")
async def bad_async_cpu():
    """
    DON'T DO THIS

    No I/O operations, async overhead for no benefit
    """
    result = sum(i * i for i in range(1000000))
    return {"result": result}

# GOOD: Use sync for CPU-bound
@app.get("/good-sync-cpu")
def good_sync_cpu():
    """
    CORRECT: Sync for CPU-bound work

    FastAPI runs in thread pool automatically
    """
    result = sum(i * i for i in range(1000000))
    return {"result": result}
```

### ❌ DON'T: Forgetting to Await

```python
# BAD: Not awaiting coroutine
@app.get("/bad-no-await")
async def bad_no_await():
    """
    DON'T DO THIS

    Returns coroutine object, not result
    """
    result = fetch_data("API")  # Missing await
    return {"result": result}  # Returns coroutine object

# GOOD: Always await coroutines
@app.get("/good-await")
async def good_await():
    """
    CORRECT: Await coroutine

    Returns actual result
    """
    result = await fetch_data("API")
    return {"result": result}
```

## ❓ Interview Questions

### Q1: What is the circuit breaker pattern?

**Answer**:

- **Fails fast** when external service is down
- **Three states**: CLOSED (normal), OPEN (failing), HALF_OPEN (testing)
- **Prevents cascading failures** in microservices
- Opens after threshold failures, closes after successful tests

### Q2: How to handle rate limiting in async code?

**Answer**:

- **Semaphore**: Limit concurrent operations
- **Token bucket**: Rate limit by tokens/second
- **Exponential backoff**: Retry with increasing delays
- **Circuit breaker**: Stop trying when service is down

### Q3: What are common async anti-patterns?

**Answer**:

- **Blocking event loop**: time.sleep(), synchronous I/O
- **Not awaiting coroutines**: Forgetting await
- **Unnecessary async**: CPU-bound code as async
- **Not handling exceptions**: No try/except in concurrent tasks

### Q4: How to optimize concurrent API calls?

**Answer**:

1. **Use asyncio.gather()** for concurrent execution
2. **Connection pooling** with httpx.AsyncClient
3. **Rate limiting** with semaphore
4. **Timeouts** to prevent hanging
5. **Retry with backoff** for transient failures

## 📚 Summary

**Key Takeaways**:

1. **Parallel requests** with asyncio.gather()
2. **Rate limiting** with Semaphore or TokenBucket
3. **Connection pooling** for database efficiency
4. **Retry with exponential backoff** for resilience
5. **Timeouts** prevent hanging requests
6. **Circuit breaker** fails fast when service down
7. **DON'T block event loop** with time.sleep()
8. **DON'T forget to await** coroutines
9. **Use sync for CPU-bound** work
10. **Error handling** with return_exceptions=True

Async patterns make FastAPI applications resilient and performant!
