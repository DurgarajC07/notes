# 🏗️ System Design: Rate Limiter

## Overview

Design a rate limiting system to control API request rates, prevent abuse, and ensure fair resource usage across clients.

---

## Requirements

### Functional Requirements

1. **Limit requests** based on various rules (per user, IP, API key)
2. **Multiple algorithms** (token bucket, leaky bucket, fixed/sliding window)
3. **Distributed support** for multiple servers
4. **Configurable rules** per endpoint/user tier
5. **Return rate limit** info in response headers

### Non-Functional Requirements

1. **Low latency**: <10ms overhead per request
2. **High availability**: 99.9% uptime
3. **Scalable**: Handle millions of requests/second
4. **Accurate**: Minimal false positives/negatives
5. **Fault tolerant**: Degrade gracefully

---

## Capacity Estimation

### Traffic Estimates

- **Total users**: 10M
- **Requests per user**: 100/day
- **Total requests**: 1B/day = ~12K requests/second
- **Peak traffic**: 3x = 36K requests/second

### Storage Estimates

- **Data per user**: user_id (8 bytes) + count (4 bytes) + timestamp (8 bytes) = 20 bytes
- **Active users** in window: 1M concurrent
- **Total storage**: 1M × 20 bytes = 20 MB (fits in memory)

---

## Rate Limiting Algorithms

### 1. Token Bucket Algorithm

```python
import time
import threading
from dataclasses import dataclass
from typing import Dict

@dataclass
class TokenBucket:
    """
    Token Bucket Algorithm

    Concepts:
    - Bucket holds tokens up to capacity
    - Tokens added at fixed rate
    - Request consumes token(s)
    - Request rejected if insufficient tokens

    Pros:
    - Allows burst traffic up to capacity
    - Memory efficient

    Cons:
    - Two parameters to tune
    """
    capacity: int          # Maximum tokens
    refill_rate: float     # Tokens per second
    tokens: float = None   # Current tokens
    last_refill: float = None

    def __post_init__(self):
        if self.tokens is None:
            self.tokens = self.capacity
        if self.last_refill is None:
            self.last_refill = time.time()

    def _refill(self):
        """Add tokens based on time elapsed"""
        now = time.time()
        elapsed = now - self.last_refill

        # Add tokens based on elapsed time
        tokens_to_add = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now

    def consume(self, tokens: int = 1) -> bool:
        """Try to consume tokens"""
        self._refill()

        if self.tokens >= tokens:
            self.tokens -= tokens
            return True

        return False

    def get_wait_time(self, tokens: int = 1) -> float:
        """Get time to wait for tokens"""
        self._refill()

        if self.tokens >= tokens:
            return 0

        tokens_needed = tokens - self.tokens
        return tokens_needed / self.refill_rate

# Example usage
bucket = TokenBucket(capacity=10, refill_rate=1.0)  # 10 tokens, 1 per second

for i in range(15):
    if bucket.consume():
        print(f"Request {i+1}: Allowed")
    else:
        wait_time = bucket.get_wait_time()
        print(f"Request {i+1}: Denied (wait {wait_time:.2f}s)")

    time.sleep(0.5)  # Request every 0.5 seconds
```

### 2. Leaky Bucket Algorithm

```python
from collections import deque
import time

class LeakyBucket:
    """
    Leaky Bucket Algorithm

    Concepts:
    - Requests enter bucket as queue
    - Processed at fixed rate (leak rate)
    - Bucket has maximum capacity
    - Overflow requests rejected

    Pros:
    - Smooth output rate
    - Simple implementation

    Cons:
    - No burst allowance
    - Queue requires memory
    """

    def __init__(self, capacity: int, leak_rate: float):
        self.capacity = capacity
        self.leak_rate = leak_rate  # requests per second
        self.queue = deque()
        self.last_leak = time.time()
        self.lock = threading.Lock()

    def _leak(self):
        """Remove processed requests from queue"""
        now = time.time()
        elapsed = now - self.last_leak

        # Number of requests that leaked out
        leaked = int(elapsed * self.leak_rate)

        if leaked > 0:
            for _ in range(min(leaked, len(self.queue))):
                self.queue.popleft()

            self.last_leak = now

    def allow_request(self) -> bool:
        """Check if request can be accepted"""
        with self.lock:
            self._leak()

            if len(self.queue) < self.capacity:
                self.queue.append(time.time())
                return True

            return False

# Example
bucket = LeakyBucket(capacity=5, leak_rate=2.0)  # 5 capacity, 2/sec leak

for i in range(10):
    if bucket.allow_request():
        print(f"Request {i+1}: Allowed (queue size: {len(bucket.queue)})")
    else:
        print(f"Request {i+1}: Denied (queue full)")

    time.sleep(0.3)
```

### 3. Fixed Window Counter

```python
import time
from collections import defaultdict

class FixedWindowCounter:
    """
    Fixed Window Counter Algorithm

    Concepts:
    - Time divided into fixed windows
    - Count requests in current window
    - Reset counter at window boundary

    Pros:
    - Very simple
    - Memory efficient

    Cons:
    - Burst at window boundaries
    - Twice limit possible at boundary
    """

    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.counters: Dict[int, int] = defaultdict(int)
        self.lock = threading.Lock()

    def allow_request(self, user_id: str) -> bool:
        """Check if request allowed"""
        with self.lock:
            now = time.time()
            window = int(now // self.window_seconds)

            key = f"{user_id}:{window}"

            if self.counters[key] < self.limit:
                self.counters[key] += 1
                return True

            return False

    def get_remaining(self, user_id: str) -> int:
        """Get remaining requests in window"""
        now = time.time()
        window = int(now // self.window_seconds)
        key = f"{user_id}:{window}"

        return max(0, self.limit - self.counters.get(key, 0))

# Example
limiter = FixedWindowCounter(limit=5, window_seconds=10)

for i in range(8):
    user_id = "user123"

    if limiter.allow_request(user_id):
        remaining = limiter.get_remaining(user_id)
        print(f"Request {i+1}: Allowed ({remaining} remaining)")
    else:
        print(f"Request {i+1}: Denied")

    time.sleep(1)
```

### 4. Sliding Window Log

```python
from collections import deque
import time

class SlidingWindowLog:
    """
    Sliding Window Log Algorithm

    Concepts:
    - Store timestamp of each request
    - Remove old requests outside window
    - Count requests in current window

    Pros:
    - No boundary issues
    - Accurate

    Cons:
    - Memory intensive
    - O(n) lookup time
    """

    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.logs: Dict[str, deque] = defaultdict(deque)
        self.lock = threading.Lock()

    def allow_request(self, user_id: str) -> bool:
        """Check if request allowed"""
        with self.lock:
            now = time.time()
            window_start = now - self.window_seconds

            # Remove old requests
            while self.logs[user_id] and self.logs[user_id][0] <= window_start:
                self.logs[user_id].popleft()

            # Check limit
            if len(self.logs[user_id]) < self.limit:
                self.logs[user_id].append(now)
                return True

            return False

    def get_reset_time(self, user_id: str) -> float:
        """Get time until next available slot"""
        if not self.logs[user_id]:
            return 0

        oldest = self.logs[user_id][0]
        return max(0, oldest + self.window_seconds - time.time())

# Example
limiter = SlidingWindowLog(limit=3, window_seconds=5)

for i in range(6):
    user_id = "user123"

    if limiter.allow_request(user_id):
        print(f"Request {i+1}: Allowed")
    else:
        reset_time = limiter.get_reset_time(user_id)
        print(f"Request {i+1}: Denied (reset in {reset_time:.1f}s)")

    time.sleep(1)
```

### 5. Sliding Window Counter (Hybrid)

```python
class SlidingWindowCounter:
    """
    Sliding Window Counter Algorithm

    Concepts:
    - Combine current and previous window
    - Weight previous window by overlap
    - More accurate than fixed window

    Formula:
    count = current_count + (previous_count * overlap_percentage)

    Pros:
    - Memory efficient
    - Better than fixed window

    Cons:
    - Approximation
    """

    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.counters: Dict[str, Dict] = defaultdict(lambda: {
            'current_window': 0,
            'current_count': 0,
            'previous_count': 0
        })
        self.lock = threading.Lock()

    def allow_request(self, user_id: str) -> bool:
        """Check if request allowed"""
        with self.lock:
            now = time.time()
            current_window = int(now // self.window_seconds)

            data = self.counters[user_id]

            # New window - rotate
            if data['current_window'] != current_window:
                data['previous_count'] = data['current_count']
                data['current_count'] = 0
                data['current_window'] = current_window

            # Calculate weighted count
            window_position = now % self.window_seconds
            previous_weight = 1 - (window_position / self.window_seconds)

            estimated_count = (
                data['current_count'] +
                data['previous_count'] * previous_weight
            )

            if estimated_count < self.limit:
                data['current_count'] += 1
                return True

            return False

# Example
limiter = SlidingWindowCounter(limit=5, window_seconds=10)
```

---

## Redis-Based Distributed Rate Limiter

### Token Bucket with Redis

```python
import redis
import time

class RedisTokenBucket:
    """Distributed token bucket using Redis"""

    def __init__(self, redis_client: redis.Redis, capacity: int, refill_rate: float):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate

    def allow_request(self, key: str, tokens: int = 1) -> bool:
        """Check if request allowed"""
        lua_script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local tokens_requested = tonumber(ARGV[3])
        local now = tonumber(ARGV[4])

        -- Get current state
        local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1]) or capacity
        local last_refill = tonumber(bucket[2]) or now

        -- Refill tokens
        local elapsed = now - last_refill
        local tokens_to_add = elapsed * refill_rate
        tokens = math.min(capacity, tokens + tokens_to_add)

        -- Try to consume
        if tokens >= tokens_requested then
            tokens = tokens - tokens_requested

            -- Update state
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            redis.call('EXPIRE', key, 3600)  -- TTL 1 hour

            return 1  -- Allowed
        else
            return 0  -- Denied
        end
        """

        result = self.redis.eval(
            lua_script,
            1,
            key,
            self.capacity,
            self.refill_rate,
            tokens,
            time.time()
        )

        return bool(result)

# Example usage
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
limiter = RedisTokenBucket(redis_client, capacity=100, refill_rate=10.0)

# Different rate limits per user
if limiter.allow_request("user:123"):
    print("Request allowed")
else:
    print("Request denied")
```

### Fixed Window with Redis

```python
class RedisFixedWindow:
    """Fixed window counter with Redis"""

    def __init__(self, redis_client: redis.Redis, limit: int, window_seconds: int):
        self.redis = redis_client
        self.limit = limit
        self.window_seconds = window_seconds

    def allow_request(self, key: str) -> tuple[bool, dict]:
        """Check if request allowed and return rate limit info"""
        now = time.time()
        window = int(now // self.window_seconds)
        redis_key = f"rate_limit:{key}:{window}"

        # Use pipeline for atomic operations
        pipe = self.redis.pipeline()
        pipe.incr(redis_key)
        pipe.expire(redis_key, self.window_seconds)
        count, _ = pipe.execute()

        allowed = count <= self.limit

        # Calculate reset time
        reset_time = (window + 1) * self.window_seconds

        info = {
            'allowed': allowed,
            'limit': self.limit,
            'remaining': max(0, self.limit - count),
            'reset': int(reset_time),
            'retry_after': int(reset_time - now) if not allowed else 0
        }

        return allowed, info

# Example
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
limiter = RedisFixedWindow(redis_client, limit=100, window_seconds=60)

allowed, info = limiter.allow_request("api:user:123")

print(f"Allowed: {info['allowed']}")
print(f"Remaining: {info['remaining']}/{info['limit']}")
print(f"Reset in: {info['retry_after']}s")
```

---

## FastAPI Integration

### Rate Limiting Middleware

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
import time

class RateLimitMiddleware(BaseHTTPMiddleware):
    """Rate limiting middleware for FastAPI"""

    def __init__(self, app, redis_client, default_limit=100, window_seconds=60):
        super().__init__(app)
        self.limiter = RedisFixedWindow(redis_client, default_limit, window_seconds)

    async def dispatch(self, request: Request, call_next):
        # Get client identifier
        client_id = self.get_client_id(request)

        # Check rate limit
        allowed, info = self.limiter.allow_request(client_id)

        if not allowed:
            return JSONResponse(
                status_code=429,
                content={
                    'error': 'Rate limit exceeded',
                    'retry_after': info['retry_after']
                },
                headers={
                    'X-RateLimit-Limit': str(info['limit']),
                    'X-RateLimit-Remaining': '0',
                    'X-RateLimit-Reset': str(info['reset']),
                    'Retry-After': str(info['retry_after'])
                }
            )

        # Process request
        response = await call_next(request)

        # Add rate limit headers
        response.headers['X-RateLimit-Limit'] = str(info['limit'])
        response.headers['X-RateLimit-Remaining'] = str(info['remaining'])
        response.headers['X-RateLimit-Reset'] = str(info['reset'])

        return response

    def get_client_id(self, request: Request) -> str:
        """Extract client identifier"""
        # Try API key first
        api_key = request.headers.get('X-API-Key')
        if api_key:
            return f"api_key:{api_key}"

        # Fall back to IP address
        client_ip = request.client.host
        return f"ip:{client_ip}"

# FastAPI application
app = FastAPI()

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
app.add_middleware(RateLimitMiddleware, redis_client=redis_client)

@app.get("/api/data")
async def get_data():
    return {"message": "Success"}
```

### Decorator-Based Rate Limiting

```python
from functools import wraps
from fastapi import Header, HTTPException

def rate_limit(limit: int = 100, window: int = 60):
    """Rate limit decorator for endpoints"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Get request from kwargs
            request = kwargs.get('request')

            if not request:
                return await func(*args, **kwargs)

            # Get client ID
            client_id = request.client.host
            api_key = request.headers.get('X-API-Key')

            if api_key:
                client_id = f"api_key:{api_key}"
            else:
                client_id = f"ip:{client_id}"

            # Check rate limit
            limiter = RedisFixedWindow(redis_client, limit, window)
            allowed, info = limiter.allow_request(client_id)

            if not allowed:
                raise HTTPException(
                    status_code=429,
                    detail="Rate limit exceeded",
                    headers={
                        'Retry-After': str(info['retry_after'])
                    }
                )

            return await func(*args, **kwargs)

        return wrapper
    return decorator

# Usage
@app.get("/api/expensive")
@rate_limit(limit=10, window=60)  # 10 requests per minute
async def expensive_operation(request: Request):
    return {"message": "Expensive operation"}

@app.get("/api/premium")
@rate_limit(limit=1000, window=60)  # Premium tier: 1000/min
async def premium_endpoint(request: Request, x_api_key: str = Header(None)):
    return {"message": "Premium data"}
```

---

## Best Practices

### ✅ Do's:

1. **Use Redis** for distributed rate limiting
2. **Add headers** (X-RateLimit-\*) to responses
3. **Return 429** status code when rate limited
4. **Include Retry-After** header
5. **Use Lua scripts** for atomic Redis operations
6. **Set TTL** on Redis keys to prevent memory leaks
7. **Different limits** for different tiers
8. **Monitor** rate limit usage
9. **Log** rate limit violations
10. **Test** with realistic traffic patterns

### ❌ Don'ts:

1. **Don't use** in-memory counters for distributed systems
2. **Don't forget** to handle clock skew
3. **Don't block** legitimate users
4. **Don't use** overly strict limits
5. **Don't ignore** burst traffic patterns
6. **Don't forget** cleanup of old data

---

## Interview Questions

### Q1: Which rate limiting algorithm to choose?

**Answer**:

- **Token bucket**: Allows bursts, most flexible
- **Leaky bucket**: Smooth output rate
- **Fixed window**: Simple but boundary issues
- **Sliding window**: Accurate, memory efficient
  **Choice depends on**: Burst tolerance, accuracy needs, memory constraints

### Q2: How to implement distributed rate limiting?

**Answer**:

- **Use Redis**: Centralized counter storage
- **Lua scripts**: Atomic operations
- **Pipeline**: Batch Redis commands
- **TTL**: Auto-cleanup old keys
- **Replication**: HA with Redis Sentinel/Cluster
  Avoid: In-memory (not shared), database (too slow)

### Q3: How to handle rate limit at window boundary?

**Answer**:

- **Problem**: Fixed window allows 2x limit at boundary
- **Solution 1**: Sliding window log (accurate but memory)
- **Solution 2**: Sliding window counter (approximate)
- **Solution 3**: Token bucket (no boundary issue)
  Choose based on accuracy vs efficiency trade-off

### Q4: How to differentiate rate limits by user tier?

**Answer**:

- **API key**: Include tier in key (api_key:premium:123)
- **Database lookup**: Get tier, apply corresponding limit
- **Multiple limiters**: Different limits per tier
- **Headers**: Return tier-specific limits in headers
  Example: Free=100/hr, Pro=1000/hr, Enterprise=10000/hr

### Q5: What to return when rate limited?

**Answer**:

- **Status code**: 429 Too Many Requests
- **Headers**: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, Retry-After
- **Body**: Error message with retry time
- **Don't**: Return 500 or other error codes
  Standard response helps clients handle gracefully

---

## Summary

Rate limiter design essentials:

- **Token bucket**: Best general-purpose algorithm
- **Redis**: Centralized storage for distributed systems
- **Lua scripts**: Atomic operations
- **Headers**: Communicate limits to clients
- **429 status**: Standard rate limit response
- **Different tiers**: Premium users get higher limits
- **Monitor**: Track usage patterns

Control access, prevent abuse! 🏗️
