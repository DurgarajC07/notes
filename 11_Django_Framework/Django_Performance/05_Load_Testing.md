# Django Load Testing & Performance Profiling

## 📖 Concept Explanation

Load testing measures how your Django application performs under expected and peak traffic. Profiling identifies performance bottlenecks in code and database queries.

### Key Metrics

```
Response Time: How long requests take
Throughput: Requests per second (RPS)
Error Rate: % of failed requests
Resource Usage: CPU, memory, database connections
Concurrency: Simultaneous users
```

**Why It Matters**:

- Identify bottlenecks before production
- Validate performance requirements (e.g., < 200ms response)
- Plan infrastructure scaling
- Prevent outages under high load

## 🧠 Load Testing with Locust

Locust is a Python-based load testing tool with a simple, code-driven approach.

### 1. Installation

```bash
pip install locust
```

### 2. Basic Locustfile

```python
# locustfile.py
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(1, 3)  # Wait 1-3 seconds between tasks

    def on_start(self):
        """Called when a user starts"""
        # Login once
        self.client.post("/api/token/", json={
            "username": "testuser",
            "password": "testpass123"
        })

    @task(3)  # Weight: 3 (executed 3x more often than task(1))
    def view_posts(self):
        """Browse posts"""
        self.client.get("/api/posts/")

    @task(2)
    def view_post_detail(self):
        """View specific post"""
        self.client.get("/api/posts/1/")

    @task(1)
    def create_post(self):
        """Create a new post"""
        self.client.post("/api/posts/", json={
            "title": "Load Test Post",
            "content": "Testing under load"
        })

    @task(1)
    def search_posts(self):
        """Search functionality"""
        self.client.get("/api/posts/?search=django")
```

### 3. Running Locust

```bash
# Start Locust web interface
locust -f locustfile.py --host=http://localhost:8000

# Or headless mode
locust -f locustfile.py --host=http://localhost:8000 \
    --users 100 \
    --spawn-rate 10 \
    --run-time 5m \
    --html report.html

# Visit http://localhost:8089 to configure test
```

### 4. Advanced Locustfile with Authentication

```python
# locustfile.py
from locust import HttpUser, task, between, SequentialTaskSet
import random

class UserBehavior(SequentialTaskSet):
    """Sequential tasks simulating real user flow"""

    @task
    def login(self):
        response = self.client.post("/api/token/", json={
            "username": "testuser",
            "password": "testpass123"
        })
        if response.status_code == 200:
            token = response.json()['access']
            self.user.token = token
            self.user.headers = {'Authorization': f'Bearer {token}'}

    @task
    def browse_posts(self):
        self.client.get("/api/posts/", headers=self.user.headers)

    @task
    def view_random_post(self):
        post_id = random.randint(1, 1000)
        self.client.get(f"/api/posts/{post_id}/", headers=self.user.headers)

    @task
    def create_post(self):
        self.client.post("/api/posts/",
            json={
                "title": f"Post {random.randint(1, 10000)}",
                "content": "Load test content"
            },
            headers=self.user.headers
        )

    @task
    def logout(self):
        self.interrupt()  # End task sequence

class WebsiteUser(HttpUser):
    tasks = [UserBehavior]
    wait_time = between(1, 5)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.token = None
        self.headers = {}
```

### 5. Custom Load Shapes

```python
# locustfile.py
from locust import HttpUser, task, LoadTestShape

class StagesShape(LoadTestShape):
    """
    Simulate realistic traffic patterns:
    - Ramp up gradually
    - Hold at peak
    - Ramp down
    """

    stages = [
        {"duration": 60, "users": 10, "spawn_rate": 2},   # Ramp up to 10 users
        {"duration": 120, "users": 50, "spawn_rate": 5},  # Ramp up to 50
        {"duration": 180, "users": 100, "spawn_rate": 10}, # Ramp up to 100
        {"duration": 300, "users": 100, "spawn_rate": 0},  # Hold at 100 for 2 min
        {"duration": 360, "users": 50, "spawn_rate": 10},  # Ramp down to 50
        {"duration": 420, "users": 0, "spawn_rate": 10},   # Ramp down to 0
    ]

    def tick(self):
        run_time = self.get_run_time()

        for stage in self.stages:
            if run_time < stage["duration"]:
                return (stage["users"], stage["spawn_rate"])

        return None  # Test complete

class MyUser(HttpUser):
    wait_time = between(1, 3)

    @task
    def my_task(self):
        self.client.get("/api/posts/")
```

## 🎯 Profiling with django-silk

django-silk provides detailed profiling of Django requests, SQL queries, and function calls.

### 1. Installation

```python
# Install
pip install django-silk

# settings.py
INSTALLED_APPS = [
    ...
    'silk',
]

MIDDLEWARE = [
    'silk.middleware.SilkyMiddleware',
    ...
]

# urls.py
urlpatterns = [
    path('silk/', include('silk.urls', namespace='silk')),
    ...
]

# Run migrations
python manage.py migrate
```

### 2. Accessing Silk Dashboard

```python
# Visit: http://localhost:8000/silk/

# Silk shows:
# - All requests with timing
# - SQL queries per request
# - Most time-consuming queries
# - Request/response profiling
# - Function call profiling
```

### 3. Profiling Specific Code

```python
# views.py
from silk.profiling.profiler import silk_profile

@silk_profile(name='Get User Posts')
def get_user_posts(user_id):
    """This function will appear in Silk profiler"""
    user = User.objects.select_related('profile').get(id=user_id)
    posts = Post.objects.filter(author=user).select_related('category').prefetch_related('tags')
    return posts

# Decorator for views
from silk.profiling.profiler import silk_profile

@silk_profile()
def post_list(request):
    posts = Post.objects.all()
    return render(request, 'posts/list.html', {'posts': posts})
```

### 4. Silk Configuration

```python
# settings.py

# Limit stored requests (prevent database bloat)
SILKY_MAX_RECORDED_REQUESTS = 10000

# Only profile specific paths
SILKY_INTERCEPT_PERCENT = 100  # Profile 100% of requests
SILKY_INTERCEPT_FUNC = lambda request: request.path.startswith('/api/')

# Exclude media/static files
SILKY_PYTHON_PROFILER = True
SILKY_PYTHON_PROFILER_BINARY = True

# Authentication
SILKY_AUTHENTICATION = True
SILKY_AUTHORISATION = True
```

## 🔍 Django Debug Toolbar

### Setup

```python
# Install
pip install django-debug-toolbar

# settings.py
INSTALLED_APPS = [
    ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    ...
]

INTERNAL_IPS = ['127.0.0.1']

# urls.py
from django.conf import settings

if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [
        path('__debug__/', include(debug_toolbar.urls)),
    ] + urlpatterns
```

### Using Debug Toolbar

```python
# Toolbar shows:
# - SQL queries (with EXPLAIN)
# - Query duplicates (N+1 detection)
# - Template rendering time
# - Cache hits/misses
# - Signal timing
# - Request/response headers

# Optimize based on toolbar findings
# Example: Toolbar shows 101 queries

# Before optimization
def post_list(request):
    posts = Post.objects.all()  # N+1 problem
    return render(request, 'posts/list.html', {'posts': posts})

# After optimization (1 query)
def post_list(request):
    posts = Post.objects.select_related('author').prefetch_related('tags').all()
    return render(request, 'posts/list.html', {'posts': posts})
```

## ⚡ Application Performance Monitoring (APM)

### 1. New Relic

```python
# Install
pip install newrelic

# Generate config
newrelic-admin generate-config LICENSE_KEY newrelic.ini

# Run Django with New Relic
NEW_RELIC_CONFIG_FILE=newrelic.ini newrelic-admin run-program python manage.py runserver

# Or add to Django
# settings.py
import newrelic.agent
newrelic.agent.initialize('newrelic.ini')

# Custom instrumentation
from newrelic.agent import function_trace

@function_trace()
def expensive_function():
    # This function will be traced
    pass
```

### 2. Sentry Performance

```python
# Install
pip install sentry-sdk

# settings.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn="your-dsn-here",
    integrations=[DjangoIntegration()],
    traces_sample_rate=1.0,  # 100% of transactions
    profiles_sample_rate=1.0,  # 100% profiling
)

# Sentry automatically tracks:
# - Slow database queries
# - Slow HTTP requests
# - Error rates
# - Custom transactions
```

### 3. Custom Metrics

```python
# middleware.py
import time
from django.utils.deprecation import MiddlewareMixin
import logging

logger = logging.getLogger(__name__)

class PerformanceMiddleware(MiddlewareMixin):
    """Log slow requests"""

    def process_request(self, request):
        request._start_time = time.time()

    def process_response(self, request, response):
        if hasattr(request, '_start_time'):
            duration = time.time() - request._start_time

            if duration > 1.0:  # Log requests slower than 1s
                logger.warning(
                    f"Slow request: {request.method} {request.path} "
                    f"took {duration:.2f}s"
                )

        return response
```

## ✅ Best Practices

### 1. Establish Performance Baselines

```python
# tests/performance_tests.py
from django.test import TestCase, Client
import time

class PerformanceTest(TestCase):
    def setUp(self):
        # Create test data
        for i in range(100):
            Post.objects.create(title=f'Post {i}', content='Test')

    def test_post_list_performance(self):
        """Post list should respond within 200ms"""
        client = Client()

        start = time.time()
        response = client.get('/api/posts/')
        duration = time.time() - start

        self.assertEqual(response.status_code, 200)
        self.assertLess(duration, 0.2, f"Response took {duration:.3f}s")

    def test_post_list_query_count(self):
        """Post list should use ≤ 3 queries"""
        from django.test.utils import override_settings
        from django.db import connection, reset_queries

        with override_settings(DEBUG=True):
            reset_queries()

            client = Client()
            response = client.get('/api/posts/')

            query_count = len(connection.queries)
            self.assertLessEqual(
                query_count, 3,
                f"Used {query_count} queries (expected ≤ 3)"
            )
```

### 2. Load Test Environment Setup

```python
# Separate load test settings
# settings/loadtest.py
from .base import *

DEBUG = False
ALLOWED_HOSTS = ['loadtest.example.com']

# Use production-like database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'loadtest_db',
        'CONN_MAX_AGE': 600,
    }
}

# Disable debug toolbar
INSTALLED_APPS = [app for app in INSTALLED_APPS if app != 'debug_toolbar']
MIDDLEWARE = [mw for mw in MIDDLEWARE if 'debug_toolbar' not in mw]

# Enable caching
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}
```

### 3. Gradual Load Increase

```bash
# Start with low load
locust -f locustfile.py --users 10 --spawn-rate 1

# Gradually increase
locust -f locustfile.py --users 50 --spawn-rate 5
locust -f locustfile.py --users 100 --spawn-rate 10
locust -f locustfile.py --users 500 --spawn-rate 50

# Find breaking point
locust -f locustfile.py --users 1000 --spawn-rate 100
```

### 4. Monitor Server Resources

```bash
# Monitor during load test
# CPU usage
top

# Memory usage
free -h

# Database connections (PostgreSQL)
psql -c "SELECT count(*) FROM pg_stat_activity;"

# Django logs
tail -f /var/log/django/debug.log

# Gunicorn workers
ps aux | grep gunicorn
```

## ❌ Common Mistakes

### 1. Testing Without Production Settings

```python
# ❌ Bad: Testing with DEBUG=True
# DEBUG mode is much slower (tracks queries, templates, etc.)

# ✅ Good: Test with production-like settings
DEBUG = False
ALLOWED_HOSTS = ['*']
LOGGING = {'disable_existing_loggers': False}
```

### 2. Insufficient Test Data

```python
# ❌ Bad: Testing with 10 records
# Performance issues only appear with realistic data

# ✅ Good: Test with production-like data volume
for i in range(10000):
    Post.objects.create(title=f'Post {i}')
```

### 3. Not Testing Edge Cases

```python
# Test scenarios:
# - Empty database
# - Large database (1M+ records)
# - Complex queries (joins, aggregations)
# - File uploads
# - API pagination (page 1 vs page 1000)
# - Concurrent writes
# - Cache cold vs warm
```

## ❓ Interview Questions

### Q1: How do you identify performance bottlenecks in Django?

**Answer**:

1. **django-debug-toolbar**: Identify N+1 queries
2. **django-silk**: Profile requests and SQL
3. **Locust**: Load test under realistic traffic
4. **APM tools**: New Relic, Sentry for production monitoring
5. **Database EXPLAIN**: Analyze query plans

### Q2: What's the difference between load testing and stress testing?

**Answer**:

- **Load testing**: Test with expected traffic (e.g., 100 concurrent users)
- **Stress testing**: Test beyond capacity to find breaking point (e.g., 10,000 users)
- **Spike testing**: Sudden traffic increase
- **Endurance testing**: Sustained load over time (memory leaks)

### Q3: What metrics matter most in load testing?

**Answer**:

1. **Response time**: 50th, 95th, 99th percentile (not just average)
2. **Throughput**: Requests per second
3. **Error rate**: % of failed requests
4. **Resource usage**: CPU, memory, DB connections
5. **Scalability**: Performance as load increases

## 📚 Summary

**Key Takeaways**:

1. Use Locust for load testing (Python-based, flexible)
2. Use django-silk for detailed request profiling
3. Use django-debug-toolbar during development
4. Test with production-like settings (DEBUG=False)
5. Establish performance baselines (e.g., < 200ms)
6. Gradually increase load to find limits
7. Monitor server resources during tests
8. Test with realistic data volumes
9. Use APM tools in production (New Relic, Sentry)
10. Profile before optimizing (measure, don't guess)

Load testing and profiling are essential for production-ready Django applications!
