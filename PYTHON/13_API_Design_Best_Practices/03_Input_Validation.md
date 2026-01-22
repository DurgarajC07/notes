# ✅ Input Validation and Sanitization

## Overview

Input validation and sanitization are critical for API security and data integrity. They prevent injection attacks, ensure data quality, and provide clear feedback to clients about invalid requests.

---

## Pydantic Validation

### 1. **Basic Field Validation**

```python
from pydantic import BaseModel, Field, validator, root_validator
from typing import Optional
from datetime import date

class UserCreate(BaseModel):
    """User creation with comprehensive validation"""
    username: str = Field(
        ...,
        min_length=3,
        max_length=50,
        regex=r"^[a-zA-Z0-9_-]+$",
        description="Username (alphanumeric, underscore, hyphen)"
    )
    email: str = Field(
        ...,
        regex=r"^[\w\.-]+@[\w\.-]+\.\w+$",
        description="Valid email address"
    )
    password: str = Field(
        ...,
        min_length=8,
        max_length=100,
        description="Password (minimum 8 characters)"
    )
    age: int = Field(
        ...,
        ge=18,
        le=120,
        description="Age (18-120)"
    )
    website: Optional[str] = Field(
        None,
        regex=r"^https?://",
        description="Website URL (must start with http:// or https://)"
    )

    @validator('username')
    def username_no_spaces(cls, v):
        """Ensure username has no spaces"""
        if ' ' in v:
            raise ValueError('Username cannot contain spaces')
        return v.lower()  # Normalize to lowercase

    @validator('email')
    def email_lowercase(cls, v):
        """Normalize email to lowercase"""
        return v.lower()

    @validator('password')
    def password_strength(cls, v):
        """Validate password strength"""
        if not any(char.isdigit() for char in v):
            raise ValueError('Password must contain at least one digit')
        if not any(char.isupper() for char in v):
            raise ValueError('Password must contain at least one uppercase letter')
        if not any(char.islower() for char in v):
            raise ValueError('Password must contain at least one lowercase letter')
        return v
```

### 2. **Custom Validators**

```python
from pydantic import validator
import re

class ProductCreate(BaseModel):
    """Product with custom validators"""
    name: str
    price: float
    discount_price: Optional[float] = None
    sku: str
    tags: list[str] = []

    @validator('price')
    def price_must_be_positive(cls, v):
        """Ensure price is positive"""
        if v <= 0:
            raise ValueError('Price must be greater than zero')
        return round(v, 2)  # Round to 2 decimal places

    @validator('sku')
    def sku_format(cls, v):
        """Validate SKU format"""
        if not re.match(r'^[A-Z]{3}-\d{4}$', v):
            raise ValueError('SKU must be in format XXX-1234')
        return v

    @validator('tags')
    def validate_tags(cls, v):
        """Validate and clean tags"""
        if len(v) > 10:
            raise ValueError('Maximum 10 tags allowed')
        # Remove duplicates and empty strings
        return list(set(tag.strip().lower() for tag in v if tag.strip()))

    @root_validator
    def validate_discount(cls, values):
        """Validate discount price against regular price"""
        price = values.get('price')
        discount_price = values.get('discount_price')

        if discount_price is not None:
            if discount_price >= price:
                raise ValueError('Discount price must be less than regular price')
            if discount_price < 0:
                raise ValueError('Discount price cannot be negative')

        return values
```

---

## FastAPI Query Parameter Validation

### 1. **Query Parameters with Constraints**

```python
from fastapi import Query, Path
from typing import Annotated

@app.get("/products")
async def search_products(
    q: Annotated[str | None, Query(
        min_length=3,
        max_length=50,
        regex=r"^[a-zA-Z0-9\s]+$",
        description="Search query (alphanumeric and spaces only)"
    )] = None,

    category: Annotated[str | None, Query(
        description="Product category",
        enum=["electronics", "clothing", "books", "food"]
    )] = None,

    min_price: Annotated[float | None, Query(
        ge=0,
        le=10000,
        description="Minimum price (0-10000)"
    )] = None,

    max_price: Annotated[float | None, Query(
        ge=0,
        le=10000,
        description="Maximum price (0-10000)"
    )] = None,

    tags: Annotated[list[str] | None, Query(
        max_length=20,
        description="Filter by tags (max 10)"
    )] = None,

    page: Annotated[int, Query(
        ge=1,
        le=1000,
        description="Page number (1-1000)"
    )] = 1,

    page_size: Annotated[int, Query(
        ge=1,
        le=100,
        description="Items per page (1-100)"
    )] = 20
):
    """
    Search products with validated query parameters.

    Validates:
    - Search query format
    - Category enum values
    - Price ranges
    - Tag limits
    - Pagination bounds
    """
    # Additional validation logic
    if min_price and max_price and min_price > max_price:
        raise HTTPException(400, "min_price cannot be greater than max_price")

    if tags and len(tags) > 10:
        raise HTTPException(400, "Maximum 10 tags allowed")

    return search_products_in_db(
        query=q,
        category=category,
        min_price=min_price,
        max_price=max_price,
        tags=tags,
        page=page,
        page_size=page_size
    )
```

### 2. **Path Parameter Validation**

```python
@app.get("/users/{user_id}/posts/{post_id}")
async def get_user_post(
    user_id: Annotated[int, Path(
        gt=0,
        le=2147483647,  # Max int value
        description="User ID (must be positive integer)"
    )],
    post_id: Annotated[int, Path(
        gt=0,
        description="Post ID (must be positive integer)"
    )]
):
    """Get post with validated path parameters"""
    return get_post(user_id, post_id)

@app.get("/files/{file_path:path}")
async def get_file(
    file_path: Annotated[str, Path(
        regex=r"^[a-zA-Z0-9/_.-]+$",
        description="File path (alphanumeric, /, _, ., - only)"
    )]
):
    """Get file with validated path"""
    # Additional security: prevent directory traversal
    if ".." in file_path:
        raise HTTPException(400, "Invalid file path")

    return FileResponse(file_path)
```

---

## Input Sanitization

### 1. **HTML Sanitization**

```python
import bleach
from fastapi import HTTPException

def sanitize_html(html: str) -> str:
    """Sanitize HTML input to prevent XSS"""
    allowed_tags = ['p', 'br', 'strong', 'em', 'u', 'a', 'ul', 'ol', 'li']
    allowed_attributes = {'a': ['href', 'title']}

    sanitized = bleach.clean(
        html,
        tags=allowed_tags,
        attributes=allowed_attributes,
        strip=True
    )

    return sanitized

class PostCreate(BaseModel):
    """Post with sanitized content"""
    title: str
    content: str

    @validator('title', 'content')
    def sanitize_fields(cls, v):
        """Sanitize HTML in text fields"""
        return sanitize_html(v)

@app.post("/posts")
async def create_post(post: PostCreate):
    """Create post with sanitized content"""
    return create_post_in_db(post)
```

### 2. **SQL Injection Prevention**

```python
from sqlalchemy import text

# ❌ BAD: Vulnerable to SQL injection
@app.get("/users/search")
async def search_users_bad(username: str, db: Session = Depends(get_db)):
    """VULNERABLE - DO NOT USE"""
    query = f"SELECT * FROM users WHERE username = '{username}'"
    result = db.execute(text(query))
    return result.fetchall()

# ✅ GOOD: Using parameterized queries
@app.get("/users/search")
async def search_users_good(username: str, db: Session = Depends(get_db)):
    """Safe from SQL injection"""
    query = text("SELECT * FROM users WHERE username = :username")
    result = db.execute(query, {"username": username})
    return result.fetchall()

# ✅ EVEN BETTER: Using ORM
@app.get("/users/search")
async def search_users_orm(username: str, db: Session = Depends(get_db)):
    """Safe using SQLAlchemy ORM"""
    return db.query(User).filter(User.username == username).all()
```

### 3. **Command Injection Prevention**

```python
import subprocess
import shlex

# ❌ BAD: Vulnerable to command injection
@app.post("/process-file")
async def process_file_bad(filename: str):
    """VULNERABLE - DO NOT USE"""
    subprocess.run(f"process {filename}", shell=True)  # Dangerous!
    return {"status": "processed"}

# ✅ GOOD: Using list arguments, no shell
@app.post("/process-file")
async def process_file_good(filename: str):
    """Safe from command injection"""
    # Validate filename first
    if not filename.replace(".", "").replace("_", "").isalnum():
        raise HTTPException(400, "Invalid filename")

    # Use list arguments, avoid shell=True
    subprocess.run(["process", filename], shell=False)
    return {"status": "processed"}
```

---

## File Upload Validation

### 1. **File Type and Size Validation**

```python
from fastapi import File, UploadFile
import magic  # python-magic

ALLOWED_IMAGE_TYPES = {"image/jpeg", "image/png", "image/gif"}
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5 MB

async def validate_image_upload(file: UploadFile) -> UploadFile:
    """Validate uploaded image"""
    # Check file size
    contents = await file.read()
    if len(contents) > MAX_FILE_SIZE:
        raise HTTPException(400, f"File too large. Maximum size: {MAX_FILE_SIZE / 1024 / 1024}MB")

    # Check file type by content (not just extension)
    mime_type = magic.from_buffer(contents, mime=True)
    if mime_type not in ALLOWED_IMAGE_TYPES:
        raise HTTPException(400, f"Invalid file type: {mime_type}. Allowed: {ALLOWED_IMAGE_TYPES}")

    # Reset file pointer
    await file.seek(0)

    return file

@app.post("/upload-image")
async def upload_image(
    file: UploadFile = File(..., description="Image file (JPEG, PNG, GIF, max 5MB)")
):
    """Upload image with validation"""
    validated_file = await validate_image_upload(file)

    # Save file
    file_path = f"uploads/{validated_file.filename}"
    with open(file_path, "wb") as f:
        f.write(await validated_file.read())

    return {"filename": validated_file.filename, "size": len(contents)}
```

### 2. **Multiple File Upload Validation**

```python
@app.post("/upload-multiple")
async def upload_multiple(
    files: list[UploadFile] = File(..., description="Multiple files")
):
    """Upload multiple files with validation"""
    if len(files) > 10:
        raise HTTPException(400, "Maximum 10 files allowed")

    uploaded_files = []
    total_size = 0

    for file in files:
        # Validate each file
        validated_file = await validate_image_upload(file)

        contents = await validated_file.read()
        total_size += len(contents)

        # Check total size
        if total_size > 20 * 1024 * 1024:  # 20 MB total
            raise HTTPException(400, "Total upload size exceeds 20MB")

        # Save file
        file_path = f"uploads/{validated_file.filename}"
        with open(file_path, "wb") as f:
            f.write(contents)

        uploaded_files.append(validated_file.filename)

    return {"files": uploaded_files, "total_size": total_size}
```

---

## Custom Validation Middleware

### 1. **Request Body Size Limit**

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class RequestSizeLimitMiddleware(BaseHTTPMiddleware):
    """Limit request body size"""

    def __init__(self, app, max_size: int = 10 * 1024 * 1024):  # 10 MB default
        super().__init__(app)
        self.max_size = max_size

    async def dispatch(self, request: Request, call_next):
        if request.method in ["POST", "PUT", "PATCH"]:
            content_length = request.headers.get("content-length")

            if content_length:
                content_length = int(content_length)
                if content_length > self.max_size:
                    return JSONResponse(
                        status_code=413,
                        content={"detail": f"Request body too large. Maximum: {self.max_size} bytes"}
                    )

        return await call_next(request)

app.add_middleware(RequestSizeLimitMiddleware, max_size=10 * 1024 * 1024)
```

### 2. **Content-Type Validation**

```python
class ContentTypeValidationMiddleware(BaseHTTPMiddleware):
    """Validate request Content-Type"""

    ALLOWED_CONTENT_TYPES = {
        "application/json",
        "multipart/form-data",
        "application/x-www-form-urlencoded"
    }

    async def dispatch(self, request: Request, call_next):
        if request.method in ["POST", "PUT", "PATCH"]:
            content_type = request.headers.get("content-type", "").split(";")[0]

            if content_type and content_type not in self.ALLOWED_CONTENT_TYPES:
                return JSONResponse(
                    status_code=415,
                    content={
                        "detail": f"Unsupported Media Type: {content_type}",
                        "allowed": list(self.ALLOWED_CONTENT_TYPES)
                    }
                )

        return await call_next(request)

app.add_middleware(ContentTypeValidationMiddleware)
```

---

## Best Practices

### ✅ Do's:

1. **Validate all inputs** (query, path, body)
2. **Use Pydantic models** for request validation
3. **Sanitize HTML content**
4. **Use parameterized queries** (prevent SQL injection)
5. **Validate file uploads** (type, size)
6. **Provide clear error messages**
7. **Normalize data** (lowercase emails, trim whitespace)
8. **Use regex** for format validation

### ❌ Don'ts:

1. **Don't trust user input**
2. **Don't use string concatenation** for SQL queries
3. **Don't skip file type validation**
4. **Don't allow unlimited file sizes**
5. **Don't execute user input** as code
6. **Don't ignore edge cases**
7. **Don't use `shell=True`** in subprocess

---

## Interview Questions

### Q1: What's the difference between validation and sanitization?

**Answer**:

- **Validation**: Check if input meets requirements (format, length, range). Reject invalid input.
- **Sanitization**: Clean/modify input to make it safe (remove HTML tags, escape special characters).
  Both are needed for security.

### Q2: How does Pydantic help with validation?

**Answer**: Pydantic:

- Validates types automatically
- Supports custom validators (@validator)
- Provides detailed error messages
- Coerces compatible types
- Integrates with FastAPI for automatic validation

### Q3: How do you prevent SQL injection in Python?

**Answer**:

- Use parameterized queries with placeholders
- Use ORM (SQLAlchemy) which escapes inputs
- Never concatenate user input into SQL strings
- Validate and sanitize all inputs

### Q4: What should you validate in file uploads?

**Answer**:

- File size (prevent DOS)
- File type (by content, not extension)
- Filename (prevent path traversal)
- Total upload size (multiple files)
- Image dimensions (for images)
- Virus scanning (production)

### Q5: How do you handle validation errors in FastAPI?

**Answer**: FastAPI automatically:

- Returns 422 with validation details
- Includes field location and error message
- Uses Pydantic's error handling
  Custom handling via `@app.exception_handler(RequestValidationError)`

---

## Summary

Input validation and sanitization involve:

- **Pydantic models** for structured validation
- **Field constraints** (min/max, regex, enum)
- **Custom validators** for business logic
- **HTML sanitization** to prevent XSS
- **Parameterized queries** to prevent SQL injection
- **File upload validation** for security
- **Clear error messages** for clients

Proper validation is the first line of defense! ✅
