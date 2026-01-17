# 🎨 RESTful API Design Principles

## Overview

REST (Representational State Transfer) is an architectural style for designing networked applications. Understanding RESTful principles is essential for building intuitive, scalable, and maintainable APIs.

---

## Core REST Principles

### 1. **Client-Server Architecture**

- Separation of concerns
- Client handles UI, server handles data
- Independent evolution of each side

### 2. **Statelessness**

- Each request contains all necessary information
- No session state stored on server
- Improves scalability

### 3. **Cacheability**

- Responses must define themselves as cacheable or not
- Improves performance and scalability

### 4. **Uniform Interface**

- Resource identification via URIs
- Resource manipulation through representations
- Self-descriptive messages
- HATEOAS (Hypermedia as the Engine of Application State)

### 5. **Layered System**

- Client cannot tell if connected directly to end server
- Allows for load balancers, proxies, caches

### 6. **Code on Demand** (Optional)

- Server can extend client functionality
- JavaScript execution in browsers

---

## Resource Design

### 1. **Resource Naming Conventions**

```
✅ GOOD:
/users                  # Collection
/users/123              # Specific resource
/users/123/orders       # Sub-resource collection
/users/123/orders/456   # Specific sub-resource
/products?category=electronics&sort=price  # Query parameters

❌ BAD:
/getUsers               # Don't use verbs
/user                   # Use plural for collections
/users/getUserOrders    # Redundant, HTTP methods define action
/user-orders            # Use sub-resources instead
```

### 2. **Resource Hierarchy**

```python
# Well-designed resource hierarchy
/users                    # All users
/users/{userId}           # Specific user
/users/{userId}/posts     # User's posts
/users/{userId}/posts/{postId}  # Specific post
/users/{userId}/posts/{postId}/comments  # Post comments

# FastAPI implementation
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/users")
async def get_users():
    """Get all users"""
    return {"users": []}

@app.get("/users/{user_id}")
async def get_user(user_id: int = Path(..., gt=0)):
    """Get specific user"""
    return {"user_id": user_id}

@app.get("/users/{user_id}/posts")
async def get_user_posts(user_id: int):
    """Get user's posts"""
    return {"user_id": user_id, "posts": []}

@app.get("/users/{user_id}/posts/{post_id}")
async def get_user_post(user_id: int, post_id: int):
    """Get specific post"""
    return {"user_id": user_id, "post_id": post_id}
```

---

## HTTP Methods

### 1. **Standard HTTP Methods**

```python
# GET - Retrieve resources (Safe & Idempotent)
@app.get("/products")
async def get_products():
    """List all products"""
    return {"products": [...]}

@app.get("/products/{product_id}")
async def get_product(product_id: int):
    """Get single product"""
    return {"id": product_id, "name": "Product"}

# POST - Create new resource (Not idempotent)
@app.post("/products", status_code=status.HTTP_201_CREATED)
async def create_product(product: ProductCreate):
    """Create new product"""
    new_product = {...}
    return new_product

# PUT - Update/Replace entire resource (Idempotent)
@app.put("/products/{product_id}")
async def update_product(product_id: int, product: ProductUpdate):
    """Replace entire product"""
    return {"id": product_id, **product.dict()}

# PATCH - Partial update (Not necessarily idempotent)
@app.patch("/products/{product_id}")
async def partial_update_product(product_id: int, updates: dict):
    """Update specific fields"""
    return {"id": product_id, "updated_fields": updates}

# DELETE - Remove resource (Idempotent)
@app.delete("/products/{product_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_product(product_id: int):
    """Delete product"""
    # Deleting resource
    return Response(status_code=status.HTTP_204_NO_CONTENT)

# HEAD - Get headers only (like GET but no body)
@app.head("/products/{product_id}")
async def head_product(product_id: int):
    """Check if product exists"""
    return Response(headers={"X-Product-Exists": "true"})

# OPTIONS - Get supported methods
@app.options("/products")
async def options_products():
    """Return allowed methods"""
    return Response(
        headers={"Allow": "GET, POST, PUT, PATCH, DELETE, OPTIONS"}
    )
```

### 2. **Method Properties**

| Method  | Safe | Idempotent | Cacheable |
| ------- | ---- | ---------- | --------- |
| GET     | ✅   | ✅         | ✅        |
| POST    | ❌   | ❌         | ⚠️        |
| PUT     | ❌   | ✅         | ❌        |
| PATCH   | ❌   | ❌         | ❌        |
| DELETE  | ❌   | ✅         | ❌        |
| HEAD    | ✅   | ✅         | ✅        |
| OPTIONS | ✅   | ✅         | ❌        |

---

## HTTP Status Codes

### 1. **Success Codes (2xx)**

```python
from fastapi import status
from fastapi.responses import Response

# 200 OK - Request succeeded
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"id": user_id}  # Default 200

# 201 Created - Resource created
@app.post("/users", status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate):
    new_user = create_user_in_db(user)
    return new_user

# 202 Accepted - Request accepted for processing
@app.post("/batch-import", status_code=status.HTTP_202_ACCEPTED)
async def batch_import(background_tasks: BackgroundTasks):
    background_tasks.add_task(process_import)
    return {"message": "Import started"}

# 204 No Content - Success but no content to return
@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    delete_user_from_db(user_id)
    return Response(status_code=status.HTTP_204_NO_CONTENT)
```

### 2. **Client Error Codes (4xx)**

```python
from fastapi import HTTPException

# 400 Bad Request - Invalid request
@app.post("/users")
async def create_user(user: UserCreate):
    if not user.email:
        raise HTTPException(400, "Email is required")

# 401 Unauthorized - Authentication required
@app.get("/profile")
async def get_profile(token: str = Depends(verify_token)):
    if not token:
        raise HTTPException(401, "Authentication required")

# 403 Forbidden - Insufficient permissions
@app.delete("/admin/users/{user_id}")
async def delete_user_admin(user_id: int, current_user = Depends(get_current_user)):
    if not current_user.is_admin:
        raise HTTPException(403, "Admin access required")

# 404 Not Found - Resource doesn't exist
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = find_user(user_id)
    if not user:
        raise HTTPException(404, f"User {user_id} not found")
    return user

# 405 Method Not Allowed - HTTP method not supported
# Automatically handled by FastAPI

# 409 Conflict - Resource conflict
@app.post("/users")
async def create_user(user: UserCreate):
    if user_exists(user.email):
        raise HTTPException(409, "User with this email already exists")

# 422 Unprocessable Entity - Validation error
# Automatically handled by Pydantic

# 429 Too Many Requests - Rate limit exceeded
@app.get("/api/data")
async def get_data(request: Request):
    if rate_limit_exceeded(request.client.host):
        raise HTTPException(
            429,
            "Rate limit exceeded",
            headers={"Retry-After": "60"}
        )
```

### 3. **Server Error Codes (5xx)**

```python
# 500 Internal Server Error - Unexpected server error
@app.get("/data")
async def get_data():
    try:
        return fetch_data()
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        raise HTTPException(500, "Internal server error")

# 502 Bad Gateway - Invalid response from upstream
# 503 Service Unavailable - Service temporarily unavailable
@app.get("/health")
async def health_check():
    if not service_available():
        raise HTTPException(503, "Service temporarily unavailable")
    return {"status": "healthy"}

# 504 Gateway Timeout - Upstream timeout
```

---

## Request/Response Format

### 1. **Request Structure**

```python
from pydantic import BaseModel, Field, validator

class UserCreate(BaseModel):
    """Request model for user creation"""
    username: str = Field(..., min_length=3, max_length=50)
    email: str = Field(..., regex=r"^[\w\.-]+@[\w\.-]+\.\w+$")
    password: str = Field(..., min_length=8)
    full_name: str | None = None

    @validator('username')
    def username_alphanumeric(cls, v):
        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')
        return v

@app.post("/users")
async def create_user(user: UserCreate):
    """
    Create new user with validated data

    Request body:
    {
        "username": "john_doe",
        "email": "john@example.com",
        "password": "SecurePass123",
        "full_name": "John Doe"
    }
    """
    return create_user_in_db(user)
```

### 2. **Response Structure**

```python
class UserResponse(BaseModel):
    """Response model for user"""
    id: int
    username: str
    email: str
    full_name: str | None
    created_at: datetime

    class Config:
        from_attributes = True

class PaginatedResponse(BaseModel):
    """Generic paginated response"""
    items: list
    total: int
    page: int
    page_size: int
    total_pages: int

@app.get("/users", response_model=PaginatedResponse)
async def get_users(page: int = 1, page_size: int = 20):
    """
    Response:
    {
        "items": [
            {"id": 1, "username": "user1", ...},
            {"id": 2, "username": "user2", ...}
        ],
        "total": 100,
        "page": 1,
        "page_size": 20,
        "total_pages": 5
    }
    """
    return get_paginated_users(page, page_size)
```

### 3. **Error Response Format**

```python
class ErrorResponse(BaseModel):
    """Standardized error response"""
    error: str
    message: str
    details: dict | None = None
    timestamp: datetime = Field(default_factory=datetime.utcnow)

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    """Custom error response format"""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": f"HTTP_{exc.status_code}",
            "message": exc.detail,
            "path": str(request.url),
            "timestamp": datetime.utcnow().isoformat()
        }
    )
```

---

## Filtering, Sorting, Pagination

### 1. **Query Parameters**

```python
from typing import Optional
from enum import Enum

class SortOrder(str, Enum):
    asc = "asc"
    desc = "desc"

@app.get("/products")
async def get_products(
    # Filtering
    category: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    in_stock: Optional[bool] = None,

    # Sorting
    sort_by: Optional[str] = "created_at",
    sort_order: SortOrder = SortOrder.desc,

    # Pagination
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),

    # Search
    search: Optional[str] = None
):
    """
    Example requests:
    GET /products?category=electronics&min_price=100&max_price=500
    GET /products?sort_by=price&sort_order=asc
    GET /products?page=2&page_size=50
    GET /products?search=laptop
    """
    filters = {}
    if category:
        filters['category'] = category
    if min_price:
        filters['min_price'] = min_price
    if max_price:
        filters['max_price'] = max_price

    return get_filtered_products(
        filters=filters,
        sort_by=sort_by,
        sort_order=sort_order,
        page=page,
        page_size=page_size,
        search=search
    )
```

---

## Content Negotiation

### 1. **Accept Header**

```python
from fastapi.responses import JSONResponse, PlainTextResponse, HTMLResponse

@app.get("/users/{user_id}")
async def get_user(user_id: int, request: Request):
    """Support multiple response formats"""
    user = get_user_by_id(user_id)

    accept = request.headers.get("accept", "application/json")

    if "application/json" in accept:
        return JSONResponse(user)
    elif "text/html" in accept:
        html = f"<h1>{user['name']}</h1><p>{user['email']}</p>"
        return HTMLResponse(html)
    elif "text/plain" in accept:
        text = f"Name: {user['name']}\nEmail: {user['email']}"
        return PlainTextResponse(text)
    else:
        raise HTTPException(406, "Not Acceptable")
```

---

## Versioning Strategies

### 1. **URI Versioning**

```python
# Version in path
@app.get("/v1/users")
async def get_users_v1():
    return {"version": "1.0", "users": []}

@app.get("/v2/users")
async def get_users_v2():
    return {"version": "2.0", "users": [], "meta": {}}
```

### 2. **Header Versioning**

```python
@app.get("/users")
async def get_users(request: Request):
    version = request.headers.get("API-Version", "1.0")

    if version == "1.0":
        return {"users": []}
    elif version == "2.0":
        return {"users": [], "meta": {}}
```

### 3. **Query Parameter Versioning**

```python
@app.get("/users")
async def get_users(version: str = "1.0"):
    if version == "1.0":
        return {"users": []}
    elif version == "2.0":
        return {"users": [], "meta": {}}
```

---

## Best Practices

### ✅ Do's:

1. **Use nouns for resources**, not verbs
2. **Use plural names** for collections
3. **Return appropriate status codes**
4. **Support pagination** for collections
5. **Use HTTP methods correctly**
6. **Version your API**
7. **Document with OpenAPI/Swagger**
8. **Use consistent naming conventions**

### ❌ Don'ts:

1. **Don't use verbs** in URIs (`/getUser`)
2. **Don't expose database structure** directly
3. **Don't use different data formats** inconsistently
4. **Don't ignore HTTP status codes**
5. **Don't return arrays** at root level
6. **Don't break backwards compatibility** without versioning

---

## Interview Questions

### Q1: What makes an API RESTful?

**Answer**: RESTful APIs follow REST principles:

- Stateless client-server architecture
- Resources identified by URIs
- Standard HTTP methods (GET, POST, PUT, DELETE)
- Self-descriptive messages
- Cacheable responses
- Uniform interface

### Q2: What's the difference between PUT and PATCH?

**Answer**:

- **PUT**: Replace entire resource. Idempotent. Requires all fields.
- **PATCH**: Partial update. Not necessarily idempotent. Updates only specified fields.

### Q3: When should you use status code 201 vs 200?

**Answer**:

- **201 Created**: Resource successfully created (POST)
- **200 OK**: Request succeeded (GET, PUT, PATCH)
  Use 201 for POST requests that create new resources.

### Q4: How do you design pagination for large datasets?

**Answer**: Three approaches:

- **Offset-based**: `?page=2&page_size=20` - Simple but slow for large offsets
- **Cursor-based**: `?cursor=abc123&limit=20` - Better performance
- **Keyset**: `?after_id=100&limit=20` - Best for sorted data

### Q5: What's HATEOAS and why is it important?

**Answer**: Hypermedia As The Engine Of Application State. API responses include links to related resources, making API self-discoverable. Example:

```json
{
  "id": 1,
  "name": "User",
  "links": {
    "self": "/users/1",
    "posts": "/users/1/posts"
  }
}
```

---

## Summary

RESTful API design involves:

- **Resource-oriented** URIs (nouns, not verbs)
- **Standard HTTP methods** for CRUD operations
- **Appropriate status codes** for responses
- **Consistent data formats** (JSON)
- **Pagination and filtering** for collections
- **Versioning** for backwards compatibility

Well-designed APIs are intuitive, scalable, and maintainable! 🎨
