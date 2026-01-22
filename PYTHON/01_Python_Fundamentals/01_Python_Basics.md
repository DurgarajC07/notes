# Python Basics - Core Fundamentals

## 📖 Concept Explanation

Python is a high-level, interpreted, dynamically-typed programming language known for its readability and simplicity. Understanding the basics is crucial for writing clean, maintainable code.

### Python Syntax & Indentation

Python uses **indentation** (not braces) to define code blocks. This enforces readable code structure.

```python
# Correct indentation
if True:
    print("Indented correctly")

# IndentationError
if True:
print("Wrong indentation")
```

### Keywords & Reserved Words

Python has 35 keywords (as of Python 3.11) that cannot be used as variable names:

```python
import keyword
print(keyword.kwlist)
# ['False', 'None', 'True', 'and', 'as', 'assert', 'async', 'await',
#  'break', 'class', 'continue', 'def', 'del', 'elif', 'else', 'except',
#  'finally', 'for', 'from', 'global', 'if', 'import', 'in', 'is',
#  'lambda', 'nonlocal', 'not', 'or', 'pass', 'raise', 'return',
#  'try', 'while', 'with', 'yield']
```

## 🧠 Why It Matters in Real Projects

- **Readability**: Python's clean syntax reduces cognitive load in large codebases
- **Maintenance**: Enforced indentation prevents formatting inconsistencies across teams
- **Productivity**: Less boilerplate means faster development cycles
- **Onboarding**: New team members can understand Python code quickly

## ⚙️ Internal Working

### CPython Execution Flow

1. **Source Code** (.py file)
2. **Lexical Analysis**: Tokenization by the parser
3. **Parsing**: Creates Abstract Syntax Tree (AST)
4. **Compilation**: AST → Bytecode (.pyc files)
5. **Execution**: Python Virtual Machine (PVM) executes bytecode

```python
import dis

def simple_function():
    x = 10
    return x + 5

# View bytecode
dis.dis(simple_function)
"""
  2           0 LOAD_CONST               1 (10)
              2 STORE_FAST               0 (x)

  3           4 LOAD_FAST                0 (x)
              6 LOAD_CONST               2 (5)
              8 BINARY_ADD
             10 RETURN_VALUE
"""
```

## ✅ Best Practices

1. **Follow PEP 8** - Python's official style guide

```python
# Good
def calculate_total_price(items, tax_rate):
    return sum(items) * (1 + tax_rate)

# Bad
def CalculateTotalPrice(items,taxRate):
    return sum(items)*(1+taxRate)
```

2. **Use 4 spaces for indentation** (not tabs)

3. **Maximum line length: 79 characters** for code, 72 for docstrings

4. **Naming conventions**:
   - `snake_case` for functions and variables
   - `PascalCase` for classes
   - `UPPER_CASE` for constants

```python
# Good naming
MAX_CONNECTIONS = 100
user_count = 0

class UserRepository:
    def get_user_by_id(self, user_id):
        pass
```

## ❌ Common Mistakes

### 1. Mixing Tabs and Spaces

```python
# Bad - will cause TabError in Python 3
def bad_function():
    if True:
        print("4 spaces")
	print("1 tab")  # TabError!
```

### 2. Using Reserved Keywords as Variable Names

```python
# Bad
class = "Python"  # SyntaxError
for = 10          # SyntaxError

# Good
class_name = "Python"
loop_count = 10
```

### 3. Forgetting the Colon

```python
# Bad
if x > 5
    print("Greater")  # SyntaxError

# Good
if x > 5:
    print("Greater")
```

### 4. Inconsistent Indentation

```python
# Bad
def process_data():
    if True:
        print("2 spaces")
      print("Different indentation")  # IndentationError
```

## 🔐 Security Considerations

### 1. Avoid `eval()` with User Input

```python
# DANGEROUS - never do this
user_input = input("Enter expression: ")
result = eval(user_input)  # User could input: __import__('os').system('rm -rf /')

# Safe alternative
import ast
def safe_eval(expression):
    try:
        return ast.literal_eval(expression)  # Only evaluates literals
    except (ValueError, SyntaxError):
        return None
```

### 2. Don't Use `exec()` with Untrusted Code

```python
# DANGEROUS
code = request.data.get('code')
exec(code)  # Arbitrary code execution vulnerability

# Safe: Use sandboxed environments or avoid entirely
```

## 🚀 Performance Optimization Techniques

### 1. Use Built-in Functions (Implemented in C)

```python
# Slower
result = []
for i in range(1000):
    result.append(i * 2)

# Faster
result = list(map(lambda x: x * 2, range(1000)))

# Even faster (list comprehension)
result = [i * 2 for i in range(1000)]
```

### 2. String Concatenation

```python
# Slow for large datasets
result = ""
for i in range(10000):
    result += str(i)

# Fast
result = "".join(str(i) for i in range(10000))
```

## 🧪 Code Examples

### Simple to Advanced Examples

#### Basic: Variables and Types

```python
# Dynamic typing
x = 10           # int
x = "Hello"      # str (reassignment)
x = [1, 2, 3]    # list

# Type hints (Python 3.5+)
def greet(name: str) -> str:
    return f"Hello, {name}!"

# Multiple assignment
a, b, c = 1, 2, 3
```

#### Intermediate: Control Flow

```python
# Match-case (Python 3.10+)
def handle_status_code(code: int) -> str:
    match code:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500 | 502 | 503:
            return "Server Error"
        case _:
            return "Unknown"

# Walrus operator (Python 3.8+)
if (n := len(data)) > 10:
    print(f"Large dataset: {n} items")
```

#### Advanced: Context Managers

```python
# Custom context manager
class Timer:
    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, *args):
        self.end = time.time()
        print(f"Elapsed: {self.end - self.start:.2f}s")

with Timer():
    # Some expensive operation
    time.sleep(2)
```

## 🏗️ Real-World Use Cases

### 1. Configuration Management

```python
import os
from typing import Optional

class Config:
    """Production-grade configuration with environment variables"""

    def __init__(self):
        self.debug: bool = os.getenv('DEBUG', 'False').lower() == 'true'
        self.database_url: str = os.getenv('DATABASE_URL', 'sqlite:///db.sqlite3')
        self.secret_key: Optional[str] = os.getenv('SECRET_KEY')

        if not self.secret_key:
            raise ValueError("SECRET_KEY environment variable must be set")

    def __repr__(self):
        # Don't expose secrets
        return f"Config(debug={self.debug}, database_url={self.database_url[:10]}...)"

config = Config()
```

### 2. Error-Safe User Input Processing

```python
def get_integer_input(prompt: str, min_val: int = None, max_val: int = None) -> int:
    """Robust integer input with validation"""
    while True:
        try:
            value = int(input(prompt))

            if min_val is not None and value < min_val:
                print(f"Value must be at least {min_val}")
                continue

            if max_val is not None and value > max_val:
                print(f"Value must be at most {max_val}")
                continue

            return value
        except ValueError:
            print("Invalid input. Please enter a number.")
        except KeyboardInterrupt:
            print("\nOperation cancelled.")
            raise
```

## ❓ Frequently Asked Interview Questions

### Q1: What is the difference between `is` and `==`?

**Answer:**

- `==` checks **value equality**
- `is` checks **identity** (same object in memory)

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)  # True (same values)
print(a is b)  # False (different objects)
print(a is c)  # True (same object)

# Integer caching (-5 to 256)
x = 256
y = 256
print(x is y)  # True (cached)

x = 257
y = 257
print(x is y)  # False (not cached, different objects)
```

### Q2: Explain Python's indentation rules

**Answer:**

- Python uses indentation to define code blocks (no braces like C/Java)
- Standard is 4 spaces per indentation level
- Mixing tabs and spaces raises `TabError` in Python 3
- All statements in a block must have the same indentation

### Q3: What are Python's naming conventions?

**Answer:**

- **Variables/Functions**: `snake_case` - `user_name`, `calculate_total()`
- **Classes**: `PascalCase` - `UserRepository`, `HTTPRequest`
- **Constants**: `UPPER_CASE` - `MAX_SIZE`, `API_KEY`
- **Private**: Single underscore prefix - `_internal_method()`
- **Name mangling**: Double underscore - `__private_var`
- **Magic methods**: Double underscore both sides - `__init__`, `__str__`

### Q4: How does Python execute code?

**Answer:**

1. **Parsing**: Source code → Abstract Syntax Tree (AST)
2. **Compilation**: AST → Bytecode (.pyc files stored in `__pycache__`)
3. **Execution**: Python Virtual Machine interprets bytecode
4. **Optimization**: CPython caches bytecode to avoid recompilation

```python
# View AST
import ast
code = "x = 10 + 5"
tree = ast.parse(code)
print(ast.dump(tree))

# View bytecode
import dis
dis.dis(compile(code, '<string>', 'exec'))
```

### Q5: What is PEP 8 and why is it important?

**Answer:**
PEP 8 is Python's official style guide that ensures:

- **Consistency**: Uniform code style across projects
- **Readability**: Easier code review and collaboration
- **Maintainability**: Reduces cognitive load

Key rules:

- 4 spaces for indentation
- Max 79 characters per line
- 2 blank lines between top-level functions/classes
- 1 blank line between methods
- Imports at the top, grouped by standard library → third-party → local

### Q6: Explain the difference between statement and expression

**Answer:**

- **Statement**: Performs an action (no return value)
- **Expression**: Evaluates to a value

```python
# Statements (no value)
if x > 5:
    pass
for i in range(10):
    pass

# Expressions (has value)
x + 5           # Returns a value
[i for i in range(10)]  # List comprehension
lambda x: x * 2  # Lambda function

# Assignment is a statement in Python
# (unlike C where it's an expression)
x = 10  # No return value
```

### Q7: What happens when you run a Python script?

**Answer:**

```python
# When you run: python script.py

# 1. Python checks __pycache__ for compiled bytecode
# 2. If not found or source is newer, compiles .py → .pyc
# 3. Loads bytecode into memory
# 4. Python VM executes bytecode instruction by instruction
# 5. __name__ is set to '__main__'

if __name__ == '__main__':
    # This block only runs when script is executed directly
    # Not when imported as a module
    main()
```

### Q8: How do you handle different Python versions in a project?

**Answer:**

```python
import sys

# Check minimum version
if sys.version_info < (3, 8):
    raise RuntimeError("Python 3.8 or higher required")

# Feature-specific checking
try:
    from typing import TypedDict  # Python 3.8+
except ImportError:
    from typing_extensions import TypedDict

# Version-specific code
if sys.version_info >= (3, 10):
    # Use match-case
    match value:
        case 1:
            print("One")
else:
    # Fallback to if-elif
    if value == 1:
        print("One")
```

## 🧩 Design Patterns

### 1. Singleton Pattern for Configuration

```python
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Thread-safe singleton
import threading

class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
```

### 2. Factory Pattern for Object Creation

```python
class Animal:
    def speak(self):
        pass

class Dog(Animal):
    def speak(self):
        return "Woof!"

class Cat(Animal):
    def speak(self):
        return "Meow!"

class AnimalFactory:
    @staticmethod
    def create_animal(animal_type: str) -> Animal:
        if animal_type == "dog":
            return Dog()
        elif animal_type == "cat":
            return Cat()
        else:
            raise ValueError(f"Unknown animal type: {animal_type}")

# Usage
animal = AnimalFactory.create_animal("dog")
print(animal.speak())  # Woof!
```

## 📚 Additional Resources

- **PEP 8**: https://pep8.org/
- **Python Enhancement Proposals**: https://www.python.org/dev/peps/
- **CPython Source Code**: https://github.com/python/cpython
- **Python Internals**: https://realpython.com/cpython-source-code-guide/
