# Protocol Classes - Structural Subtyping (Duck Typing)

## 📖 Concept Explanation

**Protocol** (PEP 544, Python 3.8+) provides **structural subtyping** (duck typing with type checking) rather than nominal subtyping (explicit inheritance).

### Nominal vs Structural Subtyping

**Nominal Subtyping** (ABC, traditional inheritance):

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self): pass

class Dog(Animal):  # Must explicitly inherit
    def speak(self): return "Woof"
```

**Structural Subtyping** (Protocol):

```python
from typing import Protocol

class Speakable(Protocol):
    def speak(self) -> str: ...

class Dog:  # No inheritance needed!
    def speak(self) -> str: return "Woof"

def make_speak(animal: Speakable):
    print(animal.speak())

make_speak(Dog())  # Type checker accepts this!
```

### Key Differences

| Feature       | ABC (Nominal)      | Protocol (Structural) |
| ------------- | ------------------ | --------------------- |
| Inheritance   | Required           | Not required          |
| Type checking | Runtime + static   | Primarily static      |
| Flexibility   | Less flexible      | More flexible         |
| Duck typing   | No                 | Yes                   |
| Use case      | Explicit contracts | Implicit interfaces   |

## 🧠 Why Protocols Matter

### Benefits

1. **Backward Compatibility**: Add type hints to existing code without changes
2. **Flexibility**: No need to modify third-party classes
3. **Duck Typing with Type Safety**: Best of both worlds
4. **Interface Segregation**: Define minimal required interfaces

### Real-World Use Cases

- Type-checking existing classes without refactoring
- Defining interfaces for third-party libraries
- Gradual typing migration
- Plugin systems
- Mock-friendly testing

## 🔧 Basic Protocol Usage

### Defining a Protocol

```python
from typing import Protocol

class Drawable(Protocol):
    """Any object with a draw() method"""
    def draw(self) -> str:
        ...

class Circle:
    def draw(self) -> str:
        return "Drawing circle"

class Square:
    def draw(self) -> str:
        return "Drawing square"

def render(shape: Drawable):
    print(shape.draw())

# Both work without inheriting from Drawable!
render(Circle())  # "Drawing circle"
render(Square())  # "Drawing square"

# Type checker validates at compile time
# render("string")  # Type error!
```

### Protocol with Multiple Methods

```python
from typing import Protocol

class Persistable(Protocol):
    def save(self) -> None: ...
    def load(self, id: int) -> None: ...
    def delete(self) -> None: ...

class User:
    def save(self) -> None:
        print("Saving user")

    def load(self, id: int) -> None:
        print(f"Loading user {id}")

    def delete(self) -> None:
        print("Deleting user")

class Product:
    def save(self) -> None:
        print("Saving product")

    def load(self, id: int) -> None:
        print(f"Loading product {id}")

    def delete(self) -> None:
        print("Deleting product")

def persist_entity(entity: Persistable):
    entity.save()

persist_entity(User())    # Works!
persist_entity(Product()) # Works!
```

### Protocol with Properties

```python
from typing import Protocol

class Named(Protocol):
    @property
    def name(self) -> str: ...

class Person:
    def __init__(self, name: str):
        self._name = name

    @property
    def name(self) -> str:
        return self._name

class Company:
    def __init__(self, name: str):
        self._name = name

    @property
    def name(self) -> str:
        return self._name

def print_name(obj: Named):
    print(f"Name: {obj.name}")

print_name(Person("John"))      # Works!
print_name(Company("Acme Inc")) # Works!
```

## 🏗️ Runtime Checkable Protocols

### @runtime_checkable Decorator

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

class File:
    def close(self) -> None:
        print("Closing file")

class Database:
    def close(self) -> None:
        print("Closing database")

# Runtime type checking with isinstance
file = File()
print(isinstance(file, Closeable))  # True

db = Database()
print(isinstance(db, Closeable))  # True

# Can use in dynamic code
def close_resource(resource):
    if isinstance(resource, Closeable):
        resource.close()
    else:
        print("Resource not closeable")

close_resource(file)  # "Closing file"
close_resource("string")  # "Resource not closeable"
```

## 🚀 Django Examples

### Repository Protocol

```python
from typing import Protocol, List, Optional
from django.db.models import Model

class Repository(Protocol):
    """Protocol for any repository implementation"""

    def get_by_id(self, id: int) -> Optional[Model]:
        ...

    def get_all(self) -> List[Model]:
        ...

    def save(self, entity: Model) -> Model:
        ...

    def delete(self, id: int) -> bool:
        ...

# Django implementation
class DjangoUserRepository:
    def get_by_id(self, id: int) -> Optional[Model]:
        from django.contrib.auth.models import User
        try:
            return User.objects.get(id=id)
        except User.DoesNotExist:
            return None

    def get_all(self) -> List[Model]:
        from django.contrib.auth.models import User
        return list(User.objects.all())

    def save(self, entity: Model) -> Model:
        entity.save()
        return entity

    def delete(self, id: int) -> bool:
        from django.contrib.auth.models import User
        deleted, _ = User.objects.filter(id=id).delete()
        return deleted > 0

# Service layer uses protocol
class UserService:
    def __init__(self, repository: Repository):
        self.repository = repository

    def create_user(self, username: str, email: str):
        from django.contrib.auth.models import User
        user = User(username=username, email=email)
        return self.repository.save(user)

    def get_user(self, user_id: int):
        return self.repository.get_by_id(user_id)

# Can swap implementations easily
service = UserService(DjangoUserRepository())
```

### Cache Protocol

```python
from typing import Protocol, Optional, Any
import pickle

class Cache(Protocol):
    def get(self, key: str) -> Optional[Any]: ...
    def set(self, key: str, value: Any, ttl: int = 3600) -> bool: ...
    def delete(self, key: str) -> bool: ...
    def clear(self) -> bool: ...

# Redis implementation
class RedisCache:
    def __init__(self):
        import redis
        self.client = redis.Redis(host='localhost', port=6379)

    def get(self, key: str) -> Optional[Any]:
        data = self.client.get(key)
        return pickle.loads(data) if data else None

    def set(self, key: str, value: Any, ttl: int = 3600) -> bool:
        return self.client.setex(key, ttl, pickle.dumps(value))

    def delete(self, key: str) -> bool:
        return bool(self.client.delete(key))

    def clear(self) -> bool:
        self.client.flushdb()
        return True

# Memory implementation
class MemoryCache:
    def __init__(self):
        self._cache = {}

    def get(self, key: str) -> Optional[Any]:
        return self._cache.get(key)

    def set(self, key: str, value: Any, ttl: int = 3600) -> bool:
        self._cache[key] = value
        return True

    def delete(self, key: str) -> bool:
        return self._cache.pop(key, None) is not None

    def clear(self) -> bool:
        self._cache.clear()
        return True

# Service uses protocol
class CachedUserService:
    def __init__(self, cache: Cache):
        self.cache = cache

    def get_user(self, user_id: int):
        cache_key = f"user:{user_id}"

        # Try cache
        cached = self.cache.get(cache_key)
        if cached:
            return cached

        # Fetch from database
        from django.contrib.auth.models import User
        user = User.objects.get(id=user_id)

        # Cache result
        self.cache.set(cache_key, user, ttl=600)

        return user

# Can use either implementation
service = CachedUserService(RedisCache())
# or
service = CachedUserService(MemoryCache())
```

### Notification Protocol

```python
from typing import Protocol

class Notifier(Protocol):
    def send(self, recipient: str, message: str) -> bool:
        ...

class EmailNotifier:
    def send(self, recipient: str, message: str) -> bool:
        from django.core.mail import send_mail
        try:
            send_mail(
                subject="Notification",
                message=message,
                from_email="noreply@example.com",
                recipient_list=[recipient]
            )
            return True
        except:
            return False

class SMSNotifier:
    def send(self, recipient: str, message: str) -> bool:
        # Twilio integration
        print(f"SMS to {recipient}: {message}")
        return True

class PushNotifier:
    def send(self, recipient: str, message: str) -> bool:
        # Firebase integration
        print(f"Push to {recipient}: {message}")
        return True

# Service accepts any notifier
class NotificationService:
    def __init__(self, notifier: Notifier):
        self.notifier = notifier

    def notify_user(self, user_id: int, message: str):
        from django.contrib.auth.models import User
        user = User.objects.get(id=user_id)

        return self.notifier.send(user.email, message)

# Flexible - can use any notifier
service = NotificationService(EmailNotifier())
# or
service = NotificationService(SMSNotifier())
```

## 🎯 FastAPI Examples

### Dependency Injection with Protocols

```python
from fastapi import FastAPI, Depends
from typing import Protocol, Optional

app = FastAPI()

class UserRepository(Protocol):
    def get_user(self, user_id: int) -> Optional[dict]:
        ...

    def create_user(self, username: str, email: str) -> dict:
        ...

# Implementation 1: Database
class DatabaseUserRepository:
    def get_user(self, user_id: int) -> Optional[dict]:
        # Query database
        return {"id": user_id, "username": "john"}

    def create_user(self, username: str, email: str) -> dict:
        # Insert into database
        return {"id": 1, "username": username, "email": email}

# Implementation 2: In-memory (for testing)
class InMemoryUserRepository:
    def __init__(self):
        self.users = {}
        self.next_id = 1

    def get_user(self, user_id: int) -> Optional[dict]:
        return self.users.get(user_id)

    def create_user(self, username: str, email: str) -> dict:
        user = {
            "id": self.next_id,
            "username": username,
            "email": email
        }
        self.users[self.next_id] = user
        self.next_id += 1
        return user

# Dependency
def get_repository() -> UserRepository:
    # Can switch implementations based on config
    return DatabaseUserRepository()

@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    repo: UserRepository = Depends(get_repository)
):
    user = repo.get_user(user_id)
    return user or {"error": "User not found"}

@app.post("/users")
async def create_user(
    username: str,
    email: str,
    repo: UserRepository = Depends(get_repository)
):
    return repo.create_user(username, email)
```

### Authentication Protocol

```python
from typing import Protocol, Optional
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

class Authenticator(Protocol):
    def authenticate(self, token: str) -> Optional[dict]:
        ...

class JWTAuthenticator:
    def authenticate(self, token: str) -> Optional[dict]:
        import jwt
        try:
            payload = jwt.decode(token, "secret", algorithms=["HS256"])
            return payload
        except jwt.InvalidTokenError:
            return None

class OAuth2Authenticator:
    def authenticate(self, token: str) -> Optional[dict]:
        # Validate OAuth2 token
        return {"user_id": 1, "scope": "read write"}

# Dependency
def get_authenticator() -> Authenticator:
    return JWTAuthenticator()

def get_current_user(
    token: str,
    authenticator: Authenticator = Depends(get_authenticator)
):
    user = authenticator.authenticate(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user

@app.get("/protected")
async def protected_route(user: dict = Depends(get_current_user)):
    return {"message": f"Hello user {user['user_id']}"}
```

## ✅ Best Practices

### 1. Keep Protocols Small

```python
# GOOD - Small, focused protocols
class Readable(Protocol):
    def read(self) -> str: ...

class Writable(Protocol):
    def write(self, data: str) -> None: ...

# Compose when needed
class File:
    def read(self) -> str: ...
    def write(self, data: str) -> None: ...

def process_file(file: Readable):  # Only needs read
    data = file.read()
    return data.upper()
```

### 2. Use @runtime_checkable Sparingly

```python
# Use @runtime_checkable only when needed at runtime
from typing import Protocol, runtime_checkable

@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

def cleanup(resources: list):
    for resource in resources:
        if isinstance(resource, Closeable):  # Runtime check
            resource.close()
```

### 3. Protocol for Testing

```python
from typing import Protocol

class EmailSender(Protocol):
    def send(self, to: str, subject: str, body: str) -> bool:
        ...

# Production implementation
class SMTPEmailSender:
    def send(self, to: str, subject: str, body: str) -> bool:
        # Real SMTP sending
        return True

# Test implementation
class MockEmailSender:
    def __init__(self):
        self.sent_emails = []

    def send(self, to: str, subject: str, body: str) -> bool:
        self.sent_emails.append((to, subject, body))
        return True

# Service uses protocol
class UserRegistrationService:
    def __init__(self, email_sender: EmailSender):
        self.email_sender = email_sender

    def register(self, email: str):
        # Register user...
        self.email_sender.send(email, "Welcome", "Welcome to our service!")

# Test
def test_registration():
    mock = MockEmailSender()
    service = UserRegistrationService(mock)

    service.register("user@example.com")

    assert len(mock.sent_emails) == 1
    assert mock.sent_emails[0][0] == "user@example.com"
```

## ⚡ Performance Considerations

```python
# Protocols have minimal runtime overhead
from typing import Protocol
import timeit

class Addable(Protocol):
    def add(self, x: int, y: int) -> int: ...

class Calculator:
    def add(self, x: int, y: int) -> int:
        return x + y

# Protocol does not affect runtime performance
calc = Calculator()

# Same performance whether typed or not
def with_protocol(calc: Addable):
    return calc.add(1, 2)

def without_protocol(calc):
    return calc.add(1, 2)

# Both have same runtime performance
time1 = timeit.timeit(lambda: with_protocol(calc), number=100000)
time2 = timeit.timeit(lambda: without_protocol(calc), number=100000)

print(f"With protocol: {time1:.4f}s")
print(f"Without protocol: {time2:.4f}s")
# Nearly identical performance
```

## ❌ Common Mistakes

### 1. Using Protocol When ABC is Better

```python
# WRONG - Use ABC when you control all implementations
from typing import Protocol

class Animal(Protocol):  # Should be ABC!
    def speak(self) -> str: ...

# CORRECT - Use ABC for explicit contracts
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self) -> str: ...
```

### 2. Forgetting @runtime_checkable

```python
from typing import Protocol

class Closeable(Protocol):
    def close(self) -> None: ...

# WRONG - Can't use isinstance without @runtime_checkable
# isinstance(obj, Closeable)  # TypeError

# CORRECT
from typing import runtime_checkable

@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

isinstance(obj, Closeable)  # Works!
```

### 3. Protocols with Implementation

```python
# WRONG - Protocols should not have implementation
class Wrong(Protocol):
    def method(self) -> str:
        return "implementation"  # BAD!

# CORRECT - Use ... or pass
class Correct(Protocol):
    def method(self) -> str: ...
```

## ❓ Interview Questions

### Q1: Protocol vs ABC - when to use each?

**Answer**:

- **Protocol**: Duck typing, third-party classes, no inheritance needed, gradual typing
- **ABC**: Explicit contracts, you control implementations, shared behavior, runtime enforcement

### Q2: Are Protocols checked at runtime?

**Answer**: By default, no. Protocols are primarily for static type checkers (mypy, pyright). Use `@runtime_checkable` for runtime isinstance checks.

### Q3: Can a class explicitly implement a Protocol?

**Answer**: No need! If a class has the required methods, it automatically satisfies the Protocol. This is structural subtyping.

```python
class MyProtocol(Protocol):
    def method(self): ...

class MyClass:  # No inheritance needed
    def method(self): ...

# MyClass satisfies MyProtocol structurally
```

## 📚 Summary

**Protocols** (PEP 544):

- ✅ Structural subtyping (duck typing with type safety)
- ✅ No inheritance required
- ✅ Static type checking
- ✅ Backward compatible
- ✅ Flexible, testable

**When to Use**:

- Type-checking existing code
- Third-party library interfaces
- Plugin systems
- Testing with mocks
- Gradual typing migration

**Protocol vs ABC**:

- Protocol → Structural, implicit, flexible
- ABC → Nominal, explicit, enforced

Essential for modern Python type hints in Django and FastAPI projects!
