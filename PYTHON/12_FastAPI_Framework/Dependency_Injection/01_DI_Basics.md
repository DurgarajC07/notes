# 💉 Dependency Injection Basics in FastAPI

## Overview

FastAPI has a powerful **Dependency Injection (DI)** system that allows you to declare dependencies for your path operations. Dependencies are automatically resolved, executed, and injected into your functions.

---

## What is Dependency Injection?

**Dependency Injection** is a design pattern where components receive their dependencies from external sources rather than creating them internally.

### Benefits:

- ✅ **Reusability**: Share common logic across endpoints
- ✅ **Testability**: Easy to mock dependencies
- ✅ **Maintainability**: Centralized logic
- ✅ **Type Safety**: Full type checking support
- ✅ **Automatic Validation**: FastAPI validates dependency inputs

---

## Basic Dependency

### 1. **Simple Function Dependency**

```python
from fastapi import FastAPI, Depends
from typing import Optional

app = FastAPI()

# Dependency function
def get_query_params(
    q: Optional[str] = None,
    skip: int = 0,
    limit: int = 100
):
    return {"q": q, "skip": skip, "limit": limit}

# Use dependency
@app.get("/items/")
async def read_items(params: dict = Depends(get_query_params)):
    return params

# Request: GET /items/?q=search&skip=10&limit=50
# Response: {"q": "search", "skip": 10, "limit": 50}
```

### 2. **Dependency with Validation**

```python
from fastapi import HTTPException

def pagination_params(
    skip: int = 0,
    limit: int = 10
):
    if limit > 100:
        raise HTTPException(400, "Limit cannot exceed 100")
    if skip < 0:
        raise HTTPException(400, "Skip cannot be negative")

    return {"skip": skip, "limit": limit}

@app.get("/users/")
async def get_users(params: dict = Depends(pagination_params)):
    # params is validated automatically
    return {"pagination": params, "users": [...]}
```

---

## Class-Based Dependencies

### 1. **Using Classes as Dependencies**

```python
from typing import Optional

class CommonQueryParams:
    def __init__(
        self,
        q: Optional[str] = None,
        skip: int = 0,
        limit: int = 100
    ):
        self.q = q
        self.skip = skip
        self.limit = limit

@app.get("/items/")
async def read_items(commons: CommonQueryParams = Depends()):
    # Depends() without arguments uses the type hint
    return {
        "query": commons.q,
        "skip": commons.skip,
        "limit": commons.limit
    }

# Equivalent to:
@app.get("/items/")
async def read_items(commons: CommonQueryParams = Depends(CommonQueryParams)):
    return {"query": commons.q}
```

### 2. **Class with Additional Logic**

```python
class UserQueryParams:
    def __init__(
        self,
        name: Optional[str] = None,
        age_min: Optional[int] = None,
        age_max: Optional[int] = None
    ):
        self.name = name
        self.age_min = age_min
        self.age_max = age_max

    def build_filter(self) -> dict:
        """Build database filter from params"""
        filters = {}
        if self.name:
            filters["name__icontains"] = self.name
        if self.age_min:
            filters["age__gte"] = self.age_min
        if self.age_max:
            filters["age__lte"] = self.age_max
        return filters

@app.get("/users/search")
async def search_users(params: UserQueryParams = Depends()):
    filters = params.build_filter()
    # Use filters in database query
    return {"filters": filters, "results": [...]}
```

---

## Nested Dependencies

### 1. **Dependencies with Dependencies**

```python
def get_current_token(token: str = Header(...)):
    """Extract token from header"""
    return token

def verify_token(token: str = Depends(get_current_token)):
    """Verify token and return user"""
    if token != "secret-token":
        raise HTTPException(401, "Invalid token")
    return {"user_id": 123, "username": "john"}

def get_current_user(user: dict = Depends(verify_token)):
    """Get current user from verified token"""
    # Additional processing
    return user

@app.get("/me")
async def read_current_user(user: dict = Depends(get_current_user)):
    return user

# Dependency chain:
# get_current_user -> verify_token -> get_current_token
```

### 2. **Multiple Dependencies**

```python
def get_db():
    """Database session dependency"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_current_user(
    token: str = Header(...),
    db = Depends(get_db)
):
    """Get user from token using database"""
    user = db.query(User).filter(User.token == token).first()
    if not user:
        raise HTTPException(401, "Invalid credentials")
    return user

@app.get("/profile")
async def get_profile(
    user = Depends(get_current_user),
    db = Depends(get_db)
):
    # Both user and db are available
    profile = db.query(Profile).filter(Profile.user_id == user.id).first()
    return profile
```

---

## Dependencies with `yield`

### 1. **Setup and Teardown**

```python
def get_db_session():
    """Database session with automatic cleanup"""
    session = SessionLocal()
    try:
        yield session  # Control returns here
    finally:
        session.close()  # Cleanup after endpoint completes

@app.post("/users/")
async def create_user(
    user: UserCreate,
    db = Depends(get_db_session)
):
    db_user = User(**user.dict())
    db.add(db_user)
    db.commit()
    return db_user
# db.close() is called automatically after this
```

### 2. **Exception Handling in Dependencies**

```python
from contextlib import asynccontextmanager

async def get_redis_connection():
    """Redis connection with error handling"""
    redis = await aioredis.create_redis_pool("redis://localhost")
    try:
        yield redis
    except Exception as e:
        logger.error(f"Redis error: {e}")
        raise
    finally:
        redis.close()
        await redis.wait_closed()

@app.get("/cache/{key}")
async def get_cached(
    key: str,
    redis = Depends(get_redis_connection)
):
    value = await redis.get(key)
    return {"key": key, "value": value}
```

---

## Dependency Scopes

### 1. **Path-Level Dependencies**

```python
# Apply to single endpoint
@app.get("/items/", dependencies=[Depends(verify_token)])
async def read_items():
    # verify_token runs, but result not used
    return {"items": [...]}
```

### 2. **Router-Level Dependencies**

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(verify_admin)]  # Applied to all routes
)

@router.get("/users")
async def get_users():
    # verify_admin runs automatically
    return {"users": [...]}

@router.delete("/users/{user_id}")
async def delete_user(user_id: int):
    # verify_admin runs automatically
    return {"deleted": user_id}

app.include_router(router)
```

### 3. **Application-Level Dependencies**

```python
async def log_request():
    """Log every request"""
    logger.info("Request received")

app = FastAPI(dependencies=[Depends(log_request)])

# log_request runs for every endpoint
@app.get("/")
async def root():
    return {"message": "Hello"}
```

---

## Dependency Caching

### 1. **Shared Dependencies**

```python
# This dependency will be called once per request
def expensive_operation(x: int = Query(...)):
    print("Computing expensive operation...")
    time.sleep(1)  # Expensive computation
    return x * 2

@app.get("/endpoint1")
async def endpoint1(
    result1: int = Depends(expensive_operation),
    result2: int = Depends(expensive_operation)  # Same function
):
    # expensive_operation called only ONCE
    return {"result1": result1, "result2": result2}
```

### 2. **Disable Caching**

```python
def get_timestamp():
    return datetime.now()

# Without use_cache, called every time
@app.get("/time")
async def get_time(
    time1: datetime = Depends(get_timestamp, use_cache=False),
    time2: datetime = Depends(get_timestamp, use_cache=False)
):
    # Different timestamps
    return {"time1": time1, "time2": time2}
```

---

## Dependency Override (Testing)

### 1. **Override for Testing**

```python
# Production dependency
def get_db():
    db = ProductionDB()
    try:
        yield db
    finally:
        db.close()

# Test override
def override_get_db():
    db = TestDB()
    try:
        yield db
    finally:
        db.close()

# In tests
app.dependency_overrides[get_db] = override_get_db

# Now all endpoints use TestDB
client = TestClient(app)
response = client.get("/users/")

# Clean up
app.dependency_overrides.clear()
```

### 2. **Override with Mock**

```python
from unittest.mock import Mock

# Override with mock
def mock_get_current_user():
    return {"user_id": 1, "username": "testuser"}

app.dependency_overrides[get_current_user] = mock_get_current_user

# Test authenticated endpoint
response = client.get("/me")
assert response.json()["username"] == "testuser"
```

---

## Sub-Dependencies

### 1. **Reusable Sub-Dependencies**

```python
def get_user_agent(user_agent: str = Header(None)):
    """Extract user agent"""
    return user_agent

def get_client_ip(x_forwarded_for: str = Header(None)):
    """Extract client IP"""
    return x_forwarded_for

def get_request_info(
    user_agent: str = Depends(get_user_agent),
    client_ip: str = Depends(get_client_ip)
):
    """Combine multiple sub-dependencies"""
    return {
        "user_agent": user_agent,
        "client_ip": client_ip
    }

@app.get("/info")
async def get_info(info: dict = Depends(get_request_info)):
    return info
```

---

## Practical Examples

### 1. **Authentication Dependency**

```python
from jose import JWTError, jwt
from fastapi.security import HTTPBearer

security = HTTPBearer()

def get_current_user(
    credentials = Depends(security)
):
    """Validate JWT token and return user"""
    token = credentials.credentials

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(401, "Invalid token")
    except JWTError:
        raise HTTPException(401, "Invalid token")

    return {"user_id": user_id}

@app.get("/protected")
async def protected_route(user: dict = Depends(get_current_user)):
    return {"message": f"Hello user {user['user_id']}"}
```

### 2. **Permission Checking**

```python
def require_permission(permission: str):
    """Factory function to create permission dependency"""
    def check_permission(user: dict = Depends(get_current_user)):
        if permission not in user.get("permissions", []):
            raise HTTPException(403, f"Permission denied: {permission}")
        return user
    return check_permission

# Usage
@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    user: dict = Depends(require_permission("delete_user"))
):
    # user has "delete_user" permission
    return {"deleted": user_id}

@app.post("/posts/")
async def create_post(
    post: PostCreate,
    user: dict = Depends(require_permission("create_post"))
):
    return {"created": True}
```

### 3. **Rate Limiting**

```python
from datetime import datetime, timedelta

# Simple in-memory rate limiter
rate_limit_storage = {}

def rate_limiter(
    max_requests: int = 10,
    window_seconds: int = 60
):
    """Rate limiting dependency"""
    def check_rate_limit(
        client_ip: str = Header(None, alias="X-Real-IP")
    ):
        now = datetime.now()
        key = f"{client_ip}"

        if key not in rate_limit_storage:
            rate_limit_storage[key] = []

        # Remove old requests
        rate_limit_storage[key] = [
            req_time for req_time in rate_limit_storage[key]
            if now - req_time < timedelta(seconds=window_seconds)
        ]

        # Check limit
        if len(rate_limit_storage[key]) >= max_requests:
            raise HTTPException(
                429,
                "Rate limit exceeded. Try again later."
            )

        # Record request
        rate_limit_storage[key].append(now)

        return True

    return check_rate_limit

@app.get("/api/data")
async def get_data(
    _: bool = Depends(rate_limiter(max_requests=5, window_seconds=60))
):
    return {"data": "..."}
```

---

## Comparison: Django vs FastAPI DI

| Aspect        | Django                     | FastAPI               |
| ------------- | -------------------------- | --------------------- |
| DI System     | Middleware, decorators     | Built-in `Depends()`  |
| Type Safety   | Limited                    | Full type checking    |
| Reusability   | View mixins, decorators    | Dependencies          |
| Testing       | Mock middleware/decorators | Override dependencies |
| Async Support | Limited                    | Full support          |
| Nesting       | Complex                    | Simple and clean      |

```python
# Django
from django.contrib.auth.decorators import login_required

@login_required
def my_view(request):
    user = request.user  # From middleware
    return JsonResponse({"user": user.username})

# FastAPI
@app.get("/me")
async def my_endpoint(user: dict = Depends(get_current_user)):
    return {"user": user["username"]}
```

---

## Best Practices

### ✅ Do's:

1. **Keep dependencies simple** and focused
2. **Use `yield`** for cleanup (DB connections, files)
3. **Type hint** dependency parameters
4. **Cache expensive** operations
5. **Use class-based** dependencies for complex logic
6. **Override dependencies** for testing
7. **Nest dependencies** for code reuse

### ❌ Don'ts:

1. **Don't put business logic** in dependencies
2. **Don't create circular** dependencies
3. **Don't forget cleanup** in yield dependencies
4. **Don't overuse** global dependencies
5. **Don't ignore** dependency return types

---

## Interview Questions

### Q1: What is Dependency Injection in FastAPI?

**Answer**: A system where FastAPI automatically resolves and injects dependencies (functions/classes) into path operations. Dependencies are declared using `Depends()` and FastAPI handles execution, validation, and cleanup.

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/")
async def get_users(db = Depends(get_db)):
    # db is automatically injected and cleaned up
    return db.query(User).all()
```

### Q2: How do you test endpoints with dependencies?

**Answer**: Override dependencies using `app.dependency_overrides`:

```python
app.dependency_overrides[get_db] = lambda: test_db
response = client.get("/users/")
app.dependency_overrides.clear()
```

### Q3: What's the difference between `Depends()` and regular parameters?

**Answer**: `Depends()` enables dependency injection with automatic execution, validation, and cleanup. Regular parameters only extract data from the request.

### Q4: Can dependencies have dependencies?

**Answer**: Yes, dependencies can be nested infinitely:

```python
def dep1():
    return "value1"

def dep2(val = Depends(dep1)):
    return f"{val}-value2"

@app.get("/")
async def endpoint(result = Depends(dep2)):
    # dep1 -> dep2 -> endpoint
    return {"result": result}
```

### Q5: What happens to dependencies with `yield` if an exception occurs?

**Answer**: The cleanup code (after `yield`) still executes, similar to a finally block:

```python
def get_resource():
    resource = acquire()
    try:
        yield resource
    finally:
        resource.close()  # Always runs
```

---

## Summary

FastAPI's Dependency Injection system provides:

- **Automatic execution** and validation
- **Type safety** with full IDE support
- **Reusability** across endpoints
- **Easy testing** with overrides
- **Clean code** with separation of concerns

Dependencies are the foundation for building modular, testable, and maintainable FastAPI applications! 💉
