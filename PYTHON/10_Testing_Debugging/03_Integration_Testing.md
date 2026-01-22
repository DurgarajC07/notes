# 🧪 Integration Testing

## Overview

Testing how different components work together, from database integration to API testing and end-to-end scenarios.

---

## Database Integration Testing

### Testing with TestContainers

```python
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture(scope="session")
def postgres_container():
    """PostgreSQL container for testing"""
    with PostgresContainer("postgres:14") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def db_engine(postgres_container):
    """Database engine for testing"""
    connection_url = postgres_container.get_connection_url()
    engine = create_engine(connection_url)

    # Create tables
    Base.metadata.create_all(engine)

    yield engine

    # Cleanup
    Base.metadata.drop_all(engine)
    engine.dispose()

@pytest.fixture
def db_session(db_engine):
    """Database session per test"""
    Session = sessionmaker(bind=db_engine)
    session = Session()

    yield session

    # Rollback after each test
    session.rollback()
    session.close()

# Test repository
def test_user_repository_create(db_session):
    """Test creating user"""
    repo = UserRepository(db_session)

    user = repo.create({
        'username': 'testuser',
        'email': 'test@example.com'
    })

    assert user.id is not None
    assert user.username == 'testuser'

def test_user_repository_get(db_session):
    """Test getting user"""
    repo = UserRepository(db_session)

    # Create user
    created = repo.create({
        'username': 'testuser',
        'email': 'test@example.com'
    })

    # Get user
    user = repo.get(created.id)

    assert user is not None
    assert user.id == created.id
    assert user.username == 'testuser'

def test_user_repository_query(db_session):
    """Test querying users"""
    repo = UserRepository(db_session)

    # Create multiple users
    for i in range(5):
        repo.create({
            'username': f'user{i}',
            'email': f'user{i}@example.com'
        })

    # Query all
    users = repo.query({'limit': 10})
    assert len(users) == 5

    # Query with filter
    active_users = repo.query({'is_active': True})
    assert all(user.is_active for user in active_users)
```

### Testing Transactions

```python
def test_transaction_commit(db_session):
    """Test transaction commits"""
    repo = UserRepository(db_session)

    # Create in transaction
    user = repo.create({'username': 'txuser', 'email': 'tx@example.com'})
    user_id = user.id

    db_session.commit()

    # Verify persisted
    db_session.expire_all()
    user = repo.get(user_id)
    assert user is not None

def test_transaction_rollback(db_session):
    """Test transaction rollback"""
    repo = UserRepository(db_session)

    # Create user
    user = repo.create({'username': 'rollback', 'email': 'rb@example.com'})
    user_id = user.id

    # Rollback
    db_session.rollback()

    # Verify not persisted
    user = repo.get(user_id)
    assert user is None

def test_concurrent_updates(db_session, db_engine):
    """Test concurrent updates"""
    repo1 = UserRepository(db_session)

    # Create user
    user = repo1.create({'username': 'concurrent', 'email': 'c@example.com'})
    db_session.commit()
    user_id = user.id

    # Simulate concurrent update
    Session = sessionmaker(bind=db_engine)
    session2 = Session()
    repo2 = UserRepository(session2)

    # Both update
    user1 = repo1.get(user_id)
    user2 = repo2.get(user_id)

    user1.username = 'updated1'
    user2.username = 'updated2'

    db_session.commit()

    # Second update should fail or override
    with pytest.raises(Exception):
        session2.commit()

    session2.close()
```

---

## API Integration Testing

### FastAPI Test Client

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_user():
    """Test user creation endpoint"""
    response = client.post(
        "/users/",
        json={
            "username": "testuser",
            "email": "test@example.com",
            "password": "securepassword123"
        }
    )

    assert response.status_code == 201
    data = response.json()
    assert data["username"] == "testuser"
    assert data["email"] == "test@example.com"
    assert "password" not in data  # Not exposed

def test_get_user(test_user):
    """Test getting user"""
    response = client.get(f"/users/{test_user['id']}")

    assert response.status_code == 200
    data = response.json()
    assert data["id"] == test_user["id"]

def test_get_user_not_found():
    """Test getting non-existent user"""
    response = client.get("/users/99999")

    assert response.status_code == 404
    assert "not found" in response.json()["detail"].lower()

def test_update_user(test_user, auth_headers):
    """Test updating user"""
    response = client.put(
        f"/users/{test_user['id']}",
        headers=auth_headers,
        json={"username": "newusername"}
    )

    assert response.status_code == 200
    data = response.json()
    assert data["username"] == "newusername"

def test_delete_user(test_user, auth_headers):
    """Test deleting user"""
    response = client.delete(
        f"/users/{test_user['id']}",
        headers=auth_headers
    )

    assert response.status_code == 204

    # Verify deleted
    response = client.get(f"/users/{test_user['id']}")
    assert response.status_code == 404

def test_authentication():
    """Test authentication flow"""
    # Register
    response = client.post(
        "/auth/register",
        json={
            "username": "authuser",
            "email": "auth@example.com",
            "password": "password123"
        }
    )
    assert response.status_code == 201

    # Login
    response = client.post(
        "/auth/login",
        data={
            "username": "authuser",
            "password": "password123"
        }
    )
    assert response.status_code == 200
    token = response.json()["access_token"]

    # Access protected endpoint
    response = client.get(
        "/users/me",
        headers={"Authorization": f"Bearer {token}"}
    )
    assert response.status_code == 200
    assert response.json()["username"] == "authuser"
```

### Testing with Dependencies Override

```python
from fastapi import Depends

# Override database dependency
def override_get_db():
    """Test database session"""
    try:
        db = TestingSessionLocal()
        yield db
    finally:
        db.close()

app.dependency_overrides[get_db] = override_get_db

# Override authentication
def override_get_current_user():
    """Mock authenticated user"""
    return {"id": 1, "username": "testuser", "role": "admin"}

app.dependency_overrides[get_current_user] = override_get_current_user

def test_with_override():
    """Test with overridden dependencies"""
    response = client.get("/users/me")
    assert response.status_code == 200
    assert response.json()["username"] == "testuser"

# Clean up overrides after tests
@pytest.fixture(autouse=True)
def cleanup():
    yield
    app.dependency_overrides = {}
```

---

## External Service Integration

### Mocking External APIs

```python
import responses
import httpx

@responses.activate
def test_external_api_success():
    """Test external API call success"""
    # Mock external API
    responses.add(
        responses.GET,
        'https://api.example.com/data',
        json={'status': 'success', 'data': [1, 2, 3]},
        status=200
    )

    # Call service that uses external API
    result = fetch_external_data()

    assert result == [1, 2, 3]

@responses.activate
def test_external_api_error():
    """Test external API error handling"""
    # Mock error response
    responses.add(
        responses.GET,
        'https://api.example.com/data',
        json={'error': 'Server error'},
        status=500
    )

    # Should handle error gracefully
    with pytest.raises(ExternalAPIError):
        fetch_external_data()

# Using pytest-httpx
from pytest_httpx import HTTPXMock

def test_httpx_mock(httpx_mock: HTTPXMock):
    """Test with httpx mock"""
    httpx_mock.add_response(
        url='https://api.example.com/users/1',
        json={'id': 1, 'name': 'Test User'}
    )

    async def fetch_user():
        async with httpx.AsyncClient() as client:
            response = await client.get('https://api.example.com/users/1')
            return response.json()

    result = asyncio.run(fetch_user())
    assert result['name'] == 'Test User'
```

### Testing with Redis

```python
import fakeredis
import pytest

@pytest.fixture
def redis_client():
    """Fake Redis client for testing"""
    return fakeredis.FakeRedis()

def test_caching(redis_client):
    """Test caching logic"""
    cache = RedisCache(redis_client)

    # Cache miss
    value = cache.get('key1')
    assert value is None

    # Set value
    cache.set('key1', 'value1', ttl=60)

    # Cache hit
    value = cache.get('key1')
    assert value == 'value1'

def test_cache_expiration(redis_client):
    """Test cache expiration"""
    import time

    cache = RedisCache(redis_client)
    cache.set('key1', 'value1', ttl=1)

    # Should exist
    assert cache.get('key1') == 'value1'

    # Wait for expiration
    time.sleep(2)

    # Should be expired
    assert cache.get('key1') is None
```

---

## End-to-End Testing

### Full User Journey

```python
def test_user_registration_flow():
    """Test complete registration flow"""
    # 1. Register user
    response = client.post(
        "/auth/register",
        json={
            "username": "e2euser",
            "email": "e2e@example.com",
            "password": "password123"
        }
    )
    assert response.status_code == 201
    user_id = response.json()["id"]

    # 2. Login
    response = client.post(
        "/auth/login",
        data={"username": "e2euser", "password": "password123"}
    )
    assert response.status_code == 200
    token = response.json()["access_token"]

    headers = {"Authorization": f"Bearer {token}"}

    # 3. Get profile
    response = client.get("/users/me", headers=headers)
    assert response.status_code == 200
    assert response.json()["username"] == "e2euser"

    # 4. Update profile
    response = client.put(
        "/users/me",
        headers=headers,
        json={"bio": "Test bio"}
    )
    assert response.status_code == 200
    assert response.json()["bio"] == "Test bio"

    # 5. Logout (if implemented)
    response = client.post("/auth/logout", headers=headers)
    assert response.status_code == 200

def test_order_workflow():
    """Test complete order workflow"""
    # Setup
    user_token = create_test_user_and_login()
    headers = {"Authorization": f"Bearer {user_token}"}

    # 1. Browse products
    response = client.get("/products")
    assert response.status_code == 200
    products = response.json()
    assert len(products) > 0

    # 2. Add to cart
    response = client.post(
        "/cart/items",
        headers=headers,
        json={"product_id": products[0]["id"], "quantity": 2}
    )
    assert response.status_code == 201

    # 3. View cart
    response = client.get("/cart", headers=headers)
    assert response.status_code == 200
    cart = response.json()
    assert len(cart["items"]) == 1

    # 4. Checkout
    response = client.post(
        "/orders/checkout",
        headers=headers,
        json={
            "shipping_address": "123 Test St",
            "payment_method": "credit_card"
        }
    )
    assert response.status_code == 201
    order = response.json()
    order_id = order["id"]

    # 5. Verify order
    response = client.get(f"/orders/{order_id}", headers=headers)
    assert response.status_code == 200
    assert response.json()["status"] == "pending"

    # 6. Process payment (background task)
    import time
    time.sleep(2)  # Wait for async processing

    response = client.get(f"/orders/{order_id}", headers=headers)
    assert response.json()["status"] == "paid"
```

---

## Testing Async Code

### Async Tests with pytest-asyncio

```python
import pytest
import asyncio

@pytest.mark.asyncio
async def test_async_endpoint():
    """Test async endpoint"""
    from httpx import AsyncClient

    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/async-endpoint")

    assert response.status_code == 200

@pytest.mark.asyncio
async def test_async_database_operation():
    """Test async database operation"""
    async with async_session() as session:
        repo = AsyncUserRepository(session)

        user = await repo.create({
            'username': 'asyncuser',
            'email': 'async@example.com'
        })

        assert user.id is not None

        # Fetch
        fetched = await repo.get(user.id)
        assert fetched.username == 'asyncuser'

@pytest.mark.asyncio
async def test_concurrent_requests():
    """Test concurrent request handling"""
    from httpx import AsyncClient

    async with AsyncClient(app=app, base_url="http://test") as ac:
        # Make 10 concurrent requests
        tasks = [
            ac.get(f"/users/{i}")
            for i in range(1, 11)
        ]

        responses = await asyncio.gather(*tasks)

        # All should succeed
        assert all(r.status_code in [200, 404] for r in responses)
```

---

## Test Fixtures and Factories

### Using Factory Pattern

```python
import factory
from factory.alchemy import SQLAlchemyModelFactory

class UserFactory(SQLAlchemyModelFactory):
    """User factory for testing"""

    class Meta:
        model = User
        sqlalchemy_session = db_session

    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')
    password_hash = 'hashed_password'
    is_active = True

class PostFactory(SQLAlchemyModelFactory):
    """Post factory"""

    class Meta:
        model = Post
        sqlalchemy_session = db_session

    title = factory.Sequence(lambda n: f'Post {n}')
    content = factory.Faker('text')
    author = factory.SubFactory(UserFactory)

# Usage in tests
def test_with_factory(db_session):
    """Test using factories"""
    # Create user
    user = UserFactory()
    assert user.username.startswith('user')

    # Create post with user
    post = PostFactory(author=user)
    assert post.author == user

    # Create multiple
    users = UserFactory.create_batch(5)
    assert len(users) == 5
```

### Fixture Sharing

```python
# conftest.py - shared fixtures
import pytest

@pytest.fixture(scope="session")
def db_engine():
    """Database engine for all tests"""
    engine = create_test_engine()
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)

@pytest.fixture
def db_session(db_engine):
    """Database session per test"""
    Session = sessionmaker(bind=db_engine)
    session = Session()
    yield session
    session.rollback()
    session.close()

@pytest.fixture
def test_user(db_session):
    """Create test user"""
    user = UserFactory()
    db_session.commit()
    return user

@pytest.fixture
def auth_headers(test_user):
    """Authentication headers"""
    token = create_access_token({"sub": test_user.username})
    return {"Authorization": f"Bearer {token}"}

@pytest.fixture
def test_client():
    """Test client with app"""
    return TestClient(app)
```

---

## Best Practices

### ✅ Do's:

1. **Use test containers** for real databases
2. **Test transactions** and rollbacks
3. **Mock external services** to control behavior
4. **Test error scenarios** not just happy path
5. **Use factories** for test data
6. **Test async code** properly with pytest-asyncio
7. **Share fixtures** in conftest.py
8. **Test authentication** flows completely
9. **Clean up** test data after tests
10. **Run integration tests** in CI/CD

### ❌ Don'ts:

1. **Don't use production** database for tests
2. **Don't share state** between tests
3. **Don't skip cleanup** - isolate tests
4. **Don't hardcode** test data
5. **Don't test** implementation details
6. **Don't ignore** flaky tests
7. **Don't make tests** dependent on order

---

## Interview Questions

### Q1: Unit tests vs Integration tests?

**Answer**:

- **Unit**: Single component, mocked dependencies, fast
- **Integration**: Multiple components, real dependencies, slower
- **Both needed**: Unit for logic, integration for interactions
- **Pyramid**: More unit, fewer integration, few E2E

### Q2: How to test database transactions?

**Answer**:

- **Test session**: Rollback after each test
- **Test containers**: Real database
- **Test commit**: Verify persistence
- **Test rollback**: Verify cleanup
- **Test concurrency**: Locking, race conditions

### Q3: How to handle external API in tests?

**Answer**:

- **Mock responses**: responses library, httpx mock
- **Controlled behavior**: Success, errors, timeouts
- **Integration tests**: Test against sandbox
- **Contract tests**: Verify API contract
  Don't hit real API in tests.

### Q4: What are test fixtures?

**Answer**:

- **Setup/teardown**: Prepare test environment
- **Reusable**: Share across tests
- **Scopes**: Function, class, module, session
- **Factories**: Generate test data
  Use fixtures for DRY tests.

### Q5: How to test async endpoints?

**Answer**:

- **pytest-asyncio**: Mark tests with @pytest.mark.asyncio
- **AsyncClient**: For HTTP requests
- **await**: All async operations
- **Test concurrency**: Multiple simultaneous requests
  Use proper async testing tools.

---

## Summary

Integration testing essentials:

- **Database**: Test containers, transactions, rollbacks
- **API**: TestClient, dependency overrides
- **External**: Mock external services
- **E2E**: Full user journeys
- **Async**: pytest-asyncio for async code
- **Fixtures**: Share setup across tests

Test interactions, not just units! 🧪
