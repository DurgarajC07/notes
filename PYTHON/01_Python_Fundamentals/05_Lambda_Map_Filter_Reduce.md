# Lambda, Map, Filter, Reduce - Functional Programming in Python

## 📖 Concept Explanation

Functional programming features in Python allow you to write concise, expressive code. These tools enable declarative programming where you describe _what_ you want, not _how_ to do it.

### Lambda Functions

Anonymous functions defined with `lambda` keyword:

```python
# Regular function
def add(x, y):
    return x + y

# Lambda equivalent
add = lambda x, y: x + y

# Lambda functions are typically used inline
sorted_names = sorted(names, key=lambda x: x.lower())
```

### Map Function

Applies a function to every item in an iterable:

```python
# map(function, iterable)
numbers = [1, 2, 3, 4, 5]
squared = map(lambda x: x ** 2, numbers)
print(list(squared))  # [1, 4, 9, 16, 25]

# Multiple iterables
numbers1 = [1, 2, 3]
numbers2 = [10, 20, 30]
added = map(lambda x, y: x + y, numbers1, numbers2)
print(list(added))  # [11, 22, 33]
```

### Filter Function

Filters items based on a predicate function:

```python
# filter(function, iterable)
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
evens = filter(lambda x: x % 2 == 0, numbers)
print(list(evens))  # [2, 4, 6, 8, 10]

# Filter with None removes falsy values
mixed = [0, 1, False, True, '', 'hello', [], [1], {}, {'a': 1}]
truthy = filter(None, mixed)
print(list(truthy))  # [1, True, 'hello', [1], {'a': 1}]
```

### Reduce Function

Applies a function cumulatively to items, reducing them to a single value:

```python
from functools import reduce

# reduce(function, iterable[, initializer])
numbers = [1, 2, 3, 4, 5]
sum_all = reduce(lambda x, y: x + y, numbers)
print(sum_all)  # 15

# With initial value
product = reduce(lambda x, y: x * y, numbers, 1)
print(product)  # 120

# Find maximum
maximum = reduce(lambda x, y: x if x > y else y, numbers)
print(maximum)  # 5
```

## 🧠 Why It Matters in Real Projects

### 1. Data Transformation Pipelines

```python
# Transform user data from API
raw_users = [
    {'name': 'JOHN', 'age': '25', 'email': 'john@example.com'},
    {'name': 'JANE', 'age': '30', 'email': 'jane@example.com'},
    {'name': 'BOB', 'age': '17', 'email': 'bob@example.com'}
]

# Functional pipeline
processed_users = list(map(
    lambda u: {
        'name': u['name'].title(),
        'age': int(u['age']),
        'email': u['email'].lower()
    },
    filter(lambda u: int(u['age']) >= 18, raw_users)
))

print(processed_users)
# [{'name': 'John', 'age': 25, 'email': 'john@example.com'},
#  {'name': 'Jane', 'age': 30, 'email': 'jane@example.com'}]
```

### 2. Code Conciseness

```python
# Without functional approach
result = []
for item in data:
    if condition(item):
        result.append(transform(item))

# With functional approach
result = list(map(transform, filter(condition, data)))
```

### 3. Declarative Code

```python
# Calculate total price of items in cart
cart = [
    {'name': 'laptop', 'price': 1000, 'quantity': 1},
    {'name': 'mouse', 'price': 25, 'quantity': 2},
    {'name': 'keyboard', 'price': 75, 'quantity': 1}
]

# Declarative and readable
total = reduce(
    lambda acc, item: acc + (item['price'] * item['quantity']),
    cart,
    0
)
print(total)  # 1125
```

## ⚙️ Internal Working

### 1. Lambda Functions are Function Objects

```python
# Lambda creates a function object
f = lambda x: x * 2

print(type(f))  # <class 'function'>
print(f.__name__)  # '<lambda>'

# Equivalent to
def f(x):
    return x * 2

# Limitation: lambda can only contain expressions, not statements
# lambda x: x = 5  # SyntaxError
# lambda x: print(x)  # Works but bad practice
```

### 2. Map Returns Iterator (Lazy Evaluation)

```python
# map() returns an iterator, not a list
numbers = [1, 2, 3, 4, 5]
squared = map(lambda x: x ** 2, numbers)

print(type(squared))  # <class 'map'>
print(squared)  # <map object at 0x...>

# Values computed on-demand
print(list(squared))  # [1, 4, 9, 16, 25]
print(list(squared))  # [] - iterator exhausted!

# Memory efficient for large datasets
import sys
big_list = list(range(1000000))
big_map = map(lambda x: x * 2, big_list)

print(sys.getsizeof(big_list))  # ~8MB
print(sys.getsizeof(big_map))   # ~120 bytes
```

### 3. Filter Returns Iterator

```python
# Like map, filter is lazy
numbers = range(1000000)
evens = filter(lambda x: x % 2 == 0, numbers)

# No computation yet
print(type(evens))  # <class 'filter'>

# Computed on iteration
for i, even in enumerate(evens):
    if i >= 5:
        break
    print(even)  # 0, 2, 4, 6, 8
```

### 4. Reduce Accumulates Sequentially

```python
# Reduce works left-to-right
from functools import reduce

numbers = [1, 2, 3, 4, 5]

# Step-by-step execution
def add_verbose(x, y):
    result = x + y
    print(f"{x} + {y} = {result}")
    return result

total = reduce(add_verbose, numbers)
# Output:
# 1 + 2 = 3
# 3 + 3 = 6
# 6 + 4 = 10
# 10 + 5 = 15
```

## ✅ Best Practices

### 1. Prefer List Comprehensions Over map/filter

```python
# map - readable but not pythonic
squared = list(map(lambda x: x ** 2, numbers))

# List comprehension - more pythonic
squared = [x ** 2 for x in numbers]

# filter - readable but not pythonic
evens = list(filter(lambda x: x % 2 == 0, numbers))

# List comprehension - more pythonic
evens = [x for x in numbers if x % 2 == 0]

# Exception: when you already have a function
def is_valid(item):
    # Complex validation logic
    return True

# Here filter is cleaner than comprehension
valid_items = filter(is_valid, items)
```

### 2. Don't Use Lambda for Complex Logic

```python
# BAD - complex lambda
users = sorted(
    users,
    key=lambda u: (
        u['priority'],
        -u['age'],
        u['name'].lower()
    ) if u['active'] else (999, 0, '')
)

# GOOD - named function
def sort_key(user):
    if not user['active']:
        return (999, 0, '')
    return (user['priority'], -user['age'], user['name'].lower())

users = sorted(users, key=sort_key)
```

### 3. Use sum() Instead of reduce() for Addition

```python
# BAD
from functools import reduce
total = reduce(lambda x, y: x + y, numbers)

# GOOD
total = sum(numbers)

# BAD
total = reduce(lambda x, y: x + y, numbers, 0)

# GOOD
total = sum(numbers, start=0)
```

### 4. Chain Operations for Readability

```python
# Hard to read - nested functions
result = list(map(
    lambda x: x ** 2,
    filter(
        lambda x: x % 2 == 0,
        range(10)
    )
))

# Better - separate steps
evens = filter(lambda x: x % 2 == 0, range(10))
squared = map(lambda x: x ** 2, evens)
result = list(squared)

# Best - generator expression or comprehension
result = [x ** 2 for x in range(10) if x % 2 == 0]
```

### 5. Use operator Module for Simple Operations

```python
from functools import reduce
import operator

numbers = [1, 2, 3, 4, 5]

# AVOID - lambda for simple operations
product = reduce(lambda x, y: x * y, numbers)

# BETTER - operator module
product = reduce(operator.mul, numbers)

# Other useful operators
items = [('a', 2), ('b', 1), ('c', 3)]
sorted_items = sorted(items, key=operator.itemgetter(1))
# [('b', 1), ('a', 2), ('c', 3)]

users = [User('John', 25), User('Jane', 30)]
sorted_users = sorted(users, key=operator.attrgetter('age'))
```

## ❌ Common Mistakes

### 1. Forgetting to Convert Map/Filter to List

```python
# MISTAKE - map returns iterator
numbers = [1, 2, 3, 4, 5]
squared = map(lambda x: x ** 2, numbers)
print(squared)  # <map object> - not the values!

# CORRECT
squared = list(map(lambda x: x ** 2, numbers))
print(squared)  # [1, 4, 9, 16, 25]

# Or use list comprehension
squared = [x ** 2 for x in numbers]
```

### 2. Reusing Exhausted Iterators

```python
# MISTAKE
numbers = [1, 2, 3, 4, 5]
doubled = map(lambda x: x * 2, numbers)

list1 = list(doubled)  # [2, 4, 6, 8, 10]
list2 = list(doubled)  # [] - exhausted!

# CORRECT - recreate or use list
doubled = list(map(lambda x: x * 2, numbers))
list1 = doubled  # [2, 4, 6, 8, 10]
list2 = doubled  # [2, 4, 6, 8, 10]
```

### 3. Using Lambda with Side Effects

```python
# MISTAKE - side effects in lambda
results = []
values = map(lambda x: results.append(x) or x * 2, numbers)
# Side effects happen during iteration, not at map creation!

# CORRECT - use regular loop for side effects
results = []
values = []
for x in numbers:
    results.append(x)
    values.append(x * 2)
```

### 4. Overcomplicating with reduce()

```python
# MISTAKE - using reduce when better alternatives exist
from functools import reduce

# Finding max with reduce
maximum = reduce(lambda x, y: x if x > y else y, numbers)

# BETTER - use built-in
maximum = max(numbers)

# Finding sum with reduce
total = reduce(lambda x, y: x + y, numbers)

# BETTER - use built-in
total = sum(numbers)
```

### 5. Not Understanding Operator Precedence in Lambda

```python
# MISTAKE
f = lambda x: x + 1 ** 2  # Means x + (1 ** 2) = x + 1

# CORRECT - explicit parentheses
f = lambda x: (x + 1) ** 2

# MISTAKE
f = lambda x: x > 0 and x < 10  # Works but

# BETTER - more readable
f = lambda x: 0 < x < 10
```

## 🔐 Security Considerations

### 1. Don't Use Lambda/Map/Filter with Untrusted Code

```python
# DANGEROUS - don't execute user input
user_function = input("Enter lambda function: ")
f = eval(f"lambda x: {user_function}")  # NEVER DO THIS!

# User could input: __import__('os').system('rm -rf /')

# SAFE - whitelist allowed operations
ALLOWED_OPS = {
    'double': lambda x: x * 2,
    'square': lambda x: x ** 2,
    'increment': lambda x: x + 1
}

operation = input("Choose operation: double, square, increment: ")
if operation in ALLOWED_OPS:
    f = ALLOWED_OPS[operation]
```

### 2. Validate Data Before Transformation

```python
# UNSAFE - no validation
user_ages = map(int, user_inputs)

# SAFE - validate first
def safe_int(value):
    try:
        num = int(value)
        if 0 <= num <= 150:
            return num
        raise ValueError("Age out of range")
    except ValueError:
        return None

user_ages = list(filter(None, map(safe_int, user_inputs)))
```

## 🚀 Performance Optimization Techniques

### 1. Use Iterators for Memory Efficiency

```python
# Memory inefficient - creates intermediate lists
numbers = range(1000000)
squared = [x ** 2 for x in numbers]
evens = [x for x in squared if x % 2 == 0]
result = [x for x in evens if x > 1000]

# Memory efficient - generator pipeline
numbers = range(1000000)
squared = (x ** 2 for x in numbers)
evens = (x for x in squared if x % 2 == 0)
result = (x for x in evens if x > 1000)
print(sum(result))  # Computed on-the-fly
```

### 2. Choose Right Tool for the Job

```python
import timeit

numbers = list(range(10000))

# List comprehension - fastest
time1 = timeit.timeit(
    lambda: [x * 2 for x in numbers],
    number=1000
)

# Map with lambda - slower
time2 = timeit.timeit(
    lambda: list(map(lambda x: x * 2, numbers)),
    number=1000
)

# Map with function - faster than lambda
def double(x):
    return x * 2

time3 = timeit.timeit(
    lambda: list(map(double, numbers)),
    number=1000
)

print(f"Comprehension: {time1:.4f}s")  # Fastest
print(f"Map+lambda: {time2:.4f}s")     # Slowest
print(f"Map+function: {time3:.4f}s")   # Middle
```

### 3. Use Built-ins When Available

```python
from functools import reduce
import operator
import timeit

numbers = list(range(1000))

# Reduce with lambda - slower
time1 = timeit.timeit(
    lambda: reduce(lambda x, y: x + y, numbers),
    number=10000
)

# Built-in sum - much faster
time2 = timeit.timeit(
    lambda: sum(numbers),
    number=10000
)

print(f"reduce+lambda: {time1:.4f}s")  # ~0.3s
print(f"sum: {time2:.4f}s")            # ~0.03s
# sum() is 10x faster!
```

### 4. Avoid Unnecessary Conversions

```python
# Inefficient - multiple list conversions
result = list(map(lambda x: x * 2, list(filter(lambda x: x > 0, numbers))))

# Efficient - single conversion at end
filtered = filter(lambda x: x > 0, numbers)
mapped = map(lambda x: x * 2, filtered)
result = list(mapped)

# Most efficient - generator expression (if you need list)
result = [x * 2 for x in numbers if x > 0]
```

## 🧪 Code Examples

### Example 1: Data Processing Pipeline

```python
# Real-world example: Process log entries
log_entries = [
    "2024-01-15 10:30:00 ERROR Connection timeout",
    "2024-01-15 10:30:05 INFO User logged in",
    "2024-01-15 10:30:10 ERROR Database connection failed",
    "2024-01-15 10:30:15 WARNING High memory usage",
    "2024-01-15 10:30:20 INFO User logged out"
]

from datetime import datetime

# Parse log entry
parse_log = lambda entry: {
    'timestamp': datetime.strptime(entry[:19], '%Y-%m-%d %H:%M:%S'),
    'level': entry.split()[2],
    'message': ' '.join(entry.split()[3:])
}

# Filter only errors
is_error = lambda log: log['level'] == 'ERROR'

# Format error message
format_error = lambda log: f"[{log['timestamp']}] {log['message']}"

# Processing pipeline
parsed = map(parse_log, log_entries)
errors = filter(is_error, parsed)
formatted = map(format_error, errors)
error_list = list(formatted)

for error in error_list:
    print(error)
# [2024-01-15 10:30:00] Connection timeout
# [2024-01-15 10:30:10] Database connection failed
```

### Example 2: Complex reduce() Application

```python
from functools import reduce

# Flatten nested lists
nested = [[1, 2], [3, 4], [5, 6]]
flattened = reduce(lambda acc, lst: acc + lst, nested, [])
print(flattened)  # [1, 2, 3, 4, 5, 6]

# Group by key
data = [
    {'category': 'fruit', 'name': 'apple'},
    {'category': 'vegetable', 'name': 'carrot'},
    {'category': 'fruit', 'name': 'banana'},
    {'category': 'vegetable', 'name': 'broccoli'}
]

grouped = reduce(
    lambda acc, item: {
        **acc,
        item['category']: acc.get(item['category'], []) + [item['name']]
    },
    data,
    {}
)
print(grouped)
# {'fruit': ['apple', 'banana'], 'vegetable': ['carrot', 'broccoli']}

# Note: For grouping, itertools.groupby or collections.defaultdict is better
```

### Example 3: Functional Composition

```python
# Compose functions
def compose(*functions):
    """Compose functions right-to-left"""
    return reduce(lambda f, g: lambda x: f(g(x)), functions, lambda x: x)

# Individual transformations
strip = lambda s: s.strip()
lower = lambda s: s.lower()
remove_spaces = lambda s: s.replace(' ', '')
capitalize = lambda s: s.capitalize()

# Compose into single function
clean_name = compose(capitalize, remove_spaces, lower, strip)

print(clean_name("  John DOE  "))  # Johndoe

# Another example: data validation pipeline
def not_empty(x): return x if x else None
def to_int(x): return int(x) if x else None
def in_range(x): return x if x and 0 <= x <= 100 else None

validate_score = compose(in_range, to_int, not_empty)

print(validate_score("85"))    # 85
print(validate_score("150"))   # None
print(validate_score(""))      # None
```

## 🏗️ Real-World Use Cases

### 1. API Response Transformation

```python
# Transform API response to frontend format
api_users = [
    {
        'id': 1,
        'first_name': 'John',
        'last_name': 'Doe',
        'email': 'john@example.com',
        'is_active': True,
        'created_at': '2024-01-15T10:30:00Z',
        'roles': ['user', 'admin']
    },
    # ... more users
]

# Transformation pipeline
transform_user = lambda user: {
    'id': user['id'],
    'fullName': f"{user['first_name']} {user['last_name']}",
    'email': user['email'],
    'status': 'active' if user['is_active'] else 'inactive',
    'roles': list(map(str.upper, user['roles']))
}

# Filter active users only
is_active = lambda user: user['is_active']

# Process
active_users = filter(is_active, api_users)
transformed = map(transform_user, active_users)
response = list(transformed)
```

### 2. Configuration Validation

```python
# Validate environment configuration
config_rules = [
    ('DATABASE_URL', lambda x: x and x.startswith('postgresql://')),
    ('SECRET_KEY', lambda x: x and len(x) >= 32),
    ('DEBUG', lambda x: x in ['True', 'False']),
    ('MAX_CONNECTIONS', lambda x: x and x.isdigit() and int(x) > 0)
]

env_config = {
    'DATABASE_URL': 'postgresql://localhost/db',
    'SECRET_KEY': 'a' * 32,
    'DEBUG': 'False',
    'MAX_CONNECTIONS': '10'
}

# Validate all rules
validate_rule = lambda rule: (rule[0], rule[1](env_config.get(rule[0])))
validation_results = list(map(validate_rule, config_rules))

# Check if all valid
all_valid = reduce(lambda acc, result: acc and result[1], validation_results, True)

if not all_valid:
    invalid = filter(lambda x: not x[1], validation_results)
    print("Invalid config:", list(invalid))
```

### 3. Data Aggregation

```python
from functools import reduce

# Calculate statistics from transactions
transactions = [
    {'amount': 100, 'type': 'credit', 'category': 'salary'},
    {'amount': 50, 'type': 'debit', 'category': 'food'},
    {'amount': 200, 'type': 'credit', 'category': 'bonus'},
    {'amount': 30, 'type': 'debit', 'category': 'transport'},
    {'amount': 80, 'type': 'debit', 'category': 'food'}
]

# Calculate totals
stats = reduce(
    lambda acc, t: {
        'total_credit': acc['total_credit'] + (t['amount'] if t['type'] == 'credit' else 0),
        'total_debit': acc['total_debit'] + (t['amount'] if t['type'] == 'debit' else 0),
        'transactions': acc['transactions'] + 1
    },
    transactions,
    {'total_credit': 0, 'total_debit': 0, 'transactions': 0}
)

stats['balance'] = stats['total_credit'] - stats['total_debit']
print(stats)
# {'total_credit': 300, 'total_debit': 160, 'transactions': 5, 'balance': 140}
```

## ❓ Frequently Asked Interview Questions

### Q1: What is the difference between map() and list comprehension?

**Answer:**

- `map()` returns an iterator (lazy evaluation)
- List comprehension returns a list immediately
- List comprehension is generally more pythonic

```python
# map - returns iterator
mapped = map(lambda x: x ** 2, range(5))
print(type(mapped))  # <class 'map'>
print(list(mapped))  # [0, 1, 4, 9, 16]

# List comprehension - returns list
comprehension = [x ** 2 for x in range(5)]
print(type(comprehension))  # <class 'list'>
print(comprehension)  # [0, 1, 4, 9, 16]

# When to use map():
# 1. You already have a named function
def complex_transform(x):
    # ... complex logic
    return result

result = map(complex_transform, data)  # Cleaner

# 2. You need an iterator for memory efficiency
# Don't use: list(map(...)) - defeats the purpose
for item in map(transform, huge_dataset):  # Memory efficient
    process(item)
```

### Q2: Explain lazy evaluation in map/filter

**Answer:**
`map()` and `filter()` return iterators that compute values on-demand, not upfront.

```python
def square(x):
    print(f"Squaring {x}")
    return x ** 2

numbers = [1, 2, 3, 4, 5]

# No computation yet!
squared = map(square, numbers)
print("Map created")

# Computation happens during iteration
print("Starting iteration")
for num in squared:
    print(num)
    if num > 10:
        break  # Can stop early without computing all values

# Output:
# Map created
# Starting iteration
# Squaring 1
# 1
# Squaring 2
# 4
# Squaring 3
# 9
# Squaring 4
# 16
# (stops here, 5 never squared)
```

### Q3: When should you use reduce()?

**Answer:**
Use `reduce()` when you need to cumulatively apply a binary function to reduce a sequence to a single value.

**Good use cases:**

```python
from functools import reduce

# 1. Complex aggregations
data = [{'value': 10}, {'value': 20}, {'value': 30}]
total = reduce(lambda acc, item: acc + item['value'], data, 0)

# 2. Composing functions
functions = [f1, f2, f3]
composed = reduce(lambda f, g: lambda x: f(g(x)), functions)

# 3. Flattening nested structures
nested = [[1, 2], [3, 4], [5, 6]]
flat = reduce(lambda acc, lst: acc + lst, nested, [])
```

**Bad use cases (use built-ins instead):**

```python
# BAD - use sum()
total = reduce(lambda x, y: x + y, numbers)
total = sum(numbers)  # Better

# BAD - use max()
maximum = reduce(lambda x, y: x if x > y else y, numbers)
maximum = max(numbers)  # Better

# BAD - use any()
has_positive = reduce(lambda acc, x: acc or x > 0, numbers, False)
has_positive = any(x > 0 for x in numbers)  # Better
```

### Q4: What are the limitations of lambda functions?

**Answer:**

1. **Single expression only** - no statements
2. **No annotations** - can't add type hints
3. **Limited debugging** - unnamed, harder to debug
4. **Less readable** for complex logic

```python
# ❌ LIMITATIONS

# 1. No statements
# lambda x: x = 5  # SyntaxError
# lambda x: print(x); return x  # SyntaxError

# 2. No type hints
# lambda x: int -> x * 2  # SyntaxError
# Can't do: lambda x: int -> int: x * 2

# 3. Unnamed (harder to debug)
f = lambda x: x / 0  # Traceback shows '<lambda>'

# 4. Complex logic is unreadable
# BAD
calculate = lambda x, y: (
    x + y if x > 0 and y > 0
    else x - y if x < 0 or y < 0
    else x * y
)

# GOOD - use regular function
def calculate(x, y):
    if x > 0 and y > 0:
        return x + y
    elif x < 0 or y < 0:
        return x - y
    else:
        return x * y
```

### Q5: How do you handle None values in map/filter?

**Answer:**

```python
data = ['10', '20', None, '30', '', '40']

# Problem: None causes TypeError
# result = list(map(int, data))  # TypeError

# Solution 1: Filter out None first
filtered = filter(lambda x: x is not None and x != '', data)
result = list(map(int, filtered))
# [10, 20, 30, 40]

# Solution 2: Handle in map
safe_int = lambda x: int(x) if x and x != '' else None
result = list(filter(None, map(safe_int, data)))
# [10, 20, 30, 40]

# Solution 3: Try-except in function
def safe_int_convert(value):
    try:
        return int(value) if value else None
    except (ValueError, TypeError):
        return None

result = list(filter(None, map(safe_int_convert, data)))
# [10, 20, 30, 40]
```

### Q6: Compare functional vs. imperative approaches

**Answer:**

```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# Imperative - how to do it
result = []
for num in numbers:
    if num % 2 == 0:
        squared = num ** 2
        if squared > 20:
            result.append(squared)

# Functional - what to do
result = list(filter(
    lambda x: x > 20,
    map(lambda x: x ** 2, filter(lambda x: x % 2 == 0, numbers))
))

# Most Pythonic - comprehension
result = [x ** 2 for x in numbers if x % 2 == 0 and x ** 2 > 20]

# All produce: [36, 64, 100]

# Tradeoffs:
# Imperative: More verbose, easier to debug, step-through
# Functional: Concise, composable, harder to debug
# Comprehension: Python-idiomatic, readable, performant
```

### Q7: How do map/filter compare in performance?

**Answer:**

```python
import timeit

numbers = list(range(10000))

# map + lambda
time1 = timeit.timeit(
    lambda: list(map(lambda x: x * 2, numbers)),
    number=1000
)

# List comprehension
time2 = timeit.timeit(
    lambda: [x * 2 for x in numbers],
    number=1000
)

# Generator expression
time3 = timeit.timeit(
    lambda: list(x * 2 for x in numbers),
    number=1000
)

print(f"map+lambda: {time1:.4f}s")        # ~0.45s
print(f"comprehension: {time2:.4f}s")     # ~0.35s
print(f"generator: {time3:.4f}s")         # ~0.40s

# Comprehension fastest due to:
# 1. No function call overhead
# 2. Optimized in C
# 3. Direct iteration

# Use map() when:
# 1. You already have a named function
# 2. Code is clearer with map
# 3. Memory efficiency matters (keep as iterator)
```

### Q8: Explain partial function application with map

**Answer:**

```python
from functools import partial

# Problem: Need to pass extra arguments to map
def power(base, exponent):
    return base ** exponent

numbers = [2, 3, 4, 5]

# Can't do: map(power, numbers, 3) - wrong!
# map only passes items from iterable

# Solution 1: Lambda
squared = list(map(lambda x: power(x, 2), numbers))

# Solution 2: Partial (better)
square = partial(power, exponent=2)
squared = list(map(square, numbers))
# [4, 9, 16, 25]

# Another example: API calls with config
def api_request(url, timeout, verify):
    # Make request
    pass

# Configure once, use multiple times
safe_request = partial(api_request, timeout=30, verify=True)
results = map(safe_request, urls)  # Clean!
```

---

**Next**: [06_Comprehensions.md](06_Comprehensions.md)
