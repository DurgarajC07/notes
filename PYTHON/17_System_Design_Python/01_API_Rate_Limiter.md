# System Design: API Rate Limiter

## 📖 Problem Statement

Design a rate limiting system that:

- Limits requests per user/IP/API key
- Prevents abuse and DDoS attacks
- Supports multiple rate limit tiers (free, premium, enterprise)
- Provides clear feedback when limits exceeded
- Works in distributed systems (multiple servers)
- Has low latency overhead (<5ms)

## 🎯 Requirements

### Functional Requirements

1. Limit requests per time window (e.g., 100 requests/minute)
2. Support multiple limit types (per-user, per-IP, per-endpoint)
3. Return remaining quota in response headers
4. Support different tiers with different limits
5. Allow temporary limit increases (burst capacity)
6. Provide clear error messages when limit exceeded

### Non-Functional Requirements

1. **Low Latency**: <5ms overhead per request
2. **High Availability**: 99.99% uptime
3. **Scalability**: Handle millions of requests/second
4. **Accuracy**: 99%+ accuracy in limit enforcement
5. **Distributed**: Work across multiple servers
6. **Fault Tolerance**: Fail open (allow requests if rate limiter down)

## 📊 Capacity Estimation

### Traffic Estimates

- **API requests**: 10,000 requests/second
- **Rate limit checks**: 10,000 checks/second (1 per request)
- **Users**: 1 million active users
- **Cache memory**: ~100 MB for counters (1M users × 100 bytes)

## 🏗️ Rate Limiting Algorithms

### 1. Token Bucket Algorithm

```python
import time
from typing import Optional

class TokenBucket:
    """
    Token Bucket rate limiter

    - Tokens added at fixed rate (refill_rate)
    - Each request consumes 1 token
    - Max tokens stored = capacity
    - Allows burst traffic up to capacity
    """

    def __init__(self, capacity: int, refill_rate: float):
        """
        Args:
            capacity: Maximum tokens in bucket
            refill_rate: Tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_refill = time.time()

    def _refill(self):
        """Refill tokens based on time elapsed"""
        now = time.time()
        elapsed = now - self.last_refill

        # Calculate tokens to add
        tokens_to_add = elapsed * self.refill_rate

        # Add tokens, don't exceed capacity
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now

    def consume(self, tokens: int = 1) -> bool:
        """
        Try to consume tokens

        Returns:
            True if request allowed, False if rate limited
        """
        self._refill()

        if self.tokens >= tokens:
            self.tokens -= tokens
            return True

        return False

    def get_remaining(self) -> int:
        """Get remaining tokens"""
        self._refill()
        return int(self.tokens)

# Example usage
bucket = TokenBucket(capacity=100, refill_rate=10)  # 100 tokens, 10/second

# Process requests
for i in range(105):
    if bucket.consume():
        print(f"Request {i}: Allowed (remaining: {bucket.get_remaining()})")
    else:
        print(f"Request {i}: Rate limited")
    time.sleep(0.01)
```

### 2. Leaky Bucket Algorithm

```python
import time
from collections import deque
from typing import Optional

class LeakyBucket:
    """
    Leaky Bucket rate limiter

    - Requests added to queue (bucket)
    - Processed at fixed rate (leak_rate)
    - Queue has max size (capacity)
    - Smooths traffic to constant rate
    """

    def __init__(self, capacity: int, leak_rate: float):
        """
        Args:
            capacity: Maximum requests in queue
            leak_rate: Requests processed per second
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.queue = deque()
        self.last_leak = time.time()

    def _leak(self):
        """Process (leak) requests at fixed rate"""
        now = time.time()
        elapsed = now - self.last_leak

        # Calculate how many requests to process
        requests_to_leak = int(elapsed * self.leak_rate)

        # Remove processed requests from queue
        for _ in range(min(requests_to_leak, len(self.queue))):
            self.queue.popleft()

        if requests_to_leak > 0:
            self.last_leak = now

    def add_request(self) -> bool:
        """
        Add request to bucket

        Returns:
            True if accepted, False if bucket full
        """
        self._leak()

        if len(self.queue) < self.capacity:
            self.queue.append(time.time())
            return True

        return False

    def get_queue_size(self) -> int:
        """Get current queue size"""
        self._leak()
        return len(self.queue)

# Example usage
bucket = LeakyBucket(capacity=100, leak_rate=10)

for i in range(110):
    if bucket.add_request():
        print(f"Request {i}: Queued (queue size: {bucket.get_queue_size()})")
    else:
        print(f"Request {i}: Rejected - bucket full")
```

### 3. Fixed Window Counter

```python
import time
from collections import defaultdict

class FixedWindowCounter:
    """
    Fixed Window rate limiter

    - Time divided into fixed windows (e.g., 1 minute)
    - Count requests in current window
    - Reset counter at window boundary
    - Simple but has edge case issues
    """

    def __init__(self, limit: int, window_seconds: int):
        """
        Args:
            limit: Max requests per window
            window_seconds: Window size in seconds
        """
        self.limit = limit
        self.window_seconds = window_seconds
        self.counters = defaultdict(int)

    def _get_window_key(self, user_id: str) -> str:
        """Get current window key"""
        window_start = int(time.time() / self.window_seconds) * self.window_seconds
        return f"{user_id}:{window_start}"

    def is_allowed(self, user_id: str) -> bool:
        """
        Check if request allowed

        Returns:
            True if allowed, False if rate limited
        """
        key = self._get_window_key(user_id)

        if self.counters[key] < self.limit:
            self.counters[key] += 1
            return True

        return False

    def get_remaining(self, user_id: str) -> int:
        """Get remaining requests in window"""
        key = self._get_window_key(user_id)
        return max(0, self.limit - self.counters[key])

# Example usage
limiter = FixedWindowCounter(limit=10, window_seconds=60)

for i in range(15):
    user = "user123"
    if limiter.is_allowed(user):
        print(f"Request {i}: Allowed (remaining: {limiter.get_remaining(user)})")
    else:
        print(f"Request {i}: Rate limited")
```

### 4. Sliding Window Log

```python
import time
from collections import defaultdict, deque

class SlidingWindowLog:
    """
    Sliding Window Log rate limiter

    - Store timestamp of each request
    - Remove old timestamps outside window
    - More accurate than fixed window
    - Higher memory usage
    """

    def __init__(self, limit: int, window_seconds: int):
        """
        Args:
            limit: Max requests per window
            window_seconds: Window size in seconds
        """
        self.limit = limit
        self.window_seconds = window_seconds
        self.logs = defaultdict(deque)

    def _clean_old_logs(self, user_id: str):
        """Remove requests outside current window"""
        now = time.time()
        cutoff = now - self.window_seconds

        user_log = self.logs[user_id]

        # Remove old timestamps
        while user_log and user_log[0] < cutoff:
            user_log.popleft()

    def is_allowed(self, user_id: str) -> bool:
        """
        Check if request allowed

        Returns:
            True if allowed, False if rate limited
        """
        self._clean_old_logs(user_id)

        user_log = self.logs[user_id]

        if len(user_log) < self.limit:
            user_log.append(time.time())
            return True

        return False

    def get_remaining(self, user_id: str) -> int:
        """Get remaining requests in window"""
        self._clean_old_logs(user_id)
        return max(0, self.limit - len(self.logs[user_id]))

# Example usage
limiter = SlidingWindowLog(limit=10, window_seconds=60)

for i in range(15):
    user = "user123"
    if limiter.is_allowed(user):
        print(f"Request {i}: Allowed (remaining: {limiter.get_remaining(user)})")
    else:
        print(f"Request {i}: Rate limited")
```

## 🔍 Redis-Based Implementation

### 1. Redis Token Bucket

```python
import redis
import time
from typing import Optional

class RedisTokenBucket:
    """
    Distributed Token Bucket with Redis

    Supports multiple servers sharing rate limits
    """

    def __init__(self, redis_client: redis.Redis, capacity: int, refill_rate: float):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate

    def _get_keys(self, user_id: str):
        """Get Redis keys for user"""
        return f"rate_limit:{user_id}:tokens", f"rate_limit:{user_id}:last_refill"

    def consume(self, user_id: str, tokens: int = 1) -> tuple[bool, int]:
        """
        Try to consume tokens

        Returns:
            (allowed, remaining_tokens)
        """
        tokens_key, refill_key = self._get_keys(user_id)

        # Lua script for atomic operation
        lua_script = """
        local tokens_key = KEYS[1]
        local refill_key = KEYS[2]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local tokens_to_consume = tonumber(ARGV[3])
        local now = tonumber(ARGV[4])

        -- Get current values
        local tokens = tonumber(redis.call('GET', tokens_key) or capacity)
        local last_refill = tonumber(redis.call('GET', refill_key) or now)

        -- Calculate tokens to add
        local elapsed = now - last_refill
        local tokens_to_add = elapsed * refill_rate
        tokens = math.min(capacity, tokens + tokens_to_add)

        -- Try to consume
        local allowed = 0
        if tokens >= tokens_to_consume then
            tokens = tokens - tokens_to_consume
            allowed = 1
        end

        -- Update Redis
        redis.call('SET', tokens_key, tokens, 'EX', 3600)
        redis.call('SET', refill_key, now, 'EX', 3600)

        return {allowed, math.floor(tokens)}
        """

        now = time.time()
        result = self.redis.eval(
            lua_script,
            2,
            tokens_key,
            refill_key,
            self.capacity,
            self.refill_rate,
            tokens,
            now
        )

        allowed = bool(result[0])
        remaining = int(result[1])

        return allowed, remaining

# Example usage
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
limiter = RedisTokenBucket(redis_client, capacity=100, refill_rate=10)

allowed, remaining = limiter.consume("user123")
if allowed:
    print(f"Request allowed. Remaining: {remaining}")
else:
    print("Rate limited")
```

### 2. Redis Sliding Window

```python
import redis
import time

class RedisSlidingWindow:
    """
    Distributed Sliding Window with Redis Sorted Set

    - Store timestamps in sorted set
    - Score = timestamp
    - Remove old timestamps
    - Count items in window
    """

    def __init__(self, redis_client: redis.Redis, limit: int, window_seconds: int):
        self.redis = redis_client
        self.limit = limit
        self.window_seconds = window_seconds

    def is_allowed(self, user_id: str) -> tuple[bool, int]:
        """
        Check if request allowed

        Returns:
            (allowed, remaining_requests)
        """
        key = f"rate_limit:sliding:{user_id}"
        now = time.time()
        window_start = now - self.window_seconds

        # Lua script for atomic operation
        lua_script = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local window_start = tonumber(ARGV[2])
        local limit = tonumber(ARGV[3])

        -- Remove old timestamps
        redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

        -- Count requests in window
        local count = redis.call('ZCARD', key)

        local allowed = 0
        if count < limit then
            -- Add current request
            redis.call('ZADD', key, now, now)
            redis.call('EXPIRE', key, 3600)
            allowed = 1
            count = count + 1
        end

        return {allowed, limit - count}
        """

        result = self.redis.eval(
            lua_script,
            1,
            key,
            now,
            window_start,
            self.limit
        )

        allowed = bool(result[0])
        remaining = int(result[1])

        return allowed, remaining

# Example usage
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
limiter = RedisSlidingWindow(redis_client, limit=100, window_seconds=60)

allowed, remaining = limiter.is_allowed("user123")
print(f"Allowed: {allowed}, Remaining: {remaining}")
```

## 🎯 Django Implementation

```python
# middleware.py
from django.http import JsonResponse
from django.core.cache import cache
import time

class RateLimitMiddleware:
    """
    Django middleware for rate limiting
    """

    def __init__(self, get_response):
        self.get_response = get_response
        self.limits = {
            'default': (100, 60),  # 100 requests per 60 seconds
            'premium': (1000, 60),  # 1000 requests per 60 seconds
        }

    def __call__(self, request):
        # Get user tier
        tier = self._get_user_tier(request)
        limit, window = self.limits.get(tier, self.limits['default'])

        # Get user identifier
        user_id = self._get_user_id(request)

        # Check rate limit
        allowed, remaining, reset_time = self._check_rate_limit(
            user_id, limit, window
        )

        if not allowed:
            return JsonResponse({
                'error': 'Rate limit exceeded',
                'retry_after': int(reset_time - time.time())
            }, status=429)

        # Process request
        response = self.get_response(request)

        # Add rate limit headers
        response['X-RateLimit-Limit'] = str(limit)
        response['X-RateLimit-Remaining'] = str(remaining)
        response['X-RateLimit-Reset'] = str(int(reset_time))

        return response

    def _get_user_id(self, request):
        """Get user identifier (user ID or IP)"""
        if request.user.is_authenticated:
            return f"user:{request.user.id}"
        return f"ip:{self._get_client_ip(request)}"

    def _get_client_ip(self, request):
        """Get client IP address"""
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            return x_forwarded_for.split(',')[0].strip()
        return request.META.get('REMOTE_ADDR')

    def _get_user_tier(self, request):
        """Get user subscription tier"""
        if not request.user.is_authenticated:
            return 'default'

        # Check user's subscription tier
        if hasattr(request.user, 'subscription'):
            return request.user.subscription.tier

        return 'default'

    def _check_rate_limit(self, user_id: str, limit: int, window: int):
        """
        Check rate limit using Redis sliding window

        Returns:
            (allowed, remaining, reset_time)
        """
        now = time.time()
        window_start = now - window
        reset_time = now + window

        key = f"rate_limit:{user_id}"

        # Get timestamps from cache
        timestamps = cache.get(key, [])

        # Remove old timestamps
        timestamps = [ts for ts in timestamps if ts > window_start]

        # Check limit
        if len(timestamps) < limit:
            timestamps.append(now)
            cache.set(key, timestamps, window)
            return True, limit - len(timestamps), reset_time

        return False, 0, reset_time

# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'myapp.middleware.RateLimitMiddleware',  # Add rate limiting
    ...
]
```

## 🚀 FastAPI Implementation

```python
# rate_limiter.py
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
import redis
import time

app = FastAPI()

# Redis client
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

class RateLimitMiddleware(BaseHTTPMiddleware):
    """
    FastAPI middleware for rate limiting
    """

    async def dispatch(self, request: Request, call_next):
        # Get user identifier
        user_id = self._get_user_id(request)

        # Rate limit configuration
        limit = 100
        window = 60

        # Check rate limit
        allowed, remaining, reset_time = await self._check_rate_limit(
            user_id, limit, window
        )

        if not allowed:
            return JSONResponse(
                status_code=429,
                content={
                    'error': 'Rate limit exceeded',
                    'retry_after': int(reset_time - time.time())
                },
                headers={
                    'X-RateLimit-Limit': str(limit),
                    'X-RateLimit-Remaining': '0',
                    'X-RateLimit-Reset': str(int(reset_time)),
                    'Retry-After': str(int(reset_time - time.time()))
                }
            )

        # Process request
        response = await call_next(request)

        # Add rate limit headers
        response.headers['X-RateLimit-Limit'] = str(limit)
        response.headers['X-RateLimit-Remaining'] = str(remaining)
        response.headers['X-RateLimit-Reset'] = str(int(reset_time))

        return response

    def _get_user_id(self, request: Request) -> str:
        """Get user identifier"""
        # Try to get user from JWT or session
        # For now, use IP address
        if request.client:
            return f"ip:{request.client.host}"
        return "unknown"

    async def _check_rate_limit(self, user_id: str, limit: int, window: int):
        """
        Check rate limit using Redis

        Returns:
            (allowed, remaining, reset_time)
        """
        key = f"rate_limit:sliding:{user_id}"
        now = time.time()
        window_start = now - window
        reset_time = now + window

        # Lua script for atomic operation
        lua_script = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local window_start = tonumber(ARGV[2])
        local limit = tonumber(ARGV[3])

        redis.call('ZREMRANGEBYSCORE', key, 0, window_start)
        local count = redis.call('ZCARD', key)

        local allowed = 0
        if count < limit then
            redis.call('ZADD', key, now, now)
            redis.call('EXPIRE', key, 3600)
            allowed = 1
            count = count + 1
        end

        return {allowed, limit - count}
        """

        result = redis_client.eval(
            lua_script,
            1,
            key,
            now,
            window_start,
            limit
        )

        allowed = bool(result[0])
        remaining = int(result[1])

        return allowed, remaining, reset_time

# Add middleware
app.add_middleware(RateLimitMiddleware)

@app.get("/api/data")
async def get_data():
    return {"message": "Data retrieved successfully"}
```

## ❓ Interview Questions

### Q1: Compare Token Bucket vs Leaky Bucket algorithms

**Answer**:

- **Token Bucket**: Allows burst traffic up to capacity. Good for APIs with occasional spikes.
- **Leaky Bucket**: Smooths traffic to constant rate. Good for backends that can't handle bursts.
- **When to use**: Token bucket for most APIs, leaky bucket for rate-sensitive backends.

### Q2: How do you implement rate limiting in a distributed system?

**Answer**:

1. **Centralized Redis**: Single source of truth for counters
2. **Lua scripts**: Atomic operations (no race conditions)
3. **Sharding**: Partition keys across Redis cluster
4. **Replication**: Redis replicas for high availability
5. **Fail open**: If Redis down, allow requests (don't block legitimate traffic)

### Q3: What algorithm is most accurate?

**Answer**:

- **Sliding Window Log**: Most accurate, tracks each request timestamp
- **Fixed Window**: Least accurate, allows 2x traffic at window boundaries
- **Token Bucket**: Good balance of accuracy and efficiency
- **Trade-off**: Accuracy vs memory usage

## 📚 Summary

**Key Takeaways**:

1. **Token Bucket**: Best for most use cases (allows bursts, efficient)
2. **Sliding Window**: Most accurate (higher memory cost)
3. **Redis + Lua**: For distributed systems (atomic operations)
4. **Fail Open**: Don't block traffic if rate limiter fails
5. **Response Headers**: Always include X-RateLimit-\* headers
6. **Multiple Tiers**: Support different limits per tier
7. **User + IP**: Rate limit by both user and IP
8. **Clear Errors**: Provide retry_after in 429 responses

Rate limiting is essential for API security and stability!
