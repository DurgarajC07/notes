# SOLID Principles in Python - Production-Ready Design

## 📖 Concept Explanation

SOLID is an acronym for five design principles that make software designs more understandable, flexible, and maintainable. These principles are fundamental for building production-grade backend systems.

### The Five SOLID Principles

1. **S**ingle Responsibility Principle (SRP)
2. **O**pen/Closed Principle (OCP)
3. **L**iskov Substitution Principle (LSP)
4. **I**nterface Segregation Principle (ISP)
5. **D**ependency Inversion Principle (DIP)

## 🧠 Why It Matters in Real Projects

- **Maintainability**: Easier to understand and modify code
- **Testability**: Components can be tested in isolation
- **Scalability**: System can grow without breaking existing code
- **Team Collaboration**: Clear responsibilities and interfaces
- **Reduced Bugs**: Changes in one area don't break others
- **Framework Design**: Django, FastAPI follow SOLID principles

---

## 1️⃣ Single Responsibility Principle (SRP)

**Definition**: A class should have one, and only one, reason to change.

### ❌ Violation Example

```python
class User:
    """VIOLATION: Too many responsibilities"""

    def __init__(self, name, email):
        self.name = name
        self.email = email

    def save_to_database(self):
        """Database responsibility"""
        # Database code
        pass

    def send_email(self):
        """Email responsibility"""
        # Email code
        pass

    def generate_report(self):
        """Reporting responsibility"""
        # Report generation code
        pass
```

**Problems**:

- Hard to test
- Changes to email logic affect User class
- Database changes affect User class
- Violates separation of concerns

### ✅ Correct Example

```python
# Domain Model - Single responsibility: Represent user data
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

# Repository - Single responsibility: Data persistence
class UserRepository:
    def __init__(self, database):
        self.database = database

    def save(self, user):
        self.database.execute(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            (user.name, user.email)
        )

    def find_by_id(self, user_id):
        result = self.database.query(
            "SELECT * FROM users WHERE id = ?", (user_id,)
        )
        return User(**result) if result else None

# Email Service - Single responsibility: Send emails
class EmailService:
    def __init__(self, smtp_config):
        self.smtp_config = smtp_config

    def send_welcome_email(self, user):
        # Send email logic
        pass

# Report Generator - Single responsibility: Generate reports
class UserReportGenerator:
    def generate_pdf(self, user):
        # PDF generation logic
        pass
```

### Real-World Django Example

```python
# models.py - Data model only
class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

# services.py - Business logic
class OrderService:
    def __init__(self, payment_gateway, inventory):
        self.payment_gateway = payment_gateway
        self.inventory = inventory

    def create_order(self, user, items):
        order = Order.objects.create(user=user, status='pending')

        # Check inventory
        if not self.inventory.check_availability(items):
            raise OutOfStockError()

        # Process payment
        payment = self.payment_gateway.charge(order.total)

        if payment.success:
            order.status = 'paid'
            order.save()
            self.inventory.reserve_items(items)

        return order

# views.py - HTTP handling only
class OrderCreateView(APIView):
    def post(self, request):
        serializer = OrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        service = OrderService(PaymentGateway(), InventoryService())
        order = service.create_order(request.user, serializer.validated_data)

        return Response(OrderSerializer(order).data)
```

---

## 2️⃣ Open/Closed Principle (OCP)

**Definition**: Software entities should be open for extension but closed for modification.

### ❌ Violation Example

```python
class PaymentProcessor:
    """VIOLATION: Must modify for each new payment type"""

    def process_payment(self, amount, payment_type):
        if payment_type == "credit_card":
            # Credit card logic
            pass
        elif payment_type == "paypal":
            # PayPal logic
            pass
        elif payment_type == "crypto":
            # Crypto logic - MODIFIED CODE!
            pass
        # Every new payment type requires modification!
```

### ✅ Correct Example (Strategy Pattern)

```python
from abc import ABC, abstractmethod

# Abstract base - closed for modification
class PaymentMethod(ABC):
    @abstractmethod
    def process(self, amount):
        pass

# Concrete implementations - open for extension
class CreditCardPayment(PaymentMethod):
    def process(self, amount):
        print(f"Processing ${amount} via Credit Card")
        # Credit card API call
        return {"status": "success", "method": "credit_card"}

class PayPalPayment(PaymentMethod):
    def process(self, amount):
        print(f"Processing ${amount} via PayPal")
        # PayPal API call
        return {"status": "success", "method": "paypal"}

class CryptoPayment(PaymentMethod):
    def process(self, amount):
        print(f"Processing ${amount} via Cryptocurrency")
        # Crypto API call
        return {"status": "success", "method": "crypto"}

# Client code - unchanged when adding new payment methods
class PaymentProcessor:
    def __init__(self, payment_method: PaymentMethod):
        self.payment_method = payment_method

    def process(self, amount):
        return self.payment_method.process(amount)

# Usage - easily extensible
processor = PaymentProcessor(CreditCardPayment())
processor.process(100.00)

processor = PaymentProcessor(CryptoPayment())  # New method, no code changes!
processor.process(100.00)
```

### Django Example - Notification System

```python
# base.py
from abc import ABC, abstractmethod

class NotificationChannel(ABC):
    @abstractmethod
    def send(self, user, message):
        pass

# channels.py - Each channel is a new class (open for extension)
class EmailChannel(NotificationChannel):
    def send(self, user, message):
        send_mail(
            subject=message.subject,
            message=message.body,
            from_email='noreply@example.com',
            recipient_list=[user.email]
        )

class SMSChannel(NotificationChannel):
    def __init__(self, twilio_client):
        self.twilio_client = twilio_client

    def send(self, user, message):
        self.twilio_client.messages.create(
            to=user.phone,
            from_=settings.TWILIO_NUMBER,
            body=message.body
        )

class PushNotificationChannel(NotificationChannel):
    def send(self, user, message):
        firebase.send_notification(
            token=user.device_token,
            title=message.subject,
            body=message.body
        )

# service.py - Closed for modification
class NotificationService:
    def __init__(self, channels: List[NotificationChannel]):
        self.channels = channels

    def notify(self, user, message):
        for channel in self.channels:
            try:
                channel.send(user, message)
            except Exception as e:
                logger.error(f"Notification failed: {e}")

# Usage - Add new channels without modifying NotificationService
service = NotificationService([
    EmailChannel(),
    SMSChannel(twilio_client),
    PushNotificationChannel(),  # New channel added!
])
```

---

## 3️⃣ Liskov Substitution Principle (LSP)

**Definition**: Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.

### ❌ Violation Example

```python
class Bird:
    def fly(self):
        return "Flying"

class Penguin(Bird):
    def fly(self):
        raise Exception("Penguins can't fly!")  # VIOLATION!

def make_bird_fly(bird: Bird):
    return bird.fly()

# This breaks!
penguin = Penguin()
make_bird_fly(penguin)  # Exception! Violated LSP
```

### ✅ Correct Example

```python
from abc import ABC, abstractmethod

class Bird(ABC):
    @abstractmethod
    def move(self):
        pass

class FlyingBird(Bird):
    def move(self):
        return self.fly()

    def fly(self):
        return "Flying in the sky"

class Sparrow(FlyingBird):
    def fly(self):
        return "Sparrow flying"

class Penguin(Bird):
    def move(self):
        return self.swim()

    def swim(self):
        return "Penguin swimming"

def make_bird_move(bird: Bird):
    return bird.move()  # Works for all birds!

# Both work correctly
sparrow = Sparrow()
penguin = Penguin()

print(make_bird_move(sparrow))  # "Sparrow flying"
print(make_bird_move(penguin))  # "Penguin swimming"
```

### Django Example - User Authentication

```python
# ❌ VIOLATION
class User(AbstractBaseUser):
    def authenticate(self, password):
        return check_password(password, self.password)

class SocialUser(User):
    def authenticate(self, password):
        raise NotImplementedError("Social users don't use passwords")
        # VIOLATION: Can't substitute SocialUser for User

# ✅ CORRECT
class User(AbstractBaseUser):
    @abstractmethod
    def authenticate(self, credentials):
        pass

class PasswordUser(User):
    def authenticate(self, credentials):
        return check_password(credentials['password'], self.password)

class OAuth2User(User):
    def authenticate(self, credentials):
        return verify_oauth_token(credentials['token'])

class SocialUser(User):
    def authenticate(self, credentials):
        return verify_social_auth(
            credentials['provider'],
            credentials['token']
        )

# Works with any User type
def login_user(user: User, credentials):
    if user.authenticate(credentials):
        create_session(user)
        return True
    return False
```

### Rectangle-Square Problem (Classic LSP Example)

```python
# ❌ VIOLATION - Classic problem
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
        self._height = width  # Square constraint

    def set_height(self, height):
        self._width = height  # Square constraint
        self._height = height

def test_rectangle(rect: Rectangle):
    rect.set_width(5)
    rect.set_height(4)
    assert rect.area() == 20  # Fails for Square!

# ✅ CORRECT - Use composition
class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

class Square(Shape):
    def __init__(self, side):
        self.side = side

    def area(self):
        return self.side * self.side
```

---

## 4️⃣ Interface Segregation Principle (ISP)

**Definition**: No client should be forced to depend on methods it does not use.

### ❌ Violation Example

```python
from abc import ABC, abstractmethod

class Worker(ABC):
    """VIOLATION: Forces implementations to define unused methods"""

    @abstractmethod
    def work(self):
        pass

    @abstractmethod
    def eat(self):
        pass

    @abstractmethod
    def sleep(self):
        pass

class HumanWorker(Worker):
    def work(self):
        return "Working"

    def eat(self):
        return "Eating lunch"

    def sleep(self):
        return "Sleeping"

class RobotWorker(Worker):
    def work(self):
        return "Processing tasks"

    def eat(self):
        pass  # Robots don't eat! Forced to implement

    def sleep(self):
        pass  # Robots don't sleep! Forced to implement
```

### ✅ Correct Example

```python
from abc import ABC, abstractmethod

# Segregated interfaces
class Workable(ABC):
    @abstractmethod
    def work(self):
        pass

class Eatable(ABC):
    @abstractmethod
    def eat(self):
        pass

class Sleepable(ABC):
    @abstractmethod
    def sleep(self):
        pass

# Human implements all interfaces
class HumanWorker(Workable, Eatable, Sleepable):
    def work(self):
        return "Working"

    def eat(self):
        return "Eating lunch"

    def sleep(self):
        return "Sleeping"

# Robot only implements what it needs
class RobotWorker(Workable):
    def work(self):
        return "Processing tasks"

class SuperRobot(Workable, Sleepable):
    """Robot with power-saving mode"""
    def work(self):
        return "Processing"

    def sleep(self):
        return "Power-saving mode"
```

### Django REST Framework Example

```python
# ❌ VIOLATION
class APIView(ABC):
    @abstractmethod
    def get(self, request):
        pass

    @abstractmethod
    def post(self, request):
        pass

    @abstractmethod
    def put(self, request):
        pass

    @abstractmethod
    def delete(self, request):
        pass

class ReadOnlyView(APIView):
    def get(self, request):
        return Response(data)

    # Forced to implement unused methods
    def post(self, request):
        raise MethodNotAllowed("POST")

    def put(self, request):
        raise MethodNotAllowed("PUT")

    def delete(self, request):
        raise MethodNotAllowed("DELETE")

# ✅ CORRECT - Django's actual approach
class ListModelMixin:
    def list(self, request, *args, **kwargs):
        queryset = self.get_queryset()
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)

class CreateModelMixin:
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        return Response(serializer.data, status=201)

class RetrieveModelMixin:
    def retrieve(self, request, *args, **kwargs):
        instance = self.get_object()
        serializer = self.get_serializer(instance)
        return Response(serializer.data)

# Use only what you need
class UserListView(ListModelMixin, GenericAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer

class UserCreateView(CreateModelMixin, GenericAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer

class UserDetailView(RetrieveModelMixin, GenericAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

---

## 5️⃣ Dependency Inversion Principle (DIP)

**Definition**:

- High-level modules should not depend on low-level modules. Both should depend on abstractions.
- Abstractions should not depend on details. Details should depend on abstractions.

### ❌ Violation Example

```python
class MySQLDatabase:
    """Low-level module"""
    def connect(self):
        return "MySQL Connection"

    def execute(self, query):
        return f"Executing: {query}"

class UserRepository:
    """High-level module depending on concrete implementation"""
    def __init__(self):
        self.database = MySQLDatabase()  # VIOLATION: Direct dependency

    def get_user(self, user_id):
        return self.database.execute(f"SELECT * FROM users WHERE id={user_id}")

# Problems:
# - Can't switch databases without modifying UserRepository
# - Hard to test (can't mock database)
# - Tight coupling
```

### ✅ Correct Example

```python
from abc import ABC, abstractmethod

# Abstraction (Interface)
class Database(ABC):
    @abstractmethod
    def connect(self):
        pass

    @abstractmethod
    def execute(self, query):
        pass

# Low-level modules depend on abstraction
class MySQLDatabase(Database):
    def connect(self):
        return "MySQL Connection"

    def execute(self, query):
        return f"MySQL: {query}"

class PostgreSQLDatabase(Database):
    def connect(self):
        return "PostgreSQL Connection"

    def execute(self, query):
        return f"PostgreSQL: {query}"

class MongoDatabase(Database):
    def connect(self):
        return "MongoDB Connection"

    def execute(self, query):
        return f"MongoDB: {query}"

# High-level module depends on abstraction
class UserRepository:
    def __init__(self, database: Database):  # Dependency injection
        self.database = database

    def get_user(self, user_id):
        return self.database.execute(f"SELECT * FROM users WHERE id={user_id}")

    def save_user(self, user):
        return self.database.execute(f"INSERT INTO users VALUES (...)")

# Usage - Easy to switch implementations
mysql_repo = UserRepository(MySQLDatabase())
postgres_repo = UserRepository(PostgreSQLDatabase())
mongo_repo = UserRepository(MongoDatabase())

# Easy to test with mock
class MockDatabase(Database):
    def connect(self):
        return "Mock"

    def execute(self, query):
        return {"id": 1, "name": "Test User"}

test_repo = UserRepository(MockDatabase())
```

### FastAPI Example with Dependency Injection

```python
# abstractions.py
from abc import ABC, abstractmethod

class CacheService(ABC):
    @abstractmethod
    def get(self, key):
        pass

    @abstractmethod
    def set(self, key, value, ttl=None):
        pass

class EmailService(ABC):
    @abstractmethod
    def send(self, to, subject, body):
        pass

# implementations.py
class RedisCache(CacheService):
    def __init__(self, redis_client):
        self.redis = redis_client

    def get(self, key):
        return self.redis.get(key)

    def set(self, key, value, ttl=3600):
        self.redis.setex(key, ttl, value)

class SMTPEmailService(EmailService):
    def __init__(self, smtp_config):
        self.config = smtp_config

    def send(self, to, subject, body):
        # SMTP logic
        pass

# dependencies.py
from fastapi import Depends

def get_cache() -> CacheService:
    redis_client = redis.Redis(host='localhost')
    return RedisCache(redis_client)

def get_email_service() -> EmailService:
    return SMTPEmailService(smtp_config)

# routes.py
from fastapi import APIRouter, Depends

router = APIRouter()

@router.post("/users")
async def create_user(
    user_data: UserCreate,
    cache: CacheService = Depends(get_cache),  # Injected!
    email: EmailService = Depends(get_email_service)  # Injected!
):
    # Use abstractions, not concrete implementations
    user = User(**user_data.dict())

    # Cache user
    cache.set(f"user:{user.id}", user.json())

    # Send welcome email
    email.send(user.email, "Welcome!", "Welcome to our platform")

    return user

# Testing is easy - just provide mock dependencies
def test_create_user():
    async def mock_cache():
        return MockCache()

    async def mock_email():
        return MockEmailService()

    app.dependency_overrides[get_cache] = mock_cache
    app.dependency_overrides[get_email_service] = mock_email
```

### Django Example

```python
# services/base.py
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount, card):
        pass

class NotificationService(ABC):
    @abstractmethod
    def notify(self, user, message):
        pass

# services/implementations.py
class StripeGateway(PaymentGateway):
    def charge(self, amount, card):
        # Stripe API logic
        return stripe.Charge.create(amount=amount, source=card)

class EmailNotification(NotificationService):
    def notify(self, user, message):
        send_mail(message.subject, message.body, 'noreply@example.com', [user.email])

# views.py
class CheckoutView(APIView):
    def __init__(self, payment_gateway: PaymentGateway, notifier: NotificationService):
        self.payment_gateway = payment_gateway
        self.notifier = notifier

    def post(self, request):
        # Use injected dependencies
        result = self.payment_gateway.charge(
            amount=request.data['amount'],
            card=request.data['card']
        )

        if result.success:
            self.notifier.notify(
                request.user,
                Message("Payment Success", "Your payment was processed")
            )

        return Response(result)

# settings.py or dependency injection container
def get_checkout_view():
    return CheckoutView(
        payment_gateway=StripeGateway(),
        notifier=EmailNotification()
    )
```

---

## 🏗️ Real-World Combined Example

```python
"""
E-Commerce Order Processing System
Demonstrating all SOLID principles
"""

from abc import ABC, abstractmethod
from typing import List
from decimal import Decimal

# ============= Abstractions (DIP) =============
class PaymentProcessor(ABC):
    @abstractmethod
    def process_payment(self, amount: Decimal) -> bool:
        pass

class InventoryService(ABC):
    @abstractmethod
    def check_availability(self, product_id: str, quantity: int) -> bool:
        pass

    @abstractmethod
    def reserve_items(self, product_id: str, quantity: int) -> None:
        pass

class NotificationService(ABC):
    @abstractmethod
    def send_notification(self, user_id: str, message: str) -> None:
        pass

# ============= SRP - Single Responsibility =============
class Order:
    """Represents order data only"""
    def __init__(self, order_id: str, user_id: str, items: List[dict]):
        self.order_id = order_id
        self.user_id = user_id
        self.items = items
        self.status = "pending"
        self.total = self._calculate_total()

    def _calculate_total(self) -> Decimal:
        return sum(Decimal(str(item['price'])) * item['quantity'] for item in self.items)

class OrderRepository:
    """Responsible for data persistence only"""
    def __init__(self, database):
        self.database = database

    def save(self, order: Order) -> None:
        self.database.execute(
            "INSERT INTO orders (id, user_id, total, status) VALUES (?, ?, ?, ?)",
            (order.order_id, order.user_id, order.total, order.status)
        )

    def find_by_id(self, order_id: str) -> Order:
        # Fetch and construct Order
        pass

# ============= OCP - Open/Closed =============
# Can add new payment methods without modifying OrderService
class StripePaymentProcessor(PaymentProcessor):
    def process_payment(self, amount: Decimal) -> bool:
        # Stripe logic
        return True

class PayPalPaymentProcessor(PaymentProcessor):
    def process_payment(self, amount: Decimal) -> bool:
        # PayPal logic
        return True

# ============= ISP - Interface Segregation =============
# Segregated notification interfaces
class EmailNotification(NotificationService):
    def send_notification(self, user_id: str, message: str) -> None:
        # Send email
        pass

class SMSNotification(NotificationService):
    def send_notification(self, user_id: str, message: str) -> None:
        # Send SMS
        pass

# ============= DIP - Dependency Inversion =============
class OrderService:
    """
    High-level business logic
    Depends on abstractions, not concrete implementations
    """
    def __init__(
        self,
        repository: OrderRepository,
        payment: PaymentProcessor,
        inventory: InventoryService,
        notifier: NotificationService
    ):
        self.repository = repository
        self.payment = payment
        self.inventory = inventory
        self.notifier = notifier

    def create_order(self, user_id: str, items: List[dict]) -> Order:
        """SRP: This method only coordinates the order creation flow"""

        # Create order
        order = Order(
            order_id=self._generate_id(),
            user_id=user_id,
            items=items
        )

        # Check inventory
        for item in items:
            if not self.inventory.check_availability(item['product_id'], item['quantity']):
                raise ValueError(f"Product {item['product_id']} not available")

        # Process payment
        if not self.payment.process_payment(order.total):
            raise ValueError("Payment failed")

        # Reserve inventory
        for item in items:
            self.inventory.reserve_items(item['product_id'], item['quantity'])

        # Update order status
        order.status = "confirmed"

        # Save order
        self.repository.save(order)

        # Notify user
        self.notifier.send_notification(
            user_id,
            f"Order {order.order_id} confirmed"
        )

        return order

    def _generate_id(self) -> str:
        import uuid
        return str(uuid.uuid4())

# ============= Usage - Dependency Injection =============
# Easy to swap implementations
service = OrderService(
    repository=OrderRepository(database),
    payment=StripePaymentProcessor(),  # Can switch to PayPal easily
    inventory=WarehouseInventoryService(),
    notifier=EmailNotification()  # Can add SMS easily
)

# Easy to test with mocks
test_service = OrderService(
    repository=MockRepository(),
    payment=MockPaymentProcessor(),
    inventory=MockInventory(),
    notifier=MockNotifier()
)
```

---

## ❓ Interview Questions

### Q1: Explain SOLID principles with examples

**Answer**: See each principle section above with real-world examples.

### Q2: Why is Liskov Substitution Principle important?

**Answer**: LSP ensures that subtypes can be used interchangeably with their base types without breaking the application. Violations lead to unexpected behavior and require clients to check types before using objects, defeating the purpose of polymorphism.

### Q3: How does Dependency Inversion relate to Dependency Injection?

**Answer**:

- **Dependency Inversion**: Design principle (depend on abstractions)
- **Dependency Injection**: Implementation technique (inject dependencies)
  DI is one way to achieve DIP. Dependencies are provided (injected) from outside rather than created internally.

### Q4: When might you violate SOLID principles?

**Answer**: SOLID principles are guidelines, not laws. Valid reasons to violate:

- **Performance**: Sometimes abstraction has overhead
- **Simplicity**: Over-engineering simple code
- **Prototyping**: Rapid development phase
- **Framework Constraints**: Framework imposes structure

Always understand the trade-offs.

### Q5: How does FastAPI implement Dependency Inversion?

**Answer**: FastAPI's dependency injection system allows high-level route handlers to depend on abstractions. Dependencies are declared as function parameters and injected at runtime, making code testable and loosely coupled.

---

## 📚 Summary

SOLID principles create maintainable, scalable code:

1. **SRP**: One class, one responsibility
2. **OCP**: Extend behavior without modifying existing code
3. **LSP**: Subtypes must be substitutable for their base types
4. **ISP**: Many specific interfaces > One general interface
5. **DIP**: Depend on abstractions, inject dependencies

**Key Takeaways**:

- SOLID makes code testable, maintainable, scalable
- Use abstraction and dependency injection
- Django and FastAPI follow these principles
- Balance principles with pragmatism
- Essential for senior-level interviews
