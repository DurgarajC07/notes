# 🔒 OWASP Top 10 for APIs

## Overview

The OWASP API Security Top 10 represents the most critical security risks for APIs. Understanding and mitigating these vulnerabilities is essential for building secure production systems.

---

## API1:2023 - Broken Object Level Authorization (BOLA)

### What It Is

APIs expose endpoints that handle object identifiers, but fail to validate if the user has access to that specific object.

### Vulnerable Code

```python
# ❌ VULNERABLE: No authorization check
@app.get("/api/users/{user_id}/profile")
async def get_user_profile(user_id: int):
    """Anyone can access any user's profile"""
    user = db.query(User).filter(User.id == user_id).first()
    return user

# ❌ VULNERABLE: Can access other users' orders
@app.get("/api/orders/{order_id}")
async def get_order(order_id: int, current_user = Depends(get_current_user)):
    """Authenticated but no ownership check"""
    order = db.query(Order).filter(Order.id == order_id).first()
    return order
```

### Secure Implementation

```python
from fastapi import HTTPException, Depends

# ✅ SECURE: Check ownership
@app.get("/api/users/{user_id}/profile")
async def get_user_profile(
    user_id: int,
    current_user = Depends(get_current_user)
):
    """Only allow users to access their own profile"""
    if current_user.id != user_id and not current_user.is_admin:
        raise HTTPException(403, "Access denied")

    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(404, "User not found")

    return user

# ✅ SECURE: Verify order ownership
@app.get("/api/orders/{order_id}")
async def get_order(
    order_id: int,
    current_user = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Check if user owns the order"""
    order = db.query(Order).filter(
        Order.id == order_id,
        Order.user_id == current_user.id
    ).first()

    if not order:
        raise HTTPException(404, "Order not found")

    return order

# ✅ SECURE: Authorization helper
def verify_resource_ownership(
    resource_user_id: int,
    current_user: User
) -> bool:
    """Check if user owns resource"""
    return current_user.id == resource_user_id or current_user.is_admin

@app.get("/api/documents/{document_id}")
async def get_document(
    document_id: int,
    current_user = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Verify document ownership"""
    document = db.query(Document).filter(
        Document.id == document_id
    ).first()

    if not document:
        raise HTTPException(404, "Document not found")

    if not verify_resource_ownership(document.user_id, current_user):
        raise HTTPException(403, "Access denied")

    return document
```

---

## API2:2023 - Broken Authentication

### What It Is

Authentication mechanisms are implemented incorrectly, allowing attackers to compromise tokens or exploit implementation flaws.

### Vulnerable Code

```python
# ❌ VULNERABLE: Weak password requirements
@app.post("/register")
async def register(username: str, password: str):
    """No password strength validation"""
    hashed = hash_password(password)  # Any password accepted
    create_user(username, hashed)

# ❌ VULNERABLE: No rate limiting on login
@app.post("/login")
async def login(username: str, password: str):
    """Allows brute force attacks"""
    user = authenticate(username, password)
    return {"token": create_token(user.id)}

# ❌ VULNERABLE: Token never expires
def create_token(user_id: int):
    """Tokens are valid forever"""
    return jwt.encode({"user_id": user_id}, SECRET_KEY)
```

### Secure Implementation

```python
from passlib.context import CryptContext
from datetime import datetime, timedelta
from jose import jwt, JWTError

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class PasswordValidator:
    """Strong password validation"""

    @staticmethod
    def is_strong(password: str) -> tuple[bool, list[str]]:
        """Check password strength"""
        errors = []

        if len(password) < 12:
            errors.append("Password must be at least 12 characters")
        if not any(c.isupper() for c in password):
            errors.append("Password must contain uppercase letter")
        if not any(c.islower() for c in password):
            errors.append("Password must contain lowercase letter")
        if not any(c.isdigit() for c in password):
            errors.append("Password must contain digit")
        if not any(c in "!@#$%^&*" for c in password):
            errors.append("Password must contain special character")

        return len(errors) == 0, errors

# ✅ SECURE: Strong password requirements
@app.post("/register")
async def register(username: str, password: str):
    """Enforce strong passwords"""
    is_valid, errors = PasswordValidator.is_strong(password)
    if not is_valid:
        raise HTTPException(400, {"errors": errors})

    hashed = pwd_context.hash(password)
    user = create_user(username, hashed)
    return {"message": "User created"}

# ✅ SECURE: Rate-limited login with lockout
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

LOGIN_ATTEMPTS = defaultdict(int)
LOCKED_ACCOUNTS = defaultdict(lambda: None)

@app.post("/login")
@limiter.limit("5/minute")
async def login(
    request: Request,
    credentials: LoginCredentials,
    db: Session = Depends(get_db)
):
    """Secure login with rate limiting and account lockout"""
    # Check if account is locked
    lock_time = LOCKED_ACCOUNTS.get(credentials.username)
    if lock_time and datetime.utcnow() < lock_time:
        raise HTTPException(
            423,
            f"Account locked. Try again after {lock_time}"
        )

    # Authenticate
    user = authenticate_user(credentials.username, credentials.password, db)

    if not user:
        # Increment failed attempts
        LOGIN_ATTEMPTS[credentials.username] += 1

        # Lock after 5 failed attempts
        if LOGIN_ATTEMPTS[credentials.username] >= 5:
            LOCKED_ACCOUNTS[credentials.username] = (
                datetime.utcnow() + timedelta(minutes=30)
            )
            raise HTTPException(423, "Account locked for 30 minutes")

        raise HTTPException(401, "Invalid credentials")

    # Reset attempts on successful login
    LOGIN_ATTEMPTS[credentials.username] = 0
    LOCKED_ACCOUNTS.pop(credentials.username, None)

    # Create token with expiration
    access_token = create_access_token(
        data={"sub": str(user.id)},
        expires_delta=timedelta(minutes=30)
    )

    return {
        "access_token": access_token,
        "token_type": "bearer",
        "expires_in": 1800
    }

def create_access_token(data: dict, expires_delta: timedelta):
    """Create JWT with expiration"""
    to_encode = data.copy()
    expire = datetime.utcnow() + expires_delta
    to_encode.update({"exp": expire})

    return jwt.encode(to_encode, SECRET_KEY, algorithm="HS256")

def verify_token(token: str):
    """Verify JWT token"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id = payload.get("sub")
        exp = payload.get("exp")

        # Check expiration
        if datetime.utcfromtimestamp(exp) < datetime.utcnow():
            raise HTTPException(401, "Token expired")

        return user_id
    except JWTError:
        raise HTTPException(401, "Invalid token")
```

---

## API3:2023 - Broken Object Property Level Authorization

### What It Is

API returns more data than needed or allows modifying properties that should be read-only.

### Vulnerable Code

```python
# ❌ VULNERABLE: Exposes sensitive fields
class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    password_hash: str  # Should never be exposed!
    credit_card: str    # Sensitive data
    is_admin: bool

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """Returns all user fields including sensitive ones"""
    user = db.query(User).filter(User.id == user_id).first()
    return user

# ❌ VULNERABLE: Allows modifying is_admin
@app.patch("/users/{user_id}")
async def update_user(user_id: int, updates: dict):
    """Users can make themselves admin"""
    user = db.query(User).filter(User.id == user_id).first()

    # Blindly applies all updates
    for key, value in updates.items():
        setattr(user, key, value)

    db.commit()
    return user
```

### Secure Implementation

```python
# ✅ SECURE: Response models with only safe fields
class UserPublicResponse(BaseModel):
    """Public user data"""
    id: int
    username: str
    created_at: datetime

class UserPrivateResponse(BaseModel):
    """Private user data (own profile)"""
    id: int
    username: str
    email: str
    created_at: datetime
    # Never include password_hash, credit_card, etc.

@app.get("/users/{user_id}", response_model=UserPublicResponse)
async def get_user_public(user_id: int):
    """Returns only public fields"""
    user = db.query(User).filter(User.id == user_id).first()
    return user

@app.get("/me", response_model=UserPrivateResponse)
async def get_current_user_profile(current_user = Depends(get_current_user)):
    """Returns private fields only for own profile"""
    return current_user

# ✅ SECURE: Whitelist updatable fields
class UserUpdate(BaseModel):
    """Only allow updating specific fields"""
    username: Optional[str] = Field(None, min_length=3, max_length=50)
    email: Optional[str] = Field(None, regex=r"^[\w\.-]+@[\w\.-]+\.\w+$")
    # is_admin is NOT included - can't be modified by users

@app.patch("/users/{user_id}")
async def update_user(
    user_id: int,
    updates: UserUpdate,
    current_user = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Only allow updating whitelisted fields"""
    # Verify ownership
    if current_user.id != user_id:
        raise HTTPException(403, "Can only update own profile")

    user = db.query(User).filter(User.id == user_id).first()

    # Only update provided fields from whitelist
    update_data = updates.dict(exclude_unset=True)
    for field, value in update_data.items():
        setattr(user, field, value)

    db.commit()
    return user
```

---

## API4:2023 - Unrestricted Resource Consumption

### What It Is

APIs don't limit resource usage, leading to DoS or excessive costs.

### Vulnerable Code

```python
# ❌ VULNERABLE: No pagination limit
@app.get("/users")
async def get_users(limit: int = 100):
    """Can request unlimited users"""
    users = db.query(User).limit(limit).all()
    return users

# ❌ VULNERABLE: Unlimited file upload
@app.post("/upload")
async def upload_file(file: UploadFile):
    """No size limit"""
    contents = await file.read()  # Can exhaust memory
    save_file(contents)
```

### Secure Implementation

```python
# ✅ SECURE: Enforce pagination limits
@app.get("/users")
async def get_users(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100)  # Max 100
):
    """Limit results per page"""
    offset = (page - 1) * page_size

    users = db.query(User)\
        .offset(offset)\
        .limit(page_size)\
        .all()

    return {"users": users, "page": page, "page_size": page_size}

# ✅ SECURE: File size and type limits
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    """Enforce file size limit"""
    # Check content type
    if file.content_type not in ["image/jpeg", "image/png"]:
        raise HTTPException(400, "Only JPEG/PNG allowed")

    # Read in chunks to limit memory
    total_size = 0
    chunks = []

    async for chunk in file:
        total_size += len(chunk)
        if total_size > MAX_FILE_SIZE:
            raise HTTPException(413, "File too large")
        chunks.append(chunk)

    contents = b"".join(chunks)
    save_file(contents)

    return {"size": total_size}

# ✅ SECURE: Request timeout
@app.get("/expensive-operation")
async def expensive_operation():
    """Set timeout for long operations"""
    try:
        result = await asyncio.wait_for(
            perform_operation(),
            timeout=30.0  # 30 second timeout
        )
        return result
    except asyncio.TimeoutError:
        raise HTTPException(504, "Operation timeout")
```

---

## API5:2023 - Broken Function Level Authorization

### What It Is

APIs don't properly verify if user has permission to perform administrative functions.

### Secure Implementation

```python
# ✅ SECURE: Role-based access control
from enum import Enum

class UserRole(str, Enum):
    USER = "user"
    MODERATOR = "moderator"
    ADMIN = "admin"

def require_role(required_role: UserRole):
    """Dependency to check user role"""
    def role_checker(current_user = Depends(get_current_user)):
        role_hierarchy = {
            UserRole.USER: 1,
            UserRole.MODERATOR: 2,
            UserRole.ADMIN: 3
        }

        user_level = role_hierarchy.get(current_user.role, 0)
        required_level = role_hierarchy.get(required_role, 99)

        if user_level < required_level:
            raise HTTPException(
                403,
                f"Requires {required_role} role"
            )

        return current_user

    return role_checker

# Admin-only endpoints
@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin = Depends(require_role(UserRole.ADMIN)),
    db: Session = Depends(get_db)
):
    """Only admins can delete users"""
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(404, "User not found")

    db.delete(user)
    db.commit()

    return {"message": "User deleted"}

# Moderator endpoints
@app.post("/posts/{post_id}/hide")
async def hide_post(
    post_id: int,
    moderator = Depends(require_role(UserRole.MODERATOR))
):
    """Moderators can hide posts"""
    hide_post_in_db(post_id)
    return {"message": "Post hidden"}
```

---

## Best Practices Summary

### ✅ Do's:

1. **Always verify ownership** before returning resources
2. **Use response models** to control exposed fields
3. **Enforce pagination** and resource limits
4. **Implement role-based access control**
5. **Rate limit** all endpoints
6. **Validate all inputs** strictly
7. **Use parameterized queries**
8. **Log security events**

### ❌ Don'ts:

1. **Don't trust client-provided IDs**
2. **Don't expose sensitive fields**
3. **Don't allow unlimited requests**
4. **Don't skip authorization checks**
5. **Don't use weak passwords**
6. **Don't return detailed error messages** to clients

---

## Interview Questions

### Q1: What is BOLA and how do you prevent it?

**Answer**: Broken Object Level Authorization - accessing resources owned by other users. Prevent by:

- Always check resource ownership
- Query with both resource ID and user ID
- Return 404 instead of 403 to avoid info leakage
  Example: `WHERE id = ? AND user_id = ?`

### Q2: How do you implement proper authentication?

**Answer**:

- Strong password requirements (12+ chars, complexity)
- Hash passwords with bcrypt/argon2
- Rate limit login attempts
- Account lockout after failures
- JWT tokens with expiration
- Refresh token rotation

### Q3: What's the difference between authentication and authorization?

**Answer**:

- **Authentication**: Verify identity (who are you?)
- **Authorization**: Verify permissions (what can you do?)
  Example: Login is authentication, checking if user owns resource is authorization.

### Q4: How do you prevent resource exhaustion?

**Answer**:

- Pagination with max limits
- File upload size limits
- Request timeouts
- Rate limiting
- Connection pool limits
- Query result limits

### Q5: What is RBAC?

**Answer**: Role-Based Access Control - assign permissions based on user roles:

- Define roles (user, moderator, admin)
- Assign permissions to roles
- Check role before allowing actions
- Use role hierarchy if needed

---

## Summary

OWASP API Security Top 10:

1. **Broken Object Level Authorization** - Verify ownership
2. **Broken Authentication** - Strong passwords, rate limiting
3. **Broken Object Property Authorization** - Control exposed fields
4. **Unrestricted Resource Consumption** - Limit requests
5. **Broken Function Level Authorization** - Role-based access

Security is not optional - implement from day one! 🔒
