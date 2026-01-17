# Behavioral Design Patterns - Object Interaction & Responsibility

## 📖 Concept Explanation

Behavioral patterns are concerned with algorithms and the assignment of responsibilities between objects. They characterize complex control flow and describe how objects communicate.

### Core Behavioral Patterns

1. **Observer**: Subscribe to and receive notifications
2. **Strategy**: Family of interchangeable algorithms
3. **Command**: Encapsulate request as object
4. **State**: Alter behavior when state changes
5. **Template Method**: Define algorithm skeleton
6. **Iterator**: Sequential access without exposing underlying representation
7. **Chain of Responsibility**: Pass request along chain of handlers

## 🧠 Why It Matters in Real Projects

### Production Use Cases

- **Observer**: Event systems, notifications, real-time updates
- **Strategy**: Payment methods, pricing calculations, sorting algorithms
- **Command**: Undo/redo, task queues, transaction systems
- **State**: Order workflows, authentication states, game states
- **Template Method**: Data processing pipelines, test frameworks

## 1️⃣ Observer Pattern

### Concept

Define one-to-many dependency so when one object changes state, all dependents are notified automatically.

### Basic Observer

```python
from abc import ABC, abstractmethod
from typing import List

class Observer(ABC):
    @abstractmethod
    def update(self, message: str):
        pass

class Subject:
    def __init__(self):
        self._observers: List[Observer] = []

    def attach(self, observer: Observer):
        self._observers.append(observer)

    def detach(self, observer: Observer):
        self._observers.remove(observer)

    def notify(self, message: str):
        for observer in self._observers:
            observer.update(message)

# Concrete observers
class EmailNotifier(Observer):
    def update(self, message: str):
        print(f"Email: {message}")

class SMSNotifier(Observer):
    def update(self, message: str):
        print(f"SMS: {message}")

class PushNotifier(Observer):
    def update(self, message: str):
        print(f"Push: {message}")

# Usage
subject = Subject()
subject.attach(EmailNotifier())
subject.attach(SMSNotifier())
subject.attach(PushNotifier())

subject.notify("New order placed!")
# Email: New order placed!
# SMS: New order placed!
# Push: New order placed!
```

### Django Signals as Observer

```python
# signals.py
from django.db.models.signals import post_save, pre_delete
from django.dispatch import receiver
from django.contrib.auth.models import User
from django.core.mail import send_mail

# Observer 1: Create user profile
@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        from profiles.models import UserProfile
        UserProfile.objects.create(user=instance)
        print(f"Profile created for {instance.username}")

# Observer 2: Send welcome email
@receiver(post_save, sender=User)
def send_welcome_email(sender, instance, created, **kwargs):
    if created:
        send_mail(
            subject='Welcome!',
            message=f'Welcome {instance.username}!',
            from_email='noreply@example.com',
            recipient_list=[instance.email]
        )

# Observer 3: Log user creation
@receiver(post_save, sender=User)
def log_user_creation(sender, instance, created, **kwargs):
    if created:
        import logging
        logging.info(f"New user created: {instance.username}")

# Observer 4: Cleanup on deletion
@receiver(pre_delete, sender=User)
def cleanup_user_data(sender, instance, **kwargs):
    # Delete related data
    instance.profile.delete()
    print(f"Cleaned up data for {instance.username}")

# apps.py - Register signals
from django.apps import AppConfig

class UsersConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'users'

    def ready(self):
        import users.signals  # Import signals
```

### Event Bus Pattern

```python
from typing import Callable, Dict, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Event:
    name: str
    data: Dict
    timestamp: datetime = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()

class EventBus:
    """Centralized event management"""

    def __init__(self):
        self._subscribers: Dict[str, List[Callable]] = {}

    def subscribe(self, event_name: str, handler: Callable):
        """Subscribe to event"""
        if event_name not in self._subscribers:
            self._subscribers[event_name] = []

        self._subscribers[event_name].append(handler)

    def unsubscribe(self, event_name: str, handler: Callable):
        """Unsubscribe from event"""
        if event_name in self._subscribers:
            self._subscribers[event_name].remove(handler)

    def publish(self, event: Event):
        """Publish event to all subscribers"""
        if event.name in self._subscribers:
            for handler in self._subscribers[event.name]:
                handler(event)

# Usage
event_bus = EventBus()

# Handlers
def send_email(event: Event):
    print(f"Email sent for: {event.name}")

def log_event(event: Event):
    print(f"Logged: {event.name} at {event.timestamp}")

def update_analytics(event: Event):
    print(f"Analytics updated for: {event.name}")

# Subscribe
event_bus.subscribe('user.registered', send_email)
event_bus.subscribe('user.registered', log_event)
event_bus.subscribe('user.registered', update_analytics)

# Publish
event = Event(name='user.registered', data={'user_id': 123})
event_bus.publish(event)
```

### FastAPI WebSocket Observer

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()

class ConnectionManager:
    """Manage WebSocket connections (Observer pattern)"""

    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        """Notify all observers"""
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            # Broadcast to all connected clients
            await manager.broadcast(f"Message: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)

# Trigger notifications
@app.post("/notify")
async def notify_all(message: str):
    await manager.broadcast(message)
    return {"status": "notified"}
```

## 2️⃣ Strategy Pattern

### Concept

Define family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets algorithm vary independently from clients.

### Payment Strategy

```python
from abc import ABC, abstractmethod
from decimal import Decimal
from typing import Dict

class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount: Decimal) -> Dict:
        pass

class CreditCardPayment(PaymentStrategy):
    def __init__(self, card_number: str, cvv: str):
        self.card_number = card_number
        self.cvv = cvv

    def pay(self, amount: Decimal) -> Dict:
        # Process credit card payment
        return {
            'method': 'credit_card',
            'amount': amount,
            'status': 'success',
            'transaction_id': 'CC-123'
        }

class PayPalPayment(PaymentStrategy):
    def __init__(self, email: str):
        self.email = email

    def pay(self, amount: Decimal) -> Dict:
        # Process PayPal payment
        return {
            'method': 'paypal',
            'amount': amount,
            'status': 'success',
            'transaction_id': 'PP-456'
        }

class CryptocurrencyPayment(PaymentStrategy):
    def __init__(self, wallet_address: str):
        self.wallet_address = wallet_address

    def pay(self, amount: Decimal) -> Dict:
        # Process crypto payment
        return {
            'method': 'cryptocurrency',
            'amount': amount,
            'status': 'pending',
            'transaction_id': 'BTC-789'
        }

# Context
class PaymentProcessor:
    def __init__(self, strategy: PaymentStrategy):
        self.strategy = strategy

    def set_strategy(self, strategy: PaymentStrategy):
        self.strategy = strategy

    def process_payment(self, amount: Decimal) -> Dict:
        return self.strategy.pay(amount)

# Usage
amount = Decimal('99.99')

# Pay with credit card
processor = PaymentProcessor(CreditCardPayment('4111-1111-1111-1111', '123'))
result = processor.process_payment(amount)
print(result)

# Switch to PayPal
processor.set_strategy(PayPalPayment('user@example.com'))
result = processor.process_payment(amount)
print(result)
```

### Pricing Strategy

```python
from abc import ABC, abstractmethod
from decimal import Decimal

class PricingStrategy(ABC):
    @abstractmethod
    def calculate_price(self, base_price: Decimal) -> Decimal:
        pass

class RegularPricing(PricingStrategy):
    def calculate_price(self, base_price: Decimal) -> Decimal:
        return base_price

class DiscountPricing(PricingStrategy):
    def __init__(self, discount_percent: Decimal):
        self.discount_percent = discount_percent

    def calculate_price(self, base_price: Decimal) -> Decimal:
        discount = base_price * (self.discount_percent / 100)
        return base_price - discount

class BulkPricing(PricingStrategy):
    def __init__(self, quantity: int):
        self.quantity = quantity

    def calculate_price(self, base_price: Decimal) -> Decimal:
        if self.quantity >= 10:
            return base_price * Decimal('0.8')  # 20% off
        elif self.quantity >= 5:
            return base_price * Decimal('0.9')  # 10% off
        return base_price

class SeasonalPricing(PricingStrategy):
    def __init__(self, season: str):
        self.season = season

    def calculate_price(self, base_price: Decimal) -> Decimal:
        multipliers = {
            'summer': Decimal('1.2'),  # 20% increase
            'winter': Decimal('0.85'),  # 15% discount
            'holiday': Decimal('1.5')   # 50% increase
        }
        return base_price * multipliers.get(self.season, Decimal('1.0'))

# Product with dynamic pricing
class Product:
    def __init__(self, name: str, base_price: Decimal):
        self.name = name
        self.base_price = base_price
        self.pricing_strategy = RegularPricing()

    def set_pricing_strategy(self, strategy: PricingStrategy):
        self.pricing_strategy = strategy

    def get_price(self) -> Decimal:
        return self.pricing_strategy.calculate_price(self.base_price)

# Usage
product = Product("Widget", Decimal('100.00'))

print(product.get_price())  # $100.00

# Apply discount
product.set_pricing_strategy(DiscountPricing(Decimal('15')))
print(product.get_price())  # $85.00

# Apply bulk pricing
product.set_pricing_strategy(BulkPricing(quantity=10))
print(product.get_price())  # $80.00
```

### Django View Strategy

```python
# strategies/authentication.py
from abc import ABC, abstractmethod
from django.http import HttpRequest

class AuthenticationStrategy(ABC):
    @abstractmethod
    def authenticate(self, request: HttpRequest):
        pass

class SessionAuthentication(AuthenticationStrategy):
    def authenticate(self, request: HttpRequest):
        return request.user if request.user.is_authenticated else None

class TokenAuthentication(AuthenticationStrategy):
    def authenticate(self, request: HttpRequest):
        token = request.headers.get('Authorization')
        if token:
            # Validate token and return user
            return self._get_user_from_token(token)
        return None

    def _get_user_from_token(self, token: str):
        # JWT validation
        return {'id': 1, 'username': 'john'}

class OAuth2Authentication(AuthenticationStrategy):
    def authenticate(self, request: HttpRequest):
        # OAuth2 flow
        return None

# views.py
from django.views import View
from django.http import JsonResponse

class BaseAPIView(View):
    authentication_strategy = SessionAuthentication()

    def dispatch(self, request, *args, **kwargs):
        # Use strategy
        user = self.authentication_strategy.authenticate(request)

        if not user:
            return JsonResponse({'error': 'Unauthorized'}, status=401)

        request.auth_user = user
        return super().dispatch(request, *args, **kwargs)

class TokenAPIView(BaseAPIView):
    authentication_strategy = TokenAuthentication()

class OAuth2APIView(BaseAPIView):
    authentication_strategy = OAuth2Authentication()
```

## 3️⃣ Command Pattern

### Concept

Encapsulate request as an object, allowing parameterization of clients with queues, requests, and operations.

### Basic Command

```python
from abc import ABC, abstractmethod

class Command(ABC):
    @abstractmethod
    def execute(self):
        pass

    @abstractmethod
    def undo(self):
        pass

# Receiver
class TextEditor:
    def __init__(self):
        self.content = ""

    def write(self, text: str):
        self.content += text

    def delete(self, length: int):
        self.content = self.content[:-length]

    def read(self):
        return self.content

# Concrete commands
class WriteCommand(Command):
    def __init__(self, editor: TextEditor, text: str):
        self.editor = editor
        self.text = text

    def execute(self):
        self.editor.write(self.text)

    def undo(self):
        self.editor.delete(len(self.text))

class DeleteCommand(Command):
    def __init__(self, editor: TextEditor, length: int):
        self.editor = editor
        self.length = length
        self.deleted_text = ""

    def execute(self):
        self.deleted_text = self.editor.content[-self.length:]
        self.editor.delete(self.length)

    def undo(self):
        self.editor.write(self.deleted_text)

# Invoker
class CommandManager:
    def __init__(self):
        self.history = []
        self.redo_stack = []

    def execute(self, command: Command):
        command.execute()
        self.history.append(command)
        self.redo_stack.clear()

    def undo(self):
        if self.history:
            command = self.history.pop()
            command.undo()
            self.redo_stack.append(command)

    def redo(self):
        if self.redo_stack:
            command = self.redo_stack.pop()
            command.execute()
            self.history.append(command)

# Usage
editor = TextEditor()
manager = CommandManager()

manager.execute(WriteCommand(editor, "Hello "))
manager.execute(WriteCommand(editor, "World"))
print(editor.read())  # "Hello World"

manager.undo()
print(editor.read())  # "Hello "

manager.redo()
print(editor.read())  # "Hello World"
```

### Task Queue Command

```python
from abc import ABC, abstractmethod
from typing import List
from datetime import datetime
import time

class Task(ABC):
    def __init__(self):
        self.created_at = datetime.now()
        self.executed_at = None

    @abstractmethod
    def execute(self):
        pass

    def mark_executed(self):
        self.executed_at = datetime.now()

class SendEmailTask(Task):
    def __init__(self, to: str, subject: str, body: str):
        super().__init__()
        self.to = to
        self.subject = subject
        self.body = body

    def execute(self):
        print(f"Sending email to {self.to}: {self.subject}")
        time.sleep(0.5)  # Simulate email sending
        self.mark_executed()

class ProcessImageTask(Task):
    def __init__(self, image_path: str):
        super().__init__()
        self.image_path = image_path

    def execute(self):
        print(f"Processing image: {self.image_path}")
        time.sleep(1)  # Simulate image processing
        self.mark_executed()

class GenerateReportTask(Task):
    def __init__(self, report_type: str):
        super().__init__()
        self.report_type = report_type

    def execute(self):
        print(f"Generating {self.report_type} report")
        time.sleep(0.7)
        self.mark_executed()

# Task Queue
class TaskQueue:
    def __init__(self):
        self.queue: List[Task] = []
        self.completed: List[Task] = []

    def add_task(self, task: Task):
        self.queue.append(task)

    def process_all(self):
        while self.queue:
            task = self.queue.pop(0)
            task.execute()
            self.completed.append(task)

    def get_stats(self):
        return {
            'pending': len(self.queue),
            'completed': len(self.completed)
        }

# Usage
queue = TaskQueue()

queue.add_task(SendEmailTask('user@example.com', 'Welcome', 'Welcome to our service'))
queue.add_task(ProcessImageTask('/uploads/image.jpg'))
queue.add_task(GenerateReportTask('monthly'))

queue.process_all()
print(queue.get_stats())
```

### Django Background Task Command (with Celery)

```python
# tasks.py
from celery import shared_task
from abc import ABC, abstractmethod

class BackgroundTask(ABC):
    @abstractmethod
    def run(self):
        pass

class SendWelcomeEmailTask(BackgroundTask):
    def __init__(self, user_id: int):
        self.user_id = user_id

    def run(self):
        from django.contrib.auth.models import User
        from django.core.mail import send_mail

        user = User.objects.get(id=self.user_id)
        send_mail(
            subject='Welcome!',
            message=f'Welcome {user.username}!',
            from_email='noreply@example.com',
            recipient_list=[user.email]
        )

@shared_task
def execute_task(task_class_name: str, **kwargs):
    """Execute any task by class name"""
    tasks = {
        'SendWelcomeEmailTask': SendWelcomeEmailTask
    }

    task_class = tasks.get(task_class_name)
    if task_class:
        task = task_class(**kwargs)
        task.run()

# views.py
from django.views import View
from django.http import JsonResponse

class RegisterView(View):
    def post(self, request):
        # Create user...
        user = User.objects.create_user(...)

        # Queue background task
        execute_task.delay('SendWelcomeEmailTask', user_id=user.id)

        return JsonResponse({'status': 'User created'})
```

## 4️⃣ State Pattern

### Concept

Allow object to alter its behavior when internal state changes. Object appears to change its class.

### Order State Machine

```python
from abc import ABC, abstractmethod

class OrderState(ABC):
    @abstractmethod
    def process(self, order):
        pass

    @abstractmethod
    def cancel(self, order):
        pass

    @abstractmethod
    def ship(self, order):
        pass

class PendingState(OrderState):
    def process(self, order):
        print("Processing payment...")
        order.set_state(ProcessingState())

    def cancel(self, order):
        print("Order cancelled")
        order.set_state(CancelledState())

    def ship(self, order):
        print("Cannot ship pending order")

class ProcessingState(OrderState):
    def process(self, order):
        print("Already processing")

    def cancel(self, order):
        print("Order cancelled")
        order.set_state(CancelledState())

    def ship(self, order):
        print("Shipping order...")
        order.set_state(ShippedState())

class ShippedState(OrderState):
    def process(self, order):
        print("Already shipped")

    def cancel(self, order):
        print("Cannot cancel shipped order")

    def ship(self, order):
        print("Already shipped")

class CancelledState(OrderState):
    def process(self, order):
        print("Cannot process cancelled order")

    def cancel(self, order):
        print("Already cancelled")

    def ship(self, order):
        print("Cannot ship cancelled order")

# Context
class Order:
    def __init__(self):
        self.state = PendingState()

    def set_state(self, state: OrderState):
        self.state = state

    def process(self):
        self.state.process(self)

    def cancel(self):
        self.state.cancel(self)

    def ship(self):
        self.state.ship(self)

# Usage
order = Order()

order.process()  # "Processing payment..."
order.ship()     # "Shipping order..."
order.cancel()   # "Cannot cancel shipped order"
```

### Django Model State

```python
# models.py
from django.db import models

class Order(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('processing', 'Processing'),
        ('shipped', 'Shipped'),
        ('delivered', 'Delivered'),
        ('cancelled', 'Cancelled')
    ]

    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')

    def can_transition_to(self, new_status: str) -> bool:
        """Define valid state transitions"""
        transitions = {
            'pending': ['processing', 'cancelled'],
            'processing': ['shipped', 'cancelled'],
            'shipped': ['delivered'],
            'delivered': [],
            'cancelled': []
        }
        return new_status in transitions.get(self.status, [])

    def transition_to(self, new_status: str):
        if not self.can_transition_to(new_status):
            raise ValueError(f"Cannot transition from {self.status} to {new_status}")

        self.status = new_status
        self.save()

    def process(self):
        self.transition_to('processing')

    def ship(self):
        self.transition_to('shipped')

    def deliver(self):
        self.transition_to('delivered')

    def cancel(self):
        if self.status in ['pending', 'processing']:
            self.transition_to('cancelled')
        else:
            raise ValueError("Cannot cancel order in current state")
```

## ❌ Common Mistakes

### 1. Too Many Observers

```python
# WRONG - Memory leak from not detaching
for i in range(1000):
    subject.attach(Observer())  # Never detached!

# CORRECT - Detach when done
observer = Observer()
subject.attach(observer)
# ... use observer
subject.detach(observer)
```

### 2. Strategy Without Interface

```python
# WRONG - No common interface
class BadStrategy1:
    def method_a(self): pass

class BadStrategy2:
    def method_b(self): pass  # Different method!

# CORRECT - Common interface
class Strategy(ABC):
    @abstractmethod
    def execute(self): pass
```

## 📚 Summary

Behavioral patterns handle communication between objects:

- **Observer**: Event notification system
- **Strategy**: Interchangeable algorithms
- **Command**: Encapsulate requests (undo/redo, queues)
- **State**: Behavior based on internal state

Essential for Django signals, task queues (Celery), state machines, and strategy-based processing in production systems.
