# 🧪 Mocking and Patching

## Overview

Testing strategies using mocks, stubs, and patches to isolate code and test without external dependencies.

---

## Why Mocking?

```python
"""
Mocking Benefits:
1. Isolate unit under test
2. No external dependencies (DB, API, filesystem)
3. Fast tests (no I/O)
4. Control behavior (simulate errors, edge cases)
5. Verify interactions (calls, arguments)

When to Mock:
✅ External APIs
✅ Database calls
✅ File system operations
✅ Time-dependent code
✅ Random behavior
✅ Slow operations

When NOT to Mock:
❌ Simple functions (test directly)
❌ Data structures (use real objects)
❌ Your own code being tested
❌ Everything (integration tests needed too)
"""
```

---

## unittest.mock Basics

### Mock vs MagicMock

```python
from unittest.mock import Mock, MagicMock

# Mock: Basic mock object
mock = Mock()

# Can call with any arguments
result = mock(1, 2, foo='bar')

# Can access any attribute
value = mock.some_attribute

# Configure return value
mock.return_value = 42
assert mock() == 42

# Configure side effect (exception)
mock.side_effect = ValueError("Error!")
try:
    mock()
except ValueError as e:
    assert str(e) == "Error!"

# Verify calls
mock.assert_called()
mock.assert_called_once()
mock.assert_called_with(1, 2, foo='bar')

# MagicMock: Includes magic methods
magic_mock = MagicMock()

# Supports magic methods like __len__, __iter__, etc.
len(magic_mock)  # Works!
for item in magic_mock:  # Works!
    pass

# When to use:
# - Mock: Simple replacements
# - MagicMock: Need magic methods (len, iter, context manager)
```

### Basic Example

```python
import pytest
from unittest.mock import Mock
import requests

# Code to test
def get_user_data(user_id: int) -> dict:
    """Fetch user data from API"""
    response = requests.get(f'https://api.example.com/users/{user_id}')
    response.raise_for_status()
    return response.json()

# Test with mock
def test_get_user_data():
    """Test without real API call"""
    # Create mock response
    mock_response = Mock()
    mock_response.json.return_value = {
        'id': 1,
        'name': 'John Doe',
        'email': 'john@example.com'
    }
    mock_response.raise_for_status.return_value = None

    # Mock requests.get
    requests.get = Mock(return_value=mock_response)

    # Test
    result = get_user_data(1)

    # Assertions
    assert result['name'] == 'John Doe'
    requests.get.assert_called_once_with('https://api.example.com/users/1')
```

---

## Patching

### patch Decorator

```python
from unittest.mock import patch
import requests

# Code to test
class UserService:
    """User service fetching from API"""

    def get_user(self, user_id: int) -> dict:
        """Get user by ID"""
        response = requests.get(f'https://api.example.com/users/{user_id}')
        response.raise_for_status()
        return response.json()

    def get_active_users(self) -> list:
        """Get all active users"""
        response = requests.get('https://api.example.com/users?status=active')
        response.raise_for_status()
        return response.json()

# Test with patch decorator
@patch('requests.get')
def test_get_user(mock_get):
    """
    Patch requests.get for test duration

    mock_get is automatically injected as parameter.
    """
    # Configure mock
    mock_response = Mock()
    mock_response.json.return_value = {'id': 1, 'name': 'Alice'}
    mock_get.return_value = mock_response

    # Test
    service = UserService()
    user = service.get_user(1)

    # Assertions
    assert user['name'] == 'Alice'
    mock_get.assert_called_once_with('https://api.example.com/users/1')

# Multiple patches
@patch('requests.get')
@patch('time.sleep')  # Also mock sleep to speed up tests
def test_with_multiple_patches(mock_sleep, mock_get):
    """
    Multiple patches

    Note: Decorators applied bottom-up, parameters top-down:
    - @patch('time.sleep') -> mock_sleep (last param)
    - @patch('requests.get') -> mock_get (first param)
    """
    mock_response = Mock()
    mock_response.json.return_value = [{'id': 1}]
    mock_get.return_value = mock_response

    service = UserService()
    users = service.get_active_users()

    assert len(users) == 1
    mock_sleep.assert_not_called()  # Verify sleep not called
```

### patch Context Manager

```python
from unittest.mock import patch

def test_with_context_manager():
    """
    Use patch as context manager

    More explicit scope control.
    """
    service = UserService()

    with patch('requests.get') as mock_get:
        # Mock only within this block
        mock_response = Mock()
        mock_response.json.return_value = {'id': 1, 'name': 'Bob'}
        mock_get.return_value = mock_response

        user = service.get_user(1)
        assert user['name'] == 'Bob'

    # Outside with block, patch no longer active
```

### patch.object

```python
from unittest.mock import patch

class EmailService:
    """Email service"""

    def send_email(self, to: str, subject: str, body: str):
        """Send email via SMTP"""
        # Complex SMTP logic
        pass

class UserRegistration:
    """User registration"""

    def __init__(self, email_service: EmailService):
        self.email_service = email_service

    def register_user(self, username: str, email: str) -> dict:
        """Register user and send welcome email"""
        # Create user
        user = {'username': username, 'email': email}

        # Send welcome email
        self.email_service.send_email(
            to=email,
            subject='Welcome!',
            body=f'Welcome {username}!'
        )

        return user

def test_register_user():
    """
    Test user registration

    Mock email_service.send_email method.
    """
    email_service = EmailService()
    registration = UserRegistration(email_service)

    # Patch specific method
    with patch.object(email_service, 'send_email') as mock_send:
        user = registration.register_user('alice', 'alice@example.com')

        # Verify email sent
        assert user['username'] == 'alice'
        mock_send.assert_called_once_with(
            to='alice@example.com',
            subject='Welcome!',
            body='Welcome alice!'
        )
```

---

## Advanced Mocking Techniques

### side_effect for Multiple Calls

```python
from unittest.mock import Mock

def test_side_effect_list():
    """
    side_effect with list: Different return for each call
    """
    mock = Mock()
    mock.side_effect = [1, 2, 3]

    assert mock() == 1
    assert mock() == 2
    assert mock() == 3

    # Fourth call raises StopIteration
    with pytest.raises(StopIteration):
        mock()

def test_side_effect_exception():
    """
    side_effect with exception

    Useful for testing error handling.
    """
    mock = Mock()
    mock.side_effect = ConnectionError("Connection failed")

    with pytest.raises(ConnectionError):
        mock()

def test_side_effect_function():
    """
    side_effect with function

    Custom logic for each call.
    """
    def custom_logic(x):
        if x < 0:
            raise ValueError("Negative not allowed")
        return x * 2

    mock = Mock(side_effect=custom_logic)

    assert mock(5) == 10

    with pytest.raises(ValueError):
        mock(-1)
```

### spec and spec_set

```python
from unittest.mock import Mock

class UserRepository:
    """User repository"""

    def get_user(self, user_id: int) -> dict:
        """Get user by ID"""
        pass

    def save_user(self, user: dict):
        """Save user"""
        pass

def test_without_spec():
    """
    Without spec: Can call any method (dangerous!)
    """
    mock_repo = Mock()

    # Typo! Should be get_user
    result = mock_repo.get_usr(1)  # No error! Returns a Mock

    # Test passes but wrong method called
    assert result is not None  # True, but misleading

def test_with_spec():
    """
    With spec: Only allows methods from spec class
    """
    mock_repo = Mock(spec=UserRepository)

    # This works (method exists)
    mock_repo.get_user.return_value = {'id': 1}
    user = mock_repo.get_user(1)
    assert user['id'] == 1

    # This raises AttributeError (typo caught!)
    with pytest.raises(AttributeError):
        mock_repo.get_usr(1)

def test_with_spec_set():
    """
    spec_set: Also prevents setting attributes not in spec
    """
    mock_repo = Mock(spec_set=UserRepository)

    # Can't add new attributes
    with pytest.raises(AttributeError):
        mock_repo.new_method = Mock()
```

### Mock Configuration

```python
from unittest.mock import Mock

def test_mock_configuration():
    """Configure mock with multiple attributes"""
    mock_user = Mock(
        id=1,
        name='Alice',
        email='alice@example.com',
        is_active=True,
        get_full_name=Mock(return_value='Alice Smith')
    )

    assert mock_user.id == 1
    assert mock_user.name == 'Alice'
    assert mock_user.get_full_name() == 'Alice Smith'

def test_configure_mock():
    """Use configure_mock for dynamic configuration"""
    mock = Mock()

    mock.configure_mock(
        return_value=42,
        side_effect=None,
        attribute1='value1',
        attribute2='value2'
    )

    assert mock() == 42
    assert mock.attribute1 == 'value1'
```

---

## Real-World Examples

### Testing Database Code

```python
from unittest.mock import Mock, patch
import pytest

class UserRepository:
    """User repository"""

    def __init__(self, db):
        self.db = db

    def get_user(self, user_id: int) -> dict:
        """Get user from database"""
        cursor = self.db.cursor()
        cursor.execute('SELECT * FROM users WHERE id = ?', (user_id,))
        row = cursor.fetchone()

        if not row:
            return None

        return {
            'id': row[0],
            'username': row[1],
            'email': row[2]
        }

    def create_user(self, username: str, email: str) -> dict:
        """Create user"""
        cursor = self.db.cursor()
        cursor.execute(
            'INSERT INTO users (username, email) VALUES (?, ?)',
            (username, email)
        )
        self.db.commit()

        return {
            'id': cursor.lastrowid,
            'username': username,
            'email': email
        }

def test_get_user():
    """
    Test get_user without real database
    """
    # Mock database
    mock_db = Mock()
    mock_cursor = Mock()

    # Configure cursor to return user data
    mock_cursor.fetchone.return_value = (1, 'alice', 'alice@example.com')
    mock_db.cursor.return_value = mock_cursor

    # Test
    repo = UserRepository(mock_db)
    user = repo.get_user(1)

    # Assertions
    assert user['username'] == 'alice'
    assert user['email'] == 'alice@example.com'

    # Verify SQL executed correctly
    mock_cursor.execute.assert_called_once_with(
        'SELECT * FROM users WHERE id = ?',
        (1,)
    )

def test_get_user_not_found():
    """Test when user doesn't exist"""
    mock_db = Mock()
    mock_cursor = Mock()
    mock_cursor.fetchone.return_value = None
    mock_db.cursor.return_value = mock_cursor

    repo = UserRepository(mock_db)
    user = repo.get_user(999)

    assert user is None

def test_create_user():
    """Test user creation"""
    mock_db = Mock()
    mock_cursor = Mock()
    mock_cursor.lastrowid = 42
    mock_db.cursor.return_value = mock_cursor

    repo = UserRepository(mock_db)
    user = repo.create_user('bob', 'bob@example.com')

    assert user['id'] == 42
    assert user['username'] == 'bob'

    # Verify INSERT executed
    mock_cursor.execute.assert_called_once_with(
        'INSERT INTO users (username, email) VALUES (?, ?)',
        ('bob', 'bob@example.com')
    )

    # Verify transaction committed
    mock_db.commit.assert_called_once()
```

### Testing External API

```python
from unittest.mock import patch, Mock
import pytest
import requests

class WeatherService:
    """Weather service fetching from API"""

    API_URL = 'https://api.weather.com/current'

    def __init__(self, api_key: str):
        self.api_key = api_key

    def get_temperature(self, city: str) -> float:
        """Get current temperature for city"""
        try:
            response = requests.get(
                self.API_URL,
                params={'city': city, 'api_key': self.api_key},
                timeout=5
            )
            response.raise_for_status()
            data = response.json()
            return data['temperature']

        except requests.Timeout:
            raise TimeoutError(f"Request timeout for {city}")

        except requests.HTTPError as e:
            if e.response.status_code == 404:
                raise ValueError(f"City not found: {city}")
            raise

@patch('requests.get')
def test_get_temperature_success(mock_get):
    """Test successful temperature fetch"""
    # Configure mock response
    mock_response = Mock()
    mock_response.json.return_value = {'temperature': 22.5}
    mock_get.return_value = mock_response

    # Test
    service = WeatherService(api_key='test-key')
    temp = service.get_temperature('London')

    # Assertions
    assert temp == 22.5
    mock_get.assert_called_once_with(
        'https://api.weather.com/current',
        params={'city': 'London', 'api_key': 'test-key'},
        timeout=5
    )

@patch('requests.get')
def test_get_temperature_timeout(mock_get):
    """Test timeout handling"""
    mock_get.side_effect = requests.Timeout()

    service = WeatherService(api_key='test-key')

    with pytest.raises(TimeoutError, match="Request timeout"):
        service.get_temperature('London')

@patch('requests.get')
def test_get_temperature_city_not_found(mock_get):
    """Test 404 error handling"""
    mock_response = Mock()
    mock_response.status_code = 404
    mock_response.raise_for_status.side_effect = requests.HTTPError(response=mock_response)
    mock_get.return_value = mock_response

    service = WeatherService(api_key='test-key')

    with pytest.raises(ValueError, match="City not found"):
        service.get_temperature('InvalidCity')
```

### Testing with Fixtures

```python
import pytest
from unittest.mock import Mock, patch

@pytest.fixture
def mock_db():
    """Fixture for mock database"""
    db = Mock()
    cursor = Mock()
    db.cursor.return_value = cursor
    return db

@pytest.fixture
def user_repository(mock_db):
    """Fixture for user repository with mock DB"""
    return UserRepository(mock_db)

def test_with_fixtures(user_repository, mock_db):
    """
    Test using fixtures

    Cleaner than creating mocks in each test.
    """
    mock_cursor = mock_db.cursor.return_value
    mock_cursor.fetchone.return_value = (1, 'alice', 'alice@example.com')

    user = user_repository.get_user(1)

    assert user['username'] == 'alice'

# Fixture with patch
@pytest.fixture
def mock_requests_get():
    """Fixture that patches requests.get"""
    with patch('requests.get') as mock:
        yield mock

def test_with_patch_fixture(mock_requests_get):
    """Use patched requests.get from fixture"""
    mock_response = Mock()
    mock_response.json.return_value = {'data': 'test'}
    mock_requests_get.return_value = mock_response

    # Test code using requests.get
    pass
```

---

## Best Practices

### ✅ Do's:

1. **Mock external dependencies** - APIs, databases, filesystem
2. **Use spec** - Catch typos and wrong method names
3. **Verify calls** - assert_called_with ensures correct usage
4. **Test error paths** - Use side_effect for exceptions
5. **Keep mocks simple** - Don't recreate entire system
6. **Use fixtures** - Reuse mock setup across tests
7. **Name descriptively** - mock_user_repo, not mock1

### ❌ Don'ts:

1. **Don't mock everything** - Test real code when possible
2. **Don't over-specify** - Avoid brittle tests checking every call
3. **Don't test implementation** - Test behavior, not internals
4. **Don't mock what you own** - Mock external dependencies only
5. **Don't forget to assert** - Mock calls without assertions useless

---

## Interview Questions

### Q1: What's difference between Mock and MagicMock?

**Answer**:

- **Mock**: Basic mock object
- **MagicMock**: Includes magic methods (**len**, **iter**, **enter**, etc.)
- **Use Mock**: Simple replacements
- **Use MagicMock**: Need magic methods or context managers
- **Default**: MagicMock safer, Mock faster
  Most cases MagicMock recommended.

### Q2: When should you use spec?

**Answer**:

- **Always**: Prevents attribute errors
- **spec**: Limits to methods/attributes of class
- **spec_set**: Also prevents setting new attributes
- **Benefits**: Catch typos, enforce interface
- **Example**: mock = Mock(spec=UserRepository)
  Catches bugs at test time, not runtime.

### Q3: patch decorator vs context manager?

**Answer**:

- **Decorator**: Clean for entire test, parameters
- **Context manager**: Explicit scope, partial test
- **Multiple patches**: Decorator cleaner
- **Temporary mock**: Context manager better
- **Choice**: Decorator for most cases, context for fine control

### Q4: How to test error conditions?

**Answer**:

- **side_effect**: Set to exception instance
- **Example**: mock.side_effect = ValueError("Error")
- **Multiple**: List for different calls
- **Function**: Dynamic error logic
- **Verify**: with pytest.raises(ValueError)
  Essential for testing error handling.

### Q5: What are common mocking mistakes?

**Answer**:

- **Over-mocking**: Mocking too much, no real code tested
- **Wrong path**: Patch where used, not where defined
- **No assertions**: Mock without verifying calls
- **Testing mocks**: Testing mock behavior, not real code
- **Brittle tests**: Too specific, break on refactor
  Mock external dependencies only.

---

## Summary

Mocking essentials:

- **Mock**: Basic object, MagicMock with magic methods
- **patch**: Replace dependencies (decorator/context manager)
- **spec**: Limit to real interface
- **side_effect**: Multiple returns, exceptions, custom logic
- **Assertions**: Verify calls and arguments
- **Best practice**: Mock external dependencies, keep simple

Test behavior, not implementation! 🧪
