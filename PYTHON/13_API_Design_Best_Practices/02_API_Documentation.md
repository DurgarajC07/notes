# 📚 API Documentation with OpenAPI

## Overview

API documentation is crucial for developer experience. OpenAPI (formerly Swagger) is the industry standard for documenting REST APIs, providing interactive documentation, code generation, and validation.

---

## FastAPI Built-in Documentation

### 1. **Automatic OpenAPI Generation**

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(
    title="My API",
    description="A comprehensive API for managing resources",
    version="1.0.0",
    terms_of_service="https://example.com/terms",
    contact={
        "name": "API Support",
        "url": "https://example.com/support",
        "email": "support@example.com"
    },
    license_info={
        "name": "MIT License",
        "url": "https://opensource.org/licenses/MIT"
    },
    openapi_tags=[
        {
            "name": "users",
            "description": "Operations with users"
        },
        {
            "name": "products",
            "description": "Product management"
        }
    ]
)

# Auto-generated docs available at:
# /docs - Swagger UI
# /redoc - ReDoc
# /openapi.json - OpenAPI schema
```

### 2. **Endpoint Documentation**

```python
class UserCreate(BaseModel):
    """Model for creating a user"""
    username: str
    email: str
    full_name: str | None = None

class UserResponse(BaseModel):
    """User response model"""
    id: int
    username: str
    email: str
    full_name: str | None

@app.post(
    "/users",
    response_model=UserResponse,
    status_code=201,
    tags=["users"],
    summary="Create a new user",
    description="Create a new user with the provided information. Username must be unique.",
    response_description="The created user",
    responses={
        201: {
            "description": "User successfully created",
            "content": {
                "application/json": {
                    "example": {
                        "id": 1,
                        "username": "john_doe",
                        "email": "john@example.com",
                        "full_name": "John Doe"
                    }
                }
            }
        },
        400: {"description": "Invalid request data"},
        409: {"description": "Username already exists"}
    }
)
async def create_user(user: UserCreate):
    """
    Create a new user.

    Args:
        user: User creation data

    Returns:
        UserResponse: The created user

    Raises:
        HTTPException: 400 if validation fails
        HTTPException: 409 if username exists
    """
    return create_user_in_db(user)
```

---

## Detailed Response Documentation

### 1. **Response Models**

```python
from typing import List, Optional
from pydantic import BaseModel, Field

class PaginationMeta(BaseModel):
    """Pagination metadata"""
    total: int = Field(..., description="Total number of items")
    page: int = Field(..., description="Current page number")
    page_size: int = Field(..., description="Items per page")
    total_pages: int = Field(..., description="Total number of pages")

class UserListResponse(BaseModel):
    """Paginated list of users"""
    users: List[UserResponse] = Field(..., description="List of users")
    meta: PaginationMeta = Field(..., description="Pagination information")

    class Config:
        schema_extra = {
            "example": {
                "users": [
                    {
                        "id": 1,
                        "username": "john_doe",
                        "email": "john@example.com",
                        "full_name": "John Doe"
                    }
                ],
                "meta": {
                    "total": 100,
                    "page": 1,
                    "page_size": 20,
                    "total_pages": 5
                }
            }
        }

@app.get(
    "/users",
    response_model=UserListResponse,
    tags=["users"],
    summary="List all users"
)
async def list_users(
    page: int = Query(1, ge=1, description="Page number"),
    page_size: int = Query(20, ge=1, le=100, description="Items per page")
):
    """
    Retrieve a paginated list of users.

    - **page**: Current page (starts at 1)
    - **page_size**: Number of users per page (1-100)
    """
    return get_users_paginated(page, page_size)
```

### 2. **Error Response Documentation**

```python
class ErrorDetail(BaseModel):
    """Error detail model"""
    loc: List[str] = Field(..., description="Location of the error")
    msg: str = Field(..., description="Error message")
    type: str = Field(..., description="Error type")

class ErrorResponse(BaseModel):
    """Standard error response"""
    detail: str | List[ErrorDetail] = Field(..., description="Error details")

    class Config:
        schema_extra = {
            "example": {
                "detail": "User not found"
            }
        }

class ValidationErrorResponse(BaseModel):
    """Validation error response"""
    detail: List[ErrorDetail]

    class Config:
        schema_extra = {
            "example": {
                "detail": [
                    {
                        "loc": ["body", "email"],
                        "msg": "value is not a valid email address",
                        "type": "value_error.email"
                    }
                ]
            }
        }

@app.get(
    "/users/{user_id}",
    response_model=UserResponse,
    responses={
        404: {
            "model": ErrorResponse,
            "description": "User not found"
        }
    }
)
async def get_user(user_id: int):
    """Get user by ID"""
    user = find_user(user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return user
```

---

## Request Documentation

### 1. **Query Parameters**

```python
from enum import Enum

class SortOrder(str, Enum):
    """Sort order enum"""
    asc = "asc"
    desc = "desc"

@app.get("/products")
async def search_products(
    q: str | None = Query(
        None,
        title="Search query",
        description="Search term for product name or description",
        min_length=3,
        max_length=50,
        example="laptop"
    ),
    category: str | None = Query(
        None,
        title="Category filter",
        description="Filter products by category",
        example="electronics"
    ),
    min_price: float | None = Query(
        None,
        title="Minimum price",
        description="Filter products with price greater than or equal to this value",
        ge=0,
        example=100.0
    ),
    max_price: float | None = Query(
        None,
        title="Maximum price",
        description="Filter products with price less than or equal to this value",
        ge=0,
        example=1000.0
    ),
    sort_by: str = Query(
        "created_at",
        title="Sort field",
        description="Field to sort by",
        enum=["name", "price", "created_at"]
    ),
    sort_order: SortOrder = Query(
        SortOrder.desc,
        title="Sort order",
        description="Sort direction"
    ),
    page: int = Query(
        1,
        title="Page number",
        description="Page number for pagination",
        ge=1
    ),
    page_size: int = Query(
        20,
        title="Page size",
        description="Number of items per page",
        ge=1,
        le=100
    )
):
    """
    Search and filter products with pagination.

    Query parameters:
    - **q**: Search term (min 3 characters)
    - **category**: Filter by product category
    - **min_price**: Minimum price filter (≥ 0)
    - **max_price**: Maximum price filter (≥ 0)
    - **sort_by**: Field to sort by (name, price, created_at)
    - **sort_order**: Sort direction (asc, desc)
    - **page**: Page number (≥ 1)
    - **page_size**: Items per page (1-100)
    """
    return search_products_in_db(
        query=q,
        category=category,
        min_price=min_price,
        max_price=max_price,
        sort_by=sort_by,
        sort_order=sort_order,
        page=page,
        page_size=page_size
    )
```

### 2. **Path Parameters**

```python
@app.get("/users/{user_id}/posts/{post_id}")
async def get_user_post(
    user_id: int = Path(
        ...,
        title="User ID",
        description="The unique identifier of the user",
        gt=0,
        example=123
    ),
    post_id: int = Path(
        ...,
        title="Post ID",
        description="The unique identifier of the post",
        gt=0,
        example=456
    )
):
    """
    Get a specific post by a user.

    Path parameters:
    - **user_id**: Unique user identifier (> 0)
    - **post_id**: Unique post identifier (> 0)
    """
    return get_post_by_user(user_id, post_id)
```

### 3. **Request Body**

```python
class ProductCreate(BaseModel):
    """Product creation model"""
    name: str = Field(
        ...,
        title="Product name",
        description="Name of the product",
        min_length=1,
        max_length=200,
        example="Wireless Mouse"
    )
    description: str | None = Field(
        None,
        title="Product description",
        description="Detailed description of the product",
        max_length=1000,
        example="Ergonomic wireless mouse with 6 buttons"
    )
    price: float = Field(
        ...,
        title="Product price",
        description="Price in USD",
        gt=0,
        example=29.99
    )
    category: str = Field(
        ...,
        title="Product category",
        description="Category the product belongs to",
        example="electronics"
    )
    in_stock: bool = Field(
        True,
        title="In stock",
        description="Whether the product is currently in stock"
    )
    tags: List[str] = Field(
        default_factory=list,
        title="Product tags",
        description="Tags for categorizing the product",
        example=["wireless", "computer accessories"]
    )

    class Config:
        schema_extra = {
            "example": {
                "name": "Wireless Mouse",
                "description": "Ergonomic wireless mouse with 6 buttons",
                "price": 29.99,
                "category": "electronics",
                "in_stock": true,
                "tags": ["wireless", "computer accessories"]
            }
        }

@app.post(
    "/products",
    response_model=ProductResponse,
    status_code=201,
    summary="Create a new product"
)
async def create_product(
    product: ProductCreate = Body(
        ...,
        title="Product data",
        description="Data for creating a new product"
    )
):
    """
    Create a new product.

    Request body should include:
    - **name**: Product name (required, 1-200 chars)
    - **description**: Detailed description (optional, max 1000 chars)
    - **price**: Price in USD (required, > 0)
    - **category**: Product category (required)
    - **in_stock**: Availability status (default: true)
    - **tags**: List of tags (optional)
    """
    return create_product_in_db(product)
```

---

## Authentication Documentation

### 1. **Security Schemes**

````python
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

app = FastAPI(
    title="Secure API",
    openapi_tags=[...],
    # Define security schemes
    swagger_ui_init_oauth={
        "clientId": "your-client-id",
        "appName": "Your API"
    }
)

# Add security scheme to OpenAPI
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema

    openapi_schema = get_openapi(
        title=app.title,
        version=app.version,
        description=app.description,
        routes=app.routes,
    )

    openapi_schema["components"]["securitySchemes"] = {
        "BearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT"
        },
        "APIKeyAuth": {
            "type": "apiKey",
            "in": "header",
            "name": "X-API-Key"
        }
    }

    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi

@app.get(
    "/protected",
    dependencies=[Depends(security)],
    summary="Protected endpoint"
)
async def protected_route(
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    """
    Access protected resource.

    Requires Bearer token in Authorization header:
    ```
    Authorization: Bearer <your-jwt-token>
    ```
    """
    return {"message": "Access granted"}
````

---

## Advanced Documentation

### 1. **Multiple Response Types**

```python
from fastapi.responses import FileResponse, StreamingResponse

@app.get(
    "/export/users",
    responses={
        200: {
            "content": {
                "application/json": {
                    "example": [
                        {"id": 1, "name": "User 1"},
                        {"id": 2, "name": "User 2"}
                    ]
                },
                "text/csv": {
                    "example": "id,name\n1,User 1\n2,User 2"
                },
                "application/pdf": {
                    "description": "PDF export of users"
                }
            },
            "description": "Export users in various formats"
        }
    }
)
async def export_users(format: str = Query("json", enum=["json", "csv", "pdf"])):
    """
    Export users in different formats.

    Supported formats:
    - **json**: JSON array of users
    - **csv**: CSV format
    - **pdf**: PDF document
    """
    if format == "json":
        return get_users_json()
    elif format == "csv":
        return StreamingResponse(
            generate_csv(),
            media_type="text/csv",
            headers={"Content-Disposition": "attachment; filename=users.csv"}
        )
    elif format == "pdf":
        return FileResponse("users.pdf", media_type="application/pdf")
```

### 2. **Deprecation**

```python
@app.get(
    "/old-endpoint",
    deprecated=True,
    summary="Old endpoint (deprecated)",
    description="This endpoint is deprecated. Use /new-endpoint instead."
)
async def old_endpoint():
    """
    **DEPRECATED**: Use /new-endpoint instead.

    This endpoint will be removed in version 2.0.
    """
    return {"message": "Use /new-endpoint"}

@app.get(
    "/new-endpoint",
    summary="New endpoint",
    description="Replacement for deprecated /old-endpoint"
)
async def new_endpoint():
    """New and improved endpoint"""
    return {"message": "Welcome to the new endpoint"}
```

### 3. **Custom Examples**

```python
class UserUpdate(BaseModel):
    """User update model with multiple examples"""
    username: str | None = None
    email: str | None = None
    full_name: str | None = None

    class Config:
        schema_extra = {
            "examples": {
                "update_username": {
                    "summary": "Update username",
                    "description": "Update only the username",
                    "value": {
                        "username": "new_username"
                    }
                },
                "update_email": {
                    "summary": "Update email",
                    "description": "Update only the email",
                    "value": {
                        "email": "newemail@example.com"
                    }
                },
                "update_all": {
                    "summary": "Update all fields",
                    "description": "Update all user fields",
                    "value": {
                        "username": "new_username",
                        "email": "newemail@example.com",
                        "full_name": "New Full Name"
                    }
                }
            }
        }

@app.patch("/users/{user_id}")
async def update_user(user_id: int, updates: UserUpdate):
    """Update user with various field combinations"""
    return update_user_in_db(user_id, updates)
```

---

## External Documentation Links

```python
@app.get(
    "/complex-operation",
    summary="Complex operation",
    description="Performs a complex operation. See external docs for details.",
    externalDocs={
        "description": "External documentation",
        "url": "https://docs.example.com/complex-operation"
    }
)
async def complex_operation():
    """See external documentation for detailed explanation"""
    return {"status": "completed"}
```

---

## Best Practices

### ✅ Do's:

1. **Provide clear descriptions** for all endpoints
2. **Document all parameters** with examples
3. **Include response examples**
4. **Document error responses**
5. **Use tags** to organize endpoints
6. **Mark deprecated endpoints**
7. **Include authentication requirements**
8. **Provide external links** for complex topics

### ❌ Don'ts:

1. **Don't leave endpoints undocumented**
2. **Don't use generic descriptions**
3. **Don't forget to document errors**
4. **Don't skip parameter validation**
5. **Don't ignore security documentation**

---

## Interview Questions

### Q1: What is OpenAPI and why is it important?

**Answer**: OpenAPI (formerly Swagger) is a standard specification for documenting REST APIs. Benefits:

- Interactive documentation (Swagger UI)
- Code generation for clients
- API validation
- Developer onboarding
- Contract-first development

### Q2: How does FastAPI automatically generate documentation?

**Answer**: FastAPI uses:

- Pydantic models for request/response schemas
- Type hints for parameter validation
- Docstrings for descriptions
- Function annotations for metadata
  Generates OpenAPI JSON accessible at `/openapi.json`, with UIs at `/docs` and `/redoc`.

### Q3: What should you document in API responses?

**Answer**:

- Success response structure
- All possible status codes (200, 201, 400, 404, etc.)
- Error response formats
- Response headers
- Examples for each response type

### Q4: How do you document authentication in OpenAPI?

**Answer**: Define security schemes in OpenAPI spec:

- Bearer tokens (JWT)
- API keys
- OAuth2 flows
- Basic authentication
  Then reference scheme in endpoint definitions with `dependencies` or `security`.

### Q5: What's the difference between Swagger UI and ReDoc?

**Answer**:

- **Swagger UI**: Interactive, can make test requests, more features
- **ReDoc**: Cleaner design, better for reading, less interactive
  Both consume same OpenAPI specification.

---

## Summary

API documentation includes:

- **Automatic generation** from code annotations
- **Detailed endpoint descriptions** with examples
- **Request/response models** with validation
- **Error response documentation**
- **Authentication requirements**
- **Interactive testing** with Swagger UI

Good documentation improves developer experience! 📚
