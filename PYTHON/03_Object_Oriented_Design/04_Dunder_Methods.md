# 📘 Dunder Methods (Magic Methods) in Python

## 📖 Concept Explanation

### What are Dunder Methods?

**Dunder methods** (Double Underscore methods, also called magic methods) are special methods with names starting and ending with double underscores (`__method__`). They allow you to define how objects behave with built-in Python operations.

### Why "Magic"?

They're called "magic" because they're automatically invoked by Python in response to certain operations, making objects behave like built-in types.

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __str__(self):
        """Called by str() and print()"""
        return f"Point({self.x}, {self.y})"

    def __add__(self, other):
        """Called by + operator"""
        return Point(self.x + other.x, self.y + other.y)

    def __eq__(self, other):
        """Called by == operator"""
        return self.x == other.x and self.y == other.y

p1 = Point(1, 2)
p2 = Point(3, 4)

print(p1)          # Calls __str__
p3 = p1 + p2       # Calls __add__
print(p3)          # Point(4, 6)
print(p1 == p2)    # Calls __eq__ -> False
```

---

## 🧠 Why It Matters

### Real-World Benefits

1. **Natural Syntax**: Make objects work like built-in types
2. **Operator Overloading**: Define custom behavior for operators
3. **Container Protocol**: Make objects iterable, indexable
4. **Context Managers**: Implement `with` statement
5. **Descriptors**: Control attribute access
6. **Pythonic Code**: Write idiomatic, readable Python

### Industry Usage

- **Django Models**: `__str__`, `__repr__` for debugging
- **SQLAlchemy**: Operator overloading for queries
- **NumPy/Pandas**: Array operations via dunder methods
- **Context Managers**: Database connections, file handling
- **Custom Collections**: List-like, dict-like objects

---

## ⚙️ Categories of Dunder Methods

### 1. Object Creation and Representation

```python
class Product:
    def __new__(cls, *args, **kwargs):
        """Called before __init__ to create instance"""
        print("__new__ called")
        instance = super().__new__(cls)
        return instance

    def __init__(self, name, price):
        """Initialize instance"""
        print("__init__ called")
        self.name = name
        self.price = price

    def __repr__(self):
        """Official string representation (for developers)"""
        return f"Product(name={self.name!r}, price={self.price!r})"

    def __str__(self):
        """Informal string representation (for users)"""
        return f"{self.name}: ${self.price}"

    def __format__(self, format_spec):
        """Custom formatting"""
        if format_spec == 'price':
            return f"${self.price:.2f}"
        elif format_spec == 'name':
            return self.name
        return str(self)

    def __bytes__(self):
        """Convert to bytes"""
        return f"{self.name}:{self.price}".encode('utf-8')

    def __hash__(self):
        """Make object hashable (for sets, dict keys)"""
        return hash((self.name, self.price))

# Usage
product = Product("Laptop", 999.99)
print(product)                    # Calls __str__
print(repr(product))              # Calls __repr__
print(f"{product:price}")         # Calls __format__
print(bytes(product))             # Calls __bytes__
print(hash(product))              # Calls __hash__

# Can use in sets and as dict keys
products = {product}              # Works because of __hash__
```

### 2. Comparison Operators

```python
from functools import total_ordering

@total_ordering  # Auto-generates missing comparison methods
class Version:
    def __init__(self, major, minor, patch):
        self.major = major
        self.minor = minor
        self.patch = patch

    def __eq__(self, other):
        """Equal to (==)"""
        if not isinstance(other, Version):
            return NotImplemented
        return (self.major, self.minor, self.patch) == \
               (other.major, other.minor, other.patch)

    def __lt__(self, other):
        """Less than (<)"""
        if not isinstance(other, Version):
            return NotImplemented
        return (self.major, self.minor, self.patch) < \
               (other.major, other.minor, other.patch)

    # With @total_ordering, these are auto-generated:
    # __le__ (<=), __gt__ (>), __ge__ (>=), __ne__ (!=)

    def __str__(self):
        return f"{self.major}.{self.minor}.{self.patch}"

v1 = Version(1, 2, 3)
v2 = Version(1, 3, 0)
v3 = Version(1, 2, 3)

print(v1 == v3)  # True
print(v1 < v2)   # True
print(v1 <= v2)  # True (auto-generated)
print(v1 > v2)   # False (auto-generated)
print(v1 != v2)  # True (auto-generated)
```

### 3. Arithmetic Operators

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):
        """Addition (+)"""
        return Vector(self.x + other.x, self.y + other.y)

    def __sub__(self, other):
        """Subtraction (-)"""
        return Vector(self.x - other.x, self.y - other.y)

    def __mul__(self, scalar):
        """Multiplication (*)"""
        if isinstance(scalar, (int, float)):
            return Vector(self.x * scalar, self.y * scalar)
        return NotImplemented

    def __rmul__(self, scalar):
        """Right multiplication (scalar * vector)"""
        return self.__mul__(scalar)

    def __truediv__(self, scalar):
        """True division (/)"""
        return Vector(self.x / scalar, self.y / scalar)

    def __floordiv__(self, scalar):
        """Floor division (//)"""
        return Vector(self.x // scalar, self.y // scalar)

    def __mod__(self, scalar):
        """Modulo (%)"""
        return Vector(self.x % scalar, self.y % scalar)

    def __pow__(self, power):
        """Power (**)"""
        return Vector(self.x ** power, self.y ** power)

    def __neg__(self):
        """Negation (-)"""
        return Vector(-self.x, -self.y)

    def __pos__(self):
        """Positive (+)"""
        return Vector(+self.x, +self.y)

    def __abs__(self):
        """Absolute value"""
        import math
        return math.sqrt(self.x**2 + self.y**2)

    def __str__(self):
        return f"Vector({self.x}, {self.y})"

# Usage
v1 = Vector(3, 4)
v2 = Vector(1, 2)

print(v1 + v2)       # Vector(4, 6)
print(v1 - v2)       # Vector(2, 2)
print(v1 * 2)        # Vector(6, 8)
print(3 * v1)        # Vector(9, 12) - uses __rmul__
print(v1 / 2)        # Vector(1.5, 2.0)
print(-v1)           # Vector(-3, -4)
print(abs(v1))       # 5.0
```

### 4. Container Methods

```python
class CustomList:
    def __init__(self, items=None):
        self._items = list(items) if items else []

    def __len__(self):
        """Length (len())"""
        return len(self._items)

    def __getitem__(self, index):
        """Get item (obj[index])"""
        return self._items[index]

    def __setitem__(self, index, value):
        """Set item (obj[index] = value)"""
        self._items[index] = value

    def __delitem__(self, index):
        """Delete item (del obj[index])"""
        del self._items[index]

    def __contains__(self, item):
        """Membership test (item in obj)"""
        return item in self._items

    def __iter__(self):
        """Iteration (for item in obj)"""
        return iter(self._items)

    def __reversed__(self):
        """Reverse iteration (reversed(obj))"""
        return reversed(self._items)

    def __str__(self):
        return f"CustomList({self._items})"

# Usage
lst = CustomList([1, 2, 3, 4, 5])

print(len(lst))           # 5
print(lst[0])             # 1
lst[0] = 10               # Modify
print(lst[0])             # 10
print(3 in lst)           # True

for item in lst:          # Iteration
    print(item, end=' ')  # 10 2 3 4 5

for item in reversed(lst):  # Reverse iteration
    print(item, end=' ')    # 5 4 3 2 10

del lst[0]                # Delete
print(lst)                # CustomList([2, 3, 4, 5])
```

### 5. Callable Objects

```python
class Multiplier:
    """Function-like object"""

    def __init__(self, factor):
        self.factor = factor

    def __call__(self, value):
        """Make object callable"""
        return value * self.factor

# Usage
double = Multiplier(2)
triple = Multiplier(3)

print(double(5))    # 10
print(triple(5))    # 15

# Can be used as function
numbers = [1, 2, 3, 4, 5]
doubled = list(map(double, numbers))
print(doubled)      # [2, 4, 6, 8, 10]
```

### 6. Context Managers

```python
class DatabaseConnection:
    """Context manager for database connection"""

    def __init__(self, db_name):
        self.db_name = db_name
        self.connection = None

    def __enter__(self):
        """Called when entering 'with' block"""
        print(f"Opening connection to {self.db_name}")
        self.connection = f"Connection to {self.db_name}"
        return self.connection

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Called when exiting 'with' block"""
        print(f"Closing connection to {self.db_name}")

        if exc_type is not None:
            print(f"Exception occurred: {exc_type.__name__}: {exc_val}")
            # Return True to suppress exception, False to propagate
            return False

        return True

# Usage
with DatabaseConnection("mydb") as conn:
    print(f"Using {conn}")
    # Automatic cleanup happens in __exit__

# Output:
# Opening connection to mydb
# Using Connection to mydb
# Closing connection to mydb
```

### 7. Attribute Access

```python
class Person:
    def __init__(self, name, age):
        self._name = name
        self._age = age

    def __getattr__(self, name):
        """Called when attribute not found"""
        print(f"__getattr__ called for: {name}")
        return f"No attribute '{name}'"

    def __getattribute__(self, name):
        """Called for ALL attribute access"""
        print(f"__getattribute__ called for: {name}")
        return super().__getattribute__(name)

    def __setattr__(self, name, value):
        """Called when setting attribute"""
        print(f"__setattr__ called: {name} = {value}")
        super().__setattr__(name, value)

    def __delattr__(self, name):
        """Called when deleting attribute"""
        print(f"__delattr__ called for: {name}")
        super().__delattr__(name)

# Usage
person = Person("Alice", 30)
# Output:
# __setattr__ called: _name = Alice
# __setattr__ called: _age = 30

print(person._name)
# Output:
# __getattribute__ called for: _name
# Alice

# Accessing non-existent attribute
print(person.email)
# Output:
# __getattribute__ called for: email
# __getattr__ called for: email
# No attribute 'email'
```

### 8. Descriptor Protocol

```python
class ValidatedAttribute:
    """Descriptor with validation"""

    def __init__(self, name, validator):
        self.name = name
        self.validator = validator

    def __get__(self, obj, objtype=None):
        """Called when getting attribute"""
        if obj is None:
            return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value):
        """Called when setting attribute"""
        if not self.validator(value):
            raise ValueError(f"Invalid value for {self.name}: {value}")
        obj.__dict__[self.name] = value

    def __delete__(self, obj):
        """Called when deleting attribute"""
        del obj.__dict__[self.name]

class Person:
    name = ValidatedAttribute('_name', lambda x: isinstance(x, str) and len(x) > 0)
    age = ValidatedAttribute('_age', lambda x: isinstance(x, int) and 0 <= x <= 150)

    def __init__(self, name, age):
        self.name = name
        self.age = age

# Usage
person = Person("Alice", 30)
print(person.name)  # Alice

# person.age = 200  # ValueError: Invalid value for _age: 200
# person.name = ""  # ValueError: Invalid value for _name:
```

---

## ✅ Best Practices

### 1. Always Implement **repr**

```python
# ✅ GOOD: Implement __repr__ for debugging
class User:
    def __init__(self, username, email):
        self.username = username
        self.email = email

    def __repr__(self):
        return f"User(username={self.username!r}, email={self.email!r})"

    def __str__(self):
        return f"{self.username} ({self.email})"

user = User("alice", "alice@example.com")
print(user)        # alice (alice@example.com)
print(repr(user))  # User(username='alice', email='alice@example.com')
print([user])      # [User(username='alice', email='alice@example.com')]
```

### 2. Return NotImplemented for Unsupported Operations

```python
class Number:
    def __init__(self, value):
        self.value = value

    def __add__(self, other):
        if isinstance(other, Number):
            return Number(self.value + other.value)
        elif isinstance(other, (int, float)):
            return Number(self.value + other)
        return NotImplemented  # Let Python try other.__radd__

    def __radd__(self, other):
        """Right-hand addition"""
        return self.__add__(other)

n1 = Number(5)
n2 = Number(10)

print((n1 + n2).value)  # 15
print((n1 + 5).value)   # 10
print((5 + n1).value)   # 10 (uses __radd__)
```

### 3. Implement **eq** and **hash** Together

```python
# ✅ GOOD: Both __eq__ and __hash__
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        if not isinstance(other, Point):
            return NotImplemented
        return self.x == other.x and self.y == other.y

    def __hash__(self):
        return hash((self.x, self.y))

# Can use in sets and as dict keys
p1 = Point(1, 2)
p2 = Point(1, 2)
p3 = Point(3, 4)

points = {p1, p2, p3}
print(len(points))  # 2 (p1 and p2 are equal)

point_dict = {p1: "first", p3: "second"}
print(point_dict[p2])  # "first" (p2 equals p1)
```

### 4. Use @total_ordering for Comparison Methods

```python
from functools import total_ordering

@total_ordering
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __eq__(self, other):
        if not isinstance(other, Person):
            return NotImplemented
        return self.age == other.age

    def __lt__(self, other):
        if not isinstance(other, Person):
            return NotImplemented
        return self.age < other.age

# All comparison operators work
alice = Person("Alice", 30)
bob = Person("Bob", 25)
charlie = Person("Charlie", 30)

print(alice > bob)   # True
print(alice >= bob)  # True
print(alice <= charlie)  # True
print(alice == charlie)  # True
```

---

## ❌ Common Mistakes

### 1. Forgetting to Return NotImplemented

```python
# ❌ BAD: Returning False instead of NotImplemented
class Number:
    def __init__(self, value):
        self.value = value

    def __eq__(self, other):
        if not isinstance(other, Number):
            return False  # BAD!
        return self.value == other.value

n = Number(5)
print(n == "5")  # False (should allow string to try __eq__)

# ✅ GOOD: Return NotImplemented
class Number:
    def __init__(self, value):
        self.value = value

    def __eq__(self, other):
        if not isinstance(other, Number):
            return NotImplemented  # GOOD!
        return self.value == other.value

n = Number(5)
print(n == "5")  # False (after Python tries string.__eq__)
```

### 2. Modifying **hash** Without **eq**

```python
# ❌ BAD: Only __hash__, no __eq__
class BadPoint:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __hash__(self):
        return hash((self.x, self.y))
    # Missing __eq__!

p1 = BadPoint(1, 2)
p2 = BadPoint(1, 2)

print(hash(p1) == hash(p2))  # True
print(p1 == p2)  # False! (uses default id comparison)

# Can cause issues with sets/dicts
points = {p1, p2}
print(len(points))  # 2 (should be 1!)

# ✅ GOOD: Both __eq__ and __hash__
class GoodPoint:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        if not isinstance(other, GoodPoint):
            return NotImplemented
        return self.x == other.x and self.y == other.y

    def __hash__(self):
        return hash((self.x, self.y))
```

### 3. Incorrect **getattribute** Usage

```python
# ❌ BAD: Infinite recursion
class Bad:
    def __getattribute__(self, name):
        # Causes infinite recursion!
        return self.__dict__[name]

# ✅ GOOD: Use super()
class Good:
    def __getattribute__(self, name):
        if name.startswith('_'):
            raise AttributeError(f"Cannot access private attribute {name}")
        return super().__getattribute__(name)
```

### 4. Not Handling Exceptions in **exit**

```python
# ❌ BAD: Ignoring exception info
class BadContext:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Cleaning up")
        # Always returns None (falsy), propagates all exceptions
        # But doesn't log them!

# ✅ GOOD: Proper exception handling
class GoodContext:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Cleaning up")

        if exc_type is not None:
            print(f"Exception: {exc_type.__name__}: {exc_val}")
            # Log, clean up, decide whether to suppress

        return False  # Propagate exception
```

---

## 🚀 Performance Optimization

### 1. Use **slots** with Dunder Methods

```python
class OptimizedPoint:
    __slots__ = ['x', 'y']

    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):
        return OptimizedPoint(self.x + other.x, self.y + other.y)

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

    def __hash__(self):
        return hash((self.x, self.y))

# Faster and uses less memory
import sys
p = OptimizedPoint(1, 2)
print(sys.getsizeof(p))  # Smaller size
```

### 2. Cache **hash** Results

```python
class CachedHash:
    def __init__(self, value):
        self.value = value
        self._hash = None

    def __hash__(self):
        if self._hash is None:
            self._hash = hash(self.value)
        return self._hash

    def __eq__(self, other):
        return isinstance(other, CachedHash) and self.value == other.value
```

### 3. Optimize **getitem** for Slicing

```python
class FastList:
    def __init__(self, items):
        self._items = list(items)

    def __getitem__(self, index):
        if isinstance(index, slice):
            # Optimized slice handling
            return FastList(self._items[index])
        return self._items[index]

    def __len__(self):
        return len(self._items)
```

---

## 🏗️ Real-World Use Cases

### 1. Django Model-like **str** and **repr**

```python
from datetime import datetime

class Model:
    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            setattr(self, key, value)

    def __repr__(self):
        class_name = self.__class__.__name__
        attrs = ', '.join(f"{k}={v!r}" for k, v in self.__dict__.items())
        return f"{class_name}({attrs})"

    def __str__(self):
        return f"{self.__class__.__name__} object"

class User(Model):
    def __str__(self):
        return f"{getattr(self, 'username', 'Unknown')}"

user = User(username="alice", email="alice@example.com", created_at=datetime.now())
print(user)        # alice
print(repr(user))  # User(username='alice', email='alice@example.com', ...)
print([user])      # [User(...)]
```

### 2. SQLAlchemy-like Query Builder

```python
class Query:
    def __init__(self, table=None):
        self.table = table
        self._filters = []
        self._order = None

    def filter(self, **kwargs):
        """Add filters using keyword arguments"""
        new_query = Query(self.table)
        new_query._filters = self._filters + [kwargs]
        return new_query

    def order_by(self, field):
        """Add ordering"""
        new_query = Query(self.table)
        new_query._filters = self._filters.copy()
        new_query._order = field
        return new_query

    def __iter__(self):
        """Make query iterable"""
        # Simulate fetching results
        results = [
            {'id': 1, 'name': 'Alice', 'age': 30},
            {'id': 2, 'name': 'Bob', 'age': 25},
        ]

        # Apply filters
        for filter_dict in self._filters:
            for key, value in filter_dict.items():
                results = [r for r in results if r.get(key) == value]

        # Apply ordering
        if self._order:
            results.sort(key=lambda x: x.get(self._order))

        return iter(results)

    def __repr__(self):
        return f"Query(table={self.table}, filters={self._filters}, order={self._order})"

# Usage (similar to SQLAlchemy)
users = Query('users')
active_users = users.filter(status='active').order_by('name')

for user in active_users:
    print(user)
```

### 3. NumPy-like Array with Operators

```python
class Array:
    def __init__(self, data):
        self.data = list(data)

    def __add__(self, other):
        if isinstance(other, Array):
            return Array([a + b for a, b in zip(self.data, other.data)])
        return Array([a + other for a in self.data])

    def __mul__(self, other):
        if isinstance(other, Array):
            return Array([a * b for a, b in zip(self.data, other.data)])
        return Array([a * other for a in self.data])

    def __getitem__(self, index):
        return self.data[index]

    def __len__(self):
        return len(self.data)

    def __repr__(self):
        return f"Array({self.data})"

# Usage (NumPy-like)
a = Array([1, 2, 3, 4])
b = Array([5, 6, 7, 8])

print(a + b)      # Array([6, 8, 10, 12])
print(a * 2)      # Array([2, 4, 6, 8])
print(a * b)      # Array([5, 12, 21, 32])
print(len(a))     # 4
print(a[0])       # 1
```

---

## ❓ Interview Questions

### Q1: What's the difference between **str** and **repr**?

**Answer:**

- **`__str__`**: User-friendly, readable representation (for end users)
- **`__repr__`**: Developer-friendly, unambiguous representation (for debugging)

**Rules:**

1. `__repr__` should be unambiguous and ideally eval-able
2. `__str__` should be readable
3. If only one is defined, implement `__repr__`
4. `__repr__` is fallback for `__str__`

```python
class User:
    def __init__(self, name, id):
        self.name = name
        self.id = id

    def __repr__(self):
        # For developers - can recreate object
        return f"User(name={self.name!r}, id={self.id!r})"

    def __str__(self):
        # For users - readable
        return f"{self.name} (ID: {self.id})"

user = User("Alice", 123)
print(str(user))   # Alice (ID: 123)
print(repr(user))  # User(name='Alice', id=123)
print([user])      # [User(name='Alice', id=123)] - uses __repr__
```

### Q2: When should you implement **hash**?

**Answer:**

Implement `__hash__` when you want objects to be:

- Used as dictionary keys
- Stored in sets
- Compared for uniqueness

**Rules:**

1. If you implement `__eq__`, also implement `__hash__`
2. Hash must be immutable (based on immutable attributes)
3. Equal objects must have equal hashes
4. Unequal objects can have equal hashes (collisions OK)

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        if not isinstance(other, Point):
            return NotImplemented
        return self.x == other.x and self.y == other.y

    def __hash__(self):
        # Based on immutable data
        return hash((self.x, self.y))

p1 = Point(1, 2)
p2 = Point(1, 2)

# Can use in sets
points = {p1, p2}
print(len(points))  # 1 (deduplicated)

# Can use as dict keys
data = {p1: "first"}
print(data[p2])  # "first"
```

### Q3: What is NotImplemented and when should you use it?

**Answer:**

`NotImplemented` is a special constant that tells Python to try the reflected operation on the other operand.

```python
class Number:
    def __init__(self, value):
        self.value = value

    def __add__(self, other):
        if isinstance(other, Number):
            return Number(self.value + other.value)
        return NotImplemented  # Let Python try other.__radd__

    def __radd__(self, other):
        return self.__add__(other)

n = Number(5)
result = n + 10  # __add__ returns NotImplemented
# Python tries: 10.__radd__(n), which doesn't exist
# Falls back to TypeError

result = 10 + n  # Tries 10.__add__(n), fails
# Then tries n.__radd__(10), succeeds!
print(result.value)  # 15
```

**When to use:**

- Return `NotImplemented` when operation not supported for given type
- Don't return `False` or raise `TypeError`
- Let Python handle reflected operations

### Q4: How do context managers work internally?

**Answer:**

Context managers implement `__enter__` and `__exit__` for resource management with the `with` statement.

```python
class Transaction:
    def __init__(self, db):
        self.db = db

    def __enter__(self):
        """Called when entering 'with' block"""
        print("Beginning transaction")
        self.db.begin()
        return self  # Returned to 'as' variable

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Called when exiting 'with' block"""
        if exc_type is None:
            print("Committing transaction")
            self.db.commit()
        else:
            print(f"Rolling back: {exc_val}")
            self.db.rollback()

        # Return True to suppress exception, False to propagate
        return False

# Usage
with Transaction(db) as txn:
    # __enter__ called here
    db.execute("INSERT ...")
    # __exit__ called here (even if exception occurs)
```

**Benefits:**

- Guaranteed cleanup
- Exception handling
- Cleaner code

### Q5: What's the difference between **getattr** and **getattribute**?

**Answer:**

- **`__getattribute__`**: Called for **ALL** attribute access
- **`__getattr__`**: Called only when attribute **not found**

```python
class Demo:
    def __init__(self):
        self.x = 10

    def __getattribute__(self, name):
        """Called for ALL attribute access"""
        print(f"__getattribute__: {name}")
        return super().__getattribute__(name)

    def __getattr__(self, name):
        """Called only if attribute not found"""
        print(f"__getattr__: {name}")
        return f"No attribute '{name}'"

obj = Demo()
print(obj.x)     # __getattribute__: x -> 10
print(obj.y)     # __getattribute__: y -> __getattr__: y -> "No attribute 'y'"
```

**Use cases:**

- `__getattribute__`: Intercept all access (logging, proxying)
- `__getattr__`: Provide default values, lazy loading

---

## 📚 Summary

### Key Takeaways

1. **Dunder methods** make objects behave like built-in types
2. Always implement **`__repr__`** for debugging
3. Implement **`__eq__` and `__hash__`** together
4. Return **`NotImplemented`** for unsupported operations
5. Use **`@total_ordering`** for comparison methods
6. **Context managers** use `__enter__` and `__exit__`
7. **`__getattribute__`** for all access, **`__getattr__`** for missing
8. **Operators** can be overloaded with dunder methods
9. **Container protocol**: `__len__`, `__getitem__`, `__iter__`
10. **Callable objects** use `__call__`

### Common Dunder Methods

| Category         | Methods                                          | Purpose               |
| ---------------- | ------------------------------------------------ | --------------------- |
| Representation   | `__repr__`, `__str__`, `__format__`              | String representation |
| Comparison       | `__eq__`, `__lt__`, `__le__`, `__gt__`, `__ge__` | Comparison operators  |
| Arithmetic       | `__add__`, `__sub__`, `__mul__`, `__div__`       | Math operators        |
| Container        | `__len__`, `__getitem__`, `__iter__`             | Sequence operations   |
| Callable         | `__call__`                                       | Make object callable  |
| Context Manager  | `__enter__`, `__exit__`                          | Resource management   |
| Attribute Access | `__getattr__`, `__setattr__`, `__delattr__`      | Attribute control     |
| Hashing          | `__hash__`                                       | Sets, dict keys       |

### Best Practices Checklist

✅ Always implement `__repr__` for debugging
✅ Implement `__eq__` and `__hash__` together
✅ Return `NotImplemented` for unsupported types
✅ Use `@total_ordering` for comparisons
✅ Handle exceptions properly in `__exit__`
✅ Use `super()` in `__getattribute__`
✅ Cache `__hash__` if expensive
✅ Use `__slots__` for memory optimization
✅ Document dunder method behavior
✅ Test with built-in functions (`len`, `str`, etc.)

---

**Next**: [05_Abstract_Base_Classes.md](05_Abstract_Base_Classes.md)
