# ⚡ Caching Strategies in FastAPI

## Overview

Caching is one of the most effective performance optimization techniques. It reduces database load, speeds up response times, and improves scalability by storing frequently accessed data in memory.

---

## Types of Caching

### 1. **In-Memory Caching**

- Fastest access
- Limited to single instance
- Lost on restart

### 2. **Distributed Caching (Redis)**

- Shared across instances
- Persistent storage
- Slightly slower than in-memory

### 3. **HTTP Caching**

- Browser/CDN cache
- Cache-Control headers
- ETag validation

---

## In-Memory Caching

### 1. **Simple Dictionary Cache**

```python
from fastapi import FastAPI
from datetime import datetime, timedelta
from typing import Optional, Any
import time

app = FastAPI()

class SimpleCache:
    """Simple in-memory cache with TTL"""

    def __init__(self):
        self._cache: dict[str, tuple[Any, datetime]] = {}

    def get(self, key: str) -> Optional[Any]:
        """Get value from cache"""
        if key in self._cache:
            value, expiry = self._cache[key]
            if datetime.utcnow() < expiry:
                return value
            else:
                # Expired, remove from cache
                del self._cache[key]
        return None

    def set(self, key: str, value: Any, ttl_seconds: int = 300):
        """Set value in cache with TTL"""
        expiry = datetime.utcnow() + timedelta(seconds=ttl_seconds)
        self._cache[key] = (value, expiry)

    def delete(self, key: str):
        """Delete key from cache"""
        if key in self._cache:
            del self._cache[key]

    def clear(self):
        """Clear entire cache"""
        self._cache.clear()

# Global cache instance
cache = SimpleCache()

@app.get("/products/{product_id}")
async def get_product(product_id: int):
    """Get product with caching"""
    cache_key = f"product:{product_id}"

    # Try to get from cache
    cached_product = cache.get(cache_key)
    if cached_product:
        return {"product": cached_product, "source": "cache"}

    # Simulate database query
    time.sleep(0.5)  # Slow database query
    product = {
        "id": product_id,
        "name": f"Product {product_id}",
        "price": 99.99
    }

    # Store in cache (5 minutes)
    cache.set(cache_key, product, ttl_seconds=300)

    return {"product": product, "source": "database"}
```

### 2. **LRU Cache with functools**

```python
from functools import lru_cache
from typing import List

@lru_cache(maxsize=128)
def expensive_computation(n: int) -> int:
    """Cached expensive computation"""
    # Simulate expensive operation
    result = sum(i * i for i in range(n))
    return result

@app.get("/compute/{n}")
async def compute(n: int):
    """Endpoint using LRU cache"""
    result = expensive_computation(n)
    return {"result": result}

# Get cache statistics
@app.get("/cache/stats")
async def cache_stats():
    """Get LRU cache statistics"""
    info = expensive_computation.cache_info()
    return {
        "hits": info.hits,
        "misses": info.misses,
        "size": info.currsize,
        "maxsize": info.maxsize
    }

# Clear cache
@app.post("/cache/clear")
async def clear_cache():
    """Clear LRU cache"""
    expensive_computation.cache_clear()
    return {"message": "Cache cleared"}
```

### 3. **cachetools Library**

```python
from cachetools import TTLCache, cached
from cachetools.keys import hashkey
import asyncio

# TTL cache with max size
ttl_cache = TTLCache(maxsize=100, ttl=300)  # 5 minutes TTL

@cached(cache=ttl_cache)
def get_user_data(user_id: int) -> dict:
    """Cached user data retrieval"""
    # Simulate database query
    return {
        "id": user_id,
        "name": f"User {user_id}",
        "email": f"user{user_id}@example.com"
    }

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """Get user with caching"""
    user = get_user_data(user_id)
    return user

# Custom cache key
@cached(
    cache=ttl_cache,
    key=lambda user_id, include_posts: hashkey(user_id, include_posts)
)
def get_user_profile(user_id: int, include_posts: bool = False) -> dict:
    """Cached user profile with custom key"""
    profile = {
        "id": user_id,
        "name": f"User {user_id}"
    }

    if include_posts:
        profile["posts"] = [{"id": i, "title": f"Post {i}"} for i in range(5)]

    return profile
```

---

## Redis Caching

### 1. **Basic Redis Cache**

```python
from fastapi import FastAPI, Depends
import redis
import json
from typing import Optional

app = FastAPI()

# Redis connection
redis_client = redis.Redis(
    host="localhost",
    port=6379,
    db=0,
    decode_responses=True
)

def get_redis() -> redis.Redis:
    """Dependency for Redis client"""
    return redis_client

@app.get("/api/products/{product_id}")
async def get_product_redis(
    product_id: int,
    redis: redis.Redis = Depends(get_redis)
):
    """Get product with Redis caching"""
    cache_key = f"product:{product_id}"

    # Try to get from Redis
    cached = redis.get(cache_key)
    if cached:
        product = json.loads(cached)
        return {"product": product, "source": "cache"}

    # Fetch from database
    product = {
        "id": product_id,
        "name": f"Product {product_id}",
        "price": 99.99
    }

    # Store in Redis (5 minutes)
    redis.setex(
        cache_key,
        300,  # TTL in seconds
        json.dumps(product)
    )

    return {"product": product, "source": "database"}

@app.delete("/api/products/{product_id}")
async def delete_product(
    product_id: int,
    redis: redis.Redis = Depends(get_redis)
):
    """Delete product and invalidate cache"""
    # Delete from cache
    redis.delete(f"product:{product_id}")

    # Delete from database
    # ...

    return {"message": "Product deleted"}
```

### 2. **Redis Cache Decorator**

```python
from functools import wraps
import hashlib

def redis_cache(ttl: int = 300):
    """Decorator for Redis caching"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key from function name and arguments
            key_data = f"{func.__name__}:{str(args)}:{str(kwargs)}"
            cache_key = hashlib.md5(key_data.encode()).hexdigest()

            # Try to get from cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Execute function
            result = await func(*args, **kwargs)

            # Store in cache
            redis_client.setex(
                cache_key,
                ttl,
                json.dumps(result)
            )

            return result

        return wrapper
    return decorator

@app.get("/api/expensive-operation")
@redis_cache(ttl=600)  # Cache for 10 minutes
async def expensive_operation():
    """Expensive operation with caching"""
    # Simulate expensive computation
    await asyncio.sleep(2)

    return {
        "result": "computed value",
        "timestamp": datetime.utcnow().isoformat()
    }
```

### 3. **Redis Pipeline for Bulk Operations**

```python
@app.get("/api/products/batch")
async def get_products_batch(
    product_ids: List[int],
    redis: redis.Redis = Depends(get_redis)
):
    """Get multiple products efficiently"""
    # Build cache keys
    cache_keys = [f"product:{pid}" for pid in product_ids]

    # Use pipeline for bulk get
    pipe = redis.pipeline()
    for key in cache_keys:
        pipe.get(key)
    cached_results = pipe.execute()

    # Separate cached and missing
    products = []
    missing_ids = []

    for product_id, cached in zip(product_ids, cached_results):
        if cached:
            products.append(json.loads(cached))
        else:
            missing_ids.append(product_id)

    # Fetch missing from database
    if missing_ids:
        db_products = fetch_products_from_db(missing_ids)

        # Store in cache using pipeline
        pipe = redis.pipeline()
        for product in db_products:
            key = f"product:{product['id']}"
            pipe.setex(key, 300, json.dumps(product))
        pipe.execute()

        products.extend(db_products)

    return {"products": products}
```

---

## Cache Invalidation Strategies

### 1. **Time-Based Expiration (TTL)**

```python
class CacheManager:
    """Cache manager with multiple TTL strategies"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def cache_short(self, key: str, value: Any):
        """Short-lived cache (1 minute)"""
        self.redis.setex(key, 60, json.dumps(value))

    def cache_medium(self, key: str, value: Any):
        """Medium-lived cache (5 minutes)"""
        self.redis.setex(key, 300, json.dumps(value))

    def cache_long(self, key: str, value: Any):
        """Long-lived cache (1 hour)"""
        self.redis.setex(key, 3600, json.dumps(value))

    def cache_daily(self, key: str, value: Any):
        """Daily cache (24 hours)"""
        self.redis.setex(key, 86400, json.dumps(value))

cache_manager = CacheManager(redis_client)

@app.get("/api/trending-products")
async def get_trending():
    """Trending products with short cache"""
    cache_key = "trending:products"

    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # Fetch trending products
    products = fetch_trending_products()

    # Cache for 1 minute (frequently changing)
    cache_manager.cache_short(cache_key, products)

    return products
```

### 2. **Event-Based Invalidation**

```python
from fastapi import BackgroundTasks

class CacheInvalidator:
    """Handle cache invalidation on events"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def invalidate_pattern(self, pattern: str):
        """Invalidate all keys matching pattern"""
        keys = self.redis.keys(pattern)
        if keys:
            self.redis.delete(*keys)

    def invalidate_user_cache(self, user_id: int):
        """Invalidate all user-related cache"""
        patterns = [
            f"user:{user_id}:*",
            f"profile:{user_id}",
            f"posts:user:{user_id}"
        ]
        for pattern in patterns:
            self.invalidate_pattern(pattern)

    def invalidate_product_cache(self, product_id: int):
        """Invalidate product-related cache"""
        self.redis.delete(
            f"product:{product_id}",
            f"product:{product_id}:details",
            f"product:{product_id}:reviews"
        )

cache_invalidator = CacheInvalidator(redis_client)

@app.put("/api/users/{user_id}")
async def update_user(
    user_id: int,
    user_data: dict,
    background_tasks: BackgroundTasks
):
    """Update user and invalidate cache"""
    # Update user in database
    update_user_in_db(user_id, user_data)

    # Invalidate cache in background
    background_tasks.add_task(
        cache_invalidator.invalidate_user_cache,
        user_id
    )

    return {"message": "User updated"}
```

### 3. **Cache-Aside Pattern**

```python
class CacheAsideRepository:
    """Repository with cache-aside pattern"""

    def __init__(self, redis_client: redis.Redis, db: Session):
        self.redis = redis_client
        self.db = db

    def get_user(self, user_id: int) -> Optional[dict]:
        """Get user with cache-aside pattern"""
        cache_key = f"user:{user_id}"

        # 1. Try cache first
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 2. Cache miss - fetch from database
        user = self.db.query(User).filter(User.id == user_id).first()
        if not user:
            return None

        user_dict = {
            "id": user.id,
            "name": user.name,
            "email": user.email
        }

        # 3. Write to cache
        self.redis.setex(cache_key, 300, json.dumps(user_dict))

        return user_dict

    def update_user(self, user_id: int, data: dict) -> dict:
        """Update user with write-through caching"""
        # 1. Update database
        user = self.db.query(User).filter(User.id == user_id).first()
        for key, value in data.items():
            setattr(user, key, value)
        self.db.commit()

        # 2. Update cache immediately
        user_dict = {
            "id": user.id,
            "name": user.name,
            "email": user.email
        }
        cache_key = f"user:{user_id}"
        self.redis.setex(cache_key, 300, json.dumps(user_dict))

        return user_dict
```

---

## HTTP Caching

### 1. **Cache-Control Headers**

```python
from fastapi import Response
from fastapi.responses import JSONResponse

@app.get("/api/static-content")
async def get_static_content(response: Response):
    """Content with Cache-Control header"""
    # Set cache headers
    response.headers["Cache-Control"] = "public, max-age=3600"  # 1 hour
    response.headers["Vary"] = "Accept-Encoding"

    return {
        "content": "Static content that rarely changes",
        "version": "1.0"
    }

@app.get("/api/private-data")
async def get_private_data(response: Response):
    """Private data - no cache"""
    response.headers["Cache-Control"] = "private, no-cache, no-store, must-revalidate"
    response.headers["Pragma"] = "no-cache"
    response.headers["Expires"] = "0"

    return {"data": "Private user data"}

@app.get("/api/short-cache")
async def get_short_cache(response: Response):
    """Short-lived cache"""
    response.headers["Cache-Control"] = "public, max-age=60, s-maxage=300"
    # max-age=60: Browser caches for 1 minute
    # s-maxage=300: CDN caches for 5 minutes

    return {"data": "Frequently updated content"}
```

### 2. **ETag for Conditional Requests**

```python
import hashlib

@app.get("/api/resource/{resource_id}")
async def get_resource(resource_id: int, response: Response):
    """Resource with ETag support"""
    # Fetch resource
    resource = get_resource_from_db(resource_id)

    # Generate ETag from content
    content_str = json.dumps(resource, sort_keys=True)
    etag = hashlib.md5(content_str.encode()).hexdigest()

    # Set ETag header
    response.headers["ETag"] = f'"{etag}"'
    response.headers["Cache-Control"] = "max-age=0, must-revalidate"

    return resource

@app.get("/api/resource-conditional/{resource_id}")
async def get_resource_conditional(
    resource_id: int,
    if_none_match: Optional[str] = None,
    response: Response = None
):
    """Resource with ETag validation"""
    resource = get_resource_from_db(resource_id)

    # Generate ETag
    content_str = json.dumps(resource, sort_keys=True)
    etag = hashlib.md5(content_str.encode()).hexdigest()

    # Check if client has current version
    if if_none_match and if_none_match.strip('"') == etag:
        return Response(status_code=304)  # Not Modified

    # Resource changed, return new version
    response.headers["ETag"] = f'"{etag}"'
    return resource
```

---

## Advanced Caching Patterns

### 1. **Multi-Level Caching**

```python
class MultiLevelCache:
    """Multi-level caching (L1: Memory, L2: Redis)"""

    def __init__(self, redis_client: redis.Redis):
        self.l1_cache = SimpleCache()  # In-memory
        self.l2_cache = redis_client    # Redis

    def get(self, key: str) -> Optional[Any]:
        """Get from multi-level cache"""
        # Try L1 cache (fastest)
        value = self.l1_cache.get(key)
        if value:
            return value

        # Try L2 cache (Redis)
        cached = self.l2_cache.get(key)
        if cached:
            value = json.loads(cached)
            # Populate L1 cache
            self.l1_cache.set(key, value, ttl_seconds=60)
            return value

        return None

    def set(self, key: str, value: Any, ttl_seconds: int = 300):
        """Set in multi-level cache"""
        # Set in both caches
        self.l1_cache.set(key, value, ttl_seconds=min(ttl_seconds, 60))
        self.l2_cache.setex(key, ttl_seconds, json.dumps(value))

    def delete(self, key: str):
        """Delete from both caches"""
        self.l1_cache.delete(key)
        self.l2_cache.delete(key)

multi_cache = MultiLevelCache(redis_client)
```

### 2. **Cache Warming**

```python
from fastapi import FastAPI

app = FastAPI()

@app.on_event("startup")
async def warm_cache():
    """Warm cache on application startup"""
    print("Warming cache...")

    # Preload frequently accessed data
    popular_products = fetch_popular_products()
    for product in popular_products:
        cache_key = f"product:{product['id']}"
        redis_client.setex(cache_key, 3600, json.dumps(product))

    # Preload configuration
    config = fetch_app_config()
    redis_client.setex("app:config", 86400, json.dumps(config))

    print(f"Cache warmed with {len(popular_products)} products")
```

### 3. **Cache Stampede Prevention**

```python
import asyncio
from typing import Optional

class StampedeProtection:
    """Prevent cache stampede with locking"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.locks: dict[str, asyncio.Lock] = {}

    async def get_or_compute(
        self,
        key: str,
        compute_func,
        ttl: int = 300
    ) -> Any:
        """Get from cache or compute with stampede protection"""
        # Try cache first
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        # Acquire lock for this key
        if key not in self.locks:
            self.locks[key] = asyncio.Lock()

        async with self.locks[key]:
            # Double-check cache (another request may have filled it)
            cached = self.redis.get(key)
            if cached:
                return json.loads(cached)

            # Compute value
            value = await compute_func()

            # Store in cache
            self.redis.setex(key, ttl, json.dumps(value))

            return value

stampede_protection = StampedeProtection(redis_client)

@app.get("/api/heavy-computation")
async def heavy_computation():
    """Expensive operation with stampede protection"""
    async def compute():
        await asyncio.sleep(5)  # Simulate expensive operation
        return {"result": "computed value"}

    result = await stampede_protection.get_or_compute(
        "heavy:computation",
        compute,
        ttl=600
    )

    return result
```

---

## Best Practices

### ✅ Do's:

1. **Use appropriate TTL** for different data types
2. **Implement cache invalidation** on updates
3. **Use Redis** for distributed caching
4. **Monitor cache hit rates**
5. **Handle cache failures** gracefully
6. **Use compression** for large cached values
7. **Implement cache warming** for popular data

### ❌ Don'ts:

1. **Don't cache everything** blindly
2. **Don't use infinite TTL** without invalidation
3. **Don't ignore memory limits**
4. **Don't cache sensitive data** without encryption
5. **Don't forget to invalidate** on writes

---

## Interview Questions

### Q1: When should you use caching?

**Answer**: Cache when:

- Data is read frequently but written rarely
- Computation is expensive
- Database queries are slow
- Need to reduce load on backend systems
  Don't cache: rapidly changing data, user-specific data, sensitive information

### Q2: What is cache stampede and how to prevent it?

**Answer**: Cache stampede occurs when many requests simultaneously try to regenerate expired cache. Prevention:

- Use locks/semaphores
- Probabilistic early expiration
- Serve stale data while refreshing
- Use background refresh

### Q3: What's the difference between Cache-Aside and Write-Through?

**Answer**:

- **Cache-Aside**: Application manages cache, reads from cache then DB on miss
- **Write-Through**: Application writes to cache, cache writes to DB
  Cache-Aside is more common in web applications

### Q4: How do you handle distributed cache consistency?

**Answer**:

- Event-driven invalidation (pub/sub)
- Consistent hashing for key distribution
- Short TTLs for acceptable inconsistency
- Write-through caching for strong consistency

### Q5: What metrics should you monitor for caching?

**Answer**:

- Hit rate (hits / (hits + misses))
- Latency (cache vs database)
- Memory usage
- Eviction rate
- Network throughput (Redis)

---

## Summary

Effective caching strategies include:

- **In-memory caching** for single instances
- **Redis caching** for distributed systems
- **HTTP caching** for client-side optimization
- **Multi-level caching** for best performance
- **Proper invalidation** to maintain consistency

Caching can dramatically improve performance when used correctly! ⚡
