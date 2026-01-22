# 📘 Classes and Objects in Python

## 📖 Concept Explanation

### What are Classes and Objects?

**Class**: A blueprint or template that defines the structure and behavior of objects. It encapsulates data (attributes) and functions (methods) that operate on that data.

**Object**: An instance of a class. When you create an object, you're creating a specific realization of the class with actual values.

### Basic Class Definition

```python
class User:
    """A simple user class"""

    def __init__(self, username, email):
        self.username = username
        self.email = email

    def greet(self):
        return f"Hello, I'm {self.username}"

# Creating objects (instances)
user1 = User("alice", "alice@example.com")
user2 = User("bob", "bob@example.com")

print(user1.greet())  # "Hello, I'm alice"
print(user2.greet())  # "Hello, I'm bob"
```

### Instance vs Class vs Static Variables

```python
class Product:
    # Class variable (shared by all instances)
    store_name = "TechStore"
    product_count = 0

    def __init__(self, name, price):
        # Instance variables (unique to each instance)
        self.name = name
        self.price = price
        Product.product_count += 1

    def display(self):
        """Instance method - has access to self"""
        return f"{self.name} costs ${self.price}"

    @classmethod
    def get_product_count(cls):
        """Class method - has access to cls"""
        return cls.product_count

    @staticmethod
    def is_valid_price(price):
        """Static method - no access to cls or self"""
        return price > 0

# Usage
laptop = Product("Laptop", 1200)
phone = Product("Phone", 800)

print(Product.store_name)           # "TechStore"
print(Product.get_product_count())  # 2
print(Product.is_valid_price(100))  # True
print(laptop.display())             # "Laptop costs $1200"
```

---

## 🧠 Why It Matters

### Real-World Benefits

1. **Code Organization**: Group related data and functions together
2. **Reusability**: Create multiple instances from the same blueprint
3. **Maintainability**: Changes to class affect all instances
4. **Encapsulation**: Hide implementation details, expose clean interface
5. **Modeling**: Represent real-world entities naturally

### Industry Usage

- **Django Models**: Each model is a class representing a database table
- **FastAPI Schemas**: Pydantic models are classes for validation
- **Service Classes**: Encapsulate business logic
- **Repository Pattern**: Classes for data access layer
- **Factory Pattern**: Classes for object creation

---

## ⚙️ Internal Working

### Object Creation Process

```python
class User:
    def __init__(self, name):
        self.name = name
        print(f"3. __init__ called for {name}")

    def __new__(cls, name):
        print(f"1. __new__ called for {name}")
        instance = super().__new__(cls)
        print(f"2. Instance created: {id(instance)}")
        return instance

# Creating object triggers __new__ then __init__
user = User("Alice")
# Output:
# 1. __new__ called for Alice
# 2. Instance created: 140234567890123
# 3. __init__ called for Alice
```

### Memory Layout

```python
import sys

class SimpleClass:
    def __init__(self, value):
        self.value = value

class ComplexClass:
    def __init__(self, a, b, c, d, e):
        self.a = a
        self.b = b
        self.c = c
        self.d = d
        self.e = e

simple = SimpleClass(42)
complex_obj = ComplexClass(1, 2, 3, 4, 5)

print(f"Simple object size: {sys.getsizeof(simple)} bytes")
print(f"Complex object size: {sys.getsizeof(complex_obj)} bytes")
print(f"Object dict size: {sys.getsizeof(simple.__dict__)} bytes")
```

### Instance Dictionary

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

person = Person("Alice", 30)

# Every instance has a __dict__ containing its attributes
print(person.__dict__)  # {'name': 'Alice', 'age': 30}

# You can manipulate it directly (not recommended)
person.__dict__['city'] = 'NYC'
print(person.city)  # 'NYC'

# Check if attribute exists
print(hasattr(person, 'name'))   # True
print(hasattr(person, 'email'))  # False

# Get/Set attributes dynamically
setattr(person, 'email', 'alice@example.com')
print(getattr(person, 'email'))  # 'alice@example.com'
print(getattr(person, 'phone', 'N/A'))  # 'N/A' (default)
```

---

## ✅ Best Practices

### 1. Single Responsibility Principle

```python
# ❌ BAD: Class doing too many things
class User:
    def __init__(self, username, email):
        self.username = username
        self.email = email

    def save_to_database(self):
        # Database logic
        pass

    def send_welcome_email(self):
        # Email logic
        pass

    def generate_pdf_report(self):
        # PDF logic
        pass

# ✅ GOOD: Separate concerns
class User:
    def __init__(self, username, email):
        self.username = username
        self.email = email

    def to_dict(self):
        return {'username': self.username, 'email': self.email}

class UserRepository:
    def save(self, user):
        # Database logic
        pass

class EmailService:
    def send_welcome_email(self, user):
        # Email logic
        pass

class ReportGenerator:
    def generate_user_report(self, user):
        # PDF logic
        pass
```

### 2. Use Properties for Validation

```python
class BankAccount:
    def __init__(self, account_number, initial_balance=0):
        self.account_number = account_number
        self._balance = initial_balance  # Protected attribute

    @property
    def balance(self):
        """Get balance"""
        return self._balance

    @balance.setter
    def balance(self, amount):
        """Set balance with validation"""
        if amount < 0:
            raise ValueError("Balance cannot be negative")
        self._balance = amount

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self._balance += amount

    def withdraw(self, amount):
        if amount <= 0:
            raise ValueError("Withdrawal must be positive")
        if amount > self._balance:
            raise ValueError("Insufficient funds")
        self._balance -= amount

# Usage
account = BankAccount("ACC123", 1000)
print(account.balance)  # 1000

account.deposit(500)
print(account.balance)  # 1500

# account.balance = -100  # Raises ValueError
```

### 3. Use **slots** for Memory Optimization

```python
# Without __slots__: Each instance has __dict__
class RegularUser:
    def __init__(self, name, age):
        self.name = name
        self.age = age

# With __slots__: No __dict__, fixed attributes
class OptimizedUser:
    __slots__ = ['name', 'age']

    def __init__(self, name, age):
        self.name = name
        self.age = age

# Memory comparison
import sys

regular = RegularUser("Alice", 30)
optimized = OptimizedUser("Alice", 30)

print(f"Regular: {sys.getsizeof(regular)} bytes")
print(f"Optimized: {sys.getsizeof(optimized)} bytes")
print(f"Savings: {sys.getsizeof(regular) - sys.getsizeof(optimized)} bytes")

# regular has __dict__, optimized doesn't
print(hasattr(regular, '__dict__'))    # True
print(hasattr(optimized, '__dict__'))  # False

# Can't add new attributes to __slots__ class
# optimized.email = "test@example.com"  # Raises AttributeError
```

### 4. Class Documentation

```python
class UserService:
    """
    Service for managing user operations.

    This class provides methods for user authentication,
    registration, and profile management.

    Attributes:
        db_connection: Database connection instance
        cache_client: Redis cache client

    Example:
        >>> service = UserService(db, cache)
        >>> user = service.register("alice", "alice@example.com")
        >>> service.authenticate("alice", "password123")
    """

    def __init__(self, db_connection, cache_client):
        """
        Initialize UserService.

        Args:
            db_connection: Active database connection
            cache_client: Redis client for caching
        """
        self.db = db_connection
        self.cache = cache_client

    def register(self, username: str, email: str) -> 'User':
        """
        Register a new user.

        Args:
            username: Unique username (3-50 characters)
            email: Valid email address

        Returns:
            User: Newly created user instance

        Raises:
            ValueError: If username/email is invalid
            DuplicateUserError: If user already exists
        """
        pass
```

---

## ❌ Common Mistakes

### 1. Mutable Default Arguments

```python
# ❌ BAD: Mutable default argument
class ShoppingCart:
    def __init__(self, items=[]):  # DANGER!
        self.items = items

cart1 = ShoppingCart()
cart1.items.append("Apple")

cart2 = ShoppingCart()  # Shares same list!
print(cart2.items)  # ['Apple'] - Unexpected!

# ✅ GOOD: Use None and create new list
class ShoppingCart:
    def __init__(self, items=None):
        self.items = items if items is not None else []

cart1 = ShoppingCart()
cart1.items.append("Apple")

cart2 = ShoppingCart()
print(cart2.items)  # [] - Correct!
```

### 2. Not Using **repr** and **str**

```python
# ❌ BAD: No string representation
class User:
    def __init__(self, username, email):
        self.username = username
        self.email = email

user = User("alice", "alice@example.com")
print(user)  # <__main__.User object at 0x...> - Not helpful!

# ✅ GOOD: Implement __repr__ and __str__
class User:
    def __init__(self, username, email):
        self.username = username
        self.email = email

    def __repr__(self):
        """For developers - unambiguous"""
        return f"User(username={self.username!r}, email={self.email!r})"

    def __str__(self):
        """For end users - readable"""
        return f"{self.username} ({self.email})"

user = User("alice", "alice@example.com")
print(user)          # alice (alice@example.com)
print(repr(user))    # User(username='alice', email='alice@example.com')
print([user])        # [User(username='alice', email='alice@example.com')]
```

### 3. Modifying Class Variables

```python
# ❌ BAD: Unintended class variable modification
class Counter:
    count = 0  # Class variable

    def increment(self):
        self.count += 1  # Creates instance variable!

c1 = Counter()
c2 = Counter()

c1.increment()
print(c1.count)  # 1 (instance variable)
print(c2.count)  # 0 (class variable)
print(Counter.count)  # 0 (class variable unchanged)

# ✅ GOOD: Explicitly modify class variable
class Counter:
    count = 0

    def increment(self):
        Counter.count += 1  # Modifies class variable

c1 = Counter()
c2 = Counter()

c1.increment()
print(c1.count)  # 1
print(c2.count)  # 1
print(Counter.count)  # 1
```

### 4. Not Handling Equality

```python
# ❌ BAD: Default equality compares identity
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p1 = Point(1, 2)
p2 = Point(1, 2)
print(p1 == p2)  # False - Different objects!

# ✅ GOOD: Implement __eq__ and __hash__
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

p1 = Point(1, 2)
p2 = Point(1, 2)
print(p1 == p2)  # True

# Can now use in sets and as dict keys
points = {p1, p2}
print(len(points))  # 1 (deduplicated)
```

---

## 🔐 Security Considerations

### 1. Protect Sensitive Data

```python
class User:
    def __init__(self, username, password):
        self.username = username
        self._password_hash = self._hash_password(password)

    def _hash_password(self, password):
        """Use proper hashing (bcrypt, argon2)"""
        import hashlib
        return hashlib.sha256(password.encode()).hexdigest()

    def verify_password(self, password):
        """Verify password without exposing hash"""
        return self._hash_password(password) == self._password_hash

    def __repr__(self):
        # Never expose password/hash in repr
        return f"User(username={self.username!r})"

user = User("alice", "secret123")
print(user)  # User(username='alice') - Safe!
```

### 2. Input Validation

```python
import re
from typing import Optional

class User:
    EMAIL_REGEX = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'

    def __init__(self, username: str, email: str, age: Optional[int] = None):
        self.username = self._validate_username(username)
        self.email = self._validate_email(email)
        self.age = self._validate_age(age) if age is not None else None

    @staticmethod
    def _validate_username(username: str) -> str:
        if not username or not isinstance(username, str):
            raise ValueError("Username must be a non-empty string")

        if len(username) < 3 or len(username) > 50:
            raise ValueError("Username must be 3-50 characters")

        if not username.isalnum():
            raise ValueError("Username must be alphanumeric")

        return username.lower()

    @classmethod
    def _validate_email(cls, email: str) -> str:
        if not email or not isinstance(email, str):
            raise ValueError("Email must be a non-empty string")

        if not re.match(cls.EMAIL_REGEX, email):
            raise ValueError("Invalid email format")

        return email.lower()

    @staticmethod
    def _validate_age(age: int) -> int:
        if not isinstance(age, int):
            raise TypeError("Age must be an integer")

        if age < 0 or age > 150:
            raise ValueError("Age must be between 0 and 150")

        return age

# Usage with validation
try:
    user = User("alice", "alice@example.com", 30)
    print("Valid user created")
except ValueError as e:
    print(f"Validation error: {e}")
```

---

## 🚀 Performance Optimization

### 1. Use **slots** for Memory Efficiency

```python
import sys
from memory_profiler import profile

# Regular class - uses ~400 bytes per instance
class RegularPoint:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

# Optimized with __slots__ - uses ~200 bytes per instance
class OptimizedPoint:
    __slots__ = ['x', 'y', 'z']

    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

# Memory comparison with many instances
def test_memory():
    regular_points = [RegularPoint(i, i+1, i+2) for i in range(10000)]
    optimized_points = [OptimizedPoint(i, i+1, i+2) for i in range(10000)]

    print(f"Regular: {sys.getsizeof(regular_points[0])} bytes per instance")
    print(f"Optimized: {sys.getsizeof(optimized_points[0])} bytes per instance")
```

### 2. Lazy Initialization

```python
class DataProcessor:
    def __init__(self, data_source):
        self.data_source = data_source
        self._data = None
        self._processed = None

    @property
    def data(self):
        """Lazy load data only when accessed"""
        if self._data is None:
            print("Loading data...")
            self._data = self._load_data()
        return self._data

    @property
    def processed(self):
        """Lazy process data only when needed"""
        if self._processed is None:
            print("Processing data...")
            self._processed = self._process_data(self.data)
        return self._processed

    def _load_data(self):
        # Expensive operation
        import time
        time.sleep(1)
        return [1, 2, 3, 4, 5]

    def _process_data(self, data):
        # Expensive operation
        import time
        time.sleep(1)
        return [x * 2 for x in data]

# Data not loaded until accessed
processor = DataProcessor("file.csv")
print("Processor created")  # Instant

# Now data is loaded
result = processor.processed  # Takes 2 seconds
print(result)  # [2, 4, 6, 8, 10]

# Subsequent access is instant (cached)
result2 = processor.processed  # Instant
```

### 3. Object Pooling

```python
class DatabaseConnection:
    """Expensive resource to create"""

    def __init__(self, host, port):
        self.host = host
        self.port = port
        self._connection = self._connect()

    def _connect(self):
        import time
        print(f"Connecting to {self.host}:{self.port}...")
        time.sleep(0.5)  # Simulate connection time
        return f"Connection to {self.host}:{self.port}"

    def query(self, sql):
        return f"Executing: {sql}"

    def close(self):
        print(f"Closing connection to {self.host}:{self.port}")

class ConnectionPool:
    """Reuse connections instead of creating new ones"""

    def __init__(self, host, port, pool_size=5):
        self.host = host
        self.port = port
        self.pool_size = pool_size
        self._available = []
        self._in_use = set()
        self._initialize_pool()

    def _initialize_pool(self):
        print(f"Creating pool of {self.pool_size} connections...")
        for _ in range(self.pool_size):
            conn = DatabaseConnection(self.host, self.port)
            self._available.append(conn)

    def acquire(self):
        """Get connection from pool"""
        if not self._available:
            raise RuntimeError("No connections available")

        conn = self._available.pop()
        self._in_use.add(conn)
        return conn

    def release(self, conn):
        """Return connection to pool"""
        if conn in self._in_use:
            self._in_use.remove(conn)
            self._available.append(conn)

    def close_all(self):
        """Close all connections"""
        for conn in self._available:
            conn.close()
        for conn in self._in_use:
            conn.close()

# Usage
pool = ConnectionPool("localhost", 5432, pool_size=3)

# Reuse connections instead of creating new ones
conn1 = pool.acquire()
result = conn1.query("SELECT * FROM users")
pool.release(conn1)  # Return to pool

conn2 = pool.acquire()  # Reuses existing connection
result = conn2.query("SELECT * FROM orders")
pool.release(conn2)
```

---

## 🏗️ Real-World Use Cases

### 1. Django Model-like Class

```python
from datetime import datetime
from typing import Optional, Dict, Any

class Model:
    """Base model class similar to Django"""

    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            setattr(self, key, value)

    def save(self):
        """Simulate saving to database"""
        print(f"Saving {self.__class__.__name__}: {self.to_dict()}")

    def to_dict(self) -> Dict[str, Any]:
        """Convert to dictionary"""
        return {
            key: value for key, value in self.__dict__.items()
            if not key.startswith('_')
        }

    def __repr__(self):
        class_name = self.__class__.__name__
        attrs = ', '.join(f"{k}={v!r}" for k, v in self.to_dict().items())
        return f"{class_name}({attrs})"

class User(Model):
    """User model"""

    def __init__(self, username: str, email: str,
                 created_at: Optional[datetime] = None):
        self.username = username
        self.email = email
        self.created_at = created_at or datetime.now()
        self.is_active = True

class Order(Model):
    """Order model"""

    def __init__(self, user_id: int, amount: float,
                 created_at: Optional[datetime] = None):
        self.user_id = user_id
        self.amount = amount
        self.created_at = created_at or datetime.now()
        self.status = "pending"

# Usage
user = User(username="alice", email="alice@example.com")
user.save()

order = Order(user_id=1, amount=99.99)
order.save()

print(user)
print(order)
```

### 2. FastAPI Pydantic-like Validation

```python
from typing import Optional, List
from dataclasses import dataclass, field

@dataclass
class BaseModel:
    """Base model with validation"""

    def __post_init__(self):
        self.validate()

    def validate(self):
        """Override in subclasses"""
        pass

@dataclass
class CreateUserRequest(BaseModel):
    username: str
    email: str
    age: Optional[int] = None
    tags: List[str] = field(default_factory=list)

    def validate(self):
        if len(self.username) < 3:
            raise ValueError("Username must be at least 3 characters")

        if '@' not in self.email:
            raise ValueError("Invalid email")

        if self.age is not None and (self.age < 0 or self.age > 150):
            raise ValueError("Age must be between 0 and 150")

@dataclass
class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    age: Optional[int]
    created_at: str

    @classmethod
    def from_user(cls, user, user_id: int):
        return cls(
            id=user_id,
            username=user.username,
            email=user.email,
            age=user.age,
            created_at=datetime.now().isoformat()
        )

# Usage (FastAPI-like)
def create_user_endpoint(request: CreateUserRequest) -> UserResponse:
    # Validation happens automatically in __post_init__
    user_id = 123  # From database
    return UserResponse.from_user(request, user_id)

# Test
request = CreateUserRequest(
    username="alice",
    email="alice@example.com",
    age=30,
    tags=["developer", "python"]
)
response = create_user_endpoint(request)
print(response)
```

### 3. Repository Pattern

```python
from abc import ABC, abstractmethod
from typing import List, Optional

class Repository(ABC):
    """Abstract base repository"""

    @abstractmethod
    def find_by_id(self, id: int):
        pass

    @abstractmethod
    def find_all(self) -> List:
        pass

    @abstractmethod
    def save(self, entity):
        pass

    @abstractmethod
    def delete(self, id: int):
        pass

class UserRepository(Repository):
    """User repository with in-memory storage"""

    def __init__(self):
        self._users = {}
        self._next_id = 1

    def find_by_id(self, id: int) -> Optional['User']:
        return self._users.get(id)

    def find_all(self) -> List['User']:
        return list(self._users.values())

    def find_by_username(self, username: str) -> Optional['User']:
        for user in self._users.values():
            if user.username == username:
                return user
        return None

    def save(self, user: 'User') -> 'User':
        if not hasattr(user, 'id') or user.id is None:
            user.id = self._next_id
            self._next_id += 1
        self._users[user.id] = user
        return user

    def delete(self, id: int) -> bool:
        if id in self._users:
            del self._users[id]
            return True
        return False

class User:
    def __init__(self, username: str, email: str, id: Optional[int] = None):
        self.id = id
        self.username = username
        self.email = email

    def __repr__(self):
        return f"User(id={self.id}, username={self.username!r})"

# Service layer using repository
class UserService:
    def __init__(self, repository: UserRepository):
        self.repository = repository

    def register_user(self, username: str, email: str) -> User:
        # Check if username exists
        existing = self.repository.find_by_username(username)
        if existing:
            raise ValueError(f"Username '{username}' already exists")

        # Create and save user
        user = User(username=username, email=email)
        return self.repository.save(user)

    def get_all_users(self) -> List[User]:
        return self.repository.find_all()

# Usage
repo = UserRepository()
service = UserService(repo)

user1 = service.register_user("alice", "alice@example.com")
user2 = service.register_user("bob", "bob@example.com")

print(service.get_all_users())
```

---

## ❓ Interview Questions

### Q1: What's the difference between class variables and instance variables?

**Answer:**

- **Class variables**: Shared by all instances of the class. Defined in class body.
- **Instance variables**: Unique to each instance. Defined in `__init__`.

```python
class Example:
    class_var = "shared"  # Class variable

    def __init__(self, value):
        self.instance_var = value  # Instance variable

obj1 = Example("obj1")
obj2 = Example("obj2")

print(obj1.class_var)      # "shared"
print(obj2.class_var)      # "shared"
print(obj1.instance_var)   # "obj1"
print(obj2.instance_var)   # "obj2"

Example.class_var = "modified"
print(obj1.class_var)      # "modified"
print(obj2.class_var)      # "modified"
```

### Q2: When should you use `@classmethod` vs `@staticmethod`?

**Answer:**

- **@classmethod**: When you need access to the class (cls) but not the instance
  - Alternative constructors
  - Factory methods
  - Methods that work with class state

- **@staticmethod**: When you don't need access to class or instance
  - Utility functions
  - Helper methods
  - Functions logically grouped with the class

```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    @classmethod
    def from_string(cls, date_string):
        """Alternative constructor - needs cls"""
        year, month, day = map(int, date_string.split('-'))
        return cls(year, month, day)

    @staticmethod
    def is_valid_date(year, month, day):
        """Utility function - doesn't need cls or self"""
        return 1 <= month <= 12 and 1 <= day <= 31

# Usage
date1 = Date(2024, 1, 15)
date2 = Date.from_string("2024-01-15")
valid = Date.is_valid_date(2024, 13, 1)  # False
```

### Q3: How does Python's object creation work internally?

**Answer:**
Object creation involves two steps:

1. **`__new__`**: Allocates memory and creates the object
2. **`__init__`**: Initializes the object's attributes

```python
class Demo:
    def __new__(cls, *args, **kwargs):
        print("1. __new__ called - creating instance")
        instance = super().__new__(cls)
        return instance

    def __init__(self, value):
        print("2. __init__ called - initializing instance")
        self.value = value

obj = Demo(42)
# Output:
# 1. __new__ called - creating instance
# 2. __init__ called - initializing instance
```

### Q4: What are the benefits of using `__slots__`?

**Answer:**
**Benefits:**

1. **Memory savings**: No `__dict__` per instance (50-60% reduction)
2. **Faster attribute access**: Direct slot access vs dictionary lookup
3. **Prevents dynamic attributes**: Catches typos at runtime

**Drawbacks:**

1. Can't add attributes dynamically
2. Can't use weak references (unless `__weakref__` in `__slots__`)
3. Breaks inheritance if not all classes use `__slots__`

**When to use:** Classes with many instances and fixed attributes (e.g., data classes, coordinates, vectors).

```python
class Point:
    __slots__ = ['x', 'y']

    def __init__(self, x, y):
        self.x = x
        self.y = y

# Memory efficient for millions of instances
points = [Point(i, i) for i in range(1000000)]
```

### Q5: How would you implement a singleton pattern in Python?

**Answer:**

```python
# Method 1: Using __new__
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Method 2: Using decorator
def singleton(cls):
    instances = {}

    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

@singleton
class Database:
    def __init__(self):
        print("Connecting to database...")

# Method 3: Using metaclass (most robust)
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    def __init__(self):
        print("Connecting to database...")

# All methods ensure only one instance
db1 = Database()
db2 = Database()
print(db1 is db2)  # True
```

---

## 📚 Summary

### Key Takeaways

1. **Classes are blueprints**, objects are instances
2. **Instance variables** are unique per object, **class variables** are shared
3. Use **`@property`** for getter/setter with validation
4. Use **`@classmethod`** for alternative constructors
5. Use **`@staticmethod`** for utility functions
6. Use **`__slots__`** for memory optimization with many instances
7. Always implement **`__repr__`** and **`__str__`** for debugging
8. Implement **`__eq__`** and **`__hash__`** for proper equality
9. **Validate inputs** in `__init__` for security
10. Follow **Single Responsibility Principle**

### Best Practices Checklist

✅ Single responsibility per class
✅ Validate inputs in `__init__`
✅ Use properties for controlled access
✅ Implement `__repr__` and `__str__`
✅ Use `__slots__` for data-heavy classes
✅ Document classes and methods
✅ Use type hints
✅ Avoid mutable default arguments
✅ Use `@classmethod` for factories
✅ Keep classes focused and cohesive

### Common Patterns

- **Data Classes**: `@dataclass`, `namedtuple`, or classes with `__slots__`
- **Service Classes**: Business logic encapsulation
- **Repository Classes**: Data access layer
- **Factory Classes**: Object creation logic
- **Singleton Classes**: Single instance throughout application

---

**Next**: [02_Inheritance_Polymorphism.md](02_Inheritance_Polymorphism.md)
