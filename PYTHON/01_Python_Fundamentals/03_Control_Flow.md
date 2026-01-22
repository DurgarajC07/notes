# Control Flow - Conditional Statements and Loops

## 📖 Concept Explanation

Control flow determines the order in which code executes. Python provides several control flow statements: conditionals, loops, and control statements.

### Conditional Statements

```python
# if-elif-else
score = 85

if score >= 90:
    grade = 'A'
elif score >= 80:
    grade = 'B'
elif score >= 70:
    grade = 'C'
else:
    grade = 'F'

# Ternary operator (conditional expression)
status = "pass" if score >= 60 else "fail"

# Match-case (Python 3.10+)
def handle_http_status(status_code):
    match status_code:
        case 200:
            return "OK"
        case 201:
            return "Created"
        case 400 | 401 | 403:  # Multiple patterns
            return "Client Error"
        case 500 | 502 | 503:
            return "Server Error"
        case _:  # Default case
            return "Unknown Status"
```

### Loop Statements

```python
# For loop - iterate over sequences
for i in range(5):
    print(i)

# While loop - conditional iteration
count = 0
while count < 5:
    print(count)
    count += 1

# Loop control statements
for i in range(10):
    if i == 3:
        continue  # Skip iteration
    if i == 7:
        break  # Exit loop
    print(i)
else:
    # Executed if loop completes without break
    print("Loop completed normally")
```

## 🧠 Why It Matters in Real Projects

### 1. Business Logic Implementation

```python
def calculate_discount(total, customer_type, is_holiday):
    """Real-world discount calculation with complex logic"""
    discount = 0.0

    if customer_type == 'premium':
        discount = 0.20
    elif customer_type == 'regular':
        discount = 0.10
    elif customer_type == 'new':
        discount = 0.05

    if is_holiday:
        discount += 0.05

    # Cap maximum discount
    discount = min(discount, 0.30)

    return total * (1 - discount)
```

### 2. Input Validation

```python
def validate_user_input(data):
    """Multi-level validation with early returns"""
    if not data:
        return False, "Data cannot be empty"

    if not isinstance(data.get('email'), str):
        return False, "Email must be a string"

    if '@' not in data['email']:
        return False, "Invalid email format"

    return True, "Valid"
```

### 3. Data Processing Pipelines

```python
def process_transactions(transactions):
    """Process financial transactions with error handling"""
    processed = []
    errors = []

    for idx, transaction in enumerate(transactions):
        try:
            if transaction['amount'] <= 0:
                errors.append(f"Invalid amount at index {idx}")
                continue

            if transaction['status'] != 'pending':
                continue  # Skip non-pending transactions

            # Process transaction
            processed.append(process_single_transaction(transaction))

        except KeyError as e:
            errors.append(f"Missing field {e} at index {idx}")
        except Exception as e:
            errors.append(f"Error at index {idx}: {e}")

    return processed, errors
```

## ⚙️ Internal Working

### 1. If Statement Optimization

```python
# Python optimizes constant expressions at compile time
import dis

def test_if():
    if True:  # Compiler optimization
        return "always"
    return "never"

dis.dis(test_if)
# Notice: The 'if False' branch is completely removed by the compiler
```

### 2. Short-Circuit Evaluation

```python
# 'and' short-circuits - stops at first False
def expensive_check():
    print("Expensive check called")
    return True

# Second function never called if first is False
if False and expensive_check():
    pass  # expensive_check() is NOT called

# 'or' short-circuits - stops at first True
if True or expensive_check():
    pass  # expensive_check() is NOT called

# Practical use: avoid AttributeError
user = None
if user and user.is_admin:  # Safe - doesn't check is_admin if user is None
    grant_admin_access()
```

### 3. For Loop Implementation

```python
# For loops use iterators internally
numbers = [1, 2, 3]

# What Python does:
iterator = iter(numbers)  # Get iterator
while True:
    try:
        item = next(iterator)  # Get next item
        print(item)
    except StopIteration:  # No more items
        break
```

### 4. Match-Case Pattern Matching

```python
# Structural pattern matching (Python 3.10+)
def process_command(command):
    match command:
        case {'action': 'create', 'resource': resource}:
            return f"Creating {resource}"

        case {'action': 'delete', 'resource': resource, 'force': True}:
            return f"Force deleting {resource}"

        case {'action': action, 'resource': resource} if action in ['update', 'patch']:
            return f"Updating {resource}"

        case _:
            return "Unknown command"

# Usage
print(process_command({'action': 'create', 'resource': 'user'}))
print(process_command({'action': 'delete', 'resource': 'post', 'force': True}))
```

## ✅ Best Practices

### 1. Use Early Returns for Clarity

```python
# BAD - nested conditions
def process_user(user):
    if user is not None:
        if user.is_active:
            if user.has_permission('edit'):
                return perform_edit(user)
            else:
                return "No permission"
        else:
            return "User inactive"
    else:
        return "User not found"

# GOOD - early returns
def process_user(user):
    if user is None:
        return "User not found"

    if not user.is_active:
        return "User inactive"

    if not user.has_permission('edit'):
        return "No permission"

    return perform_edit(user)
```

### 2. Avoid Deep Nesting

```python
# BAD - hard to read
def validate_data(data):
    if data:
        if 'email' in data:
            if '@' in data['email']:
                if len(data['email']) > 5:
                    return True
    return False

# GOOD - flat structure
def validate_data(data):
    if not data:
        return False
    if 'email' not in data:
        return False
    if '@' not in data['email']:
        return False
    if len(data['email']) <= 5:
        return False
    return True

# BETTER - explicit conditions
def validate_data(data):
    return (
        data is not None
        and 'email' in data
        and '@' in data['email']
        and len(data['email']) > 5
    )
```

### 3. Use Enumerate Instead of Range(len())

```python
items = ['apple', 'banana', 'cherry']

# BAD
for i in range(len(items)):
    print(f"{i}: {items[i]}")

# GOOD
for i, item in enumerate(items):
    print(f"{i}: {item}")

# Start from custom index
for i, item in enumerate(items, start=1):
    print(f"{i}: {item}")
```

### 4. Use Loop Else Clause Appropriately

```python
# Search with for-else
def find_user(users, user_id):
    for user in users:
        if user['id'] == user_id:
            return user
    else:
        # Executed only if break was NOT called
        raise ValueError(f"User {user_id} not found")

# Retry logic with while-else
attempts = 0
max_attempts = 3

while attempts < max_attempts:
    if try_connect():
        break
    attempts += 1
else:
    # Executed if loop completed without break
    raise ConnectionError("Failed to connect after 3 attempts")
```

### 5. Use Match-Case for Complex Conditionals

```python
# Instead of multiple if-elif
def get_http_method_info(method):
    match method.upper():
        case 'GET':
            return 'Retrieve resource'
        case 'POST':
            return 'Create resource'
        case 'PUT' | 'PATCH':
            return 'Update resource'
        case 'DELETE':
            return 'Remove resource'
        case _:
            return 'Unknown method'
```

## ❌ Common Mistakes

### 1. Using Assignment in Condition (Not an Error, But Confusing)

```python
# Python doesn't allow assignment in if conditions directly
# if (x = 10):  # SyntaxError

# Use walrus operator (Python 3.8+)
if (x := expensive_computation()) > 10:
    print(f"Result {x} is greater than 10")
```

### 2. Modifying List While Iterating

```python
# BAD - unpredictable behavior
numbers = [1, 2, 3, 4, 5]
for i, num in enumerate(numbers):
    if num % 2 == 0:
        numbers.pop(i)  # Modifies list during iteration!

# GOOD - iterate over copy
numbers = [1, 2, 3, 4, 5]
for num in numbers[:]:
    if num % 2 == 0:
        numbers.remove(num)

# BEST - list comprehension
numbers = [num for num in numbers if num % 2 != 0]
```

### 3. Infinite Loops with While

```python
# BAD - easy to create infinite loop
count = 0
while count < 10:
    print(count)
    # Forgot to increment count!

# GOOD - always ensure loop variable changes
count = 0
while count < 10:
    print(count)
    count += 1

# BETTER - use for loop when possible
for count in range(10):
    print(count)
```

### 4. Not Breaking Out of Nested Loops Properly

```python
# BAD - break only exits inner loop
found = False
for i in range(10):
    for j in range(10):
        if matrix[i][j] == target:
            found = True
            break  # Only exits inner loop
    if found:
        break  # Need second break

# GOOD - use function and return
def find_in_matrix(matrix, target):
    for i in range(len(matrix)):
        for j in range(len(matrix[i])):
            if matrix[i][j] == target:
                return (i, j)
    return None

# GOOD - use exception for control flow
class Found(Exception):
    pass

try:
    for i in range(10):
        for j in range(10):
            if matrix[i][j] == target:
                raise Found()
except Found:
    print(f"Found at ({i}, {j})")
```

### 5. Comparing with True/False Explicitly

```python
# BAD - unnecessary comparison
if is_active == True:
    pass

# GOOD - use truthiness
if is_active:
    pass

# BAD
if is_active == False:
    pass

# GOOD
if not is_active:
    pass
```

## 🔐 Security Considerations

### 1. Prevent Infinite Loops from User Input

```python
# DANGEROUS - no limit
user_count = int(input("How many iterations? "))
for i in range(user_count):  # Could be billions!
    process_item(i)

# SAFE - enforce maximum
MAX_ITERATIONS = 1000
user_count = int(input("How many iterations? "))

if user_count > MAX_ITERATIONS:
    raise ValueError(f"Maximum {MAX_ITERATIONS} iterations allowed")

for i in range(min(user_count, MAX_ITERATIONS)):
    process_item(i)
```

### 2. Avoid eval() in Conditionals

```python
# DANGEROUS - arbitrary code execution
condition = input("Enter condition: ")
if eval(condition):  # User could input malicious code
    pass

# SAFE - parse and validate
def safe_check(condition_str):
    # Whitelist allowed conditions
    allowed = ['is_admin', 'is_active', 'age > 18']
    if condition_str in allowed:
        # Safe evaluation logic here
        pass
```

### 3. Timeout for Long-Running Loops

```python
import time
import signal

def timeout_handler(signum, frame):
    raise TimeoutError("Loop execution timeout")

# Set timeout for loop
signal.signal(signal.SIGALRM, timeout_handler)
signal.alarm(5)  # 5 seconds timeout

try:
    while True:
        # Potentially long-running operation
        process_data()
except TimeoutError:
    print("Loop timed out")
finally:
    signal.alarm(0)  # Cancel alarm
```

## 🚀 Performance Optimization Techniques

### 1. Use List Comprehension Over Loops

```python
import timeit

# Slower - explicit loop
def loop_version():
    result = []
    for i in range(1000):
        result.append(i * 2)
    return result

# Faster - list comprehension (optimized in C)
def comprehension_version():
    return [i * 2 for i in range(1000)]

print(timeit.timeit(loop_version, number=10000))
print(timeit.timeit(comprehension_version, number=10000))
# Comprehension is ~30% faster
```

### 2. Use 'any' and 'all' for Conditional Checks

```python
# Slow - explicit loop
has_negative = False
for num in numbers:
    if num < 0:
        has_negative = True
        break

# Fast - built-in any()
has_negative = any(num < 0 for num in numbers)

# Check if all elements satisfy condition
all_positive = all(num > 0 for num in numbers)
```

### 3. Avoid Repeated Lookups in Loops

```python
# Slow - repeated attribute lookup
for item in items:
    result.append(item)  # Looks up 'append' each iteration

# Fast - cache method reference
append = result.append
for item in items:
    append(item)
```

### 4. Use Itertools for Complex Iterations

```python
from itertools import islice, chain, groupby

# Efficient iteration over large files
with open('large_file.txt') as f:
    # Process in chunks without loading entire file
    while chunk := list(islice(f, 1000)):
        process_chunk(chunk)

# Flatten nested lists efficiently
nested = [[1, 2], [3, 4], [5, 6]]
flat = list(chain.from_iterable(nested))

# Group consecutive items
data = [1, 1, 2, 2, 2, 3, 1, 1]
for key, group in groupby(data):
    print(f"{key}: {list(group)}")
```

## 🧪 Code Examples

### Example 1: Advanced Pattern Matching

```python
def process_api_response(response):
    """Handle different API response structures"""
    match response:
        case {'status': 'success', 'data': data}:
            return data

        case {'status': 'error', 'message': msg, 'code': code}:
            raise APIError(f"Error {code}: {msg}")

        case {'status': 'partial', 'data': data, 'warnings': warnings}:
            log_warnings(warnings)
            return data

        case _:
            raise ValueError("Unknown response format")

# Advanced pattern with guards
def classify_transaction(transaction):
    match transaction:
        case {'amount': amount, 'type': 'credit'} if amount > 10000:
            return 'large_credit'

        case {'amount': amount, 'type': 'debit'} if amount > 10000:
            flag_for_review(transaction)
            return 'large_debit'

        case {'type': 'credit'}:
            return 'normal_credit'

        case {'type': 'debit'}:
            return 'normal_debit'

        case _:
            return 'unknown'
```

### Example 2: Retry Logic with Exponential Backoff

```python
import time
import random

def retry_with_backoff(func, max_attempts=5, base_delay=1):
    """Retry function with exponential backoff"""

    for attempt in range(max_attempts):
        try:
            return func()
        except Exception as e:
            if attempt == max_attempts - 1:
                # Last attempt failed
                raise

            # Calculate delay with exponential backoff and jitter
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            print(f"Attempt {attempt + 1} failed: {e}")
            print(f"Retrying in {delay:.2f} seconds...")
            time.sleep(delay)

# Usage
def unstable_api_call():
    # Simulated API call that might fail
    if random.random() < 0.7:
        raise ConnectionError("Network error")
    return "Success"

result = retry_with_backoff(unstable_api_call)
```

### Example 3: State Machine with Match-Case

```python
class OrderStateMachine:
    """Production-ready order state machine"""

    def __init__(self):
        self.state = 'pending'

    def transition(self, event, **kwargs):
        match (self.state, event):
            case ('pending', 'confirm'):
                self.state = 'confirmed'
                self.send_confirmation_email()

            case ('confirmed', 'ship'):
                if self.validate_inventory():
                    self.state = 'shipped'
                    self.update_inventory()
                else:
                    raise ValueError("Insufficient inventory")

            case ('shipped', 'deliver'):
                self.state = 'delivered'
                self.notify_customer()

            case ('pending' | 'confirmed', 'cancel'):
                self.state = 'cancelled'
                self.process_refund()

            case (state, event):
                raise ValueError(f"Invalid transition: {state} -> {event}")

        return self.state
```

## 🏗️ Real-World Use Cases

### 1. API Request Validation Pipeline

```python
from typing import Dict, Tuple, Optional

def validate_api_request(data: Dict) -> Tuple[bool, Optional[str]]:
    """Multi-stage validation with early exit"""

    # Stage 1: Check required fields
    required_fields = ['user_id', 'action', 'timestamp']
    for field in required_fields:
        if field not in data:
            return False, f"Missing required field: {field}"

    # Stage 2: Type validation
    if not isinstance(data['user_id'], int):
        return False, "user_id must be an integer"

    if data['action'] not in ['create', 'read', 'update', 'delete']:
        return False, f"Invalid action: {data['action']}"

    # Stage 3: Business rules
    if data['action'] == 'delete' and not data.get('force', False):
        if has_dependencies(data['user_id']):
            return False, "Cannot delete user with dependencies"

    # Stage 4: Rate limiting check
    if is_rate_limited(data['user_id']):
        return False, "Rate limit exceeded"

    return True, None

# Usage in API handler
def handle_request(request_data):
    is_valid, error = validate_api_request(request_data)

    if not is_valid:
        return {'status': 'error', 'message': error}, 400

    # Process valid request
    result = process_request(request_data)
    return {'status': 'success', 'data': result}, 200
```

### 2. Batch Processing with Progress Tracking

```python
from typing import List, Callable
import time

def batch_process(
    items: List,
    processor: Callable,
    batch_size: int = 100,
    show_progress: bool = True
):
    """Process items in batches with error handling"""

    total = len(items)
    processed = 0
    errors = []

    for i in range(0, total, batch_size):
        batch = items[i:i + batch_size]

        for idx, item in enumerate(batch):
            try:
                processor(item)
                processed += 1

                if show_progress and (processed % 10 == 0):
                    progress = (processed / total) * 100
                    print(f"Progress: {progress:.1f}% ({processed}/{total})")

            except Exception as e:
                errors.append({
                    'index': i + idx,
                    'item': item,
                    'error': str(e)
                })

        # Small delay between batches to avoid overwhelming system
        time.sleep(0.1)

    return {
        'processed': processed,
        'failed': len(errors),
        'errors': errors
    }

# Usage
def process_user(user):
    # User processing logic
    update_user_profile(user)

users = get_all_users()
result = batch_process(users, process_user, batch_size=50)
```

### 3. Configuration-Driven Control Flow

```python
from typing import Dict, Any
from enum import Enum

class Environment(Enum):
    DEVELOPMENT = 'dev'
    STAGING = 'staging'
    PRODUCTION = 'prod'

class Config:
    def __init__(self, env: Environment):
        self.env = env
        self.settings = self._load_settings()

    def _load_settings(self) -> Dict[str, Any]:
        """Load environment-specific settings"""
        match self.env:
            case Environment.DEVELOPMENT:
                return {
                    'debug': True,
                    'log_level': 'DEBUG',
                    'cache_timeout': 60,
                    'enable_profiling': True
                }

            case Environment.STAGING:
                return {
                    'debug': True,
                    'log_level': 'INFO',
                    'cache_timeout': 300,
                    'enable_profiling': True
                }

            case Environment.PRODUCTION:
                return {
                    'debug': False,
                    'log_level': 'WARNING',
                    'cache_timeout': 3600,
                    'enable_profiling': False
                }

# Usage in application
config = Config(Environment.PRODUCTION)

if config.settings['debug']:
    enable_debug_toolbar()

if config.settings['enable_profiling']:
    start_profiler()
```

## ❓ Frequently Asked Interview Questions

### Q1: What is the difference between 'break', 'continue', and 'pass'?

**Answer:**

- **break**: Exits the loop entirely
- **continue**: Skips current iteration, continues with next
- **pass**: Does nothing (placeholder for empty code blocks)

```python
for i in range(5):
    if i == 2:
        break  # Stops at 2
    print(i)  # Prints: 0, 1

for i in range(5):
    if i == 2:
        continue  # Skips 2
    print(i)  # Prints: 0, 1, 3, 4

def empty_function():
    pass  # Placeholder - no operation

if condition:
    pass  # TODO: implement later
```

### Q2: Explain the 'else' clause in loops

**Answer:**
The `else` clause executes only if the loop completes normally (without `break`).

```python
def find_prime(numbers):
    for num in numbers:
        if num > 1:
            for i in range(2, num):
                if num % i == 0:
                    break  # Not prime
            else:
                # Loop completed without break - is prime
                return num
    return None

# Practical use: search with default behavior
def find_user(users, user_id):
    for user in users:
        if user.id == user_id:
            return user
    else:
        # Not found - create default
        return create_default_user()
```

### Q3: How does short-circuit evaluation work?

**Answer:**
Logical operators (`and`, `or`) stop evaluating as soon as the result is determined.

```python
# 'and' stops at first False
def expensive_check():
    print("Expensive check called")
    return True

if False and expensive_check():
    pass  # expensive_check() NOT called

# 'or' stops at first True
if True or expensive_check():
    pass  # expensive_check() NOT called

# Practical use: avoid None errors
user = get_user()
if user and user.is_admin:  # Safe - doesn't check is_admin if user is None
    grant_access()

# Chaining for default values
name = user.name or user.email or "Anonymous"
```

### Q4: What is the walrus operator?

**Answer:**
The walrus operator (`:=`) assigns and returns a value in one expression (Python 3.8+).

```python
# Without walrus - two lookups
data = get_data()
if data:
    process(data)

# With walrus - one lookup
if (data := get_data()):
    process(data)

# In while loops
while (line := file.readline()):
    process(line)

# In list comprehensions
[y for x in data if (y := transform(x)) > 10]

# Practical: avoid expensive computations
if (match := re.search(pattern, text)) and match.group(1):
    result = match.group(1)
```

### Q5: Explain match-case pattern matching

**Answer:**
Pattern matching (Python 3.10+) matches values against patterns, similar to switch statements but more powerful.

```python
# Simple matching
match status_code:
    case 200:
        return "OK"
    case 404:
        return "Not Found"
    case _:
        return "Unknown"

# Multiple patterns (OR)
match code:
    case 400 | 401 | 403:
        return "Client Error"

# Structural matching
match point:
    case (0, 0):
        return "Origin"
    case (0, y):
        return f"Y-axis at {y}"
    case (x, 0):
        return f"X-axis at {x}"
    case (x, y):
        return f"Point at ({x}, {y})"

# Dictionary matching
match response:
    case {'status': 'success', 'data': data}:
        return data
    case {'status': 'error', 'message': msg}:
        raise Error(msg)

# With guards
match value:
    case x if x < 0:
        return "Negative"
    case x if x == 0:
        return "Zero"
    case x if x > 0:
        return "Positive"
```

### Q6: What are the performance differences between loops?

**Answer:**

```python
import timeit

data = list(range(1000))

# For loop - fastest for simple iteration
def for_loop():
    result = []
    for x in data:
        result.append(x * 2)
    return result

# List comprehension - faster (C optimization)
def list_comp():
    return [x * 2 for x in data]

# Map - similar to comprehension
def map_func():
    return list(map(lambda x: x * 2, data))

# While loop - slowest
def while_loop():
    result = []
    i = 0
    while i < len(data):
        result.append(data[i] * 2)
        i += 1
    return result

print(timeit.timeit(for_loop, number=10000))      # ~0.8s
print(timeit.timeit(list_comp, number=10000))     # ~0.6s
print(timeit.timeit(map_func, number=10000))      # ~0.7s
print(timeit.timeit(while_loop, number=10000))    # ~1.2s
```

### Q7: How do you handle nested loop breaks?

**Answer:**

```python
# Method 1: Use function and return
def find_in_matrix(matrix, target):
    for i in range(len(matrix)):
        for j in range(len(matrix[i])):
            if matrix[i][j] == target:
                return (i, j)
    return None

# Method 2: Use flag variable
found = False
for i in range(len(matrix)):
    for j in range(len(matrix[i])):
        if matrix[i][j] == target:
            found = True
            break
    if found:
        break

# Method 3: Use exception (pythonic for deeply nested)
class BreakNestedLoop(Exception):
    pass

try:
    for i in range(len(matrix)):
        for j in range(len(matrix[i])):
            for k in range(len(matrix[i][j])):
                if condition:
                    raise BreakNestedLoop
except BreakNestedLoop:
    pass
```

### Q8: What are best practices for conditional expressions?

**Answer:**

```python
# ✅ GOOD - simple ternary
status = "active" if user.is_active else "inactive"

# ❌ BAD - complex nested ternary
result = "A" if score > 90 else "B" if score > 80 else "C" if score > 70 else "F"

# ✅ GOOD - use if-elif for readability
if score > 90:
    result = "A"
elif score > 80:
    result = "B"
elif score > 70:
    result = "C"
else:
    result = "F"

# ✅ GOOD - dictionary dispatch for many conditions
grade_map = {
    range(90, 101): "A",
    range(80, 90): "B",
    range(70, 80): "C",
    range(0, 70): "F"
}

def get_grade(score):
    for score_range, grade in grade_map.items():
        if score in score_range:
            return grade
    return "Invalid"
```

---

**Next**: [04_Functions.md](04_Functions.md)
