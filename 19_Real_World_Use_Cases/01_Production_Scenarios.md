# 💼 Real-World Production Scenarios

## Overview

Common production scenarios and their solutions based on real-world experience. Understanding these helps you handle similar situations effectively.

---

## Scenario 1: Database Connection Pool Exhausted

### Problem

```python
"""
Issue: Application crashes during peak traffic
Error: "QueuePool limit of size 5 overflow 10 reached, connection timed out"

Symptoms:
- Slow response times during peak hours
- Connection timeout errors
- Database refusing connections
- Application becoming unresponsive
"""
```

### Root Cause Analysis

```python
# Poor connection management
def bad_connection_usage():
    """Connections not being released"""
    # ❌ BAD: Connection never closed
    session = SessionLocal()
    users = session.query(User).all()
    return users  # Session never closed!

# Long-running transactions
def bad_transaction():
    """Transaction blocks connections"""
    # ❌ BAD: Long transaction
    session = SessionLocal()
    try:
        users = session.query(User).all()

        # Expensive operation in transaction
        for user in users:
            time.sleep(0.1)  # Simulating slow processing
            user.last_seen = datetime.now()

        session.commit()  # Holds connection entire time
    finally:
        session.close()
```

### Solution

```python
from contextlib import contextmanager
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# 1. Increase pool size
engine = create_engine(
    DATABASE_URL,
    pool_size=20,           # Increased from 5
    max_overflow=30,        # Increased from 10
    pool_pre_ping=True,     # Verify connections
    pool_recycle=3600,      # Recycle after 1 hour
)

# 2. Use context managers
@contextmanager
def get_db_session():
    """Ensure session is always closed"""
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# ✅ GOOD: Proper connection usage
def good_connection_usage():
    """Always close connections"""
    with get_db_session() as session:
        users = session.query(User).all()
        return [user.to_dict() for user in users]

# 3. FastAPI dependency (auto-cleanup)
async def get_db():
    """FastAPI dependency"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users")
async def get_users(db: Session = Depends(get_db)):
    """Connection automatically closed"""
    users = db.query(User).all()
    return users

# 4. Minimize transaction time
def optimized_transaction():
    """Process outside transaction"""
    with get_db_session() as session:
        users = session.query(User).all()

        # Convert to dicts (detach from session)
        user_dicts = [user.to_dict() for user in users]

    # Process outside transaction
    for user_dict in user_dicts:
        expensive_processing(user_dict)

    # Update in new transaction
    with get_db_session() as session:
        for user_dict in user_dicts:
            user = session.query(User).get(user_dict['id'])
            user.last_seen = datetime.now()
        session.commit()

# 5. Monitor pool usage
def monitor_pool():
    """Monitor connection pool"""
    pool = engine.pool

    logger.info(
        "Pool status",
        extra={
            "size": pool.size(),
            "checked_in": pool.checkedin(),
            "checked_out": pool.checkedout(),
            "overflow": pool.overflow(),
            "total": pool.size() + pool.overflow()
        }
    )

# Add monitoring middleware
@app.middleware("http")
async def monitor_connections(request: Request, call_next):
    """Monitor connection pool"""
    monitor_pool()

    response = await call_next(request)

    monitor_pool()

    return response
```

### Lessons Learned

```python
"""
✅ Lessons:
1. Always use context managers for DB sessions
2. Keep transactions short
3. Monitor pool usage in production
4. Set appropriate pool_size and max_overflow
5. Use pool_pre_ping to handle stale connections
6. Process data outside transactions when possible
7. Set alerts for pool exhaustion
"""
```

---

## Scenario 2: Memory Leak in Production

### Problem

```python
"""
Issue: Application memory grows continuously
Symptoms:
- Memory usage increases over time
- Eventually triggers OOM killer
- Requires daily restarts
- Performance degrades gradually

Tools used:
- memory_profiler
- tracemalloc
- objgraph
- heapdump
"""
```

### Root Cause Analysis

```python
# Found issue: Global cache without size limit
class LeakyCache:
    """❌ BAD: Unbounded cache"""
    cache = {}  # Class variable - never cleared!

    @classmethod
    def get(cls, key):
        if key not in cls.cache:
            cls.cache[key] = expensive_computation(key)
        return cls.cache[key]

# Issue: Event handlers not removed
class LeakyEventSystem:
    """❌ BAD: Accumulating event handlers"""
    def __init__(self):
        self.handlers = []

    def register_handler(self, handler):
        self.handlers.append(handler)
        # Never removed!

# Issue: Circular references
class LeakyNode:
    """❌ BAD: Circular reference"""
    def __init__(self, value):
        self.value = value
        self.next = None

    def set_next(self, node):
        self.next = node
        node.prev = self  # Circular reference!
```

### Solution

```python
from functools import lru_cache
import weakref
from collections import OrderedDict

# 1. Use LRU cache with size limit
class BoundedCache:
    """✅ GOOD: Cache with size limit"""

    def __init__(self, maxsize=1000):
        self.cache = OrderedDict()
        self.maxsize = maxsize

    def get(self, key):
        if key in self.cache:
            # Move to end (most recently used)
            self.cache.move_to_end(key)
            return self.cache[key]

        value = expensive_computation(key)

        # Evict oldest if full
        if len(self.cache) >= self.maxsize:
            self.cache.popitem(last=False)

        self.cache[key] = value
        return value

# 2. Use weak references for event handlers
class FixedEventSystem:
    """✅ GOOD: Weak references"""

    def __init__(self):
        self.handlers = []

    def register_handler(self, handler):
        """Register with weak reference"""
        weak_handler = weakref.ref(handler, self._remove_handler)
        self.handlers.append(weak_handler)

    def _remove_handler(self, weak_ref):
        """Remove dead weak reference"""
        self.handlers.remove(weak_ref)

    def emit(self, event):
        """Emit to alive handlers only"""
        for weak_handler in self.handlers[:]:
            handler = weak_handler()
            if handler is not None:
                handler(event)

# 3. Break circular references
class FixedNode:
    """✅ GOOD: Weak reference for backlink"""

    def __init__(self, value):
        self.value = value
        self.next = None
        self._prev = None

    @property
    def prev(self):
        return self._prev() if self._prev else None

    @prev.setter
    def prev(self, node):
        self._prev = weakref.ref(node) if node else None

# 4. Memory monitoring
import tracemalloc
import gc

class MemoryMonitor:
    """Monitor memory usage"""

    def __init__(self):
        tracemalloc.start()
        self.snapshots = []

    def take_snapshot(self):
        """Take memory snapshot"""
        snapshot = tracemalloc.take_snapshot()
        self.snapshots.append(snapshot)

        if len(self.snapshots) > 1:
            self.compare_snapshots()

    def compare_snapshots(self):
        """Compare memory growth"""
        current = self.snapshots[-1]
        previous = self.snapshots[-2]

        top_stats = current.compare_to(previous, 'lineno')

        print("Top memory increases:")
        for stat in top_stats[:10]:
            print(stat)

    def get_top_allocations(self):
        """Get top memory allocations"""
        snapshot = tracemalloc.take_snapshot()
        top_stats = snapshot.statistics('lineno')

        return top_stats[:20]

# Schedule periodic memory checks
from apscheduler.schedulers.background import BackgroundScheduler

memory_monitor = MemoryMonitor()
scheduler = BackgroundScheduler()

scheduler.add_job(
    memory_monitor.take_snapshot,
    'interval',
    minutes=10
)
scheduler.start()

# 5. Force garbage collection periodically
import gc

@app.middleware("http")
async def gc_middleware(request: Request, call_next):
    """Periodic garbage collection"""
    # Every 100 requests
    if request.state.request_count % 100 == 0:
        collected = gc.collect()
        logger.info(f"Garbage collected: {collected} objects")

    return await call_next(request)
```

### Lessons Learned

```python
"""
✅ Lessons:
1. Always set size limits on caches
2. Use weak references for callback/event handlers
3. Break circular references explicitly
4. Monitor memory in production
5. Use LRU cache for automatic eviction
6. Profile memory before deploying
7. Set up alerts for memory growth
8. Consider using Redis instead of in-memory cache
"""
```

---

## Scenario 3: Slow API Response Times

### Problem

```python
"""
Issue: API responses taking 3-5 seconds
Target: < 500ms
Impact: Poor user experience, timeouts

Initial investigation:
- Database queries look fast in isolation
- No obvious bottlenecks in code
- CPU and memory usage normal
"""
```

### Root Cause Analysis

```python
# Found issue: N+1 query problem
@app.get("/users/{user_id}/posts")
async def get_user_posts(user_id: int, db: Session = Depends(get_db)):
    """❌ BAD: N+1 queries"""
    user = db.query(User).filter(User.id == user_id).first()

    # 1 query for user
    posts = user.posts  # N queries (one per post)

    result = []
    for post in posts:
        result.append({
            "id": post.id,
            "title": post.title,
            "author": post.author.name,  # N more queries!
            "comments_count": len(post.comments)  # N more queries!
        })

    return result
```

### Solution

```python
from sqlalchemy.orm import joinedload, selectinload, subqueryload
from sqlalchemy import func

# 1. Use eager loading
@app.get("/users/{user_id}/posts")
async def get_user_posts_fixed(user_id: int, db: Session = Depends(get_db)):
    """✅ GOOD: Eager loading"""
    user = db.query(User)\
        .options(
            selectinload(User.posts)
            .joinedload(Post.author)
            .selectinload(Post.comments)
        )\
        .filter(User.id == user_id)\
        .first()

    result = []
    for post in user.posts:
        result.append({
            "id": post.id,
            "title": post.title,
            "author": post.author.name,
            "comments_count": len(post.comments)
        })

    return result

# 2. Use aggregation instead of loading all
@app.get("/users/{user_id}/posts/optimized")
async def get_user_posts_optimized(user_id: int, db: Session = Depends(get_db)):
    """✅ BETTER: Use aggregation"""
    posts = db.query(
        Post.id,
        Post.title,
        User.name.label('author_name'),
        func.count(Comment.id).label('comments_count')
    )\
    .join(User, Post.author_id == User.id)\
    .outerjoin(Comment, Post.id == Comment.post_id)\
    .filter(Post.user_id == user_id)\
    .group_by(Post.id, Post.title, User.name)\
    .all()

    return [
        {
            "id": post.id,
            "title": post.title,
            "author": post.author_name,
            "comments_count": post.comments_count
        }
        for post in posts
    ]

# 3. Add caching
from functools import lru_cache
import redis

redis_client = redis.Redis()

@app.get("/users/{user_id}/posts/cached")
async def get_user_posts_cached(user_id: int, db: Session = Depends(get_db)):
    """✅ BEST: With caching"""
    cache_key = f"user_posts:{user_id}"

    # Try cache
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # Query database
    posts = get_user_posts_optimized(user_id, db)

    # Cache for 5 minutes
    redis_client.setex(cache_key, 300, json.dumps(posts))

    return posts

# 4. Add monitoring
import time

@app.middleware("http")
async def timing_middleware(request: Request, call_next):
    """Log slow requests"""
    start = time.time()

    response = await call_next(request)

    duration = time.time() - start

    # Log slow requests
    if duration > 1.0:
        logger.warning(
            "Slow request",
            extra={
                "path": request.url.path,
                "method": request.method,
                "duration": duration
            }
        )

    return response

# 5. Add database query logging
import logging

logging.basicConfig()
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)

# Or programmatically
@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    conn.info.setdefault('query_start_time', []).append(time.time())

@event.listens_for(Engine, "after_cursor_execute")
def after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    total = time.time() - conn.info['query_start_time'].pop()

    if total > 0.1:  # Log queries > 100ms
        logger.warning(
            f"Slow query: {total:.2f}s",
            extra={"query": statement}
        )
```

### Lessons Learned

```python
"""
✅ Lessons:
1. Profile before optimizing (measure, don't guess)
2. Watch for N+1 queries (use eager loading)
3. Use aggregation instead of loading all data
4. Add caching for frequently accessed data
5. Monitor query performance in production
6. Set SLAs and alert on violations
7. Use database query logging
8. Consider pagination for large datasets
"""
```

---

## Scenario 4: Race Condition in Order Processing

### Problem

```python
"""
Issue: Some orders processed twice
Impact: Double charges, inventory issues

Symptoms:
- Occasional duplicate orders
- Inconsistent inventory counts
- More common during high traffic
"""
```

### Root Cause Analysis

```python
# Found issue: No locking on order processing
async def process_order(order_id: int):
    """❌ BAD: Race condition"""
    order = db.query(Order).filter(Order.id == order_id).first()

    if order.status == "pending":
        # Another request might reach here too!

        # Process payment
        charge_customer(order)

        # Update inventory
        update_inventory(order.items)

        # Mark as processed
        order.status = "processed"
        db.commit()
```

### Solution

```python
from sqlalchemy import select
from redis import Redis
import redis.lock

redis_client = Redis()

# 1. Use database row locking
async def process_order_with_lock(order_id: int, db: Session):
    """✅ GOOD: Database row lock"""
    # SELECT ... FOR UPDATE
    order = db.query(Order)\
        .filter(Order.id == order_id)\
        .with_for_update()\
        .first()

    if order.status == "pending":
        # Process payment
        charge_customer(order)

        # Update inventory
        update_inventory(order.items)

        # Mark as processed
        order.status = "processed"
        db.commit()

# 2. Use distributed lock (Redis)
async def process_order_with_redis_lock(order_id: int, db: Session):
    """✅ GOOD: Distributed lock"""
    lock_key = f"order_lock:{order_id}"

    # Try to acquire lock
    lock = redis_client.lock(lock_key, timeout=30)

    if not lock.acquire(blocking=False):
        raise HTTPException(409, "Order is being processed")

    try:
        order = db.query(Order).filter(Order.id == order_id).first()

        if order.status == "pending":
            charge_customer(order)
            update_inventory(order.items)

            order.status = "processed"
            db.commit()

    finally:
        lock.release()

# 3. Idempotency with idempotency key
async def process_order_idempotent(
    order_id: int,
    idempotency_key: str,
    db: Session
):
    """✅ BEST: Idempotent with key"""
    # Check if already processed
    existing = redis_client.get(f"idempotency:{idempotency_key}")
    if existing:
        return json.loads(existing)

    # Process with lock
    lock_key = f"order_lock:{order_id}"
    with redis_client.lock(lock_key, timeout=30):
        order = db.query(Order).filter(Order.id == order_id).first()

        if order.status == "pending":
            charge_customer(order)
            update_inventory(order.items)

            order.status = "processed"
            db.commit()

        result = order.to_dict()

        # Store result with idempotency key
        redis_client.setex(
            f"idempotency:{idempotency_key}",
            86400,  # 24 hours
            json.dumps(result)
        )

        return result

# FastAPI endpoint with idempotency
@app.post("/orders/{order_id}/process")
async def process_order_endpoint(
    order_id: int,
    idempotency_key: str = Header(...),
    db: Session = Depends(get_db)
):
    """Process order idempotently"""
    return await process_order_idempotent(order_id, idempotency_key, db)
```

### Lessons Learned

```python
"""
✅ Lessons:
1. Use locking for critical operations
2. Implement idempotency for state-changing operations
3. Use database row locking when possible
4. Use distributed locks (Redis) for multi-instance apps
5. Add idempotency keys to API endpoints
6. Test race conditions explicitly
7. Monitor for duplicate processing
8. Use transactions appropriately
"""
```

---

## Best Practices

### ✅ Production Checklist:

1. **Connection Management**: Use context managers, pool monitoring
2. **Memory Management**: Size limits, weak references, monitoring
3. **Performance**: Eager loading, caching, pagination
4. **Concurrency**: Locking, idempotency, transactions
5. **Monitoring**: Logging, metrics, alerts
6. **Testing**: Load testing, race condition testing
7. **Documentation**: Runbooks, incident response

---

## Summary

Key takeaways from production:

- **Measure before optimizing** - profile first
- **Monitor everything** - metrics, logs, traces
- **Test under load** - find issues before production
- **Use proper locking** - prevent race conditions
- **Implement idempotency** - safe retries
- **Cache strategically** - but invalidate properly
- **Learn from incidents** - document and improve

Real-world experience builds expertise! 💼
