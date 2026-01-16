# Decorators - Advanced Function Wrapping

## 📖 Concept Explanation

Decorators are a powerful Python feature that allows you to modify or enhance functions and classes without changing their source code. They implement the **Decorator Pattern** and are essentially **higher-order functions** that take a function as input and return a modified version.

### Basic Decorator Syntax

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before function call")
        result = func(*args, **kwargs)
        print("After function call")
        return result
    return wrapper

@my_decorator
def greet(name):
    print(f"Hello, {name}!")

# Equivalent to: greet = my_decorator(greet)
```

### How Decorators Work

```python
# Without decorator syntax
def original_function():
    return "Original"

def decorator(func):
    def wrapper():
        return f"Decorated: {func()}"
    return wrapper

decorated = decorator(original_function)
print(decorated())  # "Decorated: Original"

# With @ syntax
@decorator
def another_function():
    return "Another"

print(another_function())  # "Decorated: Another"
```

## 🧠 Why It Matters in Real Projects

- **Cross-Cutting Concerns**: Authentication, logging, caching, rate limiting
- **DRY Principle**: Avoid repeating code across multiple functions
- **Separation of Concerns**: Keep business logic separate from infrastructure code
- **Framework Integration**: Django, Flask, FastAPI heavily use decorators
- **Testing**: Mock and patch functionality
- **Performance Monitoring**: Add timing, profiling automatically

## ⚙️ Internal Working

### What Happens Under the Hood

```python
def trace(func):
    def wrapper(*args, **kwargs):
        print(f"TRACE: Calling {func.__name__}()")
        result = func(*args, **kwargs)
        print(f"TRACE: {func.__name__}() returned {result!r}")
        return result
    return wrapper

@trace
def square(x):
    return x * x

# When you call square(3):
# 1. Python looks up 'square' → finds the wrapper function
# 2. wrapper(3) is called
# 3. Inside wrapper, func(3) calls the original square function
# 4. Result is returned through the wrapper
```

### Function Metadata Loss Problem

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        """Wrapper docstring"""
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def original_function():
    """Original docstring"""
    pass

print(original_function.__name__)  # "wrapper" (WRONG!)
print(original_function.__doc__)   # "Wrapper docstring" (WRONG!)
```

### Solution: functools.wraps

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # Preserves metadata
    def wrapper(*args, **kwargs):
        """Wrapper docstring"""
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def original_function():
    """Original docstring"""
    pass

print(original_function.__name__)  # "original_function" (CORRECT!)
print(original_function.__doc__)   # "Original docstring" (CORRECT!)
```

## ✅ Best Practices

### 1. Always Use functools.wraps

```python
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        import time
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper
```

### 2. Decorators with Arguments

```python
from functools import wraps

def repeat(times):
    """Decorator that repeats function execution"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            results = []
            for _ in range(times):
                results.append(func(*args, **kwargs))
            return results
        return wrapper
    return decorator

@repeat(times=3)
def greet(name):
    return f"Hello, {name}!"

print(greet("Alice"))  # Executes 3 times
```

### 3. Class-Based Decorators

```python
from functools import wraps

class CountCalls:
    """Decorator that counts function calls"""
    
    def __init__(self, func):
        self.func = func
        self.count = 0
        wraps(func)(self)  # Preserve metadata
    
    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call {self.count} of {self.func.__name__}()")
        return self.func(*args, **kwargs)

@CountCalls
def process_data(data):
    return data.upper()

process_data("hello")  # Call 1
process_data("world")  # Call 2
```

### 4. Preserving Function Signatures with wrapt

```python
import wrapt

@wrapt.decorator
def pass_through(wrapped, instance, args, kwargs):
    """Perfect signature preservation"""
    print(f"Calling {wrapped.__name__}")
    return wrapped(*args, **kwargs)

@pass_through
def example(x: int, y: int = 10) -> int:
    """Add two numbers"""
    return x + y

# Signature is perfectly preserved
import inspect
print(inspect.signature(example))  # (x: int, y: int = 10) -> int
```

## ❌ Common Mistakes

### 1. Forgetting to Return the Result

```python
# WRONG - Function returns None!
def broken_decorator(func):
    def wrapper(*args, **kwargs):
        func(*args, **kwargs)  # Missing return!
    return wrapper

@broken_decorator
def get_value():
    return 42

print(get_value())  # None (BUG!)
```

### 2. Not Using *args and **kwargs

```python
# WRONG - Only works with no-argument functions
def bad_decorator(func):
    def wrapper():  # Missing *args, **kwargs
        return func()
    return wrapper

@bad_decorator
def greet(name):  # Will fail!
    return f"Hello, {name}"

# greet("Alice")  # TypeError: wrapper() takes 0 positional arguments
```

### 3. Decorator Order Matters

```python
from functools import wraps

def decorator1(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print("Decorator 1 - Before")
        result = func(*args, **kwargs)
        print("Decorator 1 - After")
        return result
    return wrapper

def decorator2(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print("Decorator 2 - Before")
        result = func(*args, **kwargs)
        print("Decorator 2 - After")
        return result
    return wrapper

@decorator1  # Applied SECOND (outer)
@decorator2  # Applied FIRST (inner)
def my_function():
    print("Function body")

my_function()
"""Output:
Decorator 1 - Before
Decorator 2 - Before
Function body
Decorator 2 - After
Decorator 1 - After
"""
```

## 🔐 Security Considerations

### 1. Authentication Decorator

```python
from functools import wraps
from flask import request, jsonify, g

def require_auth(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        token = request.headers.get('Authorization')
        if not token:
            return jsonify({"error": "No token provided"}), 401
        
        try:
            user = verify_token(token)
            g.user = user  # Store in Flask global context
            return func(*args, **kwargs)
        except InvalidTokenError:
            return jsonify({"error": "Invalid token"}), 401
    
    return wrapper

@app.route('/api/protected')
@require_auth
def protected_resource():
    return jsonify({"data": "Secret data", "user": g.user.username})
```

### 2. Rate Limiting Decorator

```python
from functools import wraps
import time
from collections import defaultdict
from threading import Lock

class RateLimiter:
    def __init__(self, max_calls, time_window):
        self.max_calls = max_calls
        self.time_window = time_window
        self.calls = defaultdict(list)
        self.lock = Lock()
    
    def __call__(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Use IP address as key (in production, use request.remote_addr)
            key = "user_identifier"
            now = time.time()
            
            with self.lock:
                # Remove old entries
                self.calls[key] = [
                    call_time for call_time in self.calls[key]
                    if now - call_time < self.time_window
                ]
                
                if len(self.calls[key]) >= self.max_calls:
                    raise Exception("Rate limit exceeded")
                
                self.calls[key].append(now)
            
            return func(*args, **kwargs)
        return wrapper

@RateLimiter(max_calls=5, time_window=60)  # 5 calls per minute
def api_endpoint():
    return {"status": "success"}
```

## 🚀 Performance Optimization

### 1. Caching with LRU Cache

```python
from functools import lru_cache
import time

@lru_cache(maxsize=128)
def expensive_computation(n):
    """Fibonacci with memoization"""
    if n < 2:
        return n
    return expensive_computation(n - 1) + expensive_computation(n - 2)

# First call: slow
start = time.time()
result1 = expensive_computation(35)
print(f"First call: {time.time() - start:.4f}s")

# Second call: instant (cached)
start = time.time()
result2 = expensive_computation(35)
print(f"Second call: {time.time() - start:.4f}s")

# Cache info
print(expensive_computation.cache_info())
```

### 2. Custom Timed Cache Decorator

```python
from functools import wraps
import time

def timed_cache(seconds):
    """Cache results for specified seconds"""
    def decorator(func):
        cache = {}
        cache_time = {}
        
        @wraps(func)
        def wrapper(*args):
            now = time.time()
            
            # Check if result exists and is not expired
            if args in cache:
                if now - cache_time[args] < seconds:
                    print(f"Cache hit for {args}")
                    return cache[args]
            
            # Compute and cache result
            print(f"Computing for {args}")
            result = func(*args)
            cache[args] = result
            cache_time[args] = now
            return result
        
        return wrapper
    return decorator

@timed_cache(seconds=5)
def fetch_user_data(user_id):
    time.sleep(1)  # Simulate slow API call
    return {"id": user_id, "name": f"User{user_id}"}

# First call: computes
print(fetch_user_data(1))
# Second call within 5 seconds: cached
print(fetch_user_data(1))
# After 5 seconds: computes again
time.sleep(6)
print(fetch_user_data(1))
```

## 🧪 Code Examples

### 1. Logging Decorator

```python
import logging
from functools import wraps

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def log_function_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        args_repr = [repr(a) for a in args]
        kwargs_repr = [f"{k}={v!r}" for k, v in kwargs.items()]
        signature = ", ".join(args_repr + kwargs_repr)
        
        logger.info(f"Calling {func.__name__}({signature})")
        
        try:
            result = func(*args, **kwargs)
            logger.info(f"{func.__name__} returned {result!r}")
            return result
        except Exception as e:
            logger.exception(f"{func.__name__} raised {e.__class__.__name__}")
            raise
    
    return wrapper

@log_function_call
def divide(a, b):
    return a / b

divide(10, 2)   # Logged
divide(10, 0)   # Exception logged
```

### 2. Retry Decorator

```python
from functools import wraps
import time
import random

def retry(max_attempts=3, delay=1, backoff=2, exceptions=(Exception,)):
    """Retry decorator with exponential backoff"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            attempt = 1
            current_delay = delay
            
            while attempt <= max_attempts:
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts:
                        raise
                    
                    print(f"Attempt {attempt} failed: {e}")
                    print(f"Retrying in {current_delay}s...")
                    time.sleep(current_delay)
                    
                    attempt += 1
                    current_delay *= backoff
            
        return wrapper
    return decorator

@retry(max_attempts=3, delay=1, backoff=2)
def unstable_api_call():
    """Simulates an unstable API"""
    if random.random() < 0.7:  # 70% failure rate
        raise ConnectionError("API unavailable")
    return {"status": "success"}

# Will retry up to 3 times with exponential backoff
result = unstable_api_call()
```

### 3. Validation Decorator

```python
from functools import wraps

def validate_types(**expected_types):
    """Validate function argument types"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Get function signature
            import inspect
            sig = inspect.signature(func)
            bound = sig.bind(*args, **kwargs)
            bound.apply_defaults()
            
            # Validate each argument
            for param_name, expected_type in expected_types.items():
                if param_name in bound.arguments:
                    value = bound.arguments[param_name]
                    if not isinstance(value, expected_type):
                        raise TypeError(
                            f"{param_name} must be {expected_type.__name__}, "
                            f"got {type(value).__name__}"
                        )
            
            return func(*args, **kwargs)
        return wrapper
    return decorator

@validate_types(name=str, age=int, salary=float)
def create_employee(name, age, salary=50000.0):
    return {"name": name, "age": age, "salary": salary}

# Valid
create_employee("Alice", 30, 75000.0)

# Invalid - raises TypeError
# create_employee("Bob", "thirty", 60000.0)
```

### 4. Singleton Decorator

```python
from functools import wraps

def singleton(cls):
    """Ensure only one instance of a class exists"""
    instances = {}
    
    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    
    return get_instance

@singleton
class Database:
    def __init__(self):
        print("Initializing database connection")
        self.connection = "DB_CONNECTION"

# Both variables reference the same instance
db1 = Database()  # Prints "Initializing..."
db2 = Database()  # Does not print anything
print(db1 is db2)  # True
```

## 🏗️ Real-World Use Cases

### Django View Authorization

```python
from functools import wraps
from django.http import HttpResponseForbidden

def require_permission(permission_name):
    """Django decorator for permission-based access control"""
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(request, *args, **kwargs):
            if not request.user.has_perm(permission_name):
                return HttpResponseForbidden("You don't have permission")
            return view_func(request, *args, **kwargs)
        return wrapper
    return decorator

@require_permission('blog.can_publish')
def publish_post(request, post_id):
    # Only users with 'blog.can_publish' permission can access
    post = Post.objects.get(id=post_id)
    post.published = True
    post.save()
    return HttpResponse("Post published")
```

### FastAPI Dependency Injection Alternative

```python
from functools import wraps
from fastapi import HTTPException, Header

def require_api_key(func):
    """FastAPI decorator for API key authentication"""
    @wraps(func)
    async def wrapper(*args, x_api_key: str = Header(...), **kwargs):
        if x_api_key != "secret-api-key":
            raise HTTPException(status_code=401, detail="Invalid API Key")
        return await func(*args, **kwargs)
    return wrapper

@app.get("/secure-endpoint")
@require_api_key
async def secure_endpoint():
    return {"message": "Access granted"}
```

### Database Transaction Decorator

```python
from functools import wraps
import psycopg2

def atomic(func):
    """Ensure function executes in a database transaction"""
    @wraps(func)
    def wrapper(connection, *args, **kwargs):
        cursor = connection.cursor()
        try:
            result = func(connection, *args, **kwargs)
            connection.commit()
            return result
        except Exception as e:
            connection.rollback()
            raise
        finally:
            cursor.close()
    return wrapper

@atomic
def transfer_money(connection, from_account, to_account, amount):
    cursor = connection.cursor()
    
    # Deduct from sender
    cursor.execute(
        "UPDATE accounts SET balance = balance - %s WHERE id = %s",
        (amount, from_account)
    )
    
    # Add to receiver
    cursor.execute(
        "UPDATE accounts SET balance = balance + %s WHERE id = %s",
        (amount, to_account)
    )
    
    return True
```

## ❓ Interview Questions

### Q1: What is a decorator and how does it work internally?

**Answer**: A decorator is a callable that takes a function as input and returns a modified version of that function. When you use `@decorator` syntax above a function, Python automatically calls `decorator(function)` and replaces the original function with the returned value.

```python
# These two are equivalent:
@my_decorator
def func():
    pass

# Same as:
def func():
    pass
func = my_decorator(func)
```

### Q2: What's the difference between @decorator and @decorator()?

**Answer**:
- `@decorator` - Direct decorator application (decorator is a function)
- `@decorator()` - Decorator factory (decorator is a function that returns a decorator)

```python
# Direct decorator
def simple_decorator(func):
    def wrapper():
        print("Decorated")
        return func()
    return wrapper

@simple_decorator
def func1():
    pass

# Decorator factory
def decorator_with_args(arg):
    def actual_decorator(func):
        def wrapper():
            print(f"Arg: {arg}")
            return func()
        return wrapper
    return actual_decorator

@decorator_with_args("hello")
def func2():
    pass
```

### Q3: How do you preserve function metadata when using decorators?

**Answer**: Use `functools.wraps` decorator on the wrapper function:

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # This line preserves __name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

### Q4: Can you chain multiple decorators? What's the execution order?

**Answer**: Yes, decorators are applied bottom-to-top, but execute top-to-bottom:

```python
@decorator1  # Applied second (outer)
@decorator2  # Applied first (inner)
def func():
    pass

# Equivalent to:
func = decorator1(decorator2(func))

# Execution order: decorator1 → decorator2 → func → decorator2 → decorator1
```

### Q5: What's the difference between function decorators and class decorators?

**Answer**:
- **Function decorators**: Wrap functions to modify their behavior
- **Class decorators**: Wrap entire classes to modify class behavior or instances

```python
# Function decorator
def log(func):
    def wrapper(*args):
        print(f"Calling {func.__name__}")
        return func(*args)
    return wrapper

# Class decorator
def add_methods(cls):
    cls.new_method = lambda self: "New method"
    return cls

@add_methods
class MyClass:
    pass

obj = MyClass()
print(obj.new_method())  # "New method"
```

### Q6: How would you implement a decorator that can be used with or without arguments?

**Answer**:

```python
from functools import wraps

def optional_decorator(func=None, *, prefix="LOG"):
    """Can be used as @optional_decorator or @optional_decorator(prefix="DEBUG")"""
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            print(f"{prefix}: Calling {f.__name__}")
            return f(*args, **kwargs)
        return wrapper
    
    if func is None:
        # Called with arguments: @optional_decorator(prefix="DEBUG")
        return decorator
    else:
        # Called without arguments: @optional_decorator
        return decorator(func)

@optional_decorator
def func1():
    pass

@optional_decorator(prefix="DEBUG")
def func2():
    pass
```

### Q7: What are the performance implications of decorators?

**Answer**:
- **Function call overhead**: Each decorator adds one extra function call
- **Memory overhead**: Closures capture variables
- **Caching solutions**: Use `functools.lru_cache` for expensive computations
- **Production tip**: Profile before optimizing; most decorator overhead is negligible

### Q8: How do decorators work with async functions?

**Answer**: You need to make the wrapper async as well:

```python
from functools import wraps
import asyncio

def async_timer(func):
    @wraps(func)
    async def wrapper(*args, **kwargs):
        import time
        start = time.time()
        result = await func(*args, **kwargs)
        print(f"Took {time.time() - start:.2f}s")
        return result
    return wrapper

@async_timer
async def fetch_data():
    await asyncio.sleep(1)
    return "Data"

asyncio.run(fetch_data())
```

## 🧩 Design Patterns

### Decorator Pattern Implementation

```python
from abc import ABC, abstractmethod

# Component interface
class Coffee(ABC):
    @abstractmethod
    def cost(self):
        pass
    
    @abstractmethod
    def description(self):
        pass

# Concrete component
class SimpleCoffee(Coffee):
    def cost(self):
        return 5
    
    def description(self):
        return "Simple coffee"

# Decorator base class
class CoffeeDecorator(Coffee):
    def __init__(self, coffee):
        self._coffee = coffee
    
    def cost(self):
        return self._coffee.cost()
    
    def description(self):
        return self._coffee.description()

# Concrete decorators
class Milk(CoffeeDecorator):
    def cost(self):
        return self._coffee.cost() + 2
    
    def description(self):
        return self._coffee.description() + ", milk"

class Sugar(CoffeeDecorator):
    def cost(self):
        return self._coffee.cost() + 1
    
    def description(self):
        return self._coffee.description() + ", sugar"

# Usage
coffee = SimpleCoffee()
print(f"{coffee.description()}: ${coffee.cost()}")

coffee_with_milk = Milk(coffee)
print(f"{coffee_with_milk.description()}: ${coffee_with_milk.cost()}")

coffee_with_milk_and_sugar = Sugar(Milk(SimpleCoffee()))
print(f"{coffee_with_milk_and_sugar.description()}: ${coffee_with_milk_and_sugar.cost()}")
```

---

## 📚 Summary

Decorators are a fundamental Python feature for:
- Adding functionality without modifying code
- Implementing cross-cutting concerns (auth, logging, caching)
- Following DRY and SOLID principles
- Building elegant, maintainable APIs

**Key Takeaways**:
1. Always use `@wraps` to preserve metadata
2. Use `*args, **kwargs` for flexibility
3. Understand the difference between `@dec` and `@dec()`
4. Be aware of execution order with multiple decorators
5. Consider performance implications
6. Use built-in decorators like `@lru_cache`, `@property`, `@staticmethod`
