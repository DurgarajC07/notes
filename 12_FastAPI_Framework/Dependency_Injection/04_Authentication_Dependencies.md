# 🔐 Authentication Dependencies

## Overview

Authentication dependencies in FastAPI enable secure, reusable authentication logic across your API endpoints. FastAPI provides built-in security utilities and flexible dependency injection for implementing various authentication schemes.

---

## Basic Authentication

### 1. **HTTP Basic Authentication**

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBasic, HTTPBasicCredentials
import secrets

security = HTTPBasic()

def verify_basic_auth(
    credentials: HTTPBasicCredentials = Depends(security)
) -> str:
    """Verify HTTP Basic authentication"""
    correct_username = secrets.compare_digest(
        credentials.username, "admin"
    )
    correct_password = secrets.compare_digest(
        credentials.password, "secret123"
    )

    if not (correct_username and correct_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Basic"},
        )

    return credentials.username

@app.get("/admin")
async def admin_panel(username: str = Depends(verify_basic_auth)):
    return {"message": f"Welcome {username}"}
```

### 2. **Basic Auth with Database Lookup**

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

async def verify_user(
    credentials: HTTPBasicCredentials = Depends(security),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Verify credentials against database"""
    result = await db.execute(
        select(User).where(User.username == credentials.username)
    )
    user = result.scalar_one_or_none()

    if not user or not pwd_context.verify(
        credentials.password,
        user.hashed_password
    ):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Basic"},
        )

    return user

@app.get("/profile")
async def get_profile(user: User = Depends(verify_user)):
    return {"username": user.username, "email": user.email}
```

---

## Bearer Token Authentication

### 1. **Simple Bearer Token**

```python
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

bearer_scheme = HTTPBearer()

# In-memory token store (use Redis in production)
valid_tokens = {
    "secret-token-123": {"user_id": 1, "username": "john"},
    "secret-token-456": {"user_id": 2, "username": "jane"}
}

def verify_bearer_token(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme)
) -> dict:
    """Verify bearer token"""
    token = credentials.credentials

    user = valid_tokens.get(token)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token"
        )

    return user

@app.get("/me")
async def get_current_user(user: dict = Depends(verify_bearer_token)):
    return user
```

### 2. **JWT Bearer Authentication**

```python
from jose import JWTError, jwt
from datetime import datetime, timedelta
from pydantic import BaseModel

# JWT Configuration
SECRET_KEY = "your-secret-key-keep-it-secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

class TokenData(BaseModel):
    username: str
    user_id: int
    exp: datetime

def create_access_token(data: dict) -> str:
    """Create JWT access token"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})

    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user_jwt(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Verify JWT token and return user"""
    token = credentials.credentials

    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        user_id: int = payload.get("user_id")

        if username is None or user_id is None:
            raise credentials_exception

        token_data = TokenData(
            username=username,
            user_id=user_id,
            exp=datetime.fromtimestamp(payload.get("exp"))
        )

    except JWTError:
        raise credentials_exception

    # Fetch user from database
    result = await db.execute(
        select(User).where(User.id == token_data.user_id)
    )
    user = result.scalar_one_or_none()

    if user is None:
        raise credentials_exception

    return user

# Login endpoint
@app.post("/login")
async def login(
    credentials: HTTPBasicCredentials,
    db: AsyncSession = Depends(get_db)
):
    """Login and return JWT token"""
    # Verify credentials
    user = await verify_user(credentials, db)

    # Create token
    access_token = create_access_token({
        "sub": user.username,
        "user_id": user.id
    })

    return {
        "access_token": access_token,
        "token_type": "bearer",
        "expires_in": ACCESS_TOKEN_EXPIRE_MINUTES * 60
    }

# Protected endpoint
@app.get("/protected")
async def protected_route(user: User = Depends(get_current_user_jwt)):
    return {"message": f"Hello {user.username}"}
```

---

## API Key Authentication

### 1. **Header-Based API Key**

```python
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

# API keys (use database in production)
API_KEYS = {
    "api-key-123": {"client": "mobile-app", "permissions": ["read"]},
    "api-key-456": {"client": "admin-dashboard", "permissions": ["read", "write"]},
}

async def verify_api_key(
    api_key: str = Depends(api_key_header)
) -> dict:
    """Verify API key from header"""
    if not api_key:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API key required"
        )

    client = API_KEYS.get(api_key)
    if not client:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid API key"
        )

    return client

@app.get("/api/data")
async def get_data(client: dict = Depends(verify_api_key)):
    return {
        "client": client["client"],
        "data": "sensitive data"
    }
```

### 2. **Query Parameter API Key**

```python
from fastapi.security import APIKeyQuery

api_key_query = APIKeyQuery(name="api_key", auto_error=False)

async def verify_api_key_query(
    api_key: str = Depends(api_key_query),
    db: AsyncSession = Depends(get_db)
) -> dict:
    """Verify API key from query parameter"""
    if not api_key:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API key required"
        )

    # Lookup in database
    result = await db.execute(
        select(APIKey).where(
            APIKey.key == api_key,
            APIKey.is_active == True
        )
    )
    api_key_obj = result.scalar_one_or_none()

    if not api_key_obj:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or inactive API key"
        )

    # Update last used
    api_key_obj.last_used = datetime.utcnow()
    await db.commit()

    return {
        "client_id": api_key_obj.client_id,
        "permissions": api_key_obj.permissions
    }

# Usage: GET /api/data?api_key=abc123
@app.get("/api/data")
async def get_data(client: dict = Depends(verify_api_key_query)):
    return {"client_id": client["client_id"]}
```

---

## OAuth2 Authentication

### 1. **OAuth2 Password Flow**

```python
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user_oauth2(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Get current user from OAuth2 token"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    result = await db.execute(
        select(User).where(User.id == user_id)
    )
    user = result.scalar_one_or_none()

    if user is None:
        raise credentials_exception

    return user

@app.post("/token")
async def login_oauth2(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db)
):
    """OAuth2 token endpoint"""
    # Verify credentials
    result = await db.execute(
        select(User).where(User.username == form_data.username)
    )
    user = result.scalar_one_or_none()

    if not user or not pwd_context.verify(
        form_data.password,
        user.hashed_password
    ):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # Create token
    access_token = create_access_token({"sub": user.id})

    return {
        "access_token": access_token,
        "token_type": "bearer"
    }

@app.get("/users/me")
async def read_users_me(user: User = Depends(get_current_user_oauth2)):
    return user
```

### 2. **Refresh Token Pattern**

```python
from typing import Tuple

REFRESH_TOKEN_EXPIRE_DAYS = 7

def create_tokens(user_id: int) -> Tuple[str, str]:
    """Create access and refresh tokens"""
    # Access token (short-lived)
    access_token = create_access_token({
        "sub": user_id,
        "type": "access"
    })

    # Refresh token (long-lived)
    refresh_expire = datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    refresh_token = jwt.encode(
        {"sub": user_id, "type": "refresh", "exp": refresh_expire},
        SECRET_KEY,
        algorithm=ALGORITHM
    )

    return access_token, refresh_token

@app.post("/token/refresh")
async def refresh_access_token(
    refresh_token: str,
    db: AsyncSession = Depends(get_db)
):
    """Refresh access token using refresh token"""
    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])

        if payload.get("type") != "refresh":
            raise HTTPException(400, "Invalid token type")

        user_id = payload.get("sub")

        # Verify user still exists
        result = await db.execute(
            select(User).where(User.id == user_id)
        )
        user = result.scalar_one_or_none()

        if not user:
            raise HTTPException(401, "User not found")

        # Create new access token
        new_access_token = create_access_token({"sub": user_id, "type": "access"})

        return {
            "access_token": new_access_token,
            "token_type": "bearer"
        }

    except JWTError:
        raise HTTPException(401, "Invalid refresh token")
```

---

## Role-Based Access Control (RBAC)

### 1. **Simple Role Check**

```python
from enum import Enum

class UserRole(str, Enum):
    ADMIN = "admin"
    MODERATOR = "moderator"
    USER = "user"

def require_role(required_role: UserRole):
    """Factory function for role-based access"""
    async def role_checker(
        current_user: User = Depends(get_current_user_jwt)
    ):
        if current_user.role != required_role:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{required_role}' required"
            )
        return current_user

    return role_checker

# Create role dependencies
require_admin = require_role(UserRole.ADMIN)
require_moderator = require_role(UserRole.MODERATOR)

# Usage
@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin: User = Depends(require_admin)
):
    # Only admins can delete users
    return {"deleted": user_id}
```

### 2. **Permission-Based Access**

```python
from typing import List

class Permission(str, Enum):
    READ_USERS = "read:users"
    WRITE_USERS = "write:users"
    DELETE_USERS = "delete:users"
    READ_POSTS = "read:posts"
    WRITE_POSTS = "write:posts"

def require_permissions(required_permissions: List[Permission]):
    """Factory for permission-based access control"""
    async def permission_checker(
        current_user: User = Depends(get_current_user_jwt)
    ):
        user_permissions = set(current_user.permissions)
        required = set(p.value for p in required_permissions)

        if not required.issubset(user_permissions):
            missing = required - user_permissions
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Missing permissions: {', '.join(missing)}"
            )

        return current_user

    return permission_checker

# Usage
@app.get("/users/")
async def list_users(
    user: User = Depends(require_permissions([Permission.READ_USERS]))
):
    return {"users": [...]}

@app.post("/users/")
async def create_user(
    user_data: UserCreate,
    admin: User = Depends(require_permissions([
        Permission.WRITE_USERS
    ]))
):
    return {"created": True}

@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin: User = Depends(require_permissions([
        Permission.DELETE_USERS,
        Permission.WRITE_USERS
    ]))
):
    return {"deleted": user_id}
```

---

## Optional Authentication

### 1. **Optional User Dependency**

```python
async def get_current_user_optional(
    credentials: HTTPAuthorizationCredentials = Depends(
        HTTPBearer(auto_error=False)
    ),
    db: AsyncSession = Depends(get_db)
) -> Optional[User]:
    """Get current user if authenticated, None otherwise"""
    if not credentials:
        return None

    try:
        token = credentials.credentials
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = payload.get("sub")

        result = await db.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    except JWTError:
        return None

@app.get("/posts/")
async def list_posts(
    user: Optional[User] = Depends(get_current_user_optional)
):
    """Public endpoint with optional authentication"""
    if user:
        # Show personalized content
        return {"posts": [...], "personalized": True}
    else:
        # Show public content
        return {"posts": [...], "personalized": False}
```

---

## Security Best Practices

### 1. **Token Blacklisting**

```python
# Redis for token blacklist
import aioredis

async def get_redis():
    return await aioredis.create_redis_pool("redis://localhost")

async def verify_token_not_blacklisted(
    token: str,
    redis = Depends(get_redis)
):
    """Check if token is blacklisted"""
    is_blacklisted = await redis.exists(f"blacklist:{token}")
    if is_blacklisted:
        raise HTTPException(401, "Token has been revoked")
    return token

async def get_current_user_with_blacklist(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
    redis = Depends(get_redis),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Verify token isn't blacklisted"""
    token = credentials.credentials
    await verify_token_not_blacklisted(token, redis)

    # Continue with normal verification
    return await get_current_user_jwt(credentials, db)

@app.post("/logout")
async def logout(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
    redis = Depends(get_redis)
):
    """Logout by blacklisting token"""
    token = credentials.credentials

    # Decode to get expiration
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    exp = payload.get("exp")
    ttl = exp - datetime.utcnow().timestamp()

    # Add to blacklist with TTL
    await redis.setex(f"blacklist:{token}", int(ttl), "1")

    return {"message": "Logged out successfully"}
```

### 2. **Rate Limiting by User**

```python
from collections import defaultdict
from datetime import timedelta

user_rate_limits = defaultdict(list)

async def rate_limit_by_user(
    current_user: User = Depends(get_current_user_jwt),
    max_requests: int = 100,
    window_seconds: int = 60
):
    """Rate limit per authenticated user"""
    now = datetime.utcnow()
    user_id = current_user.id

    # Clean old requests
    cutoff = now - timedelta(seconds=window_seconds)
    user_rate_limits[user_id] = [
        req_time for req_time in user_rate_limits[user_id]
        if req_time > cutoff
    ]

    # Check limit
    if len(user_rate_limits[user_id]) >= max_requests:
        raise HTTPException(
            status_code=429,
            detail=f"Rate limit exceeded: {max_requests} requests per {window_seconds}s"
        )

    # Record request
    user_rate_limits[user_id].append(now)

    return current_user

@app.get("/api/resource")
async def get_resource(
    user: User = Depends(rate_limit_by_user)
):
    return {"data": "resource"}
```

---

## Testing Authentication

### 1. **Override Auth Dependency**

```python
@pytest.fixture
def test_user():
    return User(id=1, username="testuser", role="admin")

@pytest.mark.asyncio
async def test_protected_endpoint(test_user):
    """Test with mocked authentication"""
    async def mock_get_current_user():
        return test_user

    app.dependency_overrides[get_current_user_jwt] = mock_get_current_user

    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/protected")

        assert response.status_code == 200
        assert "testuser" in response.json()["message"]

    app.dependency_overrides.clear()
```

### 2. **Test Token Generation**

```python
@pytest.mark.asyncio
async def test_login():
    """Test login endpoint"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/login",
            auth=("admin", "secret123")
        )

        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert data["token_type"] == "bearer"

        # Test using token
        token = data["access_token"]
        protected_response = await client.get(
            "/protected",
            headers={"Authorization": f"Bearer {token}"}
        )

        assert protected_response.status_code == 200
```

---

## Best Practices

### ✅ Do's:

1. **Use HTTPS** in production
2. **Hash passwords** with bcrypt/argon2
3. **Use short-lived** access tokens
4. **Implement refresh tokens** for long sessions
5. **Validate tokens** on every request
6. **Use `secrets.compare_digest`** for comparison
7. **Implement rate limiting**
8. **Log authentication failures**

### ❌ Don'ts:

1. **Don't store passwords** in plain text
2. **Don't use weak** encryption algorithms
3. **Don't expose** sensitive error details
4. **Don't ignore** token expiration
5. **Don't share** secret keys
6. **Don't trust** client-side validation

---

## Interview Questions

### Q1: What's the difference between authentication and authorization?

**Answer**:

- **Authentication**: Verifying identity (who you are) - e.g., login with username/password
- **Authorization**: Verifying permissions (what you can do) - e.g., checking if user has admin role

### Q2: Why use JWT for authentication in APIs?

**Answer**: JWTs are stateless (server doesn't need to store sessions), can be verified without database lookup, work well in distributed systems, and can carry user information in the payload.

### Q3: How do you secure FastAPI endpoints?

**Answer**: Use security dependencies like `HTTPBearer`, `OAuth2PasswordBearer`, validate tokens in dependencies, implement RBAC, and apply dependencies at path/router/app level.

### Q4: What's the purpose of refresh tokens?

**Answer**: Refresh tokens are long-lived tokens used to obtain new short-lived access tokens without requiring user to re-authenticate, improving security (short access token lifetime) while maintaining good UX.

### Q5: How do you implement token blacklisting?

**Answer**: Store revoked tokens in Redis with TTL equal to token expiration time, check blacklist before verifying token:

```python
is_blacklisted = await redis.exists(f"blacklist:{token}")
if is_blacklisted:
    raise HTTPException(401, "Token revoked")
```

---

## Summary

Authentication dependencies in FastAPI provide:

- **Multiple auth schemes**: Basic, Bearer, OAuth2, API Keys
- **Reusable logic**: DRY authentication code
- **Easy testing**: Override dependencies
- **Flexible access control**: RBAC and permissions
- **Security**: JWT, token blacklisting, rate limiting

Proper authentication is essential for building secure FastAPI applications! 🔐
