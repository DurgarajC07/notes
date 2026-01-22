# 🔄 Functional Programming Concepts

## Overview

Functional programming emphasizes immutable data, pure functions, and declarative code. Python supports functional paradigms alongside OOP.

---

## Pure Functions

### Definition and Benefits

```python
# ❌ IMPURE: Side effects and external state
total = 0

def add_impure(x):
    """Impure - modifies external state"""
    global total
    total += x
    return total

# ✅ PURE: No side effects, same input = same output
def add_pure(x, y):
    """Pure - only depends on inputs"""
    return x + y

# Pure function characteristics:
# 1. Deterministic - same inputs always produce same output
# 2. No side effects - doesn't modify external state
# 3. No I/O operations
# 4. Testable and predictable

def calculate_discount(price, discount_percent):
    """Pure function"""
    return price * (1 - discount_percent / 100)

# Always returns same result
assert calculate_discount(100, 10) == 90.0
assert calculate_discount(100, 10) == 90.0

# ❌ IMPURE: Depends on external state
import datetime

def get_greeting_impure():
    """Impure - depends on current time"""
    hour = datetime.datetime.now().hour
    if hour < 12:
        return "Good morning"
    return "Good afternoon"

# ✅ PURE: Pass time as parameter
def get_greeting_pure(hour):
    """Pure - explicit dependency"""
    if hour < 12:
        return "Good morning"
    return "Good afternoon"
```

---

## First-Class Functions

### Functions as Values

```python
# Functions are objects
def greet(name):
    return f"Hello, {name}"

# Assign to variable
say_hello = greet
print(say_hello("Alice"))  # "Hello, Alice"

# Store in data structures
operations = {
    'add': lambda x, y: x + y,
    'subtract': lambda x, y: x - y,
    'multiply': lambda x, y: x * y,
    'divide': lambda x, y: x / y if y != 0 else None
}

result = operations['multiply'](5, 3)  # 15

# Pass as arguments
def apply_operation(func, x, y):
    """Higher-order function"""
    return func(x, y)

result = apply_operation(lambda x, y: x ** y, 2, 3)  # 8

# Return from functions
def create_multiplier(factor):
    """Function factory"""
    def multiplier(x):
        return x * factor
    return multiplier

double = create_multiplier(2)
triple = create_multiplier(3)

print(double(5))  # 10
print(triple(5))  # 15
```

---

## Map, Filter, Reduce

### Functional Data Transformations

```python
from functools import reduce

# MAP: Transform each element
numbers = [1, 2, 3, 4, 5]

# Using map
squared = map(lambda x: x ** 2, numbers)
print(list(squared))  # [1, 4, 9, 16, 25]

# List comprehension (more Pythonic)
squared = [x ** 2 for x in numbers]

# FILTER: Select elements
even_numbers = filter(lambda x: x % 2 == 0, numbers)
print(list(even_numbers))  # [2, 4]

# List comprehension alternative
even_numbers = [x for x in numbers if x % 2 == 0]

# REDUCE: Aggregate to single value
total = reduce(lambda acc, x: acc + x, numbers, 0)
print(total)  # 15

# Built-in alternatives (preferred)
total = sum(numbers)

# Complex example: Process data pipeline
data = [
    {"name": "Alice", "age": 30, "salary": 50000},
    {"name": "Bob", "age": 25, "salary": 45000},
    {"name": "Charlie", "age": 35, "salary": 60000}
]

# Get names of people over 30
result = list(map(
    lambda p: p["name"],
    filter(lambda p: p["age"] > 30, data)
))
# ['Charlie']

# List comprehension (more readable)
result = [p["name"] for p in data if p["age"] > 30]

# Calculate total salary
total_salary = reduce(
    lambda acc, p: acc + p["salary"],
    data,
    0
)
# 155000
```

---

## Lambda Functions

### Anonymous Functions

```python
# Basic lambda
add = lambda x, y: x + y
print(add(3, 5))  # 8

# Lambda with map
numbers = [1, 2, 3, 4, 5]
doubled = list(map(lambda x: x * 2, numbers))

# Lambda with filter
evens = list(filter(lambda x: x % 2 == 0, numbers))

# Lambda with sorted
people = [
    {"name": "Alice", "age": 30},
    {"name": "Bob", "age": 25},
    {"name": "Charlie", "age": 35}
]

# Sort by age
sorted_people = sorted(people, key=lambda p: p["age"])

# Sort by name length descending
sorted_people = sorted(people, key=lambda p: len(p["name"]), reverse=True)

# Lambda limitations
# ❌ Can't have statements (only expressions)
# ❌ Can't have multiple lines
# ❌ Harder to debug (no name)
# ✅ Use for simple, one-time operations

# When to use named function instead
def calculate_bonus(employee):
    """Named function - more readable"""
    base = employee["salary"] * 0.1
    if employee["performance"] == "excellent":
        return base * 1.5
    return base

# This would be ugly as lambda
employees_with_bonus = [
    {**emp, "bonus": calculate_bonus(emp)}
    for emp in employees
]
```

---

## Function Composition

### Combining Functions

```python
# Function composition
def compose(*functions):
    """Compose functions right to left"""
    def inner(arg):
        result = arg
        for func in reversed(functions):
            result = func(result)
        return result
    return inner

# Example functions
def add_ten(x):
    return x + 10

def multiply_by_two(x):
    return x * 2

def square(x):
    return x ** 2

# Compose: square(multiply_by_two(add_ten(x)))
composed = compose(square, multiply_by_two, add_ten)
result = composed(5)  # square(multiply_by_two(15)) = square(30) = 900

# Pipe (left to right)
def pipe(*functions):
    """Compose functions left to right"""
    def inner(arg):
        result = arg
        for func in functions:
            result = func(result)
        return result
    return inner

# Pipe: add_ten -> multiply_by_two -> square
piped = pipe(add_ten, multiply_by_two, square)
result = piped(5)  # square(multiply_by_two(add_ten(5))) = 900

# Practical example: Data processing pipeline
def clean_data(data):
    """Remove None values"""
    return [x for x in data if x is not None]

def normalize(data):
    """Normalize to 0-1 range"""
    min_val = min(data)
    max_val = max(data)
    return [(x - min_val) / (max_val - min_val) for x in data]

def round_values(data):
    """Round to 2 decimals"""
    return [round(x, 2) for x in data]

# Compose pipeline
process_data = pipe(clean_data, normalize, round_values)

raw_data = [10, None, 20, 30, None, 40]
processed = process_data(raw_data)
# [0.0, 0.33, 0.67, 1.0]
```

---

## Closures

### Functions with Enclosed State

```python
# Closure - function with access to outer scope
def make_counter():
    """Counter with private state"""
    count = 0

    def increment():
        nonlocal count  # Modify outer variable
        count += 1
        return count

    return increment

counter1 = make_counter()
counter2 = make_counter()

print(counter1())  # 1
print(counter1())  # 2
print(counter2())  # 1 (separate state)

# Practical example: Memoization
def memoize(func):
    """Cache function results"""
    cache = {}

    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]

    return wrapper

@memoize
def fibonacci(n):
    """Fibonacci with memoization"""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# Fast even for large n
print(fibonacci(100))

# Closure for configuration
def create_validator(min_value, max_value):
    """Create validator with configured range"""
    def validate(value):
        return min_value <= value <= max_value
    return validate

age_validator = create_validator(0, 120)
percentage_validator = create_validator(0, 100)

print(age_validator(25))    # True
print(age_validator(150))   # False
print(percentage_validator(50))  # True
```

---

## Immutability

### Immutable Data Structures

```python
# Immutable by default
numbers = (1, 2, 3)  # Tuple
# numbers[0] = 10  # TypeError

# Mutable
numbers_list = [1, 2, 3]
numbers_list[0] = 10  # Works

# Benefits of immutability:
# 1. Thread-safe
# 2. Hashable (can be dict keys)
# 3. Predictable
# 4. Easier debugging

# Working with immutable data
def add_item_mutable(items, item):
    """Mutates original list"""
    items.append(item)
    return items

def add_item_immutable(items, item):
    """Returns new list"""
    return [*items, item]

original = [1, 2, 3]
new = add_item_immutable(original, 4)
print(original)  # [1, 2, 3] - unchanged
print(new)       # [1, 2, 3, 4]

# Immutable dictionary updates
from types import MappingProxyType

# Read-only dict
user = {"name": "Alice", "age": 30}
readonly_user = MappingProxyType(user)
# readonly_user["age"] = 31  # TypeError

# Update immutably
def update_user_immutable(user, **updates):
    """Return new dict with updates"""
    return {**user, **updates}

user = {"name": "Alice", "age": 30}
updated = update_user_immutable(user, age=31, city="NYC")
# Original unchanged, new dict created

# Named tuples (immutable with named fields)
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p1 = Point(10, 20)
# p1.x = 15  # AttributeError

# Create new with changes
p2 = p1._replace(x=15)

# Dataclasses with frozen
from dataclasses import dataclass

@dataclass(frozen=True)
class User:
    """Immutable user"""
    name: str
    age: int

user = User("Alice", 30)
# user.age = 31  # FrozenInstanceError
```

---

## Partial Application & Currying

### Partial Function Application

```python
from functools import partial

# Regular function
def power(base, exponent):
    return base ** exponent

# Partial application
square = partial(power, exponent=2)
cube = partial(power, exponent=3)

print(square(5))  # 25
print(cube(5))    # 125

# Practical example: Logging
import logging

def log_message(level, message, logger_name="app"):
    """Log with specific level"""
    logger = logging.getLogger(logger_name)
    logger.log(level, message)

# Create specialized loggers
log_error = partial(log_message, logging.ERROR)
log_warning = partial(log_message, logging.WARNING)

log_error("Database connection failed")
log_warning("Cache miss")

# Currying - transform f(a, b, c) to f(a)(b)(c)
def curry(func):
    """Convert function to curried version"""
    def curried(*args):
        if len(args) >= func.__code__.co_argcount:
            return func(*args)
        return lambda *more: curried(*(args + more))
    return curried

@curry
def add_three(a, b, c):
    return a + b + c

# Can call all at once
print(add_three(1, 2, 3))  # 6

# Or partially
add_1 = add_three(1)
add_1_2 = add_1(2)
result = add_1_2(3)  # 6

# Practical example: Validators
@curry
def validate_range(min_val, max_val, value):
    """Validate value in range"""
    return min_val <= value <= max_val

# Create specific validators
validate_age = validate_range(0, 120)
validate_percentage = validate_range(0, 100)

print(validate_age(25))        # True
print(validate_percentage(50)) # True
```

---

## Decorators as Higher-Order Functions

### Function Transformation

```python
from functools import wraps
import time

# Basic decorator
def timer(func):
    """Measure execution time"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "Done"

# Decorator with arguments
def repeat(times):
    """Repeat function execution"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            results = []
            for _ in range(times):
                results.append(func(*args, **kwargs))
            return results
        return wrapper
    return decorator

@repeat(3)
def greet(name):
    return f"Hello, {name}"

print(greet("Alice"))  # ["Hello, Alice", "Hello, Alice", "Hello, Alice"]

# Chaining decorators
@timer
@repeat(5)
def compute():
    return sum(range(1000))

# Executes: timer(repeat(5)(compute))
```

---

## Recursion

### Recursive Functions

```python
# Basic recursion
def factorial(n):
    """Factorial using recursion"""
    if n <= 1:
        return 1
    return n * factorial(n - 1)

# Tail recursion (not optimized in Python)
def factorial_tail(n, accumulator=1):
    """Tail-recursive factorial"""
    if n <= 1:
        return accumulator
    return factorial_tail(n - 1, n * accumulator)

# List processing
def sum_list_recursive(items):
    """Sum list recursively"""
    if not items:
        return 0
    return items[0] + sum_list_recursive(items[1:])

# Tree traversal
class Node:
    def __init__(self, value, children=None):
        self.value = value
        self.children = children or []

def find_node(root, target):
    """Find node in tree recursively"""
    if root.value == target:
        return root

    for child in root.children:
        result = find_node(child, target)
        if result:
            return result

    return None

# Recursion with memoization
from functools import lru_cache

@lru_cache(maxsize=None)
def fibonacci_memo(n):
    """Fibonacci with memoization"""
    if n < 2:
        return n
    return fibonacci_memo(n - 1) + fibonacci_memo(n - 2)

# Much faster than naive recursion
print(fibonacci_memo(100))

# When to use recursion:
# ✅ Tree/graph traversal
# ✅ Divide and conquer algorithms
# ✅ Backtracking problems
# ❌ Simple iterations (use loops)
# ❌ Deep recursion (Python has limit ~1000)
```

---

## Best Practices

### ✅ Do's:

1. **Use pure functions** when possible
2. **Prefer immutability** for shared data
3. **Use list comprehensions** over map/filter
4. **Use built-in functions** (sum, max, min)
5. **Use generator expressions** for large datasets
6. **Combine FP with OOP** - use both paradigms
7. **Document side effects** clearly

### ❌ Don'ts:

1. **Don't force FP** everywhere
2. **Don't sacrifice readability** for functional style
3. **Don't use lambda** for complex logic
4. **Don't ignore Python idioms**
5. **Don't recurse deeply** (stack limit)

---

## Interview Questions

### Q1: What is a pure function?

**Answer**: Function with no side effects, deterministic:

- Same inputs → same output always
- No external state modification
- No I/O operations
- Benefits: Testable, cacheable, parallelizable

### Q2: Explain map, filter, reduce.

**Answer**:

- **map**: Transform each element
- **filter**: Select elements matching condition
- **reduce**: Aggregate to single value
  In Python, prefer list comprehensions for readability.

### Q3: What is a closure?

**Answer**: Function that captures variables from outer scope:

- Inner function accesses outer function's variables
- State persists across calls
- Use for: Memoization, factories, private state

### Q4: What's the difference between map and list comprehension?

**Answer**:

- **map**: Returns iterator, applies function
- **List comp**: More Pythonic, more flexible, more readable
- List comp can filter inline: `[x*2 for x in nums if x > 0]`
  Prefer list comprehension in Python.

### Q5: Explain currying.

**Answer**: Transform f(a,b,c) to f(a)(b)(c):

- Partial application of arguments
- Creates specialized functions
- Enables function composition
- Less common in Python than other FP languages

---

## Summary

Functional programming in Python:

- **Pure functions**: No side effects
- **First-class functions**: Pass/return/store
- **Higher-order functions**: map, filter, reduce
- **Immutability**: Unchangeable data
- **Closures**: Capture outer scope
- **Composition**: Combine functions

Use FP for cleaner, more testable code! 🔄
