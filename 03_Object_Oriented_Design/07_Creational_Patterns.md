# Creational Design Patterns - Object Creation Strategies

## 📖 Concept Explanation

Creational patterns deal with object creation mechanisms, trying to create objects in a manner suitable for the situation. They abstract the instantiation process and help make systems independent of how objects are created, composed, and represented.

### Five Core Creational Patterns

1. **Singleton**: Ensure only one instance exists
2. **Factory Method**: Create objects without specifying exact class
3. **Abstract Factory**: Families of related objects
4. **Builder**: Construct complex objects step by step
5. **Prototype**: Clone existing objects

## 🧠 Why It Matters in Real Projects

### Production Use Cases

- **Singleton**: Database connections, configuration, logging
- **Factory**: Payment gateways, notification services, file parsers
- **Abstract Factory**: Cross-platform UI, database adapters
- **Builder**: Query builders, request builders, complex object construction
- **Prototype**: Document templates, cloning configurations

## 1️⃣ Singleton Pattern

### Concept

Ensures a class has only one instance and provides global access point.

### Implementation

```python
class Singleton:
    _instance = None
    _initialized = False

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        if not self._initialized:
            # Initialize only once
            self._initialized = True
            self.setup()

    def setup(self):
        print("Singleton initialized")

# Test
s1 = Singleton()  # "Singleton initialized"
s2 = Singleton()  # No output
print(s1 is s2)   # True - same instance
```

### Thread-Safe Singleton

```python
import threading

class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                # Double-checked locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

# Using metaclass (cleaner approach)
class SingletonMeta(type):
    _instances = {}
    _lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            with cls._lock:
                if cls not in cls._instances:
                    cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class DatabaseConnection(metaclass=SingletonMeta):
    def __init__(self):
        self.connection = self._create_connection()

    def _create_connection(self):
        print("Creating database connection")
        return "DB_CONNECTION_OBJECT"

# Usage
db1 = DatabaseConnection()  # "Creating database connection"
db2 = DatabaseConnection()  # No output
print(db1 is db2)           # True
```

### Django Settings Singleton

```python
# settings_manager.py
from django.conf import settings

class SettingsManager(metaclass=SingletonMeta):
    """Singleton for managing application settings"""

    def __init__(self):
        self._cache = {}

    def get(self, key, default=None):
        if key not in self._cache:
            self._cache[key] = getattr(settings, key, default)
        return self._cache[key]

    def reload(self):
        """Reload settings"""
        self._cache.clear()

# Usage in Django views
from django.http import JsonResponse

def config_view(request):
    manager = SettingsManager()
    return JsonResponse({
        'debug': manager.get('DEBUG'),
        'allowed_hosts': manager.get('ALLOWED_HOSTS')
    })
```

## 2️⃣ Factory Method Pattern

### Concept

Define an interface for creating objects, but let subclasses decide which class to instantiate.

### Simple Factory

```python
from abc import ABC, abstractmethod

class Notification(ABC):
    @abstractmethod
    def send(self, message: str):
        pass

class EmailNotification(Notification):
    def send(self, message: str):
        print(f"Email: {message}")

class SMSNotification(Notification):
    def send(self, message: str):
        print(f"SMS: {message}")

class PushNotification(Notification):
    def send(self, message: str):
        print(f"Push: {message}")

# Factory
class NotificationFactory:
    @staticmethod
    def create(notification_type: str) -> Notification:
        if notification_type == 'email':
            return EmailNotification()
        elif notification_type == 'sms':
            return SMSNotification()
        elif notification_type == 'push':
            return PushNotification()
        else:
            raise ValueError(f"Unknown type: {notification_type}")

# Usage
notification = NotificationFactory.create('email')
notification.send("Hello World")
```

### Factory Method with Django

```python
# models.py
from django.db import models

class Report(models.Model):
    report_type = models.CharField(max_length=20)
    data = models.JSONField()
    created_at = models.DateTimeField(auto_now_add=True)

# services/report_generator.py
from abc import ABC, abstractmethod
import pandas as pd
from io import BytesIO

class ReportGenerator(ABC):
    @abstractmethod
    def generate(self, data) -> BytesIO:
        pass

class PDFReportGenerator(ReportGenerator):
    def generate(self, data) -> BytesIO:
        from reportlab.pdfgen import canvas
        buffer = BytesIO()
        p = canvas.Canvas(buffer)
        p.drawString(100, 750, f"Report: {data}")
        p.save()
        buffer.seek(0)
        return buffer

class ExcelReportGenerator(ReportGenerator):
    def generate(self, data) -> BytesIO:
        buffer = BytesIO()
        df = pd.DataFrame(data)
        df.to_excel(buffer, index=False)
        buffer.seek(0)
        return buffer

class CSVReportGenerator(ReportGenerator):
    def generate(self, data) -> BytesIO:
        buffer = BytesIO()
        df = pd.DataFrame(data)
        csv_data = df.to_csv(index=False)
        buffer.write(csv_data.encode())
        buffer.seek(0)
        return buffer

class ReportGeneratorFactory:
    _generators = {
        'pdf': PDFReportGenerator,
        'excel': ExcelReportGenerator,
        'csv': CSVReportGenerator
    }

    @classmethod
    def create(cls, report_type: str) -> ReportGenerator:
        generator_class = cls._generators.get(report_type.lower())
        if not generator_class:
            raise ValueError(f"Unsupported report type: {report_type}")
        return generator_class()

# views.py
from django.http import HttpResponse
from django.views import View

class GenerateReportView(View):
    def get(self, request, report_id):
        report = Report.objects.get(id=report_id)

        # Factory creates appropriate generator
        generator = ReportGeneratorFactory.create(report.report_type)
        file_buffer = generator.generate(report.data)

        response = HttpResponse(
            file_buffer,
            content_type=f'application/{report.report_type}'
        )
        response['Content-Disposition'] = f'attachment; filename="report.{report.report_type}"'
        return response
```

### Payment Gateway Factory

```python
# payment/gateways.py
from abc import ABC, abstractmethod
from decimal import Decimal
from typing import Dict

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount: Decimal, token: str) -> Dict:
        pass

    @abstractmethod
    def refund(self, transaction_id: str) -> Dict:
        pass

class StripeGateway(PaymentGateway):
    def __init__(self, api_key: str):
        import stripe
        self.stripe = stripe
        self.stripe.api_key = api_key

    def charge(self, amount: Decimal, token: str) -> Dict:
        charge = self.stripe.Charge.create(
            amount=int(amount * 100),
            currency='usd',
            source=token
        )
        return {
            'transaction_id': charge.id,
            'status': 'success' if charge.paid else 'failed'
        }

    def refund(self, transaction_id: str) -> Dict:
        refund = self.stripe.Refund.create(charge=transaction_id)
        return {'refund_id': refund.id, 'status': refund.status}

class PayPalGateway(PaymentGateway):
    def __init__(self, client_id: str, secret: str):
        self.client_id = client_id
        self.secret = secret

    def charge(self, amount: Decimal, token: str) -> Dict:
        # PayPal API implementation
        return {'transaction_id': 'PP-123', 'status': 'success'}

    def refund(self, transaction_id: str) -> Dict:
        return {'refund_id': 'REF-123', 'status': 'completed'}

class PaymentGatewayFactory:
    @staticmethod
    def create(provider: str) -> PaymentGateway:
        from django.conf import settings

        if provider == 'stripe':
            return StripeGateway(settings.STRIPE_API_KEY)
        elif provider == 'paypal':
            return PayPalGateway(
                settings.PAYPAL_CLIENT_ID,
                settings.PAYPAL_SECRET
            )
        else:
            raise ValueError(f"Unknown provider: {provider}")

# services/payment_service.py
class PaymentService:
    def __init__(self, provider: str):
        self.gateway = PaymentGatewayFactory.create(provider)

    def process_payment(self, amount: Decimal, token: str) -> Dict:
        return self.gateway.charge(amount, token)
```

## 3️⃣ Abstract Factory Pattern

### Concept

Provides interface for creating families of related objects without specifying their concrete classes.

### Database Factory Example

```python
from abc import ABC, abstractmethod

# Abstract Products
class Connection(ABC):
    @abstractmethod
    def connect(self): pass

    @abstractmethod
    def disconnect(self): pass

class Query(ABC):
    @abstractmethod
    def execute(self, sql: str): pass

# Concrete Products - PostgreSQL
class PostgreSQLConnection(Connection):
    def connect(self):
        print("Connected to PostgreSQL")
        return self

    def disconnect(self):
        print("Disconnected from PostgreSQL")

class PostgreSQLQuery(Query):
    def __init__(self, connection):
        self.connection = connection

    def execute(self, sql: str):
        print(f"PostgreSQL executing: {sql}")
        return []

# Concrete Products - MySQL
class MySQLConnection(Connection):
    def connect(self):
        print("Connected to MySQL")
        return self

    def disconnect(self):
        print("Disconnected from MySQL")

class MySQLQuery(Query):
    def __init__(self, connection):
        self.connection = connection

    def execute(self, sql: str):
        print(f"MySQL executing: {sql}")
        return []

# Abstract Factory
class DatabaseFactory(ABC):
    @abstractmethod
    def create_connection(self) -> Connection:
        pass

    @abstractmethod
    def create_query(self, connection: Connection) -> Query:
        pass

# Concrete Factories
class PostgreSQLFactory(DatabaseFactory):
    def create_connection(self) -> Connection:
        return PostgreSQLConnection()

    def create_query(self, connection: Connection) -> Query:
        return PostgreSQLQuery(connection)

class MySQLFactory(DatabaseFactory):
    def create_connection(self) -> Connection:
        return MySQLConnection()

    def create_query(self, connection: Connection) -> Query:
        return MySQLQuery(connection)

# Client code
def database_operations(factory: DatabaseFactory):
    connection = factory.create_connection()
    connection.connect()

    query = factory.create_query(connection)
    query.execute("SELECT * FROM users")

    connection.disconnect()

# Usage
postgres_factory = PostgreSQLFactory()
database_operations(postgres_factory)

mysql_factory = MySQLFactory()
database_operations(mysql_factory)
```

### FastAPI Abstract Factory

```python
# factories/auth_factory.py
from abc import ABC, abstractmethod
from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

class TokenValidator(ABC):
    @abstractmethod
    def validate(self, token: str) -> dict:
        pass

class UserProvider(ABC):
    @abstractmethod
    def get_user(self, user_id: int):
        pass

# JWT Implementation
class JWTTokenValidator(TokenValidator):
    def __init__(self, secret_key: str):
        self.secret_key = secret_key

    def validate(self, token: str) -> dict:
        import jwt
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=['HS256'])
            return payload
        except jwt.InvalidTokenError:
            raise HTTPException(status_code=401, detail="Invalid token")

class DatabaseUserProvider(UserProvider):
    def get_user(self, user_id: int):
        # Query database
        return {"id": user_id, "username": "user"}

# OAuth2 Implementation
class OAuth2TokenValidator(TokenValidator):
    def validate(self, token: str) -> dict:
        # Validate OAuth2 token
        return {"user_id": 1, "scope": "read write"}

class LDAPUserProvider(UserProvider):
    def get_user(self, user_id: int):
        # Query LDAP
        return {"id": user_id, "username": "ldap_user"}

# Abstract Factory
class AuthFactory(ABC):
    @abstractmethod
    def create_token_validator(self) -> TokenValidator:
        pass

    @abstractmethod
    def create_user_provider(self) -> UserProvider:
        pass

class JWTAuthFactory(AuthFactory):
    def __init__(self, secret_key: str):
        self.secret_key = secret_key

    def create_token_validator(self) -> TokenValidator:
        return JWTTokenValidator(self.secret_key)

    def create_user_provider(self) -> UserProvider:
        return DatabaseUserProvider()

class OAuth2AuthFactory(AuthFactory):
    def create_token_validator(self) -> TokenValidator:
        return OAuth2TokenValidator()

    def create_user_provider(self) -> UserProvider:
        return LDAPUserProvider()

# Dependency
def get_auth_factory() -> AuthFactory:
    # Based on configuration
    return JWTAuthFactory(secret_key="secret")

# Usage in routes
from fastapi import FastAPI, Depends

app = FastAPI()
security = HTTPBearer()

@app.get("/protected")
async def protected_route(
    credentials: HTTPAuthorizationCredentials = Security(security),
    auth_factory: AuthFactory = Depends(get_auth_factory)
):
    validator = auth_factory.create_token_validator()
    payload = validator.validate(credentials.credentials)

    user_provider = auth_factory.create_user_provider()
    user = user_provider.get_user(payload['user_id'])

    return {"user": user}
```

## 4️⃣ Builder Pattern

### Concept

Construct complex objects step by step. Separates object construction from its representation.

### Query Builder

```python
class QueryBuilder:
    def __init__(self):
        self._select = []
        self._from = None
        self._where = []
        self._order = []
        self._limit = None

    def select(self, *fields):
        self._select.extend(fields)
        return self

    def from_table(self, table):
        self._from = table
        return self

    def where(self, condition):
        self._where.append(condition)
        return self

    def order_by(self, field, direction='ASC'):
        self._order.append(f"{field} {direction}")
        return self

    def limit(self, count):
        self._limit = count
        return self

    def build(self) -> str:
        if not self._from:
            raise ValueError("FROM clause is required")

        query = f"SELECT {', '.join(self._select) or '*'}"
        query += f" FROM {self._from}"

        if self._where:
            query += f" WHERE {' AND '.join(self._where)}"

        if self._order:
            query += f" ORDER BY {', '.join(self._order)}"

        if self._limit:
            query += f" LIMIT {self._limit}"

        return query

# Usage
query = (QueryBuilder()
    .select('id', 'name', 'email')
    .from_table('users')
    .where('age > 18')
    .where('active = true')
    .order_by('created_at', 'DESC')
    .limit(10)
    .build())

print(query)
# SELECT id, name, email FROM users WHERE age > 18 AND active = true ORDER BY created_at DESC LIMIT 10
```

### HTTP Request Builder

```python
from typing import Dict, Optional
import requests

class RequestBuilder:
    def __init__(self, url: str):
        self.url = url
        self.method = 'GET'
        self._headers = {}
        self._params = {}
        self._data = None
        self._json = None
        self._timeout = 30

    def get(self):
        self.method = 'GET'
        return self

    def post(self):
        self.method = 'POST'
        return self

    def put(self):
        self.method = 'PUT'
        return self

    def delete(self):
        self.method = 'DELETE'
        return self

    def header(self, key: str, value: str):
        self._headers[key] = value
        return self

    def headers(self, headers: Dict):
        self._headers.update(headers)
        return self

    def param(self, key: str, value):
        self._params[key] = value
        return self

    def params(self, params: Dict):
        self._params.update(params)
        return self

    def json_body(self, data: Dict):
        self._json = data
        return self

    def form_data(self, data: Dict):
        self._data = data
        return self

    def timeout(self, seconds: int):
        self._timeout = seconds
        return self

    def execute(self):
        return requests.request(
            method=self.method,
            url=self.url,
            headers=self._headers,
            params=self._params,
            json=self._json,
            data=self._data,
            timeout=self._timeout
        )

# Usage
response = (RequestBuilder('https://api.example.com/users')
    .get()
    .header('Authorization', 'Bearer token')
    .param('page', 1)
    .param('limit', 10)
    .timeout(15)
    .execute())

# POST example
response = (RequestBuilder('https://api.example.com/users')
    .post()
    .header('Content-Type', 'application/json')
    .json_body({'name': 'John', 'email': 'john@example.com'})
    .execute())
```

### Django Model Builder

```python
# builders/user_builder.py
from django.contrib.auth.models import User

class UserBuilder:
    def __init__(self):
        self._username = None
        self._email = None
        self._password = None
        self._first_name = ''
        self._last_name = ''
        self._is_staff = False
        self._is_superuser = False
        self._groups = []
        self._permissions = []

    def with_username(self, username: str):
        self._username = username
        return self

    def with_email(self, email: str):
        self._email = email
        return self

    def with_password(self, password: str):
        self._password = password
        return self

    def with_name(self, first_name: str, last_name: str):
        self._first_name = first_name
        self._last_name = last_name
        return self

    def as_staff(self):
        self._is_staff = True
        return self

    def as_superuser(self):
        self._is_superuser = True
        self._is_staff = True
        return self

    def with_groups(self, *groups):
        self._groups.extend(groups)
        return self

    def with_permissions(self, *permissions):
        self._permissions.extend(permissions)
        return self

    def build(self) -> User:
        if not self._username:
            raise ValueError("Username is required")

        user = User.objects.create_user(
            username=self._username,
            email=self._email,
            password=self._password,
            first_name=self._first_name,
            last_name=self._last_name,
            is_staff=self._is_staff,
            is_superuser=self._is_superuser
        )

        if self._groups:
            user.groups.set(self._groups)

        if self._permissions:
            user.user_permissions.set(self._permissions)

        return user

# Usage
from django.contrib.auth.models import Group

admin_group = Group.objects.get(name='Admins')

user = (UserBuilder()
    .with_username('johndoe')
    .with_email('john@example.com')
    .with_password('secure_password')
    .with_name('John', 'Doe')
    .as_staff()
    .with_groups(admin_group)
    .build())
```

## 5️⃣ Prototype Pattern

### Concept

Create new objects by cloning existing objects (prototypes) instead of creating from scratch.

### Basic Prototype

```python
import copy

class Prototype:
    def clone(self):
        """Shallow copy"""
        return copy.copy(self)

    def deep_clone(self):
        """Deep copy"""
        return copy.deepcopy(self)

class Document(Prototype):
    def __init__(self, title, content, metadata=None):
        self.title = title
        self.content = content
        self.metadata = metadata or {}

    def __str__(self):
        return f"Document(title='{self.title}', metadata={self.metadata})"

# Usage
original = Document("Report", "Content here", {"author": "John"})
print(original)  # Document(title='Report', metadata={'author': 'John'})

# Shallow clone
clone1 = original.clone()
clone1.title = "Report Copy"
clone1.metadata['version'] = 2
print(original)  # metadata changed! (shallow copy)
print(clone1)

# Deep clone
clone2 = original.deep_clone()
clone2.title = "Report Deep Copy"
clone2.metadata['version'] = 3
print(original)  # metadata unchanged (deep copy)
print(clone2)
```

### Configuration Prototype

```python
class Configuration:
    def __init__(self):
        self.settings = {}

    def set(self, key, value):
        self.settings[key] = value
        return self

    def clone(self):
        new_config = Configuration()
        new_config.settings = copy.deepcopy(self.settings)
        return new_config

# Base configurations
dev_config = (Configuration()
    .set('DEBUG', True)
    .set('DATABASE', 'dev_db')
    .set('CACHE_TIMEOUT', 300))

# Clone for different environments
prod_config = dev_config.clone()
prod_config.set('DEBUG', False)
prod_config.set('DATABASE', 'prod_db')

staging_config = prod_config.clone()
staging_config.set('DATABASE', 'staging_db')
```

### Django Model Cloning

```python
# models.py
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    description = models.TextField()

    def clone(self):
        """Clone product with new ID"""
        self.pk = None  # Reset primary key
        self._state.adding = True
        self.name = f"{self.name} (Copy)"
        self.save()
        return self

# Usage in views
def duplicate_product(request, product_id):
    original = Product.objects.get(id=product_id)
    cloned = original.clone()
    return JsonResponse({'id': cloned.id, 'name': cloned.name})
```

## ❌ Common Mistakes

### 1. Not Using Singleton Correctly

```python
# WRONG - Multiple instances possible
class BadSingleton:
    instance = None

    def __init__(self):
        BadSingleton.instance = self  # Race condition!

# CORRECT - Thread-safe
class GoodSingleton(metaclass=SingletonMeta):
    pass
```

### 2. Factory Returning Wrong Types

```python
# WRONG - Returns different types without interface
def bad_factory(type):
    if type == 'a':
        return ClassA()  # Different interface
    return ClassB()      # Different interface

# CORRECT - Common interface
def good_factory(type) -> Animal:
    if type == 'dog':
        return Dog()  # Both implement Animal
    return Cat()
```

## 🔐 Security Considerations

```python
# Secure singleton for API keys
class SecureConfig(metaclass=SingletonMeta):
    def __init__(self):
        self._secrets = {}

    def set_secret(self, key: str, value: str):
        # Encrypt before storing
        encrypted = self._encrypt(value)
        self._secrets[key] = encrypted

    def get_secret(self, key: str) -> str:
        encrypted = self._secrets.get(key)
        if encrypted:
            return self._decrypt(encrypted)
        return None

    def _encrypt(self, value: str) -> str:
        from cryptography.fernet import Fernet
        # Use proper encryption
        return value  # Simplified

    def _decrypt(self, value: str) -> str:
        return value  # Simplified
```

## 📚 Summary

Creational patterns provide flexible object creation:

- **Singleton**: One instance globally
- **Factory Method**: Delegate object creation
- **Abstract Factory**: Family of related objects
- **Builder**: Step-by-step construction
- **Prototype**: Clone existing objects

Use in Django/FastAPI for payment gateways, notifications, database connections, and configuration management.
