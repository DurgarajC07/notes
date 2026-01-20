# 🔨 Refactoring Patterns and Techniques

## Overview

Systematic approaches to improve code structure, readability, and maintainability without changing external behavior.

---

## Why Refactor?

```python
"""
Refactoring Goals:
1. Improve readability
2. Reduce complexity
3. Remove duplication
4. Enhance testability
5. Prepare for features

When to Refactor:
✅ Before adding features
✅ During code review
✅ When fixing bugs
✅ When code is hard to understand
✅ When tests are difficult to write

When NOT to Refactor:
❌ Without tests
❌ On working code before deadline
❌ Instead of rewriting bad design
❌ Without understanding purpose
"""
```

---

## Code Smells and Solutions

### 1. Long Method

```python
# ❌ BAD: Long method doing too much
def process_order(order_data):
    """Process order - 100+ lines"""
    # Validate data
    if not order_data.get('items'):
        raise ValueError("No items")
    if not order_data.get('customer_id'):
        raise ValueError("No customer")
    # ... 20 more validations

    # Calculate totals
    subtotal = 0
    for item in order_data['items']:
        subtotal += item['price'] * item['quantity']
    tax = subtotal * 0.1
    shipping = 10 if subtotal < 50 else 0
    total = subtotal + tax + shipping

    # Check inventory
    for item in order_data['items']:
        product = db.get_product(item['product_id'])
        if product.stock < item['quantity']:
            raise ValueError(f"Insufficient stock for {product.name}")

    # Process payment
    payment = {
        'amount': total,
        'customer_id': order_data['customer_id'],
        'method': order_data['payment_method']
    }
    payment_result = payment_gateway.charge(payment)
    if not payment_result['success']:
        raise PaymentError(payment_result['error'])

    # Update inventory
    # Send email
    # Create invoice
    # ... more logic

# ✅ GOOD: Extract methods
class OrderProcessor:
    """Order processing with extracted methods"""

    def process_order(self, order_data: dict) -> dict:
        """
        Process order - delegating to smaller methods

        Benefits:
        - Each method has single responsibility
        - Easy to test individually
        - Clear flow of operations
        """
        self._validate_order_data(order_data)

        totals = self._calculate_totals(order_data)

        self._check_inventory(order_data['items'])

        payment_result = self._process_payment(
            order_data['customer_id'],
            totals['total'],
            order_data['payment_method']
        )

        self._update_inventory(order_data['items'])

        order = self._create_order(order_data, totals, payment_result)

        self._send_confirmation_email(order)

        return order

    def _validate_order_data(self, data: dict):
        """Validate order data"""
        if not data.get('items'):
            raise ValueError("Order must have items")

        if not data.get('customer_id'):
            raise ValueError("Order must have customer")

        # ... focused validation logic

    def _calculate_totals(self, data: dict) -> dict:
        """Calculate order totals"""
        subtotal = sum(
            item['price'] * item['quantity']
            for item in data['items']
        )

        tax = subtotal * 0.1
        shipping = 0 if subtotal >= 50 else 10
        total = subtotal + tax + shipping

        return {
            'subtotal': subtotal,
            'tax': tax,
            'shipping': shipping,
            'total': total
        }

    def _check_inventory(self, items: list):
        """Check inventory availability"""
        for item in items:
            product = db.get_product(item['product_id'])

            if product.stock < item['quantity']:
                raise InsufficientStockError(
                    f"Insufficient stock for {product.name}"
                )

    def _process_payment(self, customer_id: int, amount: float, method: str):
        """Process payment"""
        # Focused payment logic
        pass

    def _update_inventory(self, items: list):
        """Update inventory after order"""
        # Focused inventory logic
        pass

    def _create_order(self, data: dict, totals: dict, payment: dict):
        """Create order record"""
        # Focused order creation
        pass

    def _send_confirmation_email(self, order: dict):
        """Send confirmation email"""
        # Focused email logic
        pass
```

### 2. Duplicate Code

```python
# ❌ BAD: Duplicated code
def get_active_users():
    """Get active users"""
    users = User.objects.filter(is_active=True)
    results = []
    for user in users:
        results.append({
            'id': user.id,
            'name': user.name,
            'email': user.email,
            'created_at': user.created_at.isoformat()
        })
    return results

def get_premium_users():
    """Get premium users"""
    users = User.objects.filter(is_premium=True)
    results = []
    for user in users:
        results.append({
            'id': user.id,
            'name': user.name,
            'email': user.email,
            'created_at': user.created_at.isoformat()
        })
    return results

# ✅ GOOD: Extract common logic
def serialize_user(user: User) -> dict:
    """Serialize user to dict"""
    return {
        'id': user.id,
        'name': user.name,
        'email': user.email,
        'created_at': user.created_at.isoformat()
    }

def get_users_by_filter(**filters) -> list:
    """Get users with filters"""
    users = User.objects.filter(**filters)
    return [serialize_user(user) for user in users]

def get_active_users():
    """Get active users"""
    return get_users_by_filter(is_active=True)

def get_premium_users():
    """Get premium users"""
    return get_users_by_filter(is_premium=True)
```

### 3. Large Class (God Object)

```python
# ❌ BAD: God class doing everything
class UserManager:
    """User manager - too many responsibilities"""

    def create_user(self, data):
        """Create user"""
        pass

    def authenticate(self, username, password):
        """Authenticate user"""
        pass

    def send_welcome_email(self, user):
        """Send email"""
        pass

    def process_payment(self, user, amount):
        """Process payment"""
        pass

    def generate_report(self, user):
        """Generate report"""
        pass

    # ... 20 more methods

# ✅ GOOD: Separate responsibilities
class UserRepository:
    """Handle user data persistence"""

    def create(self, data: dict) -> User:
        """Create user"""
        return User.objects.create(**data)

    def get_by_id(self, user_id: int) -> User:
        """Get user by ID"""
        return User.objects.get(id=user_id)

    def update(self, user_id: int, data: dict) -> User:
        """Update user"""
        user = self.get_by_id(user_id)
        for key, value in data.items():
            setattr(user, key, value)
        user.save()
        return user

class AuthenticationService:
    """Handle authentication"""

    def authenticate(self, username: str, password: str) -> User:
        """Authenticate user"""
        user = User.objects.get(username=username)

        if not self._verify_password(password, user.password):
            raise AuthenticationError("Invalid credentials")

        return user

    def _verify_password(self, plain: str, hashed: str) -> bool:
        """Verify password"""
        from argon2 import PasswordHasher
        ph = PasswordHasher()

        try:
            return ph.verify(hashed, plain)
        except:
            return False

class EmailService:
    """Handle email operations"""

    def send_welcome_email(self, user: User):
        """Send welcome email"""
        subject = "Welcome!"
        body = f"Welcome {user.name}!"
        self._send_email(user.email, subject, body)

    def _send_email(self, to: str, subject: str, body: str):
        """Send email via provider"""
        pass

class PaymentService:
    """Handle payments"""

    def process_payment(self, user: User, amount: float):
        """Process payment"""
        pass
```

### 4. Long Parameter List

```python
# ❌ BAD: Too many parameters
def create_user(
    username: str,
    email: str,
    password: str,
    first_name: str,
    last_name: str,
    age: int,
    country: str,
    city: str,
    address: str,
    phone: str
):
    """Create user - hard to call"""
    pass

# ✅ GOOD: Use dataclass or dict
from dataclasses import dataclass

@dataclass
class UserData:
    """User creation data"""
    username: str
    email: str
    password: str
    first_name: str
    last_name: str
    age: int
    country: str
    city: str
    address: str
    phone: str

def create_user(user_data: UserData) -> User:
    """Create user - clean signature"""
    return User.objects.create(**user_data.__dict__)

# Or with Pydantic
from pydantic import BaseModel, EmailStr

class UserCreateSchema(BaseModel):
    """User creation schema"""
    username: str
    email: EmailStr
    password: str
    first_name: str
    last_name: str
    age: int
    country: str
    city: str
    address: str
    phone: str

def create_user_validated(user_data: UserCreateSchema) -> User:
    """Create user with validation"""
    return User.objects.create(**user_data.dict())
```

### 5. Primitive Obsession

```python
# ❌ BAD: Using primitives everywhere
def send_email(to: str, subject: str, body: str):
    """Send email - no validation"""
    if '@' not in to:
        raise ValueError("Invalid email")

    # ... send logic

def calculate_price(amount: float, currency: str) -> float:
    """Calculate price"""
    if currency not in ['USD', 'EUR', 'GBP']:
        raise ValueError("Invalid currency")

    # ... calculation

# ✅ GOOD: Create value objects
from dataclasses import dataclass
import re

@dataclass(frozen=True)
class Email:
    """Email value object"""
    address: str

    def __post_init__(self):
        """Validate email"""
        pattern = r'^[\w\.-]+@[\w\.-]+\.\w+$'

        if not re.match(pattern, self.address):
            raise ValueError(f"Invalid email: {self.address}")

    def __str__(self):
        return self.address

@dataclass(frozen=True)
class Money:
    """Money value object"""
    amount: float
    currency: str

    VALID_CURRENCIES = {'USD', 'EUR', 'GBP'}

    def __post_init__(self):
        """Validate money"""
        if self.currency not in self.VALID_CURRENCIES:
            raise ValueError(f"Invalid currency: {self.currency}")

        if self.amount < 0:
            raise ValueError("Amount cannot be negative")

    def __add__(self, other):
        """Add money"""
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")

        return Money(self.amount + other.amount, self.currency)

    def __str__(self):
        return f"{self.currency} {self.amount:.2f}"

# Now use value objects
def send_email(to: Email, subject: str, body: str):
    """Send email - validated at type level"""
    # to.address is guaranteed to be valid
    pass

def calculate_total(items: list[Money]) -> Money:
    """Calculate total"""
    if not items:
        return Money(0, 'USD')

    total = items[0]
    for item in items[1:]:
        total = total + item  # Uses __add__, validates currency

    return total

# Usage
email = Email("user@example.com")  # Validates
price1 = Money(10.50, 'USD')
price2 = Money(5.25, 'USD')
total = price1 + price2  # Money(15.75, 'USD')
```

---

## Refactoring Techniques

### 1. Extract Method

```python
# Before: Complex method
def calculate_discount(order):
    total = sum(item.price * item.quantity for item in order.items)

    if order.customer.is_vip:
        if total > 1000:
            discount = total * 0.20
        else:
            discount = total * 0.15
    else:
        if total > 500:
            discount = total * 0.10
        else:
            discount = total * 0.05

    return discount

# After: Extracted methods
def calculate_discount(order):
    """Calculate discount based on customer and total"""
    total = _calculate_order_total(order)
    discount_rate = _get_discount_rate(order.customer, total)
    return total * discount_rate

def _calculate_order_total(order) -> float:
    """Calculate order total"""
    return sum(item.price * item.quantity for item in order.items)

def _get_discount_rate(customer, total: float) -> float:
    """Get discount rate based on customer and total"""
    if customer.is_vip:
        return 0.20 if total > 1000 else 0.15
    else:
        return 0.10 if total > 500 else 0.05
```

### 2. Replace Conditional with Polymorphism

```python
# Before: Type checking
class PaymentProcessor:
    """Process different payment types"""

    def process(self, payment_type: str, amount: float):
        """Process payment based on type"""
        if payment_type == 'credit_card':
            # Credit card logic
            print(f"Processing credit card payment: ${amount}")
            fee = amount * 0.03
            return amount + fee

        elif payment_type == 'paypal':
            # PayPal logic
            print(f"Processing PayPal payment: ${amount}")
            fee = amount * 0.02
            return amount + fee

        elif payment_type == 'crypto':
            # Crypto logic
            print(f"Processing crypto payment: ${amount}")
            fee = amount * 0.01
            return amount + fee

        else:
            raise ValueError(f"Unknown payment type: {payment_type}")

# After: Polymorphism
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    """Abstract payment method"""

    @abstractmethod
    def process(self, amount: float) -> float:
        """Process payment and return total with fees"""
        pass

    @abstractmethod
    def get_fee_rate(self) -> float:
        """Get fee rate"""
        pass

class CreditCardPayment(PaymentMethod):
    """Credit card payment"""

    def get_fee_rate(self) -> float:
        return 0.03

    def process(self, amount: float) -> float:
        """Process credit card payment"""
        print(f"Processing credit card payment: ${amount}")
        fee = amount * self.get_fee_rate()
        return amount + fee

class PayPalPayment(PaymentMethod):
    """PayPal payment"""

    def get_fee_rate(self) -> float:
        return 0.02

    def process(self, amount: float) -> float:
        """Process PayPal payment"""
        print(f"Processing PayPal payment: ${amount}")
        fee = amount * self.get_fee_rate()
        return amount + fee

class CryptoPayment(PaymentMethod):
    """Cryptocurrency payment"""

    def get_fee_rate(self) -> float:
        return 0.01

    def process(self, amount: float) -> float:
        """Process crypto payment"""
        print(f"Processing crypto payment: ${amount}")
        fee = amount * self.get_fee_rate()
        return amount + fee

# Factory for creating payment methods
class PaymentFactory:
    """Create payment method instances"""

    _methods = {
        'credit_card': CreditCardPayment,
        'paypal': PayPalPayment,
        'crypto': CryptoPayment,
    }

    @classmethod
    def create(cls, payment_type: str) -> PaymentMethod:
        """Create payment method"""
        method_class = cls._methods.get(payment_type)

        if not method_class:
            raise ValueError(f"Unknown payment type: {payment_type}")

        return method_class()

# Usage
payment_method = PaymentFactory.create('credit_card')
total = payment_method.process(100.0)
```

### 3. Introduce Parameter Object

```python
# Before: Many related parameters
def create_rectangle(x: float, y: float, width: float, height: float, color: str):
    """Create rectangle"""
    pass

def move_rectangle(x: float, y: float, dx: float, dy: float):
    """Move rectangle"""
    pass

# After: Parameter object
@dataclass
class Point:
    """2D point"""
    x: float
    y: float

    def move(self, dx: float, dy: float):
        """Move point"""
        return Point(self.x + dx, self.y + dy)

@dataclass
class Size:
    """2D size"""
    width: float
    height: float

@dataclass
class Rectangle:
    """Rectangle"""
    position: Point
    size: Size
    color: str

    def move(self, dx: float, dy: float):
        """Move rectangle"""
        self.position = self.position.move(dx, dy)

    def area(self) -> float:
        """Calculate area"""
        return self.size.width * self.size.height

# Usage
rect = Rectangle(
    position=Point(10, 20),
    size=Size(100, 50),
    color='red'
)

rect.move(5, 10)
```

---

## Best Practices

### ✅ Do's:

1. **Refactor in small steps** - One change at a time
2. **Run tests after each step** - Ensure nothing breaks
3. **Commit frequently** - Easy to revert if needed
4. **Have test coverage** - Tests verify behavior unchanged
5. **Use IDE refactoring tools** - Automated and safe
6. **Extract methods** - Break down complex logic
7. **Remove duplication** - DRY principle

### ❌ Don'ts:

1. **Don't refactor without tests** - How do you know it works?
2. **Don't change behavior** - Refactoring preserves functionality
3. **Don't refactor and add features** - Separate concerns
4. **Don't over-engineer** - Keep it simple
5. **Don't rename just for fun** - Have good reason

---

## Interview Questions

### Q1: What is refactoring?

**Answer**: Improving code structure without changing behavior:

- **Goal**: Better readability, maintainability, extensibility
- **Not**: Bug fixes, feature additions, optimization
- **Requires**: Tests to verify behavior unchanged
- **Process**: Small steps, test after each
  Essential for code quality and team productivity.

### Q2: When should you refactor?

**Answer**:

- **Before feature**: Clean code easier to extend
- **During review**: Improve code quality
- **Bug fix**: Clean up while fixing
- **Hard to test**: Refactor for testability
- **Not when**: No tests, tight deadline, unclear requirements
  Refactor continuously, not as separate phase.

### Q3: What are code smells?

**Answer**: Signs that code needs refactoring:

- **Long method**: Extract smaller methods
- **Large class**: Split responsibilities
- **Duplicate code**: Extract common logic
- **Long parameter list**: Use objects
- **Primitive obsession**: Create value objects
  Smells indicate design issues.

### Q4: How to refactor safely?

**Answer**:

- **Tests first**: Ensure behavior verified
- **Small steps**: One change at a time
- **Run tests**: After each change
- **Use IDE tools**: Automated refactoring
- **Version control**: Commit frequently, easy rollback
  Safety comes from tests and discipline.

### Q5: Extract method vs inline method?

**Answer**:

- **Extract**: Complex logic → separate method
- **Inline**: Trivial method → put back in caller
- **Extract when**: Improves readability, reusability
- **Inline when**: Method is trivial, not reused
- **Balance**: Don't over-extract or under-extract
  Use judgment based on code clarity.

---

## Summary

Refactoring essentials:

- **Code smells**: Long method, large class, duplication
- **Techniques**: Extract method, polymorphism, parameter object
- **Process**: Small steps, test after each
- **Safety**: Tests, version control, IDE tools
- **When**: Continuously, before features
- **Goal**: Maintainable, readable code

Clean code is refactored code! 🔨
