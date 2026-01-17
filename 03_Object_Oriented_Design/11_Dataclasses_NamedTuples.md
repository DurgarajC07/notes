# Dataclasses & NamedTuples - Modern Python Data Structures

## 📖 Concept Explanation

Python provides several built-in ways to create data-holding classes with less boilerplate:

1. **`dataclasses`** (Python 3.7+): Full-featured classes with less boilerplate
2. **`NamedTuple`** (Python 3.6+): Immutable, typed tuples
3. **`typing.TypedDict`**: Type hints for dictionaries
4. **`attrs`**: Third-party library (inspiration for dataclasses)

### Why Use Them?

- **Less Boilerplate**: No need for `__init__`, `__repr__`, `__eq__`
- **Type Safety**: Built-in type hints
- **Readability**: Clear data structures
- **Performance**: Optimized implementations
- **Immutability**: NamedTuples are immutable by default

## 🧠 Traditional Class vs Dataclass

### Traditional Approach

```python
class User:
    def __init__(self, id: int, username: str, email: str, active: bool = True):
        self.id = id
        self.username = username
        self.email = email
        self.active = active
    
    def __repr__(self):
        return f"User(id={self.id}, username={self.username}, email={self.email}, active={self.active})"
    
    def __eq__(self, other):
        if not isinstance(other, User):
            return NotImplemented
        return (self.id, self.username, self.email, self.active) == \
               (other.id, other.username, other.email, other.active)

user = User(1, "john", "john@example.com")
print(user)  # User(id=1, username=john, email=john@example.com, active=True)
```

### Dataclass Approach

```python
from dataclasses import dataclass

@dataclass
class User:
    id: int
    username: str
    email: str
    active: bool = True

user = User(1, "john", "john@example.com")
print(user)  # User(id=1, username='john', email='john@example.com', active=True)

# Automatic equality
user1 = User(1, "john", "john@example.com")
user2 = User(1, "john", "john@example.com")
print(user1 == user2)  # True
```

**Result**: 15+ lines → 6 lines with same functionality!

## 1️⃣ Dataclasses Deep Dive

### Basic Dataclass

```python
from dataclasses import dataclass, field
from typing import List, Optional
from datetime import datetime

@dataclass
class Product:
    id: int
    name: str
    price: float
    description: str = ""
    in_stock: bool = True
    created_at: datetime = field(default_factory=datetime.now)
    tags: List[str] = field(default_factory=list)

product = Product(
    id=1,
    name="Laptop",
    price=999.99,
    tags=["electronics", "computers"]
)
print(product)
```

### Dataclass Parameters

```python
@dataclass(
    init=True,           # Generate __init__
    repr=True,           # Generate __repr__
    eq=True,             # Generate __eq__
    order=False,         # Generate __lt__, __le__, __gt__, __ge__
    frozen=False,        # Make immutable (like NamedTuple)
    unsafe_hash=False,   # Generate __hash__
    kw_only=False        # Python 3.10+ keyword-only fields
)
class Example:
    field1: str
    field2: int
```

### Field Options

```python
from dataclasses import dataclass, field

@dataclass
class Config:
    # Default value
    debug: bool = False
    
    # Default factory (for mutable defaults)
    allowed_hosts: List[str] = field(default_factory=list)
    
    # Exclude from __init__
    computed: int = field(init=False)
    
    # Exclude from __repr__
    password: str = field(repr=False, default="")
    
    # Exclude from comparison
    last_modified: datetime = field(compare=False, default_factory=datetime.now)
    
    # Metadata (doesn't affect behavior, used for documentation)
    api_key: str = field(metadata={"sensitive": True}, default="")
    
    def __post_init__(self):
        # Called after __init__
        self.computed = len(self.allowed_hosts)

config = Config(allowed_hosts=["localhost", "example.com"])
print(config.computed)  # 2
```

### Frozen Dataclass (Immutable)

```python
@dataclass(frozen=True)
class Point:
    x: int
    y: int

point = Point(10, 20)
# point.x = 30  # FrozenInstanceError!

# Can be used as dict key (hashable)
points = {point: "location"}
```

### Inheritance with Dataclasses

```python
@dataclass
class Person:
    name: str
    age: int

@dataclass
class Employee(Person):
    employee_id: str
    salary: float

emp = Employee(
    name="John",
    age=30,
    employee_id="E123",
    salary=75000.0
)
print(emp)
```

### Post-Init Processing

```python
from dataclasses import dataclass, field

@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False)
    
    def __post_init__(self):
        self.area = self.width * self.height

rect = Rectangle(10, 20)
print(rect.area)  # 200
```

## 2️⃣ NamedTuple

### Basic NamedTuple

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: int
    y: int

point = Point(10, 20)
print(point.x, point.y)  # 10 20
print(point[0], point[1])  # 10 20 (tuple-like access)

# Immutable
# point.x = 30  # AttributeError

# Unpack like tuple
x, y = point
```

### NamedTuple with Defaults

```python
class User(NamedTuple):
    id: int
    username: str
    email: str
    active: bool = True
    role: str = "user"

user = User(1, "john", "john@example.com")
print(user)  # User(id=1, username='john', email='john@example.com', active=True, role='user')
```

### NamedTuple with Methods

```python
from typing import NamedTuple

class Vector(NamedTuple):
    x: float
    y: float
    
    def magnitude(self) -> float:
        return (self.x ** 2 + self.y ** 2) ** 0.5
    
    def dot(self, other: 'Vector') -> float:
        return self.x * other.x + self.y * other.y
    
    def __add__(self, other: 'Vector') -> 'Vector':
        return Vector(self.x + other.x, self.y + other.y)

v1 = Vector(3, 4)
v2 = Vector(1, 2)

print(v1.magnitude())  # 5.0
print(v1.dot(v2))      # 11.0
print(v1 + v2)         # Vector(x=4, y=6)
```

## 🏗️ Django Integration

### Dataclass for API Responses

```python
from dataclasses import dataclass, asdict
from typing import List, Optional
from django.http import JsonResponse

@dataclass
class UserResponse:
    id: int
    username: str
    email: str
    is_active: bool
    
    @classmethod
    def from_model(cls, user):
        return cls(
            id=user.id,
            username=user.username,
            email=user.email,
            is_active=user.is_active
        )

@dataclass
class PaginatedResponse:
    count: int
    next: Optional[str]
    previous: Optional[str]
    results: List[UserResponse]

# views.py
from django.contrib.auth.models import User

def list_users(request):
    users = User.objects.all()[:10]
    
    response = PaginatedResponse(
        count=users.count(),
        next=None,
        previous=None,
        results=[UserResponse.from_model(u) for u in users]
    )
    
    return JsonResponse(asdict(response))
```

### Django Form Data with Dataclass

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class UserRegistrationData:
    username: str
    email: str
    password: str
    first_name: Optional[str] = None
    last_name: Optional[str] = None
    
    def validate(self) -> List[str]:
        errors = []
        
        if len(self.username) < 3:
            errors.append("Username must be at least 3 characters")
        
        if "@" not in self.email:
            errors.append("Invalid email")
        
        if len(self.password) < 8:
            errors.append("Password must be at least 8 characters")
        
        return errors
    
    def create_user(self):
        from django.contrib.auth.models import User
        
        return User.objects.create_user(
            username=self.username,
            email=self.email,
            password=self.password,
            first_name=self.first_name or "",
            last_name=self.last_name or ""
        )

# views.py
def register(request):
    data = UserRegistrationData(
        username=request.POST['username'],
        email=request.POST['email'],
        password=request.POST['password']
    )
    
    errors = data.validate()
    if errors:
        return JsonResponse({'errors': errors}, status=400)
    
    user = data.create_user()
    return JsonResponse({'user_id': user.id})
```

### Dataclass for Query Results

```python
from dataclasses import dataclass
from decimal import Decimal
from django.db.models import Sum, Count, Avg

@dataclass
class OrderStatistics:
    total_orders: int
    total_revenue: Decimal
    average_order_value: Decimal
    top_product: str
    
    @classmethod
    def from_queryset(cls, orders_qs):
        stats = orders_qs.aggregate(
            total_orders=Count('id'),
            total_revenue=Sum('total_amount'),
            average_order_value=Avg('total_amount')
        )
        
        top_product = orders_qs.values('product__name') \
            .annotate(count=Count('id')) \
            .order_by('-count') \
            .first()
        
        return cls(
            total_orders=stats['total_orders'],
            total_revenue=stats['total_revenue'] or Decimal('0'),
            average_order_value=stats['average_order_value'] or Decimal('0'),
            top_product=top_product['product__name'] if top_product else "N/A"
        )

# views.py
def order_stats(request):
    from orders.models import Order
    
    stats = OrderStatistics.from_queryset(Order.objects.all())
    return JsonResponse(asdict(stats))
```

## 🚀 FastAPI Integration

### Pydantic vs Dataclass

```python
from fastapi import FastAPI
from pydantic import BaseModel
from dataclasses import dataclass

app = FastAPI()

# Pydantic (recommended for FastAPI)
class UserCreate(BaseModel):
    username: str
    email: str
    password: str

# Dataclass (also works)
@dataclass
class UserResponse:
    id: int
    username: str
    email: str

@app.post("/users", response_model=UserResponse)
async def create_user(user: UserCreate):
    # Create user...
    return UserResponse(id=1, username=user.username, email=user.email)
```

### Using Dataclasses for Configuration

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class DatabaseConfig:
    host: str
    port: int
    database: str
    username: str
    password: str
    pool_size: int = 10
    ssl_mode: Optional[str] = None
    
    def get_connection_url(self) -> str:
        return f"postgresql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"

@dataclass
class AppConfig:
    debug: bool
    secret_key: str
    database: DatabaseConfig
    redis_url: str
    
    @classmethod
    def from_env(cls):
        import os
        
        return cls(
            debug=os.getenv('DEBUG', 'False') == 'True',
            secret_key=os.getenv('SECRET_KEY', 'dev-secret'),
            database=DatabaseConfig(
                host=os.getenv('DB_HOST', 'localhost'),
                port=int(os.getenv('DB_PORT', '5432')),
                database=os.getenv('DB_NAME', 'mydb'),
                username=os.getenv('DB_USER', 'postgres'),
                password=os.getenv('DB_PASSWORD', '')
            ),
            redis_url=os.getenv('REDIS_URL', 'redis://localhost')
        )

# Usage
config = AppConfig.from_env()
print(config.database.get_connection_url())
```

## ✅ Best Practices

### 1. Use Dataclass for DTOs (Data Transfer Objects)

```python
from dataclasses import dataclass
from typing import List

@dataclass
class CreateOrderRequest:
    user_id: int
    product_ids: List[int]
    shipping_address: str
    payment_method: str

@dataclass
class OrderCreatedResponse:
    order_id: int
    total_amount: float
    estimated_delivery: str
```

### 2. Use NamedTuple for Immutable Data

```python
from typing import NamedTuple

class Coordinates(NamedTuple):
    latitude: float
    longitude: float

class DatabaseCredentials(NamedTuple):
    host: str
    port: int
    username: str
    password: str
```

### 3. Dataclass for Complex Validation

```python
from dataclasses import dataclass
from typing import Optional
import re

@dataclass
class Email:
    address: str
    
    def __post_init__(self):
        if not re.match(r'^[\w\.-]+@[\w\.-]+\.\w+$', self.address):
            raise ValueError(f"Invalid email: {self.address}")

@dataclass
class User:
    username: str
    email: Email
    age: int
    
    def __post_init__(self):
        if self.age < 0:
            raise ValueError("Age cannot be negative")
        
        if len(self.username) < 3:
            raise ValueError("Username too short")

# Usage
try:
    user = User(
        username="john",
        email=Email("invalid-email"),
        age=25
    )
except ValueError as e:
    print(e)  # "Invalid email: invalid-email"
```

### 4. Default Factory for Mutable Defaults

```python
from dataclasses import dataclass, field
from typing import List
from datetime import datetime

# WRONG - Mutable default shared across instances
# @dataclass
# class Wrong:
#     items: List[str] = []  # BAD!

# CORRECT - Use default_factory
@dataclass
class Correct:
    items: List[str] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
```

## ⚡ Performance Comparison

```python
import timeit
from dataclasses import dataclass
from typing import NamedTuple

# Traditional class
class RegularUser:
    def __init__(self, id: int, name: str):
        self.id = id
        self.name = name

# Dataclass
@dataclass
class DataclassUser:
    id: int
    name: str

# NamedTuple
class NamedTupleUser(NamedTuple):
    id: int
    name: str

# Performance test
regular = timeit.timeit('RegularUser(1, "John")', globals=globals(), number=100000)
dataclass = timeit.timeit('DataclassUser(1, "John")', globals=globals(), number=100000)
namedtuple = timeit.timeit('NamedTupleUser(1, "John")', globals=globals(), number=100000)

print(f"Regular: {regular:.4f}s")
print(f"Dataclass: {dataclass:.4f}s")
print(f"NamedTuple: {namedtuple:.4f}s")  # Usually fastest
```

## ❌ Common Mistakes

### 1. Mutable Default Arguments

```python
# WRONG
@dataclass
class Wrong:
    items: List[str] = []  # Shared across all instances!

# CORRECT
@dataclass
class Correct:
    items: List[str] = field(default_factory=list)
```

### 2. Forgetting frozen=True for Hashable

```python
# WRONG - Can't use as dict key
@dataclass
class Point:
    x: int
    y: int

# point_dict = {Point(1, 2): "value"}  # TypeError: unhashable type

# CORRECT
@dataclass(frozen=True)
class Point:
    x: int
    y: int

point_dict = {Point(1, 2): "value"}  # Works!
```

## ❓ Interview Questions

### Q1: When to use dataclass vs NamedTuple?

**Answer**:
- **Dataclass**: Mutable, need methods, complex validation, Django/FastAPI models
- **NamedTuple**: Immutable, simple data, need hashability, return multiple values from functions

### Q2: Can dataclasses have methods?

**Answer**: Yes! Dataclasses are regular classes with generated methods. You can add any methods you want.

```python
@dataclass
class Rectangle:
    width: float
    height: float
    
    def area(self) -> float:
        return self.width * self.height
```

### Q3: How to convert dataclass to dict?

**Answer**: Use `asdict()`:

```python
from dataclasses import asdict

user = User(1, "john")
user_dict = asdict(user)  # {'id': 1, 'username': 'john'}
```

## 📚 Summary

**Dataclasses**:
- ✅ Less boilerplate
- ✅ Type hints
- ✅ Mutable by default
- ✅ Can add methods
- ✅ Use for DTOs, API responses, configuration

**NamedTuples**:
- ✅ Immutable
- ✅ Hashable (can be dict keys)
- ✅ Lightweight
- ✅ Tuple-like access
- ✅ Use for coordinates, return values, immutable data

**When to Use**:
- API models → Dataclass
- Configuration → Dataclass
- Coordinates/Points → NamedTuple
- Return multiple values → NamedTuple
- Database records → Dataclass

Both are essential for modern Python backend development with Django and FastAPI!
