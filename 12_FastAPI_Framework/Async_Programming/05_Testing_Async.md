# 🧪 Testing Async Code in FastAPI

## Overview

Testing async code requires special tools and patterns to properly handle coroutines and event loops. FastAPI provides excellent support for testing through `httpx.AsyncClient` and `pytest-asyncio`.

---

## Test Setup

### 1. **Required Packages**

```bash
pip install pytest pytest-asyncio httpx faker
```

### 2. **pytest Configuration**

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
```

```ini
# pytest.ini (alternative)
[pytest]
asyncio_mode = auto
testpaths = tests
```

---

## Testing FastAPI Endpoints

### 1. **Basic Async Test**

```python
import pytest
from httpx import AsyncClient
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def read_root():
    return {"message": "Hello World"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id, "name": "John Doe"}

# Test file: test_main.py
@pytest.mark.asyncio
async def test_read_root():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/")

    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

@pytest.mark.asyncio
async def test_get_user():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/users/123")

    assert response.status_code == 200
    assert response.json()["user_id"] == 123
```

### 2. **Test Fixtures**

```python
import pytest
from httpx import AsyncClient
from typing import AsyncGenerator

@pytest.fixture
async def async_client() -> AsyncGenerator[AsyncClient, None]:
    """Reusable async client fixture"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.mark.asyncio
async def test_with_fixture(async_client: AsyncClient):
    response = await async_client.get("/")
    assert response.status_code == 200

# Test POST requests
@pytest.mark.asyncio
async def test_create_user(async_client: AsyncClient):
    user_data = {
        "email": "test@example.com",
        "name": "Test User"
    }

    response = await async_client.post("/users/", json=user_data)

    assert response.status_code == 201
    data = response.json()
    assert data["email"] == user_data["email"]
    assert "id" in data
```

### 3. **Testing with Headers and Authentication**

```python
@pytest.fixture
def auth_headers() -> dict:
    """Authentication headers fixture"""
    token = "fake-jwt-token"
    return {"Authorization": f"Bearer {token}"}

@pytest.mark.asyncio
async def test_protected_endpoint(
    async_client: AsyncClient,
    auth_headers: dict
):
    response = await async_client.get(
        "/protected",
        headers=auth_headers
    )

    assert response.status_code == 200

@pytest.mark.asyncio
async def test_unauthorized_access(async_client: AsyncClient):
    response = await async_client.get("/protected")
    assert response.status_code == 401
```

---

## Testing Database Operations

### 1. **Test Database Setup**

```python
import pytest
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker
)
from sqlalchemy.pool import NullPool

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

@pytest.fixture
async def test_db() -> AsyncGenerator[AsyncSession, None]:
    """Create test database session"""
    # Create engine with no connection pooling
    engine = create_async_engine(
        TEST_DATABASE_URL,
        echo=False,
        poolclass=NullPool
    )

    # Create tables
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    # Create session
    TestSessionLocal = async_sessionmaker(
        engine,
        class_=AsyncSession,
        expire_on_commit=False
    )

    async with TestSessionLocal() as session:
        yield session

    # Cleanup
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

    await engine.dispose()

# Override dependency
from app.database import get_db

@pytest.fixture
async def async_client_with_db(
    test_db: AsyncSession
) -> AsyncGenerator[AsyncClient, None]:
    """Client with test database"""

    async def override_get_db():
        yield test_db

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

    app.dependency_overrides.clear()

# Test with database
@pytest.mark.asyncio
async def test_create_user_in_db(async_client_with_db: AsyncClient):
    user_data = {
        "email": "test@example.com",
        "name": "Test User"
    }

    response = await async_client_with_db.post("/users/", json=user_data)

    assert response.status_code == 201

    # Verify created
    user_id = response.json()["id"]
    get_response = await async_client_with_db.get(f"/users/{user_id}")
    assert get_response.status_code == 200
```

### 2. **Testing Repository Layer**

```python
@pytest.mark.asyncio
async def test_user_repository_create(test_db: AsyncSession):
    """Test repository create method"""
    repo = UserRepository(test_db)

    user = await repo.create(
        email="test@example.com",
        name="Test User"
    )

    assert user.id is not None
    assert user.email == "test@example.com"

@pytest.mark.asyncio
async def test_user_repository_get_by_email(test_db: AsyncSession):
    """Test repository get_by_email method"""
    repo = UserRepository(test_db)

    # Create user
    await repo.create(email="test@example.com", name="Test")

    # Fetch by email
    user = await repo.get_by_email("test@example.com")

    assert user is not None
    assert user.email == "test@example.com"

@pytest.mark.asyncio
async def test_user_repository_update(test_db: AsyncSession):
    """Test repository update method"""
    repo = UserRepository(test_db)

    # Create user
    user = await repo.create(email="test@example.com", name="Old Name")

    # Update
    updated = await repo.update(user.id, name="New Name")

    assert updated.name == "New Name"
```

---

## Mocking Async Functions

### 1. **Using AsyncMock**

```python
from unittest.mock import AsyncMock, patch
import pytest

# Service to test
class EmailService:
    async def send_email(self, to: str, subject: str, body: str):
        # Actually sends email
        pass

# Test with mock
@pytest.mark.asyncio
async def test_user_registration_sends_email():
    email_service = EmailService()
    email_service.send_email = AsyncMock(return_value=True)

    # Test code that calls send_email
    result = await email_service.send_email(
        "user@example.com",
        "Welcome",
        "Thanks for joining"
    )

    assert result is True
    email_service.send_email.assert_called_once_with(
        "user@example.com",
        "Welcome",
        "Thanks for joining"
    )
```

### 2. **Patching External API Calls**

```python
import httpx

async def fetch_external_data(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()

@pytest.mark.asyncio
async def test_fetch_external_data():
    mock_response = AsyncMock()
    mock_response.json = AsyncMock(return_value={"data": "test"})

    with patch("httpx.AsyncClient.get", return_value=mock_response):
        result = await fetch_external_data("https://api.example.com")

        assert result == {"data": "test"}

# Alternative: Mock entire client
@pytest.mark.asyncio
async def test_fetch_with_mock_client():
    mock_client = AsyncMock()
    mock_client.__aenter__.return_value = mock_client
    mock_client.get = AsyncMock(
        return_value=AsyncMock(
            json=AsyncMock(return_value={"data": "test"})
        )
    )

    with patch("httpx.AsyncClient", return_value=mock_client):
        result = await fetch_external_data("https://api.example.com")
        assert result == {"data": "test"}
```

### 3. **Mocking Database Queries**

```python
from sqlalchemy.ext.asyncio import AsyncSession
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_get_user_mock():
    # Mock session
    mock_session = AsyncMock(spec=AsyncSession)

    # Mock query result
    mock_result = AsyncMock()
    mock_result.scalar_one_or_none.return_value = User(
        id=1,
        email="test@example.com",
        name="Test User"
    )

    mock_session.execute.return_value = mock_result

    # Test repository
    repo = UserRepository(mock_session)
    user = await repo.get_by_id(1)

    assert user.id == 1
    assert user.email == "test@example.com"
    mock_session.execute.assert_called_once()
```

---

## Testing Background Tasks

### 1. **Testing BackgroundTasks**

```python
from fastapi import BackgroundTasks

# Endpoint with background task
@app.post("/send-notification")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks
):
    background_tasks.add_task(send_email, email)
    return {"message": "Notification scheduled"}

async def send_email(email: str):
    # Send email logic
    await asyncio.sleep(1)
    print(f"Email sent to {email}")

# Test
@pytest.mark.asyncio
async def test_background_task(async_client: AsyncClient):
    with patch("app.main.send_email", new_callable=AsyncMock) as mock_send:
        response = await async_client.post(
            "/send-notification",
            json={"email": "test@example.com"}
        )

        assert response.status_code == 200

        # Wait for background task
        await asyncio.sleep(0.1)

        mock_send.assert_called_once_with("test@example.com")
```

---

## Testing Concurrent Operations

### 1. **Testing Parallel Requests**

```python
@pytest.mark.asyncio
async def test_concurrent_requests(async_client: AsyncClient):
    """Test multiple concurrent requests"""

    async def make_request(user_id: int):
        return await async_client.get(f"/users/{user_id}")

    # Make 10 concurrent requests
    tasks = [make_request(i) for i in range(1, 11)]
    responses = await asyncio.gather(*tasks)

    # Verify all succeeded
    assert all(r.status_code == 200 for r in responses)
    assert len(responses) == 10

@pytest.mark.asyncio
async def test_rate_limiting():
    """Test rate limiting with concurrent requests"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        # Make 100 concurrent requests
        tasks = [client.get("/api/data") for _ in range(100)]
        responses = await asyncio.gather(*tasks)

        # Check rate limit responses
        success = sum(1 for r in responses if r.status_code == 200)
        rate_limited = sum(1 for r in responses if r.status_code == 429)

        assert success > 0
        assert rate_limited > 0
```

### 2. **Testing Timeouts**

```python
@pytest.mark.asyncio
async def test_timeout_handling():
    """Test that slow operations timeout properly"""

    @app.get("/slow")
    async def slow_endpoint():
        await asyncio.sleep(10)
        return {"status": "done"}

    with pytest.raises(httpx.ReadTimeout):
        async with AsyncClient(
            app=app,
            base_url="http://test",
            timeout=1.0  # 1 second timeout
        ) as client:
            await client.get("/slow")
```

---

## Testing Error Handling

### 1. **Testing Exception Handlers**

```python
from fastapi import HTTPException

@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail}
    )

@pytest.mark.asyncio
async def test_404_error(async_client: AsyncClient):
    response = await async_client.get("/nonexistent")

    assert response.status_code == 404
    assert "error" in response.json()

@pytest.mark.asyncio
async def test_validation_error(async_client: AsyncClient):
    # Invalid data
    response = await async_client.post(
        "/users/",
        json={"email": "invalid-email"}  # Missing required fields
    )

    assert response.status_code == 422
    assert "detail" in response.json()
```

### 2. **Testing Database Errors**

```python
@pytest.mark.asyncio
async def test_duplicate_email_error(async_client_with_db: AsyncClient):
    user_data = {
        "email": "duplicate@example.com",
        "name": "User"
    }

    # Create first user
    response1 = await async_client_with_db.post("/users/", json=user_data)
    assert response1.status_code == 201

    # Try to create duplicate
    response2 = await async_client_with_db.post("/users/", json=user_data)
    assert response2.status_code == 400
    assert "already exists" in response2.json()["detail"].lower()
```

---

## Testing WebSockets

### 1. **WebSocket Test Setup**

```python
from fastapi import WebSocket

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Echo: {data}")

@pytest.mark.asyncio
async def test_websocket():
    async with AsyncClient(app=app, base_url="http://test") as client:
        async with client.websocket_connect("/ws") as websocket:
            # Send message
            await websocket.send_text("Hello")

            # Receive response
            data = await websocket.receive_text()

            assert data == "Echo: Hello"
```

---

## Test Data Factories

### 1. **Using Faker for Test Data**

```python
from faker import Faker

fake = Faker()

@pytest.fixture
def user_factory():
    """Factory for creating test user data"""
    def _create_user(**kwargs):
        default_data = {
            "email": fake.email(),
            "name": fake.name(),
            "age": fake.random_int(18, 80)
        }
        default_data.update(kwargs)
        return default_data

    return _create_user

@pytest.mark.asyncio
async def test_with_factory(
    async_client_with_db: AsyncClient,
    user_factory
):
    # Create user with random data
    user_data = user_factory()
    response = await async_client_with_db.post("/users/", json=user_data)

    assert response.status_code == 201

    # Create user with specific email
    specific_user = user_factory(email="specific@example.com")
    response2 = await async_client_with_db.post("/users/", json=specific_user)

    assert response2.json()["email"] == "specific@example.com"
```

### 2. **Fixture Parametrization**

```python
@pytest.mark.parametrize("email,name,expected_status", [
    ("valid@example.com", "John Doe", 201),
    ("invalid-email", "John Doe", 422),
    ("test@example.com", "", 422),
    ("", "John Doe", 422),
])
@pytest.mark.asyncio
async def test_user_creation_validation(
    async_client: AsyncClient,
    email: str,
    name: str,
    expected_status: int
):
    response = await async_client.post(
        "/users/",
        json={"email": email, "name": name}
    )

    assert response.status_code == expected_status
```

---

## Performance Testing

### 1. **Load Testing**

```python
import time

@pytest.mark.asyncio
async def test_endpoint_performance():
    """Test endpoint response time"""
    start = time.time()

    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/users/")

    duration = time.time() - start

    assert response.status_code == 200
    assert duration < 1.0  # Should respond in under 1 second

@pytest.mark.asyncio
async def test_concurrent_load():
    """Test performance under concurrent load"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        start = time.time()

        # 100 concurrent requests
        tasks = [client.get("/users/") for _ in range(100)]
        responses = await asyncio.gather(*tasks)

        duration = time.time() - start

        assert all(r.status_code == 200 for r in responses)
        assert duration < 5.0  # 100 requests in under 5 seconds

        print(f"Handled 100 requests in {duration:.2f}s")
        print(f"Throughput: {100/duration:.2f} req/s")
```

---

## Coverage and Best Practices

### 1. **Code Coverage**

```bash
# Install coverage
pip install pytest-cov

# Run tests with coverage
pytest --cov=app --cov-report=html

# View coverage report
# Open htmlcov/index.html
```

### 2. **Test Organization**

```
tests/
├── __init__.py
├── conftest.py          # Shared fixtures
├── test_main.py         # Main app tests
├── test_users.py        # User endpoint tests
├── test_auth.py         # Authentication tests
├── test_database.py     # Database tests
└── test_integration.py  # Integration tests
```

```python
# conftest.py - Shared fixtures
import pytest
from httpx import AsyncClient
from typing import AsyncGenerator

@pytest.fixture(scope="session")
def anyio_backend():
    return "asyncio"

@pytest.fixture
async def async_client() -> AsyncGenerator[AsyncClient, None]:
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.fixture
async def test_db() -> AsyncGenerator[AsyncSession, None]:
    # Database setup
    yield session
    # Cleanup
```

---

## Comparison: Testing Sync vs Async

| Aspect         | Sync (Django)        | Async (FastAPI)        |
| -------------- | -------------------- | ---------------------- |
| Test Client    | `TestClient` (sync)  | `AsyncClient` (async)  |
| Test Decorator | Not needed           | `@pytest.mark.asyncio` |
| Database       | Django test database | In-memory or test DB   |
| Fixtures       | Standard fixtures    | Async fixtures         |
| Mocking        | `Mock`               | `AsyncMock`            |
| Setup/Teardown | `setUp()/tearDown()` | `async with` fixtures  |

```python
# Django (Sync)
from django.test import TestCase

class UserTestCase(TestCase):
    def test_create_user(self):
        response = self.client.post('/users/', data)
        self.assertEqual(response.status_code, 201)

# FastAPI (Async)
@pytest.mark.asyncio
async def test_create_user(async_client):
    response = await async_client.post('/users/', json=data)
    assert response.status_code == 201
```

---

## Best Practices

### ✅ Do's:

1. **Use fixtures** for common setup
2. **Mock external dependencies** (APIs, services)
3. **Test edge cases** and error conditions
4. **Use parametrize** for multiple test cases
5. **Isolate tests** - each test should be independent
6. **Use in-memory database** for speed
7. **Test async operations** properly with AsyncClient
8. **Measure coverage** aim for >80%

### ❌ Don'ts:

1. **Don't share state** between tests
2. **Don't test external APIs** directly
3. **Don't forget** to mark async tests
4. **Don't use real database** in tests
5. **Don't skip cleanup** in fixtures
6. **Don't test implementation** details

---

## Interview Questions

### Q1: How do you test async FastAPI endpoints?

**Answer**: Use `httpx.AsyncClient` with `pytest-asyncio`:

```python
@pytest.mark.asyncio
async def test_endpoint(async_client: AsyncClient):
    response = await async_client.get("/api/users")
    assert response.status_code == 200
```

### Q2: What's the difference between `Mock` and `AsyncMock`?

**Answer**:

- `Mock`: For synchronous functions, returns regular values
- `AsyncMock`: For coroutines, can be awaited

```python
# Mock sync function
mock_func = Mock(return_value="result")
result = mock_func()

# Mock async function
mock_async = AsyncMock(return_value="result")
result = await mock_async()
```

### Q3: How do you test database transactions in async code?

**Answer**: Use test database with rollback after each test:

```python
@pytest.fixture
async def test_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async with TestSessionLocal() as session:
        yield session
        # Rollback happens automatically
```

### Q4: How do you test concurrent requests?

**Answer**: Use `asyncio.gather()` to run requests in parallel:

```python
tasks = [client.get(f"/users/{i}") for i in range(10)]
responses = await asyncio.gather(*tasks)
assert all(r.status_code == 200 for r in responses)
```

### Q5: What's `asyncio_mode = "auto"` in pytest config?

**Answer**: Automatically detects and runs async test functions without requiring `@pytest.mark.asyncio` on every test:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"  # Auto-detect async tests
```

---

## Summary

Testing async FastAPI applications requires:

- **AsyncClient**: For testing HTTP endpoints
- **pytest-asyncio**: For running async tests
- **AsyncMock**: For mocking async functions
- **Test Database**: Isolated database for tests
- **Fixtures**: Reusable test setup
- **Coverage**: Measure test coverage

Proper testing ensures your async FastAPI application is reliable, maintainable, and production-ready! 🧪
