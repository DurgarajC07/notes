# 📊 Profiling and Monitoring FastAPI Applications

## Overview

Profiling and monitoring help identify performance bottlenecks, track resource usage, and ensure application health in production. Understanding what to measure and how to measure it is critical for maintaining high-performance applications.

---

## Application Profiling

### 1. **cProfile - Built-in Python Profiler**

```python
import cProfile
import pstats
from io import StringIO
from fastapi import FastAPI, Request
import time

app = FastAPI()

@app.middleware("http")
async def profile_request(request: Request, call_next):
    """Profile each request"""
    # Create profiler
    profiler = cProfile.Profile()
    profiler.enable()

    # Process request
    response = await call_next(request)

    # Stop profiling
    profiler.disable()

    # Print stats
    stream = StringIO()
    stats = pstats.Stats(profiler, stream=stream)
    stats.sort_stats('cumulative')
    stats.print_stats(20)  # Top 20 functions

    print(f"\n{request.url.path} Profile:")
    print(stream.getvalue())

    return response

@app.get("/slow-endpoint")
async def slow_endpoint():
    """Endpoint to profile"""
    time.sleep(0.1)
    result = sum(i * i for i in range(10000))
    return {"result": result}
```

### 2. **line_profiler - Line-by-Line Profiling**

```python
from line_profiler import LineProfiler

def profile_function(func):
    """Decorator to profile function line by line"""
    def wrapper(*args, **kwargs):
        profiler = LineProfiler()
        profiler.add_function(func)
        profiler.enable()

        result = func(*args, **kwargs)

        profiler.disable()
        profiler.print_stats()

        return result
    return wrapper

@app.get("/compute")
@profile_function
def compute_intensive():
    """Function to profile line by line"""
    data = []
    for i in range(1000):
        data.append(i * i)

    total = sum(data)
    avg = total / len(data)

    return {"total": total, "average": avg}
```

### 3. **py-spy - Sampling Profiler**

```python
# Run py-spy from command line:
# py-spy top --pid <pid>
# py-spy record -o profile.svg --pid <pid>
# py-spy dump --pid <pid>

# Or integrate in code for specific sections
import subprocess

@app.get("/profile/start")
async def start_profiling():
    """Start py-spy profiling"""
    import os
    pid = os.getpid()

    # Start py-spy in background
    process = subprocess.Popen([
        "py-spy", "record",
        "-o", f"profile_{pid}.svg",
        "--pid", str(pid),
        "--duration", "60"
    ])

    return {"message": "Profiling started", "pid": pid}
```

---

## Request/Response Timing

### 1. **Timing Middleware**

```python
import time
from fastapi import Request, Response
import logging

logger = logging.getLogger(__name__)

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    """Add timing information to response"""
    start_time = time.time()

    response = await call_next(request)

    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)

    # Log slow requests
    if process_time > 1.0:  # Slower than 1 second
        logger.warning(
            f"Slow request: {request.method} {request.url.path} "
            f"took {process_time:.2f}s"
        )

    return response

@app.middleware("http")
async def detailed_timing(request: Request, call_next):
    """Detailed timing breakdown"""
    timings = {
        "start": time.time()
    }

    # Time request processing
    response = await call_next(request)

    timings["end"] = time.time()
    timings["total"] = timings["end"] - timings["start"]

    # Add to response header as JSON
    response.headers["Server-Timing"] = (
        f'total;dur={timings["total"]*1000:.2f}'
    )

    return response
```

### 2. **Endpoint-Specific Timing**

```python
from functools import wraps
import asyncio

def time_async(func):
    """Decorator to time async functions"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.time()
        result = await func(*args, **kwargs)
        duration = time.time() - start

        logger.info(f"{func.__name__} took {duration:.3f}s")
        return result
    return wrapper

@app.get("/data/expensive")
@time_async
async def expensive_operation():
    """Timed endpoint"""
    await asyncio.sleep(0.5)

    # Expensive computation
    result = sum(i * i for i in range(100000))

    return {"result": result}

# Timing context manager
from contextlib import asynccontextmanager

@asynccontextmanager
async def timer(name: str):
    """Context manager for timing code blocks"""
    start = time.time()
    yield
    duration = time.time() - start
    logger.info(f"{name} took {duration:.3f}s")

@app.get("/data/detailed-timing")
async def detailed_timing_endpoint():
    """Endpoint with detailed timing"""
    results = {}

    async with timer("Database query"):
        await asyncio.sleep(0.2)
        results["db"] = "data"

    async with timer("External API call"):
        await asyncio.sleep(0.3)
        results["api"] = "data"

    async with timer("Processing"):
        await asyncio.sleep(0.1)
        results["processed"] = True

    return results
```

---

## Memory Profiling

### 1. **memory_profiler**

```python
from memory_profiler import profile

@app.get("/memory-test")
@profile
async def memory_intensive():
    """Profile memory usage"""
    # Create large list
    big_list = [i for i in range(1000000)]

    # Create dictionary
    big_dict = {i: i * i for i in range(100000)}

    # Process data
    result = sum(big_list)

    return {"result": result}

# Run with: python -m memory_profiler app.py
```

### 2. **tracemalloc - Built-in Memory Tracking**

```python
import tracemalloc
from fastapi import BackgroundTasks

class MemoryTracker:
    """Track memory usage"""

    def __init__(self):
        self.snapshots = []

    def start(self):
        """Start tracking memory"""
        tracemalloc.start()

    def snapshot(self, name: str):
        """Take memory snapshot"""
        snapshot = tracemalloc.take_snapshot()
        self.snapshots.append((name, snapshot))

    def compare(self, snapshot1_idx: int, snapshot2_idx: int):
        """Compare two snapshots"""
        name1, snap1 = self.snapshots[snapshot1_idx]
        name2, snap2 = self.snapshots[snapshot2_idx]

        top_stats = snap2.compare_to(snap1, 'lineno')

        print(f"\nMemory changes from {name1} to {name2}:")
        for stat in top_stats[:10]:
            print(stat)

    def current_memory(self):
        """Get current memory usage"""
        current, peak = tracemalloc.get_traced_memory()
        return {
            "current_mb": current / 1024 / 1024,
            "peak_mb": peak / 1024 / 1024
        }

memory_tracker = MemoryTracker()
memory_tracker.start()

@app.get("/memory/snapshot")
async def take_snapshot(name: str = "snapshot"):
    """Take memory snapshot"""
    memory_tracker.snapshot(name)
    current = memory_tracker.current_memory()
    return {
        "message": f"Snapshot '{name}' taken",
        "current_memory_mb": current["current_mb"],
        "peak_memory_mb": current["peak_mb"]
    }

@app.get("/memory/current")
async def get_current_memory():
    """Get current memory usage"""
    return memory_tracker.current_memory()
```

### 3. **objgraph - Object Tracking**

```python
import objgraph
from typing import List

@app.get("/memory/objects")
async def get_object_stats():
    """Get object type statistics"""
    # Most common objects
    stats = objgraph.most_common_types(limit=20)

    return {
        "most_common": [
            {"type": name, "count": count}
            for name, count in stats
        ]
    }

@app.get("/memory/growth")
async def show_growth():
    """Show object growth since last call"""
    growth = objgraph.growth(limit=10)

    return {
        "growth": [
            {
                "type": name,
                "count": count,
                "delta": delta
            }
            for name, count, delta in growth
        ]
    }
```

---

## Application Metrics

### 1. **Prometheus Integration**

```python
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from prometheus_client import CONTENT_TYPE_LATEST
from fastapi.responses import Response

# Define metrics
request_count = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint']
)

active_requests = Gauge(
    'http_requests_active',
    'Active HTTP requests',
    ['method', 'endpoint']
)

@app.middleware("http")
async def prometheus_metrics(request: Request, call_next):
    """Collect Prometheus metrics"""
    method = request.method
    path = request.url.path

    # Increment active requests
    active_requests.labels(method=method, endpoint=path).inc()

    # Time request
    with request_duration.labels(method=method, endpoint=path).time():
        response = await call_next(request)

    # Decrement active requests
    active_requests.labels(method=method, endpoint=path).dec()

    # Count request
    request_count.labels(
        method=method,
        endpoint=path,
        status=response.status_code
    ).inc()

    return response

@app.get("/metrics")
async def metrics():
    """Expose metrics for Prometheus"""
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )
```

### 2. **Custom Business Metrics**

```python
from prometheus_client import Counter, Histogram, Summary

# Business metrics
user_registrations = Counter(
    'user_registrations_total',
    'Total user registrations'
)

order_value = Histogram(
    'order_value_dollars',
    'Order value in dollars',
    buckets=[10, 50, 100, 500, 1000, 5000]
)

api_response_size = Summary(
    'api_response_size_bytes',
    'API response size in bytes',
    ['endpoint']
)

@app.post("/users/register")
async def register_user(user: UserCreate):
    """Register user with metrics"""
    # Create user
    new_user = create_user(user)

    # Track metric
    user_registrations.inc()

    return new_user

@app.post("/orders")
async def create_order(order: OrderCreate):
    """Create order with metrics"""
    # Create order
    new_order = create_order_in_db(order)

    # Track order value
    order_value.observe(new_order.total_amount)

    return new_order

@app.get("/api/data")
async def get_data():
    """Endpoint tracking response size"""
    data = fetch_data()
    response_json = json.dumps(data)

    # Track response size
    api_response_size.labels(endpoint="/api/data").observe(
        len(response_json)
    )

    return data
```

---

## Logging and Tracing

### 1. **Structured Logging**

```python
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    """Format logs as JSON"""

    def format(self, record):
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno
        }

        # Add extra fields
        if hasattr(record, 'request_id'):
            log_data['request_id'] = record.request_id

        if hasattr(record, 'user_id'):
            log_data['user_id'] = record.user_id

        return json.dumps(log_data)

# Configure logging
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())

logger = logging.getLogger(__name__)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    """Add request context to logs"""
    # Generate request ID
    request_id = str(uuid.uuid4())

    # Add to request state
    request.state.request_id = request_id

    # Log request
    logger.info(
        f"Request started: {request.method} {request.url.path}",
        extra={"request_id": request_id}
    )

    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time

    # Log response
    logger.info(
        f"Request completed: {request.method} {request.url.path} "
        f"status={response.status_code} duration={duration:.3f}s",
        extra={"request_id": request_id}
    )

    return response
```

### 2. **OpenTelemetry Tracing**

```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# Configure OpenTelemetry
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Configure Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831
)

span_processor = BatchSpanProcessor(jaeger_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

@app.get("/traced-endpoint")
async def traced_endpoint():
    """Endpoint with custom spans"""
    with tracer.start_as_current_span("database_query") as span:
        span.set_attribute("query.type", "select")
        await asyncio.sleep(0.1)  # Simulate DB query

    with tracer.start_as_current_span("external_api") as span:
        span.set_attribute("api.endpoint", "/external")
        await asyncio.sleep(0.2)  # Simulate API call

    return {"message": "Traced operation complete"}
```

---

## Health Checks and Monitoring

### 1. **Health Check Endpoints**

```python
from typing import Dict
import psutil

@app.get("/health")
async def health_check():
    """Basic health check"""
    return {"status": "healthy"}

@app.get("/health/detailed")
async def detailed_health_check(db: Session = Depends(get_db)):
    """Detailed health check with dependencies"""
    health_status = {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "checks": {}
    }

    # Check database
    try:
        db.execute(text("SELECT 1"))
        health_status["checks"]["database"] = "healthy"
    except Exception as e:
        health_status["checks"]["database"] = f"unhealthy: {str(e)}"
        health_status["status"] = "unhealthy"

    # Check Redis
    try:
        redis_client.ping()
        health_status["checks"]["redis"] = "healthy"
    except Exception as e:
        health_status["checks"]["redis"] = f"unhealthy: {str(e)}"
        health_status["status"] = "unhealthy"

    # System metrics
    health_status["checks"]["system"] = {
        "cpu_percent": psutil.cpu_percent(),
        "memory_percent": psutil.virtual_memory().percent,
        "disk_percent": psutil.disk_usage('/').percent
    }

    return health_status

@app.get("/health/readiness")
async def readiness_check():
    """Kubernetes readiness probe"""
    # Check if application is ready to serve traffic
    if not application_ready():
        return Response(status_code=503)

    return {"status": "ready"}

@app.get("/health/liveness")
async def liveness_check():
    """Kubernetes liveness probe"""
    # Check if application is alive
    return {"status": "alive"}
```

### 2. **Performance Monitoring Dashboard**

```python
from fastapi.responses import HTMLResponse

@app.get("/dashboard", response_class=HTMLResponse)
async def monitoring_dashboard():
    """Simple monitoring dashboard"""
    # Get metrics
    memory = memory_tracker.current_memory()
    cpu_percent = psutil.cpu_percent(interval=1)

    # Get request stats
    info = expensive_computation.cache_info()

    html_content = f"""
    <html>
        <head>
            <title>Monitoring Dashboard</title>
            <meta http-equiv="refresh" content="5">
            <style>
                body {{ font-family: Arial, sans-serif; margin: 20px; }}
                .metric {{ padding: 10px; margin: 10px 0; background: #f0f0f0; }}
                .healthy {{ color: green; }}
                .warning {{ color: orange; }}
                .critical {{ color: red; }}
            </style>
        </head>
        <body>
            <h1>FastAPI Monitoring Dashboard</h1>

            <div class="metric">
                <h3>System Metrics</h3>
                <p>CPU Usage: {cpu_percent}%</p>
                <p>Memory: {memory['current_mb']:.2f} MB (Peak: {memory['peak_mb']:.2f} MB)</p>
            </div>

            <div class="metric">
                <h3>Cache Statistics</h3>
                <p>Hits: {info.hits}</p>
                <p>Misses: {info.misses}</p>
                <p>Hit Rate: {info.hits/(info.hits+info.misses)*100:.1f}%</p>
            </div>
        </body>
    </html>
    """

    return HTMLResponse(content=html_content)
```

---

## Best Practices

### ✅ Do's:

1. **Profile in production-like environment**
2. **Monitor key business metrics**
3. **Set up alerts** for anomalies
4. **Use structured logging**
5. **Implement health checks**
6. **Track request latency**
7. **Monitor resource usage** (CPU, memory, DB connections)
8. **Use distributed tracing** for microservices

### ❌ Don'ts:

1. **Don't profile only in development**
2. **Don't ignore slow requests**
3. **Don't log sensitive data**
4. **Don't skip monitoring in production**
5. **Don't alert on every minor issue**

---

## Interview Questions

### Q1: What metrics should you monitor in production?

**Answer**: Key metrics include:

- Request rate and latency (p50, p95, p99)
- Error rate by endpoint and status code
- Resource usage (CPU, memory, connections)
- Business metrics (registrations, orders, revenue)
- Database query performance
- Cache hit rates

### Q2: What's the difference between profiling and monitoring?

**Answer**:

- **Profiling**: Detailed analysis of code execution (development/debugging)
- **Monitoring**: Continuous tracking of metrics (production)
  Profiling helps find bottlenecks, monitoring tracks ongoing health.

### Q3: How do you debug performance issues in production?

**Answer**:

1. Check monitoring dashboards for anomalies
2. Review application logs for errors
3. Analyze distributed traces
4. Check database query logs
5. Profile specific endpoints with py-spy
6. Review recent deployments

### Q4: What is distributed tracing and when is it useful?

**Answer**: Distributed tracing tracks requests across multiple services, showing the complete path and timing. Useful for:

- Microservices architectures
- Identifying slow services
- Understanding request flow
- Debugging cross-service issues

### Q5: How do you set up effective alerts?

**Answer**:

- Alert on symptoms, not causes
- Use appropriate thresholds (error rate > 5%)
- Avoid alert fatigue
- Include context in notifications
- Set up escalation policies
- Test alerts regularly

---

## Summary

Effective profiling and monitoring include:

- **Code profiling** with cProfile, py-spy
- **Request timing** with middleware
- **Memory tracking** with tracemalloc
- **Metrics collection** with Prometheus
- **Distributed tracing** with OpenTelemetry
- **Health checks** for reliability

Proper monitoring ensures high-performance, reliable applications! 📊
