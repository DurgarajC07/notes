# FastAPI Core: Background Tasks

## 📖 Introduction

FastAPI's **Background Tasks** allow you to run operations after returning a response, ideal for non-critical tasks like sending emails, processing data, or updating caches. Background tasks run in the same process as the request handler.

## 🔑 Basic Background Tasks

### Simple Background Task

```python
from fastapi import FastAPI, BackgroundTasks
import time

app = FastAPI()

def write_log(message: str):
    """
    Background task function

    - Runs after response is sent
    - Can be regular function (not async)
    """
    with open("log.txt", "a") as f:
        f.write(f"{message}\n")

@app.post("/send-notification")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    """
    Add background task

    - Response returned immediately
    - Task runs after response sent
    """
    background_tasks.add_task(write_log, f"Notification sent to {email}")

    return {"message": "Notification will be sent in background"}
```

### Async Background Task

```python
import asyncio
import httpx

async def send_email(email: str, message: str):
    """
    Async background task

    - Use async for I/O operations
    - More efficient for network/database calls
    """
    # Simulate email sending
    await asyncio.sleep(1)

    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://api.example.com/send-email",
            json={"email": email, "message": message}
        )

    print(f"Email sent to {email}: {response.status_code}")

@app.post("/register")
async def register_user(email: str, password: str, background_tasks: BackgroundTasks):
    """
    Register user with background email

    - User creation returns immediately
    - Welcome email sent in background
    """
    # Create user (simulated)
    user_id = 123

    # Add background task
    background_tasks.add_task(
        send_email,
        email=email,
        message=f"Welcome! Your account has been created."
    )

    return {"user_id": user_id, "email": email}
```

## 📧 Practical Use Cases

### Send Welcome Email

```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    """User registration"""
    username: str
    email: EmailStr
    password: str

async def send_welcome_email(email: str, username: str):
    """Send welcome email"""
    await asyncio.sleep(2)  # Simulate email service delay

    print(f"Welcome email sent to {email}")

    # In production: use email service (SendGrid, AWS SES, etc.)
    # await email_service.send(
    #     to=email,
    #     subject="Welcome!",
    #     body=f"Hello {username}, welcome to our service!"
    # )

@app.post("/users")
async def create_user(user: UserCreate, background_tasks: BackgroundTasks):
    """
    Create user with background email

    - User created immediately
    - Welcome email sent after response
    """
    # Save user to database (simulated)
    user_id = 123

    # Add background task
    background_tasks.add_task(send_welcome_email, user.email, user.username)

    return {"user_id": user_id, "email": user.email}
```

### Update Cache After Write

```python
import redis.asyncio as redis

redis_client = redis.from_url("redis://localhost")

async def update_cache(key: str, value: str):
    """Update Redis cache"""
    await redis_client.set(key, value, ex=3600)  # 1 hour TTL
    print(f"Cache updated: {key}")

@app.post("/posts")
async def create_post(title: str, content: str, background_tasks: BackgroundTasks):
    """
    Create post with cache update

    - Post created and response returned
    - Cache updated in background
    """
    # Save post to database (simulated)
    post_id = 456

    # Update cache in background
    cache_key = f"post:{post_id}"
    cache_value = f"{title}|{content}"

    background_tasks.add_task(update_cache, cache_key, cache_value)

    return {"post_id": post_id, "title": title}
```

### Generate Report

```python
async def generate_report(user_id: int, report_type: str):
    """
    Generate report (expensive operation)

    - Runs in background
    - Notifies user when complete
    """
    # Simulate report generation
    await asyncio.sleep(5)

    # Generate report
    report_data = {"user_id": user_id, "type": report_type, "status": "completed"}

    # Save to storage
    report_filename = f"reports/user_{user_id}_{report_type}.pdf"

    # Notify user (send email with download link)
    print(f"Report generated: {report_filename}")

@app.post("/reports")
async def request_report(user_id: int, report_type: str, background_tasks: BackgroundTasks):
    """
    Request report generation

    - Returns immediately with request ID
    - Report generated in background
    """
    background_tasks.add_task(generate_report, user_id, report_type)

    return {
        "message": "Report generation started",
        "user_id": user_id,
        "report_type": report_type
    }
```

### Log Request Analytics

```python
from datetime import datetime

async def log_analytics(endpoint: str, user_id: int, duration_ms: float):
    """
    Log request analytics

    - Logs to analytics database
    - Doesn't block response
    """
    analytics_data = {
        "endpoint": endpoint,
        "user_id": user_id,
        "duration_ms": duration_ms,
        "timestamp": datetime.utcnow()
    }

    # Save to analytics database (simulated)
    await asyncio.sleep(0.1)
    print(f"Analytics logged: {analytics_data}")

@app.get("/items/{item_id}")
async def get_item(item_id: int, user_id: int, background_tasks: BackgroundTasks):
    """
    Get item with analytics

    - Returns item immediately
    - Logs analytics in background
    """
    start_time = time.time()

    # Get item (simulated)
    item = {"id": item_id, "name": "Item"}

    # Calculate duration
    duration_ms = (time.time() - start_time) * 1000

    # Log in background
    background_tasks.add_task(
        log_analytics,
        endpoint=f"/items/{item_id}",
        user_id=user_id,
        duration_ms=duration_ms
    )

    return item
```

## 🔄 Multiple Background Tasks

### Chain Multiple Tasks

```python
async def task_1(data: str):
    """First task"""
    await asyncio.sleep(1)
    print(f"Task 1 completed: {data}")

async def task_2(data: str):
    """Second task"""
    await asyncio.sleep(1)
    print(f"Task 2 completed: {data}")

async def task_3(data: str):
    """Third task"""
    await asyncio.sleep(1)
    print(f"Task 3 completed: {data}")

@app.post("/process")
async def process_data(data: str, background_tasks: BackgroundTasks):
    """
    Multiple background tasks

    - Tasks run sequentially in order added
    - All complete before process ends
    """
    background_tasks.add_task(task_1, data)
    background_tasks.add_task(task_2, data)
    background_tasks.add_task(task_3, data)

    return {"message": "Processing started", "data": data}
```

### Parallel vs Sequential Execution

```python
import asyncio

async def task_parallel_1():
    """Task 1 (parallel)"""
    await asyncio.sleep(2)
    print("Parallel task 1 done")

async def task_parallel_2():
    """Task 2 (parallel)"""
    await asyncio.sleep(2)
    print("Parallel task 2 done")

async def run_parallel_tasks():
    """
    Run tasks in parallel

    - Use asyncio.gather for concurrent execution
    - Total time = max(task times), not sum
    """
    await asyncio.gather(
        task_parallel_1(),
        task_parallel_2()
    )
    print("All parallel tasks done")

@app.post("/parallel")
async def parallel_processing(background_tasks: BackgroundTasks):
    """
    Parallel background tasks

    - Tasks run concurrently
    - Faster than sequential
    """
    background_tasks.add_task(run_parallel_tasks)

    return {"message": "Parallel processing started"}
```

## 💾 Background Tasks with Database

### Update Database in Background

```python
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import Depends

async def update_user_stats(user_id: int, db: AsyncSession):
    """
    Update user statistics

    - Expensive aggregation query
    - Runs in background after response
    """
    # Calculate stats (simulated)
    await asyncio.sleep(2)

    # Update database
    # await db.execute(
    #     update(UserStats)
    #     .where(UserStats.user_id == user_id)
    #     .values(post_count=10, last_active=datetime.utcnow())
    # )
    # await db.commit()

    print(f"User {user_id} stats updated")

@app.post("/posts")
async def create_post(
    title: str,
    user_id: int,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db)
):
    """
    Create post with background stats update

    - Post created immediately
    - Stats updated in background
    """
    # Create post (simulated)
    post_id = 789

    # Update stats in background
    background_tasks.add_task(update_user_stats, user_id, db)

    return {"post_id": post_id, "title": title}
```

## ⚠️ Limitations & Alternatives

### When NOT to Use Background Tasks

```python
"""
DON'T use BackgroundTasks for:

1. Long-running tasks (> 30 seconds)
   - Use Celery, RQ, or Dramatiq instead

2. Tasks requiring retries on failure
   - Use message queue (RabbitMQ, Redis)

3. Tasks needing monitoring/status updates
   - Use task queue with result backend

4. CPU-intensive operations
   - Use multiprocessing or separate worker processes

5. Tasks requiring distributed execution
   - Use Celery with multiple workers

BackgroundTasks are good for:
- Quick operations (< 10 seconds)
- Non-critical tasks (failures OK)
- Simple, synchronous execution
- Tasks that don't need status tracking
"""
```

### Alternative: Celery

```python
from celery import Celery

# Celery app
celery_app = Celery(
    "tasks",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/0"
)

@celery_app.task
def send_email_celery(email: str, message: str):
    """
    Celery task

    - Runs in separate worker process
    - Supports retries, scheduling, monitoring
    - Better for production
    """
    # Send email
    time.sleep(2)
    print(f"Email sent to {email}")

@app.post("/register-celery")
async def register_user_celery(email: str, password: str):
    """
    Use Celery for background task

    - More robust than BackgroundTasks
    - Survives server restarts
    - Distributed execution
    """
    # Create user
    user_id = 123

    # Queue task
    send_email_celery.delay(email, "Welcome!")

    return {"user_id": user_id, "email": email}
```

## 🧪 Testing Background Tasks

### Test Background Tasks

```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_background_task():
    """
    Test endpoint with background task

    - Background tasks execute during test
    - Check side effects (logs, database, etc.)
    """
    response = client.post(
        "/send-notification",
        params={"email": "test@example.com"}
    )

    assert response.status_code == 200

    # Check background task side effect
    with open("log.txt", "r") as f:
        logs = f.read()
        assert "test@example.com" in logs
```

## ❓ Interview Questions

### Q1: What are FastAPI Background Tasks?

**Answer**:

- Functions that run **after response is sent**
- Run in **same process** as request handler
- Good for **quick, non-critical tasks**
- Use **BackgroundTasks.add_task()** to add tasks

### Q2: When to use Background Tasks vs Celery?

**Answer**:

- **Background Tasks**: Quick tasks (<10s), simple, no retries needed
- **Celery**: Long-running, distributed, needs retries/monitoring
- **Celery better for**: Production, critical tasks, scalability

### Q3: Do background tasks run sequentially or in parallel?

**Answer**:

- **Sequential by default** (in order added)
- **Can run parallel** with asyncio.gather() inside task
- Each task completes before next starts (unless using gather)

### Q4: What happens if background task fails?

**Answer**:

- **Exception is logged** but not raised to client
- **Response already sent** before task runs
- **No automatic retries** (unlike Celery)
- Use **try/except** in task for error handling

## 📚 Summary

**Key Takeaways**:

1. **BackgroundTasks** run after response sent
2. **add_task()** queues function for execution
3. **async or sync** functions both supported
4. **Sequential execution** by default
5. **Good for**: emails, logging, cache updates, analytics
6. **Not for**: long tasks (>30s), distributed work, retries
7. **Use Celery** for production-grade task queues
8. **Same process** as request (not separate worker)
9. **No status tracking** or result retrieval
10. **Simple and effective** for quick background operations

Background tasks make FastAPI responses faster without blocking!
