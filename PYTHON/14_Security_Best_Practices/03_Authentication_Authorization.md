# 🔐 Authentication and Authorization

## Overview

Authentication verifies identity ("who are you?"), while authorization determines permissions ("what can you do?"). Both are critical for API security.

---

## JWT Authentication

### JWT Structure

```
header.payload.signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.     # Header
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ik.   # Payload
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  # Signature
```

### Implement JWT Authentication

```python
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

# Configuration
SECRET_KEY = "your-secret-key-keep-it-secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Password hashing
def hash_password(password: str) -> str:
    """Hash password with bcrypt"""
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password against hash"""
    return pwd_context.verify(plain_password, hashed_password)

# Token creation
def create_access_token(data: dict, expires_delta: timedelta = None):
    """Create JWT access token"""
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)

    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

    return encoded_jwt

def create_refresh_token(data: dict):
    """Create refresh token with longer expiry"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=7)
    to_encode.update({"exp": expire, "type": "refresh"})

    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

# Token verification
def verify_token(token: str) -> dict:
    """Verify and decode JWT token"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")

        if user_id is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Could not validate credentials"
            )

        return payload

    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not validate credentials",
            headers={"WWW-Authenticate": "Bearer"}
        )

# Get current user from token
async def get_current_user(token: str = Depends(oauth2_scheme)):
    """Extract user from JWT token"""
    payload = verify_token(token)
    user_id = payload.get("sub")

    # Fetch user from database
    user = await get_user_by_id(user_id)

    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found"
        )

    return user

# Login endpoint
@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login endpoint returning JWT token"""
    # Authenticate user
    user = await authenticate_user(form_data.username, form_data.password)

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"}
        )

    # Create tokens
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": str(user.id)},
        expires_delta=access_token_expires
    )

    refresh_token = create_refresh_token(
        data={"sub": str(user.id)}
    )

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }

# Protected endpoint
@app.get("/users/me")
async def read_users_me(current_user = Depends(get_current_user)):
    """Get current user information"""
    return current_user

# Refresh token endpoint
@app.post("/token/refresh")
async def refresh_access_token(refresh_token: str):
    """Get new access token using refresh token"""
    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])

        if payload.get("type") != "refresh":
            raise HTTPException(400, "Invalid token type")

        user_id = payload.get("sub")

        # Create new access token
        access_token = create_access_token(
            data={"sub": user_id},
            expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
        )

        return {"access_token": access_token, "token_type": "bearer"}

    except JWTError:
        raise HTTPException(401, "Invalid refresh token")
```

---

## Role-Based Access Control (RBAC)

### Define Roles and Permissions

```python
from enum import Enum
from typing import List

class Permission(str, Enum):
    """System permissions"""
    READ_USERS = "read:users"
    WRITE_USERS = "write:users"
    DELETE_USERS = "delete:users"
    READ_POSTS = "read:posts"
    WRITE_POSTS = "write:posts"
    DELETE_POSTS = "delete:posts"
    ADMIN = "admin:all"

class Role(str, Enum):
    """User roles"""
    GUEST = "guest"
    USER = "user"
    MODERATOR = "moderator"
    ADMIN = "admin"

# Role permissions mapping
ROLE_PERMISSIONS = {
    Role.GUEST: [
        Permission.READ_POSTS
    ],
    Role.USER: [
        Permission.READ_USERS,
        Permission.READ_POSTS,
        Permission.WRITE_POSTS
    ],
    Role.MODERATOR: [
        Permission.READ_USERS,
        Permission.READ_POSTS,
        Permission.WRITE_POSTS,
        Permission.DELETE_POSTS
    ],
    Role.ADMIN: [
        Permission.ADMIN  # Has all permissions
    ]
}

def has_permission(user_role: Role, required_permission: Permission) -> bool:
    """Check if role has permission"""
    permissions = ROLE_PERMISSIONS.get(user_role, [])

    # Admin has all permissions
    if Permission.ADMIN in permissions:
        return True

    return required_permission in permissions
```

### Permission Dependencies

```python
from fastapi import Depends, HTTPException, status

def require_permission(required_permission: Permission):
    """Dependency to check permission"""
    async def permission_checker(current_user = Depends(get_current_user)):
        if not has_permission(current_user.role, required_permission):
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Permission denied: {required_permission}"
            )
        return current_user

    return permission_checker

def require_role(required_role: Role):
    """Dependency to check role"""
    async def role_checker(current_user = Depends(get_current_user)):
        # Define role hierarchy
        role_hierarchy = {
            Role.GUEST: 0,
            Role.USER: 1,
            Role.MODERATOR: 2,
            Role.ADMIN: 3
        }

        user_level = role_hierarchy.get(current_user.role, 0)
        required_level = role_hierarchy.get(required_role, 99)

        if user_level < required_level:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Requires {required_role} role"
            )

        return current_user

    return role_checker

# Protected endpoints
@app.get("/users")
async def list_users(
    current_user = Depends(require_permission(Permission.READ_USERS))
):
    """List users - requires read:users permission"""
    users = await get_all_users()
    return users

@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    current_user = Depends(require_permission(Permission.DELETE_USERS))
):
    """Delete user - requires delete:users permission"""
    await delete_user_by_id(user_id)
    return {"message": "User deleted"}

@app.get("/admin/dashboard")
async def admin_dashboard(
    current_user = Depends(require_role(Role.ADMIN))
):
    """Admin dashboard - requires admin role"""
    stats = await get_admin_statistics()
    return stats
```

---

## Resource-Based Authorization

### Ownership Check

```python
class ResourceOwnershipChecker:
    """Check resource ownership"""

    @staticmethod
    async def check_post_ownership(
        post_id: int,
        current_user = Depends(get_current_user),
        db: Session = Depends(get_db)
    ):
        """Verify user owns the post"""
        post = db.query(Post).filter(Post.id == post_id).first()

        if not post:
            raise HTTPException(404, "Post not found")

        # Admin can access any post
        if current_user.role == Role.ADMIN:
            return post

        # Check ownership
        if post.user_id != current_user.id:
            raise HTTPException(403, "Not authorized to access this post")

        return post

@app.put("/posts/{post_id}")
async def update_post(
    post_id: int,
    post_data: PostUpdate,
    post = Depends(ResourceOwnershipChecker.check_post_ownership),
    db: Session = Depends(get_db)
):
    """Update post - only owner can update"""
    for key, value in post_data.dict(exclude_unset=True).items():
        setattr(post, key, value)

    db.commit()
    return post

@app.delete("/posts/{post_id}")
async def delete_post(
    post_id: int,
    post = Depends(ResourceOwnershipChecker.check_post_ownership),
    db: Session = Depends(get_db)
):
    """Delete post - only owner can delete"""
    db.delete(post)
    db.commit()
    return {"message": "Post deleted"}
```

---

## API Key Authentication

### API Key Management

```python
import secrets
from datetime import datetime

class APIKey(Base):
    """API Key model"""
    __tablename__ = "api_keys"

    id = Column(Integer, primary_key=True)
    key = Column(String, unique=True, index=True)
    name = Column(String)
    user_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.utcnow)
    last_used = Column(DateTime, nullable=True)
    expires_at = Column(DateTime, nullable=True)
    is_active = Column(Boolean, default=True)

    # Relationships
    user = relationship("User", back_populates="api_keys")

def generate_api_key() -> str:
    """Generate secure API key"""
    return secrets.token_urlsafe(32)

async def create_api_key(
    user_id: int,
    name: str,
    expires_days: int = None,
    db: Session = Depends(get_db)
) -> APIKey:
    """Create new API key for user"""
    key = generate_api_key()

    api_key = APIKey(
        key=key,
        name=name,
        user_id=user_id,
        expires_at=(
            datetime.utcnow() + timedelta(days=expires_days)
            if expires_days else None
        )
    )

    db.add(api_key)
    db.commit()
    db.refresh(api_key)

    return api_key

# API Key authentication
async def get_api_key(
    api_key: str = Header(..., alias="X-API-Key"),
    db: Session = Depends(get_db)
):
    """Verify API key from header"""
    key = db.query(APIKey).filter(
        APIKey.key == api_key,
        APIKey.is_active == True
    ).first()

    if not key:
        raise HTTPException(401, "Invalid API key")

    # Check expiration
    if key.expires_at and key.expires_at < datetime.utcnow():
        raise HTTPException(401, "API key expired")

    # Update last used
    key.last_used = datetime.utcnow()
    db.commit()

    return key

@app.get("/api/data")
async def get_data(api_key = Depends(get_api_key)):
    """Endpoint protected by API key"""
    return {"message": "Access granted", "user_id": api_key.user_id}
```

---

## OAuth2 with Third-Party Providers

### Google OAuth2

```python
from authlib.integrations.starlette_client import OAuth
from starlette.config import Config

# Configuration
config = Config('.env')
oauth = OAuth(config)

oauth.register(
    name='google',
    client_id=config('GOOGLE_CLIENT_ID'),
    client_secret=config('GOOGLE_CLIENT_SECRET'),
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid email profile'}
)

@app.get("/login/google")
async def login_google(request: Request):
    """Redirect to Google login"""
    redirect_uri = request.url_for('auth_google')
    return await oauth.google.authorize_redirect(request, redirect_uri)

@app.get("/auth/google")
async def auth_google(request: Request):
    """Handle Google OAuth callback"""
    token = await oauth.google.authorize_access_token(request)
    user_info = token.get('userinfo')

    if user_info:
        # Create or update user
        user = await get_or_create_user(
            email=user_info['email'],
            name=user_info.get('name'),
            google_id=user_info['sub']
        )

        # Create JWT token
        access_token = create_access_token(data={"sub": str(user.id)})

        return {"access_token": access_token, "token_type": "bearer"}

    raise HTTPException(400, "Failed to get user info")
```

---

## Session-Based Authentication

### Session Management

```python
from starlette.middleware.sessions import SessionMiddleware
import redis

# Add session middleware
app.add_middleware(
    SessionMiddleware,
    secret_key="your-secret-key",
    max_age=3600  # 1 hour
)

# Redis session store
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

class SessionManager:
    """Manage user sessions"""

    @staticmethod
    def create_session(user_id: int) -> str:
        """Create new session"""
        session_id = secrets.token_urlsafe(32)

        # Store in Redis with expiration
        redis_client.setex(
            f"session:{session_id}",
            3600,  # 1 hour
            user_id
        )

        return session_id

    @staticmethod
    def get_session(session_id: str) -> int:
        """Get user ID from session"""
        user_id = redis_client.get(f"session:{session_id}")
        return int(user_id) if user_id else None

    @staticmethod
    def delete_session(session_id: str):
        """Delete session"""
        redis_client.delete(f"session:{session_id}")

@app.post("/login-session")
async def login_session(
    credentials: LoginCredentials,
    request: Request
):
    """Login with session"""
    user = await authenticate_user(credentials.username, credentials.password)

    if not user:
        raise HTTPException(401, "Invalid credentials")

    # Create session
    session_id = SessionManager.create_session(user.id)
    request.session['session_id'] = session_id

    return {"message": "Logged in"}

@app.post("/logout-session")
async def logout_session(request: Request):
    """Logout and destroy session"""
    session_id = request.session.get('session_id')

    if session_id:
        SessionManager.delete_session(session_id)
        request.session.clear()

    return {"message": "Logged out"}
```

---

## Best Practices

### ✅ Do's:

1. **Use HTTPS** for all authentication
2. **Hash passwords** with bcrypt/argon2
3. **Use short expiration** for access tokens
4. **Implement refresh tokens**
5. **Rate limit** login attempts
6. **Log authentication events**
7. **Use strong secrets** for JWT
8. **Validate all tokens** on each request
9. **Implement account lockout**
10. **Use MFA** for sensitive operations

### ❌ Don'ts:

1. **Don't store passwords** in plain text
2. **Don't use weak secrets**
3. **Don't trust client-provided** user IDs
4. **Don't skip authorization** checks
5. **Don't expose sensitive data** in tokens
6. **Don't use predictable** API keys
7. **Don't forget to revoke** old tokens

---

## Interview Questions

### Q1: What's the difference between authentication and authorization?

**Answer**:

- **Authentication**: Verify identity (login, who are you?)
- **Authorization**: Check permissions (what can you do?)
  Example: Login is authentication, checking if user can delete post is authorization.

### Q2: How do JWT tokens work?

**Answer**: JSON Web Tokens have three parts:

- **Header**: Algorithm and type
- **Payload**: Claims (user data)
- **Signature**: Verify integrity
  Server verifies signature, no database lookup needed. Stateless.

### Q3: What are refresh tokens and why use them?

**Answer**: Long-lived tokens to get new access tokens:

- Access tokens short-lived (15-30 min)
- Refresh tokens long-lived (days/weeks)
- Reduces exposure if access token stolen
- Can revoke refresh tokens

### Q4: What is RBAC?

**Answer**: Role-Based Access Control:

- Assign users to roles
- Roles have permissions
- Check role/permission before action
- Easier to manage than per-user permissions

### Q5: How do you prevent brute force attacks on login?

**Answer**:

- Rate limiting (max attempts per IP/user)
- Account lockout after failed attempts
- CAPTCHA after several failures
- Progressive delays between attempts
- Monitor and alert on patterns

---

## Summary

Authentication & authorization essentials:

- **JWT** for stateless auth
- **Refresh tokens** for security
- **RBAC** for permissions
- **Resource ownership** checks
- **API keys** for service access
- **OAuth2** for third-party login
- **Rate limiting** for protection

Security first! 🔐
