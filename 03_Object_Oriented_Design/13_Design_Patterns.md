# 🎭 Design Patterns in Python

## Overview

Design patterns are reusable solutions to common software design problems. Understanding these patterns is essential for writing maintainable, scalable code.

---

## Creational Patterns

### 1. **Singleton Pattern**

```python
class Singleton:
    """Ensure only one instance exists"""
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Usage
s1 = Singleton()
s2 = Singleton()
print(s1 is s2)  # True

# Thread-safe singleton
import threading

class ThreadSafeSingleton:
    """Thread-safe singleton"""
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

# Singleton with decorator
def singleton(cls):
    """Singleton decorator"""
    instances = {}
    lock = threading.Lock()

    def get_instance(*args, **kwargs):
        if cls not in instances:
            with lock:
                if cls not in instances:
                    instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

@singleton
class Database:
    """Database connection singleton"""
    def __init__(self):
        print("Initializing database connection")
```

### 2. **Factory Pattern**

```python
from abc import ABC, abstractmethod

# Product interface
class Animal(ABC):
    @abstractmethod
    def speak(self) -> str:
        pass

# Concrete products
class Dog(Animal):
    def speak(self) -> str:
        return "Woof!"

class Cat(Animal):
    def speak(self) -> str:
        return "Meow!"

# Factory
class AnimalFactory:
    """Create animals based on type"""

    @staticmethod
    def create_animal(animal_type: str) -> Animal:
        """Factory method"""
        if animal_type == "dog":
            return Dog()
        elif animal_type == "cat":
            return Cat()
        else:
            raise ValueError(f"Unknown animal type: {animal_type}")

# Usage
factory = AnimalFactory()
dog = factory.create_animal("dog")
print(dog.speak())  # "Woof!"

# Factory Method Pattern
class AnimalCreator(ABC):
    """Abstract creator"""

    @abstractmethod
    def factory_method(self) -> Animal:
        """Override in subclasses"""
        pass

    def make_sound(self) -> str:
        """Use factory method"""
        animal = self.factory_method()
        return animal.speak()

class DogCreator(AnimalCreator):
    def factory_method(self) -> Animal:
        return Dog()

class CatCreator(AnimalCreator):
    def factory_method(self) -> Animal:
        return Cat()
```

### 3. **Builder Pattern**

```python
class Pizza:
    """Complex object to build"""

    def __init__(self):
        self.dough = None
        self.sauce = None
        self.toppings = []

    def __str__(self):
        return f"Pizza(dough={self.dough}, sauce={self.sauce}, toppings={self.toppings})"

class PizzaBuilder:
    """Builder for Pizza"""

    def __init__(self):
        self.pizza = Pizza()

    def set_dough(self, dough: str):
        """Set dough type"""
        self.pizza.dough = dough
        return self

    def set_sauce(self, sauce: str):
        """Set sauce type"""
        self.pizza.sauce = sauce
        return self

    def add_topping(self, topping: str):
        """Add topping"""
        self.pizza.toppings.append(topping)
        return self

    def build(self) -> Pizza:
        """Return built pizza"""
        return self.pizza

# Usage - method chaining
pizza = (PizzaBuilder()
    .set_dough("thin")
    .set_sauce("tomato")
    .add_topping("mozzarella")
    .add_topping("pepperoni")
    .build())

print(pizza)
```

### 4. **Prototype Pattern**

```python
import copy

class Prototype:
    """Prototype with clone method"""

    def clone(self):
        """Deep copy of self"""
        return copy.deepcopy(self)

class Document(Prototype):
    """Document with complex structure"""

    def __init__(self, title, content, metadata):
        self.title = title
        self.content = content
        self.metadata = metadata

    def __str__(self):
        return f"Document({self.title})"

# Usage - clone and modify
original = Document(
    "Original",
    "Content here",
    {"author": "John", "date": "2024-01-01"}
)

# Clone instead of creating from scratch
copy1 = original.clone()
copy1.title = "Copy 1"
copy1.metadata["author"] = "Jane"

print(original)  # Original metadata unchanged
print(copy1)
```

---

## Structural Patterns

### 1. **Adapter Pattern**

```python
# Legacy interface
class OldPaymentSystem:
    """Legacy payment system"""

    def make_payment(self, amount):
        return f"Old system: Paid ${amount}"

# New interface
class PaymentProcessor(ABC):
    """Modern payment interface"""

    @abstractmethod
    def process_payment(self, amount: float) -> str:
        pass

# Adapter
class PaymentAdapter(PaymentProcessor):
    """Adapt old system to new interface"""

    def __init__(self, old_system: OldPaymentSystem):
        self.old_system = old_system

    def process_payment(self, amount: float) -> str:
        """Translate new interface to old"""
        return self.old_system.make_payment(amount)

# Usage
old_system = OldPaymentSystem()
adapter = PaymentAdapter(old_system)
result = adapter.process_payment(100.0)
```

### 2. **Decorator Pattern**

```python
# Component interface
class Coffee(ABC):
    @abstractmethod
    def cost(self) -> float:
        pass

    @abstractmethod
    def description(self) -> str:
        pass

# Concrete component
class SimpleCoffee(Coffee):
    def cost(self) -> float:
        return 2.0

    def description(self) -> str:
        return "Simple coffee"

# Decorator base
class CoffeeDecorator(Coffee):
    def __init__(self, coffee: Coffee):
        self._coffee = coffee

# Concrete decorators
class MilkDecorator(CoffeeDecorator):
    def cost(self) -> float:
        return self._coffee.cost() + 0.5

    def description(self) -> str:
        return self._coffee.description() + ", milk"

class SugarDecorator(CoffeeDecorator):
    def cost(self) -> float:
        return self._coffee.cost() + 0.2

    def description(self) -> str:
        return self._coffee.description() + ", sugar"

# Usage - wrap components
coffee = SimpleCoffee()
coffee = MilkDecorator(coffee)
coffee = SugarDecorator(coffee)

print(coffee.description())  # "Simple coffee, milk, sugar"
print(f"${coffee.cost()}")   # "$2.7"

# Python decorator pattern (function decorators)
def log_execution(func):
    """Decorator to log function execution"""
    def wrapper(*args, **kwargs):
        print(f"Executing {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Completed {func.__name__}")
        return result
    return wrapper

@log_execution
def process_data(data):
    return data.upper()
```

### 3. **Facade Pattern**

```python
# Complex subsystems
class CPU:
    def freeze(self): print("CPU: Freezing")
    def jump(self, position): print(f"CPU: Jumping to {position}")
    def execute(self): print("CPU: Executing")

class Memory:
    def load(self, position, data): print(f"Memory: Loading {data} at {position}")

class HardDrive:
    def read(self, sector, size): return f"Data from sector {sector}"

# Facade - simple interface
class ComputerFacade:
    """Simplify computer startup"""

    def __init__(self):
        self.cpu = CPU()
        self.memory = Memory()
        self.hard_drive = HardDrive()

    def start(self):
        """Simple start method hiding complexity"""
        self.cpu.freeze()
        boot_data = self.hard_drive.read(0, 1024)
        self.memory.load(0, boot_data)
        self.cpu.jump(0)
        self.cpu.execute()

# Usage - simple interface
computer = ComputerFacade()
computer.start()  # All complex operations hidden
```

### 4. **Proxy Pattern**

```python
# Subject interface
class Image(ABC):
    @abstractmethod
    def display(self):
        pass

# Real subject
class RealImage(Image):
    """Heavy resource"""

    def __init__(self, filename):
        self.filename = filename
        self._load_from_disk()

    def _load_from_disk(self):
        print(f"Loading {self.filename} from disk (expensive)")

    def display(self):
        print(f"Displaying {self.filename}")

# Proxy
class ImageProxy(Image):
    """Lazy loading proxy"""

    def __init__(self, filename):
        self.filename = filename
        self._real_image = None

    def display(self):
        """Load only when needed"""
        if self._real_image is None:
            self._real_image = RealImage(self.filename)
        self._real_image.display()

# Usage
image = ImageProxy("photo.jpg")
# Not loaded yet
image.display()  # Loads now
image.display()  # Already loaded
```

---

## Behavioral Patterns

### 1. **Strategy Pattern**

```python
# Strategy interface
class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount: float):
        pass

# Concrete strategies
class CreditCardPayment(PaymentStrategy):
    def __init__(self, card_number):
        self.card_number = card_number

    def pay(self, amount: float):
        print(f"Paid ${amount} with credit card {self.card_number}")

class PayPalPayment(PaymentStrategy):
    def __init__(self, email):
        self.email = email

    def pay(self, amount: float):
        print(f"Paid ${amount} via PayPal ({self.email})")

# Context
class ShoppingCart:
    """Context using strategy"""

    def __init__(self):
        self.items = []
        self.payment_strategy = None

    def add_item(self, item, price):
        self.items.append((item, price))

    def set_payment_strategy(self, strategy: PaymentStrategy):
        self.payment_strategy = strategy

    def checkout(self):
        total = sum(price for _, price in self.items)
        if self.payment_strategy:
            self.payment_strategy.pay(total)

# Usage - swap strategies
cart = ShoppingCart()
cart.add_item("Book", 20.0)
cart.add_item("Pen", 5.0)

cart.set_payment_strategy(CreditCardPayment("1234-5678"))
cart.checkout()

cart.set_payment_strategy(PayPalPayment("user@email.com"))
cart.checkout()
```

### 2. **Observer Pattern**

```python
# Subject
class Subject:
    """Observable subject"""

    def __init__(self):
        self._observers = []
        self._state = None

    def attach(self, observer):
        """Add observer"""
        self._observers.append(observer)

    def detach(self, observer):
        """Remove observer"""
        self._observers.remove(observer)

    def notify(self):
        """Notify all observers"""
        for observer in self._observers:
            observer.update(self._state)

    def set_state(self, state):
        """Change state and notify"""
        self._state = state
        self.notify()

# Observer interface
class Observer(ABC):
    @abstractmethod
    def update(self, state):
        pass

# Concrete observers
class EmailObserver(Observer):
    def update(self, state):
        print(f"Email: State changed to {state}")

class SMSObserver(Observer):
    def update(self, state):
        print(f"SMS: State changed to {state}")

# Usage
subject = Subject()
subject.attach(EmailObserver())
subject.attach(SMSObserver())

subject.set_state("new state")  # Both observers notified
```

### 3. **Command Pattern**

```python
# Command interface
class Command(ABC):
    @abstractmethod
    def execute(self):
        pass

    @abstractmethod
    def undo(self):
        pass

# Receiver
class Light:
    """Command receiver"""

    def on(self):
        print("Light is ON")

    def off(self):
        print("Light is OFF")

# Concrete commands
class LightOnCommand(Command):
    def __init__(self, light: Light):
        self.light = light

    def execute(self):
        self.light.on()

    def undo(self):
        self.light.off()

class LightOffCommand(Command):
    def __init__(self, light: Light):
        self.light = light

    def execute(self):
        self.light.off()

    def undo(self):
        self.light.on()

# Invoker
class RemoteControl:
    """Execute commands"""

    def __init__(self):
        self.history = []

    def execute(self, command: Command):
        command.execute()
        self.history.append(command)

    def undo(self):
        if self.history:
            command = self.history.pop()
            command.undo()

# Usage
light = Light()
remote = RemoteControl()

on_command = LightOnCommand(light)
off_command = LightOffCommand(light)

remote.execute(on_command)   # Light is ON
remote.execute(off_command)  # Light is OFF
remote.undo()                # Light is ON
```

### 4. **Iterator Pattern**

```python
# Built-in Python iteration protocol
class MyCollection:
    """Custom collection with iterator"""

    def __init__(self):
        self._items = []

    def add(self, item):
        self._items.append(item)

    def __iter__(self):
        """Return iterator"""
        return MyIterator(self._items)

class MyIterator:
    """Custom iterator"""

    def __init__(self, items):
        self._items = items
        self._index = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self._index < len(self._items):
            item = self._items[self._index]
            self._index += 1
            return item
        raise StopIteration

# Usage
collection = MyCollection()
collection.add(1)
collection.add(2)
collection.add(3)

for item in collection:
    print(item)

# Generator as iterator (simpler)
def fibonacci(n):
    """Fibonacci iterator"""
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

for num in fibonacci(10):
    print(num)
```

---

## Best Practices

### ✅ Do's:

1. **Use patterns appropriately** - don't force them
2. **Favor composition** over inheritance
3. **Keep it simple** - don't over-engineer
4. **Use Python features** - decorators, context managers
5. **Follow SOLID principles**
6. **Document pattern usage**

### ❌ Don'ts:

1. **Don't use patterns everywhere**
2. **Don't implement patterns** you don't need
3. **Don't ignore Python idioms**
4. **Don't cargo cult** - understand why
5. **Don't sacrifice readability**

---

## Interview Questions

### Q1: What is the Singleton pattern and when to use it?

**Answer**: Ensures only one instance of a class exists:

- **Use for**: Database connections, configuration, logging
- **Implement with**: `__new__` method or decorator
- **Thread-safe**: Use lock for multi-threading
- **Consider**: May make testing harder, could violate SRP

### Q2: Explain Factory vs Builder pattern.

**Answer**:

- **Factory**: Creates objects based on type/condition, simple creation
- **Builder**: Constructs complex objects step-by-step, fluent interface
  Factory for simple object creation, Builder for complex configuration.

### Q3: What is the Strategy pattern?

**Answer**: Defines family of algorithms, makes them interchangeable:

- Encapsulates algorithms in separate classes
- Client can switch strategies at runtime
- Example: Different payment methods, sorting algorithms
- Follows Open/Closed principle

### Q4: Difference between Adapter and Facade?

**Answer**:

- **Adapter**: Converts interface to match expected interface (one-to-one)
- **Facade**: Simplifies complex subsystem (many-to-one)
  Adapter wraps incompatible interface, Facade provides simple interface.

### Q5: What is the Observer pattern?

**Answer**: One-to-many dependency where observers get notified of state changes:

- Subject maintains list of observers
- Observers implement update method
- Used in: Event systems, MVC, pub/sub
- Python: Can use callbacks or reactive libraries

---

## Summary

Key design patterns:

- **Creational**: Singleton, Factory, Builder (object creation)
- **Structural**: Adapter, Decorator, Facade, Proxy (object composition)
- **Behavioral**: Strategy, Observer, Command, Iterator (object interaction)

Use patterns to solve problems, not to show knowledge! 🎭
