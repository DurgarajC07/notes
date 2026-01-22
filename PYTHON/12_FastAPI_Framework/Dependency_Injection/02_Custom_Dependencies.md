# 🔧 Creating Custom Dependencies

## Overview

Custom dependencies in FastAPI allow you to encapsulate reusable logic, validation, authentication, and business rules in a clean, modular way.

---

## Dependency Types

### 1. **Function-Based Dependencies**

```python
from fastapi import Depends, HTTPException, Header
from typing import Optional

# Simple dependency
def common_parameters(
    q: Optional[str] = None,
    skip: int = 0,
    limit: int = 100
):
    return {"q": q, "skip": skip, "limit": limit}

# Dependency with validation
def validate_api_key(api_key: str = Header(...)):
    if api_key != "secret-key":
        raise HTTPException(403, "Invalid API key")
    return api_key

# Dependency with logic
def get_pagination(page: int = 1, size: int = 10):
    if page < 1:
        raise HTTPException(400, "Page must be >= 1")
    if size < 1 or size > 100:
        raise HTTPException(400, "Size must be between 1 and 100")

    offset = (page - 1) * size
    return {"offset": offset, "limit": size, "page": page}
```

### 2. **Class-Based Dependencies**

```python
from typing import Optional
from pydantic import BaseModel

class FilterParams:
    """Reusable filter parameters"""
    def __init__(
        self,
        search: Optional[str] = None,
        category: Optional[str] = None,
        min_price: Optional[float] = None,
        max_price: Optional[float] = None,
        sort_by: str = "created_at",
        order: str = "desc"
    ):
        self.search = search
        self.category = category
        self.min_price = min_price
        self.max_price = max_price
        self.sort_by = sort_by
        self.order = order

    def build_query_filters(self) -> dict:
        """Convert to database query filters"""
        filters = {}
        if self.search:
            filters["name__icontains"] = self.search
        if self.category:
            filters["category"] = self.category
        if self.min_price:
            filters["price__gte"] = self.min_price
        if self.max_price:
            filters["price__lte"] = self.max_price
        return filters

    def get_order_by(self) -> str:
        """Get SQL ORDER BY clause"""
        direction = "DESC" if self.order == "desc" else "ASC"
        return f"{self.sort_by} {direction}"

# Usage
@app.get("/products/")
async def list_products(filters: FilterParams = Depends()):
    query_filters = filters.build_query_filters()
    order = filters.get_order_by()

    # Use in query
    products = await Product.filter(**query_filters).order_by(order)
    return products
```

### 3. **Callable Classes (Custom Callable)**

```python
class RequireRole:
    """Factory pattern for role-based access control"""
    def __init__(self, required_role: str):
        self.required_role = required_role

    def __call__(self, current_user: User = Depends(get_current_user)):
        if current_user.role != self.required_role:
            raise HTTPException(
                403,
                f"Role '{self.required_role}' required"
            )
        return current_user

# Create specific dependencies
require_admin = RequireRole("admin")
require_moderator = RequireRole("moderator")

# Usage
@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin: User = Depends(require_admin)
):
    # Only admins can access
    return {"deleted": user_id}

@app.post("/posts/{post_id}/approve")
async def approve_post(
    post_id: int,
    mod: User = Depends(require_moderator)
):
    # Only moderators can access
    return {"approved": post_id}
```

---

## Advanced Dependency Patterns

### 1. **Dependency Factory Functions**

```python
from typing import List

def permission_checker(required_permissions: List[str]):
    """Factory that creates permission checking dependency"""
    def check_permissions(user: User = Depends(get_current_user)):
        user_permissions = set(user.permissions)
        required = set(required_permissions)

        if not required.issubset(user_permissions):
            missing = required - user_permissions
            raise HTTPException(
                403,
                f"Missing permissions: {', '.join(missing)}"
            )
        return user

    return check_permissions

# Create specific permission dependencies
require_read = permission_checker(["read"])
require_write = permission_checker(["write"])
require_admin_access = permission_checker(["admin", "write", "delete"])

# Usage
@app.get("/data/")
async def read_data(user: User = Depends(require_read)):
    return {"data": "..."}

@app.post("/data/")
async def create_data(
    data: DataCreate,
    user: User = Depends(require_write)
):
    return {"created": True}

@app.delete("/data/{id}")
async def delete_data(
    id: int,
    user: User = Depends(require_admin_access)
):
    return {"deleted": id}
```

### 2. **Dependency with State**

```python
from datetime import datetime, timedelta
from collections import defaultdict

class RateLimiter:
    """Rate limiter with state"""
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(list)

    def __call__(self, request: Request):
        client_id = request.client.host
        now = datetime.now()

        # Clean old requests
        cutoff = now - timedelta(seconds=self.window_seconds)
        self.requests[client_id] = [
            req_time for req_time in self.requests[client_id]
            if req_time > cutoff
        ]

        # Check limit
        if len(self.requests[client_id]) >= self.max_requests:
            raise HTTPException(
                429,
                f"Rate limit exceeded: {self.max_requests} requests per {self.window_seconds}s"
            )

        # Record request
        self.requests[client_id].append(now)
        return True

# Create rate limiters
strict_limiter = RateLimiter(max_requests=5, window_seconds=60)
relaxed_limiter = RateLimiter(max_requests=100, window_seconds=60)

# Usage
@app.get("/api/public")
async def public_api(_: bool = Depends(relaxed_limiter)):
    return {"data": "public"}

@app.get("/api/expensive")
async def expensive_api(_: bool = Depends(strict_limiter)):
    return {"data": "expensive operation"}
```

### 3. **Async Dependencies**

```python
from sqlalchemy.ext.asyncio import AsyncSession

async def get_current_user_async(
    token: str = Header(...),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Async dependency with database lookup"""
    result = await db.execute(
        select(User).where(User.token == token)
    )
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(401, "Invalid credentials")

    return user

async def cache_check(key: str, redis = Depends(get_redis)):
    """Check Redis cache"""
    cached = await redis.get(key)
    if cached:
        return json.loads(cached)
    return None

# Usage
@app.get("/profile")
async def get_profile(user: User = Depends(get_current_user_async)):
    return user

@app.get("/data/{key}")
async def get_data(
    key: str,
    cached = Depends(cache_check)
):
    if cached:
        return cached

    # Fetch from database
    data = await fetch_data(key)
    return data
```

---

## Context Manager Dependencies

### 1. **Database Transaction**

```python
from contextlib import asynccontextmanager

async def get_transaction():
    """Automatic transaction management"""
    async with get_db() as session:
        async with session.begin():
            try:
                yield session
            except Exception:
                await session.rollback()
                raise

@app.post("/transfer")
async def transfer_money(
    transfer: TransferRequest,
    db: AsyncSession = Depends(get_transaction)
):
    # Transaction automatically commits or rolls back
    from_account = await db.get(Account, transfer.from_id)
    to_account = await db.get(Account, transfer.to_id)

    from_account.balance -= transfer.amount
    to_account.balance += transfer.amount

    return {"status": "success"}
```

### 2. **File Handling**

```python
async def temp_file():
    """Temporary file dependency"""
    import tempfile

    temp = tempfile.NamedTemporaryFile(delete=False)
    try:
        yield temp
    finally:
        temp.close()
        os.unlink(temp.name)

@app.post("/process-file")
async def process_file(
    file: UploadFile,
    temp = Depends(temp_file)
):
    # Write to temp file
    content = await file.read()
    temp.write(content)
    temp.flush()

    # Process temp file
    result = process_image(temp.name)

    return result
    # Temp file automatically cleaned up
```

---

## Validation Dependencies

### 1. **Request Body Validation**

```python
from pydantic import validator

class CreateUserValidator:
    """Validate user creation data"""
    def __init__(self, user: UserCreate):
        self.user = user
        self._validate()

    def _validate(self):
        # Email validation
        if not "@" in self.user.email:
            raise HTTPException(400, "Invalid email format")

        # Password strength
        if len(self.user.password) < 8:
            raise HTTPException(400, "Password too short")

        # Username validation
        if not self.user.username.isalnum():
            raise HTTPException(400, "Username must be alphanumeric")

    def get_user_data(self) -> dict:
        return self.user.dict()

@app.post("/users/")
async def create_user(
    validator: CreateUserValidator = Depends()
):
    user_data = validator.get_user_data()
    # Create user
    return {"created": True}
```

### 2. **Header Validation**

```python
def validate_content_type(content_type: str = Header(...)):
    """Ensure correct content type"""
    allowed = ["application/json", "application/xml"]
    if content_type not in allowed:
        raise HTTPException(
            415,
            f"Unsupported content type: {content_type}"
        )
    return content_type

def validate_accept(accept: str = Header(...)):
    """Validate Accept header"""
    if "application/json" not in accept:
        raise HTTPException(
            406,
            "Only application/json is supported"
        )
    return accept

@app.post("/data/")
async def create_data(
    data: dict,
    content_type: str = Depends(validate_content_type),
    accept: str = Depends(validate_accept)
):
    return {"received": data}
```

---

## Caching Dependencies

### 1. **Request-Level Cache**

```python
from functools import lru_cache

@lru_cache()
def get_settings():
    """Cached settings (singleton pattern)"""
    return Settings()

@app.get("/config")
async def get_config(settings: Settings = Depends(get_settings)):
    # Settings loaded only once per application
    return settings
```

### 2. **Time-Based Cache**

```python
from datetime import datetime, timedelta

class CachedDependency:
    """Dependency with time-based cache"""
    def __init__(self, ttl_seconds: int = 60):
        self.ttl_seconds = ttl_seconds
        self._cache = None
        self._cached_at = None

    def __call__(self):
        now = datetime.now()

        # Check if cache is valid
        if (self._cache is None or
            self._cached_at is None or
            now - self._cached_at > timedelta(seconds=self.ttl_seconds)):

            # Refresh cache
            self._cache = self._fetch_data()
            self._cached_at = now

        return self._cache

    def _fetch_data(self):
        # Expensive operation
        return {"data": "expensive computation"}

# Create cached dependency (60 second TTL)
get_cached_data = CachedDependency(ttl_seconds=60)

@app.get("/data")
async def get_data(data: dict = Depends(get_cached_data)):
    return data
```

---

## Error Handling Dependencies

### 1. **Graceful Degradation**

```python
async def get_cache_with_fallback(
    key: str,
    redis = Depends(get_redis)
):
    """Try cache, fallback to database"""
    try:
        cached = await redis.get(key)
        if cached:
            return {"source": "cache", "data": json.loads(cached)}
    except Exception as e:
        logger.warning(f"Cache error: {e}")

    # Fallback to database
    return {"source": "database", "data": await fetch_from_db(key)}
```

### 2. **Retry Logic**

```python
from tenacity import retry, stop_after_attempt, wait_exponential

class RetryableDependency:
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10)
    )
    async def __call__(self, url: str):
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            response.raise_for_status()
            return response.json()

get_external_data = RetryableDependency()

@app.get("/external/{resource}")
async def get_external(
    resource: str,
    data: dict = Depends(get_external_data)
):
    return data
```

---

## Composite Dependencies

### 1. **Combining Multiple Dependencies**

```python
class RequestContext:
    """Combine multiple dependencies into single context"""
    def __init__(
        self,
        user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db),
        redis = Depends(get_redis),
        request: Request = None
    ):
        self.user = user
        self.db = db
        self.redis = redis
        self.request = request

    async def log_action(self, action: str):
        """Log user action"""
        log_entry = {
            "user_id": self.user.id,
            "action": action,
            "ip": self.request.client.host,
            "timestamp": datetime.now()
        }
        await self.db.execute(insert(AuditLog).values(log_entry))

    async def get_user_cache(self, key: str):
        """Get user-specific cached data"""
        cache_key = f"user:{self.user.id}:{key}"
        return await self.redis.get(cache_key)

# Usage
@app.post("/posts/")
async def create_post(
    post: PostCreate,
    ctx: RequestContext = Depends()
):
    # Use context
    new_post = Post(**post.dict(), author_id=ctx.user.id)
    ctx.db.add(new_post)
    await ctx.db.commit()

    # Log action
    await ctx.log_action("create_post")

    return new_post
```

---

## Testing Custom Dependencies

### 1. **Override Dependencies**

```python
# Original dependency
def get_settings():
    return Settings(database_url="postgresql://prod")

# Test override
def override_settings():
    return Settings(database_url="sqlite:///:memory:")

# In tests
app.dependency_overrides[get_settings] = override_settings

client = TestClient(app)
response = client.get("/config")

app.dependency_overrides.clear()
```

### 2. **Mock Dependencies**

```python
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_endpoint_with_mock():
    # Mock user dependency
    mock_user = User(id=1, username="testuser")

    async def mock_get_user():
        return mock_user

    app.dependency_overrides[get_current_user] = mock_get_user

    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/profile")

        assert response.status_code == 200
        assert response.json()["username"] == "testuser"

    app.dependency_overrides.clear()
```

---

## Best Practices

### ✅ Do's:

1. **Keep dependencies focused** on single responsibility
2. **Use type hints** for better IDE support
3. **Handle errors gracefully** in dependencies
4. **Use factories** for configurable dependencies
5. **Cache expensive operations** appropriately
6. **Document dependency behavior** clearly
7. **Test dependencies** independently

### ❌ Don'ts:

1. **Don't put business logic** in dependencies
2. **Don't create deep nesting** (>3 levels)
3. **Don't ignore cleanup** in yield dependencies
4. **Don't share mutable state** carelessly
5. **Don't forget error handling**
6. **Don't over-complicate** simple dependencies

---

## Interview Questions

### Q1: What's the difference between a regular function and a dependency?

**Answer**: A dependency is a function marked with `Depends()` that FastAPI automatically executes, validates, and injects. Regular functions must be called manually.

### Q2: How do you create a dependency factory?

**Answer**: Create a function that returns a dependency function:

```python
def permission_factory(permission: str):
    def check(user = Depends(get_user)):
        if permission not in user.permissions:
            raise HTTPException(403)
        return user
    return check

require_admin = permission_factory("admin")
```

### Q3: When should you use class-based dependencies?

**Answer**: When you need complex logic, state management, or multiple related methods. Classes provide better organization than simple functions.

### Q4: How do you handle dependency errors?

**Answer**: Raise `HTTPException` in the dependency:

```python
def validate_token(token: str = Header(...)):
    if not is_valid(token):
        raise HTTPException(401, "Invalid token")
    return token
```

### Q5: Can async and sync dependencies be mixed?

**Answer**: Yes, FastAPI handles both automatically. However, sync dependencies block the event loop, so prefer async when possible.

---

## Summary

Custom dependencies in FastAPI enable:

- **Code reuse** across endpoints
- **Separation of concerns**
- **Easy testing** with overrides
- **Type safety** and validation
- **Flexible composition**

Well-designed dependencies make your FastAPI application modular, testable, and maintainable! 🔧
