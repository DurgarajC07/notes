# 🔑 JWT Authentication in FastAPI

## Overview

JSON Web Tokens (JWT) provide a stateless, scalable authentication mechanism for APIs. FastAPI has excellent support for JWT-based authentication through its security utilities and dependency injection system.

---

## JWT Basics

### What is JWT?

A JWT consists of three parts separated by dots:

```
header.payload.signature
```

Example:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Structure:**

- **Header**: Algorithm and token type
- **Payload**: Claims (user data)
- **Signature**: Verification signature

---

## Basic JWT Implementation

### 1. **Setup and Dependencies**

```bash
pip install python-jose[cryptography] passlib[bcrypt] python-multipart
```

### 2. **JWT Configuration**

```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel

# Configuration
SECRET_KEY = "your-secret-key-keep-it-secret-and-random"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Pydantic models
class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: Optional[str] = None
    user_id: Optional[int] = None

class UserInDB(BaseModel):
    id: int
    username: str
    email: str
    hashed_password: str
    disabled: bool = False
```

### 3. **Password Hashing**

```python
def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password against hash"""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """Hash password"""
    return pwd_context.hash(password)

# Usage
hashed = get_password_hash("mysecretpassword")
print(hashed)  # $2b$12$...

is_valid = verify_password("mysecretpassword", hashed)
print(is_valid)  # True
```

### 4. **Create JWT Token**

```python
def create_access_token(
    data: dict,
    expires_delta: Optional[timedelta] = None
) -> str:
    """Create JWT access token"""
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)

    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

    return encoded_jwt

# Usage
token = create_access_token(
    data={"sub": "john@example.com", "user_id": 123},
    expires_delta=timedelta(hours=1)
)
print(token)
```

### 5. **Verify JWT Token**

```python
from fastapi import HTTPException, status

def decode_access_token(token: str) -> TokenData:
    """Decode and verify JWT token"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        user_id: int = payload.get("user_id")

        if username is None:
            raise credentials_exception

        token_data = TokenData(username=username, user_id=user_id)
        return token_data

    except JWTError:
        raise credentials_exception

# Usage
try:
    token_data = decode_access_token(token)
    print(f"User: {token_data.username}, ID: {token_data.user_id}")
except HTTPException as e:
    print(f"Invalid token: {e.detail}")
```

---

## Complete Authentication System

### 1. **User Authentication**

```python
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Simulated database
fake_users_db = {
    "john@example.com": {
        "id": 1,
        "username": "john",
        "email": "john@example.com",
        "hashed_password": get_password_hash("secret123"),
        "disabled": False,
    }
}

async def get_user(email: str) -> Optional[UserInDB]:
    """Get user from database"""
    if email in fake_users_db:
        user_dict = fake_users_db[email]
        return UserInDB(**user_dict)
    return None

async def authenticate_user(email: str, password: str) -> Optional[UserInDB]:
    """Authenticate user credentials"""
    user = await get_user(email)
    if not user:
        return None
    if not verify_password(password, user.hashed_password):
        return None
    return user

async def get_current_user(
    token: str = Depends(oauth2_scheme)
) -> UserInDB:
    """Get current authenticated user from token"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        email: str = payload.get("sub")
        if email is None:
            raise credentials_exception
        token_data = TokenData(username=email)
    except JWTError:
        raise credentials_exception

    user = await get_user(email=token_data.username)
    if user is None:
        raise credentials_exception

    return user

async def get_current_active_user(
    current_user: UserInDB = Depends(get_current_user)
) -> UserInDB:
    """Ensure user is active"""
    if current_user.disabled:
        raise HTTPException(400, "Inactive user")
    return current_user
```

### 2. **Login Endpoint**

```python
@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login endpoint - returns JWT token"""
    user = await authenticate_user(form_data.username, form_data.password)

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.email, "user_id": user.id},
        expires_delta=access_token_expires
    )

    return {
        "access_token": access_token,
        "token_type": "bearer"
    }
```

### 3. **Protected Endpoints**

```python
@app.get("/users/me")
async def read_users_me(
    current_user: UserInDB = Depends(get_current_active_user)
):
    """Get current user profile"""
    return {
        "id": current_user.id,
        "username": current_user.username,
        "email": current_user.email
    }

@app.get("/users/me/items")
async def read_own_items(
    current_user: UserInDB = Depends(get_current_active_user)
):
    """Get items belonging to current user"""
    return [
        {"item_id": 1, "owner": current_user.username},
        {"item_id": 2, "owner": current_user.username}
    ]
```

---

## Advanced JWT Features

### 1. **Refresh Tokens**

```python
REFRESH_TOKEN_EXPIRE_DAYS = 7

def create_refresh_token(data: dict) -> str:
    """Create long-lived refresh token"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode.update({"exp": expire, "type": "refresh"})

    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

@app.post("/token/refresh")
async def refresh_access_token(refresh_token: str):
    """Get new access token using refresh token"""
    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])

        if payload.get("type") != "refresh":
            raise HTTPException(400, "Invalid token type")

        email: str = payload.get("sub")
        user_id: int = payload.get("user_id")

        # Verify user still exists
        user = await get_user(email)
        if not user:
            raise HTTPException(401, "User not found")

        # Create new access token
        new_access_token = create_access_token(
            data={"sub": email, "user_id": user_id}
        )

        return {
            "access_token": new_access_token,
            "token_type": "bearer"
        }

    except JWTError:
        raise HTTPException(401, "Invalid refresh token")

@app.post("/login/extended", response_model=dict)
async def login_with_refresh(
    form_data: OAuth2PasswordRequestForm = Depends()
):
    """Login endpoint that returns both access and refresh tokens"""
    user = await authenticate_user(form_data.username, form_data.password)

    if not user:
        raise HTTPException(401, "Incorrect email or password")

    access_token = create_access_token(
        data={"sub": user.email, "user_id": user.id}
    )
    refresh_token = create_refresh_token(
        data={"sub": user.email, "user_id": user.id}
    )

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }
```

### 2. **Token Claims and Permissions**

```python
def create_token_with_claims(
    user_id: int,
    email: str,
    roles: list,
    permissions: list
) -> str:
    """Create token with custom claims"""
    data = {
        "sub": email,
        "user_id": user_id,
        "roles": roles,
        "permissions": permissions,
        "iat": datetime.utcnow(),  # Issued at
    }

    return create_access_token(data)

async def get_current_user_with_permissions(
    token: str = Depends(oauth2_scheme)
) -> dict:
    """Extract user with roles and permissions"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

        return {
            "user_id": payload.get("user_id"),
            "email": payload.get("sub"),
            "roles": payload.get("roles", []),
            "permissions": payload.get("permissions", [])
        }
    except JWTError:
        raise HTTPException(401, "Invalid token")

def require_permission(required_permission: str):
    """Factory for permission checking"""
    async def check_permission(
        user: dict = Depends(get_current_user_with_permissions)
    ):
        if required_permission not in user.get("permissions", []):
            raise HTTPException(403, f"Permission '{required_permission}' required")
        return user
    return check_permission

# Usage
@app.delete("/posts/{post_id}")
async def delete_post(
    post_id: int,
    user: dict = Depends(require_permission("delete:posts"))
):
    return {"deleted": post_id, "by": user["email"]}
```

### 3. **Token Blacklisting**

```python
import aioredis
from typing import Set

# In-memory blacklist (use Redis in production)
blacklisted_tokens: Set[str] = set()

async def get_redis():
    """Get Redis connection"""
    return await aioredis.create_redis_pool("redis://localhost")

async def blacklist_token(token: str, redis = Depends(get_redis)):
    """Add token to blacklist"""
    # Decode to get expiration
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        exp = payload.get("exp")

        # Calculate TTL
        ttl = exp - datetime.utcnow().timestamp()

        if ttl > 0:
            # Store in Redis with expiration
            await redis.setex(
                f"blacklist:{token}",
                int(ttl),
                "1"
            )
    except JWTError:
        pass

async def is_token_blacklisted(
    token: str,
    redis = Depends(get_redis)
) -> bool:
    """Check if token is blacklisted"""
    exists = await redis.exists(f"blacklist:{token}")
    return exists > 0

async def get_current_user_with_blacklist_check(
    token: str = Depends(oauth2_scheme),
    redis = Depends(get_redis)
) -> UserInDB:
    """Verify token and check blacklist"""
    # Check blacklist
    if await is_token_blacklisted(token, redis):
        raise HTTPException(401, "Token has been revoked")

    # Standard verification
    return await get_current_user(token)

@app.post("/logout")
async def logout(
    token: str = Depends(oauth2_scheme),
    redis = Depends(get_redis)
):
    """Logout by blacklisting token"""
    await blacklist_token(token, redis)
    return {"message": "Successfully logged out"}
```

---

## JWT Security Best Practices

### 1. **Secure Token Storage**

```python
from fastapi.responses import JSONResponse

@app.post("/login/secure")
async def login_secure(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login with HttpOnly cookie"""
    user = await authenticate_user(form_data.username, form_data.password)

    if not user:
        raise HTTPException(401, "Incorrect credentials")

    access_token = create_access_token(
        data={"sub": user.email, "user_id": user.id}
    )

    response = JSONResponse(content={"message": "Login successful"})

    # Set HttpOnly cookie
    response.set_cookie(
        key="access_token",
        value=access_token,
        httponly=True,      # Not accessible via JavaScript
        secure=True,        # HTTPS only
        samesite="lax",     # CSRF protection
        max_age=1800        # 30 minutes
    )

    return response

from fastapi import Cookie

async def get_current_user_from_cookie(
    access_token: str = Cookie(None)
) -> UserInDB:
    """Get user from HttpOnly cookie"""
    if not access_token:
        raise HTTPException(401, "Not authenticated")

    return await get_current_user(access_token)
```

### 2. **Token Rotation**

```python
from datetime import datetime

async def rotate_token_if_needed(
    current_token: str,
    user: UserInDB
) -> Optional[str]:
    """Rotate token if close to expiration"""
    try:
        payload = jwt.decode(current_token, SECRET_KEY, algorithms=[ALGORITHM])
        exp = datetime.fromtimestamp(payload.get("exp"))

        # If token expires in less than 5 minutes, rotate
        if (exp - datetime.utcnow()).total_seconds() < 300:
            new_token = create_access_token(
                data={"sub": user.email, "user_id": user.id}
            )
            return new_token

    except JWTError:
        pass

    return None

@app.get("/protected")
async def protected_endpoint(
    current_token: str = Depends(oauth2_scheme),
    current_user: UserInDB = Depends(get_current_active_user)
):
    """Endpoint with automatic token rotation"""
    new_token = await rotate_token_if_needed(current_token, current_user)

    response = {"user": current_user.username, "data": "protected data"}

    if new_token:
        response["new_token"] = new_token

    return response
```

### 3. **Multi-Device Management**

```python
import uuid

def create_token_with_jti(data: dict) -> tuple[str, str]:
    """Create token with unique JTI (JWT ID)"""
    jti = str(uuid.uuid4())
    data["jti"] = jti

    token = create_access_token(data)
    return token, jti

async def store_active_token(
    user_id: int,
    jti: str,
    device_info: str,
    redis = Depends(get_redis)
):
    """Store active token info"""
    key = f"user:{user_id}:tokens"
    value = {
        "jti": jti,
        "device": device_info,
        "created_at": datetime.utcnow().isoformat()
    }

    await redis.lpush(key, json.dumps(value))
    await redis.ltrim(key, 0, 4)  # Keep last 5 devices

@app.post("/login/multi-device")
async def login_multi_device(
    form_data: OAuth2PasswordRequestForm = Depends(),
    user_agent: str = Header(None),
    redis = Depends(get_redis)
):
    """Login with device tracking"""
    user = await authenticate_user(form_data.username, form_data.password)

    if not user:
        raise HTTPException(401, "Incorrect credentials")

    # Create token with JTI
    token, jti = create_token_with_jti({
        "sub": user.email,
        "user_id": user.id
    })

    # Store active token
    await store_active_token(user.id, jti, user_agent, redis)

    return {
        "access_token": token,
        "token_type": "bearer",
        "jti": jti
    }

@app.get("/sessions")
async def list_active_sessions(
    current_user: UserInDB = Depends(get_current_active_user),
    redis = Depends(get_redis)
):
    """List all active sessions for current user"""
    key = f"user:{current_user.id}:tokens"
    sessions = await redis.lrange(key, 0, -1)

    return {
        "active_sessions": [
            json.loads(session) for session in sessions
        ]
    }

@app.post("/logout/device")
async def logout_device(
    jti: str,
    current_user: UserInDB = Depends(get_current_active_user),
    redis = Depends(get_redis)
):
    """Logout specific device by revoking JTI"""
    await redis.sadd(f"revoked_jti", jti)
    return {"message": "Device logged out"}
```

---

## Testing JWT Authentication

### 1. **Test Token Creation**

```python
import pytest

def test_create_access_token():
    """Test token creation"""
    data = {"sub": "test@example.com", "user_id": 1}
    token = create_access_token(data)

    assert isinstance(token, str)
    assert len(token) > 0

    # Decode and verify
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    assert payload["sub"] == "test@example.com"
    assert payload["user_id"] == 1
    assert "exp" in payload

def test_token_expiration():
    """Test token expiration"""
    data = {"sub": "test@example.com"}

    # Create token that expires immediately
    token = create_access_token(
        data,
        expires_delta=timedelta(seconds=-1)
    )

    # Should fail verification
    with pytest.raises(JWTError):
        jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
```

### 2. **Test Protected Endpoints**

```python
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_protected_endpoint_without_token():
    """Test accessing protected endpoint without token"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/users/me")
        assert response.status_code == 401

@pytest.mark.asyncio
async def test_protected_endpoint_with_token():
    """Test accessing protected endpoint with valid token"""
    # Create test token
    token = create_access_token(
        data={"sub": "john@example.com", "user_id": 1}
    )

    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get(
            "/users/me",
            headers={"Authorization": f"Bearer {token}"}
        )
        assert response.status_code == 200
        data = response.json()
        assert data["email"] == "john@example.com"

@pytest.mark.asyncio
async def test_login_endpoint():
    """Test login endpoint"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/token",
            data={
                "username": "john@example.com",
                "password": "secret123"
            }
        )
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert data["token_type"] == "bearer"
```

---

## Common Pitfalls and Solutions

### ❌ **Problem 1: Storing Sensitive Data in JWT**

```python
# BAD: Storing password in token
token = create_access_token({
    "sub": user.email,
    "password": user.password  # Never do this!
})
```

✅ **Solution:**

```python
# GOOD: Only store non-sensitive identifiers
token = create_access_token({
    "sub": user.email,
    "user_id": user.id
})
```

### ❌ **Problem 2: Not Checking Token Expiration**

```python
# BAD: Ignoring expiration
payload = jwt.decode(
    token,
    SECRET_KEY,
    algorithms=[ALGORITHM],
    options={"verify_exp": False}  # Don't do this!
)
```

✅ **Solution:**

```python
# GOOD: Always verify expiration
payload = jwt.decode(
    token,
    SECRET_KEY,
    algorithms=[ALGORITHM]
)
```

### ❌ **Problem 3: Weak Secret Key**

```python
# BAD: Weak or hardcoded key
SECRET_KEY = "secret"
```

✅ **Solution:**

```python
# GOOD: Strong, random key from environment
import secrets
SECRET_KEY = os.getenv("SECRET_KEY") or secrets.token_urlsafe(32)
```

---

## Interview Questions

### Q1: What are the advantages of JWT over session-based authentication?

**Answer**:

- **Stateless**: Server doesn't need to store sessions
- **Scalable**: Works across multiple servers without shared state
- **Cross-domain**: Can be used across different domains
- **Mobile-friendly**: Easy to use in mobile apps
- **Decoupled**: Auth server separate from resource server

### Q2: How do you invalidate a JWT token?

**Answer**: JWTs are stateless, so you can't truly invalidate them. Solutions:

1. **Token blacklisting**: Store revoked tokens in Redis
2. **Short expiration**: Use short-lived tokens + refresh tokens
3. **Versioning**: Include version number in token, increment on logout
4. **JTI tracking**: Track unique token IDs

### Q3: Where should JWT tokens be stored in the frontend?

**Answer**:

- **Best**: HttpOnly cookies (not accessible to JavaScript, prevents XSS)
- **Alternative**: Memory (lost on refresh)
- **Avoid**: localStorage or sessionStorage (vulnerable to XSS)

### Q4: What's the difference between access and refresh tokens?

**Answer**:

- **Access Token**: Short-lived (15-30 min), used for API requests
- **Refresh Token**: Long-lived (days/weeks), used to get new access tokens
- This pattern limits exposure if access token is compromised

### Q5: How do you implement logout with JWT?

**Answer**: Since JWTs are stateless:

1. Client discards token (basic logout)
2. Add token to blacklist in Redis (server-side logout)
3. Revoke refresh token in database
4. Clear HttpOnly cookies on logout endpoint

---

## Best Practices Summary

### ✅ Do's:

1. **Use HTTPS** in production
2. **Short expiration** for access tokens (15-30 min)
3. **Strong secret key** (min 256 bits)
4. **Validate expiration** on every request
5. **Use refresh tokens** for long sessions
6. **Store in HttpOnly cookies** when possible
7. **Implement token rotation**
8. **Use standard claims** (sub, exp, iat, jti)

### ❌ Don'ts:

1. **Don't store sensitive data** in JWT
2. **Don't use weak algorithms** (avoid "none")
3. **Don't ignore expiration**
4. **Don't share secret keys**
5. **Don't store in localStorage**
6. **Don't forget to validate** on server
7. **Don't make tokens too long-lived**

---

## Summary

JWT authentication in FastAPI provides:

- **Stateless authentication** for scalable APIs
- **Built-in security utilities** (OAuth2PasswordBearer)
- **Flexible claims** for roles and permissions
- **Easy integration** with dependency injection
- **Production-ready** patterns for token management

Properly implemented JWT authentication ensures your FastAPI application is secure, scalable, and maintainable! 🔑
