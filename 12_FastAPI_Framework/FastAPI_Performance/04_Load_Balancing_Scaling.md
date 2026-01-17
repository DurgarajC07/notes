# ⚙️ Load Balancing and Scaling FastAPI

## Overview

Load balancing and scaling strategies ensure your FastAPI application can handle increasing traffic while maintaining performance and reliability. Understanding horizontal vs vertical scaling and various load balancing approaches is essential for production systems.

---

## Scaling Strategies

### 1. **Vertical Scaling (Scale Up)**

Increasing resources of a single instance:

```python
# Uvicorn with optimized settings
import uvicorn

if __name__ == "__main__":
    uvicorn.run(
        "app:app",
        host="0.0.0.0",
        port=8000,
        workers=4,  # Number of worker processes
        loop="uvloop",  # Faster event loop
        http="httptools",  # Faster HTTP parser
        limit_concurrency=1000,  # Max concurrent connections
        limit_max_requests=10000,  # Restart worker after N requests
        timeout_keep_alive=5,  # Keep-alive timeout
    )
```

**Pros:**

- Simpler to implement
- No code changes needed
- Lower complexity

**Cons:**

- Limited by hardware
- Single point of failure
- Diminishing returns

### 2. **Horizontal Scaling (Scale Out)**

Running multiple instances:

```bash
# Run multiple Uvicorn instances
uvicorn app:app --host 0.0.0.0 --port 8001 &
uvicorn app:app --host 0.0.0.0 --port 8002 &
uvicorn app:app --host 0.0.0.0 --port 8003 &
uvicorn app:app --host 0.0.0.0 --port 8004 &
```

**Pros:**

- Better fault tolerance
- Easier to scale
- Cost-effective with cloud

**Cons:**

- Requires load balancer
- State management complexity
- More operational overhead

---

## Load Balancing with Nginx

### 1. **Basic Round-Robin Load Balancing**

```nginx
# nginx.conf

upstream fastapi_backend {
    # Round-robin (default)
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
    server 127.0.0.1:8004;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://fastapi_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
}
```

### 2. **Least Connections Load Balancing**

```nginx
upstream fastapi_backend {
    least_conn;  # Route to server with fewest active connections

    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
    server 127.0.0.1:8004;
}
```

### 3. **IP Hash (Sticky Sessions)**

```nginx
upstream fastapi_backend {
    ip_hash;  # Same client always goes to same server

    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
    server 127.0.0.1:8004;
}
```

### 4. **Weighted Load Balancing**

```nginx
upstream fastapi_backend {
    server 127.0.0.1:8001 weight=3;  # Gets 3x more requests
    server 127.0.0.1:8002 weight=2;  # Gets 2x more requests
    server 127.0.0.1:8003 weight=1;  # Gets 1x requests
    server 127.0.0.1:8004 weight=1;
}
```

### 5. **Health Checks**

```nginx
upstream fastapi_backend {
    server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8002 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8003 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8004 max_fails=3 fail_timeout=30s;

    # Backup server
    server 127.0.0.1:8005 backup;
}
```

---

## Gunicorn with Uvicorn Workers

### 1. **Basic Configuration**

```python
# gunicorn_config.py

import multiprocessing

# Server socket
bind = "0.0.0.0:8000"
backlog = 2048

# Worker processes
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn.workers.UvicornWorker"
worker_connections = 1000
max_requests = 10000  # Restart worker after N requests
max_requests_jitter = 1000  # Add randomness to prevent all workers restarting at once
timeout = 30
keepalive = 5

# Logging
accesslog = "-"
errorlog = "-"
loglevel = "info"
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" %(D)s'

# Process naming
proc_name = "fastapi_app"

# Server mechanics
daemon = False
pidfile = "/tmp/gunicorn.pid"
user = None
group = None
tmp_upload_dir = None

# SSL (if needed)
# keyfile = "/path/to/key.pem"
# certfile = "/path/to/cert.pem"
```

```bash
# Run with Gunicorn
gunicorn app:app -c gunicorn_config.py
```

### 2. **Worker Management**

```python
# Worker lifecycle hooks

def on_starting(server):
    """Called before master process starts"""
    print("Starting Gunicorn server")

def on_reload(server):
    """Called on reload"""
    print("Reloading configuration")

def when_ready(server):
    """Called when server is ready"""
    print(f"Server ready with {server.cfg.workers} workers")

def pre_fork(server, worker):
    """Called before worker is forked"""
    print(f"Pre-fork worker {worker.pid}")

def post_fork(server, worker):
    """Called after worker is forked"""
    print(f"Post-fork worker {worker.pid}")

def pre_exec(server):
    """Called before new master process"""
    print("Pre-exec")

def worker_int(worker):
    """Called when worker receives SIGINT/SIGQUIT"""
    print(f"Worker {worker.pid} interrupted")

def worker_abort(worker):
    """Called when worker receives SIGABRT"""
    print(f"Worker {worker.pid} aborted")
```

---

## Container-Based Scaling

### 1. **Docker Setup**

```dockerfile
# Dockerfile

FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Run with Gunicorn
CMD ["gunicorn", "app:app", "-c", "gunicorn_config.py"]

EXPOSE 8000
```

### 2. **Docker Compose for Multiple Instances**

```yaml
# docker-compose.yml

version: "3.8"

services:
  fastapi1:
    build: .
    ports:
      - "8001:8000"
    environment:
      - WORKER_ID=1
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  fastapi2:
    build: .
    ports:
      - "8002:8000"
    environment:
      - WORKER_ID=2
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  fastapi3:
    build: .
    ports:
      - "8003:8000"
    environment:
      - WORKER_ID=3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - fastapi1
      - fastapi2
      - fastapi3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:latest
    ports:
      - "6379:6379"

  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

---

## Kubernetes Deployment

### 1. **Deployment Configuration**

```yaml
# deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
  labels:
    app: fastapi
spec:
  replicas: 3 # Number of pods
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
        - name: fastapi
          image: your-registry/fastapi-app:latest
          ports:
            - containerPort: 8000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/liveness
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/readiness
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
```

### 2. **Service Configuration**

```yaml
# service.yaml

apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  selector:
    app: fastapi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer
```

### 3. **Horizontal Pod Autoscaler**

```yaml
# hpa.yaml

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastapi-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fastapi-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # Wait 5 minutes before scaling down
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
        - type: Pods
          value: 2
          periodSeconds: 30
```

---

## Session Management in Scaled Environment

### 1. **Stateless Sessions with JWT**

```python
# No server-side storage needed
from fastapi import Depends, HTTPException
from jose import jwt, JWTError

def get_current_user(token: str = Depends(oauth2_scheme)):
    """Get user from JWT token (stateless)"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id = payload.get("sub")
        return user_id
    except JWTError:
        raise HTTPException(401, "Invalid token")

@app.get("/profile")
async def get_profile(user_id: str = Depends(get_current_user)):
    """Stateless endpoint - works across all instances"""
    return {"user_id": user_id}
```

### 2. **Centralized Session Storage (Redis)**

```python
import redis
import json

redis_client = redis.Redis(host="redis", port=6379)

class SessionManager:
    """Centralized session management"""

    def create_session(self, user_id: int, data: dict) -> str:
        """Create new session"""
        session_id = str(uuid.uuid4())
        session_data = {
            "user_id": user_id,
            **data
        }

        # Store in Redis (1 hour TTL)
        redis_client.setex(
            f"session:{session_id}",
            3600,
            json.dumps(session_data)
        )

        return session_id

    def get_session(self, session_id: str) -> Optional[dict]:
        """Get session data"""
        data = redis_client.get(f"session:{session_id}")
        if data:
            return json.loads(data)
        return None

    def delete_session(self, session_id: str):
        """Delete session"""
        redis_client.delete(f"session:{session_id}")

session_manager = SessionManager()

@app.post("/login")
async def login(credentials: LoginCredentials):
    """Login with session creation"""
    user = authenticate_user(credentials)

    # Create session
    session_id = session_manager.create_session(
        user.id,
        {"email": user.email}
    )

    return {"session_id": session_id}

def get_session_user(session_id: str = Cookie(None)):
    """Dependency to get user from session"""
    if not session_id:
        raise HTTPException(401, "Not authenticated")

    session = session_manager.get_session(session_id)
    if not session:
        raise HTTPException(401, "Invalid session")

    return session["user_id"]

@app.get("/profile")
async def get_profile(user_id: int = Depends(get_session_user)):
    """Endpoint using centralized sessions"""
    return {"user_id": user_id}
```

---

## Rate Limiting in Distributed Environment

### 1. **Redis-Based Rate Limiting**

```python
from fastapi import Request
import time

class DistributedRateLimiter:
    """Rate limiter using Redis"""

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

        # Use sorted set with scores as timestamps
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

rate_limiter = DistributedRateLimiter(redis_client)

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    """Apply rate limiting"""
    # Get client identifier
    client_id = request.client.host

    # Check rate limit
    if not rate_limiter.is_allowed(
        f"ratelimit:{client_id}",
        max_requests=100,
        window_seconds=60
    ):
        return Response(
            content="Rate limit exceeded",
            status_code=429,
            headers={"Retry-After": "60"}
        )

    return await call_next(request)
```

---

## Best Practices

### ✅ Do's:

1. **Use horizontal scaling** for better resilience
2. **Implement health checks** for load balancers
3. **Use connection pooling** for databases
4. **Centralize session storage** (Redis)
5. **Monitor all instances** individually
6. **Use gradual rollouts** for deployments
7. **Implement circuit breakers** for dependencies
8. **Use auto-scaling** based on metrics

### ❌ Don'ts:

1. **Don't store state** in application memory
2. **Don't ignore failed health checks**
3. **Don't deploy to all instances** at once
4. **Don't use IP hash** unless necessary
5. **Don't skip monitoring** individual instances
6. **Don't over-provision** resources

---

## Interview Questions

### Q1: What's the difference between horizontal and vertical scaling?

**Answer**:

- **Vertical**: Increase resources (CPU, RAM) of single instance. Limited by hardware.
- **Horizontal**: Add more instances. Better for high availability and unlimited scaling.
  Horizontal is preferred for modern cloud applications.

### Q2: How do you handle sessions in a load-balanced environment?

**Answer**:

- Use stateless authentication (JWT)
- Centralized session storage (Redis)
- Sticky sessions (IP hash) - last resort
  Stateless JWT is best for scalability.

### Q3: What load balancing algorithms are available?

**Answer**:

- **Round-robin**: Equal distribution
- **Least connections**: Route to server with fewest connections
- **IP hash**: Sticky sessions based on client IP
- **Weighted**: Distribute based on server capacity

### Q4: How does Kubernetes auto-scaling work?

**Answer**: HPA (Horizontal Pod Autoscaler) monitors metrics (CPU, memory, custom) and scales pods based on thresholds. For example, if CPU > 70%, add pods. If CPU < 30%, remove pods. Includes stabilization windows to prevent flapping.

### Q5: What are health checks and why are they important?

**Answer**: Health checks verify instance status:

- **Liveness**: Is the app running?
- **Readiness**: Can it serve traffic?
  Load balancers use health checks to route traffic only to healthy instances, improving reliability.

---

## Summary

Effective scaling and load balancing include:

- **Horizontal scaling** with multiple instances
- **Load balancing** with Nginx/cloud load balancers
- **Container orchestration** with Kubernetes
- **Stateless design** for scalability
- **Centralized storage** for shared state
- **Auto-scaling** based on metrics

Proper scaling ensures your FastAPI application handles any load! ⚙️
