# Descriptors and Properties - Attribute Access Control

## 📖 Concept Explanation

Descriptors are Python objects that define how attributes are accessed, modified, or deleted. They implement the **Descriptor Protocol** through special methods: `__get__()`, `__set__()`, and `__delete__()`.

### The Descriptor Protocol

```python
class Descriptor:
    def __get__(self, instance, owner):
        """Called when attribute is accessed"""
        print(f"__get__ called on {instance} from {owner}")
        return "value"
    
    def __set__(self, instance, value):
        """Called when attribute is assigned"""
        print(f"__set__ called with {value}")
    
    def __delete__(self, instance):
        """Called when attribute is deleted"""
        print(f"__delete__ called")

class MyClass:
    attr = Descriptor()

obj = MyClass()
print(obj.attr)    # Calls __get__
obj.attr = "new"   # Calls __set__
del obj.attr       # Calls __delete__
```

### Types of Descriptors

```python
# Data Descriptor: defines __get__ AND __set__/__delete__
class DataDescriptor:
    def __get__(self, instance, owner):
        return "data"
    
    def __set__(self, instance, value):
        pass

# Non-Data Descriptor: defines only __get__
class NonDataDescriptor:
    def __get__(self, instance, owner):
        return "non-data"

# Non-data descriptors can be overridden by instance attributes
# Data descriptors cannot
```

### Property Decorator

```python
class Person:
    def __init__(self, name):
        self._name = name
    
    @property
    def name(self):
        """Getter method"""
        return self._name
    
    @name.setter
    def name(self, value):
        """Setter method with validation"""
        if not isinstance(value, str):
            raise TypeError("Name must be a string")
        self._name = value
    
    @name.deleter
    def name(self):
        """Deleter method"""
        del self._name

person = Person("Alice")
print(person.name)      # Calls getter
person.name = "Bob"     # Calls setter
del person.name         # Calls deleter
```

## 🧠 Why It Matters in Real Projects

- **Data Validation**: Validate values before setting attributes
- **Lazy Loading**: Compute expensive values only when accessed
- **Type Safety**: Enforce type constraints
- **Computed Properties**: Calculate values on-the-fly
- **Backwards Compatibility**: Add logic to existing attributes without breaking API
- **ORM Field Definitions**: Django, SQLAlchemy use descriptors extensively
- **Clean APIs**: Hide implementation details behind simple attribute access

## ⚙️ Internal Working

### Attribute Lookup Order

```python
# Priority order for attribute access (obj.attr):
# 1. Data descriptors from type(obj).__mro__
# 2. Instance dictionary (obj.__dict__)
# 3. Non-data descriptors from type(obj).__mro__
# 4. Class dictionary (type(obj).__dict__)
# 5. Raise AttributeError

class DataDesc:
    def __get__(self, instance, owner):
        return "data descriptor"
    
    def __set__(self, instance, value):
        pass

class NonDataDesc:
    def __get__(self, instance, owner):
        return "non-data descriptor"

class Example:
    data_desc = DataDesc()
    non_data_desc = NonDataDesc()

obj = Example()

# Data descriptor takes priority over instance dict
obj.__dict__['data_desc'] = "instance value"
print(obj.data_desc)  # "data descriptor" (descriptor wins)

# Non-data descriptor is overridden by instance dict
obj.__dict__['non_data_desc'] = "instance value"
print(obj.non_data_desc)  # "instance value" (instance wins)
```

### How property() Works

```python
# property() is implemented as a descriptor
class Property:
    """Simplified property implementation"""
    
    def __init__(self, fget=None, fset=None, fdel=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(instance)
    
    def __set__(self, instance, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(instance, value)
    
    def __delete__(self, instance):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(instance)
    
    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel)
    
    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel)
```

## ✅ Best Practices

### 1. Typed Attribute Descriptor

```python
class TypedAttribute:
    """Descriptor that enforces type checking"""
    
    def __init__(self, name, expected_type):
        self.name = name
        self.expected_type = expected_type
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name)
    
    def __set__(self, instance, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"{self.name} must be {self.expected_type.__name__}, "
                f"got {type(value).__name__}"
            )
        instance.__dict__[self.name] = value

class Person:
    name = TypedAttribute('name', str)
    age = TypedAttribute('age', int)
    
    def __init__(self, name, age):
        self.name = name
        self.age = age

person = Person("Alice", 30)
# person.age = "thirty"  # TypeError: age must be int
```

### 2. Lazy Property

```python
class LazyProperty:
    """Compute value only once, then cache"""
    
    def __init__(self, func):
        self.func = func
        self.name = func.__name__
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        
        # Check if already computed
        value = instance.__dict__.get(self.name)
        if value is None:
            # Compute and cache
            value = self.func(instance)
            instance.__dict__[self.name] = value
        
        return value

class DataProcessor:
    def __init__(self, data):
        self.data = data
    
    @LazyProperty
    def expensive_computation(self):
        """Computed only on first access"""
        print("Computing...")
        import time
        time.sleep(1)
        return sum(self.data)

processor = DataProcessor([1, 2, 3, 4, 5])
print(processor.expensive_computation)  # Prints "Computing...", then 15
print(processor.expensive_computation)  # Returns cached 15 immediately
```

### 3. Validated Property with Multiple Constraints

```python
class ValidatedProperty:
    """Property with multiple validation rules"""
    
    def __init__(self, name, *validators):
        self.name = name
        self.validators = validators
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name)
    
    def __set__(self, instance, value):
        # Run all validators
        for validator in self.validators:
            validator(value)
        instance.__dict__[self.name] = value

def positive(value):
    if value <= 0:
        raise ValueError("Must be positive")

def less_than_100(value):
    if value >= 100:
        raise ValueError("Must be less than 100")

def is_number(value):
    if not isinstance(value, (int, float)):
        raise TypeError("Must be a number")

class Product:
    price = ValidatedProperty('price', is_number, positive)
    quantity = ValidatedProperty('quantity', is_number, positive, less_than_100)
    
    def __init__(self, price, quantity):
        self.price = price
        self.quantity = quantity

product = Product(19.99, 50)
# product.price = -10  # ValueError: Must be positive
```

### 4. Readonly Property

```python
class ReadOnly:
    """Descriptor for read-only attributes"""
    
    def __init__(self, value):
        self._value = value
    
    def __get__(self, instance, owner):
        return self._value
    
    def __set__(self, instance, value):
        raise AttributeError("Cannot modify read-only attribute")

class Config:
    API_KEY = ReadOnly("secret-api-key-123")
    VERSION = ReadOnly("1.0.0")

config = Config()
print(config.API_KEY)  # "secret-api-key-key-123"
# config.API_KEY = "new"  # AttributeError
```

### 5. Cached Property with TTL (Time-To-Live)

```python
import time
from functools import wraps

class CachedProperty:
    """Property with time-based cache expiration"""
    
    def __init__(self, ttl=60):
        self.ttl = ttl
        self.func = None
    
    def __call__(self, func):
        self.func = func
        return self
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        
        cache_attr = f'_cache_{self.func.__name__}'
        time_attr = f'_time_{self.func.__name__}'
        
        cached_value = getattr(instance, cache_attr, None)
        cached_time = getattr(instance, time_attr, None)
        
        now = time.time()
        
        # Return cached value if not expired
        if cached_value is not None and cached_time is not None:
            if now - cached_time < self.ttl:
                return cached_value
        
        # Compute new value
        value = self.func(instance)
        setattr(instance, cache_attr, value)
        setattr(instance, time_attr, now)
        
        return value

class WeatherAPI:
    @CachedProperty(ttl=300)  # Cache for 5 minutes
    def current_temperature(self):
        print("Fetching temperature from API...")
        return 72  # Simulated API call

api = WeatherAPI()
print(api.current_temperature)  # Fetches from API
print(api.current_temperature)  # Uses cache
time.sleep(301)
print(api.current_temperature)  # Cache expired, fetches again
```

## ❌ Common Mistakes

### 1. Forgetting to Return self for Chaining

```python
# WRONG - Property setters don't return self
class Wrong:
    def __init__(self, value):
        self._value = value
    
    @property
    def value(self):
        return self._value
    
    @value.setter
    def value(self, val):
        self._value = val
        return self  # WRONG! Setters don't work this way

# Can't chain: obj.value = 10.value = 20

# CORRECT - Use fluent interface pattern instead
class Builder:
    def set_value(self, value):
        self._value = value
        return self

# Can chain: builder.set_value(10).set_value(20)
```

### 2. Storing Data in Descriptor Instead of Instance

```python
# WRONG - Shared across all instances!
class BadDescriptor:
    def __init__(self):
        self.value = None  # BAD! Shared state
    
    def __get__(self, instance, owner):
        return self.value
    
    def __set__(self, instance, value):
        self.value = value

# CORRECT - Store in instance
class GoodDescriptor:
    def __init__(self, name):
        self.name = name
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name)
    
    def __set__(self, instance, value):
        instance.__dict__[self.name] = value
```

### 3. Not Handling instance=None Case

```python
# WRONG - Breaks when accessed on class
class BadDescriptor:
    def __get__(self, instance, owner):
        return instance.value  # AttributeError when instance is None!

class MyClass:
    attr = BadDescriptor()

# MyClass.attr  # AttributeError!

# CORRECT - Handle class access
class GoodDescriptor:
    def __get__(self, instance, owner):
        if instance is None:
            return self  # Return descriptor when accessed on class
        return instance.value
```

## 🔐 Security Considerations

### 1. Sanitized String Property

```python
import re
import html

class SanitizedString:
    """Descriptor that sanitizes string input"""
    
    def __init__(self, name, max_length=255):
        self.name = name
        self.max_length = max_length
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name)
    
    def __set__(self, instance, value):
        if not isinstance(value, str):
            raise TypeError(f"{self.name} must be a string")
        
        # Sanitize
        value = html.escape(value)  # Escape HTML
        value = value[:self.max_length]  # Limit length
        value = re.sub(r'[^\w\s-]', '', value)  # Remove special chars
        
        instance.__dict__[self.name] = value

class UserProfile:
    username = SanitizedString('username', max_length=50)
    bio = SanitizedString('bio', max_length=500)

profile = UserProfile()
profile.username = "<script>alert('xss')</script>admin"
print(profile.username)  # Sanitized output
```

### 2. Password Property

```python
import hashlib

class PasswordProperty:
    """Descriptor for secure password handling"""
    
    def __init__(self, name):
        self.name = name
        self.hash_name = f'_{name}_hash'
    
    def __get__(self, instance, owner):
        raise AttributeError("Password is write-only")
    
    def __set__(self, instance, value):
        if not isinstance(value, str):
            raise TypeError("Password must be a string")
        
        if len(value) < 8:
            raise ValueError("Password must be at least 8 characters")
        
        # Hash password
        hashed = hashlib.sha256(value.encode()).hexdigest()
        instance.__dict__[self.hash_name] = hashed
    
    def verify(self, instance, password):
        """Verify password against stored hash"""
        hashed = hashlib.sha256(password.encode()).hexdigest()
        return hashed == instance.__dict__.get(self.hash_name)

class User:
    password = PasswordProperty('password')
    
    def check_password(self, password):
        return User.password.verify(self, password)

user = User()
user.password = "securepass123"
# print(user.password)  # AttributeError: write-only
print(user.check_password("securepass123"))  # True
```

## 🚀 Performance Optimization

### 1. Weakref Descriptor (Memory Efficient)

```python
import weakref

class WeakRefDescriptor:
    """Descriptor using weak references to save memory"""
    
    def __init__(self):
        self.data = weakref.WeakKeyDictionary()
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return self.data.get(instance)
    
    def __set__(self, instance, value):
        self.data[instance] = value

class LargeObject:
    cached_data = WeakRefDescriptor()

# Objects automatically garbage collected when no longer referenced
```

### 2. Slotted Descriptor

```python
class SlottedDescriptor:
    """Use __slots__ for memory efficiency"""
    __slots__ = ('name', 'validator')
    
    def __init__(self, name, validator=None):
        self.name = name
        self.validator = validator
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return getattr(instance, f'_{self.name}', None)
    
    def __set__(self, instance, value):
        if self.validator:
            self.validator(value)
        setattr(instance, f'_{self.name}', value)

class OptimizedClass:
    __slots__ = ('_value',)
    value = SlottedDescriptor('value')
```

## 🧪 Code Examples

### 1. Range-Validated Descriptor

```python
class RangeValidated:
    """Descriptor with min/max validation"""
    
    def __init__(self, name, min_value=None, max_value=None):
        self.name = name
        self.min_value = min_value
        self.max_value = max_value
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name)
    
    def __set__(self, instance, value):
        if self.min_value is not None and value < self.min_value:
            raise ValueError(f"{self.name} must be >= {self.min_value}")
        if self.max_value is not None and value > self.max_value:
            raise ValueError(f"{self.name} must be <= {self.max_value}")
        instance.__dict__[self.name] = value

class Temperature:
    celsius = RangeValidated('celsius', min_value=-273.15, max_value=None)
    
    def __init__(self, celsius):
        self.celsius = celsius

temp = Temperature(25)
# temp.celsius = -300  # ValueError
```

### 2. History-Tracking Descriptor

```python
class HistoryDescriptor:
    """Track all changes to an attribute"""
    
    def __init__(self, name):
        self.name = name
        self.history_name = f'_{name}_history'
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        history = instance.__dict__.get(self.history_name, [])
        return history[-1] if history else None
    
    def __set__(self, instance, value):
        history = instance.__dict__.setdefault(self.history_name, [])
        history.append(value)
    
    def get_history(self, instance):
        return instance.__dict__.get(self.history_name, [])

class Account:
    balance = HistoryDescriptor('balance')
    
    def __init__(self, initial_balance):
        self.balance = initial_balance
    
    def get_balance_history(self):
        return Account.balance.get_history(self)

account = Account(1000)
account.balance = 1500
account.balance = 1200
print(account.balance)  # 1200
print(account.get_balance_history())  # [1000, 1500, 1200]
```

### 3. Unit Conversion Descriptor

```python
class Distance:
    """Descriptor with automatic unit conversion"""
    
    def __init__(self, name):
        self.name = name
        self.meters_name = f'_{name}_meters'
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.meters_name, 0)
    
    def __set__(self, instance, value):
        # Store in meters
        instance.__dict__[self.meters_name] = value
    
    def kilometers(self, instance):
        return self.__get__(instance, None) / 1000
    
    def miles(self, instance):
        return self.__get__(instance, None) * 0.000621371

class Route:
    distance = Distance('distance')
    
    def __init__(self, meters):
        self.distance = meters
    
    def distance_km(self):
        return Route.distance.kilometers(self)
    
    def distance_miles(self):
        return Route.distance.miles(self)

route = Route(5000)  # 5000 meters
print(f"{route.distance}m")
print(f"{route.distance_km()}km")
print(f"{route.distance_miles()}miles")
```

## 🏗️ Real-World Use Cases

### Django Model Field

```python
class Field:
    """Simplified Django field descriptor"""
    
    def __init__(self, field_type, max_length=None, default=None):
        self.field_type = field_type
        self.max_length = max_length
        self.default = default
    
    def __set_name__(self, owner, name):
        """Called when descriptor is assigned to class"""
        self.name = name
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name, self.default)
    
    def __set__(self, instance, value):
        if not isinstance(value, self.field_type):
            raise TypeError(f"Expected {self.field_type.__name__}")
        
        if self.max_length and len(value) > self.max_length:
            raise ValueError(f"Max length is {self.max_length}")
        
        instance.__dict__[self.name] = value

class CharField(Field):
    def __init__(self, max_length=255):
        super().__init__(str, max_length=max_length)

class IntegerField(Field):
    def __init__(self):
        super().__init__(int)

class User:
    username = CharField(max_length=50)
    age = IntegerField()
```

### SQLAlchemy Column

```python
class Column:
    """Simplified SQLAlchemy column descriptor"""
    
    def __init__(self, type_, nullable=True, default=None):
        self.type = type_
        self.nullable = nullable
        self.default = default
    
    def __set_name__(self, owner, name):
        self.name = name
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name, self.default)
    
    def __set__(self, instance, value):
        if value is None and not self.nullable:
            raise ValueError(f"{self.name} cannot be null")
        
        if value is not None and not isinstance(value, self.type):
            raise TypeError(
                f"{self.name} must be {self.type.__name__}"
            )
        
        instance.__dict__[self.name] = value

class User:
    id = Column(int, nullable=False)
    username = Column(str, nullable=False)
    email = Column(str)
```

## ❓ Interview Questions

### Q1: What is a descriptor?

**Answer**: A descriptor is any object that defines `__get__()`, `__set__()`, or `__delete__()` methods. It controls attribute access on another object and implements the descriptor protocol.

### Q2: What's the difference between data and non-data descriptors?

**Answer**:
- **Data descriptor**: Defines `__get__` AND `__set__` or `__delete__`
- **Non-data descriptor**: Defines only `__get__`
- **Priority**: Data descriptors override instance `__dict__`, non-data don't

### Q3: How does property() work internally?

**Answer**: `property()` is implemented as a descriptor that stores getter/setter/deleter functions and calls them in `__get__`, `__set__`, `__delete__` methods.

### Q4: When would you use a descriptor over @property?

**Answer**: Use descriptors when:
- Need reusable validation logic across multiple classes
- Want to share descriptor logic
- Building frameworks/ORMs
- Need more control than @property offers

Use @property for simple, class-specific cases.

### Q5: What is __set_name__ and when is it called?

**Answer**: `__set_name__(self, owner, name)` is called when a descriptor is assigned to a class attribute. It receives the owner class and attribute name, useful for storing the name.

```python
class Descriptor:
    def __set_name__(self, owner, name):
        self.name = name

class MyClass:
    attr = Descriptor()  # __set_name__ called with 'attr'
```

### Q6: How do you make a read-only property?

**Answer**: Define getter but no setter:

```python
@property
def readonly(self):
    return self._value
# No @readonly.setter defined
```

### Q7: Can descriptors be used with class attributes?

**Answer**: Yes, but behavior differs. When accessed on class (not instance), `instance` parameter to `__get__` is None. Handle this case:

```python
def __get__(self, instance, owner):
    if instance is None:
        return self  # Accessed on class
    # Accessed on instance
```

## 🧩 Design Patterns

### Observer Pattern with Descriptors

```python
class Observable:
    """Descriptor that notifies observers of changes"""
    
    def __init__(self, name):
        self.name = name
        self.observers_name = f'_{name}_observers'
    
    def __set_name__(self, owner, name):
        self.name = name
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name)
    
    def __set__(self, instance, value):
        old_value = instance.__dict__.get(self.name)
        instance.__dict__[self.name] = value
        
        # Notify observers
        observers = instance.__dict__.get(self.observers_name, [])
        for observer in observers:
            observer(old_value, value)
    
    def add_observer(self, instance, observer):
        observers = instance.__dict__.setdefault(self.observers_name, [])
        observers.append(observer)

class Model:
    value = Observable('value')
    
    def __init__(self, value):
        self.value = value

def log_change(old, new):
    print(f"Value changed from {old} to {new}")

model = Model(10)
Model.value.add_observer(model, log_change)
model.value = 20  # Prints: Value changed from 10 to 20
```

---

## 📚 Summary

Descriptors and properties provide powerful attribute access control:
- Validation, lazy loading, computed properties
- Type safety and data sanitization
- Foundation for ORMs and frameworks
- Clean, Pythonic APIs

**Key Takeaways**:
1. Descriptors implement `__get__`, `__set__`, `__delete__`
2. Data descriptors override instance dict
3. Use `@property` for simple cases
4. Use descriptors for reusable logic
5. Always handle `instance=None` case
6. Store data in instance, not descriptor
7. `__set_name__` provides attribute name
