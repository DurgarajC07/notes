# FastAPI Core: Validation & Error Handling

## 📖 Introduction

FastAPI provides automatic request validation using Pydantic and type hints. When validation fails, FastAPI returns structured error responses with details about what went wrong.

## ✅ Automatic Validation

### Type-Based Validation

```python
from fastapi import FastAPI, Query, Path, Body
from typing import Optional

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    """
    Automatic type validation

    - item_id must be integer
    - Returns 422 if validation fails
    """
    return {"item_id": item_id}

"""
Valid: /items/123
Invalid: /items/abc
Response: 422 Unprocessable Entity
{
  "detail": [
    {
      "loc": ["path", "item_id"],
      "msg": "value is not a valid integer",
      "type": "type_error.integer"
    }
  ]
}
"""
```

### Query Parameter Validation

```python
@app.get("/search")
async def search(
    q: str = Query(..., min_length=3, max_length=50),
    limit: int = Query(10, ge=1, le=100),
    offset: int = Query(0, ge=0)
):
    """
    Query parameter validation

    - q: required, 3-50 characters
    - limit: 1-100 (default 10)
    - offset: >= 0 (default 0)
    """
    return {
        "query": q,
        "limit": limit,
        "offset": offset
    }
```

### Pydantic Model Validation

```python
from pydantic import BaseModel, Field, validator

class CreateUserRequest(BaseModel):
    """User creation with validation"""
    username: str = Field(..., min_length=3, max_length=50, regex="^[a-zA-Z0-9_]+$")
    email: str = Field(..., regex=r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")
    password: str = Field(..., min_length=8)
    age: int = Field(..., ge=13, le=120)

    @validator('password')
    def password_strength(cls, v):
        """Validate password strength"""
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        return v

@app.post("/users")
async def create_user(user: CreateUserRequest):
    """
    Request body validation

    All Pydantic validations applied automatically
    """
    return user

"""
Invalid request:
{
  "username": "ab",
  "email": "invalid",
  "password": "weak",
  "age": 5
}

Response: 422 Unprocessable Entity
{
  "detail": [
    {
      "loc": ["body", "username"],
      "msg": "ensure this value has at least 3 characters",
      "type": "value_error.any_str.min_length"
    },
    {
      "loc": ["body", "email"],
      "msg": "string does not match regex",
      "type": "value_error.str.regex"
    },
    {
      "loc": ["body", "password"],
      "msg": "Password must contain uppercase letter",
      "type": "value_error"
    },
    {
      "loc": ["body", "age"],
      "msg": "ensure this value is greater than or equal to 13",
      "type": "value_error.number.not_ge"
    }
  ]
}
"""
```

## 🔧 Custom Validators

### Field-Level Validators

```python
from pydantic import validator

class Product(BaseModel):
    """Product with custom validation"""
    name: str
    price: float
    discount: float = 0

    @validator('price')
    def price_must_be_positive(cls, v):
        """Validate price is positive"""
        if v <= 0:
            raise ValueError('Price must be positive')
        return v

    @validator('discount')
    def discount_valid_range(cls, v):
        """Validate discount range"""
        if not 0 <= v <= 100:
            raise ValueError('Discount must be between 0 and 100')
        return v

@app.post("/products")
async def create_product(product: Product):
    """Create product with custom validation"""
    return product
```

### Cross-Field Validation

```python
from pydantic import root_validator

class BookingRequest(BaseModel):
    """Booking with date validation"""
    check_in: datetime
    check_out: datetime
    guests: int = Field(..., ge=1, le=10)

    @root_validator
    def check_dates(cls, values):
        """Validate check_out is after check_in"""
        check_in = values.get('check_in')
        check_out = values.get('check_out')

        if check_in and check_out:
            if check_out <= check_in:
                raise ValueError('check_out must be after check_in')

            # Check minimum stay
            duration = (check_out - check_in).days
            if duration < 1:
                raise ValueError('Minimum stay is 1 night')

        return values

@app.post("/bookings")
async def create_booking(booking: BookingRequest):
    """Create booking with date validation"""
    return booking
```

## ⚠️ Error Handling

### HTTPException

```python
from fastapi import HTTPException, status

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """
    HTTPException for application errors

    - Raises HTTP error with status code
    - Returns JSON with detail message
    """
    # Check if user exists (simulated)
    if user_id not in [1, 2, 3]:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User {user_id} not found"
        )

    return {"user_id": user_id, "name": "John"}

@app.delete("/items/{item_id}")
async def delete_item(item_id: int):
    """HTTPException with custom headers"""
    if item_id < 0:
        raise HTTPException(
            status_code=400,
            detail="Invalid item ID",
            headers={"X-Error": "Invalid-ID"}
        )

    return {"message": "Item deleted"}
```

### Custom Exception Classes

```python
class ItemNotFoundError(HTTPException):
    """Custom exception for item not found"""
    def __init__(self, item_id: int):
        super().__init__(
            status_code=404,
            detail=f"Item {item_id} not found"
        )

class InsufficientStockError(HTTPException):
    """Custom exception for insufficient stock"""
    def __init__(self, item_id: int, available: int, requested: int):
        super().__init__(
            status_code=400,
            detail=f"Item {item_id}: only {available} available, requested {requested}"
        )

@app.post("/orders")
async def create_order(item_id: int, quantity: int):
    """Use custom exceptions"""
    # Check if item exists
    if item_id not in [1, 2, 3]:
        raise ItemNotFoundError(item_id)

    # Check stock (simulated)
    available = 5
    if quantity > available:
        raise InsufficientStockError(item_id, available, quantity)

    return {"message": "Order created", "item_id": item_id, "quantity": quantity}
```

### Exception Handlers

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """
    Custom validation error handler

    - Customizes validation error response
    - Can log errors, send to monitoring, etc.
    """
    return JSONResponse(
        status_code=422,
        content={
            "message": "Validation failed",
            "errors": exc.errors(),
            "body": exc.body
        }
    )

# Custom exception handler
class BusinessLogicError(Exception):
    """Custom business logic exception"""
    def __init__(self, message: str, error_code: str):
        self.message = message
        self.error_code = error_code

@app.exception_handler(BusinessLogicError)
async def business_logic_exception_handler(request: Request, exc: BusinessLogicError):
    """Handle business logic errors"""
    return JSONResponse(
        status_code=400,
        content={
            "error_code": exc.error_code,
            "message": exc.message
        }
    )

@app.post("/transfer")
async def transfer_funds(from_account: int, to_account: int, amount: float):
    """Endpoint using custom exception"""
    if amount <= 0:
        raise BusinessLogicError(
            message="Transfer amount must be positive",
            error_code="INVALID_AMOUNT"
        )

    # Check balance (simulated)
    balance = 100
    if amount > balance:
        raise BusinessLogicError(
            message=f"Insufficient funds: balance {balance}, requested {amount}",
            error_code="INSUFFICIENT_FUNDS"
        )

    return {"message": "Transfer successful", "amount": amount}
```

## 🔒 Request Validation Errors

### Detailed Error Information

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import PlainTextResponse

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """
    Detailed validation errors

    Error structure:
    - loc: location of error (path, query, body)
    - msg: error message
    - type: error type
    - ctx: additional context (optional)
    """
    errors = []

    for error in exc.errors():
        loc = " -> ".join(str(l) for l in error['loc'])
        msg = error['msg']
        error_type = error['type']

        errors.append(f"{loc}: {msg} ({error_type})")

    return PlainTextResponse(
        content="\n".join(errors),
        status_code=422
    )
```

### Custom Validation Error Response

```python
from typing import List, Dict, Any

class ValidationError(BaseModel):
    """Structured validation error"""
    field: str
    message: str
    value: Any = None

class ValidationErrorResponse(BaseModel):
    """Validation error response"""
    message: str
    errors: List[ValidationError]

@app.exception_handler(RequestValidationError)
async def custom_validation_handler(request: Request, exc: RequestValidationError):
    """Custom validation error format"""
    errors = []

    for error in exc.errors():
        field = ".".join(str(l) for l in error['loc'][1:])  # Skip 'body'

        errors.append(ValidationError(
            field=field,
            message=error['msg'],
            value=error.get('ctx', {}).get('given')
        ))

    response = ValidationErrorResponse(
        message="Request validation failed",
        errors=errors
    )

    return JSONResponse(
        status_code=422,
        content=response.dict()
    )
```

## 🛡️ Security Validation

### SQL Injection Prevention

```python
from fastapi import Depends
from sqlalchemy.orm import Session

@app.get("/users/search")
async def search_users(
    name: str = Query(..., regex="^[a-zA-Z0-9 ]+$"),  # Only alphanumeric
    db: Session = Depends(get_db)
):
    """
    Prevent SQL injection

    - Validate input with regex
    - Use parameterized queries
    - Never concatenate user input into SQL
    """
    # SAFE: Parameterized query
    users = db.query(User).filter(User.name.contains(name)).all()

    return {"users": [u.name for u in users]}
```

### XSS Prevention

```python
import html

class CommentRequest(BaseModel):
    """Comment with XSS prevention"""
    content: str = Field(..., max_length=1000)

    @validator('content')
    def sanitize_content(cls, v):
        """Escape HTML to prevent XSS"""
        return html.escape(v)

@app.post("/comments")
async def create_comment(comment: CommentRequest):
    """Create comment with XSS prevention"""
    return {"content": comment.content}
```

### File Upload Validation

```python
from fastapi import UploadFile, File

ALLOWED_EXTENSIONS = {'.jpg', '.jpeg', '.png', '.gif'}
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5 MB

@app.post("/upload")
async def upload_image(file: UploadFile = File(...)):
    """
    Validate file upload

    - Check file extension
    - Check file size
    - Verify file content (magic bytes)
    """
    # Check extension
    import os
    ext = os.path.splitext(file.filename)[1].lower()

    if ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(
            status_code=400,
            detail=f"File type {ext} not allowed. Allowed: {ALLOWED_EXTENSIONS}"
        )

    # Check size
    contents = await file.read()

    if len(contents) > MAX_FILE_SIZE:
        raise HTTPException(
            status_code=400,
            detail=f"File too large: {len(contents)} bytes (max: {MAX_FILE_SIZE})"
        )

    # Verify it's actually an image (check magic bytes)
    if not contents.startswith(b'\xff\xd8') and not contents.startswith(b'\x89PNG'):
        raise HTTPException(
            status_code=400,
            detail="Invalid image file"
        )

    # Save file
    return {"filename": file.filename, "size": len(contents)}
```

## 📊 Validation Performance

### Caching Validators

```python
from functools import lru_cache

class Product(BaseModel):
    """Product with cached validation"""
    sku: str

    @validator('sku')
    def validate_sku(cls, v):
        """Validate SKU (cached)"""
        return cls._validate_sku_cached(v)

    @staticmethod
    @lru_cache(maxsize=1000)
    def _validate_sku_cached(sku: str) -> str:
        """Cached SKU validation"""
        # Expensive validation logic
        if not sku.startswith('SKU-'):
            raise ValueError('SKU must start with "SKU-"')
        return sku
```

## ❓ Interview Questions

### Q1: What happens when validation fails in FastAPI?

**Answer**:

- Returns **422 Unprocessable Entity**
- Response includes detailed error information
- Lists all validation errors (field, message, type)
- Can customize with exception handlers

### Q2: How to validate cross-field dependencies?

**Answer**:

- Use **@root_validator** in Pydantic
- Has access to all field values
- Can raise ValueError for validation errors
- Example: end_date must be after start_date

### Q3: How does FastAPI prevent common security vulnerabilities?

**Answer**:

- **SQL Injection**: Pydantic validation + parameterized queries
- **XSS**: HTML escaping with validators
- **Path Traversal**: Path validation
- **File Upload**: Extension/size/content validation

### Q4: When to use HTTPException vs custom exception classes?

**Answer**:

- **HTTPException**: Simple, one-off errors
- **Custom classes**: Reusable, domain-specific errors
- **Exception handlers**: Global error handling
- Custom classes provide better code organization

## 📚 Summary

**Key Takeaways**:

1. **Automatic validation** from type hints and Pydantic
2. **Field() constraints**: min/max length, ge/le, regex
3. **@validator** for custom field validation
4. **@root_validator** for cross-field validation
5. **HTTPException** for application errors
6. **Custom exception classes** for domain errors
7. **Exception handlers** customize error responses
8. **422 status** for validation errors
9. **Security validation**: SQL injection, XSS, file uploads
10. **Detailed error messages** help debugging

FastAPI makes validation declarative and comprehensive!
