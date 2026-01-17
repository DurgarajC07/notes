# Structural Design Patterns - Object Composition

## 📖 Concept Explanation

Structural patterns explain how to assemble objects and classes into larger structures while keeping these structures flexible and efficient. They focus on simplifying relationships between entities.

### Seven Core Structural Patterns

1. **Adapter**: Convert interface to another interface
2. **Decorator**: Add behavior without modifying class
3. **Facade**: Simplified interface to complex subsystem
4. **Proxy**: Placeholder/surrogate for another object
5. **Composite**: Tree structure of objects
6. **Bridge**: Separate abstraction from implementation
7. **Flyweight**: Share objects to minimize memory

## 🧠 Why It Matters in Real Projects

### Production Use Cases

- **Adapter**: Third-party API integration, legacy system compatibility
- **Decorator**: Logging, caching, authentication, validation
- **Facade**: Complex library wrappers, microservice APIs
- **Proxy**: Lazy loading, access control, caching, monitoring
- **Composite**: File systems, UI components, organizational hierarchies

## 1️⃣ Adapter Pattern

### Concept

Convert the interface of a class into another interface clients expect. Allows classes with incompatible interfaces to work together.

### Basic Adapter

```python
# Target interface (what client expects)
class MediaPlayer:
    def play(self, filename: str):
        pass

# Adaptee (existing class with different interface)
class VLCPlayer:
    def play_vlc(self, filename: str):
        print(f"Playing {filename} with VLC")

class WindowsMediaPlayer:
    def play_wmp(self, filename: str):
        print(f"Playing {filename} with Windows Media Player")

# Adapter
class VLCAdapter(MediaPlayer):
    def __init__(self):
        self.vlc = VLCPlayer()

    def play(self, filename: str):
        self.vlc.play_vlc(filename)

class WMPAdapter(MediaPlayer):
    def __init__(self):
        self.wmp = WindowsMediaPlayer()

    def play(self, filename: str):
        self.wmp.play_wmp(filename)

# Client code
def play_audio(player: MediaPlayer, filename: str):
    player.play(filename)

# Usage
vlc_adapter = VLCAdapter()
play_audio(vlc_adapter, "song.mp3")

wmp_adapter = WMPAdapter()
play_audio(wmp_adapter, "song.mp3")
```

### Payment Gateway Adapter

```python
# Target interface
from abc import ABC, abstractmethod
from decimal import Decimal
from typing import Dict

class PaymentProcessor(ABC):
    @abstractmethod
    def process_payment(self, amount: Decimal, card_data: Dict) -> Dict:
        pass

# Adaptee 1 - Stripe (different interface)
class StripeAPI:
    def create_charge(self, amount_cents: int, token: str, currency: str = 'usd'):
        print(f"Stripe: Charging {amount_cents} cents")
        return {'id': 'ch_123', 'status': 'succeeded'}

# Adaptee 2 - PayPal (different interface)
class PayPalAPI:
    def make_payment(self, total: float, payment_method: Dict):
        print(f"PayPal: Processing ${total}")
        return {'transaction_id': 'PP-456', 'state': 'approved'}

# Adapters
class StripeAdapter(PaymentProcessor):
    def __init__(self, api_key: str):
        self.stripe = StripeAPI()
        self.api_key = api_key

    def process_payment(self, amount: Decimal, card_data: Dict) -> Dict:
        # Convert amount to cents
        amount_cents = int(amount * 100)

        # Call Stripe with their expected interface
        result = self.stripe.create_charge(
            amount_cents=amount_cents,
            token=card_data['token']
        )

        # Convert to our standard format
        return {
            'transaction_id': result['id'],
            'status': 'success' if result['status'] == 'succeeded' else 'failed',
            'amount': amount
        }

class PayPalAdapter(PaymentProcessor):
    def __init__(self, client_id: str, secret: str):
        self.paypal = PayPalAPI()
        self.client_id = client_id

    def process_payment(self, amount: Decimal, card_data: Dict) -> Dict:
        # Call PayPal with their expected interface
        result = self.paypal.make_payment(
            total=float(amount),
            payment_method={'type': 'card', 'details': card_data}
        )

        # Convert to our standard format
        return {
            'transaction_id': result['transaction_id'],
            'status': 'success' if result['state'] == 'approved' else 'failed',
            'amount': amount
        }

# Django view using adapter
from django.http import JsonResponse
from decimal import Decimal

def process_payment_view(request):
    amount = Decimal(request.POST.get('amount'))
    card_data = {
        'token': request.POST.get('card_token'),
        'cvv': request.POST.get('cvv')
    }
    provider = request.POST.get('provider')  # 'stripe' or 'paypal'

    # Select adapter based on provider
    if provider == 'stripe':
        processor = StripeAdapter(api_key='sk_test_...')
    else:
        processor = PayPalAdapter(client_id='...', secret='...')

    # Process payment using unified interface
    result = processor.process_payment(amount, card_data)

    return JsonResponse(result)
```

### External API Adapter

```python
# Our interface
class UserRepository(ABC):
    @abstractmethod
    def get_user(self, user_id: int) -> Dict:
        pass

    @abstractmethod
    def get_all_users(self) -> list:
        pass

# External API with different structure
class LegacyUserAPI:
    def fetch_user_by_id(self, id: int):
        # Returns different structure
        return {
            'userId': id,
            'fullName': 'John Doe',
            'emailAddress': 'john@example.com',
            'accountStatus': 'ACTIVE'
        }

    def fetch_all_users(self):
        return [
            {'userId': 1, 'fullName': 'John Doe'},
            {'userId': 2, 'fullName': 'Jane Smith'}
        ]

# Adapter
class LegacyUserAdapter(UserRepository):
    def __init__(self):
        self.legacy_api = LegacyUserAPI()

    def get_user(self, user_id: int) -> Dict:
        # Adapt the response
        legacy_user = self.legacy_api.fetch_user_by_id(user_id)

        return {
            'id': legacy_user['userId'],
            'name': legacy_user['fullName'],
            'email': legacy_user['emailAddress'],
            'status': legacy_user['accountStatus'].lower()
        }

    def get_all_users(self) -> list:
        legacy_users = self.legacy_api.fetch_all_users()

        return [
            {
                'id': user['userId'],
                'name': user['fullName']
            }
            for user in legacy_users
        ]

# Usage in FastAPI
from fastapi import FastAPI, Depends

app = FastAPI()

def get_user_repository() -> UserRepository:
    return LegacyUserAdapter()

@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    repository: UserRepository = Depends(get_user_repository)
):
    user = repository.get_user(user_id)
    return user
```

## 2️⃣ Decorator Pattern

### Concept

Attach additional responsibilities to an object dynamically. Provides a flexible alternative to subclassing.

### Function Decorators (Python Native)

```python
import time
from functools import wraps

def timing_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.2f}s")
        return result
    return wrapper

def cache_decorator(func):
    cache = {}

    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

@timing_decorator
@cache_decorator
def expensive_operation(n):
    time.sleep(1)
    return n ** 2

# First call: 1 second
print(expensive_operation(5))  # "expensive_operation took 1.00s"

# Second call: instant (cached)
print(expensive_operation(5))  # "expensive_operation took 0.00s"
```

### Class-Based Decorator

```python
from abc import ABC, abstractmethod

# Component interface
class DataSource(ABC):
    @abstractmethod
    def read(self) -> str:
        pass

    @abstractmethod
    def write(self, data: str):
        pass

# Concrete component
class FileDataSource(DataSource):
    def __init__(self, filename: str):
        self.filename = filename

    def read(self) -> str:
        with open(self.filename, 'r') as f:
            return f.read()

    def write(self, data: str):
        with open(self.filename, 'w') as f:
            f.write(data)

# Base decorator
class DataSourceDecorator(DataSource):
    def __init__(self, wrapped: DataSource):
        self._wrapped = wrapped

    def read(self) -> str:
        return self._wrapped.read()

    def write(self, data: str):
        self._wrapped.write(data)

# Concrete decorators
class EncryptionDecorator(DataSourceDecorator):
    def read(self) -> str:
        data = self._wrapped.read()
        return self._decrypt(data)

    def write(self, data: str):
        encrypted = self._encrypt(data)
        self._wrapped.write(encrypted)

    def _encrypt(self, data: str) -> str:
        # Simple encryption (use proper encryption in production)
        return data[::-1]

    def _decrypt(self, data: str) -> str:
        return data[::-1]

class CompressionDecorator(DataSourceDecorator):
    def read(self) -> str:
        compressed = self._wrapped.read()
        return self._decompress(compressed)

    def write(self, data: str):
        compressed = self._compress(data)
        self._wrapped.write(compressed)

    def _compress(self, data: str) -> str:
        import zlib
        return zlib.compress(data.encode()).hex()

    def _decompress(self, data: str) -> str:
        import zlib
        return zlib.decompress(bytes.fromhex(data)).decode()

# Usage - Stack decorators
source = FileDataSource('data.txt')
source = EncryptionDecorator(source)
source = CompressionDecorator(source)

source.write("Sensitive data")
print(source.read())  # Decompressed and decrypted
```

### Django Middleware as Decorator

```python
# middleware/logging_middleware.py
import time
import logging

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Before view
        start_time = time.time()
        request_id = request.META.get('HTTP_X_REQUEST_ID', 'N/A')

        logger.info(f"Request {request_id}: {request.method} {request.path}")

        # Call next middleware/view
        response = self.get_response(request)

        # After view
        duration = time.time() - start_time
        logger.info(
            f"Response {request_id}: {response.status_code} "
            f"({duration:.2f}s)"
        )

        return response

# middleware/auth_middleware.py
from django.http import JsonResponse

class AuthenticationMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Skip authentication for public endpoints
        if request.path.startswith('/public'):
            return self.get_response(request)

        # Check authentication
        token = request.headers.get('Authorization')
        if not token:
            return JsonResponse({'error': 'Unauthorized'}, status=401)

        # Validate token and attach user to request
        user = self.validate_token(token)
        if not user:
            return JsonResponse({'error': 'Invalid token'}, status=401)

        request.user = user
        return self.get_response(request)

    def validate_token(self, token):
        # Validate JWT token
        return {'id': 1, 'username': 'john'}

# settings.py
MIDDLEWARE = [
    'middleware.logging_middleware.RequestLoggingMiddleware',
    'middleware.auth_middleware.AuthenticationMiddleware',
    # ... other middleware
]
```

### FastAPI Dependencies as Decorators

```python
from fastapi import FastAPI, Depends, HTTPException
from typing import Optional

app = FastAPI()

# Dependency that acts like decorator
def require_auth(token: Optional[str] = None):
    if not token:
        raise HTTPException(status_code=401, detail="Token required")

    # Validate token
    if token != "valid_token":
        raise HTTPException(status_code=401, detail="Invalid token")

    return {"user_id": 1, "username": "john"}

def require_role(role: str):
    def role_checker(user: dict = Depends(require_auth)):
        if user.get('role') != role:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return role_checker

# Usage
@app.get("/protected")
async def protected_route(user: dict = Depends(require_auth)):
    return {"message": f"Hello {user['username']}"}

@app.get("/admin")
async def admin_route(user: dict = Depends(require_role('admin'))):
    return {"message": "Admin access granted"}
```

## 3️⃣ Facade Pattern

### Concept

Provides a simplified interface to a complex subsystem. Hides the complexity behind a simple interface.

### E-commerce Order Facade

```python
# Complex subsystems
class InventorySystem:
    def check_stock(self, product_id: int, quantity: int) -> bool:
        print(f"Checking stock for product {product_id}")
        return True

    def reserve_stock(self, product_id: int, quantity: int):
        print(f"Reserving {quantity} units of product {product_id}")

class PaymentSystem:
    def process_payment(self, amount: float, card_token: str) -> str:
        print(f"Processing payment of ${amount}")
        return "PAYMENT_123"

class ShippingSystem:
    def calculate_shipping(self, address: dict) -> float:
        print("Calculating shipping cost")
        return 9.99

    def create_shipment(self, order_id: int, address: dict) -> str:
        print(f"Creating shipment for order {order_id}")
        return "SHIP_456"

class NotificationSystem:
    def send_order_confirmation(self, email: str, order_id: int):
        print(f"Sending confirmation email to {email}")

    def send_shipping_notification(self, email: str, tracking_number: str):
        print(f"Sending shipping notification to {email}")

# Facade - Simplified interface
class OrderFacade:
    def __init__(self):
        self.inventory = InventorySystem()
        self.payment = PaymentSystem()
        self.shipping = ShippingSystem()
        self.notification = NotificationSystem()

    def place_order(self, user_email: str, product_id: int,
                   quantity: int, address: dict, card_token: str) -> dict:
        """Single method handles entire order workflow"""

        # 1. Check inventory
        if not self.inventory.check_stock(product_id, quantity):
            return {'success': False, 'error': 'Out of stock'}

        # 2. Calculate total
        product_price = 29.99  # Would fetch from database
        shipping_cost = self.shipping.calculate_shipping(address)
        total = (product_price * quantity) + shipping_cost

        # 3. Process payment
        try:
            payment_id = self.payment.process_payment(total, card_token)
        except Exception as e:
            return {'success': False, 'error': 'Payment failed'}

        # 4. Reserve inventory
        self.inventory.reserve_stock(product_id, quantity)

        # 5. Create shipment
        order_id = 12345  # Would be generated
        tracking = self.shipping.create_shipment(order_id, address)

        # 6. Send notifications
        self.notification.send_order_confirmation(user_email, order_id)
        self.notification.send_shipping_notification(user_email, tracking)

        return {
            'success': True,
            'order_id': order_id,
            'payment_id': payment_id,
            'tracking_number': tracking,
            'total': total
        }

# Client code is simple
facade = OrderFacade()

result = facade.place_order(
    user_email='customer@example.com',
    product_id=101,
    quantity=2,
    address={'street': '123 Main St', 'city': 'NYC'},
    card_token='tok_visa'
)

print(result)
```

### Django Service Facade

```python
# services/user_service_facade.py
from django.contrib.auth.models import User
from django.core.mail import send_mail
from typing import Dict

class UserServiceFacade:
    """Facade for user-related operations"""

    @staticmethod
    def register_user(username: str, email: str, password: str) -> Dict:
        """Complete user registration workflow"""

        # 1. Create user
        user = User.objects.create_user(
            username=username,
            email=email,
            password=password
        )

        # 2. Create profile
        from profiles.models import UserProfile
        profile = UserProfile.objects.create(user=user)

        # 3. Send welcome email
        send_mail(
            subject='Welcome!',
            message=f'Welcome {username}!',
            from_email='noreply@example.com',
            recipient_list=[email]
        )

        # 4. Create default settings
        from settings.models import UserSettings
        UserSettings.objects.create(user=user)

        # 5. Log registration
        import logging
        logging.info(f"New user registered: {username}")

        return {
            'success': True,
            'user_id': user.id,
            'username': user.username
        }

    @staticmethod
    def deactivate_user(user_id: int) -> Dict:
        """Complete user deactivation workflow"""

        user = User.objects.get(id=user_id)

        # 1. Deactivate account
        user.is_active = False
        user.save()

        # 2. Clear sessions
        from django.contrib.sessions.models import Session
        Session.objects.filter(session_data__contains=str(user_id)).delete()

        # 3. Notify user
        send_mail(
            subject='Account Deactivated',
            message='Your account has been deactivated.',
            from_email='noreply@example.com',
            recipient_list=[user.email]
        )

        # 4. Log deactivation
        import logging
        logging.info(f"User deactivated: {user.username}")

        return {'success': True}

# views.py
from django.views import View
from django.http import JsonResponse

class RegisterView(View):
    def post(self, request):
        # Simple client code thanks to facade
        result = UserServiceFacade.register_user(
            username=request.POST['username'],
            email=request.POST['email'],
            password=request.POST['password']
        )
        return JsonResponse(result)
```

## 4️⃣ Proxy Pattern

### Concept

Provide a surrogate or placeholder for another object to control access to it.

### Types of Proxies

1. **Virtual Proxy**: Lazy loading expensive objects
2. **Protection Proxy**: Access control
3. **Remote Proxy**: Represent remote object locally
4. **Caching Proxy**: Cache results

### Virtual Proxy - Lazy Loading

```python
class ExpensiveObject:
    def __init__(self):
        print("Loading expensive object...")
        import time
        time.sleep(2)  # Simulate expensive initialization
        self.data = "Expensive data"

    def process(self):
        return f"Processing: {self.data}"

class LazyProxy:
    def __init__(self):
        self._real_object = None

    def process(self):
        # Load only when needed
        if self._real_object is None:
            self._real_object = ExpensiveObject()

        return self._real_object.process()

# Usage
proxy = LazyProxy()  # Instant
print("Proxy created")

# Object loaded only when accessed
print(proxy.process())  # 2 second delay
print(proxy.process())  # Instant (already loaded)
```

### Protection Proxy - Access Control

```python
from abc import ABC, abstractmethod

class DatabaseConnection(ABC):
    @abstractmethod
    def execute_query(self, query: str):
        pass

class RealDatabaseConnection(DatabaseConnection):
    def execute_query(self, query: str):
        print(f"Executing: {query}")
        return [{"id": 1, "name": "John"}]

class DatabaseConnectionProxy(DatabaseConnection):
    def __init__(self, user_role: str):
        self.user_role = user_role
        self._real_connection = RealDatabaseConnection()

    def execute_query(self, query: str):
        # Check permissions
        if self._is_dangerous_query(query):
            if self.user_role != 'admin':
                raise PermissionError("Only admins can execute this query")

        # Log query
        self._log_query(query)

        # Execute
        return self._real_connection.execute_query(query)

    def _is_dangerous_query(self, query: str) -> bool:
        dangerous_keywords = ['DROP', 'DELETE', 'TRUNCATE', 'ALTER']
        return any(keyword in query.upper() for keyword in dangerous_keywords)

    def _log_query(self, query: str):
        import logging
        logging.info(f"User {self.user_role} executing: {query}")

# Usage
admin_db = DatabaseConnectionProxy(user_role='admin')
admin_db.execute_query("SELECT * FROM users")  # OK
admin_db.execute_query("DROP TABLE users")     # OK (admin)

user_db = DatabaseConnectionProxy(user_role='user')
user_db.execute_query("SELECT * FROM users")  # OK
# user_db.execute_query("DROP TABLE users")   # PermissionError
```

### Caching Proxy

```python
import time
from typing import Dict

class APIClient:
    def fetch_data(self, endpoint: str) -> Dict:
        print(f"Fetching from API: {endpoint}")
        time.sleep(1)  # Simulate network delay
        return {"data": f"Result from {endpoint}"}

class CachingAPIProxy:
    def __init__(self):
        self._client = APIClient()
        self._cache = {}
        self._cache_ttl = {}

    def fetch_data(self, endpoint: str, ttl: int = 300) -> Dict:
        current_time = time.time()

        # Check cache
        if endpoint in self._cache:
            if current_time < self._cache_ttl[endpoint]:
                print(f"Cache hit: {endpoint}")
                return self._cache[endpoint]

        # Fetch from real API
        print(f"Cache miss: {endpoint}")
        result = self._client.fetch_data(endpoint)

        # Update cache
        self._cache[endpoint] = result
        self._cache_ttl[endpoint] = current_time + ttl

        return result

# Usage
proxy = CachingAPIProxy()

# First call - fetch from API (1 second)
print(proxy.fetch_data('/users'))

# Second call - from cache (instant)
print(proxy.fetch_data('/users'))
```

### Django Caching Proxy

```python
# services/cached_repository.py
from django.core.cache import cache
from typing import Optional, List
from functools import wraps

def cached_method(timeout=300):
    def decorator(func):
        @wraps(func)
        def wrapper(self, *args, **kwargs):
            # Generate cache key
            cache_key = f"{self.__class__.__name__}:{func.__name__}:{args}:{kwargs}"

            # Try cache
            result = cache.get(cache_key)
            if result is not None:
                print(f"Cache hit: {cache_key}")
                return result

            # Call real method
            print(f"Cache miss: {cache_key}")
            result = func(self, *args, **kwargs)

            # Cache result
            cache.set(cache_key, result, timeout)

            return result
        return wrapper
    return decorator

class UserRepository:
    @cached_method(timeout=600)
    def get_user(self, user_id: int):
        from django.contrib.auth.models import User
        return User.objects.get(id=user_id)

    @cached_method(timeout=300)
    def get_all_users(self) -> List:
        from django.contrib.auth.models import User
        return list(User.objects.all())
```

## ❌ Common Mistakes

### 1. Over-using Adapter

```python
# WRONG - Adapter for similar interfaces
class BadAdapter:
    def method_a(self): pass

# CORRECT - Only when interfaces are incompatible
class GoodAdapter:
    def translate_completely_different_interface(self): pass
```

### 2. Decorator Breaking Interface

```python
# WRONG - Decorator changes interface
class BadDecorator:
    def new_method(self):  # Adds new method!
        pass

# CORRECT - Maintains same interface
class GoodDecorator(Component):
    def same_method(self):  # Same interface
        return self._wrapped.same_method()
```

## 📚 Summary

Structural patterns compose objects and classes:

- **Adapter**: Interface conversion
- **Decorator**: Add behavior dynamically
- **Facade**: Simplify complex systems
- **Proxy**: Control access, lazy loading, caching

Essential for Django/FastAPI middleware, dependency injection, caching, and API integration.
