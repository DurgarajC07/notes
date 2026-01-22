# 🚀 Caching Strategies

## Overview

Effective caching reduces database load, improves response times, and enhances scalability. Understanding different caching strategies is crucial for high-performance applications.

---

## Cache Patterns

### Cache-Aside (Lazy Loading)

```python
import redis
import json
from functools import wraps

# Redis client
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Cache-aside pattern
class UserRepository:
    """User repository with cache-aside"""

    def get_user(self, user_id: int):
        """Get user with caching"""
        # Try cache first
        cache_key = f"user:{user_id}"
        cached_data = redis_client.get(cache_key)

        if cached_data:
            print("Cache hit")
            return json.loads(cached_data)

        print("Cache miss")
        # Query database
        user = db.query(User).filter(User.id == user_id).first()

        if user:
            # Store in cache (1 hour TTL)
            redis_client.setex(
                cache_key,
                3600,
                json.dumps(user.to_dict())
            )

        return user

    def update_user(self, user_id: int, data: dict):
        """Update user and invalidate cache"""
        # Update database
        user = db.query(User).filter(User.id == user_id).first()
        for key, value in data.items():
            setattr(user, key, value)
        db.commit()

        # Invalidate cache
        redis_client.delete(f"user:{user_id}")

        return user

# Generic cache-aside decorator
def cache_aside(key_pattern: str, ttl: int = 3600):
    """Cache-aside decorator"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generate cache key
            cache_key = key_pattern.format(*args, **kwargs)

            # Try cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Execute function
            result = func(*args, **kwargs)

            # Cache result
            if result is not None:
                redis_client.setex(
                    cache_key,
                    ttl,
                    json.dumps(result)
                )

            return result

        return wrapper
    return decorator

# Usage
@cache_aside(key_pattern="user:{}", ttl=3600)
def get_user_by_id(user_id: int):
    """Get user from database"""
    user = db.query(User).filter(User.id == user_id).first()
    return user.to_dict() if user else None
```

### Read-Through Cache

```python
class ReadThroughCache:
    """Cache that loads data automatically"""

    def __init__(self, redis_client, loader_func, ttl=3600):
        self.redis = redis_client
        self.loader = loader_func
        self.ttl = ttl

    def get(self, key: str):
        """Get from cache or load"""
        # Try cache
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        # Load from source
        data = self.loader(key)

        # Cache it
        if data is not None:
            self.redis.setex(key, self.ttl, json.dumps(data))

        return data

    def invalidate(self, key: str):
        """Invalidate cache entry"""
        self.redis.delete(key)

# Usage
def load_user(user_id: str):
    """Load user from database"""
    user_id_int = int(user_id.split(':')[1])
    user = db.query(User).filter(User.id == user_id_int).first()
    return user.to_dict() if user else None

user_cache = ReadThroughCache(
    redis_client,
    loader_func=load_user,
    ttl=3600
)

# Get user (automatically loads if not in cache)
user = user_cache.get("user:123")
```

### Write-Through Cache

```python
class WriteThroughCache:
    """Write to cache and database simultaneously"""

    def __init__(self, redis_client, ttl=3600):
        self.redis = redis_client
        self.ttl = ttl

    def set(self, key: str, value: dict):
        """Write to both cache and database"""
        # Write to database
        db.execute(
            "INSERT INTO cache_data (key, value) VALUES (:key, :value) "
            "ON CONFLICT (key) DO UPDATE SET value = :value",
            {"key": key, "value": json.dumps(value)}
        )
        db.commit()

        # Write to cache
        self.redis.setex(key, self.ttl, json.dumps(value))

    def get(self, key: str):
        """Read from cache"""
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        # Fallback to database
        result = db.execute(
            "SELECT value FROM cache_data WHERE key = :key",
            {"key": key}
        ).fetchone()

        if result:
            return json.loads(result[0])

        return None

# Usage for user updates
class UserService:
    """User service with write-through cache"""

    def __init__(self, cache):
        self.cache = cache

    def update_user(self, user_id: int, data: dict):
        """Update user with write-through"""
        # Update database
        user = db.query(User).filter(User.id == user_id).first()
        for key, value in data.items():
            setattr(user, key, value)
        db.commit()

        # Update cache
        self.cache.set(f"user:{user_id}", user.to_dict())

        return user
```

### Write-Behind (Write-Back) Cache

```python
import asyncio
from queue import Queue
import threading

class WriteBehindCache:
    """Write to cache immediately, database asynchronously"""

    def __init__(self, redis_client, batch_size=100, flush_interval=5):
        self.redis = redis_client
        self.batch_size = batch_size
        self.flush_interval = flush_interval
        self.write_queue = Queue()

        # Start background writer
        self.writer_thread = threading.Thread(target=self._background_writer, daemon=True)
        self.writer_thread.start()

    def set(self, key: str, value: dict):
        """Write to cache, queue for database"""
        # Immediate cache write
        self.redis.setex(key, 3600, json.dumps(value))

        # Queue for database write
        self.write_queue.put((key, value))

    def _background_writer(self):
        """Background thread to flush to database"""
        batch = []

        while True:
            try:
                # Collect batch
                while len(batch) < self.batch_size:
                    try:
                        item = self.write_queue.get(timeout=self.flush_interval)
                        batch.append(item)
                    except:
                        break

                # Flush batch to database
                if batch:
                    self._flush_batch(batch)
                    batch = []

            except Exception as e:
                print(f"Error in background writer: {e}")

    def _flush_batch(self, batch):
        """Flush batch to database"""
        # Bulk insert/update
        values = [
            {"key": key, "value": json.dumps(value)}
            for key, value in batch
        ]

        db.execute(
            "INSERT INTO cache_data (key, value) VALUES (:key, :value) "
            "ON CONFLICT (key) DO UPDATE SET value = :value",
            values
        )
        db.commit()
```

---

## In-Memory Caching

### LRU Cache (functools)

```python
from functools import lru_cache

# Simple LRU cache
@lru_cache(maxsize=128)
def fibonacci(n):
    """Fibonacci with memoization"""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# Check cache stats
print(fibonacci.cache_info())
# CacheInfo(hits=0, misses=0, maxsize=128, currsize=0)

# Clear cache
fibonacci.cache_clear()

# LRU cache for methods
from functools import cached_property

class User:
    """User with cached property"""

    def __init__(self, user_id):
        self.user_id = user_id

    @cached_property
    def profile(self):
        """Expensive profile lookup (cached)"""
        print("Loading profile...")
        return db.query(Profile).filter(
            Profile.user_id == self.user_id
        ).first()

user = User(123)
profile1 = user.profile  # Loads from DB
profile2 = user.profile  # Returns cached (no DB query)

# Custom LRU cache with TTL
from collections import OrderedDict
import time

class TTLCache:
    """LRU cache with TTL"""

    def __init__(self, maxsize=128, ttl=300):
        self.maxsize = maxsize
        self.ttl = ttl
        self.cache = OrderedDict()
        self.timestamps = {}

    def get(self, key):
        """Get from cache"""
        if key not in self.cache:
            return None

        # Check TTL
        if time.time() - self.timestamps[key] > self.ttl:
            del self.cache[key]
            del self.timestamps[key]
            return None

        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def set(self, key, value):
        """Set in cache"""
        if key in self.cache:
            self.cache.move_to_end(key)
        else:
            # Evict if full
            if len(self.cache) >= self.maxsize:
                oldest = next(iter(self.cache))
                del self.cache[oldest]
                del self.timestamps[oldest]

        self.cache[key] = value
        self.timestamps[key] = time.time()

# Usage
cache = TTLCache(maxsize=100, ttl=300)
cache.set("key1", "value1")
value = cache.get("key1")
```

---

## Distributed Caching

### Redis Caching Patterns

```python
import redis
import json
import hashlib
from typing import Optional, Any

class RedisCache:
    """Redis cache wrapper"""

    def __init__(self, host='localhost', port=6379):
        self.redis = redis.Redis(
            host=host,
            port=port,
            decode_responses=True
        )

    def get(self, key: str) -> Optional[Any]:
        """Get from cache"""
        value = self.redis.get(key)
        return json.loads(value) if value else None

    def set(self, key: str, value: Any, ttl: int = 3600):
        """Set in cache"""
        self.redis.setex(key, ttl, json.dumps(value))

    def delete(self, key: str):
        """Delete from cache"""
        self.redis.delete(key)

    def exists(self, key: str) -> bool:
        """Check if key exists"""
        return self.redis.exists(key) > 0

    def get_many(self, keys: list) -> list:
        """Get multiple keys"""
        values = self.redis.mget(keys)
        return [json.loads(v) if v else None for v in values]

    def set_many(self, mapping: dict, ttl: int = 3600):
        """Set multiple keys"""
        pipeline = self.redis.pipeline()

        for key, value in mapping.items():
            pipeline.setex(key, ttl, json.dumps(value))

        pipeline.execute()

    def increment(self, key: str, amount: int = 1) -> int:
        """Increment counter"""
        return self.redis.incrby(key, amount)

    def expire(self, key: str, ttl: int):
        """Set expiration"""
        self.redis.expire(key, ttl)

# Query result caching
class QueryCache:
    """Cache database query results"""

    def __init__(self, redis_cache: RedisCache):
        self.cache = redis_cache

    def cache_query(self, ttl: int = 300):
        """Decorator to cache query results"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                # Generate cache key from function and args
                key_data = f"{func.__name__}:{args}:{kwargs}"
                cache_key = hashlib.md5(key_data.encode()).hexdigest()

                # Try cache
                cached = self.cache.get(cache_key)
                if cached is not None:
                    return cached

                # Execute query
                result = func(*args, **kwargs)

                # Cache result
                self.cache.set(cache_key, result, ttl)

                return result

            return wrapper
        return decorator

# Usage
cache = RedisCache()
query_cache = QueryCache(cache)

@query_cache.cache_query(ttl=600)
def get_user_posts(user_id: int):
    """Get user posts (cached)"""
    posts = db.query(Post).filter(Post.user_id == user_id).all()
    return [p.to_dict() for p in posts]
```

### Cache Invalidation Strategies

```python
class CacheInvalidator:
    """Cache invalidation strategies"""

    def __init__(self, redis_cache: RedisCache):
        self.cache = redis_cache

    def invalidate_pattern(self, pattern: str):
        """Invalidate all keys matching pattern"""
        keys = self.cache.redis.keys(pattern)
        if keys:
            self.cache.redis.delete(*keys)

    def invalidate_tags(self, tags: list):
        """Invalidate by tags"""
        for tag in tags:
            # Get keys with tag
            tag_key = f"tag:{tag}"
            keys = self.cache.redis.smembers(tag_key)

            if keys:
                # Delete all tagged keys
                self.cache.redis.delete(*keys)
                # Delete tag set
                self.cache.redis.delete(tag_key)

    def tag_cache_entry(self, key: str, tags: list):
        """Tag a cache entry"""
        for tag in tags:
            tag_key = f"tag:{tag}"
            self.cache.redis.sadd(tag_key, key)

# Usage
invalidator = CacheInvalidator(cache)

# Cache with tags
cache.set("post:123", post_data)
invalidator.tag_cache_entry("post:123", ["user:456", "posts"])

# Invalidate all posts by user
invalidator.invalidate_tags(["user:456"])

# Invalidate by pattern
invalidator.invalidate_pattern("user:*")
```

---

## HTTP Caching

### Cache Headers

```python
from fastapi import FastAPI, Response
from datetime import datetime, timedelta

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int, response: Response):
    """Get user with cache headers"""
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        raise HTTPException(404, "User not found")

    # Cache-Control header
    response.headers["Cache-Control"] = "public, max-age=3600"  # 1 hour

    # ETag for conditional requests
    etag = hashlib.md5(json.dumps(user.to_dict()).encode()).hexdigest()
    response.headers["ETag"] = etag

    # Last-Modified header
    response.headers["Last-Modified"] = user.updated_at.strftime(
        "%a, %d %b %Y %H:%M:%S GMT"
    )

    return user.to_dict()

@app.get("/api/data")
async def get_data(
    response: Response,
    if_none_match: Optional[str] = Header(None)
):
    """Handle conditional requests with ETag"""
    data = get_expensive_data()

    # Calculate ETag
    etag = hashlib.md5(json.dumps(data).encode()).hexdigest()

    # Check If-None-Match header
    if if_none_match == etag:
        # Data hasn't changed
        return Response(status_code=304)  # Not Modified

    # Set ETag header
    response.headers["ETag"] = etag
    response.headers["Cache-Control"] = "max-age=3600"

    return data

# Vary header for personalized content
@app.get("/dashboard")
async def get_dashboard(
    response: Response,
    authorization: str = Header(...)
):
    """Dashboard with user-specific caching"""
    user = get_current_user(authorization)

    # Cache varies by Authorization header
    response.headers["Vary"] = "Authorization"
    response.headers["Cache-Control"] = "private, max-age=300"

    return get_user_dashboard(user)
```

---

## Cache Warming

### Preloading Cache

```python
import asyncio

class CacheWarmer:
    """Warm cache with frequently accessed data"""

    def __init__(self, cache: RedisCache):
        self.cache = cache

    async def warm_user_cache(self, user_ids: list):
        """Warm cache for multiple users"""
        async def load_user(user_id):
            user = db.query(User).filter(User.id == user_id).first()
            if user:
                self.cache.set(f"user:{user_id}", user.to_dict(), ttl=3600)

        # Load concurrently
        await asyncio.gather(*[load_user(uid) for uid in user_ids])

    def warm_popular_content(self):
        """Warm cache with popular content"""
        # Get popular posts (e.g., high views)
        popular_posts = db.query(Post)\
            .order_by(Post.views.desc())\
            .limit(100)\
            .all()

        # Cache them
        for post in popular_posts:
            self.cache.set(
                f"post:{post.id}",
                post.to_dict(),
                ttl=7200  # 2 hours
            )

    def schedule_warming(self):
        """Schedule periodic cache warming"""
        from apscheduler.schedulers.background import BackgroundScheduler

        scheduler = BackgroundScheduler()

        # Warm cache every hour
        scheduler.add_job(
            self.warm_popular_content,
            'interval',
            hours=1
        )

        scheduler.start()

# Usage on application startup
@app.on_event("startup")
async def startup_event():
    """Warm cache on startup"""
    warmer = CacheWarmer(cache)

    # Warm critical data
    await warmer.warm_user_cache([1, 2, 3, 4, 5])
    warmer.warm_popular_content()
```

---

## Best Practices

### ✅ Do's:

1. **Set appropriate TTL** for each cache entry
2. **Invalidate on updates** - keep cache fresh
3. **Use cache keys** consistently
4. **Monitor cache hit rate** (aim for >80%)
5. **Handle cache misses** gracefully
6. **Use compression** for large values
7. **Implement circuit breaker** for cache failures
8. **Cache at right layer** (DB, API, CDN)
9. **Use distributed cache** for scalability
10. **Version cache keys** for schema changes

### ❌ Don'ts:

1. **Don't cache everything** - cost vs benefit
2. **Don't set infinite TTL** - data gets stale
3. **Don't cache user-specific** data globally
4. **Don't ignore cache failures** - have fallback
5. **Don't forget to invalidate** on updates
6. **Don't cache frequently** changing data
7. **Don't use caching** as primary storage

---

## Interview Questions

### Q1: Explain cache-aside pattern.

**Answer**: Application manages cache explicitly:

- Check cache first
- On miss, load from database and cache
- On update, invalidate cache
  Most common pattern, simple to implement.

### Q2: What is cache invalidation and why is it hard?

**Answer**: Removing stale data from cache:

- **Challenges**: Knowing when data changed, distributed caches, race conditions
- **Strategies**: TTL, explicit invalidation, event-driven
  "Two hard things: naming and cache invalidation"

### Q3: Explain different cache strategies.

**Answer**:

- **Cache-aside**: App manages cache
- **Read-through**: Cache loads automatically
- **Write-through**: Update cache and DB together
- **Write-behind**: Update cache, async DB

### Q4: How do you handle cache stampede?

**Answer**: Many requests hit DB when cache expires:

- **Solution**: Lock on cache miss (only one loads)
- **Probabilistic expiry**: Random TTL
- **Cache warming**: Preload before expiry
- **Stale-while-revalidate**: Serve stale while refreshing

### Q5: What's the difference between Redis and Memcached?

**Answer**:

- **Redis**: Persistence, data structures, pub/sub, more features
- **Memcached**: Simple, faster for simple key-value, less memory overhead
  Use Redis for complex caching, Memcached for pure caching.

---

## Summary

Caching essentials:

- **Patterns**: Cache-aside, read-through, write-through, write-behind
- **In-memory**: LRU cache, functools.lru_cache
- **Distributed**: Redis, Memcached
- **Invalidation**: TTL, explicit, pattern-based, tags
- **HTTP caching**: Cache-Control, ETag, Last-Modified
- **Warming**: Preload frequently accessed data

Cache wisely for performance! 🚀
