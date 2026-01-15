# Python Data Types - Complete Guide

## 📖 Concept Explanation

Python has several built-in data types that are fundamental to every program. Understanding their behavior, mutability, and performance characteristics is crucial.

### Core Data Types

```python
# Numeric Types
int_val = 42                    # Integer (arbitrary precision)
float_val = 3.14159            # Float (double precision)
complex_val = 3 + 4j           # Complex number

# Text Type
str_val = "Hello, World!"      # String (immutable sequence)

# Boolean Type
bool_val = True                # Boolean (subclass of int)

# None Type
none_val = None                # NoneType (null equivalent)

# Sequence Types
list_val = [1, 2, 3]           # List (mutable)
tuple_val = (1, 2, 3)          # Tuple (immutable)
range_val = range(10)          # Range (immutable sequence)

# Mapping Type
dict_val = {"key": "value"}    # Dictionary (mutable)

# Set Types
set_val = {1, 2, 3}            # Set (mutable, unordered, unique)
frozenset_val = frozenset([1, 2, 3])  # Frozen set (immutable)

# Binary Types
bytes_val = b"hello"           # Bytes (immutable)
bytearray_val = bytearray(b"hello")  # Bytearray (mutable)
memoryview_val = memoryview(bytes_val)  # Memory view
```

## 🧠 Why It Matters in Real Projects

### 1. Performance Impact

```python
# Lists vs Tuples: Memory and speed
import sys

list_data = [1, 2, 3, 4, 5]
tuple_data = (1, 2, 3, 4, 5)

print(sys.getsizeof(list_data))   # 104 bytes
print(sys.getsizeof(tuple_data))  # 80 bytes

# Tuples are faster for iteration and memory-efficient
```

### 2. Data Integrity

```python
# Immutable types prevent accidental modifications
USER_ROLES = ('admin', 'moderator', 'user')  # Can't be modified
# USER_ROLES[0] = 'superadmin'  # TypeError

# Mutable types allow changes (use when needed)
active_users = ['user1', 'user2']
active_users.append('user3')  # OK
```

### 3. Dictionary Keys Must Be Hashable

```python
# Valid keys (immutable types)
valid_dict = {
    "string_key": 1,
    42: 2,
    (1, 2): 3,
    frozenset([1, 2]): 4
}

# Invalid keys (mutable types)
# invalid_dict = {[1, 2]: "value"}  # TypeError: unhashable type: 'list'
```

## ⚙️ Internal Working

### 1. Integer Implementation

```python
# Python integers have arbitrary precision (no overflow)
big_num = 10 ** 100
print(big_num)  # Works fine

# CPython optimization: Small integers (-5 to 256) are cached
a = 256
b = 256
print(a is b)  # True (same object)

a = 257
b = 257
print(a is b)  # False (different objects, though might be True in interactive mode)
```

### 2. String Interning

```python
# Python interns certain strings for efficiency
s1 = "hello"
s2 = "hello"
print(s1 is s2)  # True (same object)

# But not all strings
s1 = "hello world!"
s2 = "hello world!"
print(s1 is s2)  # May be False

# Force interning
import sys
s1 = sys.intern("hello world!")
s2 = sys.intern("hello world!")
print(s1 is s2)  # True
```

### 3. List Memory Layout

```python
# Lists store references (pointers) to objects
import sys

# Empty list
empty_list = []
print(sys.getsizeof(empty_list))  # 56 bytes (overhead)

# Adding elements
one_item = [1]
print(sys.getsizeof(one_item))  # 64 bytes

# Lists over-allocate to reduce reallocation cost
many_items = list(range(100))
print(sys.getsizeof(many_items))  # 920 bytes (not just 56 + 8*100)
```

### 4. Dictionary Hash Tables

```python
# Dictionaries use hash tables (Python 3.7+ maintains insertion order)
d = {'a': 1, 'b': 2, 'c': 3}

# Hash function determines bucket
print(hash('a'))  # Varies per session

# Collision resolution uses open addressing
```

## ✅ Best Practices

### 1. Choose the Right Data Type

```python
# Use tuples for fixed collections
coordinates = (10.5, 20.3)  # Geographic point (immutable)

# Use lists for dynamic collections
shopping_cart = ['apple', 'banana']
shopping_cart.append('orange')

# Use sets for membership testing
allowed_ips = {'192.168.1.1', '192.168.1.2'}
if user_ip in allowed_ips:  # O(1) average case
    grant_access()

# Use frozensets for immutable sets (hashable)
admin_roles = frozenset(['admin', 'superadmin'])
permissions = {admin_roles: 'full_access'}  # Valid dictionary key
```

### 2. Type Annotations for Clarity

```python
from typing import List, Dict, Set, Tuple, Optional, Union

def process_user_data(
    users: List[Dict[str, Union[str, int]]],
    active_ids: Set[int],
    default_role: Optional[str] = None
) -> Tuple[int, int]:
    """Process user data with clear type hints"""
    active_count = 0
    inactive_count = 0

    for user in users:
        if user['id'] in active_ids:
            active_count += 1
        else:
            inactive_count += 1

    return active_count, inactive_count
```

### 3. Use Named Tuples for Better Readability

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float

    def distance_from_origin(self) -> float:
        return (self.x ** 2 + self.y ** 2) ** 0.5

# Much better than regular tuple
point = Point(3.0, 4.0)
print(point.x)  # Named access
print(point.distance_from_origin())  # 5.0
```

### 4. Avoid Mutable Default Arguments

```python
# BAD - mutable default argument
def add_item(item, items=[]):
    items.append(item)
    return items

# Problem: list is shared across calls
print(add_item(1))  # [1]
print(add_item(2))  # [1, 2] - BUG!

# GOOD - use None as default
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

print(add_item(1))  # [1]
print(add_item(2))  # [2] - Correct!
```

## ❌ Common Mistakes

### 1. Modifying List While Iterating

```python
# BAD - modifies list during iteration
numbers = [1, 2, 3, 4, 5]
for num in numbers:
    if num % 2 == 0:
        numbers.remove(num)  # Skips elements!

# GOOD - create new list
numbers = [1, 2, 3, 4, 5]
numbers = [num for num in numbers if num % 2 != 0]

# GOOD - iterate over copy
numbers = [1, 2, 3, 4, 5]
for num in numbers[:]:  # Slice creates a copy
    if num % 2 == 0:
        numbers.remove(num)
```

### 2. Confusing Shallow vs Deep Copy

```python
# Shallow copy - nested objects are shared
import copy

original = [[1, 2], [3, 4]]
shallow = original.copy()  # or copy.copy(original)

shallow[0][0] = 999
print(original)  # [[999, 2], [3, 4]] - MODIFIED!

# Deep copy - complete independent copy
original = [[1, 2], [3, 4]]
deep = copy.deepcopy(original)

deep[0][0] = 999
print(original)  # [[1, 2], [3, 4]] - Unchanged
```

### 3. Using `==` Instead of `is` for None

```python
# BAD
if value == None:
    pass

# GOOD - use identity check
if value is None:
    pass

# Why? None is a singleton
none1 = None
none2 = None
print(none1 is none2)  # True (same object)
```

### 4. Dictionary Key Errors

```python
# BAD - raises KeyError if key doesn't exist
config = {'debug': True}
value = config['api_key']  # KeyError!

# GOOD - use get() with default
value = config.get('api_key', 'default_key')

# GOOD - check existence
if 'api_key' in config:
    value = config['api_key']

# BEST - use defaultdict for multiple accesses
from collections import defaultdict

counts = defaultdict(int)
for item in ['a', 'b', 'a', 'c']:
    counts[item] += 1  # No KeyError
```

## 🔐 Security Considerations

### 1. String Formatting - Avoid User Input in format()

```python
# DANGEROUS - can access internal attributes
template = "Hello, {user.password}"  # User provides this
user = User(name="John", password="secret123")
# result = template.format(user=user)  # Exposes password!

# SAFE - use % or f-strings with validated input
name = sanitize_input(user_input)
result = f"Hello, {name}"
```

### 2. Pickle - Never Unpickle Untrusted Data

```python
import pickle

# DANGEROUS - arbitrary code execution
# data = pickle.loads(untrusted_data)  # Can execute malicious code

# SAFE - use JSON for untrusted data
import json
data = json.loads(untrusted_json)  # Only deserializes data, no code
```

### 3. Dictionary - Prevent Hash Collision DoS

```python
# Before Python 3.4, hash collision attacks were possible
# Python 3.4+ uses SipHash (randomized hashing)

# Still, limit dictionary size from user input
MAX_DICT_SIZE = 1000

user_dict = {}
for key, value in user_data:
    if len(user_dict) >= MAX_DICT_SIZE:
        raise ValueError("Dictionary size limit exceeded")
    user_dict[key] = value
```

## 🚀 Performance Optimization Techniques

### 1. Use Appropriate Data Structures

```python
import time

# Slow: List membership testing O(n)
items_list = list(range(10000))
start = time.time()
result = 9999 in items_list
print(f"List: {time.time() - start:.6f}s")

# Fast: Set membership testing O(1) average
items_set = set(range(10000))
start = time.time()
result = 9999 in items_set
print(f"Set: {time.time() - start:.6f}s")

# Fast: Dict membership testing O(1) average
items_dict = {i: True for i in range(10000)}
start = time.time()
result = 9999 in items_dict
print(f"Dict: {time.time() - start:.6f}s")
```

### 2. String Concatenation

```python
# Slow - creates new string each iteration
result = ""
for i in range(10000):
    result += str(i)  # O(n²) complexity

# Fast - join is O(n)
result = "".join(str(i) for i in range(10000))

# Fast - use list and join
parts = []
for i in range(10000):
    parts.append(str(i))
result = "".join(parts)
```

### 3. List Pre-allocation

```python
# Slower - list grows dynamically
result = []
for i in range(1000000):
    result.append(i * 2)

# Faster - list comprehension (optimized in C)
result = [i * 2 for i in range(1000000)]

# Fastest - use map for simple operations
result = list(map(lambda x: x * 2, range(1000000)))
```

### 4. Use Slots for Memory Efficiency

```python
# Regular class - uses __dict__ for attributes
class RegularUser:
    def __init__(self, name, email):
        self.name = name
        self.email = email

# Slots class - 40-50% less memory
class OptimizedUser:
    __slots__ = ['name', 'email']

    def __init__(self, name, email):
        self.name = name
        self.email = email

import sys
regular = RegularUser("John", "john@example.com")
optimized = OptimizedUser("John", "john@example.com")

print(sys.getsizeof(regular.__dict__))  # ~120 bytes
# print(sys.getsizeof(optimized.__dict__))  # AttributeError - no __dict__
```

## 🧪 Code Examples

### Advanced Example: Type Coercion

```python
# Type conversion vs coercion
num_str = "42"
num_int = int(num_str)      # Explicit conversion
num_float = float(num_str)  # Explicit conversion

# Truthy/Falsy values
values = [0, 0.0, "", [], {}, set(), None, False]
for val in values:
    print(f"{repr(val):15} -> {bool(val)}")  # All False

# Custom truthiness
class AlwaysTrue:
    def __bool__(self):
        return True

obj = AlwaysTrue()
if obj:
    print("Always executed")
```

### Real-World: Data Validation with Types

```python
from typing import Any, Dict, List, Union
from datetime import datetime

def validate_user_data(data: Dict[str, Any]) -> Dict[str, Union[str, int, datetime]]:
    """Validate and coerce user registration data"""

    validated = {}

    # Required string field
    if not isinstance(data.get('username'), str):
        raise ValueError("Username must be a string")
    validated['username'] = data['username'].strip().lower()

    # Required email
    email = data.get('email', '').strip()
    if not isinstance(email, str) or '@' not in email:
        raise ValueError("Invalid email format")
    validated['email'] = email.lower()

    # Optional age (coerce to int)
    age = data.get('age')
    if age is not None:
        try:
            validated['age'] = int(age)
            if not 0 < validated['age'] < 150:
                raise ValueError("Invalid age range")
        except (ValueError, TypeError):
            raise ValueError("Age must be a valid number")

    # Timestamp
    validated['created_at'] = datetime.utcnow()

    return validated

# Usage
try:
    user_data = {
        'username': '  JohnDoe  ',
        'email': 'JOHN@EXAMPLE.COM',
        'age': '25'
    }
    validated = validate_user_data(user_data)
    print(validated)
except ValueError as e:
    print(f"Validation error: {e}")
```

## 🏗️ Real-World Use Cases

### 1. Caching with Immutable Keys

```python
from functools import lru_cache
from typing import Tuple

# Use tuples (immutable) for cache keys
@lru_cache(maxsize=128)
def expensive_computation(params: Tuple[int, str, bool]) -> float:
    """Cached function - params must be hashable"""
    # Expensive calculation
    return sum(ord(c) for c in params[1]) * params[0]

# Usage
result1 = expensive_computation((10, "hello", True))
result2 = expensive_computation((10, "hello", True))  # Cached, instant
```

### 2. Configuration with Frozen Data Classes

```python
from dataclasses import dataclass, field
from typing import List

@dataclass(frozen=True)  # Immutable
class DatabaseConfig:
    host: str
    port: int = 5432
    username: str = "postgres"
    password: str = field(repr=False)  # Hide in repr

    def __post_init__(self):
        if not 1 <= self.port <= 65535:
            raise ValueError("Invalid port number")

# Usage
db_config = DatabaseConfig(
    host="localhost",
    password="secret123"
)
# db_config.port = 3306  # FrozenInstanceError - can't modify
```

### 3. Type-Safe API Responses

```python
from typing import TypedDict, List, Optional
from datetime import datetime

class UserResponse(TypedDict):
    id: int
    username: str
    email: str
    created_at: str
    is_active: bool
    roles: List[str]

def get_user_response(user: 'User') -> UserResponse:
    """Type-safe API response builder"""
    return {
        'id': user.id,
        'username': user.username,
        'email': user.email,
        'created_at': user.created_at.isoformat(),
        'is_active': user.is_active,
        'roles': list(user.roles)  # Ensure it's a list
    }
```

## ❓ Frequently Asked Interview Questions

### Q1: What is the difference between mutable and immutable types?

**Answer:**

- **Immutable**: Cannot be changed after creation (int, float, str, tuple, frozenset)
- **Mutable**: Can be modified in-place (list, dict, set, bytearray)

```python
# Immutable - creates new object
s = "hello"
s_id = id(s)
s += " world"
print(id(s) != s_id)  # True - new object

# Mutable - modifies in-place
lst = [1, 2, 3]
lst_id = id(lst)
lst.append(4)
print(id(lst) == lst_id)  # True - same object

# Impact on function arguments
def modify_list(lst):
    lst.append(4)  # Modifies original

def modify_string(s):
    s += " world"  # Creates new local variable

my_list = [1, 2, 3]
my_string = "hello"

modify_list(my_list)
modify_string(my_string)

print(my_list)    # [1, 2, 3, 4] - modified
print(my_string)  # "hello" - unchanged
```

### Q2: Explain integer caching in Python

**Answer:**
CPython caches integers from -5 to 256 for performance. These objects are created once and reused.

```python
# Cached integers
a = 100
b = 100
print(a is b)  # True (same object)

# Non-cached integers
a = 1000
b = 1000
print(a is b)  # False (different objects)

# Why? Frequently used integers are cached to save memory
import sys
print(sys.getsizeof(1))     # 28 bytes
print(sys.getsizeof(1000))  # 28 bytes (same size, different objects)
```

### Q3: What are the time complexities of common operations?

**Answer:**

| Operation | List           | Set      | Dict     |
| --------- | -------------- | -------- | -------- |
| Access    | O(1)           | -        | O(1)     |
| Search    | O(n)           | O(1) avg | O(1) avg |
| Insert    | O(1) amortized | O(1) avg | O(1) avg |
| Delete    | O(n)           | O(1) avg | O(1) avg |
| Iteration | O(n)           | O(n)     | O(n)     |

```python
# List access is O(1)
lst = [1, 2, 3, 4, 5]
element = lst[2]  # Direct index access

# List search is O(n)
if 3 in lst:  # Scans through list
    pass

# Set search is O(1) average
s = {1, 2, 3, 4, 5}
if 3 in s:  # Hash table lookup
    pass
```

### Q4: Explain dictionary key requirements

**Answer:**
Dictionary keys must be **hashable** (immutable types with `__hash__()` and `__eq__()`).

```python
# Valid keys
valid = {
    "string": 1,
    42: 2,
    (1, 2): 3,
    frozenset([1, 2]): 4,
    True: 5,
    None: 6
}

# Invalid keys
# invalid = {
#     [1, 2]: "list",          # TypeError: unhashable
#     {1: 2}: "dict",          # TypeError: unhashable
#     {1, 2}: "set"            # TypeError: unhashable
# }

# Custom hashable class
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __hash__(self):
        return hash((self.x, self.y))

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

# Now Point can be a dict key
points = {Point(1, 2): "location"}
```

### Q5: What is the difference between `copy()` and `deepcopy()`?

**Answer:**

- **Shallow copy**: Copies the object but nested objects are shared references
- **Deep copy**: Recursively copies all nested objects

```python
import copy

# Shallow copy
original = [[1, 2], [3, 4]]
shallow = copy.copy(original)

shallow[0][0] = 999
print(original)  # [[999, 2], [3, 4]] - Modified!
print(shallow)   # [[999, 2], [3, 4]]

# Deep copy
original = [[1, 2], [3, 4]]
deep = copy.deepcopy(original)

deep[0][0] = 999
print(original)  # [[1, 2], [3, 4]] - Unchanged
print(deep)      # [[999, 2], [3, 4]]

# Performance consideration
# deepcopy is slower and uses more memory
```

### Q6: Explain string interning

**Answer:**
String interning is an optimization where Python caches identical strings to save memory.

```python
# Automatically interned (identifiers, constants)
a = "hello"
b = "hello"
print(a is b)  # True (same object)

# Not always interned (runtime strings)
a = "hello world!"
b = "hello world!"
print(a is b)  # May be False

# Force interning
import sys
a = sys.intern("hello world!")
b = sys.intern("hello world!")
print(a is b)  # True

# When to use:
# - Many duplicate strings (saves memory)
# - Fast string comparison with 'is'
# - Dictionary keys that repeat often
```

### Q7: Why use tuples over lists?

**Answer:**

1. **Memory efficient**: Tuples use less memory
2. **Faster**: Slightly faster iteration and access
3. **Hashable**: Can be dictionary keys
4. **Immutable**: Safe for constants and concurrent access
5. **Intent**: Signals that data shouldn't change

```python
import sys
import timeit

# Memory
lst = [1, 2, 3, 4, 5]
tpl = (1, 2, 3, 4, 5)
print(f"List: {sys.getsizeof(lst)} bytes")   # 104
print(f"Tuple: {sys.getsizeof(tpl)} bytes")  # 80

# Speed
list_time = timeit.timeit('[1, 2, 3, 4, 5]', number=1000000)
tuple_time = timeit.timeit('(1, 2, 3, 4, 5)', number=1000000)
print(f"List: {list_time:.4f}s")
print(f"Tuple: {tuple_time:.4f}s")  # Faster

# Use cases
# Tuple: coordinates, RGB colors, database records
coordinates = (40.7128, -74.0060)  # NYC lat/lon
rgb = (255, 0, 0)  # Red color

# List: dynamic collections
shopping_cart = ['apple', 'banana']
shopping_cart.append('orange')
```

### Q8: How do you handle None values safely?

**Answer:**

```python
# BAD - can raise AttributeError
def get_username(user):
    return user.name.upper()  # Crashes if user is None

# GOOD - explicit None check
def get_username(user):
    if user is None:
        return "Guest"
    return user.name.upper() if user.name else "Anonymous"

# BETTER - use Optional type hint
from typing import Optional

def get_username(user: Optional['User']) -> str:
    if user is None or user.name is None:
        return "Guest"
    return user.name.upper()

# BEST - use walrus operator and getattr
def get_username(user):
    if (name := getattr(user, 'name', None)):
        return name.upper()
    return "Guest"
```

## 📊 Performance Comparison

```python
import timeit
import sys

# Memory comparison
data = list(range(1000))
print(f"List: {sys.getsizeof(data)} bytes")
print(f"Tuple: {sys.getsizeof(tuple(data))} bytes")
print(f"Set: {sys.getsizeof(set(data))} bytes")
print(f"Dict: {sys.getsizeof({i: i for i in data})} bytes")

# Speed comparison for membership testing
setup = "data_list = list(range(10000)); data_set = set(range(10000))"

list_time = timeit.timeit('9999 in data_list', setup=setup, number=10000)
set_time = timeit.timeit('9999 in data_set', setup=setup, number=10000)

print(f"\nMembership test (10000 iterations):")
print(f"List: {list_time:.4f}s")
print(f"Set: {set_time:.4f}s")
print(f"Set is {list_time/set_time:.1f}x faster")
```

---

**Next**: [03_Control_Flow.md](03_Control_Flow.md)
