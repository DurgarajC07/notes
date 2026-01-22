# 🎯 SOLID Principles in Python

## Overview

SOLID principles for writing maintainable, flexible, and scalable object-oriented code - essential for clean architecture.

---

## What is SOLID?

```python
"""
SOLID = 5 Design Principles by Robert C. Martin

S - Single Responsibility Principle (SRP)
O - Open/Closed Principle (OCP)
L - Liskov Substitution Principle (LSP)
I - Interface Segregation Principle (ISP)
D - Dependency Inversion Principle (DIP)

Benefits:
✅ Easier to maintain
✅ Easier to test
✅ Easier to extend
✅ Reduces coupling
✅ Increases cohesion

When to Apply:
- Building complex systems
- Long-term projects
- Team development
- Frequent changes expected
"""
```

---

## 1. Single Responsibility Principle (SRP)

### A class should have only one reason to change

```python
# ❌ BAD: Multiple responsibilities
class User:
    """User class doing too much"""

    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

    def save_to_database(self):
        """Database logic - different responsibility"""
        # SQL code
        pass

    def send_email(self, message: str):
        """Email logic - different responsibility"""
        # SMTP code
        pass

    def generate_report(self):
        """Report logic - different responsibility"""
        # Report generation
        pass

"""
Problems:
- Changes to database affect User class
- Changes to email affect User class
- Changes to reports affect User class
- Hard to test (need to mock DB, email, etc.)
- Violates SRP - 4 reasons to change!
"""

# ✅ GOOD: Separate responsibilities
class User:
    """User model - only data and behavior"""

    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

    def get_full_name(self) -> str:
        """User-specific behavior"""
        return f"{self.name}"

    def is_valid_email(self) -> bool:
        """User-specific validation"""
        return '@' in self.email

class UserRepository:
    """Handle user persistence - single responsibility"""

    def save(self, user: User):
        """Save user to database"""
        # SQL code
        pass

    def find_by_id(self, user_id: int) -> User:
        """Find user by ID"""
        pass

    def delete(self, user: User):
        """Delete user"""
        pass

class EmailService:
    """Handle email sending - single responsibility"""

    def send_email(self, to: str, subject: str, body: str):
        """Send email"""
        # SMTP code
        pass

    def send_welcome_email(self, user: User):
        """Send welcome email to user"""
        self.send_email(
            to=user.email,
            subject="Welcome!",
            body=f"Welcome {user.name}!"
        )

class UserReportGenerator:
    """Generate user reports - single responsibility"""

    def generate_pdf(self, user: User) -> bytes:
        """Generate PDF report"""
        pass

    def generate_csv(self, users: list) -> str:
        """Generate CSV report"""
        pass

"""
Benefits:
✅ Each class has one reason to change
✅ Easy to test (mock one thing at a time)
✅ Can change DB without touching User
✅ Can change email without touching User
✅ Clear separation of concerns
"""
```

---

## 2. Open/Closed Principle (OCP)

### Software entities should be open for extension, closed for modification

```python
# ❌ BAD: Modifying existing code for new features
class PaymentProcessor:
    """Payment processor - needs modification for new payment types"""

    def process_payment(self, payment_type: str, amount: float):
        """Process payment based on type"""
        if payment_type == 'credit_card':
            # Credit card logic
            print(f"Processing credit card payment: ${amount}")
            return True
        elif payment_type == 'paypal':
            # PayPal logic
            print(f"Processing PayPal payment: ${amount}")
            return True
        elif payment_type == 'crypto':
            # Crypto logic - NEW, had to modify this method!
            print(f"Processing crypto payment: ${amount}")
            return True
        else:
            raise ValueError(f"Unknown payment type: {payment_type}")

"""
Problems:
- Adding new payment type requires modifying process_payment
- Risk of breaking existing code
- Violates OCP - not closed for modification
"""

# ✅ GOOD: Extend through abstraction
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    """Abstract payment method - extension point"""

    @abstractmethod
    def process(self, amount: float) -> bool:
        """Process payment"""
        pass

    @abstractmethod
    def validate(self) -> bool:
        """Validate payment method"""
        pass

class CreditCardPayment(PaymentMethod):
    """Credit card payment"""

    def __init__(self, card_number: str, cvv: str):
        self.card_number = card_number
        self.cvv = cvv

    def validate(self) -> bool:
        """Validate card"""
        return len(self.card_number) == 16 and len(self.cvv) == 3

    def process(self, amount: float) -> bool:
        """Process credit card payment"""
        if not self.validate():
            raise ValueError("Invalid card")

        print(f"Processing credit card payment: ${amount}")
        # Credit card processing logic
        return True

class PayPalPayment(PaymentMethod):
    """PayPal payment"""

    def __init__(self, email: str):
        self.email = email

    def validate(self) -> bool:
        """Validate PayPal account"""
        return '@' in self.email

    def process(self, amount: float) -> bool:
        """Process PayPal payment"""
        if not self.validate():
            raise ValueError("Invalid PayPal email")

        print(f"Processing PayPal payment: ${amount}")
        return True

class CryptoPayment(PaymentMethod):
    """Crypto payment - NEW, no modification to existing code!"""

    def __init__(self, wallet_address: str):
        self.wallet_address = wallet_address

    def validate(self) -> bool:
        """Validate wallet"""
        return len(self.wallet_address) == 42

    def process(self, amount: float) -> bool:
        """Process crypto payment"""
        if not self.validate():
            raise ValueError("Invalid wallet")

        print(f"Processing crypto payment: ${amount}")
        return True

class PaymentProcessor:
    """Payment processor - closed for modification, open for extension"""

    def process_payment(self, payment_method: PaymentMethod, amount: float) -> bool:
        """
        Process payment using any payment method

        No modification needed for new payment types!
        """
        return payment_method.process(amount)

# Usage
processor = PaymentProcessor()

# Credit card
cc_payment = CreditCardPayment("1234567812345678", "123")
processor.process_payment(cc_payment, 100.0)

# PayPal
paypal_payment = PayPalPayment("user@example.com")
processor.process_payment(paypal_payment, 50.0)

# Crypto - NEW, no changes to PaymentProcessor!
crypto_payment = CryptoPayment("0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb")
processor.process_payment(crypto_payment, 75.0)

"""
Benefits:
✅ Add new payment types without modifying existing code
✅ Existing code is stable (closed for modification)
✅ New features through extension (open for extension)
✅ Reduces risk of breaking changes
"""
```

---

## 3. Liskov Substitution Principle (LSP)

### Subtypes must be substitutable for their base types

```python
# ❌ BAD: Violates LSP
class Bird:
    """Base bird class"""

    def fly(self):
        """All birds can fly"""
        return "Flying..."

class Sparrow(Bird):
    """Sparrow - can fly"""
    pass

class Penguin(Bird):
    """Penguin - CAN'T fly! Violates LSP"""

    def fly(self):
        """Penguins can't fly"""
        raise NotImplementedError("Penguins can't fly!")

def make_bird_fly(bird: Bird):
    """Make any bird fly"""
    return bird.fly()

# Works fine
sparrow = Sparrow()
make_bird_fly(sparrow)  # "Flying..."

# Breaks! Penguin is not substitutable for Bird
penguin = Penguin()
make_bird_fly(penguin)  # Raises exception! Violates LSP

"""
Problem:
- Can't substitute Penguin for Bird
- Breaks polymorphism
- Caller needs to check type before calling
"""

# ✅ GOOD: Proper abstraction
class Bird(ABC):
    """Base bird class"""

    @abstractmethod
    def move(self):
        """All birds can move"""
        pass

class FlyingBird(Bird):
    """Birds that can fly"""

    def move(self):
        """Flying birds move by flying"""
        return self.fly()

    def fly(self):
        """Fly"""
        return "Flying..."

class FlightlessBird(Bird):
    """Birds that can't fly"""

    def move(self):
        """Flightless birds move by walking"""
        return self.walk()

    def walk(self):
        """Walk"""
        return "Walking..."

class Sparrow(FlyingBird):
    """Sparrow - flying bird"""
    pass

class Penguin(FlightlessBird):
    """Penguin - flightless bird"""

    def swim(self):
        """Penguins can swim"""
        return "Swimming..."

def make_bird_move(bird: Bird):
    """Make any bird move - works for all birds!"""
    return bird.move()

# Works for all birds
sparrow = Sparrow()
penguin = Penguin()

make_bird_move(sparrow)   # "Flying..."
make_bird_move(penguin)   # "Walking..."

"""
Benefits:
✅ Penguin is substitutable for Bird
✅ No unexpected exceptions
✅ Proper abstraction (move, not fly)
✅ Maintains polymorphism
"""

# Another example: Rectangle vs Square
class Rectangle:
    """Rectangle"""

    def __init__(self, width: int, height: int):
        self._width = width
        self._height = height

    @property
    def width(self):
        return self._width

    @width.setter
    def width(self, value):
        self._width = value

    @property
    def height(self):
        return self._height

    @height.setter
    def height(self, value):
        self._height = value

    def area(self):
        return self._width * self._height

class Square(Rectangle):
    """Square - IS-A Rectangle? Mathematically yes, but...

    Violates LSP because setting width/height independently breaks square invariant.
    """

    @Rectangle.width.setter
    def width(self, value):
        self._width = value
        self._height = value  # Keep square property

    @Rectangle.height.setter
    def height(self, value):
        self._width = value  # Keep square property
        self._height = value

def test_rectangle(rect: Rectangle):
    """Test rectangle properties"""
    rect.width = 5
    rect.height = 10
    assert rect.area() == 50  # Fails for Square! Violates LSP

# Better: Use composition instead of inheritance
class Shape(ABC):
    """Base shape"""

    @abstractmethod
    def area(self):
        pass

class Rectangle(Shape):
    """Rectangle"""

    def __init__(self, width: int, height: int):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

class Square(Shape):
    """Square - not inheriting from Rectangle"""

    def __init__(self, side: int):
        self.side = side

    def area(self):
        return self.side * self.side
```

---

## 4. Interface Segregation Principle (ISP)

### Clients should not depend on interfaces they don't use

```python
# ❌ BAD: Fat interface
class Worker(ABC):
    """Worker interface - too broad"""

    @abstractmethod
    def work(self):
        """Do work"""
        pass

    @abstractmethod
    def eat(self):
        """Eat lunch"""
        pass

    @abstractmethod
    def sleep(self):
        """Sleep"""
        pass

class HumanWorker(Worker):
    """Human worker - uses all methods"""

    def work(self):
        print("Human working...")

    def eat(self):
        print("Human eating...")

    def sleep(self):
        print("Human sleeping...")

class RobotWorker(Worker):
    """Robot worker - forced to implement unnecessary methods!"""

    def work(self):
        print("Robot working...")

    def eat(self):
        """Robots don't eat!"""
        pass  # Empty implementation - violates ISP

    def sleep(self):
        """Robots don't sleep!"""
        pass  # Empty implementation - violates ISP

"""
Problems:
- RobotWorker forced to implement eat() and sleep()
- Empty implementations indicate bad design
- Violates ISP - depends on methods it doesn't use
"""

# ✅ GOOD: Segregated interfaces
class Workable(ABC):
    """Can work"""

    @abstractmethod
    def work(self):
        pass

class Eatable(ABC):
    """Can eat"""

    @abstractmethod
    def eat(self):
        pass

class Sleepable(ABC):
    """Can sleep"""

    @abstractmethod
    def sleep(self):
        pass

class HumanWorker(Workable, Eatable, Sleepable):
    """Human worker - implements all interfaces it needs"""

    def work(self):
        print("Human working...")

    def eat(self):
        print("Human eating...")

    def sleep(self):
        print("Human sleeping...")

class RobotWorker(Workable):
    """Robot worker - only implements what it needs!"""

    def work(self):
        print("Robot working...")

# Usage
def manage_work(worker: Workable):
    """Manage any worker - only needs work()"""
    worker.work()

def manage_break(worker: Eatable):
    """Manage break - only needs eat()"""
    worker.eat()

human = HumanWorker()
robot = RobotWorker()

manage_work(human)   # Works
manage_work(robot)   # Works

manage_break(human)  # Works
# manage_break(robot)  # Type error - robot doesn't implement Eatable

"""
Benefits:
✅ Clients only depend on what they need
✅ No empty implementations
✅ More flexible - can mix and match
✅ Easier to maintain
"""
```

---

## 5. Dependency Inversion Principle (DIP)

### Depend on abstractions, not concretions

```python
# ❌ BAD: High-level depends on low-level
class MySQLDatabase:
    """Low-level module - concrete implementation"""

    def connect(self):
        print("Connected to MySQL")

    def save(self, data: dict):
        print(f"Saving to MySQL: {data}")

class UserService:
    """High-level module - depends on concrete MySQLDatabase"""

    def __init__(self):
        self.db = MySQLDatabase()  # Tight coupling!

    def create_user(self, user: dict):
        self.db.connect()
        self.db.save(user)

"""
Problems:
- UserService tightly coupled to MySQLDatabase
- Can't switch to PostgreSQL without modifying UserService
- Hard to test (need real MySQL connection)
- Violates DIP - depends on concrete class
"""

# ✅ GOOD: Depend on abstraction
class Database(ABC):
    """Abstract database - abstraction"""

    @abstractmethod
    def connect(self):
        pass

    @abstractmethod
    def save(self, data: dict):
        pass

class MySQLDatabase(Database):
    """MySQL implementation"""

    def connect(self):
        print("Connected to MySQL")

    def save(self, data: dict):
        print(f"Saving to MySQL: {data}")

class PostgreSQLDatabase(Database):
    """PostgreSQL implementation"""

    def connect(self):
        print("Connected to PostgreSQL")

    def save(self, data: dict):
        print(f"Saving to PostgreSQL: {data}")

class MockDatabase(Database):
    """Mock database for testing"""

    def __init__(self):
        self.data = []

    def connect(self):
        pass  # No-op

    def save(self, data: dict):
        self.data.append(data)

class UserService:
    """High-level module - depends on abstraction!"""

    def __init__(self, database: Database):
        """Inject database dependency"""
        self.db = database

    def create_user(self, user: dict):
        self.db.connect()
        self.db.save(user)

# Usage - flexible!
mysql_db = MySQLDatabase()
service_mysql = UserService(mysql_db)
service_mysql.create_user({'name': 'Alice'})

postgres_db = PostgreSQLDatabase()
service_postgres = UserService(postgres_db)
service_postgres.create_user({'name': 'Bob'})

# Testing - easy!
mock_db = MockDatabase()
service_test = UserService(mock_db)
service_test.create_user({'name': 'Charlie'})
assert len(mock_db.data) == 1

"""
Benefits:
✅ Loose coupling - easy to change database
✅ Easy to test - inject mock
✅ Follows DIP - depends on abstraction
✅ Open for extension
"""

# Dependency Injection in FastAPI
from fastapi import Depends

class UserRepository(ABC):
    """Abstract repository"""

    @abstractmethod
    def create(self, user: dict) -> dict:
        pass

class SQLUserRepository(UserRepository):
    """SQL implementation"""

    def create(self, user: dict) -> dict:
        # Save to SQL database
        return user

# Dependency injection
def get_user_repository() -> UserRepository:
    """Get user repository - can return different implementations"""
    return SQLUserRepository()

@app.post("/users/")
async def create_user(
    user: dict,
    repo: UserRepository = Depends(get_user_repository)
):
    """Create user - depends on abstraction!"""
    return repo.create(user)
```

---

## Best Practices

### ✅ Do's:

1. **Apply gradually** - Don't over-engineer simple code
2. **Favor composition** - Over inheritance
3. **Use abstractions** - Interfaces/abstract classes
4. **Inject dependencies** - Don't create in constructor
5. **Keep classes small** - Single responsibility
6. **Write tests** - SOLID makes testing easier

### ❌ Don'ts:

1. **Don't apply blindly** - Context matters
2. **Don't over-abstract** - YAGNI (You Aren't Gonna Need It)
3. **Don't ignore simplicity** - SOLID for complex systems
4. **Don't create deep hierarchies** - Favor flat composition

---

## Interview Questions

### Q1: What's Single Responsibility Principle?

**Answer**:

- **Definition**: Class should have one reason to change
- **Example**: UserRepository (persistence), EmailService (email)
- **Not**: User class handling DB, email, reports
- **Benefit**: Easy to maintain, test, understand
- **How to identify**: If class has "and" in description, likely violates SRP
  One responsibility = one reason to change.

### Q2: Explain Open/Closed Principle with example.

**Answer**:

- **Definition**: Open for extension, closed for modification
- **Example**: PaymentMethod interface, add new payment types without changing processor
- **Not**: Adding if/elif for each new payment type
- **Benefit**: Add features without breaking existing code
- **Implementation**: Abstract classes, interfaces, polymorphism
  Extend behavior, don't modify.

### Q3: What's Liskov Substitution Principle?

**Answer**:

- **Definition**: Subtype must be substitutable for base type
- **Example**: If Bird has fly(), Penguin can't inherit (can't fly)
- **Fix**: Separate FlyingBird and FlightlessBird
- **Test**: Can you replace base with subtype without breaking?
- **Benefit**: Maintains polymorphism, no surprises
  Don't break expectations!

### Q4: Why Interface Segregation?

**Answer**:

- **Problem**: Fat interfaces force empty implementations
- **Example**: Worker with work/eat/sleep, Robot doesn't eat/sleep
- **Fix**: Separate Workable, Eatable, Sleepable interfaces
- **Benefit**: Classes only implement what they need
- **Python**: Use ABCs with specific methods
  Many small interfaces > one large.

### Q5: What's Dependency Inversion?

**Answer**:

- **Definition**: Depend on abstractions, not concrete classes
- **Example**: UserService depends on Database interface, not MySQLDatabase
- **Not**: Creating MySQLDatabase directly in UserService
- **Implementation**: Dependency injection, abstract base classes
- **Benefit**: Loose coupling, easy to test, swap implementations
  Invert the dependency direction!

---

## Summary

SOLID principles:

- **SRP**: One class, one responsibility
- **OCP**: Extend, don't modify
- **LSP**: Subtypes are substitutable
- **ISP**: Many small interfaces
- **DIP**: Depend on abstractions

Clean, maintainable, flexible code! 🎯
