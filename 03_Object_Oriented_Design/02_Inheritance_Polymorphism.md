# 📘 Inheritance and Polymorphism in Python

## 📖 Concept Explanation

### What is Inheritance?

**Inheritance** allows a class (child/derived) to inherit attributes and methods from another class (parent/base). It promotes code reuse and establishes "is-a" relationships.

### What is Polymorphism?

**Polymorphism** means "many forms." It allows objects of different classes to be treated through a common interface while behaving differently based on their actual type.

### Basic Inheritance

```python
# Parent class (Base class)
class Animal:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def speak(self):
        return "Some sound"

    def info(self):
        return f"{self.name} is {self.age} years old"

# Child class (Derived class)
class Dog(Animal):
    def __init__(self, name, age, breed):
        super().__init__(name, age)  # Call parent constructor
        self.breed = breed

    def speak(self):  # Override parent method
        return "Woof!"

    def fetch(self):  # New method specific to Dog
        return f"{self.name} is fetching the ball"

class Cat(Animal):
    def speak(self):  # Override parent method
        return "Meow!"

    def climb(self):  # New method specific to Cat
        return f"{self.name} is climbing a tree"

# Usage
dog = Dog("Buddy", 3, "Golden Retriever")
cat = Cat("Whiskers", 2)

print(dog.info())    # "Buddy is 3 years old" (inherited)
print(dog.speak())   # "Woof!" (overridden)
print(dog.fetch())   # "Buddy is fetching the ball" (new method)

print(cat.info())    # "Whiskers is 2 years old" (inherited)
print(cat.speak())   # "Meow!" (overridden)
print(cat.climb())   # "Whiskers is climbing a tree" (new method)
```

### Types of Inheritance

```python
# 1. Single Inheritance
class Parent:
    pass

class Child(Parent):
    pass

# 2. Multiple Inheritance
class Father:
    def skills(self):
        return "Carpentry"

class Mother:
    def skills(self):
        return "Cooking"

class Child(Father, Mother):  # Multiple inheritance
    pass

child = Child()
print(child.skills())  # "Carpentry" (from Father - left to right)

# 3. Multilevel Inheritance
class Grandparent:
    def hobby(self):
        return "Gardening"

class Parent(Grandparent):
    def profession(self):
        return "Engineer"

class Child(Parent):
    def interest(self):
        return "Gaming"

child = Child()
print(child.hobby())      # "Gardening" (from Grandparent)
print(child.profession()) # "Engineer" (from Parent)
print(child.interest())   # "Gaming" (own method)

# 4. Hierarchical Inheritance
class Vehicle:
    def start(self):
        return "Vehicle starting"

class Car(Vehicle):
    pass

class Motorcycle(Vehicle):
    pass

# Both Car and Motorcycle inherit from Vehicle
```

### Polymorphism Examples

```python
# 1. Method Overriding (Runtime Polymorphism)
class Shape:
    def area(self):
        pass

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def area(self):
        import math
        return math.pi * self.radius ** 2

# Polymorphic behavior
shapes = [
    Rectangle(10, 5),
    Circle(7),
    Rectangle(3, 4)
]

for shape in shapes:
    print(f"Area: {shape.area():.2f}")
# Output:
# Area: 50.00
# Area: 153.94
# Area: 12.00

# 2. Duck Typing (If it walks like a duck...)
class Duck:
    def speak(self):
        return "Quack!"

class Dog:
    def speak(self):
        return "Woof!"

class Cat:
    def speak(self):
        return "Meow!"

def make_it_speak(animal):
    """Polymorphic function - works with any object having speak()"""
    print(animal.speak())

# All different types, but same interface
make_it_speak(Duck())  # "Quack!"
make_it_speak(Dog())   # "Woof!"
make_it_speak(Cat())   # "Meow!"
```

---

## 🧠 Why It Matters

### Real-World Benefits

1. **Code Reuse**: Write once, use in multiple classes
2. **Maintainability**: Changes in parent affect all children
3. **Extensibility**: Easy to add new types without modifying existing code
4. **Abstraction**: Hide implementation details, expose interface
5. **Design Patterns**: Foundation for Factory, Strategy, Template Method, etc.

### Industry Usage

- **Django Models**: All models inherit from `models.Model`
- **Django Views**: Class-based views use inheritance hierarchy
- **Exception Handling**: Custom exceptions inherit from base exceptions
- **FastAPI Dependencies**: Dependency injection through inheritance
- **Testing**: Test cases inherit from `unittest.TestCase` or `pytest`

---

## ⚙️ Internal Working

### Method Resolution Order (MRO)

Python uses C3 linearization algorithm to determine method lookup order in multiple inheritance.

```python
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

# Check MRO
print(D.__mro__)
# (<class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>,
#  <class '__main__.A'>, <class 'object'>)

print(D.mro())  # Same as __mro__ but as list

d = D()
print(d.method())  # "B" (found in B first according to MRO)
```

### super() Mechanism

```python
class A:
    def __init__(self):
        print("A.__init__")
        super().__init__()

class B:
    def __init__(self):
        print("B.__init__")
        super().__init__()

class C(A, B):
    def __init__(self):
        print("C.__init__")
        super().__init__()

# Creating C instance
c = C()
# Output (follows MRO):
# C.__init__
# A.__init__
# B.__init__

# MRO: C -> A -> B -> object
print(C.__mro__)
```

### isinstance() and issubclass()

```python
class Animal:
    pass

class Dog(Animal):
    pass

class GoldenRetriever(Dog):
    pass

dog = Dog()
golden = GoldenRetriever()

# isinstance checks if object is instance of class or its parents
print(isinstance(dog, Dog))         # True
print(isinstance(dog, Animal))      # True
print(isinstance(golden, Dog))      # True
print(isinstance(dog, GoldenRetriever))  # False

# issubclass checks class hierarchy
print(issubclass(Dog, Animal))      # True
print(issubclass(GoldenRetriever, Animal))  # True
print(issubclass(Animal, Dog))      # False
```

---

## ✅ Best Practices

### 1. Liskov Substitution Principle

Child classes should be substitutable for parent classes without breaking functionality.

```python
# ✅ GOOD: Child can replace parent
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

    def perimeter(self):
        return 2 * (self.width + self.height)

class Square(Rectangle):
    def __init__(self, side):
        super().__init__(side, side)

# Square can be used anywhere Rectangle is expected
def print_rectangle_info(rect: Rectangle):
    print(f"Area: {rect.area()}")
    print(f"Perimeter: {rect.perimeter()}")

rect = Rectangle(10, 5)
square = Square(5)

print_rectangle_info(rect)    # Works
print_rectangle_info(square)  # Also works (substitutable)

# ❌ BAD: Violates LSP
class Rectangle:
    def __init__(self, width, height):
        self._width = width
        self._height = height

    def set_width(self, width):
        self._width = width

    def set_height(self, height):
        self._height = height

    def area(self):
        return self._width * self._height

class Square(Rectangle):
    def set_width(self, width):
        self._width = width
        self._height = width  # Violates expectations!

    def set_height(self, height):
        self._width = height  # Violates expectations!
        self._height = height

# This breaks LSP
def test_rectangle(rect: Rectangle):
    rect.set_width(5)
    rect.set_height(10)
    assert rect.area() == 50  # Fails for Square!

rect = Rectangle(0, 0)
test_rectangle(rect)  # Pass

square = Square(0, 0)
# test_rectangle(square)  # Would fail! Square not substitutable
```

### 2. Favor Composition Over Inheritance

```python
# ❌ BAD: Deep inheritance hierarchy
class Animal:
    pass

class Mammal(Animal):
    pass

class Carnivore(Mammal):
    pass

class Dog(Carnivore):
    pass

# ✅ GOOD: Composition for behavior
class WalkBehavior:
    def walk(self):
        return "Walking on legs"

class FlyBehavior:
    def fly(self):
        return "Flying in the sky"

class SwimBehavior:
    def swim(self):
        return "Swimming in water"

class Animal:
    def __init__(self, name):
        self.name = name

class Dog(Animal):
    def __init__(self, name):
        super().__init__(name)
        self.walk_behavior = WalkBehavior()
        self.swim_behavior = SwimBehavior()

    def move(self):
        return self.walk_behavior.walk()

    def swim(self):
        return self.swim_behavior.swim()

class Duck(Animal):
    def __init__(self, name):
        super().__init__(name)
        self.walk_behavior = WalkBehavior()
        self.fly_behavior = FlyBehavior()
        self.swim_behavior = SwimBehavior()

    def move(self):
        return self.walk_behavior.walk()

    def fly(self):
        return self.fly_behavior.fly()

    def swim(self):
        return self.swim_behavior.swim()
```

### 3. Use super() Correctly

```python
# ✅ GOOD: Cooperative multiple inheritance
class Base:
    def __init__(self, value):
        self.value = value
        print(f"Base.__init__({value})")

class Mixin:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        print("Mixin.__init__")
        self.mixin_attr = "mixin"

class Derived(Mixin, Base):
    def __init__(self, value, extra):
        super().__init__(value=value)
        self.extra = extra
        print(f"Derived.__init__({extra})")

obj = Derived(10, 20)
# Output:
# Base.__init__(10)
# Mixin.__init__
# Derived.__init__(20)

# ❌ BAD: Explicitly calling parent __init__
class Derived(Mixin, Base):
    def __init__(self, value, extra):
        Base.__init__(self, value)  # Skips Mixin!
        self.extra = extra
```

### 4. Abstract Base Classes for Interfaces

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    """Abstract base class defining payment interface"""

    @abstractmethod
    def process_payment(self, amount: float) -> bool:
        """Process payment - must be implemented by subclasses"""
        pass

    @abstractmethod
    def refund_payment(self, transaction_id: str) -> bool:
        """Refund payment - must be implemented by subclasses"""
        pass

    def log_transaction(self, transaction_id: str):
        """Concrete method available to all subclasses"""
        print(f"Transaction {transaction_id} logged")

class StripePayment(PaymentProcessor):
    def process_payment(self, amount: float) -> bool:
        print(f"Processing ${amount} via Stripe")
        return True

    def refund_payment(self, transaction_id: str) -> bool:
        print(f"Refunding transaction {transaction_id} via Stripe")
        return True

class PayPalPayment(PaymentProcessor):
    def process_payment(self, amount: float) -> bool:
        print(f"Processing ${amount} via PayPal")
        return True

    def refund_payment(self, transaction_id: str) -> bool:
        print(f"Refunding transaction {transaction_id} via PayPal")
        return True

# Can't instantiate abstract class
# processor = PaymentProcessor()  # TypeError!

# Can instantiate concrete implementations
stripe = StripePayment()
paypal = PayPalPayment()

# Polymorphic usage
def process_order_payment(processor: PaymentProcessor, amount: float):
    if processor.process_payment(amount):
        processor.log_transaction("TXN123")

process_order_payment(stripe, 99.99)
process_order_payment(paypal, 49.99)
```

---

## ❌ Common Mistakes

### 1. Deep Inheritance Hierarchies

```python
# ❌ BAD: Too many levels
class A:
    pass

class B(A):
    pass

class C(B):
    pass

class D(C):
    pass

class E(D):
    pass

# Hard to understand, maintain, and extend

# ✅ GOOD: Shallow hierarchy with composition
class Component:
    pass

class Feature1:
    pass

class Feature2:
    pass

class MyClass:
    def __init__(self):
        self.component = Component()
        self.feature1 = Feature1()
        self.feature2 = Feature2()
```

### 2. Forgetting to Call super().**init**()

```python
# ❌ BAD: Parent not initialized
class Parent:
    def __init__(self, name):
        self.name = name
        self.initialized = True

class Child(Parent):
    def __init__(self, name, age):
        # Forgot to call super().__init__()!
        self.age = age

child = Child("Alice", 30)
# print(child.name)  # AttributeError!
# print(child.initialized)  # AttributeError!

# ✅ GOOD: Always call super().__init__()
class Child(Parent):
    def __init__(self, name, age):
        super().__init__(name)
        self.age = age

child = Child("Alice", 30)
print(child.name)        # "Alice"
print(child.initialized) # True
```

### 3. Multiple Inheritance Diamond Problem

```python
# ❌ BAD: Ambiguous diamond inheritance
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

# Which method() is called?
d = D()
print(d.method())  # "B" (determined by MRO)

# ✅ GOOD: Be explicit with super() or redesign
class D(B, C):
    def method(self):
        # Explicit control
        return f"{super().method()} + D"

d = D()
print(d.method())  # "B + D"
```

### 4. Overriding Without Understanding

```python
# ❌ BAD: Breaking parent contract
class FileHandler:
    def open(self, filename):
        self.file = open(filename, 'r')
        return self.file

    def close(self):
        if hasattr(self, 'file'):
            self.file.close()

class BrokenHandler(FileHandler):
    def close(self):
        # Forgot to actually close!
        print("Closing file")
        # Missing: self.file.close()

# ✅ GOOD: Extend parent behavior
class GoodHandler(FileHandler):
    def close(self):
        print("Closing file")
        super().close()  # Call parent implementation
```

---

## 🔐 Security Considerations

### 1. Method Override Validation

```python
from abc import ABC, abstractmethod

class SecureBaseClass(ABC):
    """Base class with security constraints"""

    @abstractmethod
    def authenticate(self, credentials: dict) -> bool:
        """Must validate credentials"""
        pass

    @abstractmethod
    def authorize(self, user_id: int, resource: str) -> bool:
        """Must check authorization"""
        pass

class SecureAPI(SecureBaseClass):
    def authenticate(self, credentials: dict) -> bool:
        # Must implement proper authentication
        username = credentials.get('username')
        password = credentials.get('password')

        if not username or not password:
            return False

        # Proper password hashing comparison
        import hashlib
        password_hash = hashlib.sha256(password.encode()).hexdigest()
        # Check against stored hash
        return self._verify_hash(username, password_hash)

    def authorize(self, user_id: int, resource: str) -> bool:
        # Must implement proper authorization
        return self._check_permissions(user_id, resource)

    def _verify_hash(self, username: str, password_hash: str) -> bool:
        # Implementation
        return True

    def _check_permissions(self, user_id: int, resource: str) -> bool:
        # Implementation
        return True
```

### 2. Prevent Unsafe Overrides

```python
class BankAccount:
    def __init__(self, balance: float):
        self._balance = balance

    @property
    def balance(self):
        return self._balance

    def withdraw(self, amount: float):
        """Critical method - should not be overridden unsafely"""
        if amount <= 0:
            raise ValueError("Amount must be positive")
        if amount > self._balance:
            raise ValueError("Insufficient funds")

        self._balance -= amount
        self._log_transaction("WITHDRAWAL", amount)

    def _log_transaction(self, type: str, amount: float):
        print(f"{type}: ${amount}")

    def __init_subclass__(cls, **kwargs):
        """Hook to validate subclasses"""
        super().__init_subclass__(**kwargs)

        # Ensure critical methods aren't overridden improperly
        if 'withdraw' in cls.__dict__:
            print(f"Warning: {cls.__name__} overrides withdraw()")

class SavingsAccount(BankAccount):
    def withdraw(self, amount: float):
        # Override with additional logic
        if amount > 500:
            raise ValueError("Daily withdrawal limit exceeded")
        super().withdraw(amount)  # Call parent implementation
```

---

## 🚀 Performance Optimization

### 1. Avoid Deep Inheritance for Hot Paths

```python
import timeit

# Scenario: Method called millions of times

# ❌ SLOWER: Deep inheritance
class Level1:
    def compute(self, x):
        return x

class Level2(Level1):
    pass

class Level3(Level2):
    pass

class Level4(Level3):
    def compute(self, x):
        return super().compute(x) * 2

# ✅ FASTER: Direct implementation
class Direct:
    def compute(self, x):
        return x * 2

# Benchmark
deep = Level4()
direct = Direct()

time_deep = timeit.timeit(lambda: deep.compute(10), number=1000000)
time_direct = timeit.timeit(lambda: direct.compute(10), number=1000000)

print(f"Deep inheritance: {time_deep:.4f}s")
print(f"Direct: {time_direct:.4f}s")
print(f"Speedup: {time_deep/time_direct:.2f}x")
```

### 2. Cache Method Resolution

```python
class BaseService:
    def process(self, data):
        return data

class CachedService(BaseService):
    def __init__(self):
        # Cache frequently used methods
        self._process_method = super().process

    def process(self, data):
        # Use cached method reference
        return self._process_method(data)
```

### 3. Use **slots** in Inheritance

```python
class BaseModel:
    __slots__ = ['id', 'created_at']

    def __init__(self, id, created_at):
        self.id = id
        self.created_at = created_at

class User(BaseModel):
    __slots__ = ['username', 'email']  # Add new slots

    def __init__(self, id, created_at, username, email):
        super().__init__(id, created_at)
        self.username = username
        self.email = email

# Memory efficient inheritance
import sys
user = User(1, "2024-01-01", "alice", "alice@example.com")
print(f"Size: {sys.getsizeof(user)} bytes")
```

---

## 🏗️ Real-World Use Cases

### 1. Django Model Inheritance

```python
from datetime import datetime

class TimeStampedModel:
    """Abstract base model with timestamp fields"""

    def __init__(self):
        self.created_at = datetime.now()
        self.updated_at = datetime.now()

    def save(self):
        self.updated_at = datetime.now()
        print(f"Saving {self.__class__.__name__}...")

class User(TimeStampedModel):
    def __init__(self, username, email):
        super().__init__()
        self.username = username
        self.email = email

    def __repr__(self):
        return f"User(username={self.username!r})"

class Post(TimeStampedModel):
    def __init__(self, title, content, author_id):
        super().__init__()
        self.title = title
        self.content = content
        self.author_id = author_id

    def __repr__(self):
        return f"Post(title={self.title!r})"

# Usage
user = User("alice", "alice@example.com")
user.save()

post = Post("Hello World", "My first post", 1)
post.save()

print(user)
print(f"User created at: {user.created_at}")
print(f"User updated at: {user.updated_at}")
```

### 2. Django Class-Based Views Hierarchy

```python
class View:
    """Base view"""

    def dispatch(self, request, *args, **kwargs):
        """Dispatch request to appropriate handler"""
        handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
        return handler(request, *args, **kwargs)

    def http_method_not_allowed(self, request, *args, **kwargs):
        return {"error": "Method not allowed"}, 405

class ListView(View):
    """List objects"""

    def get(self, request, *args, **kwargs):
        objects = self.get_queryset()
        return {"objects": objects}, 200

    def get_queryset(self):
        """Override in subclass"""
        raise NotImplementedError

class CreateView(View):
    """Create object"""

    def post(self, request, *args, **kwargs):
        data = request.data
        obj = self.create_object(data)
        return {"object": obj}, 201

    def create_object(self, data):
        """Override in subclass"""
        raise NotImplementedError

class UserListView(ListView):
    """Concrete view for listing users"""

    def get_queryset(self):
        return [
            {"id": 1, "username": "alice"},
            {"id": 2, "username": "bob"}
        ]

class UserCreateView(CreateView):
    """Concrete view for creating users"""

    def create_object(self, data):
        return {
            "id": 3,
            "username": data.get("username"),
            "email": data.get("email")
        }

# Simulate request
class Request:
    def __init__(self, method, data=None):
        self.method = method
        self.data = data or {}

# Usage
list_view = UserListView()
response, status = list_view.dispatch(Request("GET"))
print(f"Status: {status}, Response: {response}")

create_view = UserCreateView()
response, status = create_view.dispatch(
    Request("POST", {"username": "charlie", "email": "charlie@example.com"})
)
print(f"Status: {status}, Response: {response}")
```

### 3. Payment Processing with Strategy Pattern

```python
from abc import ABC, abstractmethod
from enum import Enum

class PaymentStatus(Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

class PaymentMethod(ABC):
    """Abstract payment method"""

    @abstractmethod
    def validate(self, amount: float, **kwargs) -> bool:
        pass

    @abstractmethod
    def charge(self, amount: float, **kwargs) -> dict:
        pass

    @abstractmethod
    def refund(self, transaction_id: str, amount: float) -> dict:
        pass

class CreditCardPayment(PaymentMethod):
    def validate(self, amount: float, **kwargs) -> bool:
        card_number = kwargs.get('card_number', '')
        cvv = kwargs.get('cvv', '')

        if len(card_number) != 16:
            return False
        if len(cvv) != 3:
            return False
        if amount <= 0:
            return False

        return True

    def charge(self, amount: float, **kwargs) -> dict:
        if not self.validate(amount, **kwargs):
            return {
                'status': PaymentStatus.FAILED.value,
                'message': 'Invalid card details'
            }

        print(f"Charging ${amount} to credit card")
        return {
            'status': PaymentStatus.COMPLETED.value,
            'transaction_id': 'CC123456',
            'amount': amount
        }

    def refund(self, transaction_id: str, amount: float) -> dict:
        print(f"Refunding ${amount} to credit card {transaction_id}")
        return {
            'status': PaymentStatus.COMPLETED.value,
            'refund_id': 'RF123456',
            'amount': amount
        }

class PayPalPayment(PaymentMethod):
    def validate(self, amount: float, **kwargs) -> bool:
        email = kwargs.get('email', '')

        if '@' not in email:
            return False
        if amount <= 0:
            return False

        return True

    def charge(self, amount: float, **kwargs) -> dict:
        if not self.validate(amount, **kwargs):
            return {
                'status': PaymentStatus.FAILED.value,
                'message': 'Invalid PayPal email'
            }

        print(f"Charging ${amount} via PayPal")
        return {
            'status': PaymentStatus.COMPLETED.value,
            'transaction_id': 'PP123456',
            'amount': amount
        }

    def refund(self, transaction_id: str, amount: float) -> dict:
        print(f"Refunding ${amount} via PayPal {transaction_id}")
        return {
            'status': PaymentStatus.COMPLETED.value,
            'refund_id': 'RF123456',
            'amount': amount
        }

class PaymentProcessor:
    """Context class using payment strategies"""

    def __init__(self, payment_method: PaymentMethod):
        self.payment_method = payment_method

    def process_payment(self, amount: float, **kwargs) -> dict:
        result = self.payment_method.charge(amount, **kwargs)
        return result

    def process_refund(self, transaction_id: str, amount: float) -> dict:
        result = self.payment_method.refund(transaction_id, amount)
        return result

# Usage
# Process credit card payment
cc_payment = CreditCardPayment()
processor = PaymentProcessor(cc_payment)
result = processor.process_payment(
    99.99,
    card_number='1234567890123456',
    cvv='123'
)
print(result)

# Switch to PayPal
paypal_payment = PayPalPayment()
processor = PaymentProcessor(paypal_payment)
result = processor.process_payment(
    49.99,
    email='user@example.com'
)
print(result)
```

---

## ❓ Interview Questions

### Q1: What's the difference between inheritance and composition?

**Answer:**

**Inheritance (is-a)**:

- Child class inherits from parent class
- Tighter coupling
- Use when there's a clear "is-a" relationship
- Example: Dog is an Animal

**Composition (has-a)**:

- Class contains instances of other classes
- Looser coupling, more flexible
- Use when there's a "has-a" relationship
- Example: Car has an Engine

```python
# Inheritance (is-a)
class Animal:
    def eat(self):
        return "eating"

class Dog(Animal):
    def bark(self):
        return "woof"

# Composition (has-a)
class Engine:
    def start(self):
        return "engine starting"

class Car:
    def __init__(self):
        self.engine = Engine()  # Composition

    def start(self):
        return self.engine.start()
```

**When to use what:**

- Inheritance: Shared behavior, extending functionality
- Composition: Flexibility, multiple behaviors, avoiding deep hierarchies

### Q2: Explain Method Resolution Order (MRO) in Python

**Answer:**

MRO determines the order in which base classes are searched when looking for a method. Python uses C3 linearization algorithm.

```python
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

# MRO: D -> B -> C -> A -> object
print(D.mro())
# [<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>]

d = D()
print(d.method())  # "B" (first in MRO after D)
```

**Rules:**

1. Child before parents
2. Parents in order they're listed
3. If multiple paths to a base, taken only once at the furthest point

### Q3: What is the Liskov Substitution Principle?

**Answer:**

Objects of a subclass should be replaceable for objects of the superclass without breaking the application.

```python
# ✅ GOOD: Follows LSP
class Bird:
    def move(self):
        return "moving"

class Sparrow(Bird):
    def move(self):
        return "flying"  # Still moves, just specifies how

class Penguin(Bird):
    def move(self):
        return "walking"  # Still moves, just specifies how

def make_bird_move(bird: Bird):
    print(bird.move())

# All substitutable
make_bird_move(Bird())
make_bird_move(Sparrow())
make_bird_move(Penguin())

# ❌ BAD: Violates LSP
class Bird:
    def fly(self):
        return "flying"

class Penguin(Bird):
    def fly(self):
        raise Exception("Can't fly!")  # Breaks contract!

# Can't substitute Penguin for Bird
def make_bird_fly(bird: Bird):
    print(bird.fly())

# make_bird_fly(Penguin())  # Raises exception!
```

### Q4: When should you use multiple inheritance?

**Answer:**

Use multiple inheritance sparingly and mainly for:

1. **Mixins**: Small, focused classes providing specific functionality
2. **Interface implementation**: Multiple protocols/interfaces
3. **Composition-like behavior**: When composition is too verbose

```python
# Good use: Mixins
class JSONMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class XMLMixin:
    def to_xml(self):
        return f"<obj>{self.__dict__}</obj>"

class User(JSONMixin, XMLMixin):
    def __init__(self, name, email):
        self.name = name
        self.email = email

user = User("Alice", "alice@example.com")
print(user.to_json())
print(user.to_xml())
```

**Avoid:**

- Deep multiple inheritance hierarchies
- Multiple inheritance with state (shared attributes)
- When composition would be clearer

### Q5: How does super() work in multiple inheritance?

**Answer:**

`super()` follows the MRO (Method Resolution Order), not just the parent class.

```python
class A:
    def method(self):
        print("A.method")
        return "A"

class B(A):
    def method(self):
        print("B.method")
        result = super().method()  # Calls C.method (next in MRO)
        return f"B->{result}"

class C(A):
    def method(self):
        print("C.method")
        result = super().method()  # Calls A.method
        return f"C->{result}"

class D(B, C):
    def method(self):
        print("D.method")
        result = super().method()  # Calls B.method (next in MRO)
        return f"D->{result}"

# MRO: D -> B -> C -> A -> object
print(D.mro())

d = D()
result = d.method()
# Output:
# D.method
# B.method
# C.method
# A.method

print(result)  # "D->B->C->A"
```

**Key point**: `super()` calls the next class in MRO, not necessarily the direct parent.

---

## 📚 Summary

### Key Takeaways

1. **Inheritance** enables code reuse and "is-a" relationships
2. **Polymorphism** allows objects to be treated through common interfaces
3. **MRO** determines method lookup order (use `.mro()` to check)
4. **super()** follows MRO, not just parent class
5. **Liskov Substitution Principle**: child must be substitutable for parent
6. **Favor composition over inheritance** for flexibility
7. **Abstract base classes** define interfaces/contracts
8. **Mixins** provide focused, reusable functionality
9. **Duck typing** enables polymorphism without inheritance
10. **Avoid deep hierarchies** (max 2-3 levels recommended)

### Best Practices Checklist

✅ Use inheritance for "is-a" relationships
✅ Use composition for "has-a" relationships
✅ Follow Liskov Substitution Principle
✅ Call `super().__init__()` in child constructors
✅ Use abstract base classes for interfaces
✅ Keep inheritance hierarchies shallow (2-3 levels)
✅ Use mixins for cross-cutting concerns
✅ Check MRO in multiple inheritance
✅ Use `isinstance()` and `issubclass()` appropriately
✅ Document inheritance relationships clearly

### When to Use What

| Pattern              | Use Case                        | Example                    |
| -------------------- | ------------------------------- | -------------------------- |
| Single Inheritance   | Clear parent-child relationship | Dog extends Animal         |
| Multiple Inheritance | Mixins, multiple protocols      | User with JSONMixin        |
| Composition          | Flexible behavior combination   | Car has Engine             |
| Abstract Base Class  | Define interface/contract       | PaymentProcessor interface |
| Duck Typing          | Flexible polymorphism           | Any object with `.speak()` |

---

**Next**: [03_MRO_Super.md](03_MRO_Super.md)
