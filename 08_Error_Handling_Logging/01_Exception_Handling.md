# ⚠️ Exception Handling and Error Management

## Overview

Proper exception handling is crucial for building robust applications. Understanding Python's exception hierarchy and best practices ensures reliable error management.

---

## Exception Hierarchy

### Built-in Exception Tree

```python
"""
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── StopIteration
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   ├── OverflowError
    │   └── FloatingPointError
    ├── AssertionError
    ├── AttributeError
    ├── EOFError
    ├── ImportError
    │   └── ModuleNotFoundError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── MemoryError
    ├── NameError
    │   └── UnboundLocalError
    ├── OSError
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   └── TimeoutError
    ├── RuntimeError
    │   ├── NotImplementedError
    │   └── RecursionError
    ├── TypeError
    ├── ValueError
    │   └── UnicodeError
    └── Warning
"""

# Never catch BaseException (includes SystemExit, KeyboardInterrupt)
try:
    # Code
    pass
except BaseException:  # ❌ BAD: Catches Ctrl+C
    pass

# Catch specific exceptions
try:
    result = 10 / 0
except ZeroDivisionError as e:  # ✅ GOOD: Specific
    print(f"Cannot divide by zero: {e}")

# Multiple exceptions
try:
    data = {"key": "value"}
    result = data["missing_key"] / 0
except (KeyError, ZeroDivisionError) as e:
    print(f"Error: {e}")

# Exception hierarchy - catch specific first
try:
    # Code
    pass
except FileNotFoundError:  # Specific
    print("File not found")
except OSError:  # More general
    print("OS error")
except Exception:  # Most general
    print("Unknown error")
```

---

## Custom Exceptions

### Creating Custom Exceptions

```python
# Base application exception
class ApplicationError(Exception):
    """Base exception for application"""
    pass

class ValidationError(ApplicationError):
    """Validation failed"""
    def __init__(self, field, message):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

class DatabaseError(ApplicationError):
    """Database operation failed"""
    pass

class ResourceNotFoundError(ApplicationError):
    """Resource not found"""
    def __init__(self, resource_type, resource_id):
        self.resource_type = resource_type
        self.resource_id = resource_id
        message = f"{resource_type} with id {resource_id} not found"
        super().__init__(message)

# Usage
def get_user(user_id):
    """Get user by ID"""
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise ResourceNotFoundError("User", user_id)
    return user

def create_user(data):
    """Create user with validation"""
    if not data.get("email"):
        raise ValidationError("email", "Email is required")

    if "@" not in data["email"]:
        raise ValidationError("email", "Invalid email format")

    try:
        user = User(**data)
        db.add(user)
        db.commit()
        return user
    except IntegrityError:
        raise DatabaseError("Failed to create user")

# Catch custom exceptions
try:
    user = create_user({"name": "Alice"})
except ValidationError as e:
    print(f"Validation failed: {e.field} - {e.message}")
except DatabaseError as e:
    print(f"Database error: {e}")
except ApplicationError as e:
    print(f"Application error: {e}")
```

---

## Exception Handling Patterns

### Try-Except-Else-Finally

```python
# Complete exception handling
def read_file(filename):
    """Read file with proper error handling"""
    file = None
    try:
        # Try to open and read
        file = open(filename, 'r')
        content = file.read()

    except FileNotFoundError:
        # Handle specific error
        print(f"File {filename} not found")
        return None

    except PermissionError:
        # Handle permission error
        print(f"No permission to read {filename}")
        return None

    except Exception as e:
        # Catch unexpected errors
        print(f"Unexpected error: {e}")
        raise  # Re-raise

    else:
        # Executes if no exception
        print(f"Successfully read {filename}")
        return content

    finally:
        # Always executes (cleanup)
        if file:
            file.close()
            print("File closed")

# Context manager (better for resources)
def read_file_better(filename):
    """Using context manager"""
    try:
        with open(filename, 'r') as file:
            content = file.read()
        return content
    except FileNotFoundError:
        print(f"File {filename} not found")
        return None
    # File automatically closed
```

### Exception Chaining

```python
# Exception chaining - preserve original exception
def process_data(data):
    """Process data with error context"""
    try:
        result = int(data)
        return result * 2
    except ValueError as e:
        # Chain exceptions with 'from'
        raise ValidationError("data", "Must be integer") from e

# Usage
try:
    result = process_data("invalid")
except ValidationError as e:
    print(f"Error: {e}")
    print(f"Original cause: {e.__cause__}")

# Suppress chaining with 'from None'
def process_data_no_chain(data):
    """Without showing original exception"""
    try:
        result = int(data)
        return result * 2
    except ValueError:
        # Suppress original exception
        raise ValidationError("data", "Must be integer") from None
```

---

## Exception Context Managers

### Custom Context Managers for Errors

```python
from contextlib import contextmanager

@contextmanager
def handle_errors(error_message="An error occurred"):
    """Context manager for error handling"""
    try:
        yield
    except Exception as e:
        print(f"{error_message}: {e}")
        raise

# Usage
with handle_errors("Failed to process data"):
    # Code that might fail
    result = 10 / 0

# Suppress specific exceptions
from contextlib import suppress

# Without suppress
try:
    os.remove("nonexistent.txt")
except FileNotFoundError:
    pass

# With suppress (cleaner)
with suppress(FileNotFoundError):
    os.remove("nonexistent.txt")

# Custom exception handler with retry
@contextmanager
def retry_on_error(max_attempts=3, delay=1):
    """Retry on exception"""
    import time

    for attempt in range(max_attempts):
        try:
            yield attempt
            break  # Success
        except Exception as e:
            if attempt == max_attempts - 1:
                raise  # Last attempt, re-raise
            print(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(delay)

# Usage
with retry_on_error(max_attempts=3) as attempt:
    response = requests.get("https://api.example.com")
    response.raise_for_status()
```

---

## Error Handling in Async Code

### Async Exception Handling

```python
import asyncio

# Basic async exception handling
async def fetch_data(url):
    """Fetch data with error handling"""
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(url, timeout=10)
            response.raise_for_status()
            return response.json()
    except httpx.TimeoutException:
        print(f"Timeout fetching {url}")
        return None
    except httpx.HTTPStatusError as e:
        print(f"HTTP error {e.response.status_code}")
        return None
    except Exception as e:
        print(f"Unexpected error: {e}")
        raise

# Handle exceptions in concurrent tasks
async def fetch_multiple_urls(urls):
    """Fetch multiple URLs concurrently"""
    async def fetch_with_error_handling(url):
        try:
            return await fetch_data(url)
        except Exception as e:
            return {"url": url, "error": str(e)}

    # Gather with return_exceptions=True
    results = await asyncio.gather(
        *[fetch_with_error_handling(url) for url in urls],
        return_exceptions=True
    )

    # Separate successful and failed
    successful = [r for r in results if not isinstance(r, Exception)]
    failed = [r for r in results if isinstance(r, Exception)]

    return successful, failed

# Timeout handling
async def operation_with_timeout():
    """Operation with timeout"""
    try:
        result = await asyncio.wait_for(
            slow_operation(),
            timeout=5.0
        )
        return result
    except asyncio.TimeoutError:
        print("Operation timed out")
        return None
```

---

## FastAPI Error Handling

### Exception Handlers

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# Custom exception handler
@app.exception_handler(ValidationError)
async def validation_exception_handler(
    request: Request,
    exc: ValidationError
):
    """Handle validation errors"""
    return JSONResponse(
        status_code=400,
        content={
            "error": "Validation Error",
            "field": exc.field,
            "message": exc.message
        }
    )

@app.exception_handler(ResourceNotFoundError)
async def not_found_exception_handler(
    request: Request,
    exc: ResourceNotFoundError
):
    """Handle not found errors"""
    return JSONResponse(
        status_code=404,
        content={
            "error": "Not Found",
            "resource_type": exc.resource_type,
            "resource_id": exc.resource_id
        }
    )

# Global exception handler
@app.exception_handler(Exception)
async def global_exception_handler(
    request: Request,
    exc: Exception
):
    """Handle unexpected errors"""
    # Log error
    logger.error(f"Unexpected error: {exc}", exc_info=True)

    return JSONResponse(
        status_code=500,
        content={"error": "Internal Server Error"}
    )

# Use HTTPException for common errors
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """Get user by ID"""
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        raise HTTPException(
            status_code=404,
            detail=f"User {user_id} not found"
        )

    return user

# Dependency with error handling
async def verify_token(token: str = Header(...)):
    """Verify authentication token"""
    try:
        payload = jwt.decode(token, SECRET_KEY)
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=401,
            detail="Token expired"
        )
    except jwt.InvalidTokenError:
        raise HTTPException(
            status_code=401,
            detail="Invalid token"
        )
```

---

## Logging Errors

### Structured Error Logging

```python
import logging
import traceback
import sys

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)

# Log exceptions
def process_data(data):
    """Process data with logging"""
    try:
        result = complex_operation(data)
        logger.info("Successfully processed data")
        return result

    except ValueError as e:
        logger.error(f"Invalid data: {e}")
        raise

    except Exception as e:
        # Log full traceback
        logger.exception("Unexpected error processing data")
        raise

# Log with context
def process_user_data(user_id, data):
    """Process with contextual logging"""
    logger = logging.getLogger(__name__)

    try:
        logger.info(f"Processing data for user {user_id}")
        result = process_data(data)
        logger.info(f"Completed for user {user_id}")
        return result

    except Exception as e:
        logger.error(
            f"Failed for user {user_id}",
            extra={
                "user_id": user_id,
                "error_type": type(e).__name__,
                "error_message": str(e)
            },
            exc_info=True
        )
        raise

# Structured logging with structlog
import structlog

logger = structlog.get_logger()

def process_order(order_id):
    """Process order with structured logging"""
    log = logger.bind(order_id=order_id)

    try:
        log.info("Processing order")
        # Process order
        log.info("Order processed successfully")

    except Exception as e:
        log.error(
            "Order processing failed",
            error=str(e),
            error_type=type(e).__name__
        )
        raise
```

---

## Best Practices

### ✅ Do's:

1. **Catch specific exceptions** first
2. **Use custom exceptions** for application errors
3. **Chain exceptions** with `from` for context
4. **Log exceptions** with full traceback
5. **Use context managers** for cleanup
6. **Validate early** to fail fast
7. **Document exceptions** in docstrings
8. **Re-raise** when can't handle

### ❌ Don'ts:

1. **Don't catch Exception** without good reason
2. **Don't use bare except:** (catches everything)
3. **Don't silence exceptions** without logging
4. **Don't use exceptions** for control flow
5. **Don't catch SystemExit/KeyboardInterrupt**
6. **Don't create too many** custom exceptions
7. **Don't return error codes** (use exceptions)

---

## Interview Questions

### Q1: What's the difference between Exception and BaseException?

**Answer**:

- **BaseException**: Root of all exceptions (includes SystemExit, KeyboardInterrupt)
- **Exception**: User code should catch/raise this
- Never catch BaseException (prevents Ctrl+C exit)

### Q2: When should you create custom exceptions?

**Answer**:

- **Application-specific errors**: Domain logic violations
- **Better error handling**: Catch specific to application
- **Context**: Add custom attributes (field, resource_id)
- **Hierarchy**: Organize related exceptions

### Q3: Explain try-except-else-finally.

**Answer**:

- **try**: Code that might raise exception
- **except**: Handle specific exceptions
- **else**: Runs if no exception (optional)
- **finally**: Always runs (cleanup, optional)

### Q4: What is exception chaining?

**Answer**: Preserve original exception with new one:

- Use `raise NewError from original_error`
- `__cause__` attribute stores original
- Provides full error context
- Use `from None` to suppress

### Q5: How do you handle errors in async code?

**Answer**:

- Same try-except syntax
- Use `asyncio.gather(return_exceptions=True)`
- Handle `asyncio.TimeoutError` for timeouts
- Log asynchronously if needed

---

## Summary

Exception handling essentials:

- **Hierarchy**: Catch specific before general
- **Custom exceptions**: Domain-specific errors
- **Patterns**: try-except-else-finally
- **Context managers**: Automatic cleanup
- **Logging**: Track errors with context
- **FastAPI**: Global exception handlers
- **Best practices**: Specific, logged, chained

Handle errors gracefully! ⚠️
