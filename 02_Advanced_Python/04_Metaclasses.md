# Metaclasses - Classes That Create Classes

## 📖 Concept Explanation

Metaclasses are classes whose instances are classes. They define how classes behave and are created. In Python, **everything is an object**, including classes themselves. The default metaclass for all classes is `type`.

### Understanding the Type Hierarchy

```python
# Regular object
my_string = "Hello"
print(type(my_string))  # <class 'str'>
print(type(str))        # <class 'type'>
print(type(type))       # <class 'type'>

# type is a metaclass - a class that creates classes
class MyClass:
    pass

print(type(MyClass))    # <class 'type'>
print(isinstance(MyClass, type))  # True
```

### Creating Classes Dynamically with type()

```python
# Method 1: Normal class definition
class Dog:
    def bark(self):
        return "Woof!"

# Method 2: Using type() dynamically
Dog = type('Dog', (), {'bark': lambda self: "Woof!"})

# Both are equivalent
dog = Dog()
print(dog.bark())  # "Woof!"

# type() signature: type(name, bases, dict)
# - name: class name
# - bases: tuple of base classes
# - dict: dictionary of class attributes/methods
```

### Custom Metaclass

```python
class MyMeta(type):
    """Custom metaclass"""
    
    def __new__(mcs, name, bases, namespace):
        """Called to create a new class"""
        print(f"Creating class: {name}")
        
        # Modify class before creation
        namespace['created_by'] = 'MyMeta'
        
        # Create the class
        return super().__new__(mcs, name, bases, namespace)
    
    def __init__(cls, name, bases, namespace):
        """Called after class is created"""
        print(f"Initializing class: {name}")
        super().__init__(name, bases, namespace)

class MyClass(metaclass=MyMeta):
    pass

"""Output:
Creating class: MyClass
Initializing class: MyClass
"""

print(MyClass.created_by)  # 'MyMeta'
```

## 🧠 Why It Matters in Real Projects

- **Framework Development**: Django ORM, SQLAlchemy use metaclasses
- **API Enforcement**: Ensure classes follow specific interfaces
- **Registration Systems**: Automatically register plugins/subclasses
- **Validation**: Validate class definitions at creation time
- **Singleton Pattern**: Enforce design patterns at metaclass level
- **ORM Mapping**: Map classes to database tables
- **Debugging**: Add automatic logging/profiling to all methods

## ⚙️ Internal Working

### Class Creation Process

```python
class MetaLogger(type):
    def __new__(mcs, name, bases, namespace):
        print(f"1. __new__ called for {name}")
        print(f"   Metaclass: {mcs}")
        print(f"   Bases: {bases}")
        print(f"   Namespace keys: {list(namespace.keys())}")
        
        cls = super().__new__(mcs, name, bases, namespace)
        print(f"2. Class object created: {cls}")
        return cls
    
    def __init__(cls, name, bases, namespace):
        print(f"3. __init__ called for {name}")
        super().__init__(name, bases, namespace)
    
    def __call__(cls, *args, **kwargs):
        print(f"4. __call__ called (creating instance of {cls.__name__})")
        instance = super().__call__(*args, **kwargs)
        print(f"5. Instance created: {instance}")
        return instance

class MyClass(metaclass=MetaLogger):
    def __init__(self, value):
        self.value = value
        print(f"6. MyClass.__init__ called with {value}")

"""Output when class is defined:
1. __new__ called for MyClass
   Metaclass: <class '__main__.MetaLogger'>
   Bases: ()
   Namespace keys: ['__module__', '__qualname__', '__init__']
2. Class object created: <class '__main__.MyClass'>
3. __init__ called for MyClass
"""

"""Output when instance is created:
4. __call__ called (creating instance of MyClass)
5. Instance created: <__main__.MyClass object at 0x...>
6. MyClass.__init__ called with 42
"""
obj = MyClass(42)
```

### Method Resolution Order (MRO) with Metaclasses

```python
class Meta1(type):
    pass

class Meta2(type):
    pass

class Base1(metaclass=Meta1):
    pass

class Base2(metaclass=Meta2):
    pass

# This will fail - conflicting metaclasses
# class Derived(Base1, Base2):
#     pass

# Solution: Create common metaclass
class CombinedMeta(Meta1, Meta2):
    pass

class Base1New(metaclass=CombinedMeta):
    pass

class Base2New(metaclass=CombinedMeta):
    pass

class Derived(Base1New, Base2New):
    pass
```

## ✅ Best Practices

### 1. Singleton Metaclass

```python
class SingletonMeta(type):
    """Ensure only one instance of a class exists"""
    _instances = {}
    
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    def __init__(self):
        print("Initializing database connection")

db1 = Database()  # Prints "Initializing..."
db2 = Database()  # Does not print
print(db1 is db2)  # True
```

### 2. Abstract Base Class Enforcement

```python
class AbstractMeta(type):
    """Enforce implementation of abstract methods"""
    
    def __new__(mcs, name, bases, namespace):
        # Check for required methods
        required_methods = ['save', 'load']
        
        if bases:  # Skip for base class itself
            for method in required_methods:
                if method not in namespace:
                    raise TypeError(
                        f"{name} must implement {method}() method"
                    )
        
        return super().__new__(mcs, name, bases, namespace)

class Model(metaclass=AbstractMeta):
    pass

# This will raise TypeError
# class User(Model):
#     pass

# This works
class User(Model):
    def save(self):
        pass
    
    def load(self):
        pass
```

### 3. Automatic Property Creation

```python
class AutoPropertyMeta(type):
    """Automatically create properties for _private attributes"""
    
    def __new__(mcs, name, bases, namespace):
        for attr_name, attr_value in list(namespace.items()):
            if attr_name.startswith('_') and not attr_name.startswith('__'):
                # Create property for _attr
                property_name = attr_name[1:]  # Remove leading underscore
                
                def make_getter(attr):
                    def getter(self):
                        return getattr(self, attr)
                    return getter
                
                def make_setter(attr):
                    def setter(self, value):
                        setattr(self, attr, value)
                    return setter
                
                namespace[property_name] = property(
                    make_getter(attr_name),
                    make_setter(attr_name)
                )
        
        return super().__new__(mcs, name, bases, namespace)

class Person(metaclass=AutoPropertyMeta):
    def __init__(self, name, age):
        self._name = name
        self._age = age

person = Person("Alice", 30)
print(person.name)  # Automatically created property
person.age = 31     # Automatically created setter
```

### 4. Plugin Registry System

```python
class PluginMeta(type):
    """Automatically register all subclasses"""
    registry = {}
    
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        
        # Register all non-base classes
        if bases:  # Skip the base class itself
            mcs.registry[name] = cls
        
        return cls
    
    @classmethod
    def get_plugins(mcs):
        return mcs.registry

class Plugin(metaclass=PluginMeta):
    """Base plugin class"""
    pass

class EmailPlugin(Plugin):
    def send(self):
        return "Sending email"

class SMSPlugin(Plugin):
    def send(self):
        return "Sending SMS"

# Automatically registered
print(PluginMeta.get_plugins())
# {'EmailPlugin': <class 'EmailPlugin'>, 'SMSPlugin': <class 'SMSPlugin'>}
```

### 5. Django ORM-Style Metaclass

```python
class Field:
    """Represents a database field"""
    def __init__(self, field_type):
        self.field_type = field_type

class ModelMeta(type):
    """Metaclass for ORM-like models"""
    
    def __new__(mcs, name, bases, namespace):
        # Extract fields
        fields = {}
        for key, value in list(namespace.items()):
            if isinstance(value, Field):
                fields[key] = value
                namespace.pop(key)  # Remove from class namespace
        
        # Create class
        cls = super().__new__(mcs, name, bases, namespace)
        
        # Store fields metadata
        cls._fields = fields
        
        # Create __init__ to accept field values
        def init(self, **kwargs):
            for field_name in cls._fields:
                setattr(self, field_name, kwargs.get(field_name))
        
        cls.__init__ = init
        
        return cls

class Model(metaclass=ModelMeta):
    pass

class User(Model):
    name = Field(str)
    age = Field(int)
    email = Field(str)

# Usage
user = User(name="Alice", age=30, email="alice@example.com")
print(user.name)  # "Alice"
print(User._fields)  # Field metadata
```

## ❌ Common Mistakes

### 1. Confusing __new__ and __init__

```python
# WRONG - Using __init__ to modify class
class WrongMeta(type):
    def __init__(cls, name, bases, namespace):
        cls.custom_attr = "value"  # Works but __new__ is better

# CORRECT - Use __new__ to modify class creation
class CorrectMeta(type):
    def __new__(mcs, name, bases, namespace):
        namespace['custom_attr'] = "value"
        return super().__new__(mcs, name, bases, namespace)
```

### 2. Not Calling super()

```python
# WRONG - Not calling super().__new__
class BadMeta(type):
    def __new__(mcs, name, bases, namespace):
        # Create class manually - BAD!
        return type(name, bases, namespace)

# CORRECT - Always call super()
class GoodMeta(type):
    def __new__(mcs, name, bases, namespace):
        return super().__new__(mcs, name, bases, namespace)
```

### 3. Modifying namespace After super().__new__

```python
# WRONG - Modify after class creation
class WrongOrder(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        namespace['attr'] = 'value'  # Too late! namespace already used
        return cls

# CORRECT - Modify before class creation
class CorrectOrder(type):
    def __new__(mcs, name, bases, namespace):
        namespace['attr'] = 'value'  # Before class creation
        return super().__new__(mcs, name, bases, namespace)
```

## 🔐 Security Considerations

### 1. Validation Metaclass

```python
class ValidatedMeta(type):
    """Validate class definitions for security"""
    
    def __new__(mcs, name, bases, namespace):
        # Prevent dangerous method names
        dangerous_names = ['exec', 'eval', '__del__']
        for dangerous in dangerous_names:
            if dangerous in namespace:
                raise SecurityError(
                    f"Method {dangerous} not allowed in {name}"
                )
        
        # Validate method signatures
        import inspect
        for attr_name, attr_value in namespace.items():
            if callable(attr_value):
                sig = inspect.signature(attr_value)
                # Enforce specific parameter patterns
                if len(sig.parameters) > 10:
                    raise ValueError(
                        f"Method {attr_name} has too many parameters"
                    )
        
        return super().__new__(mcs, name, bases, namespace)

class SecureModel(metaclass=ValidatedMeta):
    def safe_method(self):
        pass
```

### 2. Permission-Based Access

```python
class PermissionMeta(type):
    """Add permission checks to all methods"""
    
    def __new__(mcs, name, bases, namespace):
        import functools
        
        for attr_name, attr_value in namespace.items():
            if callable(attr_value) and not attr_name.startswith('_'):
                # Wrap method with permission check
                @functools.wraps(attr_value)
                def wrapper(self, *args, **kwargs):
                    if not self.has_permission(attr_name):
                        raise PermissionError(
                            f"No permission for {attr_name}"
                        )
                    return attr_value(self, *args, **kwargs)
                
                namespace[attr_name] = wrapper
        
        return super().__new__(mcs, name, bases, namespace)
```

## 🚀 Performance Optimization

### 1. Method Caching Metaclass

```python
class CacheMeta(type):
    """Automatically cache method results"""
    
    def __new__(mcs, name, bases, namespace):
        from functools import lru_cache
        
        for attr_name, attr_value in namespace.items():
            if (callable(attr_value) and 
                not attr_name.startswith('_') and
                hasattr(attr_value, '_cache_results')):
                
                namespace[attr_name] = lru_cache(maxsize=128)(attr_value)
        
        return super().__new__(mcs, name, bases, namespace)

def cache_results(func):
    """Decorator to mark methods for caching"""
    func._cache_results = True
    return func

class Calculator(metaclass=CacheMeta):
    @cache_results
    def expensive_calculation(self, n):
        """This will be automatically cached"""
        return sum(i**2 for i in range(n))
```

### 2. Slot Optimization Metaclass

```python
class SlotsMeta(type):
    """Automatically add __slots__ for memory optimization"""
    
    def __new__(mcs, name, bases, namespace):
        # Find all instance attributes from __init__
        if '__init__' in namespace:
            import inspect
            init_source = inspect.getsource(namespace['__init__'])
            
            # Simple pattern matching for self.attribute
            import re
            slots = set(re.findall(r'self\.(\w+)', init_source))
            
            if slots:
                namespace['__slots__'] = tuple(slots)
        
        return super().__new__(mcs, name, bases, namespace)

class OptimizedClass(metaclass=SlotsMeta):
    def __init__(self, name, age):
        self.name = name
        self.age = age

# Automatically has __slots__ = ('name', 'age')
print(OptimizedClass.__slots__)
```

## 🧪 Code Examples

### 1. Method Call Logger

```python
class LoggerMeta(type):
    """Log all method calls"""
    
    def __new__(mcs, name, bases, namespace):
        for attr_name, attr_value in namespace.items():
            if callable(attr_value) and not attr_name.startswith('_'):
                namespace[attr_name] = mcs.log_calls(attr_value, attr_name)
        
        return super().__new__(mcs, name, bases, namespace)
    
    @staticmethod
    def log_calls(func, method_name):
        def wrapper(*args, **kwargs):
            print(f"Calling {method_name} with {args[1:]}, {kwargs}")
            result = func(*args, **kwargs)
            print(f"{method_name} returned {result}")
            return result
        return wrapper

class Calculator(metaclass=LoggerMeta):
    def add(self, a, b):
        return a + b
    
    def multiply(self, a, b):
        return a * b

calc = Calculator()
calc.add(5, 3)
# Calling add with (5, 3), {}
# add returned 8
```

### 2. Immutable Class Metaclass

```python
class ImmutableMeta(type):
    """Create immutable classes"""
    
    def __call__(cls, *args, **kwargs):
        instance = super().__call__(*args, **kwargs)
        instance.__dict__['_frozen'] = True
        return instance
    
    def __new__(mcs, name, bases, namespace):
        # Override __setattr__
        def setattr(self, name, value):
            if hasattr(self, '_frozen') and self._frozen:
                raise AttributeError(f"{name} is immutable")
            object.__setattr__(self, name, value)
        
        namespace['__setattr__'] = setattr
        return super().__new__(mcs, name, bases, namespace)

class Point(metaclass=ImmutableMeta):
    def __init__(self, x, y):
        self.x = x
        self.y = y

point = Point(1, 2)
# point.x = 5  # AttributeError: x is immutable
```

### 3. API Versioning Metaclass

```python
class VersionedMeta(type):
    """Support API versioning"""
    _versions = {}
    
    def __new__(mcs, name, bases, namespace):
        version = namespace.get('__version__', '1.0')
        
        cls = super().__new__(mcs, name, bases, namespace)
        
        # Store in version registry
        if name not in mcs._versions:
            mcs._versions[name] = {}
        mcs._versions[name][version] = cls
        
        return cls
    
    @classmethod
    def get_version(mcs, name, version):
        return mcs._versions.get(name, {}).get(version)

class UserAPIv1(metaclass=VersionedMeta):
    __version__ = '1.0'
    
    def get_data(self):
        return {'id': 1, 'name': 'User'}

class UserAPIv2(metaclass=VersionedMeta):
    __version__ = '2.0'
    
    def get_data(self):
        return {'id': 1, 'username': 'user', 'email': 'user@example.com'}

# Get specific version
api_v1 = VersionedMeta.get_version('UserAPIv1', '1.0')()
api_v2 = VersionedMeta.get_version('UserAPIv2', '2.0')()
```

## 🏗️ Real-World Use Cases

### Django Model Metaclass (Simplified)

```python
class ModelBase(type):
    """Simplified Django model metaclass"""
    
    def __new__(mcs, name, bases, namespace):
        # Collect field definitions
        fields = {}
        for key, value in list(namespace.items()):
            if isinstance(value, Field):
                fields[key] = value
        
        # Create _meta object
        namespace['_meta'] = type('Meta', (), {
            'fields': fields,
            'db_table': name.lower() + 's',
        })
        
        return super().__new__(mcs, name, bases, namespace)

class Model(metaclass=ModelBase):
    pass

class User(Model):
    name = Field(str)
    email = Field(str)

print(User._meta.db_table)  # 'users'
print(User._meta.fields)
```

### SQLAlchemy Declarative Base

```python
class DeclarativeMeta(type):
    """Simplified SQLAlchemy declarative metaclass"""
    
    def __new__(mcs, name, bases, namespace):
        # Create table name from class name
        if '__tablename__' not in namespace and bases:
            namespace['__tablename__'] = name.lower()
        
        # Collect column definitions
        columns = {}
        for key, value in namespace.items():
            if isinstance(value, Column):
                columns[key] = value
        
        namespace['__columns__'] = columns
        
        return super().__new__(mcs, name, bases, namespace)

class Column:
    def __init__(self, column_type, **kwargs):
        self.type = column_type
        self.kwargs = kwargs

class Base(metaclass=DeclarativeMeta):
    pass

class User(Base):
    id = Column('INTEGER', primary_key=True)
    username = Column('VARCHAR', nullable=False)

print(User.__tablename__)  # 'user'
print(User.__columns__)
```

## ❓ Interview Questions

### Q1: What is a metaclass?

**Answer**: A metaclass is a class whose instances are classes. It defines how classes are created and behave. The default metaclass is `type`. Metaclasses customize class creation through `__new__` and `__init__` methods.

### Q2: What's the difference between __new__ and __init__ in metaclasses?

**Answer**:
- `__new__`: Creates and returns the class object (called first)
- `__init__`: Initializes the created class (called after __new__)
- `__new__` is more powerful - can modify class before creation
- `__init__` modifies class after it's created

### Q3: When should you use metaclasses?

**Answer**: Use metaclasses when:
- Building frameworks (Django, SQLAlchemy)
- Automatically registering subclasses
- Enforcing coding standards/patterns
- Validating class definitions
- Adding automatic functionality to classes

**Important**: "Metaclasses are deeper magic than 99% of users should ever worry about" - Tim Peters

### Q4: How is type() both a function and a class?

**Answer**: `type()` is a metaclass that works two ways:
```python
# As a function: returns type of object
type(42)  # <class 'int'>

# As a metaclass: creates new classes
MyClass = type('MyClass', (), {'attr': 'value'})
```

### Q5: Can you inherit from multiple metaclasses?

**Answer**: Direct multiple metaclass inheritance usually conflicts. Solution: create a combined metaclass:

```python
class CombinedMeta(Meta1, Meta2):
    pass

class MyClass(metaclass=CombinedMeta):
    pass
```

### Q6: What's the difference between class decorators and metaclasses?

**Answer**:
- **Class decorators**: Wrap the class after creation
- **Metaclasses**: Control class creation process itself

```python
# Decorator - modifies after creation
@decorator
class MyClass:
    pass

# Metaclass - controls creation
class MyClass(metaclass=MyMeta):
    pass
```

Decorators are simpler and usually sufficient; use metaclasses only for complex framework-level modifications.

### Q7: How does Python determine which metaclass to use?

**Answer**: Python uses this order:
1. Explicit `metaclass=` parameter
2. Metaclass of base classes
3. Default `type` metaclass

### Q8: What is `__init_subclass__` and how does it relate to metaclasses?

**Answer**: `__init_subclass__` is a simpler alternative to metaclasses for subclass customization:

```python
class Base:
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        cls.custom_attr = "value"

class Derived(Base):
    pass

print(Derived.custom_attr)  # "value"
```

Prefer `__init_subclass__` over metaclasses when possible.

## 🧩 Design Patterns

### Abstract Factory with Metaclass

```python
class FactoryMeta(type):
    """Metaclass for factory pattern"""
    _products = {}
    
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        
        # Register product types
        if 'product_type' in namespace:
            mcs._products[namespace['product_type']] = cls
        
        return cls
    
    @classmethod
    def create(mcs, product_type):
        if product_type in mcs._products:
            return mcs._products[product_type]()
        raise ValueError(f"Unknown product type: {product_type}")

class Product(metaclass=FactoryMeta):
    pass

class ConcreteProductA(Product):
    product_type = 'A'

class ConcreteProductB(Product):
    product_type = 'B'

# Create products
product_a = FactoryMeta.create('A')
product_b = FactoryMeta.create('B')
```

---

## 📚 Summary

Metaclasses are advanced Python feature for:
- Customizing class creation
- Framework development
- Automatic registration and validation
- Enforcing patterns and standards

**Key Takeaways**:
1. Metaclasses are classes that create classes
2. Default metaclass is `type`
3. Use `__new__` to modify class before creation
4. Use `__init__` to initialize after creation
5. Consider simpler alternatives first (decorators, `__init_subclass__`)
6. Metaclasses are powerful but rarely needed in application code
7. Essential for framework development (Django, SQLAlchemy)
