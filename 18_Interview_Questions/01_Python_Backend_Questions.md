# 💬 Interview Questions - Python Backend

## Overview

Comprehensive collection of real interview questions for Python backend developers (4-8 years experience) with detailed answers.

---

## Python Fundamentals

### Q1: Explain GIL and its implications

**Answer**: Global Interpreter Lock ensures only one thread executes Python bytecode at a time:

- **Purpose**: Simplifies memory management, prevents race conditions
- **Impact**: Limits multi-threading for CPU-bound tasks
- **Workarounds**: Use multiprocessing for CPU-bound, async for I/O-bound
- **When it matters**: Heavy computation with threads
- **Releases**: During I/O operations, long-running C extensions

```python
# GIL impact example
import threading
import time

def cpu_task():
    count = 0
    for i in range(10_000_000):
        count += i
    return count

# With threads (limited by GIL)
start = time.time()
threads = [threading.Thread(target=cpu_task) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(f"Threads: {time.time() - start:.2f}s")  # ~4 seconds

# With processes (bypasses GIL)
from multiprocessing import Process
start = time.time()
processes = [Process(target=cpu_task) for _ in range(4)]
for p in processes: p.start()
for p in processes: p.join()
print(f"Processes: {time.time() - start:.2f}s")  # ~1 second
```

### Q2: Difference between `@staticmethod` and `@classmethod`

**Answer**:

- **`@staticmethod`**: No access to class or instance, just namespace
- **`@classmethod`**: Receives class as first argument (cls)
- **Use static**: Utility functions related to class
- **Use class**: Alternative constructors, factory methods

```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    @staticmethod
    def is_leap_year(year):
        """Static - no access to class/instance"""
        return year % 4 == 0 and (year % 100 != 0 or year % 400 == 0)

    @classmethod
    def from_string(cls, date_string):
        """Class method - factory pattern"""
        year, month, day = map(int, date_string.split('-'))
        return cls(year, month, day)  # Returns instance

# Usage
print(Date.is_leap_year(2024))  # True
date = Date.from_string("2024-01-15")
```

### Q3: What are generators and why use them?

**Answer**: Functions that yield values one at a time:

- **Memory efficient**: Don't store entire sequence
- **Lazy evaluation**: Compute on demand
- **Can be infinite**: Generate unlimited values
- **Pipeline processing**: Chain generators

```python
# Regular function - loads everything
def get_numbers_list(n):
    return [i ** 2 for i in range(n)]

numbers = get_numbers_list(1_000_000)  # Uses ~40MB memory

# Generator - yields one at a time
def get_numbers_gen(n):
    for i in range(n):
        yield i ** 2

numbers = get_numbers_gen(1_000_000)  # Uses minimal memory

# Infinite generator
def fibonacci():
    """Generate infinite Fibonacci sequence"""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# Pipeline processing
def even_only(numbers):
    for num in numbers:
        if num % 2 == 0:
            yield num

def under_hundred(numbers):
    for num in numbers:
        if num < 100:
            yield num

# Chain generators
fib = fibonacci()
evens = even_only(fib)
small_evens = under_hundred(evens)

result = [next(small_evens) for _ in range(10)]
```

---

## Django & FastAPI

### Q4: How does Django ORM handle lazy loading?

**Answer**: Queries are lazy - executed only when evaluated:

- **Lazy**: QuerySet creation doesn't hit database
- **Evaluation triggers**: Iteration, len(), list(), bool()
- **select_related**: Eager load foreign keys (JOIN)
- **prefetch_related**: Eager load many-to-many (separate query)

```python
# Lazy - no query yet
users = User.objects.filter(is_active=True)

# Now queries execute
for user in users:  # Query executed here
    print(user.name)

# N+1 problem
users = User.objects.all()
for user in users:
    print(user.profile.bio)  # N queries for profiles

# Solution: select_related (ForeignKey/OneToOne)
users = User.objects.select_related('profile').all()  # Single JOIN query

# Solution: prefetch_related (ManyToMany/reverse FK)
users = User.objects.prefetch_related('posts').all()  # 2 queries total
```

### Q5: How does FastAPI dependency injection work?

**Answer**: FastAPI resolves dependencies automatically:

- **Declare with Depends()**: FastAPI calls function
- **Caching**: Same dependency called once per request
- **Nested dependencies**: Dependencies can have dependencies
- **Async support**: Can be async functions

```python
from fastapi import Depends, HTTPException

# Dependency
def get_db():
    """Database session dependency"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Nested dependency
def get_current_user(
    token: str = Header(...),
    db: Session = Depends(get_db)
):
    """Get user from token"""
    user = verify_token(token, db)
    if not user:
        raise HTTPException(401, "Invalid token")
    return user

# Use dependencies
@app.get("/posts")
async def get_posts(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Dependencies resolved automatically"""
    posts = db.query(Post).filter(
        Post.user_id == current_user.id
    ).all()
    return posts
```

---

## System Design

### Q6: How would you design a URL shortener?

**Answer**:

```python
"""
Requirements:
- Shorten long URLs to short codes
- Redirect short codes to original URLs
- Track clicks
- Custom URLs optional

Design:

1. Database Schema:
   urls:
     - id (auto-increment)
     - original_url
     - short_code (unique, indexed)
     - user_id
     - created_at
     - expires_at

   clicks:
     - id
     - url_id
     - clicked_at
     - ip_address
     - user_agent

2. Short Code Generation:
   - Base62 encode (a-z, A-Z, 0-9)
   - 6 characters = 62^6 = 56 billion URLs
   - Hash(url) -> Base62 -> Check collision

3. Scale Considerations:
   - Cache frequently accessed URLs (Redis)
   - Partition by short_code hash
   - Read-heavy: Read replicas
   - Rate limiting per user
"""

import hashlib
import string

class URLShortener:
    """URL shortener implementation"""

    ALPHABET = string.ascii_letters + string.digits  # Base62

    @staticmethod
    def encode_id(num: int) -> str:
        """Convert ID to base62"""
        if num == 0:
            return URLShortener.ALPHABET[0]

        result = []
        base = len(URLShortener.ALPHABET)

        while num:
            result.append(URLShortener.ALPHABET[num % base])
            num //= base

        return ''.join(reversed(result))

    @staticmethod
    def decode(short_code: str) -> int:
        """Convert base62 to ID"""
        base = len(URLShortener.ALPHABET)
        num = 0

        for char in short_code:
            num = num * base + URLShortener.ALPHABET.index(char)

        return num

# API Implementation
@app.post("/shorten")
async def shorten_url(
    url: str,
    custom: Optional[str] = None,
    db: Session = Depends(get_db)
):
    """Shorten URL"""
    # Check custom code
    if custom:
        existing = db.query(URL).filter(URL.short_code == custom).first()
        if existing:
            raise HTTPException(400, "Custom code taken")
        short_code = custom
    else:
        # Create URL entry
        url_entry = URL(original_url=url)
        db.add(url_entry)
        db.commit()

        # Generate short code from ID
        short_code = URLShortener.encode_id(url_entry.id)
        url_entry.short_code = short_code
        db.commit()

    return {"short_url": f"https://short.url/{short_code}"}

@app.get("/{short_code}")
async def redirect_url(
    short_code: str,
    db: Session = Depends(get_db)
):
    """Redirect to original URL"""
    # Try cache first
    cached = redis_client.get(f"url:{short_code}")
    if cached:
        original_url = cached
    else:
        # Query database
        url_entry = db.query(URL).filter(
            URL.short_code == short_code
        ).first()

        if not url_entry:
            raise HTTPException(404, "URL not found")

        original_url = url_entry.original_url

        # Cache for 1 hour
        redis_client.setex(f"url:{short_code}", 3600, original_url)

    # Track click asynchronously
    asyncio.create_task(track_click(short_code, request))

    return RedirectResponse(original_url)
```

### Q7: Design a rate limiter

**Answer**:

```python
"""
Requirements:
- Limit requests per user/IP
- Different limits per endpoint
- Distributed (multiple servers)

Algorithms:
1. Fixed Window: Simple, burst at boundaries
2. Sliding Window: More accurate, complex
3. Token Bucket: Allows bursts, maintains rate
4. Leaky Bucket: Smooths traffic

Implementation: Sliding Window with Redis
"""

import redis
import time

class DistributedRateLimiter:
    """Redis-based sliding window rate limiter"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def is_allowed(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> bool:
        """Check if request is allowed"""
        now = time.time()
        window_start = now - window_seconds

        # Use sorted set with timestamps as scores
        pipe = self.redis.pipeline()

        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)

        # Count requests in window
        pipe.zcard(key)

        # Add current request
        pipe.zadd(key, {str(now): now})

        # Set expiry
        pipe.expire(key, window_seconds + 1)

        # Execute pipeline
        results = pipe.execute()
        request_count = results[1]

        return request_count < max_requests

# FastAPI Integration
limiter = DistributedRateLimiter(redis_client)

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    """Rate limiting middleware"""
    # Generate key
    key = f"ratelimit:{request.client.host}:{request.url.path}"

    # Check limit
    if not limiter.is_allowed(key, max_requests=100, window_seconds=60):
        return JSONResponse(
            status_code=429,
            content={"error": "Rate limit exceeded"}
        )

    return await call_next(request)
```

---

## Performance & Optimization

### Q8: How do you identify and fix performance bottlenecks?

**Answer**:

```python
"""
Process:
1. Profile code (cProfile, py-spy)
2. Identify hotspots
3. Optimize (algorithm, caching, async)
4. Measure improvement
5. Repeat

Tools:
- cProfile: Function-level profiling
- line_profiler: Line-by-line profiling
- memory_profiler: Memory usage
- py-spy: Production profiling
- Django Debug Toolbar: Database queries
"""

# Example: Optimize slow endpoint

# Step 1: Profile
import cProfile
import pstats

def profile_endpoint():
    profiler = cProfile.Profile()
    profiler.enable()

    # Call slow endpoint
    response = client.get("/api/users")

    profiler.disable()
    stats = pstats.Stats(profiler)
    stats.sort_stats('cumulative')
    stats.print_stats(20)

# Step 2: Found issue - N+1 queries

# Before (slow)
def get_users_slow():
    users = db.query(User).all()  # 1 query
    result = []
    for user in users:
        result.append({
            "id": user.id,
            "name": user.name,
            "posts": [p.title for p in user.posts]  # N queries
        })
    return result

# After (fast)
from sqlalchemy.orm import selectinload

def get_users_fast():
    users = db.query(User)\
        .options(selectinload(User.posts))\
        .all()  # 2 queries total

    result = []
    for user in users:
        result.append({
            "id": user.id,
            "name": user.name,
            "posts": [p.title for p in user.posts]  # Already loaded
        })
    return result

# Step 3: Add caching
from functools import lru_cache

@lru_cache(maxsize=128)
def get_user_stats(user_id):
    """Cache expensive computation"""
    # Complex aggregation
    stats = db.query(
        func.count(Post.id),
        func.sum(Post.views)
    ).filter(Post.user_id == user_id).first()

    return {"post_count": stats[0], "total_views": stats[1]}
```

### Q9: Explain caching strategies

**Answer**:

```python
"""
Caching Levels:
1. Browser cache (HTTP headers)
2. CDN cache (static assets)
3. Application cache (Redis)
4. Database cache (query cache)

Strategies:
- Cache-aside: App manages cache
- Write-through: Write to cache and DB
- Write-behind: Write to cache, async to DB
- Read-through: Cache loads from DB automatically
"""

# Cache-aside pattern
class CacheAsideRepository:
    """Cache-aside implementation"""

    def __init__(self, db, cache):
        self.db = db
        self.cache = cache

    def get_user(self, user_id: int):
        """Get user with caching"""
        # Try cache first
        cache_key = f"user:{user_id}"
        cached = self.cache.get(cache_key)

        if cached:
            return json.loads(cached)

        # Cache miss - query database
        user = self.db.query(User).filter(
            User.id == user_id
        ).first()

        if user:
            # Store in cache
            self.cache.setex(
                cache_key,
                3600,  # 1 hour TTL
                json.dumps(user.to_dict())
            )

        return user

    def update_user(self, user_id: int, data: dict):
        """Update user and invalidate cache"""
        # Update database
        user = self.db.query(User).filter(
            User.id == user_id
        ).first()

        for key, value in data.items():
            setattr(user, key, value)

        self.db.commit()

        # Invalidate cache
        self.cache.delete(f"user:{user_id}")

        return user
```

---

## Security

### Q10: How do you prevent SQL injection?

**Answer**:

```python
"""
SQL Injection: Attacker manipulates SQL queries via input

Prevention:
1. Parameterized queries (BEST)
2. ORM usage
3. Input validation
4. Least privilege DB accounts
"""

# ❌ VULNERABLE: String concatenation
def get_user_vulnerable(username):
    query = f"SELECT * FROM users WHERE username = '{username}'"
    # Attacker input: ' OR '1'='1
    # Results in: SELECT * FROM users WHERE username = '' OR '1'='1'
    # Returns all users!
    return db.execute(query)

# ✅ SECURE: Parameterized query
def get_user_secure(username):
    query = text("SELECT * FROM users WHERE username = :username")
    return db.execute(query, {"username": username})

# ✅ SECURE: ORM (automatically safe)
def get_user_orm(username):
    return db.query(User).filter(User.username == username).first()

# Additional security layers
def validate_input(username):
    """Whitelist validation"""
    if not re.match(r'^[a-zA-Z0-9_-]{3,20}$', username):
        raise ValueError("Invalid username format")
    return username

# Use least privilege
"""
-- Create read-only user for SELECT queries
CREATE USER app_readonly WITH PASSWORD 'password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;

-- Create limited user for app
CREATE USER app_user WITH PASSWORD 'password';
GRANT SELECT, INSERT, UPDATE ON users, posts TO app_user;
-- No DELETE, DROP, or admin privileges
"""
```

---

## Best Practices

### ✅ Interview Tips:

1. **Think aloud** - explain your thought process
2. **Ask clarifying questions** - understand requirements
3. **Start simple** - working solution first, optimize later
4. **Consider trade-offs** - discuss pros/cons
5. **Test edge cases** - empty input, large input, errors
6. **Discuss scalability** - how it handles growth
7. **Code quality** - clean, readable, documented

### ❌ Common Mistakes:

1. **Jumping to code** without understanding problem
2. **Over-engineering** - complex when simple works
3. **Ignoring edge cases**
4. **Not explaining reasoning**
5. **Giving up too quickly**
6. **Not asking questions**

---

## Summary

Key interview topics:

- **Python internals**: GIL, generators, decorators
- **Frameworks**: Django ORM, FastAPI dependency injection
- **System design**: Scalability, caching, distributed systems
- **Performance**: Profiling, optimization, N+1 queries
- **Security**: SQL injection, authentication, authorization
- **Best practices**: SOLID, testing, documentation

Practice explaining concepts clearly! 💬
