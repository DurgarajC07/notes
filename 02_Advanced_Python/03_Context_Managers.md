# Context Managers - Resource Management with `with` Statement

## 📖 Concept Explanation

Context managers are Python objects that enable automatic setup and teardown of resources using the `with` statement. They implement the **Context Manager Protocol** through `__enter__()` and `__exit__()` methods.

### Basic Context Manager

```python
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None
    
    def __enter__(self):
        """Called when entering 'with' block"""
        self.file = open(self.filename, self.mode)
        return self.file
    
    def __exit__(self, exc_type, exc_value, traceback):
        """Called when exiting 'with' block"""
        if self.file:
            self.file.close()
        # Return False to propagate exceptions
        return False

# Usage
with FileManager('test.txt', 'w') as f:
    f.write('Hello, World!')
# File automatically closed
```

### Built-in Context Managers

```python
# File handling
with open('file.txt', 'r') as f:
    content = f.read()

# Threading locks
import threading
lock = threading.Lock()
with lock:
    # Critical section
    shared_resource += 1

# Database transactions
from django.db import transaction
with transaction.atomic():
    user.save()
    profile.save()

# Temporary directory
import tempfile
with tempfile.TemporaryDirectory() as tmpdir:
    # Use tmpdir
    pass  # Automatically deleted
```

## 🧠 Why It Matters in Real Projects

- **Resource Safety**: Guaranteed cleanup even with exceptions
- **Transaction Management**: Database ACID properties
- **Lock Management**: Prevent deadlocks in multithreading
- **Clean Code**: Reduces boilerplate try/finally blocks
- **Testability**: Mock and test resource management
- **Production Reliability**: Prevent resource leaks

## ⚙️ Internal Working

### The Context Manager Protocol

```python
class ContextManagerExample:
    def __enter__(self):
        print("1. __enter__ called")
        print("2. Acquiring resource")
        return self  # or any object you want to work with
    
    def __exit__(self, exc_type, exc_value, traceback):
        print("4. __exit__ called")
        print(f"   Exception type: {exc_type}")
        print(f"   Exception value: {exc_value}")
        print(f"   Traceback: {traceback}")
        print("5. Releasing resource")
        
        # Return True to suppress exception
        # Return False (or None) to propagate exception
        return False

with ContextManagerExample() as cm:
    print("3. Inside with block")
    # raise ValueError("Test exception")

"""Output:
1. __enter__ called
2. Acquiring resource
3. Inside with block
4. __exit__ called
   Exception type: None
   Exception value: None
   Traceback: None
5. Releasing resource
"""
```

### Exception Handling

```python
class ExceptionHandler:
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type is ValueError:
            print("Handling ValueError")
            return True  # Suppress exception
        return False  # Propagate other exceptions

# Exception suppressed
with ExceptionHandler():
    raise ValueError("This is caught")
print("Continue execution")

# Exception propagated
try:
    with ExceptionHandler():
        raise TypeError("This is not caught")
except TypeError:
    print("TypeError propagated")
```

## ✅ Best Practices

### 1. Use contextlib.contextmanager for Simple Cases

```python
from contextlib import contextmanager

@contextmanager
def file_manager(filename, mode):
    """Simple context manager using generator"""
    f = open(filename, mode)
    try:
        yield f  # Value returned to 'as' clause
    finally:
        f.close()

with file_manager('test.txt', 'w') as f:
    f.write('Hello')
```

### 2. Database Transaction Context Manager

```python
import psycopg2
from contextlib import contextmanager

@contextmanager
def database_transaction(connection):
    """Manage database transaction"""
    cursor = connection.cursor()
    try:
        yield cursor
        connection.commit()
        print("Transaction committed")
    except Exception as e:
        connection.rollback()
        print(f"Transaction rolled back: {e}")
        raise
    finally:
        cursor.close()

# Usage
conn = psycopg2.connect(database="mydb")
with database_transaction(conn) as cursor:
    cursor.execute("INSERT INTO users (name) VALUES ('Alice')")
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE user_id = 1")
    # Automatically commits or rolls back
```

### 3. Timing Context Manager

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(name="Operation"):
    """Measure execution time"""
    start = time.time()
    yield
    end = time.time()
    print(f"{name} took {end - start:.4f} seconds")

with timer("Database Query"):
    # Perform time-consuming operation
    time.sleep(1)
    result = query_database()
```

### 4. Multiple Context Managers

```python
# Stack multiple context managers
with open('input.txt', 'r') as infile, \
     open('output.txt', 'w') as outfile:
    content = infile.read()
    outfile.write(content.upper())

# Or use contextlib.ExitStack for dynamic number
from contextlib import ExitStack

def process_files(filenames):
    with ExitStack() as stack:
        files = [stack.enter_context(open(fname)) for fname in filenames]
        
        # All files automatically closed
        for f in files:
            process(f)
```

### 5. Reentrant Context Managers

```python
import threading

class ReentrantLock:
    """Context manager that can be acquired multiple times by same thread"""
    
    def __init__(self):
        self.lock = threading.RLock()
    
    def __enter__(self):
        self.lock.acquire()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.lock.release()
        return False

lock = ReentrantLock()

def recursive_function(n):
    with lock:
        if n > 0:
            print(f"Level {n}")
            recursive_function(n - 1)

recursive_function(3)  # Works! Regular locks would deadlock
```

## ❌ Common Mistakes

### 1. Not Returning Value from __enter__

```python
# WRONG - __enter__ should return something
class BadContextManager:
    def __enter__(self):
        self.resource = acquire_resource()
        # Missing return!
    
    def __exit__(self, *args):
        release_resource(self.resource)

with BadContextManager() as resource:
    print(resource)  # None!

# CORRECT
class GoodContextManager:
    def __enter__(self):
        self.resource = acquire_resource()
        return self.resource  # Return the resource
    
    def __exit__(self, *args):
        release_resource(self.resource)
```

### 2. Forgetting Exception Parameters in __exit__

```python
# WRONG - Missing required parameters
class WrongExitSignature:
    def __enter__(self):
        return self
    
    def __exit__(self):  # Missing parameters!
        cleanup()

# CORRECT - Always accept 3 exception parameters
class CorrectExitSignature:
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_value, traceback):
        cleanup()
        return False
```

### 3. Not Handling Exceptions in Cleanup

```python
# WRONG - Exception in __exit__ masks original exception
class BadCleanup:
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()  # May raise if file doesn't exist
        return False

# CORRECT - Suppress cleanup exceptions
class GoodCleanup:
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        try:
            if hasattr(self, 'file'):
                self.file.close()
        except Exception as e:
            # Log but don't raise
            print(f"Cleanup error: {e}")
        return False
```

### 4. Yielding Multiple Times in Generator-Based Context Manager

```python
from contextlib import contextmanager

# WRONG - Multiple yields
@contextmanager
def broken_context_manager():
    setup()
    yield "first"
    yield "second"  # ERROR! Can only yield once
    cleanup()

# CORRECT - Single yield
@contextmanager
def correct_context_manager():
    setup()
    try:
        yield "resource"
    finally:
        cleanup()
```

## 🔐 Security Considerations

### 1. Secure File Handling

```python
import os
from contextlib import contextmanager

@contextmanager
def secure_file(filename, mode='r'):
    """Securely handle files with permission checks"""
    # Check file permissions
    if os.path.exists(filename):
        stats = os.stat(filename)
        if stats.st_uid != os.getuid():
            raise PermissionError("File not owned by current user")
    
    # Set secure umask for file creation
    old_umask = os.umask(0o077)
    
    try:
        f = open(filename, mode)
        try:
            yield f
        finally:
            f.close()
    finally:
        os.umask(old_umask)

with secure_file('sensitive_data.txt', 'w') as f:
    f.write('Secret information')
```

### 2. Database Connection Pool

```python
from contextlib import contextmanager
from queue import Queue, Empty
import psycopg2

class ConnectionPool:
    def __init__(self, maxsize=10, **conn_params):
        self.pool = Queue(maxsize=maxsize)
        self.conn_params = conn_params
        
        # Initialize pool
        for _ in range(maxsize):
            conn = psycopg2.connect(**conn_params)
            self.pool.put(conn)
    
    @contextmanager
    def connection(self, timeout=5):
        """Get connection from pool"""
        conn = None
        try:
            conn = self.pool.get(timeout=timeout)
            yield conn
        except Empty:
            raise Exception("No available connections")
        finally:
            if conn:
                self.pool.put(conn)

# Usage
pool = ConnectionPool(maxsize=5, database="mydb")

with pool.connection() as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users")
    # Connection automatically returned to pool
```

### 3. API Rate Limiting

```python
import time
from contextlib import contextmanager
from collections import defaultdict
from threading import Lock

class RateLimiter:
    def __init__(self, calls_per_second):
        self.calls_per_second = calls_per_second
        self.last_call = defaultdict(float)
        self.lock = Lock()
    
    @contextmanager
    def limit(self, key="default"):
        """Rate limit context manager"""
        with self.lock:
            now = time.time()
            time_since_last = now - self.last_call[key]
            
            if time_since_last < 1.0 / self.calls_per_second:
                time.sleep(1.0 / self.calls_per_second - time_since_last)
            
            self.last_call[key] = time.time()
        
        yield

# Usage
limiter = RateLimiter(calls_per_second=5)

for i in range(10):
    with limiter.limit(key="api_endpoint"):
        make_api_call()  # Limited to 5 calls per second
```

## 🚀 Performance Optimization

### 1. Cached Context Manager

```python
from contextlib import contextmanager
from functools import lru_cache

class ResourcePool:
    def __init__(self):
        self.resources = {}
    
    @contextmanager
    def get_resource(self, resource_id):
        """Get or create resource"""
        if resource_id not in self.resources:
            self.resources[resource_id] = self._create_resource(resource_id)
        
        resource = self.resources[resource_id]
        try:
            yield resource
        finally:
            # Keep resource in pool for reuse
            pass
    
    def _create_resource(self, resource_id):
        """Expensive resource creation"""
        print(f"Creating resource {resource_id}")
        return f"Resource-{resource_id}"

pool = ResourcePool()

# First call creates resource
with pool.get_resource("A") as resource:
    print(resource)

# Second call reuses resource (no creation)
with pool.get_resource("A") as resource:
    print(resource)
```

### 2. Lazy Loading Context Manager

```python
from contextlib import contextmanager

class LazyConnection:
    def __init__(self, conn_string):
        self.conn_string = conn_string
        self._connection = None
    
    @contextmanager
    def connect(self):
        """Lazy connection - only connects when needed"""
        if self._connection is None:
            print("Establishing connection...")
            self._connection = create_connection(self.conn_string)
        
        try:
            yield self._connection
        except Exception:
            # Close on error
            if self._connection:
                self._connection.close()
                self._connection = None
            raise

lazy_conn = LazyConnection("postgresql://localhost/mydb")

# Connection only established when first used
with lazy_conn.connect() as conn:
    query(conn)

# Reuses existing connection
with lazy_conn.connect() as conn:
    query(conn)
```

## 🧪 Code Examples

### 1. Change Directory Context Manager

```python
import os
from contextlib import contextmanager

@contextmanager
def change_directory(path):
    """Temporarily change working directory"""
    original = os.getcwd()
    try:
        os.chdir(path)
        yield
    finally:
        os.chdir(original)

print(f"Current directory: {os.getcwd()}")

with change_directory('/tmp'):
    print(f"Inside with block: {os.getcwd()}")
    # Perform operations in /tmp

print(f"After with block: {os.getcwd()}")  # Back to original
```

### 2. Environment Variable Context Manager

```python
import os
from contextlib import contextmanager

@contextmanager
def temporary_env_var(key, value):
    """Temporarily set environment variable"""
    old_value = os.environ.get(key)
    os.environ[key] = value
    
    try:
        yield
    finally:
        if old_value is None:
            del os.environ[key]
        else:
            os.environ[key] = old_value

with temporary_env_var('DEBUG', 'True'):
    print(os.environ['DEBUG'])  # 'True'
    run_tests()

print(os.environ.get('DEBUG'))  # Original value or None
```

### 3. Suppress Exceptions Context Manager

```python
from contextlib import suppress

# Instead of try/except pass
with suppress(FileNotFoundError):
    os.remove('non_existent_file.txt')

# Instead of try/except pass for multiple exceptions
with suppress(KeyError, AttributeError):
    value = data['missing_key'].missing_attribute

# Equivalent to:
try:
    os.remove('non_existent_file.txt')
except FileNotFoundError:
    pass
```

### 4. Redirect Standard Output

```python
from contextlib import redirect_stdout, redirect_stderr
import io

# Capture stdout
output = io.StringIO()
with redirect_stdout(output):
    print("This goes to output buffer")
    print("Not to console")

captured = output.getvalue()
print(f"Captured: {captured}")

# Redirect to file
with open('output.log', 'w') as f:
    with redirect_stdout(f):
        print("This goes to file")
        run_verbose_function()
```

### 5. Mock Context Manager for Testing

```python
from contextlib import contextmanager
from unittest.mock import patch

@contextmanager
def mock_external_service():
    """Mock external service for testing"""
    with patch('requests.get') as mock_get:
        mock_get.return_value.status_code = 200
        mock_get.return_value.json.return_value = {'data': 'test'}
        yield mock_get

def test_api_call():
    with mock_external_service() as mock:
        response = fetch_data_from_api()
        assert response == {'data': 'test'}
        mock.assert_called_once()
```

## 🏗️ Real-World Use Cases

### Django Database Transaction

```python
from django.db import transaction

@contextmanager
def atomic_with_retry(max_retries=3):
    """Transaction with automatic retry on deadlock"""
    from django.db.utils import OperationalError
    
    for attempt in range(max_retries):
        try:
            with transaction.atomic():
                yield
            break
        except OperationalError as e:
            if 'deadlock' in str(e).lower() and attempt < max_retries - 1:
                print(f"Deadlock detected, retrying ({attempt + 1}/{max_retries})")
                time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
            else:
                raise

# Usage
with atomic_with_retry():
    user.balance -= 100
    user.save()
    transaction_log.create(amount=-100, user=user)
```

### FastAPI Database Session

```python
from contextlib import contextmanager
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine('postgresql://localhost/mydb')
SessionLocal = sessionmaker(bind=engine)

@contextmanager
def get_db_session():
    """FastAPI database session context manager"""
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# In FastAPI endpoint
@app.get("/users/{user_id}")
def get_user(user_id: int):
    with get_db_session() as db:
        user = db.query(User).filter(User.id == user_id).first()
        return user
```

### Redis Lock for Distributed Systems

```python
import redis
from contextlib import contextmanager
import uuid
import time

@contextmanager
def redis_lock(redis_client, lock_name, timeout=10):
    """Distributed lock using Redis"""
    identifier = str(uuid.uuid4())
    lock_key = f"lock:{lock_name}"
    
    # Acquire lock
    end_time = time.time() + timeout
    while time.time() < end_time:
        if redis_client.set(lock_key, identifier, nx=True, ex=timeout):
            break
        time.sleep(0.001)
    else:
        raise Exception(f"Could not acquire lock: {lock_name}")
    
    try:
        yield identifier
    finally:
        # Release lock (only if we still own it)
        pipe = redis_client.pipeline(True)
        while True:
            try:
                pipe.watch(lock_key)
                if pipe.get(lock_key).decode() == identifier:
                    pipe.multi()
                    pipe.delete(lock_key)
                    pipe.execute()
                break
            except redis.WatchError:
                pass
            finally:
                pipe.unwatch()

# Usage - ensure only one process runs critical section
r = redis.Redis()
with redis_lock(r, 'process_payments'):
    process_payment_batch()
```

### Temporary File Context Manager

```python
import tempfile
import os
from contextlib import contextmanager

@contextmanager
def temporary_file(suffix='', prefix='tmp', dir=None):
    """Create and cleanup temporary file"""
    fd, path = tempfile.mkstemp(suffix=suffix, prefix=prefix, dir=dir)
    try:
        yield path
    finally:
        try:
            os.close(fd)
            os.unlink(path)
        except OSError:
            pass

# Usage
with temporary_file(suffix='.txt') as temp_path:
    with open(temp_path, 'w') as f:
        f.write('Temporary data')
    
    process_file(temp_path)
# File automatically deleted
```

### AWS S3 Download Context Manager

```python
import boto3
from contextlib import contextmanager
import tempfile
import os

@contextmanager
def s3_file_context(bucket, key):
    """Download S3 file to temp location and clean up"""
    s3 = boto3.client('s3')
    
    with tempfile.NamedTemporaryFile(delete=False) as tmp:
        tmp_path = tmp.name
    
    try:
        s3.download_file(bucket, key, tmp_path)
        yield tmp_path
    finally:
        try:
            os.unlink(tmp_path)
        except OSError:
            pass

# Usage
with s3_file_context('my-bucket', 'data/file.csv') as local_path:
    process_csv(local_path)
# File automatically cleaned up
```

## ❓ Interview Questions

### Q1: What is a context manager and why use it?

**Answer**: A context manager is an object that defines `__enter__()` and `__exit__()` methods, used with the `with` statement to ensure proper resource management. It guarantees cleanup code runs even if exceptions occur, preventing resource leaks.

### Q2: How do you create a context manager?

**Answer**: Two ways:
1. **Class-based**: Implement `__enter__()` and `__exit__()`
2. **Generator-based**: Use `@contextmanager` decorator with yield

```python
# Class-based
class MyContext:
    def __enter__(self):
        return self
    def __exit__(self, *args):
        cleanup()

# Generator-based
from contextlib import contextmanager

@contextmanager
def my_context():
    setup()
    try:
        yield value
    finally:
        cleanup()
```

### Q3: What are the parameters of __exit__?

**Answer**: Three parameters capturing exception information:
- `exc_type`: Exception class (or None)
- `exc_value`: Exception instance (or None)
- `traceback`: Traceback object (or None)

Return `True` to suppress exception, `False` (or None) to propagate.

### Q4: Can you nest context managers?

**Answer**: Yes, two ways:

```python
# Stacked
with open('in.txt') as f1, open('out.txt', 'w') as f2:
    f2.write(f1.read())

# Nested
with open('in.txt') as f1:
    with open('out.txt', 'w') as f2:
        f2.write(f1.read())
```

### Q5: What's contextlib.ExitStack used for?

**Answer**: Manages dynamic number of context managers:

```python
from contextlib import ExitStack

def process_files(filenames):
    with ExitStack() as stack:
        files = [stack.enter_context(open(f)) for f in filenames]
        # All files auto-closed
        for f in files:
            process(f)
```

### Q6: How do context managers help with database transactions?

**Answer**: They ensure transactions commit on success and rollback on failure:

```python
with transaction.atomic():
    user.balance -= 100
    user.save()
    # Auto-commits if no exception
    # Auto-rolls back if exception occurs
```

### Q7: Can context managers be reusable?

**Answer**: Class-based context managers can be reused, but generator-based cannot (they're single-use):

```python
# Class-based - reusable
cm = MyContextManager()
with cm:
    pass
with cm:  # Works
    pass

# Generator-based - single use
@contextmanager
def my_cm():
    yield

cm = my_cm()
with cm:
    pass
with cm:  # ERROR! Generator exhausted
    pass
```

### Q8: What's the difference between contextlib.suppress and try/except?

**Answer**: `suppress()` is cleaner for ignoring specific exceptions:

```python
# With suppress
with suppress(FileNotFoundError):
    os.remove('file.txt')

# Equivalent try/except
try:
    os.remove('file.txt')
except FileNotFoundError:
    pass
```

## 🧩 Design Patterns

### Resource Acquisition Is Initialization (RAII)

```python
class DatabaseConnection:
    """RAII pattern - resource lifetime tied to object lifetime"""
    
    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.connection = None
    
    def __enter__(self):
        self.connection = create_connection(self.connection_string)
        return self.connection
    
    def __exit__(self, *args):
        if self.connection:
            self.connection.close()

# Resource automatically managed
with DatabaseConnection("postgresql://localhost/db") as conn:
    query(conn)
# Connection automatically closed
```

---

## 📚 Summary

Context managers provide elegant resource management:
- Guaranteed cleanup even with exceptions
- Clean, readable code
- Prevents resource leaks
- Essential for files, locks, transactions, connections

**Key Takeaways**:
1. Use `with` statement for automatic resource management
2. Implement `__enter__` and `__exit__` for class-based managers
3. Use `@contextmanager` decorator for simple cases
4. `__exit__` can suppress exceptions by returning True
5. Always handle cleanup exceptions to avoid masking original errors
6. Use `ExitStack` for dynamic number of context managers
