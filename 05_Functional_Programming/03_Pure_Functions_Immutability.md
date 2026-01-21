# Pure Functions & Immutability

## 📖 Concept Overview

**Pure Functions** are functions that:

1. Always return the same output for the same input (deterministic)
2. Have no side effects (don't modify external state)

**Immutability** means data cannot be changed after creation. Instead of modifying existing data, create new data with the desired changes.

## 🧠 Why It Matters

### Benefits of Pure Functions

- **Predictable**: Easy to reason about
- **Testable**: No setup/teardown needed
- **Cacheable**: Results can be memoized
- **Parallelizable**: Safe for concurrent execution
- **Debuggable**: No hidden dependencies

### Benefits of Immutability

- **Thread-safe**: No race conditions
- **Easier reasoning**: Data doesn't change unexpectedly
- **Time-travel debugging**: Can replay state
- **Undo/redo**: Keep history of changes
- **Caching**: Safe to share references

## ⚙️ Internal Working

### Python's Immutable Types

- **Immutable**: int, float, str, tuple, frozenset, bytes
- **Mutable**: list, dict, set, bytearray

```python
# Immutable string
s = "hello"
s_id = id(s)
s = s + " world"  # Creates new string
print(id(s) != s_id)  # True - different object

# Mutable list
lst = [1, 2, 3]
lst_id = id(lst)
lst.append(4)  # Modifies in place
print(id(lst) == lst_id)  # True - same object
```

## 🧪 Pure vs Impure Functions

### Pure Functions

```python
# ✅ Pure: Same input -> Same output, No side effects
def add(a: int, b: int) -> int:
    return a + b

def square(x: int) -> int:
    return x ** 2

def get_full_name(first: str, last: str) -> str:
    return f"{first} {last}"

# Always deterministic
print(add(2, 3))         # Always 5
print(square(4))         # Always 16
print(get_full_name("John", "Doe"))  # Always "John Doe"
```

### Impure Functions

```python
import time
from datetime import datetime

# ❌ Impure: Side effect (modifies global state)
counter = 0

def increment():
    global counter
    counter += 1  # Side effect!
    return counter

# ❌ Impure: Non-deterministic (depends on time)
def get_current_timestamp():
    return datetime.now()  # Different each call

# ❌ Impure: Side effect (modifies input)
def append_item(lst: list, item):
    lst.append(item)  # Mutates argument!
    return lst

# ❌ Impure: Side effect (I/O)
def save_to_file(data: str):
    with open("data.txt", "w") as f:
        f.write(data)  # Side effect!
```

### Converting Impure to Pure

```python
# ❌ Impure version
def add_item_impure(lst: list, item):
    lst.append(item)
    return lst

# ✅ Pure version
def add_item_pure(lst: list, item):
    return lst + [item]  # Return new list

# ❌ Impure: Modifies dict
def update_user_impure(user: dict, age: int):
    user['age'] = age
    return user

# ✅ Pure: Returns new dict
def update_user_pure(user: dict, age: int):
    return {**user, 'age': age}
```

## 🔒 Immutability Patterns

### 1. Immutable Data Structures

```python
from typing import Tuple, FrozenSet

# Use tuples instead of lists
coordinates: Tuple[int, int] = (10, 20)
# coordinates[0] = 15  # TypeError

# Use frozenset instead of set
tags: FrozenSet[str] = frozenset(['python', 'functional', 'pure'])
# tags.add('new')  # AttributeError

# Immutable string operations
original = "hello"
uppercase = original.upper()  # Returns new string
print(original)  # "hello" - unchanged
print(uppercase) # "HELLO"
```

### 2. Copying for "Modification"

```python
from copy import copy, deepcopy

# Shallow copy
original_list = [1, 2, 3]
new_list = original_list.copy()  # or list(original_list) or original_list[:]
new_list.append(4)

print(original_list)  # [1, 2, 3]
print(new_list)       # [1, 2, 3, 4]

# Deep copy for nested structures
original = {'user': {'name': 'Alice', 'age': 25}, 'active': True}
modified = deepcopy(original)
modified['user']['age'] = 26

print(original['user']['age'])  # 25 - unchanged
print(modified['user']['age'])  # 26
```

### 3. Functional Updates

```python
# List operations (return new lists)
numbers = [1, 2, 3, 4, 5]

# Add element
with_six = numbers + [6]

# Remove element
without_three = [x for x in numbers if x != 3]

# Update element
updated = [x * 2 if x == 3 else x for x in numbers]

print(numbers)      # [1, 2, 3, 4, 5] - unchanged
print(updated)      # [1, 2, 6, 4, 5]

# Dict operations (return new dicts)
user = {'name': 'Alice', 'age': 25, 'city': 'NYC'}

# Add/update key
with_email = {**user, 'email': 'alice@example.com'}

# Remove key
without_city = {k: v for k, v in user.items() if k != 'city'}

# Update value
older = {**user, 'age': user['age'] + 1}

print(user)         # Original unchanged
print(older)        # {'name': 'Alice', 'age': 26, 'city': 'NYC'}
```

## 🏗️ Real-World Example: Immutable User State

```python
from dataclasses import dataclass, replace
from typing import List, Optional
from datetime import datetime

@dataclass(frozen=True)  # Makes it immutable
class User:
    """Immutable user data"""
    id: int
    username: str
    email: str
    age: int
    is_active: bool = True
    tags: tuple = ()  # Use tuple, not list
    created_at: datetime = None

    def __post_init__(self):
        # Use object.__setattr__ for frozen dataclass
        if self.created_at is None:
            object.__setattr__(self, 'created_at', datetime.now())

    def with_email(self, new_email: str) -> 'User':
        """Return new User with updated email"""
        return replace(self, email=new_email)

    def with_age(self, new_age: int) -> 'User':
        """Return new User with updated age"""
        if new_age < 0:
            raise ValueError("Age cannot be negative")
        return replace(self, age=new_age)

    def deactivate(self) -> 'User':
        """Return new User marked as inactive"""
        return replace(self, is_active=False)

    def add_tag(self, tag: str) -> 'User':
        """Return new User with additional tag"""
        return replace(self, tags=self.tags + (tag,))

# Usage
user = User(
    id=1,
    username="alice",
    email="alice@example.com",
    age=25,
    tags=("developer", "python")
)

# All operations return new instances
user_v2 = user.with_email("alice.smith@example.com")
user_v3 = user_v2.with_age(26)
user_v4 = user_v3.add_tag("senior")

print(user.email)       # alice@example.com (unchanged)
print(user_v4.email)    # alice.smith@example.com
print(user_v4.age)      # 26
print(user_v4.tags)     # ('developer', 'python', 'senior')

# user.age = 30  # FrozenInstanceError - can't modify!
```

## 🏢 Real-World Example: Shopping Cart

```python
from typing import List, Tuple
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class CartItem:
    """Immutable cart item"""
    product_id: int
    name: str
    price: Decimal
    quantity: int

    def with_quantity(self, new_quantity: int) -> 'CartItem':
        """Return new item with updated quantity"""
        return CartItem(
            self.product_id,
            self.name,
            self.price,
            new_quantity
        )

    @property
    def subtotal(self) -> Decimal:
        return self.price * self.quantity

@dataclass(frozen=True)
class ShoppingCart:
    """Immutable shopping cart"""
    items: Tuple[CartItem, ...] = ()

    @property
    def total(self) -> Decimal:
        """Calculate total price"""
        return sum(item.subtotal for item in self.items)

    @property
    def item_count(self) -> int:
        """Total number of items"""
        return sum(item.quantity for item in self.items)

    def add_item(self, item: CartItem) -> 'ShoppingCart':
        """Return new cart with item added"""
        # Check if item already exists
        for i, existing in enumerate(self.items):
            if existing.product_id == item.product_id:
                # Update quantity
                updated_item = existing.with_quantity(
                    existing.quantity + item.quantity
                )
                new_items = (
                    self.items[:i] +
                    (updated_item,) +
                    self.items[i+1:]
                )
                return ShoppingCart(new_items)

        # Add new item
        return ShoppingCart(self.items + (item,))

    def remove_item(self, product_id: int) -> 'ShoppingCart':
        """Return new cart with item removed"""
        new_items = tuple(
            item for item in self.items
            if item.product_id != product_id
        )
        return ShoppingCart(new_items)

    def update_quantity(self, product_id: int, quantity: int) -> 'ShoppingCart':
        """Return new cart with updated quantity"""
        new_items = tuple(
            item.with_quantity(quantity) if item.product_id == product_id else item
            for item in self.items
        )
        return ShoppingCart(new_items)

    def clear(self) -> 'ShoppingCart':
        """Return empty cart"""
        return ShoppingCart()

# Usage - Each operation returns new cart
cart = ShoppingCart()

# Add items
cart = cart.add_item(CartItem(1, "Laptop", Decimal("999.99"), 1))
cart = cart.add_item(CartItem(2, "Mouse", Decimal("29.99"), 2))
cart = cart.add_item(CartItem(1, "Laptop", Decimal("999.99"), 1))  # Increases quantity

print(f"Items: {cart.item_count}")  # 4
print(f"Total: ${cart.total}")      # $2059.96

# Update quantity
cart = cart.update_quantity(2, 3)
print(f"Total: ${cart.total}")      # $2089.95

# Remove item
cart = cart.remove_item(1)
print(f"Total: ${cart.total}")      # $89.97

# Original operations don't affect previous state
# (useful for undo/redo, history tracking)
```

## 🎯 Memoization with Pure Functions

```python
from functools import lru_cache
from time import time

# Pure function - safe to cache
@lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    """Calculate fibonacci number (cached)"""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# Benchmark
start = time()
result = fibonacci(35)
duration1 = time() - start

start = time()
result = fibonacci(35)  # Cached!
duration2 = time() - start

print(f"First call: {duration1:.4f}s")
print(f"Cached call: {duration2:.4f}s")  # Much faster!

# Complex example: Expensive computation
@lru_cache(maxsize=1000)
def calculate_discount(
    price: Decimal,
    customer_tier: str,
    is_holiday: bool
) -> Decimal:
    """Pure function - safe to cache"""
    base_discount = Decimal('0.1')

    tier_discounts = {
        'bronze': Decimal('0.05'),
        'silver': Decimal('0.10'),
        'gold': Decimal('0.15'),
    }

    discount = base_discount + tier_discounts.get(customer_tier, Decimal('0'))

    if is_holiday:
        discount += Decimal('0.05')

    return price * (1 - discount)

# These calls will be cached
price1 = calculate_discount(Decimal('100'), 'gold', True)
price2 = calculate_discount(Decimal('100'), 'gold', True)  # Cached!
```

## ✅ Best Practices

### 1. Use Immutable Data Structures

```python
# ❌ Bad: Mutable default argument
def add_to_list(item, lst=[]):  # Dangerous!
    lst.append(item)
    return lst

# ✅ Good: Immutable approach
def add_to_list(item, lst=None):
    if lst is None:
        lst = []
    return lst + [item]  # Return new list
```

### 2. Return New Objects, Don't Modify

```python
from typing import List

# ❌ Bad: Modifies input
def filter_active_users(users: List[dict]) -> List[dict]:
    for user in users:
        if not user.get('active'):
            users.remove(user)  # Modifies input!
    return users

# ✅ Good: Returns new list
def filter_active_users(users: List[dict]) -> List[dict]:
    return [user for user in users if user.get('active')]
```

### 3. Use Frozen Dataclasses

```python
from dataclasses import dataclass

# ✅ Good: Immutable by default
@dataclass(frozen=True)
class Point:
    x: int
    y: int

    def move(self, dx: int, dy: int) -> 'Point':
        return Point(self.x + dx, self.y + dy)
```

### 4. Avoid Global State

```python
# ❌ Bad: Depends on global state
config = {'timeout': 30}

def make_request(url):
    timeout = config['timeout']  # Depends on global!
    # ... make request

# ✅ Good: Pass dependencies explicitly
def make_request(url, timeout):
    # ... make request
```

## ❌ Common Mistakes

### 1. Hidden Mutations

```python
# ❌ Bad: Shallow copy doesn't prevent nested mutation
def update_user_age(user: dict, age: int) -> dict:
    new_user = user.copy()  # Shallow copy
    new_user['profile']['age'] = age  # Mutates nested dict!
    return new_user

# ✅ Good: Deep copy
from copy import deepcopy

def update_user_age(user: dict, age: int) -> dict:
    new_user = deepcopy(user)
    new_user['profile']['age'] = age
    return new_user
```

### 2. Mutable Default Arguments

```python
# ❌ Bad: Shared mutable default
def add_user(name: str, users=[]):
    users.append(name)
    return users

add_user("Alice")  # ['Alice']
add_user("Bob")    # ['Alice', 'Bob'] - Surprise!

# ✅ Good: None as default
def add_user(name: str, users=None):
    if users is None:
        users = []
    return users + [name]
```

### 3. Side Effects in Comprehensions

```python
# ❌ Bad: Side effect in comprehension
results = []
[results.append(x * 2) for x in numbers]  # Don't do this!

# ✅ Good: Pure comprehension
results = [x * 2 for x in numbers]
```

## 🚀 Performance Considerations

### When Immutability Hurts Performance

```python
# ❌ Inefficient: Creating many intermediate objects
def process_large_list(items):
    result = ()
    for item in items:
        result = result + (item * 2,)  # Creates new tuple each time!
    return result

# ✅ Better: Use mutable structure, then convert
def process_large_list(items):
    result = []
    for item in items:
        result.append(item * 2)
    return tuple(result)  # Convert to immutable at end
```

### Persistent Data Structures

```python
# For large-scale immutable operations, consider libraries:
# - pyrsistent: Efficient immutable data structures
# - immutables: High-performance immutable mappings

from pyrsistent import pvector, pmap

# Efficient immutable vector
v = pvector([1, 2, 3, 4, 5])
v2 = v.append(6)  # O(log n) instead of O(n)
v3 = v.set(0, 10)  # Efficient update

# Efficient immutable map
m = pmap({'a': 1, 'b': 2})
m2 = m.set('c', 3)  # Structural sharing
```

## ❓ Interview Questions

### Q1: What makes a function pure?

**Answer**: A pure function:

1. **Deterministic**: Same input always produces same output
2. **No side effects**: Doesn't modify external state, I/O, etc.
3. **No dependencies on external state**: Only depends on parameters

Benefits: Testable, cacheable, parallelizable, predictable.

### Q2: Why is immutability important in concurrent programming?

**Answer**: Immutable data is inherently thread-safe:

- No race conditions (can't be modified)
- No need for locks
- Safe to share between threads
- Eliminates entire class of bugs

```python
# Immutable data is safe to share
user = User(name="Alice", age=25)
# Multiple threads can read user safely
```

### Q3: What's the trade-off of immutability?

**Answer**:
**Pros**:

- Thread-safe, predictable, easier reasoning
- Enables time-travel debugging, undo/redo
- Safe caching

**Cons**:

- Memory overhead (creating new objects)
- Performance cost for large structures
- Less intuitive for some developers

**Solution**: Use persistent data structures (structural sharing) or mutable locally, immutable at boundaries.

### Q4: How do you handle I/O in functional programming?

**Answer**: Separate pure logic from side effects:

```python
# ✅ Pure: Business logic
def calculate_total(items):
    return sum(item['price'] for item in items)

# Impure: I/O at boundaries
def save_order(items):
    total = calculate_total(items)  # Pure
    # I/O side effect
    db.save({'items': items, 'total': total})
```

Push side effects to the edges, keep core logic pure.

### Q5: When should you NOT use immutability?

**Answer**:

- Performance-critical code with many updates
- Large data structures updated frequently
- Streams/buffers that must be mutable
- When working with mutable-by-design APIs

Balance: Use immutability at boundaries/interfaces, mutable internally if needed.

## 📚 Summary

### Key Takeaways

1. **Pure functions** are deterministic with no side effects
2. **Immutability** prevents unexpected state changes
3. **Thread-safety** comes naturally with immutable data
4. **Use frozen dataclasses** for immutable objects
5. **Return new objects** instead of modifying existing ones
6. **Memoization** works perfectly with pure functions
7. **Balance** immutability benefits against performance needs

### When to Use Pure Functions

✅ **Use when**:

- Business logic and calculations
- Data transformations
- Anywhere testability matters
- Concurrent/parallel code
- Caching is beneficial

❌ **Avoid when**:

- I/O operations required
- Performance-critical mutations
- Working with mutable APIs
- State management requires mutability

### Immutability Checklist

- ✅ Use `frozen=True` in dataclasses
- ✅ Use tuples instead of lists when possible
- ✅ Return new objects, don't modify parameters
- ✅ Avoid mutable default arguments
- ✅ Use `deepcopy` for nested structures
- ✅ Consider persistent data structures for performance
- ✅ Keep I/O and side effects at boundaries
- ✅ Cache pure function results with `@lru_cache`
