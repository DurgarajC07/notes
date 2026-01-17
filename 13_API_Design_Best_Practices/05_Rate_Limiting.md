# ⚡ Rate Limiting and Throttling

## Overview

Rate limiting protects APIs from abuse, prevents DDoS attacks, ensures fair resource allocation, and maintains service quality. Implementing effective rate limiting is crucial for production APIs.

---

## Rate Limiting Strategies

### 1. **Fixed Window**

Simple time-based windows.

```python
from datetime import datetime, timedelta
from collections import defaultdict

class FixedWindowRateLimiter:
    """Fixed window rate limiter"""

    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(list)

    def is_allowed(self, client_id: str) -> bool:
        """Check if request is allowed"""
        now = datetime.utcnow()
        window_start = now - timedelta(seconds=self.window_seconds)

        # Remove old requests
        self.requests[client_id] = [
            req_time for req_time in self.requests[client_id]
            if req_time > window_start
        ]

        # Check limit
        if len(self.requests[client_id]) < self.max_requests:
            self.requests[client_id].append(now)
            return True

        return False

# Usage
limiter = FixedWindowRateLimiter(max_requests=100, window_seconds=60)

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    """Apply rate limiting"""
    client_id = request.client.host

    if not limiter.is_allowed(client_id):
        return JSONResponse(
            status_code=429,
            content={"error": "Rate limit exceeded"},
            headers={"Retry-After": "60"}
        )

    return await call_next(request)
```

### 2. **Sliding Window**

More accurate than fixed window.

```python
import time

class SlidingWindowRateLimiter:
    """Sliding window rate limiter"""

    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(list)

    def is_allowed(self, client_id: str) -> tuple[bool, dict]:
        """Check if request is allowed, return status and metadata"""
        now = time.time()
        window_start = now - self.window_seconds

        # Remove expired requests
        self.requests[client_id] = [
            req_time for req_time in self.requests[client_id]
            if req_time > window_start
        ]

        current_requests = len(self.requests[client_id])
        remaining = self.max_requests - current_requests

        if current_requests < self.max_requests:
            self.requests[client_id].append(now)
            return True, {
                "limit": self.max_requests,
                "remaining": remaining - 1,
                "reset": int(now + self.window_seconds)
            }

        # Calculate retry after
        oldest_request = min(self.requests[client_id])
        retry_after = int(oldest_request + self.window_seconds - now)

        return False, {
            "limit": self.max_requests,
            "remaining": 0,
            "reset": int(oldest_request + self.window_seconds),
            "retry_after": retry_after
        }

limiter = SlidingWindowRateLimiter(max_requests=100, window_seconds=60)

@app.middleware("http")
async def sliding_rate_limit(request: Request, call_next):
    """Sliding window rate limiting with headers"""
    client_id = request.client.host

    allowed, metadata = limiter.is_allowed(client_id)

    if not allowed:
        return JSONResponse(
            status_code=429,
            content={
                "error": "Rate limit exceeded",
                "retry_after": metadata["retry_after"]
            },
            headers={
                "X-RateLimit-Limit": str(metadata["limit"]),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(metadata["reset"]),
                "Retry-After": str(metadata["retry_after"])
            }
        )

    response = await call_next(request)

    # Add rate limit headers to response
    response.headers["X-RateLimit-Limit"] = str(metadata["limit"])
    response.headers["X-RateLimit-Remaining"] = str(metadata["remaining"])
    response.headers["X-RateLimit-Reset"] = str(metadata["reset"])

    return response
```

### 3. **Token Bucket**

Allows bursts while maintaining average rate.

```python
import time
from threading import Lock

class TokenBucketRateLimiter:
    """Token bucket rate limiter"""

    def __init__(self, capacity: int, refill_rate: float):
        """
        Args:
            capacity: Maximum tokens in bucket
            refill_rate: Tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.buckets = defaultdict(lambda: {
            "tokens": capacity,
            "last_refill": time.time()
        })
        self.lock = Lock()

    def is_allowed(self, client_id: str) -> bool:
        """Check if request is allowed"""
        with self.lock:
            now = time.time()
            bucket = self.buckets[client_id]

            # Refill tokens
            time_passed = now - bucket["last_refill"]
            tokens_to_add = time_passed * self.refill_rate
            bucket["tokens"] = min(
                self.capacity,
                bucket["tokens"] + tokens_to_add
            )
            bucket["last_refill"] = now

            # Check if token available
            if bucket["tokens"] >= 1:
                bucket["tokens"] -= 1
                return True

            return False

# Allows 100 requests per minute with bursts up to 200
limiter = TokenBucketRateLimiter(
    capacity=200,  # Burst capacity
    refill_rate=100/60  # 100 tokens per 60 seconds
)
```

### 4. **Leaky Bucket**

Smooths out bursty traffic.

```python
from queue import Queue
import asyncio

class LeakyBucketRateLimiter:
    """Leaky bucket rate limiter"""

    def __init__(self, capacity: int, leak_rate: float):
        """
        Args:
            capacity: Queue capacity
            leak_rate: Requests processed per second
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.buckets = defaultdict(lambda: Queue(maxsize=capacity))

    async def is_allowed(self, client_id: str) -> bool:
        """Check if request can be queued"""
        bucket = self.buckets[client_id]

        # Try to add to queue
        try:
            bucket.put_nowait(time.time())
            return True
        except:
            return False

    async def process_bucket(self, client_id: str):
        """Process requests from bucket at fixed rate"""
        bucket = self.buckets[client_id]

        while True:
            if not bucket.empty():
                bucket.get()
                await asyncio.sleep(1 / self.leak_rate)
            else:
                await asyncio.sleep(0.1)
```

---

## Redis-Based Rate Limiting

### 1. **Redis Fixed Window**

```python
import redis
from fastapi import Request, HTTPException

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

class RedisRateLimiter:
    """Redis-based rate limiter"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def is_allowed(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> tuple[bool, dict]:
        """Check if request is allowed"""
        current = self.redis.get(key)

        if current is None:
            # First request in window
            pipe = self.redis.pipeline()
            pipe.set(key, 1, ex=window_seconds)
            pipe.execute()

            return True, {
                "limit": max_requests,
                "remaining": max_requests - 1,
                "reset": int(time.time() + window_seconds)
            }

        current = int(current)

        if current < max_requests:
            # Increment counter
            new_count = self.redis.incr(key)
            ttl = self.redis.ttl(key)

            return True, {
                "limit": max_requests,
                "remaining": max_requests - new_count,
                "reset": int(time.time() + ttl)
            }

        # Limit exceeded
        ttl = self.redis.ttl(key)
        return False, {
            "limit": max_requests,
            "remaining": 0,
            "reset": int(time.time() + ttl),
            "retry_after": ttl
        }

limiter = RedisRateLimiter(redis_client)

def get_rate_limit_key(request: Request) -> str:
    """Generate rate limit key from request"""
    # Can use IP, user ID, API key, etc.
    return f"ratelimit:{request.client.host}"

@app.middleware("http")
async def redis_rate_limit(request: Request, call_next):
    """Redis-based rate limiting"""
    key = get_rate_limit_key(request)

    allowed, metadata = limiter.is_allowed(
        key,
        max_requests=100,
        window_seconds=60
    )

    if not allowed:
        return JSONResponse(
            status_code=429,
            content={"error": "Rate limit exceeded"},
            headers={
                "X-RateLimit-Limit": str(metadata["limit"]),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(metadata["reset"]),
                "Retry-After": str(metadata.get("retry_after", 60))
            }
        )

    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = str(metadata["limit"])
    response.headers["X-RateLimit-Remaining"] = str(metadata["remaining"])
    response.headers["X-RateLimit-Reset"] = str(metadata["reset"])

    return response
```

### 2. **Redis Sliding Window with Sorted Sets**

```python
class RedisSlidingWindowLimiter:
    """Redis sliding window using sorted sets"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def is_allowed(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> bool:
        """Check using sorted set timestamps"""
        now = time.time()
        window_start = now - window_seconds

        # Use pipeline for atomicity
        pipe = self.redis.pipeline()

        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)

        # Count requests in window
        pipe.zcard(key)

        # Add current request
        pipe.zadd(key, {str(now): now})

        # Set expiry
        pipe.expire(key, window_seconds)

        results = pipe.execute()
        request_count = results[1]

        return request_count < max_requests
```

---

## Tiered Rate Limiting

### 1. **User Tier-Based Limits**

```python
from enum import Enum

class UserTier(str, Enum):
    FREE = "free"
    BASIC = "basic"
    PREMIUM = "premium"
    ENTERPRISE = "enterprise"

TIER_LIMITS = {
    UserTier.FREE: {"requests": 100, "window": 3600},      # 100/hour
    UserTier.BASIC: {"requests": 1000, "window": 3600},    # 1000/hour
    UserTier.PREMIUM: {"requests": 10000, "window": 3600}, # 10k/hour
    UserTier.ENTERPRISE: {"requests": 100000, "window": 3600}  # 100k/hour
}

def get_user_tier(user_id: int) -> UserTier:
    """Get user's subscription tier"""
    # Query database for user tier
    return UserTier.PREMIUM

@app.middleware("http")
async def tiered_rate_limit(request: Request, call_next):
    """Rate limiting based on user tier"""
    # Get user from auth token
    user_id = extract_user_id(request)

    if not user_id:
        # Anonymous users get free tier
        tier = UserTier.FREE
    else:
        tier = get_user_tier(user_id)

    limits = TIER_LIMITS[tier]
    key = f"ratelimit:{tier}:{user_id or request.client.host}"

    allowed, metadata = limiter.is_allowed(
        key,
        max_requests=limits["requests"],
        window_seconds=limits["window"]
    )

    if not allowed:
        return JSONResponse(
            status_code=429,
            content={
                "error": "Rate limit exceeded",
                "tier": tier,
                "upgrade_url": "/upgrade" if tier != UserTier.ENTERPRISE else None
            },
            headers={
                "X-RateLimit-Limit": str(metadata["limit"]),
                "Retry-After": str(metadata.get("retry_after", 60))
            }
        )

    response = await call_next(request)
    response.headers["X-RateLimit-Tier"] = tier
    response.headers["X-RateLimit-Limit"] = str(metadata["limit"])
    response.headers["X-RateLimit-Remaining"] = str(metadata["remaining"])

    return response
```

### 2. **Endpoint-Specific Limits**

```python
ENDPOINT_LIMITS = {
    "/api/search": {"requests": 10, "window": 60},      # 10/minute
    "/api/upload": {"requests": 5, "window": 3600},     # 5/hour
    "/api/export": {"requests": 2, "window": 86400},    # 2/day
    "default": {"requests": 100, "window": 60}          # 100/minute
}

@app.middleware("http")
async def endpoint_rate_limit(request: Request, call_next):
    """Different limits per endpoint"""
    path = request.url.path

    # Get endpoint-specific limits
    limits = ENDPOINT_LIMITS.get(path, ENDPOINT_LIMITS["default"])

    key = f"ratelimit:{path}:{request.client.host}"

    allowed, metadata = limiter.is_allowed(
        key,
        max_requests=limits["requests"],
        window_seconds=limits["window"]
    )

    if not allowed:
        return JSONResponse(
            status_code=429,
            content={
                "error": f"Rate limit exceeded for {path}",
                "limit": metadata["limit"],
                "retry_after": metadata.get("retry_after")
            }
        )

    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = str(metadata["limit"])
    response.headers["X-RateLimit-Remaining"] = str(metadata["remaining"])

    return response
```

---

## Advanced Patterns

### 1. **Concurrent Request Limiting**

```python
from asyncio import Semaphore

class ConcurrencyLimiter:
    """Limit concurrent requests per user"""

    def __init__(self, max_concurrent: int):
        self.max_concurrent = max_concurrent
        self.semaphores = defaultdict(lambda: Semaphore(max_concurrent))

    async def acquire(self, client_id: str) -> bool:
        """Try to acquire semaphore"""
        sem = self.semaphores[client_id]
        return await sem.acquire()

    def release(self, client_id: str):
        """Release semaphore"""
        sem = self.semaphores[client_id]
        sem.release()

concurrency_limiter = ConcurrencyLimiter(max_concurrent=10)

@app.middleware("http")
async def concurrency_limit(request: Request, call_next):
    """Limit concurrent requests"""
    client_id = request.client.host

    # Try to acquire
    if not await concurrency_limiter.acquire(client_id):
        return JSONResponse(
            status_code=429,
            content={"error": "Too many concurrent requests"}
        )

    try:
        response = await call_next(request)
        return response
    finally:
        concurrency_limiter.release(client_id)
```

### 2. **Cost-Based Rate Limiting**

```python
ENDPOINT_COSTS = {
    "/api/simple": 1,
    "/api/search": 5,
    "/api/export": 50,
    "/api/ml-inference": 100
}

class CostBasedRateLimiter:
    """Rate limiting based on operation cost"""

    def __init__(self, max_cost: int, window_seconds: int):
        self.max_cost = max_cost
        self.window_seconds = window_seconds
        self.costs = defaultdict(list)

    def is_allowed(self, client_id: str, cost: int) -> bool:
        """Check if cost is within limit"""
        now = time.time()
        window_start = now - self.window_seconds

        # Remove old costs
        self.costs[client_id] = [
            (timestamp, c) for timestamp, c in self.costs[client_id]
            if timestamp > window_start
        ]

        # Calculate current total
        current_cost = sum(c for _, c in self.costs[client_id])

        if current_cost + cost <= self.max_cost:
            self.costs[client_id].append((now, cost))
            return True

        return False

cost_limiter = CostBasedRateLimiter(max_cost=1000, window_seconds=60)

@app.middleware("http")
async def cost_based_limit(request: Request, call_next):
    """Cost-based rate limiting"""
    path = request.url.path
    cost = ENDPOINT_COSTS.get(path, 1)
    client_id = request.client.host

    if not cost_limiter.is_allowed(client_id, cost):
        return JSONResponse(
            status_code=429,
            content={
                "error": "Cost limit exceeded",
                "cost": cost,
                "max_cost": cost_limiter.max_cost
            }
        )

    response = await call_next(request)
    response.headers["X-Request-Cost"] = str(cost)

    return response
```

---

## slowapi Integration

Popular library for rate limiting in FastAPI.

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/api/data")
@limiter.limit("5/minute")
async def limited_endpoint(request: Request):
    """Endpoint with 5 requests per minute limit"""
    return {"data": "response"}

@app.get("/api/search")
@limiter.limit("10/minute;100/hour;1000/day")
async def multiple_limits(request: Request):
    """Multiple time windows"""
    return {"results": []}

# Custom key function
def get_api_key(request: Request) -> str:
    """Use API key instead of IP"""
    return request.headers.get("X-API-Key", get_remote_address(request))

@app.get("/api/premium")
@limiter.limit("1000/hour", key_func=get_api_key)
async def premium_endpoint(request: Request):
    """Rate limit by API key"""
    return {"data": "premium"}
```

---

## Best Practices

### ✅ Do's:

1. **Return 429 status code** for rate limit exceeded
2. **Include Retry-After header**
3. **Add rate limit headers** (X-RateLimit-\*)
4. **Use distributed cache** (Redis) for multi-instance setups
5. **Implement tiered limits** for different users
6. **Log rate limit violations**
7. **Provide clear error messages**
8. **Allow authenticated users** higher limits

### ❌ Don'ts:

1. **Don't use in-memory** for distributed systems
2. **Don't ignore burst traffic**
3. **Don't rate limit health checks**
4. **Don't forget to clean up** old entries
5. **Don't apply same limit** to all endpoints
6. **Don't block legitimate users**

---

## Interview Questions

### Q1: What's the difference between rate limiting and throttling?

**Answer**:

- **Rate limiting**: Hard limit, reject requests after threshold (429)
- **Throttling**: Slow down requests, queue or delay
  Rate limiting is more common for APIs, throttling for background tasks.

### Q2: Explain fixed window vs sliding window rate limiting.

**Answer**:

- **Fixed window**: Resets at fixed intervals. Can allow 2x limit at boundary.
- **Sliding window**: Continuous tracking. More accurate but slightly more complex.
  Sliding window prevents boundary exploitation.

### Q3: How do you implement rate limiting in distributed systems?

**Answer**: Use centralized storage:

- Redis with atomic operations
- Sorted sets for sliding window
- TTL for automatic cleanup
- Lua scripts for atomicity
  Avoid in-memory as it's per-instance.

### Q4: What is token bucket algorithm?

**Answer**: Bucket holds tokens, refilled at constant rate:

- Each request consumes a token
- Allows bursts up to bucket capacity
- Refills maintain average rate
  Good for APIs with bursty traffic.

### Q5: How do you handle rate limits for different user tiers?

**Answer**:

- Define limits per tier (free, basic, premium)
- Use user ID in rate limit key
- Store tier in cache for fast lookup
- Return upgrade prompts to free users
- Monitor usage for billing

---

## Summary

Rate limiting strategies:

- **Fixed/Sliding window** for simple limits
- **Token bucket** for burst tolerance
- **Redis-based** for distributed systems
- **Tiered limits** for different users
- **Cost-based** for expensive operations
- **Standard headers** for communication

Proper rate limiting protects your API and ensures fair usage! ⚡
