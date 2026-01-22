# 🎯 Performance Profiling and Optimization

## Overview

Performance profiling identifies bottlenecks in code execution. Systematic profiling is essential before optimization - "Premature optimization is the root of all evil."

---

## Profiling Tools

### 1. **cProfile - Function-Level Profiling**

```python
import cProfile
import pstats
from pstats import SortKey

def slow_function():
    """Example slow function"""
    total = 0
    for i in range(1000000):
        total += i
    return total

def another_function():
    """Another function"""
    result = []
    for i in range(10000):
        result.append(i ** 2)
    return result

# Profile specific function
def profile_function():
    """Run profiling"""
    profiler = cProfile.Profile()
    profiler.enable()

    # Code to profile
    slow_function()
    another_function()

    profiler.disable()

    # Print statistics
    stats = pstats.Stats(profiler)
    stats.sort_stats(SortKey.CUMULATIVE)
    stats.print_stats(10)  # Top 10 functions

# Profile entire script
if __name__ == "__main__":
    cProfile.run('slow_function()', sort='cumtime')
```

### 2. **line_profiler - Line-by-Line Profiling**

```python
# Install: pip install line_profiler

from line_profiler import LineProfiler

@profile  # Add decorator when using kernprof
def optimize_me():
    """Function to profile line-by-line"""
    result = []

    # Slow: Creating new list each time
    for i in range(10000):
        result = result + [i]  # Line 1: Slow

    # Better: Using append
    result2 = []
    for i in range(10000):
        result2.append(i)  # Line 2: Better

    # Best: List comprehension
    result3 = [i for i in range(10000)]  # Line 3: Best

    return result3

# Manual profiling
def manual_line_profile():
    """Profile without decorator"""
    profiler = LineProfiler()
    profiler.add_function(optimize_me)
    profiler.enable()

    optimize_me()

    profiler.disable()
    profiler.print_stats()

# Run from command line:
# kernprof -l -v script.py
```

### 3. **memory_profiler - Memory Usage**

```python
# Install: pip install memory_profiler

from memory_profiler import profile

@profile
def memory_intensive():
    """Track memory usage"""
    # Bad: Creates multiple large objects
    big_list = [i for i in range(1000000)]
    big_list2 = [i * 2 for i in range(1000000)]

    # Better: Generator (lazy evaluation)
    big_gen = (i for i in range(1000000))

    return sum(big_gen)

# Run from command line:
# python -m memory_profiler script.py
```

### 4. **py-spy - Sampling Profiler**

```python
# Install: pip install py-spy

# Profile running process
# py-spy record -o profile.svg --pid 12345

# Profile from start
# py-spy record -o profile.svg -- python myapp.py

# Top-like interface
# py-spy top --pid 12345
```

---

## FastAPI Profiling

### 1. **Middleware-Based Profiling**

```python
import cProfile
import pstats
import io
from fastapi import Request

@app.middleware("http")
async def profile_middleware(request: Request, call_next):
    """Profile each request"""
    profiler = cProfile.Profile()
    profiler.enable()

    response = await call_next(request)

    profiler.disable()

    # Get statistics
    s = io.StringIO()
    stats = pstats.Stats(profiler, stream=s)
    stats.sort_stats('cumulative')
    stats.print_stats(20)

    # Log or return in debug mode
    if app.debug:
        print(s.getvalue())

    return response
```

### 2. **Endpoint-Specific Profiling**

```python
import functools
import time

def profile_endpoint(func):
    """Decorator to profile endpoint"""
    @functools.wraps(func)
    async def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()

        start_time = time.time()
        result = await func(*args, **kwargs)
        execution_time = time.time() - start_time

        profiler.disable()

        # Log statistics
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')

        print(f"\n{func.__name__} took {execution_time:.4f} seconds")
        stats.print_stats(10)

        return result

    return wrapper

@app.get("/slow-endpoint")
@profile_endpoint
async def slow_endpoint():
    """Profiled endpoint"""
    # Slow operations
    result = expensive_computation()
    return {"result": result}
```

---

## Database Query Profiling

### 1. **SQLAlchemy Query Timing**

```python
from sqlalchemy import event
from sqlalchemy.engine import Engine
import logging
import time

logging.basicConfig()
logger = logging.getLogger("sqlalchemy.engine")
logger.setLevel(logging.INFO)

@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    """Log query start"""
    conn.info.setdefault('query_start_time', []).append(time.time())

@event.listens_for(Engine, "after_cursor_execute")
def after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    """Log query completion with timing"""
    total_time = time.time() - conn.info['query_start_time'].pop()

    if total_time > 0.1:  # Log slow queries (>100ms)
        logger.warning(
            f"Slow query ({total_time:.4f}s): {statement[:100]}"
        )
```

### 2. **Query Execution Plans**

```python
from sqlalchemy import text

def analyze_query(db: Session, query):
    """Get query execution plan"""
    # PostgreSQL
    explain_query = f"EXPLAIN ANALYZE {query}"
    result = db.execute(text(explain_query))

    for row in result:
        print(row[0])

# Usage
query = """
SELECT u.*, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id
"""

analyze_query(db, query)
```

---

## Optimization Techniques

### 1. **List Comprehensions vs Loops**

```python
import timeit

# Slow: Regular loop
def loop_approach():
    result = []
    for i in range(10000):
        result.append(i ** 2)
    return result

# Fast: List comprehension
def comprehension_approach():
    return [i ** 2 for i in range(10000)]

# Fastest: Generator (for iteration)
def generator_approach():
    return (i ** 2 for i in range(10000))

# Benchmark
print("Loop:", timeit.timeit(loop_approach, number=1000))
print("Comprehension:", timeit.timeit(comprehension_approach, number=1000))
print("Generator:", timeit.timeit(lambda: list(generator_approach()), number=1000))

# Results:
# Loop: ~0.4s
# Comprehension: ~0.35s (12% faster)
# Generator: ~0.36s (similar, but memory efficient)
```

### 2. **String Concatenation**

```python
# Slow: String concatenation in loop
def slow_concat():
    result = ""
    for i in range(10000):
        result += str(i)  # Creates new string each time
    return result

# Fast: Join list
def fast_concat():
    parts = []
    for i in range(10000):
        parts.append(str(i))
    return "".join(parts)

# Fastest: Generator with join
def fastest_concat():
    return "".join(str(i) for i in range(10000))

# Benchmark
print("Slow:", timeit.timeit(slow_concat, number=100))    # ~2.5s
print("Fast:", timeit.timeit(fast_concat, number=100))    # ~0.4s (6x faster)
print("Fastest:", timeit.timeit(fastest_concat, number=100))  # ~0.35s (7x faster)
```

### 3. **Dictionary Lookups vs List Searches**

```python
# Slow: List search
def list_search():
    items = list(range(10000))
    return 9999 in items  # O(n) search

# Fast: Set membership
def set_search():
    items = set(range(10000))
    return 9999 in items  # O(1) search

# Fast: Dictionary lookup
def dict_search():
    items = {i: i for i in range(10000)}
    return 9999 in items  # O(1) search

# Benchmark
print("List:", timeit.timeit(list_search, number=10000))  # ~0.8s
print("Set:", timeit.timeit(set_search, number=10000))    # ~0.001s (800x faster)
print("Dict:", timeit.timeit(dict_search, number=10000))  # ~0.001s (800x faster)
```

### 4. **Function Call Overhead**

```python
# Slow: Function call in loop
def with_function_calls():
    def square(x):
        return x ** 2

    return [square(i) for i in range(10000)]

# Fast: Inline operation
def inline_operation():
    return [i ** 2 for i in range(10000)]

# Fast: Map with built-in
def map_builtin():
    return list(map(lambda x: x ** 2, range(10000)))

# Benchmark
print("Function calls:", timeit.timeit(with_function_calls, number=1000))
print("Inline:", timeit.timeit(inline_operation, number=1000))
print("Map:", timeit.timeit(map_builtin, number=1000))
```

### 5. **Local Variable Caching**

```python
import math

# Slow: Repeated attribute lookup
def slow_lookup():
    result = 0
    for i in range(10000):
        result += math.sqrt(i)  # Looks up math.sqrt each time
    return result

# Fast: Cache in local variable
def fast_lookup():
    result = 0
    sqrt = math.sqrt  # Cache attribute
    for i in range(10000):
        result += sqrt(i)
    return result

# Benchmark
print("Slow:", timeit.timeit(slow_lookup, number=1000))   # ~0.45s
print("Fast:", timeit.timeit(fast_lookup, number=1000))   # ~0.35s (25% faster)
```

---

## Async Performance

### 1. **Concurrent Requests**

```python
import asyncio
import httpx
import time

# Slow: Sequential requests
async def sequential_requests():
    """Make requests one by one"""
    start = time.time()

    async with httpx.AsyncClient() as client:
        for i in range(10):
            response = await client.get(f"https://api.example.com/users/{i}")

    print(f"Sequential: {time.time() - start:.2f}s")

# Fast: Concurrent requests
async def concurrent_requests():
    """Make requests concurrently"""
    start = time.time()

    async with httpx.AsyncClient() as client:
        tasks = [
            client.get(f"https://api.example.com/users/{i}")
            for i in range(10)
        ]
        responses = await asyncio.gather(*tasks)

    print(f"Concurrent: {time.time() - start:.2f}s")

# Sequential: ~5s (10 requests * 0.5s each)
# Concurrent: ~0.5s (all requests at once)
```

### 2. **Database Connection Pooling**

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Optimized connection pool
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,          # Number of connections to keep
    max_overflow=10,       # Additional connections when needed
    pool_pre_ping=True,    # Verify connection before use
    pool_recycle=3600,     # Recycle connections after 1 hour
    echo_pool=True         # Log pool activity
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)
```

---

## Caching Strategies

### 1. **Function Result Caching**

```python
from functools import lru_cache
import time

# Without cache
def expensive_computation(n):
    """Slow computation"""
    time.sleep(1)
    return n ** 2

# With cache
@lru_cache(maxsize=128)
def cached_computation(n):
    """Cached computation"""
    time.sleep(1)
    return n ** 2

# First call: 1s
start = time.time()
result1 = cached_computation(10)
print(f"First call: {time.time() - start:.2f}s")  # ~1s

# Second call: Instant
start = time.time()
result2 = cached_computation(10)
print(f"Second call: {time.time() - start:.2f}s")  # ~0.00s

# Check cache statistics
print(cached_computation.cache_info())
# CacheInfo(hits=1, misses=1, maxsize=128, currsize=1)
```

### 2. **Redis Caching**

```python
import redis
import json
import functools

redis_client = redis.Redis(host='localhost', port=6379)

def redis_cache(expire: int = 300):
    """Redis caching decorator"""
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key
            key = f"cache:{func.__name__}:{args}:{kwargs}"

            # Try to get from cache
            cached = redis_client.get(key)
            if cached:
                return json.loads(cached)

            # Compute result
            result = await func(*args, **kwargs)

            # Cache result
            redis_client.setex(
                key,
                expire,
                json.dumps(result)
            )

            return result

        return wrapper
    return decorator

@app.get("/products/{product_id}")
@redis_cache(expire=600)  # Cache for 10 minutes
async def get_product(product_id: int):
    """Cached product lookup"""
    # Expensive database query
    product = await db.query(Product).filter(
        Product.id == product_id
    ).first()

    return product
```

---

## Monitoring Performance

### 1. **Prometheus Metrics**

```python
from prometheus_client import Counter, Histogram, Gauge
import time

# Define metrics
request_count = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    """Collect performance metrics"""
    start_time = time.time()

    # Process request
    response = await call_next(request)

    # Record metrics
    duration = time.time() - start_time

    request_count.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    request_duration.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)

    # Add timing header
    response.headers["X-Process-Time"] = str(duration)

    return response
```

---

## Best Practices

### ✅ Do's:

1. **Profile before optimizing** - identify real bottlenecks
2. **Use appropriate data structures** (dict over list for lookups)
3. **Cache expensive operations**
4. **Use generators** for large datasets
5. **Pool database connections**
6. **Make concurrent requests** when possible
7. **Monitor production performance**
8. **Set performance budgets**

### ❌ Don'ts:

1. **Don't optimize prematurely**
2. **Don't guess** - always measure
3. **Don't sacrifice readability** for micro-optimizations
4. **Don't ignore database** query performance
5. **Don't forget about memory** usage
6. **Don't over-cache** (stale data issues)

---

## Interview Questions

### Q1: How do you identify performance bottlenecks?

**Answer**:

1. Use cProfile for function-level profiling
2. Use line_profiler for line-by-line analysis
3. Use py-spy for production profiling
4. Monitor database query times
5. Use APM tools (New Relic, DataDog)
6. Analyze production metrics (response times, error rates)

### Q2: What's the difference between profiling and benchmarking?

**Answer**:

- **Profiling**: Identifies where time is spent in code
- **Benchmarking**: Measures execution time of specific operations
  Use profiling to find bottlenecks, benchmarking to compare solutions.

### Q3: How do you optimize database queries?

**Answer**:

- Add appropriate indexes
- Use select_related/prefetch_related
- Avoid N+1 queries
- Use query only needed columns
- Implement pagination
- Use database connection pooling
- Cache query results
- Analyze execution plans

### Q4: What is the cost of function calls in Python?

**Answer**: Function calls have overhead:

- Name lookup
- Frame creation
- Argument packing/unpacking
  For hot loops, inline operations or use built-ins (C-level functions).

### Q5: How do you handle memory leaks in Python?

**Answer**:

- Use memory_profiler to identify leaks
- Check for circular references
- Close database connections
- Clear caches periodically
- Use weak references when appropriate
- Monitor memory usage in production

---

## Summary

Performance optimization process:

1. **Profile** to identify bottlenecks
2. **Measure** current performance
3. **Optimize** the slowest parts
4. **Verify** improvements with benchmarks
5. **Monitor** in production

Tools: cProfile, line_profiler, py-spy, memory_profiler

Common optimizations: list comprehensions, dictionary lookups, caching, async concurrency, database indexing

Remember: Profile first, optimize second! 🎯
