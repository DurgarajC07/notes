# FastAPI Core: Pydantic Models

## 📖 Introduction

**Pydantic** is the foundation of FastAPI's data validation and serialization. Pydantic models provide automatic validation, parsing, and documentation generation using Python type hints.

## 🔑 Basic Pydantic Models

### Simple Model

```python
from pydantic import BaseModel
from typing import Optional

class Item(BaseModel):
    """Basic Pydantic model"""
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None

# Usage
item = Item(name="Laptop", price=999.99)
print(item.name)  # "Laptop"
print(item.dict())  # {"name": "Laptop", "description": None, "price": 999.99, "tax": None}
print(item.json())  # '{"name": "Laptop", "description": null, "price": 999.99, "tax": null}'

# Validation
try:
    invalid_item = Item(name="Phone")  # Missing required field 'price'
except Exception as e:
    print(f"Validation error: {e}")
```

### Model with Defaults

```python
from datetime import datetime
from typing import List

class Product(BaseModel):
    """Model with default values"""
    name: str
    sku: str
    price: float
    in_stock: bool = True
    tags: List[str] = []
    created_at: datetime = datetime.utcnow()

# Usage
product = Product(name="Mouse", sku="MSE-001", price=29.99)
print(product.in_stock)  # True
print(product.tags)  # []
```

## ✅ Field Validation

### Field with Constraints

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    """User model with field constraints"""
    username: str = Field(
        ...,  # Required
        min_length=3,
        max_length=50,
        regex="^[a-zA-Z0-9_]+$",
        description="Username (3-50 alphanumeric characters)"
    )
    email: str = Field(
        ...,
        regex=r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$",
        description="Valid email address"
    )
    age: int = Field(..., ge=0, le=150, description="Age in years")
    salary: float = Field(None, gt=0, description="Annual salary (positive)")

    class Config:
        schema_extra = {
            "example": {
                "username": "john_doe",
                "email": "john@example.com",
                "age": 30,
                "salary": 50000.0
            }
        }

# Validation
try:
    user = User(
        username="ab",  # Too short
        email="invalid",
        age=200  # Too old
    )
except Exception as e:
    print(f"Validation errors: {e}")
```

### Numeric Constraints

```python
class Product(BaseModel):
    """Product with numeric validation"""
    name: str
    price: float = Field(..., gt=0, description="Price must be positive")
    quantity: int = Field(..., ge=0, description="Quantity must be non-negative")
    discount: float = Field(0, ge=0, le=100, description="Discount percentage (0-100)")
    rating: float = Field(None, ge=0, le=5, description="Rating (0-5 stars)")

# Valid product
product = Product(
    name="Keyboard",
    price=79.99,
    quantity=10,
    discount=15.5,
    rating=4.5
)
```

### String Constraints

```python
class Article(BaseModel):
    """Article with string validation"""
    title: str = Field(..., min_length=5, max_length=200)
    content: str = Field(..., min_length=100, max_length=10000)
    slug: str = Field(..., regex="^[a-z0-9-]+$")
    tags: List[str] = Field([], max_items=10)
```

## 🔄 Custom Validators

### Field Validator

```python
from pydantic import validator

class User(BaseModel):
    """User with custom validators"""
    username: str
    email: str
    password: str
    password_confirm: str

    @validator('username')
    def username_alphanumeric(cls, v):
        """Validate username is alphanumeric"""
        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')
        return v

    @validator('email')
    def email_must_be_valid(cls, v):
        """Validate email format"""
        if '@' not in v or '.' not in v.split('@')[1]:
            raise ValueError('Invalid email format')
        return v.lower()  # Normalize to lowercase

    @validator('password')
    def password_strength(cls, v):
        """Validate password strength"""
        if len(v) < 8:
            raise ValueError('Password must be at least 8 characters')
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        return v

    @validator('password_confirm')
    def passwords_match(cls, v, values):
        """Validate passwords match"""
        if 'password' in values and v != values['password']:
            raise ValueError('Passwords do not match')
        return v

# Usage
user = User(
    username="johndoe",
    email="JOHN@EXAMPLE.COM",  # Will be lowercased
    password="SecurePass123",
    password_confirm="SecurePass123"
)
print(user.email)  # "john@example.com"
```

### Root Validator

```python
from pydantic import root_validator

class DateRange(BaseModel):
    """Date range with validation"""
    start_date: datetime
    end_date: datetime

    @root_validator
    def check_dates(cls, values):
        """Validate end_date is after start_date"""
        start = values.get('start_date')
        end = values.get('end_date')

        if start and end and end <= start:
            raise ValueError('end_date must be after start_date')

        return values

class DiscountedProduct(BaseModel):
    """Product with discount validation"""
    name: str
    price: float
    discount_percentage: float = 0

    @root_validator
    def calculate_final_price(cls, values):
        """Calculate final price with discount"""
        price = values.get('price')
        discount = values.get('discount_percentage', 0)

        if price and discount:
            values['final_price'] = price * (1 - discount / 100)

        return values
```

## 🏗️ Nested Models

### Simple Nesting

```python
class Address(BaseModel):
    """Address model"""
    street: str
    city: str
    state: str
    zip_code: str
    country: str = "USA"

class Customer(BaseModel):
    """Customer with nested address"""
    name: str
    email: str
    address: Address

# Usage
customer = Customer(
    name="John Doe",
    email="john@example.com",
    address={
        "street": "123 Main St",
        "city": "New York",
        "state": "NY",
        "zip_code": "10001"
    }
)
print(customer.address.city)  # "New York"
```

### Complex Nesting

```python
class Image(BaseModel):
    """Image model"""
    url: str
    width: int
    height: int

class Tag(BaseModel):
    """Tag model"""
    name: str
    color: str = "#000000"

class Post(BaseModel):
    """Blog post with nested models"""
    title: str
    content: str
    author: Customer  # Nested model
    images: List[Image] = []  # List of nested models
    tags: List[Tag] = []
    metadata: dict = {}  # Arbitrary dict

    class Config:
        schema_extra = {
            "example": {
                "title": "FastAPI Tutorial",
                "content": "Learn FastAPI...",
                "author": {
                    "name": "John Doe",
                    "email": "john@example.com",
                    "address": {
                        "street": "123 Main St",
                        "city": "New York",
                        "state": "NY",
                        "zip_code": "10001"
                    }
                },
                "images": [
                    {"url": "https://example.com/img1.jpg", "width": 800, "height": 600}
                ],
                "tags": [
                    {"name": "python", "color": "#3776ab"}
                ]
            }
        }
```

## 🔀 Model Inheritance

### Base Model Inheritance

```python
class TimestampMixin(BaseModel):
    """Timestamp mixin"""
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)

class BaseEntity(TimestampMixin):
    """Base entity with ID"""
    id: Optional[int] = None

class User(BaseEntity):
    """User inherits id, created_at, updated_at"""
    username: str
    email: str

class Post(BaseEntity):
    """Post inherits id, created_at, updated_at"""
    title: str
    content: str
    author_id: int

# Usage
user = User(username="john", email="john@example.com")
print(user.created_at)  # Auto-generated timestamp
```

## 🔄 Model Config

### Advanced Configuration

```python
class User(BaseModel):
    """User with advanced config"""
    id: int
    username: str
    email: str
    is_active: bool = True

    class Config:
        # Validate on assignment (default: False)
        validate_assignment = True

        # Use enum values instead of enum objects
        use_enum_values = True

        # Allow population by field name or alias
        allow_population_by_field_name = True

        # Extra fields behavior: 'forbid', 'allow', 'ignore'
        extra = 'forbid'

        # Arbitrary types allowed (for custom classes)
        arbitrary_types_allowed = False

        # Field aliases
        fields = {
            'username': {'alias': 'userName'},
            'is_active': {'alias': 'isActive'}
        }

        # Example for docs
        schema_extra = {
            "example": {
                "id": 1,
                "userName": "john_doe",
                "email": "john@example.com",
                "isActive": True
            }
        }

# Usage with aliases
user_data = {
    "id": 1,
    "userName": "john_doe",  # Using alias
    "email": "john@example.com"
}
user = User(**user_data)
print(user.username)  # "john_doe"
```

### ORM Mode

```python
class User(BaseModel):
    """User model compatible with ORM"""
    id: int
    username: str
    email: str

    class Config:
        # Allow reading data from ORM models
        orm_mode = True

# Works with SQLAlchemy models
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class UserORM(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    username = Column(String)
    email = Column(String)

# Convert ORM object to Pydantic
user_orm = UserORM(id=1, username="john", email="john@example.com")
user_pydantic = User.from_orm(user_orm)
print(user_pydantic.dict())
```

## 📤 Serialization

### Model to Dict/JSON

```python
class Product(BaseModel):
    """Product model"""
    name: str
    price: float
    tags: List[str] = []

product = Product(name="Laptop", price=999.99, tags=["electronics", "computers"])

# To dict
product_dict = product.dict()
print(product_dict)  # {"name": "Laptop", "price": 999.99, "tags": ["electronics", "computers"]}

# To dict (exclude None)
product_dict = product.dict(exclude_none=True)

# To dict (exclude fields)
product_dict = product.dict(exclude={'tags'})

# To dict (include only)
product_dict = product.dict(include={'name', 'price'})

# To JSON string
product_json = product.json()
print(product_json)  # '{"name": "Laptop", "price": 999.99, "tags": ["electronics", "computers"]}'

# To JSON (pretty)
product_json = product.json(indent=2)
```

### Parse from Dict/JSON

```python
# From dict
product_dict = {"name": "Mouse", "price": 29.99}
product = Product(**product_dict)

# From JSON string
product_json = '{"name": "Keyboard", "price": 79.99}'
product = Product.parse_raw(product_json)

# From file
product = Product.parse_file('product.json')
```

## 🎯 FastAPI Integration

### Request Body Model

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

class CreateUserRequest(BaseModel):
    """User creation request"""
    username: str = Field(..., min_length=3, max_length=50)
    email: str
    password: str = Field(..., min_length=8)
    full_name: Optional[str] = None

class UserResponse(BaseModel):
    """User response (no password)"""
    id: int
    username: str
    email: str
    full_name: Optional[str] = None

    class Config:
        orm_mode = True

@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(user: CreateUserRequest):
    """
    Create user

    - Request validated by CreateUserRequest
    - Response filtered by UserResponse
    """
    # Save to database (simulated)
    user_data = user.dict()
    user_data['id'] = 123

    return UserResponse(**user_data)
```

## ❓ Interview Questions

### Q1: What is the difference between Field(...) and Field(default)?

**Answer**:

- **Field(...)**: Required field (ellipsis means required)
- **Field(default)**: Optional with default value
- **Field(None)**: Optional, defaults to None

### Q2: When to use validator vs root_validator?

**Answer**:

- **@validator**: Validate single field
- **@root_validator**: Validate multiple fields together or entire model
- **root_validator** has access to all field values

### Q3: What is orm_mode in Pydantic Config?

**Answer**:

- Allows reading data from ORM objects (SQLAlchemy, etc.)
- Uses **from_orm()** method
- Accesses attributes, not just dict keys
- Essential for Django/SQLAlchemy integration

### Q4: How does Pydantic handle nested models?

**Answer**:

- Nested models validated recursively
- Can pass dict or Pydantic model instance
- Validates structure and types at all levels
- Serializes to nested dict/JSON

## 📚 Summary

**Key Takeaways**:

1. **BaseModel** is the base for all Pydantic models
2. **Field()** adds constraints and metadata
3. **@validator** for custom field validation
4. **@root_validator** for cross-field validation
5. **Nested models** for complex structures
6. **Inheritance** for reusable model components
7. **Config class** for advanced behavior
8. **orm_mode** for ORM integration
9. **dict() / json()** for serialization
10. **Automatic validation** with type hints

Pydantic makes data validation declarative and type-safe!
