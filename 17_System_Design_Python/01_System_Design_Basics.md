# System Design Basics for Python Backend Engineers

## 📖 Introduction

System design interviews assess your ability to architect scalable, maintainable systems that handle real-world challenges. For mid-senior backend engineers, this is often the most critical part of the interview process.

## 🎯 The 4-Step Framework

### Step 1: Understand Requirements (5-10 minutes)

Ask clarifying questions to understand:

**Functional Requirements**

- What features must the system support?
- What are the core use cases?
- What is out of scope?

**Non-Functional Requirements**

- Scale: How many users? Requests per second?
- Performance: What are the latency requirements?
- Availability: What's the acceptable downtime?
- Consistency: Strong or eventual consistency?
- Security: Authentication, authorization, data privacy?

**Example Questions:**

```
For a URL Shortener:
- Do we need custom short URLs?
- Should URLs expire?
- Do we need analytics?
- Expected QPS?
- Acceptable redirect latency?
```

### Step 2: Back-of-Envelope Estimation (5 minutes)

Calculate:

- **Traffic**: Read/Write QPS, peak traffic
- **Storage**: Data size, growth rate, retention period
- **Bandwidth**: Network throughput needed
- **Memory**: Cache size for hot data

**Example Calculation:**

```python
# URL Shortener Estimation
class SystemEstimation:
    """Calculate system capacity requirements"""

    def __init__(self):
        # Traffic assumptions
        self.daily_active_users = 10_000_000
        self.urls_per_user_per_day = 0.1
        self.read_write_ratio = 100  # 100 reads per write

        # Storage assumptions
        self.avg_url_size = 500  # bytes
        self.retention_years = 5

    def calculate_qps(self):
        """Calculate queries per second"""
        # Write QPS
        daily_writes = self.daily_active_users * self.urls_per_user_per_day
        write_qps = daily_writes / (24 * 3600)

        # Read QPS
        read_qps = write_qps * self.read_write_ratio

        # Peak QPS (assume 3x average)
        peak_read_qps = read_qps * 3
        peak_write_qps = write_qps * 3

        return {
            'write_qps': write_qps,
            'read_qps': read_qps,
            'peak_read_qps': peak_read_qps,
            'peak_write_qps': peak_write_qps
        }

    def calculate_storage(self):
        """Calculate storage requirements"""
        daily_writes = self.daily_active_users * self.urls_per_user_per_day
        total_urls = daily_writes * 365 * self.retention_years
        total_storage = total_urls * self.avg_url_size

        # Convert to human-readable
        storage_gb = total_storage / (1024 ** 3)
        storage_tb = storage_gb / 1024

        return {
            'total_urls': total_urls,
            'storage_bytes': total_storage,
            'storage_gb': storage_gb,
            'storage_tb': storage_tb
        }

    def calculate_bandwidth(self):
        """Calculate bandwidth requirements"""
        qps = self.calculate_qps()

        # Write bandwidth
        write_bandwidth = qps['write_qps'] * self.avg_url_size

        # Read bandwidth
        read_bandwidth = qps['read_qps'] * self.avg_url_size

        # Convert to MB/s
        write_bandwidth_mb = write_bandwidth / (1024 * 1024)
        read_bandwidth_mb = read_bandwidth / (1024 * 1024)

        return {
            'write_bandwidth_mb': write_bandwidth_mb,
            'read_bandwidth_mb': read_bandwidth_mb
        }

    def print_estimation(self):
        """Print complete estimation"""
        qps = self.calculate_qps()
        storage = self.calculate_storage()
        bandwidth = self.calculate_bandwidth()

        print("=== System Capacity Estimation ===\n")

        print("Traffic:")
        print(f"  Write QPS: {qps['write_qps']:.2f}")
        print(f"  Read QPS: {qps['read_qps']:.2f}")
        print(f"  Peak Read QPS: {qps['peak_read_qps']:.2f}")
        print(f"  Peak Write QPS: {qps['peak_write_qps']:.2f}\n")

        print("Storage:")
        print(f"  Total URLs: {storage['total_urls']:,.0f}")
        print(f"  Total Storage: {storage['storage_tb']:.2f} TB\n")

        print("Bandwidth:")
        print(f"  Write: {bandwidth['write_bandwidth_mb']:.2f} MB/s")
        print(f"  Read: {bandwidth['read_bandwidth_mb']:.2f} MB/s")

# Usage
estimator = SystemEstimation()
estimator.print_estimation()
```

**Output:**

```
=== System Capacity Estimation ===

Traffic:
  Write QPS: 11.57
  Read QPS: 1157.41
  Peak Read QPS: 3472.22
  Peak Write QPS: 34.72

Storage:
  Total URLs: 1,825,000,000
  Total Storage: 0.85 TB

Bandwidth:
  Write: 0.01 MB/s
  Read: 0.55 MB/s
```

### Step 3: High-Level Design (10-15 minutes)

Draw architecture diagram with:

- **Clients**: Web, mobile, APIs
- **Load Balancer**: Distribute traffic
- **Application Servers**: Business logic
- **Databases**: Storage layer
- **Caches**: Performance layer
- **Message Queues**: Async processing
- **CDN**: Static content delivery

**Example Architecture:**

```
┌─────────────────────────────────────────────────────┐
│                     CLIENT TIER                      │
│  (Web Browser, Mobile App, API Consumers)           │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                  CDN (Static Assets)                 │
└─────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│              DNS / Load Balancer                     │
│         (Route 53, Nginx, HAProxy)                   │
└────────────────────┬────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
┌──────────────┐          ┌──────────────┐
│ API Server 1 │          │ API Server N │
│  (Django/    │   ...    │  (Django/    │
│   FastAPI)   │          │   FastAPI)   │
└──────┬───────┘          └──────┬───────┘
       │                         │
       └────────────┬────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌──────────────┐        ┌──────────────┐
│    Cache     │        │   Database   │
│   (Redis)    │        │ (PostgreSQL) │
│              │        │   Master     │
└──────────────┘        └──────┬───────┘
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
            ┌──────────────┐      ┌──────────────┐
            │   DB Replica │      │   DB Replica │
            │   (Read)     │      │   (Read)     │
            └──────────────┘      └──────────────┘
```

### Step 4: Deep Dive & Optimization (15-20 minutes)

Discuss:

- **Bottlenecks**: Identify and address
- **Trade-offs**: Different approaches
- **Scaling**: Horizontal vs vertical
- **Failure scenarios**: What can go wrong?
- **Monitoring**: How to detect issues?

---

## 🏗️ Core System Design Concepts

### 1. CAP Theorem

**Definition**: In a distributed system, you can only guarantee 2 out of 3:

- **C**onsistency: All nodes see the same data
- **A**vailability: System responds to requests
- **P**artition Tolerance: System works despite network failures

**In Practice:**

```python
"""
Real-world trade-offs:

CP System (Consistency + Partition Tolerance):
- MongoDB, HBase, Redis Cluster
- Use when: Banking, inventory management
- Sacrifice: Availability during network partitions

AP System (Availability + Partition Tolerance):
- Cassandra, DynamoDB, Riak
- Use when: Social media feeds, analytics
- Sacrifice: Temporary inconsistency

CA System (Consistency + Availability):
- PostgreSQL, MySQL (single node)
- Use when: Traditional RDBMS
- Sacrifice: Cannot handle network partitions (single point of failure)

Python Example - Choosing Database:
"""

class DatabaseSelector:
    """Select database based on CAP requirements"""

    @staticmethod
    def recommend(use_case: str) -> dict:
        recommendations = {
            'banking': {
                'type': 'CP',
                'database': 'PostgreSQL with ACID',
                'reason': 'Consistency is critical, temporary unavailability acceptable'
            },
            'social_media': {
                'type': 'AP',
                'database': 'Cassandra',
                'reason': 'Availability critical, eventual consistency acceptable'
            },
            'e_commerce_cart': {
                'type': 'AP',
                'database': 'DynamoDB',
                'reason': 'High availability needed, cart conflicts rare'
            },
            'e_commerce_payment': {
                'type': 'CP',
                'database': 'PostgreSQL',
                'reason': 'Payment consistency is non-negotiable'
            }
        }
        return recommendations.get(use_case, recommendations['banking'])

# Usage
print(DatabaseSelector.recommend('banking'))
# {'type': 'CP', 'database': 'PostgreSQL with ACID',
#  'reason': 'Consistency is critical...'}
```

### 2. ACID vs BASE

**ACID (Traditional Databases)**

- **A**tomicity: All or nothing
- **C**onsistency: Valid state always
- **I**solation: Concurrent transactions don't interfere
- **D**urability: Committed data persists

**BASE (Distributed Systems)**

- **B**asically **A**vailable: System available most of the time
- **S**oft state: State may change without input
- **E**ventual consistency: Data becomes consistent eventually

```python
# ACID Example - Django Transaction
from django.db import transaction
from django.core.exceptions import ValidationError

class BankAccount(models.Model):
    balance = models.DecimalField(max_digits=10, decimal_places=2)

    @transaction.atomic
    def transfer(self, to_account, amount):
        """
        ACID Transaction: Transfer money between accounts
        Atomicity: Both debit and credit happen or neither
        Consistency: Balance constraints maintained
        Isolation: Concurrent transfers don't interfere
        Durability: Once committed, transfer persists
        """
        if self.balance < amount:
            raise ValidationError("Insufficient funds")

        # Debit from source
        self.balance -= amount
        self.save()

        # Credit to destination
        to_account.balance += amount
        to_account.save()

        # If any step fails, entire transaction rolls back

# BASE Example - Eventual Consistency
import asyncio
from typing import List

class EventuallyConsistentCache:
    """
    BASE system: Updates propagate eventually
    Use for: Social media likes, view counts
    """

    def __init__(self):
        self.nodes = {}

    async def write(self, key: str, value: int):
        """Write to primary node"""
        self.nodes['primary'] = self.nodes.get('primary', {})
        self.nodes['primary'][key] = value

        # Asynchronously propagate to replicas
        asyncio.create_task(self._propagate(key, value))

    async def _propagate(self, key: str, value: int):
        """Propagate to replicas with delay (network latency)"""
        await asyncio.sleep(0.1)  # Simulate network delay

        for node_name in ['replica1', 'replica2']:
            self.nodes[node_name] = self.nodes.get(node_name, {})
            self.nodes[node_name][key] = value

    def read(self, key: str, node: str = 'primary') -> int:
        """Read may return stale data from replicas"""
        return self.nodes.get(node, {}).get(key, 0)

# Usage
async def demo_eventual_consistency():
    cache = EventuallyConsistentCache()

    # Write to primary
    await cache.write('likes', 100)

    # Immediate read from primary: consistent
    print(f"Primary: {cache.read('likes', 'primary')}")  # 100

    # Immediate read from replica: may be stale
    print(f"Replica: {cache.read('likes', 'replica1')}")  # 0 (not propagated yet)

    # Wait for propagation
    await asyncio.sleep(0.2)
    print(f"Replica after propagation: {cache.read('likes', 'replica1')}")  # 100

# asyncio.run(demo_eventual_consistency())
```

### 3. Horizontal vs Vertical Scaling

**Vertical Scaling (Scale Up)**

- Add more CPU, RAM, disk to existing server
- **Pros**: Simple, no code changes
- **Cons**: Hardware limits, single point of failure
- **Use when**: Quick fix, simpler architecture

**Horizontal Scaling (Scale Out)**

- Add more servers
- **Pros**: No limits, high availability
- **Cons**: Complex (load balancing, distributed state)
- **Use when**: Long-term scalability needed

```python
# Vertical Scaling Example
"""
Single Django server handling requests:

Before: 4 CPU, 8GB RAM
After:  16 CPU, 64GB RAM

Simple but limited!
"""

# Horizontal Scaling Example - Load Balanced Django
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# Session storage in Redis (shared across servers)
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'

# views.py - Stateless design for horizontal scaling
from django.core.cache import cache
from django.views import View
from django.http import JsonResponse

class StatelessAPIView(View):
    """
    Stateless design allows any server to handle request
    No server-specific state stored
    """

    def get(self, request):
        user_id = request.GET.get('user_id')

        # Get from shared cache (Redis)
        cache_key = f"user_data:{user_id}"
        data = cache.get(cache_key)

        if not data:
            # Fetch from database
            data = self.fetch_from_db(user_id)
            cache.set(cache_key, data, timeout=300)

        return JsonResponse(data)

    def fetch_from_db(self, user_id):
        # Database query
        return {'user_id': user_id, 'name': 'John'}
```

### 4. Load Balancing Strategies

```python
"""
Load Balancing Algorithms:
1. Round Robin: Distribute requests equally
2. Least Connections: Send to server with fewest active connections
3. IP Hash: Same client always goes to same server
4. Weighted Round Robin: Servers with different capacities
"""

import hashlib
from typing import List

class LoadBalancer:
    """Implement different load balancing strategies"""

    def __init__(self, servers: List[str]):
        self.servers = servers
        self.round_robin_index = 0
        self.connections = {server: 0 for server in servers}

    def round_robin(self) -> str:
        """Round Robin: Distribute evenly"""
        server = self.servers[self.round_robin_index]
        self.round_robin_index = (self.round_robin_index + 1) % len(self.servers)
        return server

    def least_connections(self) -> str:
        """Least Connections: Send to least busy server"""
        return min(self.connections, key=self.connections.get)

    def ip_hash(self, client_ip: str) -> str:
        """IP Hash: Consistent routing for same client"""
        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        server_index = hash_value % len(self.servers)
        return self.servers[server_index]

    def weighted_round_robin(self, weights: dict) -> str:
        """Weighted: More requests to powerful servers"""
        # Simplified: Use weight as probability
        total_weight = sum(weights.values())
        import random
        rand_weight = random.uniform(0, total_weight)

        cumulative = 0
        for server, weight in weights.items():
            cumulative += weight
            if rand_weight <= cumulative:
                return server

        return self.servers[0]

# Usage
lb = LoadBalancer(['server1', 'server2', 'server3'])

# Round robin
print(lb.round_robin())  # server1
print(lb.round_robin())  # server2
print(lb.round_robin())  # server3

# IP hash (same IP always goes to same server)
print(lb.ip_hash('192.168.1.100'))  # Always same server
print(lb.ip_hash('192.168.1.100'))  # Same as above

# Weighted (server1 is more powerful)
weights = {'server1': 3, 'server2': 1, 'server3': 1}
print(lb.weighted_round_robin(weights))  # More likely server1
```

### 5. Caching Strategies

**Cache Levels:**

1. **CDN**: Static assets (images, CSS, JS)
2. **Application Cache**: API responses, database queries
3. **Database Cache**: Query results

**Cache Patterns:**

- **Cache-Aside**: App checks cache, then DB
- **Write-Through**: Write to cache and DB together
- **Write-Back**: Write to cache, async to DB
- **Refresh-Ahead**: Proactively refresh before expiry

```python
# Django Cache Patterns
from django.core.cache import cache
from django.db import models
from functools import wraps
import hashlib

# Pattern 1: Cache-Aside (Lazy Loading)
def get_user_profile(user_id):
    """Check cache first, then database"""
    cache_key = f"user_profile:{user_id}"

    # Try cache
    profile = cache.get(cache_key)
    if profile:
        return profile

    # Cache miss - fetch from DB
    profile = UserProfile.objects.get(id=user_id)

    # Store in cache for next time
    cache.set(cache_key, profile, timeout=3600)

    return profile

# Pattern 2: Write-Through Cache
def update_user_profile(user_id, data):
    """Update both cache and database together"""
    cache_key = f"user_profile:{user_id}"

    # Update database
    profile = UserProfile.objects.get(id=user_id)
    profile.name = data['name']
    profile.save()

    # Update cache immediately
    cache.set(cache_key, profile, timeout=3600)

    return profile

# Pattern 3: Cache Decorator
def cache_result(timeout=300, key_prefix=''):
    """Decorator to cache function results"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generate cache key from function name and arguments
            key_parts = [key_prefix or func.__name__]
            key_parts.extend(str(arg) for arg in args)
            key_parts.extend(f"{k}={v}" for k, v in sorted(kwargs.items()))

            cache_key = hashlib.md5(':'.join(key_parts).encode()).hexdigest()

            # Try cache
            result = cache.get(cache_key)
            if result is not None:
                return result

            # Execute function
            result = func(*args, **kwargs)

            # Cache result
            cache.set(cache_key, result, timeout=timeout)

            return result
        return wrapper
    return decorator

# Usage
@cache_result(timeout=600, key_prefix='user_posts')
def get_user_posts(user_id, limit=10):
    """Cached function - results stored automatically"""
    return Post.objects.filter(user_id=user_id)[:limit]

# FastAPI with Redis Cache
from fastapi import FastAPI
import redis
import json

app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

@app.get("/api/posts/{user_id}")
async def get_posts(user_id: int):
    """API with cache-aside pattern"""
    cache_key = f"posts:{user_id}"

    # Check cache
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # Fetch from database
    posts = fetch_posts_from_db(user_id)

    # Cache for 5 minutes
    redis_client.setex(cache_key, 300, json.dumps(posts))

    return posts

def fetch_posts_from_db(user_id: int):
    # Database query
    return [{"id": 1, "title": "Post 1"}, {"id": 2, "title": "Post 2"}]
```

### 6. Database Scaling Techniques

**Replication:**

- Master-Slave: Write to master, read from slaves
- Master-Master: Write to multiple masters

**Sharding:**

- Horizontal partitioning: Split data across servers
- Vertical partitioning: Split tables across servers

```python
# Database Replication - Django
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'master.db.com',
        'PORT': 5432,
    },
    'replica1': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'replica1.db.com',
        'PORT': 5432,
    },
    'replica2': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'replica2.db.com',
        'PORT': 5432,
    }
}

# Database router
class ReplicationRouter:
    """
    Route reads to replicas, writes to master
    """

    def db_for_read(self, model, **hints):
        """Direct read queries to replicas"""
        import random
        return random.choice(['replica1', 'replica2'])

    def db_for_write(self, model, **hints):
        """Direct write queries to master"""
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        """Allow relations if both in same database"""
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """Only migrate on master"""
        return db == 'default'

# settings.py
DATABASE_ROUTERS = ['myapp.routers.ReplicationRouter']

# Sharding Example - Hash-based
class ShardRouter:
    """
    Shard users by user_id
    Shard 0: user_id % 4 == 0
    Shard 1: user_id % 4 == 1
    Shard 2: user_id % 4 == 2
    Shard 3: user_id % 4 == 3
    """

    def get_shard(self, user_id: int) -> str:
        """Determine which shard for given user"""
        shard_num = user_id % 4
        return f'shard_{shard_num}'

    def db_for_read(self, model, **hints):
        if model.__name__ == 'User' and 'user_id' in hints:
            return self.get_shard(hints['user_id'])
        return 'default'

    def db_for_write(self, model, **hints):
        if model.__name__ == 'User' and 'user_id' in hints:
            return self.get_shard(hints['user_id'])
        return 'default'

# Query with hint
user = User.objects.using(
    ShardRouter().get_shard(user_id=12345)
).get(id=12345)
```

### 7. Message Queues for Async Processing

**Use Cases:**

- Send emails without blocking request
- Process images/videos
- Generate reports
- Handle webhooks

```python
# Celery with Django
from celery import shared_task
from django.core.mail import send_mail

@shared_task
def send_welcome_email(user_id):
    """
    Process asynchronously - don't block HTTP response
    """
    user = User.objects.get(id=user_id)
    send_mail(
        'Welcome!',
        f'Hello {user.name}',
        'noreply@example.com',
        [user.email],
    )

# views.py
def register_user(request):
    """User registration - fast response"""
    user = User.objects.create(
        email=request.POST['email'],
        name=request.POST['name']
    )

    # Queue email task (returns immediately)
    send_welcome_email.delay(user.id)

    return JsonResponse({'success': True})

# FastAPI with Background Tasks
from fastapi import BackgroundTasks

@app.post("/register")
async def register(email: str, background_tasks: BackgroundTasks):
    """Register user with async email"""
    user = create_user(email)

    # Add task to background queue
    background_tasks.add_task(send_welcome_email, user.id)

    return {"user_id": user.id}
```

---

## 🎯 Common System Design Trade-offs

### 1. Consistency vs Availability

**Strong Consistency:**

```python
# Banking: Every read sees latest write
# Trade-off: May be unavailable during partition

@transaction.atomic
def transfer_money(from_account, to_account, amount):
    # Locks rows - other transactions wait
    from_account = Account.objects.select_for_update().get(id=from_account)
    to_account = Account.objects.select_for_update().get(id=to_account)

    from_account.balance -= amount
    to_account.balance += amount

    from_account.save()
    to_account.save()
    # Commit ensures consistency
```

**Eventual Consistency:**

```python
# Social media: Reads may be stale temporarily
# Trade-off: Always available, but inconsistent

async def like_post(post_id, user_id):
    # Update local cache immediately (fast response)
    cache.incr(f"likes:{post_id}")

    # Queue database update (async)
    await queue.enqueue('update_likes', post_id, user_id)

    # User sees +1 immediately, DB updated later
```

### 2. Latency vs Throughput

**Low Latency (optimize for speed):**

```python
# In-memory cache, small payloads
@cache_result(timeout=60)
def get_user_summary(user_id):
    # Fast: returns minimal data from cache
    return {
        'id': user_id,
        'name': user.name,
        'avatar': user.avatar_url
    }
```

**High Throughput (optimize for volume):**

```python
# Batch processing, larger payloads
def process_batch(user_ids):
    # Process 1000 users at once (slower per user, higher total)
    users = User.objects.filter(id__in=user_ids).prefetch_related('posts')

    for user in users:
        calculate_statistics(user)
```

### 3. Read-Heavy vs Write-Heavy

**Read-Heavy Optimization:**

```python
# Denormalize, add caching, read replicas
class Article(models.Model):
    title = models.CharField(max_length=200)
    view_count = models.IntegerField(default=0)
    # Denormalized: Store count directly (faster reads)

    # Read from cache
    @classmethod
    def get_cached(cls, article_id):
        cache_key = f"article:{article_id}"
        article = cache.get(cache_key)
        if not article:
            article = cls.objects.get(id=article_id)
            cache.set(cache_key, article, 3600)
        return article
```

**Write-Heavy Optimization:**

```python
# Async writes, batch updates, write-behind cache
async def track_view(article_id):
    # Write to cache immediately
    await cache.incr(f"views:{article_id}")

    # Batch update to database every 60 seconds
    # (reduces DB writes from 10K/sec to ~166/sec)
```

---

## 🔐 Security Considerations

### Authentication & Authorization

```python
# JWT-based authentication
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework.permissions import IsAuthenticated

class ProtectedView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        # Request must include valid JWT token
        return Response({'user': request.user.username})
```

### Rate Limiting

```python
from django.core.cache import cache
from rest_framework.throttling import SimpleRateThrottle

class CustomRateThrottle(SimpleRateThrottle):
    """100 requests per minute per user"""
    rate = '100/min'

    def get_cache_key(self, request, view):
        return f"throttle:{request.user.id}"
```

### Input Validation

```python
from pydantic import BaseModel, EmailStr, constr

class UserCreate(BaseModel):
    email: EmailStr  # Validates email format
    password: constr(min_length=8)  # Minimum 8 characters
    age: int = Field(ge=18, le=120)  # Between 18 and 120
```

---

## 📊 Monitoring & Observability

```python
# Logging
import logging

logger = logging.getLogger(__name__)

def process_payment(order_id, amount):
    logger.info(f"Processing payment: order={order_id}, amount={amount}")

    try:
        result = payment_gateway.charge(amount)
        logger.info(f"Payment successful: order={order_id}, txn={result.id}")
    except PaymentError as e:
        logger.error(f"Payment failed: order={order_id}, error={str(e)}")
        raise

# Metrics with Prometheus
from prometheus_client import Counter, Histogram

request_count = Counter('http_requests_total', 'Total HTTP requests')
request_duration = Histogram('http_request_duration_seconds', 'Request duration')

@app.middleware("http")
async def metrics_middleware(request, call_next):
    request_count.inc()

    with request_duration.time():
        response = await call_next(request)

    return response
```

---

## ❓ Interview Questions & Answers

### Q1: How do you handle a suddenly viral post that gets millions of views?

**Answer:**

1. **Caching**: Aggressively cache hot content in Redis/CDN
2. **Read Replicas**: Route reads to multiple database replicas
3. **Load Balancing**: Distribute traffic across multiple servers
4. **Async Processing**: Queue non-critical updates (like counts)
5. **Rate Limiting**: Prevent abuse
6. **Auto-scaling**: Add servers automatically based on load

```python
# Implementation
@cache_result(timeout=30)  # Short TTL for viral content
def get_viral_post(post_id):
    # Check if post is viral (high traffic)
    request_rate = cache.get(f"rate:{post_id}") or 0

    if request_rate > 1000:  # Viral threshold
        # Serve from CDN, extend cache
        cache.set(f"post:{post_id}", post_data, timeout=300)

    return post_data
```

### Q2: How would you migrate a monolith to microservices?

**Answer:**

1. **Start with boundaries**: Identify independent domains
2. **Extract read-only first**: Safest to start
3. **Implement API gateway**: Central entry point
4. **Use messaging**: Decouple services with queues
5. **Gradual migration**: One service at a time
6. **Maintain data consistency**: Eventual consistency or sagas

### Q3: How do you ensure data consistency across microservices?

**Answer:**

1. **Two-Phase Commit (2PC)**: Slow, not recommended
2. **Saga Pattern**: Sequence of local transactions
3. **Event Sourcing**: Store events, rebuild state
4. **CQRS**: Separate read and write models

```python
# Saga Pattern Example
class OrderSaga:
    """Distributed transaction across services"""

    async def create_order(self, order_data):
        # Step 1: Reserve inventory
        inventory_id = await inventory_service.reserve(order_data['items'])

        try:
            # Step 2: Process payment
            payment_id = await payment_service.charge(order_data['amount'])

            try:
                # Step 3: Create order
                order_id = await order_service.create(order_data)
                return order_id

            except OrderError:
                # Compensate: Refund payment
                await payment_service.refund(payment_id)
                raise

        except PaymentError:
            # Compensate: Release inventory
            await inventory_service.release(inventory_id)
            raise
```

### Q4: How do you design for failure?

**Answer:**

1. **Circuit Breaker**: Stop calling failing service
2. **Retry with Backoff**: Retry failed requests with delays
3. **Fallback**: Return cached/default data
4. **Timeout**: Don't wait forever
5. **Bulkhead**: Isolate failures

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=60)
def call_external_api():
    """Circuit breaker: stops calling after 5 failures"""
    response = requests.get('https://api.example.com')
    return response.json()
```

---

## 📚 Summary

**Key Takeaways:**

1. **Always start with requirements** - Clarify before designing
2. **Do back-of-envelope calculations** - Know your scale
3. **Start simple, then optimize** - Don't over-engineer
4. **Trade-offs are inevitable** - Understand CAP, ACID vs BASE
5. **Think about failures** - Design for resilience
6. **Monitor everything** - Observability is critical
7. **Security first** - Build it in from the start

**Common Patterns:**

- Load Balancer → App Servers → Cache → Database
- Write to master, read from replicas
- Cache aggressively, invalidate carefully
- Queue non-critical work
- Scale horizontally with stateless servers

**Python-Specific:**

- Django for traditional monoliths, tight integration
- FastAPI for async, microservices, high performance
- Celery for background tasks
- Redis for caching and sessions
- PostgreSQL for ACID, Cassandra for AP

**Remember:** There's no perfect design - only trade-offs that match your requirements.
