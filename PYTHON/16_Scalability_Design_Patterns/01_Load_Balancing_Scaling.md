# ⚖️ Load Balancing and Horizontal Scaling

## Overview

Understanding load balancing and horizontal scaling is essential for building highly available, scalable applications that can handle increasing traffic.

---

## Load Balancing Basics

### Load Balancing Algorithms

```python
"""
Common Load Balancing Algorithms:

1. Round Robin: Distribute requests sequentially
2. Least Connections: Send to server with fewest connections
3. Least Response Time: Send to fastest server
4. IP Hash: Same client always goes to same server
5. Weighted Round Robin: Distribute based on server capacity
6. Random: Random server selection
"""

# Simple Round Robin implementation
class RoundRobinLoadBalancer:
    """Round robin load balancer"""

    def __init__(self, servers: list):
        self.servers = servers
        self.current = 0

    def get_server(self):
        """Get next server in rotation"""
        server = self.servers[self.current]
        self.current = (self.current + 1) % len(self.servers)
        return server

# Usage
servers = [
    "http://server1.example.com",
    "http://server2.example.com",
    "http://server3.example.com"
]

lb = RoundRobinLoadBalancer(servers)

# Distribute requests
for _ in range(10):
    server = lb.get_server()
    print(f"Sending request to: {server}")

# Weighted Round Robin
class WeightedRoundRobinLoadBalancer:
    """Weighted round robin - more to powerful servers"""

    def __init__(self, servers: dict):
        """
        servers = {
            'server1': 5,  # Weight 5
            'server2': 3,  # Weight 3
            'server3': 2   # Weight 2
        }
        """
        self.weighted_servers = []

        for server, weight in servers.items():
            self.weighted_servers.extend([server] * weight)

        self.current = 0

    def get_server(self):
        """Get next server based on weight"""
        server = self.weighted_servers[self.current]
        self.current = (self.current + 1) % len(self.weighted_servers)
        return server

# Least Connections
class LeastConnectionsLoadBalancer:
    """Send to server with fewest active connections"""

    def __init__(self, servers: list):
        self.servers = {server: 0 for server in servers}

    def get_server(self):
        """Get server with least connections"""
        return min(self.servers, key=self.servers.get)

    def connection_started(self, server: str):
        """Track connection start"""
        self.servers[server] += 1

    def connection_ended(self, server: str):
        """Track connection end"""
        self.servers[server] -= 1

# IP Hash
import hashlib

class IPHashLoadBalancer:
    """Consistent server for same client IP"""

    def __init__(self, servers: list):
        self.servers = servers

    def get_server(self, client_ip: str):
        """Get server based on client IP hash"""
        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        index = hash_value % len(self.servers)
        return self.servers[index]

# Usage with FastAPI
from fastapi import FastAPI, Request
import httpx

app = FastAPI()
lb = RoundRobinLoadBalancer(servers)

@app.middleware("http")
async def load_balance_middleware(request: Request, call_next):
    """Load balance requests to backend servers"""

    # Get next server
    backend_server = lb.get_server()

    # Forward request
    async with httpx.AsyncClient() as client:
        backend_url = f"{backend_server}{request.url.path}"

        response = await client.request(
            method=request.method,
            url=backend_url,
            headers=dict(request.headers),
            content=await request.body()
        )

        return Response(
            content=response.content,
            status_code=response.status_code,
            headers=dict(response.headers)
        )

    return await call_next(request)
```

---

## Session Persistence (Sticky Sessions)

### Managing User Sessions

```python
# Sticky sessions with Redis
import redis
import hashlib

class StickySessionLoadBalancer:
    """Load balancer with session affinity"""

    def __init__(self, servers: list, redis_client: redis.Redis):
        self.servers = servers
        self.redis = redis_client
        self.lb = RoundRobinLoadBalancer(servers)

    def get_server(self, session_id: str):
        """Get server for session (sticky)"""
        # Check if session has existing server
        cache_key = f"session:{session_id}"
        cached_server = self.redis.get(cache_key)

        if cached_server:
            return cached_server.decode()

        # Assign new server
        server = self.lb.get_server()

        # Store for 1 hour
        self.redis.setex(cache_key, 3600, server)

        return server

# FastAPI with sticky sessions
from fastapi import Cookie, Response

@app.get("/api/data")
async def get_data(session_id: str = Cookie(None), response: Response = None):
    """Handle request with sticky session"""

    # Create session if needed
    if not session_id:
        import uuid
        session_id = str(uuid.uuid4())
        response.set_cookie("session_id", session_id)

    # Get server for this session
    server = sticky_lb.get_server(session_id)

    # Forward to assigned server
    return {"server": server, "session": session_id}

# Session replication across servers
class SessionReplicator:
    """Replicate sessions across servers"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def save_session(self, session_id: str, data: dict):
        """Save session (available to all servers)"""
        key = f"session_data:{session_id}"
        self.redis.setex(key, 3600, json.dumps(data))

    def get_session(self, session_id: str):
        """Get session from any server"""
        key = f"session_data:{session_id}"
        data = self.redis.get(key)
        return json.loads(data) if data else None
```

---

## Health Checks

### Server Health Monitoring

```python
from fastapi import FastAPI
import time
import psutil

app = FastAPI()

class HealthChecker:
    """Check server health"""

    def __init__(self):
        self.start_time = time.time()

    def get_health_status(self):
        """Get comprehensive health status"""
        return {
            "status": "healthy",
            "uptime": time.time() - self.start_time,
            "cpu_percent": psutil.cpu_percent(),
            "memory_percent": psutil.virtual_memory().percent,
            "disk_percent": psutil.disk_usage('/').percent,
            "timestamp": time.time()
        }

    def is_healthy(self):
        """Check if server is healthy"""
        try:
            # Check CPU
            if psutil.cpu_percent() > 90:
                return False, "CPU overloaded"

            # Check memory
            if psutil.virtual_memory().percent > 90:
                return False, "Memory overloaded"

            # Check disk
            if psutil.disk_usage('/').percent > 90:
                return False, "Disk full"

            return True, "Healthy"

        except Exception as e:
            return False, f"Health check failed: {e}"

health_checker = HealthChecker()

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    is_healthy, message = health_checker.is_healthy()

    status_code = 200 if is_healthy else 503

    return JSONResponse(
        status_code=status_code,
        content={
            "status": "healthy" if is_healthy else "unhealthy",
            "message": message,
            "details": health_checker.get_health_status()
        }
    )

@app.get("/ready")
async def readiness_check():
    """Readiness check - can server handle traffic"""
    try:
        # Check database connection
        db.execute("SELECT 1")

        # Check Redis connection
        redis_client.ping()

        return {"status": "ready"}

    except Exception as e:
        return JSONResponse(
            status_code=503,
            content={"status": "not ready", "error": str(e)}
        )

@app.get("/live")
async def liveness_check():
    """Liveness check - is server alive"""
    return {"status": "alive"}

# Load balancer health checking
class HealthAwareLoadBalancer:
    """Load balancer that checks server health"""

    def __init__(self, servers: list):
        self.servers = servers
        self.healthy_servers = set(servers)
        self.lb = RoundRobinLoadBalancer(list(self.healthy_servers))

        # Start health checking
        self.start_health_checks()

    async def check_server_health(self, server: str):
        """Check if server is healthy"""
        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    f"{server}/health",
                    timeout=5.0
                )
                return response.status_code == 200
        except Exception:
            return False

    async def update_healthy_servers(self):
        """Update list of healthy servers"""
        tasks = [
            self.check_server_health(server)
            for server in self.servers
        ]

        results = await asyncio.gather(*tasks)

        self.healthy_servers = {
            server for server, is_healthy in zip(self.servers, results)
            if is_healthy
        }

        # Update load balancer
        if self.healthy_servers:
            self.lb = RoundRobinLoadBalancer(list(self.healthy_servers))

    def start_health_checks(self):
        """Start periodic health checks"""
        async def health_check_loop():
            while True:
                await self.update_healthy_servers()
                await asyncio.sleep(10)  # Check every 10 seconds

        asyncio.create_task(health_check_loop())

    def get_server(self):
        """Get healthy server"""
        if not self.healthy_servers:
            raise Exception("No healthy servers available")

        return self.lb.get_server()
```

---

## Auto-Scaling

### Dynamic Scaling Based on Metrics

```python
import boto3
from dataclasses import dataclass

@dataclass
class ScalingMetrics:
    """Metrics for scaling decisions"""
    cpu_percent: float
    memory_percent: float
    request_rate: float
    response_time: float

class AutoScaler:
    """Auto-scaling controller"""

    def __init__(self, min_instances=2, max_instances=10):
        self.min_instances = min_instances
        self.max_instances = max_instances
        self.current_instances = min_instances

    def should_scale_up(self, metrics: ScalingMetrics) -> bool:
        """Determine if should scale up"""
        return (
            metrics.cpu_percent > 70 or
            metrics.memory_percent > 80 or
            metrics.response_time > 2.0  # 2 seconds
        )

    def should_scale_down(self, metrics: ScalingMetrics) -> bool:
        """Determine if should scale down"""
        return (
            metrics.cpu_percent < 30 and
            metrics.memory_percent < 50 and
            metrics.response_time < 0.5  # 500ms
        )

    def scale_up(self):
        """Scale up instances"""
        if self.current_instances < self.max_instances:
            new_count = min(
                self.current_instances + 1,
                self.max_instances
            )

            print(f"Scaling up: {self.current_instances} -> {new_count}")
            self.current_instances = new_count

            # Trigger actual scaling (e.g., AWS Auto Scaling)
            return True

        return False

    def scale_down(self):
        """Scale down instances"""
        if self.current_instances > self.min_instances:
            new_count = max(
                self.current_instances - 1,
                self.min_instances
            )

            print(f"Scaling down: {self.current_instances} -> {new_count}")
            self.current_instances = new_count

            return True

        return False

    async def monitor_and_scale(self):
        """Monitor metrics and scale"""
        while True:
            # Collect metrics
            metrics = ScalingMetrics(
                cpu_percent=psutil.cpu_percent(interval=1),
                memory_percent=psutil.virtual_memory().percent,
                request_rate=self.get_request_rate(),
                response_time=self.get_avg_response_time()
            )

            # Make scaling decision
            if self.should_scale_up(metrics):
                self.scale_up()
            elif self.should_scale_down(metrics):
                self.scale_down()

            await asyncio.sleep(60)  # Check every minute

# AWS Auto Scaling integration
class AWSAutoScaler:
    """Auto-scaling with AWS"""

    def __init__(self, auto_scaling_group_name: str):
        self.asg_name = auto_scaling_group_name
        self.client = boto3.client('autoscaling')

    def scale_up(self, desired_capacity: int):
        """Scale up AWS Auto Scaling Group"""
        self.client.set_desired_capacity(
            AutoScalingGroupName=self.asg_name,
            DesiredCapacity=desired_capacity,
            HonorCooldown=True
        )

    def get_current_capacity(self):
        """Get current instance count"""
        response = self.client.describe_auto_scaling_groups(
            AutoScalingGroupNames=[self.asg_name]
        )

        asg = response['AutoScalingGroups'][0]
        return {
            'desired': asg['DesiredCapacity'],
            'min': asg['MinSize'],
            'max': asg['MaxSize'],
            'current': len(asg['Instances'])
        }
```

---

## Database Scaling

### Read Replicas and Sharding

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
import random

class DatabasePool:
    """Database connection pool with read replicas"""

    def __init__(self, master_url: str, replica_urls: list):
        # Master for writes
        self.master_engine = create_engine(master_url)
        self.MasterSession = sessionmaker(bind=self.master_engine)

        # Replicas for reads
        self.replica_engines = [
            create_engine(url) for url in replica_urls
        ]
        self.ReplicaSessions = [
            sessionmaker(bind=engine)
            for engine in self.replica_engines
        ]

    def get_master_session(self):
        """Get master session (for writes)"""
        return self.MasterSession()

    def get_replica_session(self):
        """Get replica session (for reads)"""
        # Random replica
        session_maker = random.choice(self.ReplicaSessions)
        return session_maker()

# Usage
db_pool = DatabasePool(
    master_url="postgresql://master:5432/db",
    replica_urls=[
        "postgresql://replica1:5432/db",
        "postgresql://replica2:5432/db"
    ]
)

# Write operations
def create_user(user_data):
    """Create user (use master)"""
    session = db_pool.get_master_session()
    try:
        user = User(**user_data)
        session.add(user)
        session.commit()
        return user
    finally:
        session.close()

# Read operations
def get_users():
    """Get users (use replica)"""
    session = db_pool.get_replica_session()
    try:
        users = session.query(User).all()
        return users
    finally:
        session.close()

# Sharding by user ID
class ShardedDatabase:
    """Database sharding"""

    def __init__(self, shard_urls: dict):
        """
        shard_urls = {
            0: "postgresql://shard0:5432/db",
            1: "postgresql://shard1:5432/db",
            2: "postgresql://shard2:5432/db"
        }
        """
        self.shards = {
            shard_id: create_engine(url)
            for shard_id, url in shard_urls.items()
        }

        self.shard_count = len(self.shards)

    def get_shard_id(self, user_id: int) -> int:
        """Determine shard for user"""
        return user_id % self.shard_count

    def get_session(self, user_id: int):
        """Get session for user's shard"""
        shard_id = self.get_shard_id(user_id)
        engine = self.shards[shard_id]
        Session = sessionmaker(bind=engine)
        return Session()

# Usage
sharded_db = ShardedDatabase({
    0: "postgresql://shard0:5432/db",
    1: "postgresql://shard1:5432/db",
    2: "postgresql://shard2:5432/db"
})

def get_user_posts(user_id: int):
    """Get posts from correct shard"""
    session = sharded_db.get_session(user_id)
    try:
        posts = session.query(Post)\
            .filter(Post.user_id == user_id)\
            .all()
        return posts
    finally:
        session.close()
```

---

## Service Discovery

### Dynamic Service Registration

```python
import consul

class ServiceRegistry:
    """Service discovery with Consul"""

    def __init__(self, consul_host='localhost', consul_port=8500):
        self.consul = consul.Consul(host=consul_host, port=consul_port)

    def register_service(
        self,
        service_name: str,
        service_id: str,
        host: str,
        port: int,
        tags: list = None
    ):
        """Register service"""
        self.consul.agent.service.register(
            name=service_name,
            service_id=service_id,
            address=host,
            port=port,
            tags=tags or [],
            check=consul.Check.http(
                f"http://{host}:{port}/health",
                interval="10s",
                timeout="5s"
            )
        )

    def deregister_service(self, service_id: str):
        """Deregister service"""
        self.consul.agent.service.deregister(service_id)

    def discover_service(self, service_name: str):
        """Discover service instances"""
        _, services = self.consul.health.service(
            service_name,
            passing=True  # Only healthy instances
        )

        return [
            {
                'id': service['Service']['ID'],
                'address': service['Service']['Address'],
                'port': service['Service']['Port']
            }
            for service in services
        ]

    def get_service_url(self, service_name: str):
        """Get random healthy service URL"""
        services = self.discover_service(service_name)

        if not services:
            raise Exception(f"No healthy instances of {service_name}")

        service = random.choice(services)
        return f"http://{service['address']}:{service['port']}"

# Register on startup
@app.on_event("startup")
async def startup():
    """Register service on startup"""
    registry = ServiceRegistry()

    registry.register_service(
        service_name="api-service",
        service_id="api-service-1",
        host="10.0.1.5",
        port=8000,
        tags=["api", "v1"]
    )

# Deregister on shutdown
@app.on_event("shutdown")
async def shutdown():
    """Deregister service on shutdown"""
    registry = ServiceRegistry()
    registry.deregister_service("api-service-1")
```

---

## Best Practices

### ✅ Do's:

1. **Use health checks** for load balancers
2. **Implement graceful shutdown** - drain connections
3. **Use sticky sessions** when needed (stateful)
4. **Monitor metrics** for auto-scaling
5. **Set min/max instances** for cost control
6. **Use read replicas** for read-heavy loads
7. **Implement circuit breakers** for failures
8. **Use service discovery** for dynamic infrastructure
9. **Test failover** scenarios regularly
10. **Log server selection** for debugging

### ❌ Don'ts:

1. **Don't use sticky sessions** unnecessarily (stateless better)
2. **Don't ignore unhealthy** servers
3. **Don't scale too quickly** (causes instability)
4. **Don't forget connection draining**
5. **Don't put all eggs** in one zone (distribute)
6. **Don't skip readiness** checks
7. **Don't forget session replication**

---

## Interview Questions

### Q1: Explain different load balancing algorithms.

**Answer**:

- **Round Robin**: Sequential distribution
- **Least Connections**: Fewest active connections
- **IP Hash**: Same client → same server
- **Weighted**: Based on server capacity
- **Least Response Time**: Fastest server
  Choose based on application needs.

### Q2: What are sticky sessions and when to use them?

**Answer**: Route same user to same server:

- **Use when**: Stateful apps, in-memory sessions
- **Avoid when**: Possible (use shared state instead)
- **Implement with**: Session ID hashing, cookies
- **Trade-off**: Harder to scale, uneven load

### Q3: Explain horizontal vs vertical scaling.

**Answer**:

- **Horizontal**: Add more servers (scale out)
- **Vertical**: Add more resources (scale up)
- **Horizontal benefits**: No downtime, unlimited, fault tolerant
- **Vertical limits**: Hardware limits, downtime, cost
  Always prefer horizontal scaling.

### Q4: How do you handle database scaling?

**Answer**:

- **Read replicas**: Offload reads to replicas
- **Sharding**: Partition data across databases
- **Connection pooling**: Reuse connections
- **Caching**: Reduce database load
- **Async operations**: Non-blocking queries

### Q5: What is service discovery?

**Answer**: Dynamically find service instances:

- **Need**: Services have dynamic IPs
- **Tools**: Consul, etcd, Eureka
- **Features**: Health checks, registration, lookup
- **Benefits**: Auto-scaling, failover
  Essential for microservices.

---

## Summary

Scaling essentials:

- **Load balancing**: Distribute traffic across servers
- **Algorithms**: Round robin, least connections, IP hash
- **Health checks**: Monitor server health
- **Auto-scaling**: Scale based on metrics
- **Database scaling**: Read replicas, sharding
- **Service discovery**: Dynamic instance lookup
- **Sticky sessions**: When needed for state

Scale horizontally for reliability! ⚖️
