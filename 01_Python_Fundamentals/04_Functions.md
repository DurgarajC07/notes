# Functions - Core Concepts and Advanced Techniques

## 📖 Concept Explanation

Functions are reusable blocks of code that perform specific tasks. Python functions are first-class objects, meaning they can be passed as arguments, returned from other functions, and assigned to variables.

### Basic Function Syntax

```python
# Simple function
def greet(name):
    return f"Hello, {name}!"

# Function with multiple parameters and default values
def create_user(username, email, role='user', active=True):
    return {
        'username': username,
        'email': email,
        'role': role,
        'active': active
    }

# Function with type hints
def calculate_total(price: float, tax_rate: float = 0.1) -> float:
    """Calculate total price including tax"""
    return price * (1 + tax_rate)
```

### Variable-Length Arguments

```python
# *args - variable positional arguments
def sum_all(*args):
    return sum(args)

print(sum_all(1, 2, 3, 4))  # 10

# **kwargs - variable keyword arguments
def print_info(**kwargs):
    for key, value in kwargs.items():
        print(f"{key}: {value}")

print_info(name="John", age=30, city="NYC")

# Combined usage
def complex_function(a, b, *args, option1=None, **kwargs):
    print(f"a={a}, b={b}")
    print(f"args={args}")
    print(f"option1={option1}")
    print(f"kwargs={kwargs}")

complex_function(1, 2, 3, 4, option1="test", x=10, y=20)
```

## 🧠 Why It Matters in Real Projects

### 1. Code Reusability and DRY Principle

```python
# Without functions - code repetition
user1_email = validate_email(input1)
user1_age = validate_age(input2)

user2_email = validate_email(input3)
user2_age = validate_age(input4)

# With functions - DRY (Don't Repeat Yourself)
def validate_user_input(email_input, age_input):
    """Centralized validation logic"""
    return {
        'email': validate_email(email_input),
        'age': validate_age(age_input)
    }

user1 = validate_user_input(input1, input2)
user2 = validate_user_input(input3, input4)
```

### 2. Abstraction and Separation of Concerns

```python
# High-level function abstracts complex operations
def process_user_registration(user_data):
    """Single entry point for user registration"""
    validated_data = validate_registration_data(user_data)
    user = create_user_account(validated_data)
    send_welcome_email(user)
    log_registration(user)
    return user

# Each function handles one responsibility
```

### 3. Testing and Debugging

```python
# Small, focused functions are easier to test
def calculate_discount(price, discount_percentage):
    """Pure function - easy to test"""
    if not 0 <= discount_percentage <= 100:
        raise ValueError("Discount must be between 0 and 100")
    return price * (discount_percentage / 100)

# Unit test
assert calculate_discount(100, 10) == 10
assert calculate_discount(200, 25) == 50
```

## ⚙️ Internal Working

### 1. Function Call Stack

```python
import sys

def function_a():
    function_b()

def function_b():
    function_c()

def function_c():
    # View call stack
    import traceback
    traceback.print_stack()

function_a()
# Shows the call chain: function_a -> function_b -> function_c
```

### 2. Function Objects

```python
def greet(name):
    return f"Hello, {name}!"

# Functions are objects
print(type(greet))  # <class 'function'>
print(greet.__name__)  # 'greet'
print(greet.__doc__)  # None (no docstring)

# Functions have attributes
greet.call_count = 0

def counting_greet(name):
    counting_greet.call_count += 1
    return f"Hello, {name}! (Call #{counting_greet.call_count})"

counting_greet.call_count = 0
```

### 3. Local vs Global Scope

```python
x = 10  # Global scope

def function():
    x = 20  # Local scope
    print(f"Local x: {x}")

function()  # Local x: 20
print(f"Global x: {x}")  # Global x: 10

# Using global keyword
counter = 0

def increment():
    global counter
    counter += 1

increment()
print(counter)  # 1

# Using nonlocal keyword (for nested functions)
def outer():
    count = 0

    def inner():
        nonlocal count
        count += 1
        return count

    return inner

counter = outer()
print(counter())  # 1
print(counter())  # 2
```

### 4. Default Argument Evaluation

```python
# ⚠️ Default arguments are evaluated ONCE at function definition
def append_to_list(item, target_list=[]):
    target_list.append(item)
    return target_list

print(append_to_list(1))  # [1]
print(append_to_list(2))  # [1, 2] - BUG! Same list

# Correct approach
def append_to_list(item, target_list=None):
    if target_list is None:
        target_list = []
    target_list.append(item)
    return target_list

print(append_to_list(1))  # [1]
print(append_to_list(2))  # [2] - Correct!
```

## ✅ Best Practices

### 1. Single Responsibility Principle

```python
# BAD - function does too much
def process_user(user_data):
    # Validation
    if not user_data.get('email'):
        raise ValueError("Email required")

    # Database operation
    user = save_to_database(user_data)

    # Email sending
    send_email(user.email, "Welcome!")

    # Logging
    log_user_creation(user)

    return user

# GOOD - separate concerns
def validate_user_data(user_data):
    if not user_data.get('email'):
        raise ValueError("Email required")
    return user_data

def create_user(user_data):
    validated = validate_user_data(user_data)
    return save_to_database(validated)

def welcome_new_user(user):
    send_email(user.email, "Welcome!")
    log_user_creation(user)

# Orchestrate with clear flow
def process_user(user_data):
    user = create_user(user_data)
    welcome_new_user(user)
    return user
```

### 2. Type Hints for Clarity

```python
from typing import List, Dict, Optional, Union, Callable

def process_users(
    users: List[Dict[str, Union[str, int]]],
    filter_func: Optional[Callable[[Dict], bool]] = None
) -> List[Dict[str, Union[str, int]]]:
    """
    Process list of users with optional filtering.

    Args:
        users: List of user dictionaries
        filter_func: Optional function to filter users

    Returns:
        Processed list of users
    """
    if filter_func:
        users = [u for u in users if filter_func(u)]

    return users

# Usage is now self-documenting
active_users = process_users(
    all_users,
    filter_func=lambda u: u['is_active']
)
```

### 3. Docstrings with Clear Documentation

```python
def calculate_compound_interest(
    principal: float,
    rate: float,
    time: int,
    frequency: int = 12
) -> float:
    """
    Calculate compound interest.

    Args:
        principal: Initial investment amount in dollars
        rate: Annual interest rate as decimal (e.g., 0.05 for 5%)
        time: Investment period in years
        frequency: Compounding frequency per year (default: 12 for monthly)

    Returns:
        Final amount including interest

    Raises:
        ValueError: If principal or rate is negative, or time/frequency <= 0

    Examples:
        >>> calculate_compound_interest(1000, 0.05, 10)
        1647.01

        >>> calculate_compound_interest(5000, 0.07, 5, frequency=4)
        7068.47
    """
    if principal < 0 or rate < 0:
        raise ValueError("Principal and rate must be non-negative")

    if time <= 0 or frequency <= 0:
        raise ValueError("Time and frequency must be positive")

    return principal * (1 + rate / frequency) ** (frequency * time)
```

### 4. Avoid Side Effects in Pure Functions

```python
# BAD - function has side effects
total = 0

def add_to_total(value):
    global total
    total += value  # Side effect!
    return total

# GOOD - pure function
def add(a, b):
    return a + b  # No side effects

# Maintain state explicitly when needed
class Calculator:
    def __init__(self):
        self.total = 0

    def add(self, value):
        self.total += value
        return self.total
```

### 5. Use \*args and \*\*kwargs Appropriately

```python
# Good use case: wrapper functions
import time
from functools import wraps

def timing_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.2f}s")
        return result
    return wrapper

@timing_decorator
def expensive_operation(data, threshold=100):
    # Function works with any arguments
    return [x for x in data if x > threshold]

# Good use case: flexible APIs
class APIClient:
    def request(self, method, endpoint, **kwargs):
        """Flexible request method accepting any requests parameters"""
        # kwargs can include: headers, params, json, timeout, etc.
        return requests.request(method, endpoint, **kwargs)
```

## ❌ Common Mistakes

### 1. Mutable Default Arguments

```python
# WRONG
def add_item(item, items=[]):
    items.append(item)
    return items

list1 = add_item(1)  # [1]
list2 = add_item(2)  # [1, 2] - BUG! Shares same list

# CORRECT
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

list1 = add_item(1)  # [1]
list2 = add_item(2)  # [2] - Correct!
```

### 2. Modifying Arguments Inside Functions

```python
# WRONG - unexpected behavior
def process_list(data):
    data.append('modified')  # Modifies original list!
    return data

original = [1, 2, 3]
result = process_list(original)
print(original)  # [1, 2, 3, 'modified'] - Original modified!

# CORRECT - explicit copy or non-mutating operation
def process_list(data):
    result = data.copy()  # or list(data)
    result.append('modified')
    return result

original = [1, 2, 3]
result = process_list(original)
print(original)  # [1, 2, 3] - Original unchanged
```

### 3. Overusing Global Variables

```python
# BAD
config = {}

def set_config(key, value):
    global config
    config[key] = value

def get_config(key):
    return config.get(key)

# GOOD - use class or pass explicitly
class Config:
    def __init__(self):
        self._data = {}

    def set(self, key, value):
        self._data[key] = value

    def get(self, key):
        return self._data.get(key)

config = Config()
```

### 4. Not Returning Values Explicitly

```python
# BAD - implicit None return
def calculate_total(items):
    total = sum(item['price'] for item in items)
    # Forgot return!

result = calculate_total(items)
print(result)  # None - BUG!

# GOOD - explicit return
def calculate_total(items):
    return sum(item['price'] for item in items)
```

### 5. Too Many Parameters

```python
# BAD - too many parameters
def create_user(name, email, age, city, country, phone, address, zipcode):
    pass

# GOOD - use dataclass or dictionary
from dataclasses import dataclass

@dataclass
class UserData:
    name: str
    email: str
    age: int
    city: str
    country: str
    phone: str
    address: str
    zipcode: str

def create_user(user_data: UserData):
    pass

# Or use **kwargs
def create_user(**user_data):
    required = ['name', 'email']
    if not all(k in user_data for k in required):
        raise ValueError("Missing required fields")
    return user_data
```

## 🔐 Security Considerations

### 1. Validate Function Inputs

```python
def transfer_money(from_account, to_account, amount):
    """Secure money transfer with validation"""

    # Input validation
    if not isinstance(amount, (int, float)):
        raise TypeError("Amount must be a number")

    if amount <= 0:
        raise ValueError("Amount must be positive")

    if amount > from_account.balance:
        raise ValueError("Insufficient funds")

    # Rate limiting check
    if is_rate_limited(from_account.id):
        raise PermissionError("Rate limit exceeded")

    # Perform transfer
    from_account.balance -= amount
    to_account.balance += amount

    log_transaction(from_account, to_account, amount)
```

### 2. Avoid eval() and exec() with User Input

```python
# DANGEROUS
def calculate(expression):
    return eval(expression)  # Arbitrary code execution!

# User could input: __import__('os').system('rm -rf /')

# SAFE - use ast.literal_eval or specific parsers
import ast
import operator

OPERATORS = {
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.Div: operator.truediv
}

def safe_calculate(expression):
    """Safe mathematical expression evaluator"""
    try:
        node = ast.parse(expression, mode='eval')
        return eval_node(node.body)
    except Exception:
        raise ValueError("Invalid expression")

def eval_node(node):
    if isinstance(node, ast.Num):
        return node.n
    elif isinstance(node, ast.BinOp):
        op = OPERATORS.get(type(node.op))
        if op is None:
            raise ValueError("Unsupported operator")
        return op(eval_node(node.left), eval_node(node.right))
    raise ValueError("Invalid expression")
```

### 3. Sanitize Function Outputs

```python
def get_user_profile(user_id):
    """Secure user profile retrieval"""
    user = database.get_user(user_id)

    if not user:
        return None

    # Don't expose sensitive data
    return {
        'id': user.id,
        'username': user.username,
        'email': mask_email(user.email),
        # DON'T include: password_hash, api_keys, etc.
    }

def mask_email(email):
    """Partially mask email for privacy"""
    username, domain = email.split('@')
    masked = username[0] + '***' + username[-1] if len(username) > 2 else '***'
    return f"{masked}@{domain}"
```

## 🚀 Performance Optimization Techniques

### 1. Use Generators for Large Datasets

```python
# Memory inefficient - loads everything
def get_all_records():
    return [fetch_record(i) for i in range(1000000)]

# Memory efficient - lazy evaluation
def get_all_records():
    for i in range(1000000):
        yield fetch_record(i)

# Usage
for record in get_all_records():
    process(record)  # Processes one at a time
```

### 2. Memoization for Expensive Computations

```python
from functools import lru_cache

# Without cache - slow for repeated calls
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# With cache - fast
@lru_cache(maxsize=None)
def fibonacci_cached(n):
    if n < 2:
        return n
    return fibonacci_cached(n-1) + fibonacci_cached(n-2)

import time

start = time.time()
fibonacci(35)  # ~3 seconds
print(f"Without cache: {time.time() - start:.2f}s")

start = time.time()
fibonacci_cached(35)  # ~0.0001 seconds
print(f"With cache: {time.time() - start:.6f}s")
```

### 3. Use Built-in Functions (C Implementation)

```python
import timeit

data = list(range(1000))

# Slower - Python loop
def sum_python(data):
    total = 0
    for item in data:
        total += item
    return total

# Faster - built-in (C implementation)
def sum_builtin(data):
    return sum(data)

print(timeit.timeit(lambda: sum_python(data), number=10000))   # ~0.5s
print(timeit.timeit(lambda: sum_builtin(data), number=10000))  # ~0.05s
```

### 4. Avoid Repeated Function Calls in Loops

```python
# Slow - repeated method lookup
result = []
for i in range(10000):
    result.append(i * 2)  # Looks up 'append' each iteration

# Fast - cache method reference
result = []
append = result.append
for i in range(10000):
    append(i * 2)

# Fastest - list comprehension
result = [i * 2 for i in range(10000)]
```

## 🧪 Code Examples

### Example 1: Function Composition

```python
from functools import reduce
from typing import Callable, TypeVar

T = TypeVar('T')

def compose(*functions: Callable) -> Callable:
    """Compose multiple functions into a single function"""
    return reduce(lambda f, g: lambda x: f(g(x)), functions, lambda x: x)

# Individual functions
def add_ten(x):
    return x + 10

def multiply_by_two(x):
    return x * 2

def square(x):
    return x ** 2

# Compose them
pipeline = compose(square, multiply_by_two, add_ten)

result = pipeline(5)  # ((5 + 10) * 2) ** 2 = 900
print(result)
```

### Example 2: Decorator for Retry Logic

```python
import time
import random
from functools import wraps

def retry(max_attempts=3, delay=1, backoff=2, exceptions=(Exception,)):
    """
    Decorator to retry function on exception

    Args:
        max_attempts: Maximum number of retry attempts
        delay: Initial delay between retries in seconds
        backoff: Multiplier for delay after each attempt
        exceptions: Tuple of exceptions to catch
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            current_delay = delay

            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise

                    print(f"Attempt {attempt + 1} failed: {e}")
                    print(f"Retrying in {current_delay}s...")
                    time.sleep(current_delay)
                    current_delay *= backoff

        return wrapper
    return decorator

# Usage
@retry(max_attempts=5, delay=1, backoff=2)
def unreliable_api_call():
    if random.random() < 0.7:
        raise ConnectionError("Network error")
    return "Success!"

result = unreliable_api_call()
```

### Example 3: Partial Function Application

```python
from functools import partial

def send_email(recipient, subject, body, from_addr, smtp_server):
    """Send email with full configuration"""
    print(f"Sending to {recipient}")
    print(f"From: {from_addr}")
    print(f"Subject: {subject}")
    print(f"Via: {smtp_server}")

# Create specialized function with pre-filled arguments
send_notification = partial(
    send_email,
    from_addr="noreply@company.com",
    smtp_server="smtp.company.com"
)

# Now simpler to call
send_notification(
    recipient="user@example.com",
    subject="Welcome!",
    body="Thanks for joining"
)

# Another specialized version
send_admin_alert = partial(
    send_email,
    recipient="admin@company.com",
    from_addr="alerts@company.com",
    smtp_server="smtp.company.com"
)

send_admin_alert(
    subject="Server Alert",
    body="CPU usage high"
)
```

## 🏗️ Real-World Use Cases

### 1. API Request Builder

```python
from typing import Dict, Any, Optional
import requests

def build_api_request(
    base_url: str,
    endpoint: str,
    method: str = 'GET',
    headers: Optional[Dict[str, str]] = None,
    params: Optional[Dict[str, Any]] = None,
    json_data: Optional[Dict[str, Any]] = None,
    timeout: int = 30,
    retry_count: int = 3
) -> Dict[str, Any]:
    """
    Flexible API request builder with retry logic
    """
    url = f"{base_url.rstrip('/')}/{endpoint.lstrip('/')}"

    default_headers = {
        'Content-Type': 'application/json',
        'User-Agent': 'MyApp/1.0'
    }

    if headers:
        default_headers.update(headers)

    for attempt in range(retry_count):
        try:
            response = requests.request(
                method=method,
                url=url,
                headers=default_headers,
                params=params,
                json=json_data,
                timeout=timeout
            )
            response.raise_for_status()
            return response.json()

        except requests.exceptions.RequestException as e:
            if attempt == retry_count - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff

# Usage
api_client = partial(
    build_api_request,
    base_url="https://api.example.com",
    headers={'Authorization': 'Bearer token123'}
)

users = api_client(endpoint='/users', method='GET')
new_user = api_client(
    endpoint='/users',
    method='POST',
    json_data={'name': 'John', 'email': 'john@example.com'}
)
```

### 2. Data Validation Pipeline

```python
from typing import Callable, List, Tuple, Any

ValidationResult = Tuple[bool, Optional[str]]
Validator = Callable[[Any], ValidationResult]

def validate_required(field_name: str) -> Validator:
    """Factory for required field validator"""
    def validator(value):
        if value is None or value == '':
            return False, f"{field_name} is required"
        return True, None
    return validator

def validate_email() -> Validator:
    """Email format validator"""
    def validator(value):
        if not isinstance(value, str) or '@' not in value:
            return False, "Invalid email format"
        return True, None
    return validator

def validate_range(min_val: int, max_val: int, field_name: str) -> Validator:
    """Factory for range validator"""
    def validator(value):
        try:
            num = int(value)
            if not min_val <= num <= max_val:
                return False, f"{field_name} must be between {min_val} and {max_val}"
            return True, None
        except (ValueError, TypeError):
            return False, f"{field_name} must be a number"
    return validator

def create_validation_pipeline(*validators: Validator) -> Validator:
    """Compose multiple validators into a pipeline"""
    def pipeline(value):
        for validator in validators:
            is_valid, error = validator(value)
            if not is_valid:
                return False, error
        return True, None
    return pipeline

# Create validation rules
email_validator = create_validation_pipeline(
    validate_required("Email"),
    validate_email()
)

age_validator = create_validation_pipeline(
    validate_required("Age"),
    validate_range(18, 120, "Age")
)

# Validate user input
def validate_user_registration(data):
    validations = {
        'email': email_validator,
        'age': age_validator
    }

    errors = {}
    for field, validator in validations.items():
        is_valid, error = validator(data.get(field))
        if not is_valid:
            errors[field] = error

    return len(errors) == 0, errors

# Usage
user_data = {'email': 'john@example.com', 'age': '25'}
is_valid, errors = validate_user_registration(user_data)

if not is_valid:
    print("Validation errors:", errors)
```

### 3. Database Query Builder

```python
from typing import List, Dict, Any, Optional

class QueryBuilder:
    """Fluent interface for building database queries"""

    def __init__(self, table: str):
        self.table = table
        self._select_fields: List[str] = ['*']
        self._where_conditions: List[str] = []
        self._params: List[Any] = []
        self._order_by: Optional[str] = None
        self._limit: Optional[int] = None
        self._offset: Optional[int] = None

    def select(self, *fields: str) -> 'QueryBuilder':
        """Select specific fields"""
        self._select_fields = list(fields)
        return self

    def where(self, condition: str, *params: Any) -> 'QueryBuilder':
        """Add WHERE condition"""
        self._where_conditions.append(condition)
        self._params.extend(params)
        return self

    def order_by(self, field: str, direction: str = 'ASC') -> 'QueryBuilder':
        """Add ORDER BY clause"""
        self._order_by = f"{field} {direction}"
        return self

    def limit(self, count: int) -> 'QueryBuilder':
        """Add LIMIT clause"""
        self._limit = count
        return self

    def offset(self, count: int) -> 'QueryBuilder':
        """Add OFFSET clause"""
        self._offset = count
        return self

    def build(self) -> Tuple[str, List[Any]]:
        """Build the SQL query"""
        fields = ', '.join(self._select_fields)
        query = f"SELECT {fields} FROM {self.table}"

        if self._where_conditions:
            conditions = ' AND '.join(self._where_conditions)
            query += f" WHERE {conditions}"

        if self._order_by:
            query += f" ORDER BY {self._order_by}"

        if self._limit:
            query += f" LIMIT {self._limit}"

        if self._offset:
            query += f" OFFSET {self._offset}"

        return query, self._params

# Usage
query, params = (
    QueryBuilder('users')
    .select('id', 'username', 'email')
    .where('age >= ?', 18)
    .where('is_active = ?', True)
    .order_by('created_at', 'DESC')
    .limit(10)
    .offset(20)
    .build()
)

print(query)
# SELECT id, username, email FROM users
# WHERE age >= ? AND is_active = ?
# ORDER BY created_at DESC
# LIMIT 10 OFFSET 20

print(params)  # [18, True]
```

## ❓ Frequently Asked Interview Questions

### Q1: What is the difference between arguments and parameters?

**Answer:**

- **Parameters**: Variables in function definition
- **Arguments**: Actual values passed when calling function

```python
def greet(name, greeting="Hello"):  # name, greeting are parameters
    return f"{greeting}, {name}!"

result = greet("John", "Hi")  # "John", "Hi" are arguments
```

### Q2: Explain \*args and \*\*kwargs

**Answer:**

- `*args`: Captures variable positional arguments as tuple
- `**kwargs`: Captures variable keyword arguments as dictionary

```python
def function(*args, **kwargs):
    print(f"args: {args}")      # tuple
    print(f"kwargs: {kwargs}")  # dict

function(1, 2, 3, x=10, y=20)
# args: (1, 2, 3)
# kwargs: {'x': 10, 'y': 20}

# Unpacking
def add(a, b, c):
    return a + b + c

numbers = [1, 2, 3]
result = add(*numbers)  # Unpacks list as arguments

config = {'host': 'localhost', 'port': 5432}
connect(**config)  # Unpacks dict as keyword arguments
```

### Q3: What are first-class functions?

**Answer:**
Functions are first-class citizens in Python - they can be:

1. Assigned to variables
2. Passed as arguments
3. Returned from functions
4. Stored in data structures

```python
# 1. Assign to variable
def square(x):
    return x ** 2

func = square
print(func(5))  # 25

# 2. Pass as argument
def apply_func(func, value):
    return func(value)

result = apply_func(square, 5)  # 25

# 3. Return from function
def get_operation(op):
    if op == 'square':
        return lambda x: x ** 2
    elif op == 'double':
        return lambda x: x * 2

operation = get_operation('square')
print(operation(5))  # 25

# 4. Store in data structures
operations = {
    'square': lambda x: x ** 2,
    'double': lambda x: x * 2,
    'negate': lambda x: -x
}

result = operations['square'](5)  # 25
```

### Q4: What is a closure?

**Answer:**
A closure is a function that retains access to variables from its enclosing scope even after that scope has finished executing.

```python
def outer(x):
    def inner(y):
        return x + y  # 'inner' captures 'x' from outer scope
    return inner

add_five = outer(5)
print(add_five(3))  # 8
print(add_five(10))  # 15

# 'x' is still accessible even though outer() has returned

# Practical use: creating specialized functions
def multiplier(factor):
    def multiply(number):
        return number * factor
    return multiply

double = multiplier(2)
triple = multiplier(3)

print(double(10))  # 20
print(triple(10))  # 30
```

### Q5: Explain the difference between shallow and deep copy for function arguments

**Answer:**

```python
import copy

# Mutable objects are passed by reference
def modify_list(lst):
    lst.append(4)

original = [1, 2, 3]
modify_list(original)
print(original)  # [1, 2, 3, 4] - modified!

# Shallow copy - nested objects still shared
def safe_modify(lst):
    lst = lst.copy()  # Shallow copy
    lst.append(4)
    return lst

original = [[1, 2], [3, 4]]
result = safe_modify(original)
result[0].append(999)
print(original)  # [[1, 2, 999], [3, 4]] - nested list modified!

# Deep copy - completely independent
def truly_safe_modify(lst):
    lst = copy.deepcopy(lst)
    lst[0].append(999)
    return lst

original = [[1, 2], [3, 4]]
result = truly_safe_modify(original)
print(original)  # [[1, 2], [3, 4]] - unchanged
```

### Q6: What is the difference between `return` and `yield`?

**Answer:**

- `return`: Returns a value and exits function
- `yield`: Returns a value and suspends function, creating a generator

```python
# return - function exits
def get_numbers_return():
    return [1, 2, 3, 4, 5]

numbers = get_numbers_return()  # All numbers in memory
print(type(numbers))  # <class 'list'>

# yield - creates generator
def get_numbers_yield():
    for i in range(1, 6):
        yield i

numbers = get_numbers_yield()  # Generator object, no numbers yet
print(type(numbers))  # <class 'generator'>

for num in numbers:  # Numbers generated on-demand
    print(num)

# Memory efficiency
import sys
list_result = [i for i in range(1000000)]
gen_result = (i for i in range(1000000))

print(sys.getsizeof(list_result))  # ~8MB
print(sys.getsizeof(gen_result))   # ~120 bytes
```

### Q7: How does Python handle default mutable arguments?

**Answer:**
Default arguments are evaluated ONCE at function definition, not each call.

```python
# Problem
def append_to(element, target=[]):
    target.append(element)
    return target

print(append_to(1))  # [1]
print(append_to(2))  # [1, 2] - BUG! Same list

# Why? Default list is created once
print(append_to.__defaults__)  # ([1, 2],) - shared object

# Solution
def append_to(element, target=None):
    if target is None:
        target = []  # New list each call
    target.append(element)
    return target

print(append_to(1))  # [1]
print(append_to(2))  # [2] - Correct!
```

### Q8: What is the difference between `@staticmethod` and `@classmethod`?

**Answer:**

```python
class MyClass:
    class_var = "I'm a class variable"

    def instance_method(self):
        """Regular method - requires instance"""
        return f"Instance: {self}"

    @classmethod
    def class_method(cls):
        """Class method - receives class as first argument"""
        return f"Class: {cls}, {cls.class_var}"

    @staticmethod
    def static_method():
        """Static method - no automatic first argument"""
        return "I'm independent of class/instance"

# Usage
obj = MyClass()

# Instance method needs instance
obj.instance_method()

# Class method can be called on class or instance
MyClass.class_method()
obj.class_method()

# Static method can be called on class or instance
MyClass.static_method()
obj.static_method()

# When to use:
# - @classmethod: Factory methods, alternative constructors
# - @staticmethod: Utility functions related to the class

class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    @classmethod
    def from_string(cls, date_string):
        """Alternative constructor"""
        year, month, day = map(int, date_string.split('-'))
        return cls(year, month, day)  # Creates instance

    @staticmethod
    def is_valid_date(year, month, day):
        """Utility function"""
        return 1 <= month <= 12 and 1 <= day <= 31

# Factory method
date = Date.from_string('2024-01-15')

# Utility function
if Date.is_valid_date(2024, 1, 15):
    print("Valid date")
```

---

**Next**: [05_Lambda_Map_Filter_Reduce.md](05_Lambda_Map_Filter_Reduce.md)
