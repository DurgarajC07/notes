# 📐 PEP 8 Style Guide & Type Hints

## Overview

Writing clean, consistent Python code following PEP 8 standards and using type hints for better code quality and maintainability.

---

## PEP 8 Naming Conventions

### Naming Standards

```python
"""
PEP 8 Naming Conventions:

1. module_name.py           - lowercase with underscores
2. ClassName               - CapWords (PascalCase)
3. function_name()         - lowercase with underscores
4. variable_name           - lowercase with underscores
5. CONSTANT_NAME           - uppercase with underscores
6. _private_method()       - leading underscore
7. __private_attribute     - double leading underscore
8. __special_method__()    - double underscore both sides
"""

# ✅ GOOD: Following PEP 8
class UserRepository:
    """Repository for user data"""

    MAX_USERS = 1000  # Constant

    def __init__(self, database_url: str):
        self.database_url = database_url
        self._connection = None  # Private
        self.__private_data = {}  # Name mangled

    def get_user(self, user_id: int) -> dict:
        """Public method - lowercase with underscores"""
        return self._fetch_user_data(user_id)

    def _fetch_user_data(self, user_id: int) -> dict:
        """Private method - leading underscore"""
        pass

    def __internal_helper(self):
        """Name mangled private method"""
        pass

# ❌ BAD: Not following PEP 8
class userRepository:  # Should be UserRepository
    maxUsers = 1000    # Should be MAX_USERS

    def GetUser(self, userId):  # Should be get_user, user_id
        return self.FetchUserData(userId)

    def FetchUserData(self, userId):  # Should be lowercase
        pass
```

---

## Code Layout and Formatting

### Indentation and Line Length

```python
# Indentation: 4 spaces (never tabs)
def calculate_total_price(
    base_price: float,
    tax_rate: float,
    discount: float = 0.0
) -> float:
    """Calculate total price with tax and discount"""
    subtotal = base_price * (1 - discount)
    total = subtotal * (1 + tax_rate)
    return total

# Line length: Max 79 characters for code, 72 for docstrings
def process_data(
    input_data: list,
    transformation_func: callable,
    filters: list = None
) -> list:
    """
    Process data with transformation and optional filters.

    Keep docstrings within 72 characters per line for
    better readability.
    """
    pass

# Breaking long lines
# ✅ GOOD: Break after operators
total = (
    first_value
    + second_value
    - third_value
    * fourth_value
)

# ✅ GOOD: Hanging indent
result = some_function_that_takes_arguments(
    arg1,
    arg2,
    arg3,
    arg4
)

# ✅ GOOD: Aligned with opening delimiter
result = some_function(arg1, arg2,
                       arg3, arg4)

# Imports
# ✅ GOOD: Grouped and sorted
import os
import sys
from pathlib import Path

import numpy as np
import pandas as pd

from myapp.models import User, Post
from myapp.utils import helper_function

# ❌ BAD: All mixed up
from myapp.models import User
import sys
import numpy as np
from myapp.utils import helper_function
import os

# Blank lines
# Two blank lines before class/function at module level
# One blank line between methods

class MyClass:
    """My class"""

    def method_one(self):
        """First method"""
        pass

    def method_two(self):
        """Second method"""
        pass


def module_level_function():
    """Module level function"""
    pass


class AnotherClass:
    """Another class"""
    pass
```

---

## Type Hints

### Basic Type Hints

```python
from typing import List, Dict, Tuple, Optional, Union, Any, Callable

# Basic types
def greet(name: str) -> str:
    """Return greeting message"""
    return f"Hello, {name}!"

def add_numbers(a: int, b: int) -> int:
    """Add two integers"""
    return a + b

def calculate_average(numbers: List[float]) -> float:
    """Calculate average of numbers"""
    return sum(numbers) / len(numbers)

# Optional (can be None)
def get_user(user_id: int) -> Optional[dict]:
    """Get user by ID, None if not found"""
    # Might return None
    pass

# Union (multiple types)
def process_id(user_id: Union[int, str]) -> str:
    """Process user ID (int or string)"""
    return str(user_id)

# Dict with key/value types
def get_user_scores() -> Dict[str, int]:
    """Get user scores mapping"""
    return {"alice": 100, "bob": 95}

# Tuple with specific types
def get_coordinates() -> Tuple[float, float]:
    """Get X, Y coordinates"""
    return (10.5, 20.3)

# List of specific type
def get_user_ids() -> List[int]:
    """Get list of user IDs"""
    return [1, 2, 3, 4, 5]

# Callable (function type)
def apply_transformation(
    data: List[int],
    func: Callable[[int], int]
) -> List[int]:
    """Apply function to each element"""
    return [func(x) for x in data]

# Any (any type - avoid when possible)
def process_data(data: Any) -> Any:
    """Process data of any type"""
    pass
```

### Advanced Type Hints

```python
from typing import (
    TypeVar, Generic, Protocol, Literal,
    TypedDict, Final, ClassVar
)

# TypeVar for generics
T = TypeVar('T')

def first_element(items: List[T]) -> Optional[T]:
    """Get first element of list"""
    return items[0] if items else None

# Generic class
class Stack(Generic[T]):
    """Generic stack data structure"""

    def __init__(self) -> None:
        self._items: List[T] = []

    def push(self, item: T) -> None:
        """Push item onto stack"""
        self._items.append(item)

    def pop(self) -> T:
        """Pop item from stack"""
        return self._items.pop()

    def is_empty(self) -> bool:
        """Check if stack is empty"""
        return len(self._items) == 0

# Usage
int_stack: Stack[int] = Stack()
int_stack.push(1)
int_stack.push(2)

str_stack: Stack[str] = Stack()
str_stack.push("hello")

# TypedDict for structured dictionaries
class UserDict(TypedDict):
    """User dictionary structure"""
    id: int
    name: str
    email: str
    age: Optional[int]

def create_user(data: UserDict) -> UserDict:
    """Create user from structured dict"""
    return data

# Literal for specific values
def set_log_level(level: Literal["DEBUG", "INFO", "WARNING", "ERROR"]) -> None:
    """Set logging level"""
    pass

set_log_level("DEBUG")  # ✅ OK
# set_log_level("TRACE")  # ❌ Type checker error

# Final for constants
from typing import Final

MAX_CONNECTIONS: Final = 100

# MAX_CONNECTIONS = 200  # ❌ Type checker error

# ClassVar for class variables
from typing import ClassVar

class User:
    """User model"""

    user_count: ClassVar[int] = 0  # Class variable

    def __init__(self, name: str) -> None:
        self.name: str = name  # Instance variable
        User.user_count += 1

# Protocol for structural typing (duck typing)
class Drawable(Protocol):
    """Protocol for drawable objects"""

    def draw(self) -> None:
        """Draw the object"""
        ...

class Circle:
    """Circle implements Drawable protocol"""

    def draw(self) -> None:
        print("Drawing circle")

class Square:
    """Square implements Drawable protocol"""

    def draw(self) -> None:
        print("Drawing square")

def render(obj: Drawable) -> None:
    """Render any drawable object"""
    obj.draw()

# Works with any object that has draw() method
render(Circle())
render(Square())
```

### Type Hints with FastAPI/Pydantic

```python
from pydantic import BaseModel, Field, validator
from typing import List, Optional
from datetime import datetime

# Pydantic models with type hints
class UserBase(BaseModel):
    """Base user model"""
    username: str = Field(..., min_length=3, max_length=50)
    email: str = Field(..., regex=r'^[\w\.-]+@[\w\.-]+\.\w+$')
    age: Optional[int] = Field(None, ge=0, le=150)

class UserCreate(UserBase):
    """User creation model"""
    password: str = Field(..., min_length=8)

class UserResponse(UserBase):
    """User response model"""
    id: int
    created_at: datetime
    is_active: bool = True

    class Config:
        orm_mode = True

# FastAPI endpoints with type hints
from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy.orm import Session

app = FastAPI()

@app.post("/users/", response_model=UserResponse, status_code=201)
async def create_user(
    user: UserCreate,
    db: Session = Depends(get_db)
) -> UserResponse:
    """Create new user"""
    # Type hints ensure correct types
    db_user = User(**user.dict())
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: Session = Depends(get_db)
) -> UserResponse:
    """Get user by ID"""
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return user

@app.get("/users/", response_model=List[UserResponse])
async def list_users(
    skip: int = 0,
    limit: int = 100,
    db: Session = Depends(get_db)
) -> List[UserResponse]:
    """List users with pagination"""
    users = db.query(User).offset(skip).limit(limit).all()
    return users
```

---

## Docstrings

### Google Style Docstrings

```python
def calculate_statistics(numbers: List[float], precision: int = 2) -> Dict[str, float]:
    """
    Calculate statistical measures for a list of numbers.

    Args:
        numbers: List of numbers to analyze
        precision: Number of decimal places (default: 2)

    Returns:
        Dictionary containing mean, median, and std deviation

    Raises:
        ValueError: If numbers list is empty
        TypeError: If numbers contains non-numeric values

    Examples:
        >>> calculate_statistics([1, 2, 3, 4, 5])
        {'mean': 3.0, 'median': 3.0, 'std': 1.41}
    """
    if not numbers:
        raise ValueError("Numbers list cannot be empty")

    import statistics

    return {
        'mean': round(statistics.mean(numbers), precision),
        'median': round(statistics.median(numbers), precision),
        'std': round(statistics.stdev(numbers), precision)
    }

class UserRepository:
    """
    Repository for managing user data.

    This class provides methods for CRUD operations on users,
    with proper error handling and validation.

    Attributes:
        db_session: SQLAlchemy database session
        cache: Redis cache client for caching user data

    Example:
        >>> repo = UserRepository(session, redis_client)
        >>> user = repo.get_user(123)
    """

    def __init__(self, db_session: Session, cache: Redis) -> None:
        """
        Initialize repository.

        Args:
            db_session: SQLAlchemy session
            cache: Redis client for caching
        """
        self.db_session = db_session
        self.cache = cache
```

### NumPy Style Docstrings

```python
def process_matrix(matrix, operation='sum', axis=0):
    """
    Process a matrix with specified operation.

    Parameters
    ----------
    matrix : np.ndarray
        Input matrix to process
    operation : {'sum', 'mean', 'max', 'min'}, optional
        Operation to perform (default is 'sum')
    axis : int, optional
        Axis along which to perform operation (default is 0)

    Returns
    -------
    np.ndarray
        Processed matrix result

    Raises
    ------
    ValueError
        If operation is not recognized

    Examples
    --------
    >>> import numpy as np
    >>> matrix = np.array([[1, 2], [3, 4]])
    >>> process_matrix(matrix, 'sum', axis=0)
    array([4, 6])
    """
    pass
```

---

## Comments

### Good Comments

```python
# ✅ GOOD: Explain WHY, not WHAT
def calculate_discount(price: float, customer_type: str) -> float:
    """Calculate discount based on customer type"""

    # Premium customers get 20% because they pay annual fee
    if customer_type == "premium":
        return price * 0.20

    # Regular customers get 10% to encourage loyalty
    elif customer_type == "regular":
        return price * 0.10

    return 0.0

# ✅ GOOD: Explain complex algorithms
def find_median(numbers: List[int]) -> float:
    """Find median using quickselect algorithm"""
    # Using quickselect instead of sorting for O(n) average time
    # vs O(n log n) for full sort when we only need median
    pass

# ✅ GOOD: TODO comments
def send_notification(user_id: int, message: str) -> None:
    """Send notification to user"""
    # TODO: Add support for push notifications (JIRA-123)
    # TODO: Implement rate limiting to prevent spam
    send_email(user_id, message)

# ❌ BAD: Obvious comments
def add(a: int, b: int) -> int:
    # Add a and b  ❌ Obvious
    return a + b

def get_user(user_id: int) -> User:
    # Query the database  ❌ Code is self-explanatory
    return db.query(User).filter(User.id == user_id).first()

# ❌ BAD: Commented-out code
def process_data(data):
    result = transform(data)
    # old_result = old_transform(data)  ❌ Remove it
    # return old_result  ❌ Use version control instead
    return result
```

---

## Code Quality Tools

### Using mypy for Type Checking

```bash
# Install mypy
pip install mypy

# Check types
mypy my_module.py

# mypy.ini configuration
[mypy]
python_version = 3.10
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
```

```python
# Example with mypy
def add_numbers(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

# mypy will catch this error
result: str = add_numbers(1, 2)  # Error: incompatible type
```

### Using Black for Formatting

```bash
# Install black
pip install black

# Format files
black my_module.py

# Format entire project
black .

# Check without modifying
black --check my_module.py

# pyproject.toml configuration
[tool.black]
line-length = 88
target-version = ['py310']
include = '\.pyi?$'
```

### Using Flake8 for Linting

```bash
# Install flake8
pip install flake8

# Check code
flake8 my_module.py

# .flake8 configuration
[flake8]
max-line-length = 88
extend-ignore = E203, W503
exclude = .git,__pycache__,venv
```

---

## Best Practices

### ✅ Do's:

1. **Follow PEP 8** naming conventions
2. **Use type hints** for all public APIs
3. **Write docstrings** for all public functions/classes
4. **Keep lines** under 79 characters
5. **Use 4 spaces** for indentation
6. **Group imports** properly
7. **Use Black** for consistent formatting
8. **Run mypy** for type checking
9. **Write meaningful** variable names
10. **Comment WHY**, not what

### ❌ Don'ts:

1. **Don't use tabs** for indentation
2. **Don't write** obvious comments
3. **Don't leave** commented-out code
4. **Don't use** single letter variables (except counters)
5. **Don't ignore** linter warnings
6. **Don't use** `Any` type unnecessarily
7. **Don't mix** naming conventions

---

## Interview Questions

### Q1: What is PEP 8 and why is it important?

**Answer**: Python Enhancement Proposal 8 - style guide:

- **Consistency**: Uniform code across projects
- **Readability**: Easier to understand
- **Conventions**: Naming, formatting, layout
- **Tools**: Black, flake8, pylint enforce it
  Follow for professional code.

### Q2: Explain type hints benefits.

**Answer**:

- **Documentation**: Self-documenting code
- **IDE support**: Better autocomplete
- **Type checking**: Catch errors with mypy
- **Refactoring**: Safer changes
- **Not enforced**: Runtime doesn't check
  Use for all public APIs.

### Q3: When to use Optional vs Union[T, None]?

**Answer**: They're equivalent:

- `Optional[T]` = `Union[T, None]`
- Use `Optional[T]` for clarity
- Indicates value might be absent
- Example: `Optional[str]` for nullable string

### Q4: What are Protocol classes?

**Answer**: Structural subtyping (duck typing):

- Define interface without inheritance
- Any class matching structure works
- More Pythonic than ABC
- Example: Drawable protocol for draw() method

### Q5: How to use mypy in CI/CD?

**Answer**:

```bash
# Install in CI
pip install mypy

# Run type checking
mypy --strict src/

# Fail build on errors
mypy src/ || exit 1
```

Catches type errors before deployment.

---

## Summary

Code quality essentials:

- **PEP 8**: Naming, formatting, layout standards
- **Type hints**: Better documentation and tooling
- **Docstrings**: Google or NumPy style
- **Tools**: Black, mypy, flake8
- **Comments**: Explain WHY, not what
- **Consistency**: Use automated tools

Clean code is professional code! 📐
