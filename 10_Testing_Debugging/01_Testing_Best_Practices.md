# 🧪 Testing Best Practices

## Overview

Comprehensive testing ensures code reliability, catches bugs early, enables refactoring confidence, and serves as living documentation.

---

## Testing Pyramid

```
        /\
       /  \        E2E Tests (Few)
      /----\
     /      \      Integration Tests (Some)
    /--------\
   /          \    Unit Tests (Many)
  /____________\
```

---

## pytest Fundamentals

### Basic Test Structure

```python
# test_calculator.py
import pytest

def add(a, b):
    """Function to test"""
    return a + b

# Test naming: test_* or *_test.py
def test_add_positive_numbers():
    """Test description in docstring"""
    # Arrange
    a, b = 2, 3

    # Act
    result = add(a, b)

    # Assert
    assert result == 5

def test_add_negative_numbers():
    """Test negative numbers"""
    assert add(-1, -1) == -2

def test_add_zero():
    """Test with zero"""
    assert add(5, 0) == 5
    assert add(0, 5) == 5

# Run tests: pytest test_calculator.py
# Run with coverage: pytest --cov=myapp tests/
```

### Fixtures

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture
def db_session():
    """Database session fixture"""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)

    Session = sessionmaker(bind=engine)
    session = Session()

    yield session

    session.close()

@pytest.fixture
def sample_user(db_session):
    """Create sample user"""
    user = User(username="testuser", email="test@example.com")
    db_session.add(user)
    db_session.commit()

    return user

def test_user_creation(db_session, sample_user):
    """Test using fixtures"""
    user = db_session.query(User).filter_by(
        username="testuser"
    ).first()

    assert user is not None
    assert user.email == "test@example.com"

# Fixture scopes
@pytest.fixture(scope="function")  # Default: new for each test
def function_fixture():
    return {}

@pytest.fixture(scope="class")  # Shared within class
def class_fixture():
    return {}

@pytest.fixture(scope="module")  # Shared within module
def module_fixture():
    return {}

@pytest.fixture(scope="session")  # Shared across all tests
def session_fixture():
    return {}
```

### Parametrize

```python
@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300)
])
def test_add_parametrized(a, b, expected):
    """Test with multiple input sets"""
    assert add(a, b) == expected

@pytest.mark.parametrize("input,expected", [
    ("hello", "HELLO"),
    ("World", "WORLD"),
    ("123", "123"),
    ("", "")
])
def test_uppercase(input, expected):
    """Test string uppercase"""
    assert input.upper() == expected

# Complex parametrize
@pytest.mark.parametrize("user_data,should_pass", [
    ({"username": "john", "age": 25}, True),
    ({"username": "a", "age": 25}, False),  # Username too short
    ({"username": "john", "age": 15}, False),  # Age too low
    ({"username": "john", "age": 200}, False),  # Age too high
])
def test_user_validation(user_data, should_pass):
    """Test user validation with multiple cases"""
    validator = UserValidator()
    result = validator.validate(user_data)
    assert result == should_pass
```

---

## Testing FastAPI

### TestClient

```python
from fastapi.testclient import TestClient
from myapp import app

client = TestClient(app)

def test_read_main():
    """Test GET endpoint"""
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

def test_create_user():
    """Test POST endpoint"""
    response = client.post(
        "/users",
        json={"username": "john", "email": "john@example.com"}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["username"] == "john"
    assert "id" in data

def test_authentication_required():
    """Test protected endpoint"""
    response = client.get("/protected")
    assert response.status_code == 401

    # With token
    token = "valid_token_here"
    response = client.get(
        "/protected",
        headers={"Authorization": f"Bearer {token}"}
    )
    assert response.status_code == 200
```

### Async Tests

```python
import pytest
from httpx import AsyncClient
from myapp import app

@pytest.mark.asyncio
async def test_async_endpoint():
    """Test async endpoint"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/async-endpoint")
        assert response.status_code == 200

@pytest.mark.asyncio
async def test_concurrent_requests():
    """Test multiple concurrent requests"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        tasks = [
            client.get(f"/users/{i}")
            for i in range(10)
        ]
        responses = await asyncio.gather(*tasks)

        assert all(r.status_code == 200 for r in responses)
```

### Dependency Override

```python
from fastapi import Depends

# Original dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Test dependency
def get_test_db():
    db = TestSessionLocal()
    try:
        yield db
    finally:
        db.close()

# Override for testing
@pytest.fixture
def client():
    app.dependency_overrides[get_db] = get_test_db

    with TestClient(app) as c:
        yield c

    app.dependency_overrides.clear()

def test_with_test_db(client):
    """Test uses test database"""
    response = client.get("/users")
    assert response.status_code == 200
```

---

## Mocking

### unittest.mock

```python
from unittest.mock import Mock, patch, MagicMock

# Mock object
def test_with_mock():
    """Test with mock object"""
    mock_db = Mock()
    mock_db.query.return_value.filter.return_value.first.return_value = User(
        id=1, username="john"
    )

    service = UserService(mock_db)
    user = service.get_user(1)

    assert user.username == "john"
    mock_db.query.assert_called_once()

# Patch decorator
@patch('myapp.services.external_api_call')
def test_with_patch(mock_api):
    """Patch external API call"""
    mock_api.return_value = {"status": "success"}

    result = process_data()

    assert result["status"] == "success"
    mock_api.assert_called_once()

# Patch multiple
@patch('myapp.services.redis_client')
@patch('myapp.services.database')
def test_with_multiple_patches(mock_db, mock_redis):
    """Patch multiple dependencies"""
    mock_redis.get.return_value = None
    mock_db.query.return_value.first.return_value = User(id=1)

    result = get_user_cached(1)

    assert result.id == 1
    mock_redis.get.assert_called_once()
    mock_db.query.assert_called_once()

# Context manager patch
def test_with_context_manager():
    """Patch using context manager"""
    with patch('myapp.services.send_email') as mock_email:
        mock_email.return_value = True

        user = create_user("john@example.com")

        mock_email.assert_called_with(
            to="john@example.com",
            subject="Welcome"
        )
```

### pytest-mock

```python
def test_with_mocker(mocker):
    """pytest-mock provides mocker fixture"""
    mock_func = mocker.patch('myapp.services.expensive_operation')
    mock_func.return_value = {"result": 42}

    result = my_function()

    assert result["result"] == 42
    mock_func.assert_called_once()

def test_spy(mocker):
    """Spy on real function"""
    spy = mocker.spy(math, 'sqrt')

    result = calculate_distance(3, 4)

    assert result == 5
    spy.assert_called_with(25)  # sqrt(3^2 + 4^2)
```

---

## Test Organization

### Project Structure

```
project/
├── myapp/
│   ├── __init__.py
│   ├── models.py
│   ├── services.py
│   └── api.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Shared fixtures
│   ├── unit/
│   │   ├── test_models.py
│   │   └── test_services.py
│   ├── integration/
│   │   ├── test_database.py
│   │   └── test_api_integration.py
│   └── e2e/
│       └── test_user_flow.py
└── pytest.ini
```

### conftest.py

```python
# tests/conftest.py
import pytest
from myapp import create_app
from myapp.database import Base, engine

@pytest.fixture(scope="session")
def app():
    """Create application for testing"""
    app = create_app(testing=True)
    return app

@pytest.fixture(scope="session")
def db():
    """Create test database"""
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)

@pytest.fixture
def db_session(db):
    """Create database session for test"""
    connection = db.connect()
    transaction = connection.begin()
    session = Session(bind=connection)

    yield session

    session.close()
    transaction.rollback()
    connection.close()

@pytest.fixture
def client(app):
    """Create test client"""
    return TestClient(app)
```

---

## Test Coverage

### Measuring Coverage

```bash
# Install coverage
pip install pytest-cov

# Run with coverage
pytest --cov=myapp tests/

# Generate HTML report
pytest --cov=myapp --cov-report=html tests/

# Coverage for specific module
pytest --cov=myapp.services tests/unit/test_services.py

# Fail if coverage below threshold
pytest --cov=myapp --cov-fail-under=80 tests/
```

### Coverage Configuration

```ini
# pytest.ini
[tool:pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    --cov=myapp
    --cov-report=html
    --cov-report=term-missing
    --cov-fail-under=80
    -v

# .coveragerc
[run]
source = myapp
omit =
    */tests/*
    */migrations/*
    */__pycache__/*

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
```

---

## Integration Testing

### Database Integration

```python
@pytest.fixture(scope="module")
def test_db():
    """Create test database"""
    engine = create_engine("postgresql://user:pass@localhost/testdb")
    Base.metadata.create_all(engine)

    Session = sessionmaker(bind=engine)
    session = Session()

    yield session

    session.close()
    Base.metadata.drop_all(engine)

def test_user_repository(test_db):
    """Test user repository with real database"""
    repo = UserRepository(test_db)

    # Create user
    user = repo.create(username="john", email="john@example.com")
    assert user.id is not None

    # Retrieve user
    found = repo.get_by_id(user.id)
    assert found.username == "john"

    # Update user
    repo.update(user.id, email="newemail@example.com")
    updated = repo.get_by_id(user.id)
    assert updated.email == "newemail@example.com"

    # Delete user
    repo.delete(user.id)
    deleted = repo.get_by_id(user.id)
    assert deleted is None
```

### External API Integration

```python
import responses

@responses.activate
def test_external_api():
    """Mock external API calls"""
    responses.add(
        responses.GET,
        "https://api.example.com/users/1",
        json={"id": 1, "name": "John"},
        status=200
    )

    result = fetch_user_from_api(1)

    assert result["name"] == "John"
    assert len(responses.calls) == 1

@responses.activate
def test_api_error_handling():
    """Test API error handling"""
    responses.add(
        responses.GET,
        "https://api.example.com/users/999",
        json={"error": "Not found"},
        status=404
    )

    with pytest.raises(UserNotFoundError):
        fetch_user_from_api(999)
```

---

## E2E Testing

### Playwright Example

```python
import pytest
from playwright.sync_api import sync_playwright

@pytest.fixture(scope="session")
def browser():
    """Create browser instance"""
    with sync_playwright() as p:
        browser = p.chromium.launch()
        yield browser
        browser.close()

def test_user_registration(browser):
    """Test complete user registration flow"""
    page = browser.new_page()

    # Navigate to registration page
    page.goto("http://localhost:8000/register")

    # Fill form
    page.fill("#username", "newuser")
    page.fill("#email", "newuser@example.com")
    page.fill("#password", "SecurePass123!")

    # Submit
    page.click("#submit-button")

    # Wait for redirect
    page.wait_for_url("http://localhost:8000/dashboard")

    # Verify success
    assert page.inner_text("h1") == "Welcome, newuser"

    page.close()

def test_login_flow(browser):
    """Test login flow"""
    page = browser.new_page()

    page.goto("http://localhost:8000/login")
    page.fill("#username", "existinguser")
    page.fill("#password", "password123")
    page.click("#login-button")

    page.wait_for_selector("#user-menu")
    assert page.is_visible("#user-menu")

    page.close()
```

---

## Best Practices

### ✅ Do's:

1. **Write tests first** (TDD when appropriate)
2. **Test one thing** per test
3. **Use descriptive names** for tests
4. **Follow AAA pattern** (Arrange, Act, Assert)
5. **Keep tests independent**
6. **Use fixtures** for common setup
7. **Mock external dependencies**
8. **Aim for high coverage** (>80%)
9. **Run tests in CI/CD**

### ❌ Don'ts:

1. **Don't test implementation** details
2. **Don't write flaky tests**
3. **Don't share state** between tests
4. **Don't skip error cases**
5. **Don't test framework** code
6. **Don't ignore slow tests**
7. **Don't commit failing tests**

---

## Interview Questions

### Q1: What is the testing pyramid?

**Answer**: Testing strategy with three layers:

- **Unit tests** (70%): Fast, isolated, test single units
- **Integration tests** (20%): Test component interactions
- **E2E tests** (10%): Test complete user flows
  More unit tests, fewer E2E tests for speed and maintainability.

### Q2: What's the difference between mock, stub, and spy?

**Answer**:

- **Mock**: Fake object with assertions on calls
- **Stub**: Fake object with predefined responses
- **Spy**: Real object with call tracking
  Mocks verify behavior, stubs provide data, spies observe.

### Q3: How do you test async code?

**Answer**:

- Use `@pytest.mark.asyncio` decorator
- Use AsyncClient for FastAPI
- Use `asyncio.gather()` for concurrent operations
- Mock async functions with `async def`

### Q4: What is test coverage and what's a good target?

**Answer**: Percentage of code executed by tests:

- **80-90%** is good target for most projects
- **100%** is impractical and not necessary
- Focus on critical paths and business logic
- Coverage ≠ quality, still need good assertions

### Q5: How do you test database code?

**Answer**:

- Use test database (in-memory SQLite or separate instance)
- Rollback transactions after each test
- Use fixtures for common data setup
- Test repository layer, not ORM itself
- Mock database for unit tests, real DB for integration

---

## Summary

Testing essentials:

- **pytest** for test framework
- **Fixtures** for setup/teardown
- **Mocking** for external dependencies
- **TestClient** for FastAPI
- **Coverage** >80% target
- **AAA pattern** (Arrange, Act, Assert)
- **Test pyramid** (more unit, fewer E2E)

Good tests = confident code changes! 🧪
