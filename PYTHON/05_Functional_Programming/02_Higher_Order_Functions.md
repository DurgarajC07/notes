# Higher-Order Functions in Python

## 📖 Concept Overview

A **higher-order function** is a function that either:

1. Takes one or more functions as arguments, OR
2. Returns a function as its result

Higher-order functions are a cornerstone of functional programming, enabling code reuse, abstraction, and composition.

## 🧠 Why It Matters

- **Code Reusability**: Write generic functions that work with different behaviors
- **Abstraction**: Separate "what to do" from "how to do it"
- **Composition**: Build complex operations from simple functions
- **Declarative Code**: Express intent clearly without implementation details
- **Testing**: Easier to test pure functions passed as arguments

## ⚙️ Internal Working

In Python, functions are **first-class objects**:

- Functions can be assigned to variables
- Functions can be passed as arguments
- Functions can be returned from other functions
- Functions can be stored in data structures

```python
def greet(name):
    return f"Hello, {name}!"

# Function as first-class object
my_func = greet              # Assign to variable
print(my_func("Alice"))      # Hello, Alice!

# Store in data structure
functions = [greet, len, str.upper]
```

## 🧪 Built-in Higher-Order Functions

### 1. map() - Transform Elements

```python
# map(function, iterable) - Apply function to each element

numbers = [1, 2, 3, 4, 5]

# Square each number
squared = list(map(lambda x: x**2, numbers))
print(squared)  # [1, 4, 9, 16, 25]

# Multiple iterables
nums1 = [1, 2, 3]
nums2 = [10, 20, 30]
result = list(map(lambda x, y: x + y, nums1, nums2))
print(result)  # [11, 22, 33]

# With custom function
def fahrenheit_to_celsius(f):
    return (f - 32) * 5/9

temps_f = [32, 68, 98.6, 212]
temps_c = list(map(fahrenheit_to_celsius, temps_f))
print(temps_c)  # [0.0, 20.0, 37.0, 100.0]
```

### 2. filter() - Select Elements

```python
# filter(function, iterable) - Keep elements where function returns True

numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# Filter even numbers
evens = list(filter(lambda x: x % 2 == 0, numbers))
print(evens)  # [2, 4, 6, 8, 10]

# Filter valid emails
emails = ["user@example.com", "invalid", "test@test.com", "bad@"]
valid_emails = list(filter(lambda e: '@' in e and '.' in e.split('@')[1], emails))
print(valid_emails)  # ['user@example.com', 'test@test.com']

# Complex filtering
users = [
    {'name': 'Alice', 'age': 25, 'active': True},
    {'name': 'Bob', 'age': 17, 'active': False},
    {'name': 'Charlie', 'age': 30, 'active': True},
]

active_adults = list(filter(lambda u: u['active'] and u['age'] >= 18, users))
print([u['name'] for u in active_adults])  # ['Alice', 'Charlie']
```

### 3. reduce() - Aggregate Elements

```python
from functools import reduce

# reduce(function, iterable, initial) - Accumulate values

numbers = [1, 2, 3, 4, 5]

# Sum all numbers
total = reduce(lambda acc, x: acc + x, numbers)
print(total)  # 15

# With initial value
total_with_init = reduce(lambda acc, x: acc + x, numbers, 10)
print(total_with_init)  # 25

# Find maximum
max_val = reduce(lambda a, b: a if a > b else b, numbers)
print(max_val)  # 5

# Multiply all numbers
product = reduce(lambda a, b: a * b, numbers)
print(product)  # 120

# Complex reduction: flatten list of lists
lists = [[1, 2], [3, 4], [5, 6]]
flattened = reduce(lambda acc, lst: acc + lst, lists, [])
print(flattened)  # [1, 2, 3, 4, 5, 6]
```

### 4. sorted() - Sort with Custom Key

```python
# sorted(iterable, key=function, reverse=False)

users = [
    {'name': 'Alice', 'age': 25},
    {'name': 'Bob', 'age': 17},
    {'name': 'Charlie', 'age': 30},
]

# Sort by age
by_age = sorted(users, key=lambda u: u['age'])
print([u['name'] for u in by_age])  # ['Bob', 'Alice', 'Charlie']

# Sort by name length
words = ['apple', 'pie', 'banana', 'kiwi']
by_length = sorted(words, key=len)
print(by_length)  # ['pie', 'kiwi', 'apple', 'banana']

# Multiple sort criteria
students = [
    {'name': 'Alice', 'grade': 'A', 'age': 20},
    {'name': 'Bob', 'grade': 'B', 'age': 19},
    {'name': 'Charlie', 'grade': 'A', 'age': 19},
]

# Sort by grade then age
sorted_students = sorted(students, key=lambda s: (s['grade'], s['age']))
print([(s['name'], s['grade'], s['age']) for s in sorted_students])
```

## 🎯 Creating Higher-Order Functions

### 1. Functions that Accept Functions

```python
def apply_operation(numbers: list, operation) -> list:
    """Apply an operation to each number"""
    return [operation(num) for num in numbers]

def square(x):
    return x ** 2

def cube(x):
    return x ** 3

numbers = [1, 2, 3, 4, 5]
print(apply_operation(numbers, square))  # [1, 4, 9, 16, 25]
print(apply_operation(numbers, cube))    # [1, 8, 27, 64, 125]
```

### 2. Functions that Return Functions (Closures)

```python
def multiplier(factor):
    """Returns a function that multiplies by factor"""
    def multiply(x):
        return x * factor
    return multiply

double = multiplier(2)
triple = multiplier(3)

print(double(5))  # 10
print(triple(5))  # 15

# With lambda
def power_function(exponent):
    return lambda x: x ** exponent

square = power_function(2)
cube = power_function(3)

print(square(4))  # 16
print(cube(4))    # 64
```

### 3. Function Composition

```python
def compose(f, g):
    """Compose two functions: (f ∘ g)(x) = f(g(x))"""
    return lambda x: f(g(x))

def add_five(x):
    return x + 5

def multiply_by_two(x):
    return x * 2

# Compose: first multiply by 2, then add 5
combined = compose(add_five, multiply_by_two)
print(combined(10))  # (10 * 2) + 5 = 25

# Multiple composition
def compose_all(*functions):
    """Compose multiple functions from right to left"""
    def inner(x):
        result = x
        for func in reversed(functions):
            result = func(result)
        return result
    return inner

add_ten = lambda x: x + 10
square = lambda x: x ** 2
halve = lambda x: x / 2

pipeline = compose_all(add_ten, square, halve)
print(pipeline(4))  # ((4 / 2) ** 2) + 10 = 14
```

## 🏗️ Real-World Example: Data Processing Pipeline

```python
from typing import Callable, List, Any
from functools import reduce

class DataPipeline:
    """Functional data processing pipeline"""

    def __init__(self, data: List[Any]):
        self.data = data
        self.transformations: List[Callable] = []

    def map(self, func: Callable) -> 'DataPipeline':
        """Add map transformation"""
        self.transformations.append(lambda data: list(map(func, data)))
        return self

    def filter(self, func: Callable) -> 'DataPipeline':
        """Add filter transformation"""
        self.transformations.append(lambda data: list(filter(func, data)))
        return self

    def reduce(self, func: Callable, initial=None) -> 'DataPipeline':
        """Add reduce transformation"""
        if initial is not None:
            self.transformations.append(lambda data: [reduce(func, data, initial)])
        else:
            self.transformations.append(lambda data: [reduce(func, data)])
        return self

    def execute(self) -> Any:
        """Execute all transformations"""
        result = self.data
        for transformation in self.transformations:
            result = transformation(result)
        return result[0] if len(result) == 1 and not isinstance(self.data, dict) else result

# Usage example
transactions = [
    {'id': 1, 'amount': 100, 'type': 'debit'},
    {'id': 2, 'amount': 50, 'type': 'credit'},
    {'id': 3, 'amount': 200, 'type': 'debit'},
    {'id': 4, 'amount': 75, 'type': 'credit'},
    {'id': 5, 'amount': 150, 'type': 'debit'},
]

# Calculate total debits over 100
total = (DataPipeline(transactions)
         .filter(lambda t: t['type'] == 'debit')
         .filter(lambda t: t['amount'] > 100)
         .map(lambda t: t['amount'])
         .reduce(lambda acc, x: acc + x, 0)
         .execute())

print(f"Total debits over 100: {total}")  # 350
```

## 🏢 Real-World Example: Django Query Builder

```python
from typing import Callable, List, Dict, Any

class QueryBuilder:
    """Functional query builder pattern"""

    def __init__(self, data: List[Dict[str, Any]]):
        self.data = data
        self.operations: List[Callable] = []

    def where(self, predicate: Callable[[Dict], bool]) -> 'QueryBuilder':
        """Filter records"""
        self.operations.append(lambda data: [item for item in data if predicate(item)])
        return self

    def select(self, *fields: str) -> 'QueryBuilder':
        """Select specific fields"""
        self.operations.append(
            lambda data: [{field: item[field] for field in fields if field in item}
                         for item in data]
        )
        return self

    def order_by(self, field: str, reverse: bool = False) -> 'QueryBuilder':
        """Sort records"""
        self.operations.append(
            lambda data: sorted(data, key=lambda x: x.get(field), reverse=reverse)
        )
        return self

    def limit(self, count: int) -> 'QueryBuilder':
        """Limit results"""
        self.operations.append(lambda data: data[:count])
        return self

    def distinct(self, field: str) -> 'QueryBuilder':
        """Get distinct values"""
        seen = set()
        def unique_filter(data):
            result = []
            for item in data:
                value = item.get(field)
                if value not in seen:
                    seen.add(value)
                    result.append(item)
            return result
        self.operations.append(unique_filter)
        return self

    def execute(self) -> List[Dict[str, Any]]:
        """Execute query"""
        result = self.data
        for operation in self.operations:
            result = operation(result)
        return result

# Usage
users = [
    {'id': 1, 'name': 'Alice', 'age': 25, 'city': 'New York', 'active': True},
    {'id': 2, 'name': 'Bob', 'age': 30, 'city': 'Boston', 'active': True},
    {'id': 3, 'name': 'Charlie', 'age': 25, 'city': 'New York', 'active': False},
    {'id': 4, 'name': 'David', 'age': 35, 'city': 'Seattle', 'active': True},
    {'id': 5, 'name': 'Eve', 'age': 30, 'city': 'Boston', 'active': True},
]

# Query: Active users from Boston, ordered by age, select name and age
result = (QueryBuilder(users)
          .where(lambda u: u['active'])
          .where(lambda u: u['city'] == 'Boston')
          .order_by('age')
          .select('name', 'age')
          .execute())

print(result)  # [{'name': 'Bob', 'age': 30}, {'name': 'Eve', 'age': 30}]
```

## 🏢 Real-World Example: Validation Pipeline

```python
from typing import Callable, List, Tuple, Optional

class ValidationPipeline:
    """Functional validation using higher-order functions"""

    def __init__(self):
        self.validators: List[Callable[[Any], Tuple[bool, str]]] = []

    def add_validator(self, validator: Callable) -> 'ValidationPipeline':
        """Add a validator function"""
        self.validators.append(validator)
        return self

    def validate(self, value: Any) -> Tuple[bool, List[str]]:
        """Run all validators"""
        errors = []
        for validator in self.validators:
            is_valid, error_msg = validator(value)
            if not is_valid:
                errors.append(error_msg)
        return len(errors) == 0, errors

# Validator factory functions
def min_length(length: int) -> Callable:
    """Returns validator for minimum length"""
    def validator(value: str) -> Tuple[bool, str]:
        is_valid = len(value) >= length
        return is_valid, f"Must be at least {length} characters" if not is_valid else ""
    return validator

def max_length(length: int) -> Callable:
    """Returns validator for maximum length"""
    def validator(value: str) -> Tuple[bool, str]:
        is_valid = len(value) <= length
        return is_valid, f"Must be at most {length} characters" if not is_valid else ""
    return validator

def contains_digit() -> Callable:
    """Returns validator for digit presence"""
    def validator(value: str) -> Tuple[bool, str]:
        is_valid = any(c.isdigit() for c in value)
        return is_valid, "Must contain at least one digit" if not is_valid else ""
    return validator

def contains_special_char() -> Callable:
    """Returns validator for special character"""
    def validator(value: str) -> Tuple[bool, str]:
        special_chars = "!@#$%^&*"
        is_valid = any(c in special_chars for c in value)
        return is_valid, "Must contain special character" if not is_valid else ""
    return validator

# Usage: Password validation
password_validator = (ValidationPipeline()
                     .add_validator(min_length(8))
                     .add_validator(max_length(20))
                     .add_validator(contains_digit())
                     .add_validator(contains_special_char()))

# Test passwords
passwords = ["weak", "StrongPass123!", "short1!", "VeryLongPasswordWithoutSpecialChar123"]

for pwd in passwords:
    is_valid, errors = password_validator.validate(pwd)
    print(f"\n'{pwd}': {'✓ Valid' if is_valid else '✗ Invalid'}")
    if errors:
        for error in errors:
            print(f"  - {error}")
```

## 🚀 Performance Considerations

### Generator Expressions vs map/filter

```python
import sys

# List comprehension (eager)
numbers = range(1000000)
squared_list = [x**2 for x in numbers if x % 2 == 0]
print(f"List size: {sys.getsizeof(squared_list)} bytes")

# Generator expression (lazy)
squared_gen = (x**2 for x in numbers if x % 2 == 0)
print(f"Generator size: {sys.getsizeof(squared_gen)} bytes")

# map + filter (lazy in Python 3)
squared_map = map(lambda x: x**2, filter(lambda x: x % 2 == 0, numbers))
print(f"Map/filter size: {sys.getsizeof(squared_map)} bytes")
```

### When to Use Each Approach

```python
# ✅ Use list comprehension for small datasets
small_data = [1, 2, 3, 4, 5]
result = [x * 2 for x in small_data if x > 2]

# ✅ Use generator for large datasets
large_data = range(1000000)
result_gen = (x * 2 for x in large_data if x > 2)

# ✅ Use map/filter for function reuse
def is_even(x): return x % 2 == 0
def square(x): return x ** 2

result_functional = map(square, filter(is_even, large_data))
```

## ✅ Best Practices

### 1. Prefer Comprehensions for Simple Cases

```python
# ❌ Less readable
result = list(map(lambda x: x * 2, filter(lambda x: x > 0, numbers)))

# ✅ More Pythonic
result = [x * 2 for x in numbers if x > 0]
```

### 2. Use Higher-Order Functions for Reusable Logic

```python
# ✅ Good: Reusable transformation
def apply_discount(discount_percent):
    return lambda price: price * (1 - discount_percent / 100)

apply_10_percent = apply_discount(10)
apply_20_percent = apply_discount(20)

prices = [100, 200, 300]
print(list(map(apply_10_percent, prices)))  # [90.0, 180.0, 270.0]
```

### 3. Chain Operations for Clarity

```python
from functools import reduce

# ✅ Good: Clear pipeline
users = [{'name': 'Alice', 'age': 25}, {'name': 'Bob', 'age': 30}]

result = reduce(
    lambda acc, u: acc + u['age'],
    filter(lambda u: u['age'] > 20, users),
    0
)
```

## ❌ Common Mistakes

### 1. Overusing Lambda When Named Functions Are Better

```python
# ❌ Bad: Complex lambda
sorted_users = sorted(users, key=lambda u: (u['last_name'], u['first_name'], -u['age']))

# ✅ Good: Named function
def user_sort_key(user):
    return (user['last_name'], user['first_name'], -user['age'])

sorted_users = sorted(users, key=user_sort_key)
```

### 2. Not Considering Memory with Large Datasets

```python
# ❌ Bad: Creates large list in memory
result = list(map(expensive_operation, range(1000000)))

# ✅ Good: Process lazily
result_gen = map(expensive_operation, range(1000000))
for item in result_gen:
    process(item)
```

### 3. Mutating Data in map/filter

```python
# ❌ Bad: Side effects in map
def add_processed_flag(user):
    user['processed'] = True  # Mutation!
    return user

list(map(add_processed_flag, users))

# ✅ Good: Return new objects
def with_processed_flag(user):
    return {**user, 'processed': True}

result = list(map(with_processed_flag, users))
```

## ❓ Interview Questions

### Q1: What's the difference between map/filter and list comprehensions?

**Answer**:

- **Performance**: Similar in Python 3 (both lazy with generators)
- **Readability**: Comprehensions often more Pythonic and readable
- **Functionality**: Comprehensions can combine map + filter
- **Use case**: map/filter better for function reuse, comprehensions for inline logic

```python
# Equivalent operations
result1 = [x * 2 for x in nums if x > 0]
result2 = list(map(lambda x: x * 2, filter(lambda x: x > 0, nums)))
```

### Q2: Explain how reduce() works with an example

**Answer**: `reduce()` applies a function cumulatively to items from left to right to reduce to single value.

```python
from functools import reduce

# Sum: ((((1 + 2) + 3) + 4) + 5)
result = reduce(lambda acc, x: acc + x, [1, 2, 3, 4, 5])
# Step 1: 1 + 2 = 3
# Step 2: 3 + 3 = 6
# Step 3: 6 + 4 = 10
# Step 4: 10 + 5 = 15
```

### Q3: What is a closure and why is it useful?

**Answer**: A closure is a function that remembers values from its enclosing scope even after the outer function has finished executing.

```python
def counter():
    count = 0
    def increment():
        nonlocal count
        count += 1
        return count
    return increment

c = counter()
print(c())  # 1
print(c())  # 2
```

Useful for: factory functions, decorators, callbacks, data hiding.

### Q4: When should you use reduce() vs a for loop?

**Answer**:

- **Use reduce()**: When accumulating a single result, functional style preferred
- **Use for loop**: When logic is complex, multiple operations needed, or readability suffers

```python
# Good use of reduce
total = reduce(lambda acc, x: acc + x, numbers)

# Better as for loop (complex logic)
result = {}
for item in items:
    if complex_condition(item):
        result[item.key] = transform(item)
```

### Q5: What are the benefits of higher-order functions?

**Answer**:

1. **Abstraction**: Separate "what" from "how"
2. **Reusability**: Generic functions work with different behaviors
3. **Composition**: Build complex operations from simple functions
4. **Testing**: Easier to test pure functions
5. **Declarative**: Express intent clearly

## 📚 Summary

### Key Takeaways

1. **Higher-order functions** accept functions as arguments or return functions
2. **Built-in HOFs**: `map()`, `filter()`, `reduce()`, `sorted()`
3. **Closures** enable factory functions and data hiding
4. **Function composition** builds complex operations from simple ones
5. **Comprehensions** often more Pythonic than map/filter
6. **Lazy evaluation** with generators improves memory efficiency
7. **Pure functions** in HOFs enable better testing and reasoning

### When to Use Higher-Order Functions

✅ **Use when**:

- Building reusable transformations
- Creating pipelines or chains of operations
- Need function composition
- Want declarative, functional style
- Implementing design patterns (Strategy, Template Method)

❌ **Avoid when**:

- Simple operations (use comprehensions)
- Complex logic (use explicit loops)
- Performance is critical (profile first)
- Team unfamiliar with functional patterns

### Function Selection Guide

| Operation              | Best Choice        | Alternative |
| ---------------------- | ------------------ | ----------- |
| Transform all elements | List comprehension | `map()`     |
| Filter elements        | List comprehension | `filter()`  |
| Accumulate/aggregate   | `reduce()`         | for loop    |
| Sort with key          | `sorted(key=...)`  | Custom sort |
| Multiple operations    | Generator pipeline | Nested HOFs |
