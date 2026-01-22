# FastAPI Framework Interview Questions

## 📖 Overview

Comprehensive FastAPI interview questions for mid to senior level backend developers, covering async programming, dependency injection, performance, and production deployment.

## 🚀 FastAPI Core Concepts

### Q1: What is FastAPI and why is it gaining popularity?

**Answer**:

FastAPI is a modern, high-performance web framework for building APIs with Python 3.7+ based on standard Python type hints.

**Key Features**:

- **Fast**: On par with NodeJS and Go (thanks to Starlette and Pydantic)
- **Type hints**: Automatic data validation and serialization
- **Async support**: Native async/await for high concurrency
- **Auto documentation**: OpenAPI (Swagger) and ReDoc generated automatically
- **Standards-based**: OpenAPI, JSON Schema
- **Production-ready**: Used by Microsoft, Uber, Netflix

**Comparison with Django REST Framework**:

```python
# Django REST Framework
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated]

# FastAPI
from fastapi import FastAPI, Depends
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    id: int
    username: str
    email: str

@app.get("/users/", response_model=List[User])
async def get_users(
    current_user: User = Depends(get_current_user)
):
    return await fetch_users_from_db()
```

**Advantages**:

- Automatic data validation via Pydantic
- Built-in async support
- Faster performance
- Less boilerplate
- Type safety with IDE support

### Q2: Explain FastAPI's dependency injection system

**Answer**:

FastAPI's DI system provides a clean way to share logic, manage resources, and enforce requirements across endpoints.

**Basic dependency**:

```python
from fastapi import Depends, HTTPException

# Dependency function
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Using dependency
@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: Session = Depends(get_db)
):
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

**Class-based dependency**:

```python
class CommonQueryParams:
    def __init__(
        self,
        skip: int = 0,
        limit: int = 100,
        q: Optional[str] = None
    ):
        self.skip = skip
        self.limit = limit
        self.q = q

@app.get("/items/")
async def read_items(
    commons: CommonQueryParams = Depends()
):
    return {
        "skip": commons.skip,
        "limit": commons.limit,
        "q": commons.q
    }
```

**Dependency chain**:

```python
async def get_current_user(token: str = Depends(oauth2_scheme)):
    user = decode_token(token)
    if not user:
        raise HTTPException(status_code=401)
    return user

async def get_current_active_user(
    current_user: User = Depends(get_current_user)
):
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

@app.get("/users/me")
async def read_users_me(
    current_user: User = Depends(get_current_active_user)
):
    return current_user
```

**Benefits**:

- Reusable across endpoints
- Automatic execution order
- Type safety
- Easy testing (override dependencies)
- Clean separation of concerns

### Q3: How does FastAPI handle request validation?

**Answer**:

FastAPI uses **Pydantic** models for automatic validation, serialization, and documentation.

**Request body validation**:

```python
from pydantic import BaseModel, Field, EmailStr, validator
from typing import Optional

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(..., min_length=8)
    age: Optional[int] = Field(None, ge=0, le=150)

    @validator('username')
    def username_alphanumeric(cls, v):
        if not v.isalnum():
            raise ValueError('must be alphanumeric')
        return v

    @validator('password')
    def password_strength(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError('must contain uppercase')
        if not any(c.isdigit() for c in v):
            raise ValueError('must contain digit')
        return v

    class Config:
        schema_extra = {
            "example": {
                "username": "johndoe",
                "email": "john@example.com",
                "password": "Secret123",
                "age": 25
            }
        }

@app.post("/users/", response_model=UserResponse)
async def create_user(user: UserCreate):
    # user is automatically validated
    # Invalid data returns 422 with detailed errors
    return await save_user(user)
```

**Query parameter validation**:

```python
from fastapi import Query

@app.get("/items/")
async def read_items(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    q: Optional[str] = Query(None, min_length=3, max_length=50)
):
    return {"skip": skip, "limit": limit, "q": q}
```

**Path parameter validation**:

```python
from fastapi import Path

@app.get("/items/{item_id}")
async def read_item(
    item_id: int = Path(..., title="Item ID", ge=1)
):
    return {"item_id": item_id}
```

**Automatic error responses**:

```json
// Invalid request
{
  "detail": [
    {
      "loc": ["body", "email"],
      "msg": "value is not a valid email address",
      "type": "value_error.email"
    },
    {
      "loc": ["body", "age"],
      "msg": "ensure this value is less than or equal to 150",
      "type": "value_error.number.not_le"
    }
  ]
}
```

### Q4: Explain async/await in FastAPI

**Answer**:

FastAPI fully supports async/await for high-performance async operations.

**When to use async**:

```python
# ✅ Use async for I/O-bound operations
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # Async database query
    user = await db.fetch_one(
        "SELECT * FROM users WHERE id = :id",
        {"id": user_id}
    )
    return user

# ✅ Use regular def for CPU-bound or sync operations
@app.get("/compute")
def compute():
    # CPU-intensive work
    result = complex_calculation()
    return {"result": result}
```

**Async database operations**:

```python
from databases import Database

database = Database("postgresql://user:pass@localhost/db")

@app.on_event("startup")
async def startup():
    await database.connect()

@app.on_event("shutdown")
async def shutdown():
    await database.disconnect()

@app.get("/users/")
async def get_users():
    query = "SELECT * FROM users"
    return await database.fetch_all(query)

@app.post("/users/")
async def create_user(user: UserCreate):
    query = """
        INSERT INTO users (username, email, password)
        VALUES (:username, :email, :password)
        RETURNING id
    """
    user_id = await database.execute(
        query,
        values={
            "username": user.username,
            "email": user.email,
            "password": hash_password(user.password)
        }
    )
    return {"id": user_id}
```

**Multiple async operations**:

```python
import asyncio
import httpx

@app.get("/aggregate")
async def aggregate_data(user_id: int):
    # Run multiple async operations concurrently
    async with httpx.AsyncClient() as client:
        user_task = get_user(user_id)
        posts_task = get_user_posts(user_id)
        comments_task = get_user_comments(user_id)

        user, posts, comments = await asyncio.gather(
            user_task,
            posts_task,
            comments_task
        )

    return {
        "user": user,
        "posts": posts,
        "comments": comments
    }
```

**Performance comparison**:

```python
# Sync version (blocking)
@app.get("/sync")
def sync_endpoint():
    time.sleep(1)  # Blocks thread
    return {"status": "done"}

# Async version (non-blocking)
@app.get("/async")
async def async_endpoint():
    await asyncio.sleep(1)  # Doesn't block
    return {"status": "done"}

# Under load:
# Sync: Can handle ~100 concurrent requests
# Async: Can handle ~10,000 concurrent requests
```

### Q5: How do you handle authentication in FastAPI?

**Answer**:

FastAPI provides flexible authentication options through dependency injection.

**JWT Authentication**:

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import JWTError, jwt
from datetime import datetime, timedelta

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

security = HTTPBearer()

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    token = credentials.credentials
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = await get_user_from_db(username)
    if user is None:
        raise credentials_exception
    return user

# Login endpoint
@app.post("/login")
async def login(username: str, password: str):
    user = await authenticate_user(username, password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    access_token = create_access_token(
        data={"sub": user.username},
        expires_delta=timedelta(hours=24)
    )
    return {"access_token": access_token, "token_type": "bearer"}

# Protected endpoint
@app.get("/users/me")
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user
```

**OAuth2 with Password Flow**:

```python
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = await authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    access_token = create_access_token(data={"sub": user.username})
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/protected")
async def protected_route(token: str = Depends(oauth2_scheme)):
    return {"token": token}
```

**Role-based access control**:

```python
from enum import Enum

class Role(str, Enum):
    USER = "user"
    ADMIN = "admin"
    MODERATOR = "moderator"

def require_role(required_role: Role):
    async def role_checker(
        current_user: User = Depends(get_current_user)
    ):
        if current_user.role != required_role:
            raise HTTPException(
                status_code=403,
                detail="Insufficient permissions"
            )
        return current_user
    return role_checker

@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin: User = Depends(require_role(Role.ADMIN))
):
    await delete_user_from_db(user_id)
    return {"status": "deleted"}
```

## 🏗️ FastAPI Architecture & Design

### Q6: How do you structure a large FastAPI application?

**Answer**:

**Recommended project structure**:

```
app/
├── __init__.py
├── main.py                 # App entry point
├── config.py               # Configuration
├── dependencies.py         # Shared dependencies
│
├── api/                    # API layer
│   ├── __init__.py
│   ├── v1/
│   │   ├── __init__.py
│   │   ├── endpoints/
│   │   │   ├── users.py
│   │   │   ├── posts.py
│   │   │   └── auth.py
│   │   └── router.py
│   └── v2/
│
├── core/                   # Core functionality
│   ├── __init__.py
│   ├── security.py
│   ├── auth.py
│   └── config.py
│
├── db/                     # Database
│   ├── __init__.py
│   ├── base.py
│   ├── session.py
│   └── models/
│       ├── user.py
│       └── post.py
│
├── schemas/                # Pydantic models
│   ├── __init__.py
│   ├── user.py
│   └── post.py
│
├── services/               # Business logic
│   ├── __init__.py
│   ├── user_service.py
│   └── post_service.py
│
└── tests/
    ├── __init__.py
    ├── conftest.py
    └── test_api/
```

**main.py**:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.api.v1.router import api_router
from app.core.config import settings

app = FastAPI(
    title=settings.PROJECT_NAME,
    openapi_url=f"{settings.API_V1_STR}/openapi.json"
)

# Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(api_router, prefix=settings.API_V1_STR)

@app.on_event("startup")
async def startup_event():
    # Initialize database, cache, etc.
    await database.connect()

@app.on_event("shutdown")
async def shutdown_event():
    await database.disconnect()
```

**Router organization** (api/v1/endpoints/users.py):

```python
from fastapi import APIRouter, Depends, HTTPException
from typing import List
from app.schemas.user import User, UserCreate, UserUpdate
from app.services.user_service import UserService
from app.core.auth import get_current_user

router = APIRouter()

@router.get("/", response_model=List[User])
async def get_users(
    skip: int = 0,
    limit: int = 100,
    service: UserService = Depends()
):
    return await service.get_users(skip=skip, limit=limit)

@router.post("/", response_model=User)
async def create_user(
    user_in: UserCreate,
    service: UserService = Depends()
):
    return await service.create_user(user_in)

@router.get("/{user_id}", response_model=User)
async def get_user(
    user_id: int,
    service: UserService = Depends()
):
    user = await service.get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

**Service layer** (services/user_service.py):

```python
from typing import Optional, List
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import get_db
from app.db.models.user import User
from app.schemas.user import UserCreate

class UserService:
    def __init__(self, db: AsyncSession = Depends(get_db)):
        self.db = db

    async def get_users(
        self,
        skip: int = 0,
        limit: int = 100
    ) -> List[User]:
        query = select(User).offset(skip).limit(limit)
        result = await self.db.execute(query)
        return result.scalars().all()

    async def get_user(self, user_id: int) -> Optional[User]:
        return await self.db.get(User, user_id)

    async def create_user(self, user_in: UserCreate) -> User:
        user = User(**user_in.dict())
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        return user
```

### Q7: How do you handle errors and exceptions in FastAPI?

**Answer**:

**Custom exception handlers**:

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

app = FastAPI()

# Custom exception
class ItemNotFoundError(Exception):
    def __init__(self, item_id: int):
        self.item_id = item_id

# Exception handler
@app.exception_handler(ItemNotFoundError)
async def item_not_found_handler(
    request: Request,
    exc: ItemNotFoundError
):
    return JSONResponse(
        status_code=404,
        content={
            "error": "Item not found",
            "item_id": exc.item_id,
            "path": request.url.path
        }
    )

# Validation error handler
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(
    request: Request,
    exc: RequestValidationError
):
    return JSONResponse(
        status_code=422,
        content={
            "error": "Validation error",
            "details": exc.errors(),
            "body": exc.body
        }
    )

# Generic error handler
@app.exception_handler(Exception)
async def generic_exception_handler(
    request: Request,
    exc: Exception
):
    # Log error
    logger.error(f"Unhandled exception: {exc}", exc_info=True)

    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal server error",
            "message": "An unexpected error occurred"
        }
    )
```

**Structured error responses**:

```python
from pydantic import BaseModel
from typing import Optional, List

class ErrorDetail(BaseModel):
    field: Optional[str] = None
    message: str
    code: str

class ErrorResponse(BaseModel):
    error: str
    details: Optional[List[ErrorDetail]] = None
    request_id: Optional[str] = None

@app.post("/items/", response_model=Item, responses={
    400: {"model": ErrorResponse},
    422: {"model": ErrorResponse}
})
async def create_item(item: ItemCreate):
    try:
        return await item_service.create(item)
    except ValidationError as e:
        raise HTTPException(
            status_code=400,
            detail=ErrorResponse(
                error="Validation failed",
                details=[
                    ErrorDetail(
                        field=err["field"],
                        message=err["message"],
                        code="validation_error"
                    )
                    for err in e.errors()
                ]
            ).dict()
        )
```

## 📚 Summary

Key FastAPI interview topics:

1. **Core Concepts**: ASGI, async/await, dependency injection
2. **Validation**: Pydantic models, automatic validation
3. **Authentication**: JWT, OAuth2, role-based access
4. **Performance**: Async operations, caching, optimization
5. **Architecture**: Project structure, service layer, routers
6. **Testing**: TestClient, async tests, mocking
7. **Deployment**: Docker, Uvicorn, production settings

Prepare for:

- Code implementation questions
- Async/await understanding
- Dependency injection patterns
- Performance optimization
- Real-world API design
