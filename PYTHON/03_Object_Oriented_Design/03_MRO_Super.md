# 📘 Method Resolution Order (MRO) and super() in Python

## 📖 Concept Explanation

### What is Method Resolution Order (MRO)?

**MRO** is the order in which Python searches for methods and attributes in a class hierarchy. It's crucial for understanding inheritance, especially multiple inheritance.

### Basic MRO Example

```python
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(B):
    def method(self):
        return "C"

# MRO: C -> B -> A -> object
print(C.mro())
# [<class '__main__.C'>, <class '__main__.B'>,
#  <class '__main__.A'>, <class 'object'>]

c = C()
print(c.method())  # "C" (found in C first)
```

### The C3 Linearization Algorithm

Python uses **C3 linearization** (also called C3 superclass linearization) to compute MRO.

**Rules:**

1. Children before parents
2. Parents in the order they appear in the class definition
3. Each class appears only once (monotonicity)
4. Must preserve local precedence order

```python
class A:
    pass

class B:
    pass

class C(A, B):
    pass

# MRO: C -> A -> B -> object
print(C.mro())

class D(B, A):
    pass

# MRO: D -> B -> A -> object (different order!)
print(D.mro())
```

### The super() Function

`super()` returns a proxy object that delegates method calls to the next class in the MRO.

```python
class A:
    def method(self):
        print("A.method")
        return "A"

class B(A):
    def method(self):
        print("B.method")
        result = super().method()  # Calls A.method
        return f"B -> {result}"

class C(B):
    def method(self):
        print("C.method")
        result = super().method()  # Calls B.method
        return f"C -> {result}"

c = C()
print(c.method())
# Output:
# C.method
# B.method
# A.method
# C -> B -> A
```

---

## 🧠 Why It Matters

### Real-World Benefits

1. **Predictable Behavior**: Know which method will be called
2. **Cooperative Inheritance**: All classes work together properly
3. **Multiple Inheritance**: Avoid duplicate calls to base classes
4. **Framework Development**: Essential for mixin patterns
5. **Debugging**: Understand method lookup for troubleshooting

### Industry Usage

- **Django Mixins**: View mixins rely on proper MRO
- **Multiple Inheritance**: Database connection pooling, caching layers
- **Framework Base Classes**: SQLAlchemy, Django ORM
- **Testing Frameworks**: pytest fixtures and unittest
- **Metaclasses**: Framework magic relies on MRO

---

## ⚙️ Internal Working

### MRO Calculation

```python
class O:  # object
    pass

class A(O):
    pass

class B(O):
    pass

class C(O):
    pass

class D(A, B):
    pass

class E(B, C):
    pass

class F(D, E):
    pass

# Complex hierarchy
print("F MRO:")
for i, cls in enumerate(F.mro(), 1):
    print(f"  {i}. {cls.__name__}")

# Output:
# F MRO:
#   1. F
#   2. D
#   3. A
#   4. E
#   5. B
#   6. C
#   7. O
#   8. object
```

### How C3 Linearization Works

```python
# Manual C3 calculation example

# Given:
# class F(D, E): pass
# class D(A, B): pass
# class E(B, C): pass
# class A(O): pass
# class B(O): pass
# class C(O): pass

# Step-by-step C3 calculation:
# L(O) = [O]
# L(A) = [A, O]
# L(B) = [B, O]
# L(C) = [C, O]
# L(D) = [D] + merge(L(A), L(B), [A, B])
#      = [D] + merge([A, O], [B, O], [A, B])
#      = [D, A] + merge([O], [B, O], [B])
#      = [D, A, B] + merge([O], [O])
#      = [D, A, B, O]
# L(E) = [E] + merge(L(B), L(C), [B, C])
#      = [E] + merge([B, O], [C, O], [B, C])
#      = [E, B] + merge([O], [C, O], [C])
#      = [E, B, C] + merge([O], [O])
#      = [E, B, C, O]
# L(F) = [F] + merge(L(D), L(E), [D, E])
#      = [F] + merge([D, A, B, O], [E, B, C, O], [D, E])
#      = [F, D, A, E, B, C, O]
```

### super() Internals

```python
class A:
    def method(self):
        print(f"A.method called on {self.__class__.__name__}")

class B(A):
    def method(self):
        print(f"B.method called on {self.__class__.__name__}")
        # super() is equivalent to:
        # super(B, self).method()
        # Which means: "Start from B in self's MRO, get next class"
        super().method()

class C(A):
    def method(self):
        print(f"C.method called on {self.__class__.__name__}")
        super().method()

class D(B, C):
    def method(self):
        print(f"D.method called on {self.__class__.__name__}")
        super().method()

# MRO: D -> B -> C -> A -> object
print("MRO:", [cls.__name__ for cls in D.mro()])

d = D()
d.method()
# Output:
# MRO: ['D', 'B', 'C', 'A', 'object']
# D.method called on D
# B.method called on D
# C.method called on D
# A.method called on D
```

### **mro** Attribute

```python
class A:
    pass

class B(A):
    pass

class C(A):
    pass

class D(B, C):
    pass

# Access MRO as tuple
print(D.__mro__)
# (<class '__main__.D'>, <class '__main__.B'>,
#  <class '__main__.C'>, <class '__main__.A'>, <class 'object'>)

# Access MRO as list
print(D.mro())

# Check if class is in MRO
print(A in D.__mro__)  # True
print(object in D.__mro__)  # True

# Get position in MRO
print(D.__mro__.index(C))  # 2
```

---

## ✅ Best Practices

### 1. Always Use super() for Cooperative Inheritance

```python
# ❌ BAD: Explicitly calling parent
class A:
    def __init__(self):
        print("A.__init__")

class B(A):
    def __init__(self):
        A.__init__(self)  # Explicit call
        print("B.__init__")

class C(A):
    def __init__(self):
        A.__init__(self)  # Explicit call
        print("C.__init__")

class D(B, C):
    def __init__(self):
        B.__init__(self)
        C.__init__(self)
        print("D.__init__")

d = D()
# Output:
# A.__init__
# B.__init__
# A.__init__  <- Called twice! BAD!
# C.__init__
# D.__init__

# ✅ GOOD: Using super() for cooperative calls
class A:
    def __init__(self):
        print("A.__init__")
        super().__init__()

class B(A):
    def __init__(self):
        print("B.__init__")
        super().__init__()

class C(A):
    def __init__(self):
        print("C.__init__")
        super().__init__()

class D(B, C):
    def __init__(self):
        print("D.__init__")
        super().__init__()

d = D()
# Output:
# D.__init__
# B.__init__
# C.__init__
# A.__init__  <- Called once! GOOD!
```

### 2. Use \*\*kwargs for Flexible Initialization

```python
class Base:
    def __init__(self, base_param, **kwargs):
        self.base_param = base_param
        super().__init__(**kwargs)
        print(f"Base: {base_param}")

class Mixin1:
    def __init__(self, mixin1_param=None, **kwargs):
        self.mixin1_param = mixin1_param
        super().__init__(**kwargs)
        print(f"Mixin1: {mixin1_param}")

class Mixin2:
    def __init__(self, mixin2_param=None, **kwargs):
        self.mixin2_param = mixin2_param
        super().__init__(**kwargs)
        print(f"Mixin2: {mixin2_param}")

class Derived(Mixin1, Mixin2, Base):
    def __init__(self, derived_param, **kwargs):
        self.derived_param = derived_param
        super().__init__(**kwargs)
        print(f"Derived: {derived_param}")

# All parameters passed through
obj = Derived(
    derived_param="D",
    mixin1_param="M1",
    mixin2_param="M2",
    base_param="B"
)
# Output:
# Derived: D
# Mixin1: M1
# Mixin2: M2
# Base: B
```

### 3. Check MRO When Debugging

```python
def print_mro(cls):
    """Pretty print MRO for debugging"""
    print(f"\nMRO for {cls.__name__}:")
    for i, base in enumerate(cls.mro(), 1):
        print(f"  {i}. {base.__module__}.{base.__name__}")

class A:
    pass

class B(A):
    pass

class C(A):
    pass

class D(B, C):
    pass

print_mro(D)
# Output:
# MRO for D:
#   1. __main__.D
#   2. __main__.B
#   3. __main__.C
#   4. __main__.A
#   5. builtins.object
```

### 4. Use super() with Arguments for Advanced Cases

```python
class A:
    def method(self):
        print("A.method")

class B(A):
    def method(self):
        print("B.method")

class C(A):
    def method(self):
        print("C.method")

class D(B, C):
    def method(self):
        print("D.method")

        # Call specific class in MRO
        super(D, self).method()  # Next after D (B)
        super(B, self).method()  # Next after B (C)
        super(C, self).method()  # Next after C (A)

d = D()
d.method()
# Output:
# D.method
# B.method
# C.method
# A.method
```

---

## ❌ Common Mistakes

### 1. Diamond Problem Without super()

```python
# ❌ BAD: Diamond problem without super()
class A:
    def __init__(self):
        print("A.__init__")
        self.value = "A"

class B(A):
    def __init__(self):
        A.__init__(self)  # Direct call
        print("B.__init__")

class C(A):
    def __init__(self):
        A.__init__(self)  # Direct call
        print("C.__init__")

class D(B, C):
    def __init__(self):
        B.__init__(self)
        C.__init__(self)
        print("D.__init__")

d = D()
# Output:
# A.__init__  <- Called from B
# B.__init__
# A.__init__  <- Called again from C! BAD!
# C.__init__
# D.__init__

# ✅ GOOD: Using super()
class A:
    def __init__(self):
        print("A.__init__")
        self.value = "A"
        super().__init__()

class B(A):
    def __init__(self):
        print("B.__init__")
        super().__init__()

class C(A):
    def __init__(self):
        print("C.__init__")
        super().__init__()

class D(B, C):
    def __init__(self):
        print("D.__init__")
        super().__init__()

d = D()
# Output:
# D.__init__
# B.__init__
# C.__init__
# A.__init__  <- Called once! GOOD!
```

### 2. Inconsistent super() Usage

```python
# ❌ BAD: Mixing super() and direct calls
class A:
    def method(self):
        print("A")

class B(A):
    def method(self):
        print("B")
        super().method()

class C(A):
    def method(self):
        print("C")
        A.method(self)  # Direct call instead of super()!

class D(B, C):
    def method(self):
        print("D")
        super().method()

d = D()
d.method()
# Output:
# D
# B
# C
# A  <- Called from direct call in C
# But we never called A through super() chain!

# ✅ GOOD: Consistent super() usage
class C(A):
    def method(self):
        print("C")
        super().method()  # Use super() consistently
```

### 3. Wrong Order in Multiple Inheritance

```python
# ❌ BAD: Order matters!
class LoggerMixin:
    def save(self):
        print("Logging...")
        super().save()

class Model:
    def save(self):
        print("Saving to database")

class User(Model, LoggerMixin):  # Wrong order!
    def save(self):
        super().save()

user = User()
user.save()
# Output:
# Saving to database
# (LoggerMixin.save() never called because Model.save()
#  doesn't call super()!)

# ✅ GOOD: Mixins before base class
class Model:
    def save(self):
        print("Saving to database")
        super().save()  # Allow mixins to participate

class User(LoggerMixin, Model):  # Correct order!
    def save(self):
        super().save()

user = User()
user.save()
# Output:
# Logging...
# Saving to database
```

### 4. Not Handling object's **init**

```python
# ❌ BAD: super().__init__() when not needed
class Mixin:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)  # Passes kwargs to object!

class Base:
    def __init__(self):
        super().__init__()  # object.__init__() takes no args

class Derived(Mixin, Base):
    def __init__(self, value):
        super().__init__(value=value)  # TypeError!

# ✅ GOOD: Handle object properly
class Mixin:
    def __init__(self, **kwargs):
        # Consume your parameters
        self.mixin_value = kwargs.pop('mixin_value', None)
        # Pass rest to next in MRO
        super().__init__(**kwargs)

class Base:
    def __init__(self, **kwargs):
        # Consume your parameters
        self.base_value = kwargs.pop('base_value', None)
        # Only call super if there are no leftover kwargs
        if kwargs:
            super().__init__(**kwargs)
        else:
            super().__init__()

class Derived(Mixin, Base):
    def __init__(self, value):
        super().__init__(
            mixin_value="M",
            base_value="B"
        )
```

---

## 🔐 Security Considerations

### 1. Validate MRO in Security-Critical Classes

```python
class SecureBase:
    """Base class with security requirements"""

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)

        # Ensure security mixin is in MRO
        if not any(c.__name__ == 'SecurityMixin' for c in cls.mro()):
            raise TypeError(
                f"{cls.__name__} must inherit from SecurityMixin"
            )

class SecurityMixin:
    def validate_access(self):
        return True

class GoodClass(SecurityMixin, SecureBase):
    pass

# class BadClass(SecureBase):  # TypeError!
#     pass
```

### 2. Prevent Method Resolution Hijacking

```python
class ProtectedBase:
    """Prevent unauthorized method override"""

    _protected_methods = {'authenticate', 'authorize'}

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)

        # Check if protected methods are overridden
        for method_name in cls._protected_methods:
            if method_name in cls.__dict__:
                # Allow if it calls super()
                method = cls.__dict__[method_name]
                import inspect
                source = inspect.getsource(method)
                if 'super()' not in source:
                    raise SecurityError(
                        f"Cannot override {method_name} without calling super()"
                    )

    def authenticate(self, credentials):
        # Critical security method
        return True

    def authorize(self, resource):
        # Critical security method
        return True

class SecurityError(Exception):
    pass
```

---

## 🚀 Performance Optimization

### 1. Cache Method Lookups

```python
class MethodCache:
    """Cache frequently called methods"""

    def __init__(self):
        # Cache method references
        self._cached_methods = {}

    def get_method(self, name):
        """Get method with caching"""
        if name not in self._cached_methods:
            # Look up method once
            self._cached_methods[name] = getattr(self, name)
        return self._cached_methods[name]

class FastClass(MethodCache):
    def process(self):
        return "processing"

    def run(self):
        # Use cached method lookup
        method = self.get_method('process')
        for _ in range(1000000):
            method()
```

### 2. Avoid Deep MRO in Hot Paths

```python
# ❌ SLOWER: Deep inheritance
class Level1:
    def compute(self, x):
        return x

class Level2(Level1):
    pass

class Level3(Level2):
    pass

class Level4(Level3):
    def compute(self, x):
        return super().compute(x) * 2

# ✅ FASTER: Shallow hierarchy
class Fast:
    def compute(self, x):
        return x * 2

# Benchmark
import timeit

deep = Level4()
fast = Fast()

time_deep = timeit.timeit(lambda: deep.compute(10), number=1000000)
time_fast = timeit.timeit(lambda: fast.compute(10), number=1000000)

print(f"Deep: {time_deep:.4f}s")
print(f"Fast: {time_fast:.4f}s")
print(f"Speedup: {time_deep/time_fast:.2f}x")
```

### 3. Use **slots** to Reduce MRO Overhead

```python
class Base:
    __slots__ = ['base_attr']

class Mixin:
    __slots__ = ['mixin_attr']

class Derived(Mixin, Base):
    __slots__ = ['derived_attr']

    def __init__(self):
        self.base_attr = "base"
        self.mixin_attr = "mixin"
        self.derived_attr = "derived"

# Faster attribute access, less memory
obj = Derived()
```

---

## 🏗️ Real-World Use Cases

### 1. Django Mixin Pattern

```python
class View:
    """Base view class"""

    def dispatch(self, request):
        print("View.dispatch")
        return self.handle(request)

    def handle(self, request):
        handler = getattr(self, request.method.lower())
        return handler(request)

class LoginRequiredMixin:
    """Mixin to require authentication"""

    def dispatch(self, request):
        print("LoginRequiredMixin.dispatch - checking auth")
        if not self.is_authenticated(request):
            return {"error": "Unauthorized"}, 401
        return super().dispatch(request)

    def is_authenticated(self, request):
        return request.user is not None

class PermissionRequiredMixin:
    """Mixin to check permissions"""

    def dispatch(self, request):
        print("PermissionRequiredMixin.dispatch - checking permissions")
        if not self.has_permission(request):
            return {"error": "Forbidden"}, 403
        return super().dispatch(request)

    def has_permission(self, request):
        return request.user.is_admin

class AdminView(PermissionRequiredMixin, LoginRequiredMixin, View):
    """View requiring both auth and admin permissions"""

    def get(self, request):
        return {"message": "Admin dashboard"}, 200

# Test
class Request:
    def __init__(self, method, user=None):
        self.method = method
        self.user = user

class User:
    def __init__(self, is_admin=False):
        self.is_admin = is_admin

# MRO: AdminView -> PermissionRequiredMixin -> LoginRequiredMixin -> View
print([c.__name__ for c in AdminView.mro()])

# Request with admin user
admin_user = User(is_admin=True)
request = Request("GET", user=admin_user)
view = AdminView()
response, status = view.dispatch(request)
print(f"Response: {response}, Status: {status}")
```

### 2. SQLAlchemy-like Declarative Base

```python
class DeclarativeMeta(type):
    """Metaclass for ORM models"""

    def __new__(mcs, name, bases, attrs):
        # Process fields
        fields = {}
        for key, value in list(attrs.items()):
            if isinstance(value, Field):
                fields[key] = value
                attrs.pop(key)

        attrs['_fields'] = fields
        return super().__new__(mcs, name, bases, attrs)

class Field:
    """Base field class"""

    def __init__(self, field_type, default=None):
        self.field_type = field_type
        self.default = default

class Model(metaclass=DeclarativeMeta):
    """Base model class"""

    def __init__(self, **kwargs):
        for field_name, field in self._fields.items():
            value = kwargs.get(field_name, field.default)
            setattr(self, field_name, value)

    def save(self):
        print(f"Saving {self.__class__.__name__} with fields: {self._fields}")
        super().save() if hasattr(super(), 'save') else None

class TimestampMixin:
    """Mixin for timestamp fields"""

    def save(self):
        from datetime import datetime
        if not hasattr(self, 'created_at'):
            self.created_at = datetime.now()
        self.updated_at = datetime.now()
        super().save()

class User(TimestampMixin, Model):
    """User model with timestamps"""

    username = Field(str)
    email = Field(str)
    age = Field(int, default=0)

# Usage
user = User(username="alice", email="alice@example.com", age=30)
user.save()

print(f"Fields: {user._fields}")
print(f"Username: {user.username}")
print(f"Created at: {user.created_at}")
```

### 3. Plugin System with MRO

```python
class PluginBase:
    """Base plugin class"""

    def process(self, data):
        print(f"{self.__class__.__name__}.process")
        return self.transform(data)

    def transform(self, data):
        """Override in subclass"""
        return data

class ValidationPlugin(PluginBase):
    """Validates data"""

    def process(self, data):
        print(f"{self.__class__.__name__}.process - validating")
        if not self.validate(data):
            raise ValueError("Invalid data")
        return super().process(data)

    def validate(self, data):
        return isinstance(data, dict) and 'value' in data

class LoggingPlugin(PluginBase):
    """Logs data"""

    def process(self, data):
        print(f"{self.__class__.__name__}.process - logging")
        print(f"  Data: {data}")
        return super().process(data)

class TransformPlugin(PluginBase):
    """Transforms data"""

    def transform(self, data):
        print(f"{self.__class__.__name__}.transform")
        data['value'] = data['value'] * 2
        return data

class Pipeline(ValidationPlugin, LoggingPlugin, TransformPlugin):
    """Complete pipeline with all plugins"""
    pass

# MRO ensures correct execution order
print("MRO:", [c.__name__ for c in Pipeline.mro()])

pipeline = Pipeline()
result = pipeline.process({'value': 10})
print(f"Result: {result}")
# Output:
# MRO: ['Pipeline', 'ValidationPlugin', 'LoggingPlugin', 'TransformPlugin', 'PluginBase', 'object']
# ValidationPlugin.process - validating
# LoggingPlugin.process - logging
#   Data: {'value': 10}
# TransformPlugin.process
# TransformPlugin.transform
# Result: {'value': 20}
```

---

## ❓ Interview Questions

### Q1: What is Method Resolution Order (MRO)?

**Answer:**

MRO is the order in which Python searches for methods in a class hierarchy. Python uses the C3 linearization algorithm to determine MRO.

```python
class A:
    def method(self):
        return "A"

class B(A):
    pass

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

# MRO: D -> B -> C -> A -> object
print(D.mro())

d = D()
print(d.method())  # "C" (found in C according to MRO)
```

**Key points:**

- Children before parents
- Left-to-right order of parent classes
- Each class appears only once
- You can check MRO with `Class.mro()` or `Class.__mro__`

### Q2: How does super() work in multiple inheritance?

**Answer:**

`super()` doesn't call the parent class—it calls the **next class in the MRO**.

```python
class A:
    def method(self):
        print("A")

class B(A):
    def method(self):
        print("B")
        super().method()  # Calls C, not A!

class C(A):
    def method(self):
        print("C")
        super().method()  # Calls A

class D(B, C):
    def method(self):
        print("D")
        super().method()  # Calls B

# MRO: D -> B -> C -> A
print([c.__name__ for c in D.mro()])

d = D()
d.method()
# Output:
# D
# B
# C
# A
```

**Key insight:** `super()` follows the MRO, enabling cooperative multiple inheritance.

### Q3: What is the Diamond Problem and how does Python solve it?

**Answer:**

The Diamond Problem occurs when a class inherits from two classes that both inherit from the same base class.

```python
#     A
#    / \
#   B   C
#    \ /
#     D

class A:
    def __init__(self):
        print("A.__init__")

class B(A):
    def __init__(self):
        print("B.__init__")
        super().__init__()

class C(A):
    def __init__(self):
        print("C.__init__")
        super().__init__()

class D(B, C):
    def __init__(self):
        print("D.__init__")
        super().__init__()

d = D()
# Output:
# D.__init__
# B.__init__
# C.__init__
# A.__init__  <- Called once, not twice!
```

**Solution:** Python's C3 linearization ensures each class in the hierarchy is called exactly once, in a consistent order.

### Q4: When should mixins come before or after the base class?

**Answer:**

**Mixins should come BEFORE the base class** to allow them to intercept and modify base class behavior.

```python
# ✅ CORRECT: Mixin before base
class LoggingMixin:
    def save(self):
        print("Logging...")
        super().save()

class Model:
    def save(self):
        print("Saving...")
        super().save() if hasattr(super(), 'save') else None

class User(LoggingMixin, Model):  # Mixin first!
    pass

user = User()
user.save()
# Output:
# Logging...
# Saving...

# ❌ WRONG: Base before mixin
class User(Model, LoggingMixin):  # Mixin last!
    pass

user = User()
user.save()
# Output:
# Saving...
# (Logging never called if Model.save doesn't call super()!)
```

**Rule:** Place classes in MRO from most specific (mixins) to most general (base).

### Q5: How do you debug MRO issues?

**Answer:**

**Steps to debug MRO issues:**

1. **Print the MRO:**

```python
print(MyClass.mro())
print([c.__name__ for c in MyClass.mro()])
```

2. **Trace method calls:**

```python
class Base:
    def method(self):
        print(f"Base.method (called from {self.__class__.__name__})")
        super().method() if hasattr(super(), 'method') else None
```

3. **Check for inconsistent super() usage:**

```python
# All classes should use super() or none should
```

4. **Visualize the hierarchy:**

```python
def print_hierarchy(cls, indent=0):
    print("  " * indent + cls.__name__)
    for base in cls.__bases__:
        print_hierarchy(base, indent + 1)

print_hierarchy(MyClass)
```

5. **Use Python's error messages:**

```python
# Python will raise TypeError if MRO cannot be computed
# Example: "Cannot create a consistent method resolution order"
```

---

## 📚 Summary

### Key Takeaways

1. **MRO** determines method lookup order in inheritance
2. Python uses **C3 linearization** algorithm for MRO
3. **super()** follows MRO, not just parent class
4. MRO ensures each class is called **only once**
5. Check MRO with `.mro()` or `.__mro__`
6. **Mixins before base classes** in multiple inheritance
7. Use **super() consistently** for cooperative inheritance
8. **Diamond Problem** solved by C3 linearization
9. MRO order: **children → parents → left to right**
10. Use **`**kwargs`\*\* for flexible initialization chains

### Best Practices Checklist

✅ Always use `super()` for cooperative inheritance
✅ Use `**kwargs` for passing parameters through MRO
✅ Check MRO when debugging inheritance issues
✅ Place mixins before base classes
✅ Use `super()` consistently across all classes
✅ Avoid deep inheritance hierarchies
✅ Document expected MRO for complex hierarchies
✅ Test super() chains thoroughly
✅ Use `__init_subclass__` to validate MRO
✅ Profile MRO overhead in performance-critical code

### MRO Rules

1. **Child before parent**: Subclasses come before their parents
2. **Left to right**: Parents listed left-to-right in class definition
3. **Monotonicity**: If A comes before B in one context, it does everywhere
4. **Single appearance**: Each class appears exactly once
5. **Object last**: `object` is always last in MRO

### Common Patterns

| Pattern               | Use Case                 | Example                         |
| --------------------- | ------------------------ | ------------------------------- |
| Cooperative super()   | Multiple inheritance     | Django mixins                   |
| \*\*kwargs forwarding | Flexible initialization  | Complex class hierarchies       |
| Mixin ordering        | Feature composition      | LoginMixin before View          |
| MRO validation        | Security/constraints     | Enforce security in subclasses  |
| Method caching        | Performance optimization | Cache frequently called methods |

---

**Next**: [04_Dunder_Methods.md](04_Dunder_Methods.md)
