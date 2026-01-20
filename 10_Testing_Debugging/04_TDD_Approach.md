# 🧪 Test-Driven Development (TDD)

## Overview

Test-Driven Development is a software development approach where tests are written before the actual code, driving the design and implementation.

---

## TDD Cycle: Red-Green-Refactor

### The Three Phases

```python
"""
TDD Cycle:

1. RED: Write a failing test
   - Write test for new feature
   - Test must fail (feature doesn't exist yet)
   - Confirms test is actually testing something

2. GREEN: Make it pass
   - Write minimal code to pass test
   - Don't worry about perfection
   - Just make the test green

3. REFACTOR: Clean up code
   - Improve code quality
   - Remove duplication
   - Maintain passing tests

Repeat for each new feature or requirement.
"""
```

---

## TDD Example: Building a Calculator

### Step 1: Red - Write Failing Test

```python
# test_calculator.py
import pytest
from calculator import Calculator

class TestCalculator:
    """Test calculator functionality using TDD"""

    def test_add_two_numbers(self):
        """Test addition of two numbers"""
        calc = Calculator()
        result = calc.add(2, 3)
        assert result == 5

# Running test: FAILS (Calculator doesn't exist yet)
# ModuleNotFoundError: No module named 'calculator'
```

### Step 2: Green - Make It Pass

```python
# calculator.py
class Calculator:
    """Simple calculator"""

    def add(self, a, b):
        """Add two numbers"""
        return a + b

# Running test: PASSES ✓
```

### Step 3: Refactor - Improve Code

```python
# calculator.py (refactored with type hints and validation)
from typing import Union

class Calculator:
    """Simple calculator with type safety"""

    def add(self, a: Union[int, float], b: Union[int, float]) -> Union[int, float]:
        """
        Add two numbers

        Args:
            a: First number
            b: Second number

        Returns:
            Sum of a and b
        """
        if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
            raise TypeError("Arguments must be numbers")

        return a + b

# Running test: Still PASSES ✓
```

### Continue TDD Cycle: Add More Features

```python
# test_calculator.py
class TestCalculator:
    """Test calculator functionality"""

    @pytest.fixture
    def calculator(self):
        """Create calculator instance"""
        return Calculator()

    # RED: Test for subtraction
    def test_subtract_two_numbers(self, calculator):
        """Test subtraction"""
        result = calculator.subtract(5, 3)
        assert result == 2

    # RED: Test for multiplication
    def test_multiply_two_numbers(self, calculator):
        """Test multiplication"""
        result = calculator.multiply(4, 3)
        assert result == 12

    # RED: Test for division
    def test_divide_two_numbers(self, calculator):
        """Test division"""
        result = calculator.divide(10, 2)
        assert result == 5

    # RED: Test division by zero
    def test_divide_by_zero_raises_error(self, calculator):
        """Test division by zero raises ValueError"""
        with pytest.raises(ValueError, match="Cannot divide by zero"):
            calculator.divide(10, 0)

    # RED: Test invalid input types
    def test_add_invalid_types_raises_error(self, calculator):
        """Test invalid input raises TypeError"""
        with pytest.raises(TypeError):
            calculator.add("2", 3)

# GREEN: Implement features to pass tests
# calculator.py
class Calculator:
    """Calculator with basic operations"""

    def add(self, a: Union[int, float], b: Union[int, float]) -> Union[int, float]:
        """Add two numbers"""
        self._validate_numbers(a, b)
        return a + b

    def subtract(self, a: Union[int, float], b: Union[int, float]) -> Union[int, float]:
        """Subtract b from a"""
        self._validate_numbers(a, b)
        return a - b

    def multiply(self, a: Union[int, float], b: Union[int, float]) -> Union[int, float]:
        """Multiply two numbers"""
        self._validate_numbers(a, b)
        return a * b

    def divide(self, a: Union[int, float], b: Union[int, float]) -> float:
        """Divide a by b"""
        self._validate_numbers(a, b)

        if b == 0:
            raise ValueError("Cannot divide by zero")

        return a / b

    def _validate_numbers(self, a, b):
        """Validate that inputs are numbers"""
        if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
            raise TypeError("Arguments must be numbers")
```

---

## TDD for Real-World Features

### Example: User Registration

```python
# test_user_service.py
import pytest
from user_service import UserService
from models import User
from exceptions import ValidationError, DuplicateUserError

class TestUserRegistration:
    """Test user registration using TDD"""

    @pytest.fixture
    def user_service(self, db_session):
        """Create user service"""
        return UserService(db_session)

    # RED: Test successful registration
    def test_register_user_success(self, user_service):
        """Test successful user registration"""
        user_data = {
            'username': 'johndoe',
            'email': 'john@example.com',
            'password': 'SecurePass123!'
        }

        user = user_service.register(user_data)

        assert user.id is not None
        assert user.username == 'johndoe'
        assert user.email == 'john@example.com'
        # Password should be hashed
        assert user.password != 'SecurePass123!'
        assert len(user.password) > 50  # Hashed password length

    # RED: Test duplicate username
    def test_register_duplicate_username_raises_error(self, user_service):
        """Test duplicate username raises error"""
        user_data = {
            'username': 'johndoe',
            'email': 'john@example.com',
            'password': 'SecurePass123!'
        }

        user_service.register(user_data)

        # Try to register with same username
        duplicate_data = {
            'username': 'johndoe',
            'email': 'different@example.com',
            'password': 'SecurePass123!'
        }

        with pytest.raises(DuplicateUserError, match="Username already exists"):
            user_service.register(duplicate_data)

    # RED: Test invalid email
    def test_register_invalid_email_raises_error(self, user_service):
        """Test invalid email raises error"""
        user_data = {
            'username': 'johndoe',
            'email': 'invalid-email',
            'password': 'SecurePass123!'
        }

        with pytest.raises(ValidationError, match="Invalid email"):
            user_service.register(user_data)

    # RED: Test weak password
    def test_register_weak_password_raises_error(self, user_service):
        """Test weak password raises error"""
        user_data = {
            'username': 'johndoe',
            'email': 'john@example.com',
            'password': '123'  # Too short, no special chars
        }

        with pytest.raises(ValidationError, match="Password too weak"):
            user_service.register(user_data)

    # RED: Test missing required fields
    @pytest.mark.parametrize("missing_field", ['username', 'email', 'password'])
    def test_register_missing_field_raises_error(self, user_service, missing_field):
        """Test missing required field raises error"""
        user_data = {
            'username': 'johndoe',
            'email': 'john@example.com',
            'password': 'SecurePass123!'
        }

        del user_data[missing_field]

        with pytest.raises(ValidationError, match=f"{missing_field} is required"):
            user_service.register(user_data)

# GREEN: Implement to pass tests
# user_service.py
import re
from argon2 import PasswordHasher
from models import User
from exceptions import ValidationError, DuplicateUserError

class UserService:
    """User management service"""

    def __init__(self, db_session):
        self.db = db_session
        self.password_hasher = PasswordHasher()

    def register(self, user_data: dict) -> User:
        """
        Register new user

        Args:
            user_data: Dictionary with username, email, password

        Returns:
            Created user instance

        Raises:
            ValidationError: If data is invalid
            DuplicateUserError: If username/email exists
        """
        # Validate required fields
        self._validate_required_fields(user_data)

        # Validate email format
        self._validate_email(user_data['email'])

        # Validate password strength
        self._validate_password(user_data['password'])

        # Check for duplicates
        self._check_duplicates(user_data['username'], user_data['email'])

        # Hash password
        hashed_password = self.password_hasher.hash(user_data['password'])

        # Create user
        user = User(
            username=user_data['username'],
            email=user_data['email'],
            password=hashed_password
        )

        self.db.add(user)
        self.db.commit()

        return user

    def _validate_required_fields(self, data: dict):
        """Validate required fields present"""
        required = ['username', 'email', 'password']

        for field in required:
            if field not in data or not data[field]:
                raise ValidationError(f"{field} is required")

    def _validate_email(self, email: str):
        """Validate email format"""
        pattern = r'^[\w\.-]+@[\w\.-]+\.\w+$'

        if not re.match(pattern, email):
            raise ValidationError("Invalid email format")

    def _validate_password(self, password: str):
        """Validate password strength"""
        if len(password) < 8:
            raise ValidationError("Password too weak: minimum 8 characters")

        if not re.search(r'[A-Z]', password):
            raise ValidationError("Password too weak: needs uppercase letter")

        if not re.search(r'[0-9]', password):
            raise ValidationError("Password too weak: needs digit")

        if not re.search(r'[!@#$%^&*]', password):
            raise ValidationError("Password too weak: needs special character")

    def _check_duplicates(self, username: str, email: str):
        """Check for duplicate username or email"""
        existing = self.db.query(User).filter(
            (User.username == username) | (User.email == email)
        ).first()

        if existing:
            if existing.username == username:
                raise DuplicateUserError("Username already exists")
            else:
                raise DuplicateUserError("Email already exists")
```

---

## TDD Benefits and Trade-offs

### Benefits

```python
"""
✅ Benefits of TDD:

1. Better Design:
   - Forces you to think about API before implementation
   - Leads to more testable, modular code
   - Identifies design issues early

2. Living Documentation:
   - Tests document how code should be used
   - Always up-to-date (unlike comments)
   - Examples of usage patterns

3. Confidence in Changes:
   - Refactor without fear
   - Catch regressions immediately
   - Safe to modify code

4. Fewer Bugs:
   - Find bugs early in development
   - Edge cases considered upfront
   - Less debugging time

5. Better Coverage:
   - Natural 100% coverage
   - Tests written for all paths
   - No "we'll test it later"
"""
```

### Trade-offs

```python
"""
⚠️ Trade-offs of TDD:

1. Initial Slowdown:
   - Takes longer to write tests first
   - Learning curve for new developers
   - May feel unproductive initially

2. Maintenance:
   - More code to maintain (tests + implementation)
   - Brittle tests can slow development
   - Tests need refactoring too

3. Not Always Suitable:
   - UI/UX development (hard to test)
   - Exploratory prototyping
   - Unknown requirements

4. Over-testing:
   - Can lead to testing implementation details
   - Tests become coupled to code
   - Hard to refactor

When to use TDD:
- Well-defined requirements
- Business logic and algorithms
- API development
- Bug fixes (write test first)

When to skip:
- Rapid prototyping
- UI experimentation
- Learning new technologies
"""
```

---

## TDD Best Practices

### ✅ Do's

```python
"""
1. Start with simplest test
   - Begin with happy path
   - Add edge cases gradually
   - One test at a time

2. Write minimal code to pass
   - Don't over-engineer
   - Solve the problem at hand
   - Refactor later

3. Test behavior, not implementation
   - Test what code does, not how
   - Avoid testing private methods
   - Focus on public API

4. Keep tests independent
   - Each test should run alone
   - No shared state between tests
   - Use fixtures for setup

5. Use descriptive test names
   - test_should_return_error_when_email_invalid
   - Clear what's being tested
   - Acts as documentation
"""

# Good: Test behavior
def test_user_cannot_withdraw_more_than_balance():
    """Test withdrawal validation"""
    account = BankAccount(balance=100)

    with pytest.raises(InsufficientFundsError):
        account.withdraw(150)

# Bad: Test implementation
def test_withdraw_calls_validate_balance():
    """Don't test internal methods"""
    account = BankAccount(balance=100)

    # Coupled to implementation details
    assert account._validate_balance(150) == False
```

### ❌ Don'ts

```python
"""
1. Don't write tests after code
   - Defeats purpose of TDD
   - Biased by implementation
   - Miss design benefits

2. Don't test everything
   - Don't test framework code
   - Don't test third-party libraries
   - Focus on your logic

3. Don't make tests complex
   - Tests should be simple
   - Avoid logic in tests
   - If test is complex, refactor code

4. Don't ignore failing tests
   - Fix immediately
   - Don't commit broken tests
   - Don't skip tests

5. Don't couple tests to implementation
   - Avoid mocking everything
   - Test public interface
   - Allow implementation changes
"""

# Bad: Over-mocked test
def test_process_order_with_too_many_mocks():
    """Too many mocks = brittle test"""
    db = Mock()
    payment = Mock()
    email = Mock()
    inventory = Mock()
    logger = Mock()

    # Test becomes maintenance nightmare
    service = OrderService(db, payment, email, inventory, logger)
    service.process_order(order_id=1)

    # Assertions on mocks
    db.get_order.assert_called_once()
    payment.charge.assert_called_once()
    # ... many more assertions

# Good: Integration test
def test_process_order_integration():
    """Test with real components"""
    service = OrderService(db_session=db, payment_gateway=test_payment)

    order = service.process_order(order_id=1)

    assert order.status == 'completed'
    assert order.payment_status == 'paid'
```

---

## TDD with FastAPI

```python
# test_api.py
import pytest
from fastapi.testclient import TestClient
from main import app

class TestUserAPI:
    """Test user API endpoints with TDD"""

    @pytest.fixture
    def client(self):
        """Create test client"""
        return TestClient(app)

    # RED: Test create user endpoint
    def test_create_user_returns_201(self, client):
        """Test successful user creation"""
        response = client.post('/users/', json={
            'username': 'johndoe',
            'email': 'john@example.com',
            'password': 'SecurePass123!'
        })

        assert response.status_code == 201
        data = response.json()
        assert data['username'] == 'johndoe'
        assert data['email'] == 'john@example.com'
        assert 'password' not in data  # Don't expose password
        assert 'id' in data

    # RED: Test validation error
    def test_create_user_invalid_email_returns_422(self, client):
        """Test invalid email returns validation error"""
        response = client.post('/users/', json={
            'username': 'johndoe',
            'email': 'invalid-email',
            'password': 'SecurePass123!'
        })

        assert response.status_code == 422
        data = response.json()
        assert 'detail' in data
        assert any('email' in str(error).lower() for error in data['detail'])

    # RED: Test duplicate user
    def test_create_duplicate_user_returns_409(self, client):
        """Test duplicate username returns conflict"""
        # Create first user
        client.post('/users/', json={
            'username': 'johndoe',
            'email': 'john@example.com',
            'password': 'SecurePass123!'
        })

        # Try to create duplicate
        response = client.post('/users/', json={
            'username': 'johndoe',
            'email': 'different@example.com',
            'password': 'SecurePass123!'
        })

        assert response.status_code == 409
        data = response.json()
        assert 'already exists' in data['detail'].lower()

# GREEN: Implement endpoint
# main.py
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, EmailStr
from user_service import UserService
from exceptions import ValidationError, DuplicateUserError

app = FastAPI()

class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    id: int
    username: str
    email: str

@app.post('/users/', response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(user_data: UserCreate):
    """Create new user"""
    try:
        user = user_service.register(user_data.dict())
        return user

    except ValidationError as e:
        raise HTTPException(status_code=422, detail=str(e))

    except DuplicateUserError as e:
        raise HTTPException(status_code=409, detail=str(e))
```

---

## Interview Questions

### Q1: What is TDD and its cycle?

**Answer**: Test-Driven Development writes tests before code:

- **Red**: Write failing test for new feature
- **Green**: Write minimal code to pass test
- **Refactor**: Clean up code while keeping tests green
- **Benefits**: Better design, living documentation, confidence
- **When**: Well-defined requirements, business logic

### Q2: TDD vs writing tests after?

**Answer**:

- **TDD**: Tests drive design, think about API first
- **After**: Tests verify implementation, biased by code
- **TDD benefits**: Better design, testable code, fewer bugs
- **After**: Can miss edge cases, implementation-focused
  TDD influences architecture positively.

### Q3: How to test private methods?

**Answer**:

- **Don't test private methods** directly
- **Test through public API**: Private methods called by public
- **If needed**: Extract to separate class, make public
- **Why**: Tests should verify behavior, not implementation
  Focus on what code does, not how.

### Q4: What makes a good test?

**Answer**:

- **Independent**: Can run alone, no shared state
- **Repeatable**: Same result every time
- **Fast**: Quick feedback loop
- **Self-validating**: Pass or fail, no manual check
- **Timely**: Written at right time (before code in TDD)
  Use FIRST principles.

### Q5: How much to test?

**Answer**:

- **Test**: Business logic, algorithms, edge cases
- **Don't test**: Framework code, third-party libraries, getters/setters
- **Coverage**: Aim for high coverage, but not 100% religion
- **Focus**: Critical paths, complex logic, bug-prone areas
  Quality over quantity.

---

## Summary

TDD essentials:

- **Red-Green-Refactor**: Core TDD cycle
- **Tests first**: Drive design with tests
- **Simple tests**: Start small, add complexity
- **Behavior**: Test what, not how
- **Living docs**: Tests document usage
- **Confidence**: Refactor without fear
- **Practice**: Gets easier with experience

Write tests first, code follows! 🧪
