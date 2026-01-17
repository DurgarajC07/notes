# FastAPI Core: Routing & Path Operations

## 📖 Introduction

FastAPI routing is based on **path operations** (endpoints) decorated with HTTP methods. FastAPI automatically generates OpenAPI documentation, validates request data, and serializes responses.

## 🔑 Basic Routing

### HTTP Method Decorators

```python
from fastapi import FastAPI, HTTPException, status
from typing import Optional

app = FastAPI()

# GET - Retrieve data
@app.get("/items")
async def list_items():
    """List all items"""
    return {"items": [1, 2, 3]}

# POST - Create data
@app.post("/items")
async def create_item(name: str, price: float):
    """Create new item"""
    return {"name": name, "price": price, "id": 123}

# PUT - Update data (replace entire resource)
@app.put("/items/{item_id}")
async def update_item(item_id: int, name: str, price: float):
    """Update item completely"""
    return {"id": item_id, "name": name, "price": price}

# PATCH - Partial update
@app.patch("/items/{item_id}")
async def partial_update(item_id: int, name: Optional[str] = None):
    """Update item partially"""
    return {"id": item_id, "name": name}

# DELETE - Delete data
@app.delete("/items/{item_id}")
async def delete_item(item_id: int):
    """Delete item"""
    return {"message": f"Item {item_id} deleted"}

# HEAD - Get headers only (no body)
@app.head("/items/{item_id}")
async def item_exists(item_id: int):
    """Check if item exists"""
    # Return headers only, no response body
    return {}

# OPTIONS - Get supported methods
@app.options("/items")
async def item_options():
    """Get supported HTTP methods"""
    return {"methods": ["GET", "POST", "PUT", "PATCH", "DELETE"]}
```

## 🛣️ Path Parameters

### Basic Path Parameters

```python
# Simple path parameter
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    """
    Path parameter with type validation

    - FastAPI validates item_id is integer
    - Returns 422 if validation fails
    """
    return {"item_id": item_id}

# Multiple path parameters
@app.get("/users/{user_id}/posts/{post_id}")
async def read_user_post(user_id: int, post_id: int):
    """Multiple path parameters"""
    return {"user_id": user_id, "post_id": post_id}

# String path parameter
@app.get("/users/{username}")
async def read_user(username: str):
    """String path parameter"""
    return {"username": username}
```

### Path Parameter with Enum

```python
from enum import Enum

class ModelName(str, Enum):
    """Enum for model names"""
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    """
    Path parameter with enum validation

    - Only accepts: alexnet, resnet, lenet
    - Auto-generates dropdown in docs
    """
    if model_name == ModelName.alexnet:
        return {"model_name": model_name, "message": "AlexNet model"}

    if model_name.value == "resnet":
        return {"model_name": model_name, "message": "ResNet model"}

    return {"model_name": model_name, "message": "LeCNN model"}
```

### Path Parameter Validation

```python
from fastapi import Path

@app.get("/items/{item_id}")
async def read_item(
    item_id: int = Path(
        ...,  # Required (ellipsis)
        title="Item ID",
        description="The ID of the item to retrieve",
        ge=1,  # Greater than or equal to 1
        le=1000,  # Less than or equal to 1000
    )
):
    """
    Path parameter with constraints

    - Must be between 1 and 1000
    - Metadata for documentation
    """
    return {"item_id": item_id}

# Greater than, less than
@app.get("/products/{product_id}")
async def read_product(
    product_id: int = Path(..., gt=0, lt=10000)
):
    """Product ID must be > 0 and < 10000"""
    return {"product_id": product_id}
```

### Path Parameter with Path

```python
# File path as parameter
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    """
    Path parameter that matches any path

    Example: /files/docs/python/intro.md
    file_path = "docs/python/intro.md"
    """
    return {"file_path": file_path}
```

## 🔍 Query Parameters

### Basic Query Parameters

```python
@app.get("/items")
async def read_items(
    skip: int = 0,
    limit: int = 10,
    q: Optional[str] = None
):
    """
    Query parameters

    URL: /items?skip=0&limit=10&q=search

    - skip: offset (default 0)
    - limit: page size (default 10)
    - q: search query (optional)
    """
    items = [{"id": i} for i in range(skip, skip + limit)]

    if q:
        items = [item for item in items if q in str(item)]

    return {"skip": skip, "limit": limit, "q": q, "items": items}

# Boolean query parameter
@app.get("/products")
async def list_products(
    in_stock: bool = True,
    featured: bool = False
):
    """
    Boolean query parameters

    URL: /products?in_stock=true&featured=false

    Accepted values: true, false, 1, 0, yes, no, on, off
    """
    return {"in_stock": in_stock, "featured": featured}
```

### Query Parameter Validation

```python
from fastapi import Query

@app.get("/items")
async def read_items(
    q: Optional[str] = Query(
        None,  # Default value
        title="Query string",
        description="Search query for items",
        min_length=3,  # Minimum length
        max_length=50,  # Maximum length
        regex="^[a-zA-Z0-9 ]+$"  # Pattern validation
    )
):
    """Query parameter with validation"""
    return {"q": q}

# Required query parameter
@app.get("/search")
async def search(
    q: str = Query(
        ...,  # Required
        min_length=1,
        description="Search query (required)"
    )
):
    """Required query parameter"""
    return {"q": q}

# List query parameter
@app.get("/items")
async def read_items(
    tags: list[str] = Query(
        [],  # Default empty list
        title="Tags",
        description="Filter by tags"
    )
):
    """
    Multiple values for same parameter

    URL: /items?tags=python&tags=fastapi&tags=async
    tags = ["python", "fastapi", "async"]
    """
    return {"tags": tags}

# Numeric validation
@app.get("/products")
async def list_products(
    price_min: float = Query(0, ge=0, description="Minimum price"),
    price_max: float = Query(None, ge=0, le=10000, description="Maximum price")
):
    """Numeric range validation"""
    return {"price_min": price_min, "price_max": price_max}
```

## 🎨 Request Body

### Pydantic Models

```python
from pydantic import BaseModel, Field, validator
from datetime import datetime
from typing import List, Optional

class Item(BaseModel):
    """Item model"""
    name: str = Field(..., min_length=1, max_length=100)
    description: Optional[str] = Field(None, max_length=500)
    price: float = Field(..., gt=0)
    tax: Optional[float] = Field(None, ge=0, le=100)
    tags: List[str] = []

    @validator('price')
    def price_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError('Price must be positive')
        return v

@app.post("/items")
async def create_item(item: Item):
    """
    Request body with Pydantic model

    - Automatic validation
    - JSON schema in docs
    - Type hints for IDE
    """
    item_dict = item.dict()

    if item.tax:
        price_with_tax = item.price + item.tax
        item_dict.update({"price_with_tax": price_with_tax})

    return item_dict
```

### Multiple Body Parameters

```python
class User(BaseModel):
    """User model"""
    username: str
    email: str
    full_name: Optional[str] = None

@app.put("/items/{item_id}")
async def update_item(
    item_id: int,
    item: Item,
    user: User,
    importance: int = Body(...)  # Single body value
):
    """
    Multiple body parameters

    Request body:
    {
        "item": {...},
        "user": {...},
        "importance": 5
    }
    """
    return {
        "item_id": item_id,
        "item": item,
        "user": user,
        "importance": importance
    }
```

## 🎯 Response Model

### Response Model with Pydantic

```python
class ItemResponse(BaseModel):
    """Item response model"""
    id: int
    name: str
    price: float
    created_at: datetime

@app.post("/items", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
async def create_item(item: Item) -> ItemResponse:
    """
    Response model with Pydantic

    - Filters response fields
    - Validates response data
    - Documents response schema
    """
    # Save item to database (simulated)
    item_data = {
        "id": 123,
        "name": item.name,
        "price": item.price,
        "created_at": datetime.utcnow()
    }

    return ItemResponse(**item_data)

# List response
@app.get("/items", response_model=List[ItemResponse])
async def list_items() -> List[ItemResponse]:
    """List of items"""
    return [
        ItemResponse(id=1, name="Item 1", price=10.0, created_at=datetime.utcnow()),
        ItemResponse(id=2, name="Item 2", price=20.0, created_at=datetime.utcnow()),
    ]
```

### Response Model Filtering

```python
class UserIn(BaseModel):
    """User input (with password)"""
    username: str
    password: str
    email: str
    full_name: Optional[str] = None

class UserOut(BaseModel):
    """User output (without password)"""
    username: str
    email: str
    full_name: Optional[str] = None

@app.post("/users", response_model=UserOut)
async def create_user(user: UserIn):
    """
    Response model filters sensitive data

    - Input includes password
    - Output excludes password
    """
    # Save user (simulated)
    return user  # Password automatically filtered
```

## 🏷️ Tags and Metadata

### Organize with Tags

```python
@app.get("/items", tags=["items"])
async def list_items():
    """List items"""
    return {"items": []}

@app.post("/items", tags=["items"])
async def create_item():
    """Create item"""
    return {"id": 1}

@app.get("/users", tags=["users"])
async def list_users():
    """List users"""
    return {"users": []}

@app.post("/users", tags=["users"])
async def create_user():
    """Create user"""
    return {"id": 1}

# Multiple tags
@app.get("/admin/items", tags=["items", "admin"])
async def admin_list_items():
    """Admin view of items"""
    return {"items": []}
```

### Operation Metadata

```python
@app.post(
    "/items",
    tags=["items"],
    summary="Create an item",
    description="Create a new item with name, description, and price",
    response_description="The created item",
    status_code=status.HTTP_201_CREATED
)
async def create_item(item: Item):
    """
    Create a new item with all details:

    - **name**: Item name (required)
    - **description**: Item description (optional)
    - **price**: Item price (required, > 0)
    - **tax**: Item tax (optional)
    - **tags**: Item tags (optional list)
    """
    return item
```

## 🎭 APIRouter for Modular Apps

### Create Router

```python
# routes/items.py
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter(
    prefix="/items",
    tags=["items"],
    responses={404: {"description": "Not found"}}
)

@router.get("/")
async def list_items():
    """List all items"""
    return {"items": []}

@router.get("/{item_id}")
async def read_item(item_id: int):
    """Get single item"""
    return {"item_id": item_id}

@router.post("/")
async def create_item(item: Item):
    """Create item"""
    return item

@router.put("/{item_id}")
async def update_item(item_id: int, item: Item):
    """Update item"""
    return {"item_id": item_id, **item.dict()}

@router.delete("/{item_id}")
async def delete_item(item_id: int):
    """Delete item"""
    return {"message": "Item deleted"}
```

### Include Router in App

```python
# main.py
from fastapi import FastAPI
from routes import items, users

app = FastAPI()

# Include routers
app.include_router(items.router)
app.include_router(users.router)

# Router with different prefix
app.include_router(
    items.router,
    prefix="/api/v1",
    tags=["v1"]
)
```

## 🔀 Sub-applications

### Mount Sub-application

```python
from fastapi import FastAPI

# Main app
app = FastAPI()

# Sub-app
sub_app = FastAPI()

@sub_app.get("/items")
async def sub_items():
    """Sub-app items endpoint"""
    return {"app": "sub", "items": []}

# Mount sub-app
app.mount("/subapi", sub_app)

"""
Endpoints:
    /items - Main app
    /subapi/items - Sub-app
"""
```

## ❓ Interview Questions

### Q1: What is the difference between Path, Query, and Body parameters?

**Answer**:

- **Path**: Part of URL path (`/items/{item_id}`)
- **Query**: After `?` in URL (`/items?skip=0&limit=10`)
- **Body**: In request body (JSON for POST/PUT)
- **FastAPI determines** based on parameter location

### Q2: How does FastAPI generate API documentation?

**Answer**:

- **OpenAPI schema** generated from type hints
- **Swagger UI** at `/docs`
- **ReDoc** at `/redoc`
- **Pydantic models** define request/response schemas

### Q3: How to organize large FastAPI applications?

**Answer**:

1. **APIRouter** for modular routes
2. **Tags** for grouping endpoints
3. **Separate files** for each resource (items.py, users.py)
4. **Include routers** in main app
5. **Sub-applications** for complex apps

### Q4: What is response_model and why use it?

**Answer**:

- **Filters** response fields (exclude sensitive data)
- **Validates** response data structure
- **Documents** response schema
- **Type safety** for IDE autocomplete

## 📚 Summary

**Key Takeaways**:

1. **Path operations** with HTTP method decorators
2. **Path parameters** in URL path with type validation
3. **Query parameters** after `?` with defaults
4. **Request body** with Pydantic models
5. **Response model** filters and validates output
6. **Tags** organize endpoints in docs
7. **APIRouter** for modular applications
8. **Automatic validation** based on type hints
9. **OpenAPI docs** generated automatically
10. **FastAPI is declarative** - type hints drive behavior

FastAPI routing is simple yet powerful with automatic validation!
