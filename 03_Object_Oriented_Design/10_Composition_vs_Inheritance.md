# Composition vs Inheritance - "Has-A" vs "Is-A"

## 📖 Concept Explanation

Two fundamental ways to achieve code reuse and establish relationships between classes:

- **Inheritance (Is-A)**: Class inherits from another class
- **Composition (Has-A)**: Class contains instances of other classes

### Key Principle
**"Favor composition over inheritance"** - Gang of Four Design Patterns

### Why Composition Over Inheritance?

1. **Flexibility**: Change behavior at runtime
2. **Loose Coupling**: Less dependent on parent classes
3. **No Fragile Base Class Problem**: Changes to parent don't break children
4. **Multiple Behaviors**: Combine multiple behaviors easily
5. **Interface Segregation**: Only expose what's needed

## 🧠 When to Use Each

### Use Inheritance When:
- True "is-a" relationship exists
- Shared behavior across hierarchy
- Polymorphism needed
- Template Method pattern

### Use Composition When:
- "Has-a" or "uses-a" relationship
- Need to change behavior at runtime
- Want to avoid deep hierarchies
- Need multiple behaviors from different sources

## ⚔️ Inheritance Example

```python
# Inheritance hierarchy
class Animal:
    def __init__(self, name: str):
        self.name = name
    
    def eat(self):
        return f"{self.name} is eating"
    
    def sleep(self):
        return f"{self.name} is sleeping"

class Dog(Animal):
    def bark(self):
        return f"{self.name} is barking"

class Cat(Animal):
    def meow(self):
        return f"{self.name} is meowing"

# Usage
dog = Dog("Buddy")
print(dog.eat())   # Inherited
print(dog.bark())  # Specific to Dog

cat = Cat("Whiskers")
print(cat.eat())   # Inherited
print(cat.meow())  # Specific to Cat
```

### Problems with Deep Inheritance

```python
# PROBLEMATIC - Deep hierarchy
class Animal:
    def move(self):
        pass

class Mammal(Animal):
    def give_birth(self):
        pass

class Carnivore(Mammal):
    def hunt(self):
        pass

class Dog(Carnivore):
    def bark(self):
        pass

# What about birds? They move differently
# What about platypus? It's a mammal that lays eggs!
# What about omnivores?

# This hierarchy becomes fragile and hard to maintain
```

## 🧩 Composition Example

```python
# Composition approach
class Walker:
    def walk(self):
        return "Walking on legs"

class Swimmer:
    def swim(self):
        return "Swimming in water"

class Flyer:
    def fly(self):
        return "Flying in air"

class Barker:
    def bark(self):
        return "Woof!"

class Meower:
    def meow(self):
        return "Meow!"

# Compose behaviors
class Dog:
    def __init__(self, name: str):
        self.name = name
        self.walker = Walker()
        self.swimmer = Swimmer()
        self.barker = Barker()
    
    def move_on_land(self):
        return self.walker.walk()
    
    def move_in_water(self):
        return self.swimmer.swim()
    
    def make_sound(self):
        return self.barker.bark()

class Bird:
    def __init__(self, name: str):
        self.name = name
        self.walker = Walker()
        self.flyer = Flyer()
    
    def move_on_land(self):
        return self.walker.walk()
    
    def move_in_air(self):
        return self.flyer.fly()

class Duck:
    def __init__(self, name: str):
        self.name = name
        self.walker = Walker()
        self.swimmer = Swimmer()
        self.flyer = Flyer()
    
    def move_on_land(self):
        return self.walker.walk()
    
    def move_in_water(self):
        return self.swimmer.swim()
    
    def move_in_air(self):
        return self.flyer.fly()

# Usage
dog = Dog("Buddy")
print(dog.move_on_land())   # "Walking on legs"
print(dog.move_in_water())  # "Swimming in water"
print(dog.make_sound())     # "Woof!"

duck = Duck("Donald")
print(duck.move_on_land())  # "Walking on legs"
print(duck.move_in_water()) # "Swimming in water"
print(duck.move_in_air())   # "Flying in air"
```

## 🏗️ Real-World Django Examples

### Bad: Deep Inheritance Hierarchy

```python
# models.py - PROBLEMATIC
class BaseUser(models.Model):
    username = models.CharField(max_length=100)
    email = models.EmailField()
    
    class Meta:
        abstract = True

class PaidUser(BaseUser):
    subscription_date = models.DateField()
    
    class Meta:
        abstract = True

class PremiumUser(PaidUser):
    premium_features = models.JSONField()
    
    class Meta:
        abstract = True

class EnterpriseUser(PremiumUser):
    company_name = models.CharField(max_length=200)
    # Now we have 4 levels deep!
    # What if we need a free trial user with some premium features?
```

### Good: Composition with Mixins

```python
# models.py - BETTER
from django.db import models

class User(models.Model):
    username = models.CharField(max_length=100)
    email = models.EmailField()

class Subscription(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    tier = models.CharField(max_length=20)  # free, premium, enterprise
    start_date = models.DateField()
    end_date = models.DateField(null=True)

class PremiumFeatures(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    feature_flags = models.JSONField(default=dict)
    storage_limit_gb = models.IntegerField(default=10)

class EnterpriseSettings(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    company_name = models.CharField(max_length=200)
    sso_enabled = models.BooleanField(default=False)
    custom_domain = models.CharField(max_length=100, null=True)

# Service layer handles composition
class UserService:
    @staticmethod
    def get_user_capabilities(user: User):
        capabilities = ['basic_access']
        
        # Check subscription
        try:
            subscription = user.subscription
            if subscription.tier in ['premium', 'enterprise']:
                capabilities.append('premium_features')
        except Subscription.DoesNotExist:
            pass
        
        # Check enterprise features
        try:
            enterprise = user.enterprisesettings
            capabilities.append('enterprise_features')
        except EnterpriseSettings.DoesNotExist:
            pass
        
        return capabilities
```

### Django Mixins (Best of Both)

```python
# mixins.py
from django.views import View
from django.http import JsonResponse

class JSONResponseMixin:
    def render_json(self, data):
        return JsonResponse(data)

class AuthRequiredMixin:
    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return JsonResponse({'error': 'Unauthorized'}, status=401)
        return super().dispatch(request, *args, **kwargs)

class RateLimitMixin:
    rate_limit = 100  # requests per minute
    
    def dispatch(self, request, *args, **kwargs):
        # Check rate limit
        if self.is_rate_limited(request):
            return JsonResponse({'error': 'Rate limit exceeded'}, status=429)
        return super().dispatch(request, *args, **kwargs)
    
    def is_rate_limited(self, request):
        # Check rate limit logic
        return False

# Compose behaviors with multiple mixins
class ProtectedAPIView(AuthRequiredMixin, RateLimitMixin, JSONResponseMixin, View):
    def get(self, request):
        return self.render_json({'message': 'Hello'})

# Different combination
class PublicAPIView(RateLimitMixin, JSONResponseMixin, View):
    def get(self, request):
        return self.render_json({'message': 'Public data'})
```

## 🚀 FastAPI Composition Examples

### Dependency Injection (Composition)

```python
from fastapi import FastAPI, Depends
from abc import ABC, abstractmethod

# Abstract components
class CacheInterface(ABC):
    @abstractmethod
    def get(self, key: str): pass
    
    @abstractmethod
    def set(self, key: str, value): pass

class DatabaseInterface(ABC):
    @abstractmethod
    def query(self, sql: str): pass

# Concrete implementations
class RedisCache(CacheInterface):
    def get(self, key: str):
        # Redis implementation
        return None
    
    def set(self, key: str, value):
        pass

class PostgresDatabase(DatabaseInterface):
    def query(self, sql: str):
        # Postgres implementation
        return []

# Service using composition
class UserService:
    def __init__(self, cache: CacheInterface, db: DatabaseInterface):
        self.cache = cache
        self.db = db
    
    def get_user(self, user_id: int):
        # Try cache first
        cached = self.cache.get(f"user:{user_id}")
        if cached:
            return cached
        
        # Query database
        result = self.db.query(f"SELECT * FROM users WHERE id = {user_id}")
        
        # Cache result
        self.cache.set(f"user:{user_id}", result)
        
        return result

# Dependency injection
def get_cache() -> CacheInterface:
    return RedisCache()

def get_database() -> DatabaseInterface:
    return PostgresDatabase()

def get_user_service(
    cache: CacheInterface = Depends(get_cache),
    db: DatabaseInterface = Depends(get_database)
) -> UserService:
    return UserService(cache, db)

# Route using composed service
app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
):
    return service.get_user(user_id)
```

## 🎯 Strategy Pattern (Composition over Inheritance)

### Bad: Inheritance for Different Behaviors

```python
# PROBLEMATIC - Using inheritance
class PaymentProcessor:
    def process(self, amount):
        raise NotImplementedError

class CreditCardProcessor(PaymentProcessor):
    def process(self, amount):
        return f"Processing ${amount} via credit card"

class PayPalProcessor(PaymentProcessor):
    def process(self, amount):
        return f"Processing ${amount} via PayPal"

# What if we need to switch payment methods at runtime?
# What if one order uses multiple payment methods?
```

### Good: Composition with Strategy

```python
# BETTER - Using composition
from abc import ABC, abstractmethod
from decimal import Decimal

class PaymentMethod(ABC):
    @abstractmethod
    def charge(self, amount: Decimal) -> dict:
        pass

class CreditCard(PaymentMethod):
    def __init__(self, card_number: str):
        self.card_number = card_number
    
    def charge(self, amount: Decimal) -> dict:
        return {'method': 'credit_card', 'amount': amount}

class PayPal(PaymentMethod):
    def __init__(self, email: str):
        self.email = email
    
    def charge(self, amount: Decimal) -> dict:
        return {'method': 'paypal', 'amount': amount}

class BankTransfer(PaymentMethod):
    def __init__(self, account_number: str):
        self.account_number = account_number
    
    def charge(self, amount: Decimal) -> dict:
        return {'method': 'bank_transfer', 'amount': amount}

# Order uses composition
class Order:
    def __init__(self, order_id: int):
        self.order_id = order_id
        self.payment_methods = []
    
    def add_payment_method(self, method: PaymentMethod):
        self.payment_methods.append(method)
    
    def process_payment(self, amount: Decimal):
        results = []
        for method in self.payment_methods:
            result = method.charge(amount)
            results.append(result)
        return results

# Usage - Flexible!
order = Order(order_id=123)

# Can use multiple payment methods
order.add_payment_method(CreditCard("4111-1111-1111-1111"))
order.add_payment_method(PayPal("user@example.com"))

# Can switch at runtime
order.payment_methods.clear()
order.add_payment_method(BankTransfer("123456789"))

results = order.process_payment(Decimal('99.99'))
```

## 🔧 Delegation Pattern

```python
# Composition through delegation
class Engine:
    def start(self):
        return "Engine started"
    
    def stop(self):
        return "Engine stopped"

class GPS:
    def navigate(self, destination: str):
        return f"Navigating to {destination}"

class MusicPlayer:
    def play(self, song: str):
        return f"Playing {song}"

# Car delegates to composed objects
class Car:
    def __init__(self):
        self.engine = Engine()
        self.gps = GPS()
        self.music_player = MusicPlayer()
    
    # Delegate to engine
    def start(self):
        return self.engine.start()
    
    def stop(self):
        return self.engine.stop()
    
    # Delegate to GPS
    def navigate_to(self, destination: str):
        return self.gps.navigate(destination)
    
    # Delegate to music player
    def play_music(self, song: str):
        return self.music_player.play(song)

# Usage
car = Car()
print(car.start())  # Delegates to engine
print(car.navigate_to("Home"))  # Delegates to GPS
print(car.play_music("Bohemian Rhapsody"))  # Delegates to music player
```

## ✅ Best Practices

### 1. Use Mixins for Cross-Cutting Concerns

```python
# Django REST Framework approach
from rest_framework import viewsets, mixins

class UserViewSet(
    mixins.CreateModelMixin,
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    mixins.ListModelMixin,
    viewsets.GenericViewSet
):
    """Compose only needed behaviors"""
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

### 2. Interface Segregation

```python
# WRONG - Fat interface
class Worker:
    def work(self): pass
    def eat(self): pass
    def sleep(self): pass

# CORRECT - Segregated interfaces
class Workable:
    def work(self): pass

class Eatable:
    def eat(self): pass

class Sleepable:
    def sleep(self): pass

# Compose as needed
class Human:
    def __init__(self):
        self.work_ability = Workable()
        self.eat_ability = Eatable()
        self.sleep_ability = Sleepable()

class Robot:
    def __init__(self):
        self.work_ability = Workable()
        # No eat or sleep!
```

### 3. Dependency Injection

```python
# Constructor injection
class UserService:
    def __init__(self, repository, cache, logger):
        self.repository = repository
        self.cache = cache
        self.logger = logger
    
    def get_user(self, user_id: int):
        # Use injected dependencies
        pass

# Easy to test with mocks
def test_user_service():
    mock_repo = MockRepository()
    mock_cache = MockCache()
    mock_logger = MockLogger()
    
    service = UserService(mock_repo, mock_cache, mock_logger)
    # Test...
```

## ❌ Common Mistakes

### 1. Using Inheritance for Code Reuse

```python
# WRONG - Inheritance just for code reuse
class Utils:
    def format_date(self, date):
        pass

class UserService(Utils):  # BAD! Not an "is-a" relationship
    pass

# CORRECT - Composition
class UserService:
    def __init__(self):
        self.utils = Utils()  # Has-a relationship
```

### 2. Multiple Inheritance Diamond Problem

```python
# PROBLEMATIC
class A:
    def method(self):
        print("A")

class B(A):
    def method(self):
        print("B")

class C(A):
    def method(self):
        print("C")

class D(B, C):  # Diamond problem!
    pass

d = D()
d.method()  # Which method? B or C?

# BETTER - Use composition
class D:
    def __init__(self):
        self.b = B()
        self.c = C()
```

## 📚 Summary

### Composition Advantages:
✅ Runtime flexibility
✅ Loose coupling
✅ Easy testing
✅ Multiple behaviors
✅ No fragile base class

### Inheritance Advantages:
✅ Natural polymorphism
✅ Code reuse
✅ Clear hierarchies
✅ Template Method pattern

### Recommendation:
**Default to composition**. Use inheritance only for true "is-a" relationships and when polymorphism is essential.

### Golden Rule:
"Favor composition over inheritance, but use inheritance when it makes sense."

In Django and FastAPI, use mixins, dependency injection, and service layers to achieve flexible, maintainable designs.
