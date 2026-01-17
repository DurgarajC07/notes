# Abstract Base Classes (ABC) - Python Interfaces

## 📖 Concept Explanation

Abstract Base Classes (ABCs) provide a way to define interfaces in Python. They allow you to define methods that must be implemented by concrete subclasses, enforcing a contract that all subclasses must follow.

### Why Use ABCs?

- **Interface Definition**: Define what methods a class must implement
- **Type Safety**: Ensure subclasses implement required methods
- **Documentation**: Self-documenting code - clear expectations
- **Polymorphism**: Multiple implementations of same interface
- **Framework Design**: Django, FastAPI use ABCs extensively

### Basic ABC Example

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    """Abstract base class for animals"""
    
    @abstractmethod
    def make_sound(self):
        """All animals must make a sound"""
        pass
    
    @abstractmethod
    def move(self):
        """All animals must move"""
        pass
    
    def breathe(self):
        """Concrete method - all animals breathe the same way"""
        return "Inhale, exhale"

# This will fail - can't instantiate abstract class
# animal = Animal()  # TypeError

class Dog(Animal):
    def make_sound(self):
        return "Woof!"
    
    def move(self):
        return "Running on four legs"

class Bird(Animal):
    def make_sound(self):
        return "Chirp!"
    
    def move(self):
        return "Flying"

# Works - all abstract methods implemented
dog = Dog()
print(dog.make_sound())  # "Woof!"
print(dog.breathe())     # "Inhale, exhale"
```

## 🧠 Why It Matters in Real Projects

### Production Use Cases

1. **Database Repositories**: Define repository interface
2. **Payment Gateways**: Multiple payment provider implementations
3. **Storage Backends**: Different storage implementations (S3, local, etc.)
4. **Authentication**: Multiple auth strategies
5. **Notification Services**: Email, SMS, Push implementations
6. **Cache Providers**: Redis, Memcached, local cache

## ⚙️ Internal Working

### How ABC Works

```python
from abc import ABC, ABCMeta, abstractmethod

# These are equivalent
class MyABC(ABC):
    pass

class MyABC(metaclass=ABCMeta):
    pass

# ABC uses metaclass to track abstract methods
class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

# Check if class is abstract
print(Shape.__abstractmethods__)  # frozenset({'area'})

# Subclass that doesn't implement all methods
class IncompleteShape(Shape):
    pass

# This fails at instantiation
try:
    shape = IncompleteShape()
except TypeError as e:
    print(e)  # Can't instantiate abstract class
```

### Abstract Properties

```python
from abc import ABC, abstractmethod

class Person(ABC):
    @property
    @abstractmethod
    def name(self):
        """Abstract property"""
        pass
    
    @property
    @abstractmethod
    def age(self):
        pass

class Employee(Person):
    def __init__(self, name, age):
        self._name = name
        self._age = age
    
    @property
    def name(self):
        return self._name
    
    @property
    def age(self):
        return self._age

emp = Employee("Alice", 30)
print(emp.name)  # "Alice"
```

## ✅ Best Practices

### 1. Repository Pattern with ABC

```python
from abc import ABC, abstractmethod
from typing import List, Optional

class Repository(ABC):
    """Abstract repository interface"""
    
    @abstractmethod
    def get_by_id(self, id: int):
        """Get entity by ID"""
        pass
    
    @abstractmethod
    def get_all(self) -> List:
        """Get all entities"""
        pass
    
    @abstractmethod
    def save(self, entity) -> None:
        """Save entity"""
        pass
    
    @abstractmethod
    def delete(self, id: int) -> None:
        """Delete entity"""
        pass

# PostgreSQL implementation
class PostgresUserRepository(Repository):
    def __init__(self, connection):
        self.connection = connection
    
    def get_by_id(self, id: int):
        cursor = self.connection.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", (id,))
        return cursor.fetchone()
    
    def get_all(self) -> List:
        cursor = self.connection.cursor()
        cursor.execute("SELECT * FROM users")
        return cursor.fetchall()
    
    def save(self, entity) -> None:
        cursor = self.connection.cursor()
        cursor.execute(
            "INSERT INTO users (name, email) VALUES (%s, %s)",
            (entity.name, entity.email)
        )
        self.connection.commit()
    
    def delete(self, id: int) -> None:
        cursor = self.connection.cursor()
        cursor.execute("DELETE FROM users WHERE id = %s", (id,))
        self.connection.commit()

# MongoDB implementation
class MongoUserRepository(Repository):
    def __init__(self, collection):
        self.collection = collection
    
    def get_by_id(self, id: int):
        return self.collection.find_one({"_id": id})
    
    def get_all(self) -> List:
        return list(self.collection.find())
    
    def save(self, entity) -> None:
        self.collection.insert_one({
            "name": entity.name,
            "email": entity.email
        })
    
    def delete(self, id: int) -> None:
        self.collection.delete_one({"_id": id})

# Client code works with any repository
def process_users(repository: Repository):
    users = repository.get_all()
    for user in users:
        print(user)
```

### 2. Payment Gateway Interface

```python
from abc import ABC, abstractmethod
from decimal import Decimal
from typing import Dict

class PaymentGateway(ABC):
    """Abstract payment gateway"""
    
    @abstractmethod
    def charge(self, amount: Decimal, card_token: str) -> Dict:
        """Charge a card"""
        pass
    
    @abstractmethod
    def refund(self, transaction_id: str, amount: Decimal) -> Dict:
        """Refund a transaction"""
        pass
    
    @abstractmethod
    def get_transaction(self, transaction_id: str) -> Dict:
        """Get transaction details"""
        pass

class StripeGateway(PaymentGateway):
    def __init__(self, api_key: str):
        self.api_key = api_key
    
    def charge(self, amount: Decimal, card_token: str) -> Dict:
        # Stripe API call
        import stripe
        stripe.api_key = self.api_key
        
        charge = stripe.Charge.create(
            amount=int(amount * 100),  # Convert to cents
            currency='usd',
            source=card_token
        )
        
        return {
            'transaction_id': charge.id,
            'status': charge.status,
            'amount': amount
        }
    
    def refund(self, transaction_id: str, amount: Decimal) -> Dict:
        import stripe
        refund = stripe.Refund.create(
            charge=transaction_id,
            amount=int(amount * 100)
        )
        return {'refund_id': refund.id, 'status': refund.status}
    
    def get_transaction(self, transaction_id: str) -> Dict:
        import stripe
        charge = stripe.Charge.retrieve(transaction_id)
        return {
            'id': charge.id,
            'amount': charge.amount / 100,
            'status': charge.status
        }

class PayPalGateway(PaymentGateway):
    def __init__(self, client_id: str, secret: str):
        self.client_id = client_id
        self.secret = secret
    
    def charge(self, amount: Decimal, card_token: str) -> Dict:
        # PayPal API call
        return {
            'transaction_id': 'PAYPAL-123',
            'status': 'completed',
            'amount': amount
        }
    
    def refund(self, transaction_id: str, amount: Decimal) -> Dict:
        # PayPal refund API
        return {'refund_id': 'REFUND-123', 'status': 'completed'}
    
    def get_transaction(self, transaction_id: str) -> Dict:
        # PayPal get transaction API
        return {'id': transaction_id, 'amount': 100.0, 'status': 'completed'}

# Service layer uses abstract interface
class PaymentService:
    def __init__(self, gateway: PaymentGateway):
        self.gateway = gateway
    
    def process_payment(self, amount: Decimal, card_token: str):
        try:
            result = self.gateway.charge(amount, card_token)
            return {
                'success': True,
                'transaction_id': result['transaction_id']
            }
        except Exception as e:
            return {'success': False, 'error': str(e)}

# Easy to switch payment providers
stripe_service = PaymentService(StripeGateway(api_key="sk_test_..."))
paypal_service = PaymentService(PayPalGateway(client_id="...", secret="..."))
```

### 3. Storage Backend Interface

```python
from abc import ABC, abstractmethod
from typing import BinaryIO, Optional
import os

class StorageBackend(ABC):
    """Abstract storage interface"""
    
    @abstractmethod
    def save(self, file_path: str, content: BinaryIO) -> str:
        """Save file and return URL"""
        pass
    
    @abstractmethod
    def load(self, file_path: str) -> bytes:
        """Load file content"""
        pass
    
    @abstractmethod
    def delete(self, file_path: str) -> bool:
        """Delete file"""
        pass
    
    @abstractmethod
    def exists(self, file_path: str) -> bool:
        """Check if file exists"""
        pass
    
    @abstractmethod
    def get_url(self, file_path: str) -> str:
        """Get public URL"""
        pass

class S3Storage(StorageBackend):
    def __init__(self, bucket: str, region: str):
        import boto3
        self.bucket = bucket
        self.s3_client = boto3.client('s3', region_name=region)
    
    def save(self, file_path: str, content: BinaryIO) -> str:
        self.s3_client.upload_fileobj(content, self.bucket, file_path)
        return self.get_url(file_path)
    
    def load(self, file_path: str) -> bytes:
        obj = self.s3_client.get_object(Bucket=self.bucket, Key=file_path)
        return obj['Body'].read()
    
    def delete(self, file_path: str) -> bool:
        self.s3_client.delete_object(Bucket=self.bucket, Key=file_path)
        return True
    
    def exists(self, file_path: str) -> bool:
        try:
            self.s3_client.head_object(Bucket=self.bucket, Key=file_path)
            return True
        except:
            return False
    
    def get_url(self, file_path: str) -> str:
        return f"https://{self.bucket}.s3.amazonaws.com/{file_path}"

class LocalStorage(StorageBackend):
    def __init__(self, base_path: str):
        self.base_path = base_path
        os.makedirs(base_path, exist_ok=True)
    
    def save(self, file_path: str, content: BinaryIO) -> str:
        full_path = os.path.join(self.base_path, file_path)
        os.makedirs(os.path.dirname(full_path), exist_ok=True)
        
        with open(full_path, 'wb') as f:
            f.write(content.read())
        
        return self.get_url(file_path)
    
    def load(self, file_path: str) -> bytes:
        full_path = os.path.join(self.base_path, file_path)
        with open(full_path, 'rb') as f:
            return f.read()
    
    def delete(self, file_path: str) -> bool:
        full_path = os.path.join(self.base_path, file_path)
        if os.path.exists(full_path):
            os.remove(full_path)
            return True
        return False
    
    def exists(self, file_path: str) -> bool:
        full_path = os.path.join(self.base_path, file_path)
        return os.path.exists(full_path)
    
    def get_url(self, file_path: str) -> str:
        return f"/media/{file_path}"

# File upload service
class FileUploadService:
    def __init__(self, storage: StorageBackend):
        self.storage = storage
    
    def upload(self, file_name: str, content: BinaryIO) -> str:
        """Upload file and return URL"""
        # Generate unique file path
        import uuid
        file_path = f"uploads/{uuid.uuid4()}/{file_name}"
        
        # Save to storage
        url = self.storage.save(file_path, content)
        
        return url
    
    def delete(self, file_path: str) -> bool:
        return self.storage.delete(file_path)
```

### 4. Cache Interface

```python
from abc import ABC, abstractmethod
from typing import Any, Optional

class Cache(ABC):
    """Abstract cache interface"""
    
    @abstractmethod
    def get(self, key: str) -> Optional[Any]:
        pass
    
    @abstractmethod
    def set(self, key: str, value: Any, ttl: int = 3600) -> bool:
        pass
    
    @abstractmethod
    def delete(self, key: str) -> bool:
        pass
    
    @abstractmethod
    def exists(self, key: str) -> bool:
        pass
    
    @abstractmethod
    def clear(self) -> bool:
        pass

class RedisCache(Cache):
    def __init__(self, host='localhost', port=6379):
        import redis
        self.redis = redis.Redis(host=host, port=port, decode_responses=True)
    
    def get(self, key: str) -> Optional[Any]:
        import json
        value = self.redis.get(key)
        return json.loads(value) if value else None
    
    def set(self, key: str, value: Any, ttl: int = 3600) -> bool:
        import json
        return self.redis.setex(key, ttl, json.dumps(value))
    
    def delete(self, key: str) -> bool:
        return bool(self.redis.delete(key))
    
    def exists(self, key: str) -> bool:
        return bool(self.redis.exists(key))
    
    def clear(self) -> bool:
        self.redis.flushdb()
        return True

class MemoryCache(Cache):
    def __init__(self):
        self._cache = {}
        self._ttl = {}
    
    def get(self, key: str) -> Optional[Any]:
        import time
        if key in self._cache:
            if self._ttl[key] > time.time():
                return self._cache[key]
            else:
                del self._cache[key]
                del self._ttl[key]
        return None
    
    def set(self, key: str, value: Any, ttl: int = 3600) -> bool:
        import time
        self._cache[key] = value
        self._ttl[key] = time.time() + ttl
        return True
    
    def delete(self, key: str) -> bool:
        if key in self._cache:
            del self._cache[key]
            del self._ttl[key]
            return True
        return False
    
    def exists(self, key: str) -> bool:
        return self.get(key) is not None
    
    def clear(self) -> bool:
        self._cache.clear()
        self._ttl.clear()
        return True
```

## ❌ Common Mistakes

### 1. Forgetting @abstractmethod Decorator

```python
# WRONG - Missing @abstractmethod
class Wrong(ABC):
    def required_method(self):  # Not abstract!
        pass

class Implementation(Wrong):
    pass  # This works but shouldn't!

# CORRECT
class Correct(ABC):
    @abstractmethod
    def required_method(self):
        pass

# class BadImpl(Correct):
#     pass  # TypeError: Can't instantiate
```

### 2. Implementing Abstract Method Incorrectly

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        """Calculate area"""
        pass

# WRONG - Different signature
class BadRectangle(Shape):
    def area(self, precision: int = 2) -> float:  # Different signature!
        return 10.0

# CORRECT - Same signature
class GoodRectangle(Shape):
    def area(self) -> float:
        return self.width * self.height
```

### 3. Providing Implementation in Abstract Method

```python
# BAD PRACTICE - Don't provide default implementation
class Animal(ABC):
    @abstractmethod
    def make_sound(self):
        return "Generic sound"  # BAD!

# BETTER - Use separate method
class Animal(ABC):
    @abstractmethod
    def make_sound(self):
        """Subclass must implement"""
        pass
    
    def speak(self):
        """Common method using abstract method"""
        sound = self.make_sound()
        return f"The animal says: {sound}"
```

## 🔐 Security Considerations

### Secure Authentication Interface

```python
from abc import ABC, abstractmethod
from typing import Optional, Dict

class AuthenticationBackend(ABC):
    """Abstract authentication interface"""
    
    @abstractmethod
    def authenticate(self, credentials: Dict) -> Optional[Dict]:
        """Authenticate user and return user data"""
        pass
    
    @abstractmethod
    def validate_token(self, token: str) -> bool:
        """Validate authentication token"""
        pass
    
    @abstractmethod
    def revoke_token(self, token: str) -> bool:
        """Revoke authentication token"""
        pass

class JWTAuthentication(AuthenticationBackend):
    def __init__(self, secret_key: str):
        self.secret_key = secret_key
    
    def authenticate(self, credentials: Dict) -> Optional[Dict]:
        import jwt
        from django.contrib.auth import authenticate
        
        user = authenticate(
            username=credentials.get('username'),
            password=credentials.get('password')
        )
        
        if user:
            token = jwt.encode(
                {'user_id': user.id, 'username': user.username},
                self.secret_key,
                algorithm='HS256'
            )
            return {'user': user, 'token': token}
        return None
    
    def validate_token(self, token: str) -> bool:
        import jwt
        try:
            jwt.decode(token, self.secret_key, algorithms=['HS256'])
            return True
        except jwt.InvalidTokenError:
            return False
    
    def revoke_token(self, token: str) -> bool:
        # Add to blacklist in Redis
        return True

class OAuth2Authentication(AuthenticationBackend):
    def authenticate(self, credentials: Dict) -> Optional[Dict]:
        # OAuth2 flow
        pass
    
    def validate_token(self, token: str) -> bool:
        # Validate OAuth2 token
        pass
    
    def revoke_token(self, token: str) -> bool:
        # Revoke OAuth2 token
        pass
```

## 🚀 Performance Optimization

### Lazy Loading with ABC

```python
from abc import ABC, abstractmethod

class DataLoader(ABC):
    """Abstract data loader with lazy loading"""
    
    def __init__(self):
        self._cache = None
    
    @abstractmethod
    def _fetch_data(self):
        """Fetch data from source"""
        pass
    
    def get_data(self):
        """Get data with lazy loading"""
        if self._cache is None:
            self._cache = self._fetch_data()
        return self._cache
    
    def refresh(self):
        """Force refresh"""
        self._cache = None
        return self.get_data()

class DatabaseLoader(DataLoader):
    def __init__(self, connection):
        super().__init__()
        self.connection = connection
    
    def _fetch_data(self):
        cursor = self.connection.cursor()
        cursor.execute("SELECT * FROM data")
        return cursor.fetchall()

class APILoader(DataLoader):
    def __init__(self, url):
        super().__init__()
        self.url = url
    
    def _fetch_data(self):
        import requests
        response = requests.get(self.url)
        return response.json()
```

## 🏗️ Real-World Django Example

```python
# services/base.py
from abc import ABC, abstractmethod
from typing import List, Optional
from django.db.models import QuerySet

class BaseService(ABC):
    """Abstract service layer"""
    
    @abstractmethod
    def get_all(self) -> QuerySet:
        pass
    
    @abstractmethod
    def get_by_id(self, id: int) -> Optional[object]:
        pass
    
    @abstractmethod
    def create(self, **kwargs) -> object:
        pass
    
    @abstractmethod
    def update(self, id: int, **kwargs) -> object:
        pass
    
    @abstractmethod
    def delete(self, id: int) -> bool:
        pass

# services/user_service.py
from django.contrib.auth.models import User
from .base import BaseService

class UserService(BaseService):
    def get_all(self) -> QuerySet:
        return User.objects.all()
    
    def get_by_id(self, id: int) -> Optional[User]:
        try:
            return User.objects.get(id=id)
        except User.DoesNotExist:
            return None
    
    def create(self, **kwargs) -> User:
        return User.objects.create_user(**kwargs)
    
    def update(self, id: int, **kwargs) -> User:
        user = self.get_by_id(id)
        for key, value in kwargs.items():
            setattr(user, key, value)
        user.save()
        return user
    
    def delete(self, id: int) -> bool:
        user = self.get_by_id(id)
        if user:
            user.delete()
            return True
        return False
```

## ❓ Interview Questions

### Q1: What's the difference between ABC and regular inheritance?

**Answer**: ABC enforces that subclasses implement specific methods. Regular inheritance doesn't force implementation. ABCs use `@abstractmethod` to mark required methods, and attempting to instantiate a class without implementing them raises `TypeError`.

### Q2: When should you use abstract base classes?

**Answer**: Use ABCs when:
- Defining interfaces/contracts
- Building frameworks/libraries
- Multiple implementations needed (payment gateways, storage backends)
- Enforcing implementation of specific methods
- Providing common functionality with required overrides

### Q3: Can abstract methods have implementations?

**Answer**: Yes, in Python 3. The abstract method can have a default implementation that subclasses can call via `super()`:

```python
class Base(ABC):
    @abstractmethod
    def method(self):
        print("Default implementation")

class Derived(Base):
    def method(self):
        super().method()
        print("Extended")
```

### Q4: What's the difference between ABC and Protocol (Python 3.8+)?

**Answer**:
- **ABC**: Explicit inheritance required, runtime checks
- **Protocol**: Structural subtyping (duck typing), no inheritance needed

```python
# ABC - explicit
class Animal(ABC):
    @abstractmethod
    def speak(self): pass

class Dog(Animal):  # Must inherit
    def speak(self): return "Woof"

# Protocol - structural
from typing import Protocol

class Speaker(Protocol):
    def speak(self) -> str: ...

class Cat:  # No inheritance needed
    def speak(self): return "Meow"

def make_speak(animal: Speaker):
    print(animal.speak())

make_speak(Cat())  # Works!
```

### Q5: How do you register virtual subclasses?

**Answer**: Use `register()` to make a class a virtual subclass without inheritance:

```python
from abc import ABC

class Base(ABC):
    pass

class External:
    pass

Base.register(External)  # Now External is considered subclass
print(issubclass(External, Base))  # True
```

### Q6: Can you have abstract class methods or static methods?

**Answer**: Yes!

```python
from abc import ABC, abstractmethod

class Base(ABC):
    @classmethod
    @abstractmethod
    def class_method(cls):
        pass
    
    @staticmethod
    @abstractmethod
    def static_method():
        pass
```

## 📚 Summary

Abstract Base Classes provide:
- Interface definition and enforcement
- Multiple implementations with same contract
- Framework and library design patterns
- Type safety and documentation

**Key Takeaways**:
1. Use `@abstractmethod` to mark required methods
2. Subclasses must implement all abstract methods
3. Can't instantiate abstract classes directly
4. Perfect for defining interfaces (repositories, gateways, services)
5. Essential for SOLID principles (Dependency Inversion)
6. Use for polymorphic designs in production systems

ABCs are fundamental to building maintainable, testable backend systems with Django and FastAPI.
