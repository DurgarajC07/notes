# 🔐 API Key and Token Security

## Overview

API keys and tokens are simple authentication mechanisms commonly used for API access control. While simpler than OAuth2, they require careful implementation to maintain security.

---

## API Key Authentication

### 1. **Header-Based API Key**

```python
from fastapi import FastAPI, Security, HTTPException, status
from fastapi.security import APIKeyHeader
from typing import Optional

app = FastAPI()

API_KEY_NAME = "X-API-Key"
api_key_header = APIKeyHeader(name=API_KEY_NAME, auto_error=False)

# API keys storage (use database in production)
API_KEYS = {
    "sk_test_4eC39HqLyjWDarjtT1zdp7dc": {
        "client_name": "Mobile App",
        "permissions": ["read", "write"],
        "rate_limit": 1000,
        "is_active": True
    },
    "sk_live_51234567890": {
        "client_name": "Dashboard",
        "permissions": ["read", "write", "admin"],
        "rate_limit": 5000,
        "is_active": True
    }
}

async def get_api_key(
    api_key_header: str = Security(api_key_header)
) -> dict:
    """Validate API key from header"""
    if not api_key_header:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Missing API Key"
        )

    if api_key_header not in API_KEYS:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid API Key"
        )

    key_data = API_KEYS[api_key_header]

    if not key_data["is_active"]:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API Key has been revoked"
        )

    return {
        "key": api_key_header,
        **key_data
    }

@app.get("/api/users")
async def list_users(api_key: dict = Security(get_api_key)):
    """Protected endpoint requiring API key"""
    return {
        "users": [...],
        "client": api_key["client_name"]
    }
```

### 2. **Query Parameter API Key**

```python
from fastapi.security import APIKeyQuery

api_key_query = APIKeyQuery(name="api_key", auto_error=False)

async def get_api_key_query(
    api_key: str = Security(api_key_query)
) -> dict:
    """Validate API key from query parameter"""
    if not api_key:
        raise HTTPException(401, "Missing API Key")

    if api_key not in API_KEYS:
        raise HTTPException(401, "Invalid API Key")

    return API_KEYS[api_key]

# Usage: GET /api/data?api_key=sk_test_123
@app.get("/api/data")
async def get_data(api_key: dict = Security(get_api_key_query)):
    return {"data": "sensitive information"}
```

### 3. **Cookie-Based API Key**

```python
from fastapi.security import APIKeyCookie

api_key_cookie = APIKeyCookie(name="api_key", auto_error=False)

async def get_api_key_cookie(
    api_key: str = Security(api_key_cookie)
) -> dict:
    """Validate API key from cookie"""
    if not api_key:
        raise HTTPException(401, "Missing API Key")

    if api_key not in API_KEYS:
        raise HTTPException(401, "Invalid API Key")

    return API_KEYS[api_key]

@app.get("/api/profile")
async def get_profile(api_key: dict = Security(get_api_key_cookie)):
    return {"profile": {...}}
```

---

## API Key Generation

### 1. **Secure Key Generation**

```python
import secrets
import hashlib
from datetime import datetime
from pydantic import BaseModel

class APIKey(BaseModel):
    key: str
    name: str
    permissions: list[str]
    created_at: datetime
    last_used: Optional[datetime] = None
    is_active: bool = True

def generate_api_key(prefix: str = "sk") -> str:
    """Generate secure API key"""
    # Generate 32 random bytes
    random_bytes = secrets.token_bytes(32)

    # Convert to hex
    key_hex = random_bytes.hex()

    # Add prefix for identification
    return f"{prefix}_{key_hex}"

def hash_api_key(api_key: str) -> str:
    """Hash API key for storage"""
    return hashlib.sha256(api_key.encode()).hexdigest()

@app.post("/api-keys/create")
async def create_api_key(
    name: str,
    permissions: list[str],
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Create new API key"""
    # Generate key
    api_key = generate_api_key()

    # Hash for storage
    key_hash = hash_api_key(api_key)

    # Store in database
    db_api_key = APIKeyModel(
        key_hash=key_hash,
        name=name,
        user_id=current_user.id,
        permissions=permissions,
        created_at=datetime.utcnow()
    )
    db.add(db_api_key)
    await db.commit()

    # Return key only once
    return {
        "api_key": api_key,  # Show only once!
        "message": "Save this key, it won't be shown again"
    }
```

### 2. **Key with Metadata**

```python
from typing import List
import json

class APIKeyMetadata(BaseModel):
    name: str
    permissions: List[str]
    ip_whitelist: Optional[List[str]] = None
    rate_limit: int = 1000
    expires_at: Optional[datetime] = None

def generate_api_key_with_metadata(
    metadata: APIKeyMetadata
) -> tuple[str, str]:
    """Generate API key with encoded metadata"""
    # Generate key
    key_id = secrets.token_urlsafe(16)
    key_secret = secrets.token_urlsafe(32)

    # Create full key
    api_key = f"sk_{key_id}.{key_secret}"

    # Store metadata separately
    key_hash = hash_api_key(api_key)

    return api_key, key_hash

@app.post("/api-keys/advanced")
async def create_advanced_api_key(
    metadata: APIKeyMetadata,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Create API key with advanced features"""
    api_key, key_hash = generate_api_key_with_metadata(metadata)

    # Store in database
    db_key = APIKeyModel(
        key_hash=key_hash,
        user_id=current_user.id,
        name=metadata.name,
        permissions=metadata.permissions,
        ip_whitelist=metadata.ip_whitelist,
        rate_limit=metadata.rate_limit,
        expires_at=metadata.expires_at,
        created_at=datetime.utcnow()
    )
    db.add(db_key)
    await db.commit()

    return {
        "api_key": api_key,
        "metadata": metadata
    }
```

---

## API Key Validation

### 1. **Database-Backed Validation**

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

async def validate_api_key(
    api_key: str,
    db: AsyncSession
) -> Optional[dict]:
    """Validate API key against database"""
    # Hash the provided key
    key_hash = hash_api_key(api_key)

    # Query database
    result = await db.execute(
        select(APIKeyModel)
        .where(APIKeyModel.key_hash == key_hash)
        .where(APIKeyModel.is_active == True)
    )
    key_record = result.scalar_one_or_none()

    if not key_record:
        return None

    # Check expiration
    if key_record.expires_at and datetime.utcnow() > key_record.expires_at:
        return None

    # Update last used
    key_record.last_used = datetime.utcnow()
    await db.commit()

    return {
        "id": key_record.id,
        "user_id": key_record.user_id,
        "name": key_record.name,
        "permissions": key_record.permissions,
        "rate_limit": key_record.rate_limit
    }

async def get_api_key_from_db(
    api_key_header: str = Security(api_key_header),
    db: AsyncSession = Depends(get_db)
) -> dict:
    """Dependency for database-backed API key validation"""
    if not api_key_header:
        raise HTTPException(401, "Missing API Key")

    key_data = await validate_api_key(api_key_header, db)

    if not key_data:
        raise HTTPException(401, "Invalid or expired API Key")

    return key_data

@app.get("/api/protected")
async def protected_endpoint(
    api_key: dict = Security(get_api_key_from_db)
):
    return {
        "message": "Access granted",
        "key_name": api_key["name"]
    }
```

### 2. **IP Whitelisting**

```python
from fastapi import Request

async def validate_api_key_with_ip(
    request: Request,
    api_key_header: str = Security(api_key_header),
    db: AsyncSession = Depends(get_db)
) -> dict:
    """Validate API key and check IP whitelist"""
    if not api_key_header:
        raise HTTPException(401, "Missing API Key")

    key_data = await validate_api_key(api_key_header, db)

    if not key_data:
        raise HTTPException(401, "Invalid API Key")

    # Check IP whitelist
    if key_data.get("ip_whitelist"):
        client_ip = request.client.host

        if client_ip not in key_data["ip_whitelist"]:
            raise HTTPException(
                403,
                f"Access denied from IP: {client_ip}"
            )

    return key_data

@app.get("/api/restricted")
async def restricted_endpoint(
    api_key: dict = Security(validate_api_key_with_ip)
):
    return {"message": "IP validated"}
```

---

## Permission-Based Access

### 1. **Permission Checking**

```python
from enum import Enum

class Permission(str, Enum):
    READ_USERS = "read:users"
    WRITE_USERS = "write:users"
    DELETE_USERS = "delete:users"
    ADMIN = "admin"

def require_permission(required_permission: Permission):
    """Factory for permission-based access control"""
    async def check_permission(
        api_key: dict = Security(get_api_key_from_db)
    ):
        permissions = api_key.get("permissions", [])

        # Check for required permission or admin
        if (required_permission.value not in permissions and
            Permission.ADMIN.value not in permissions):
            raise HTTPException(
                403,
                f"Permission '{required_permission.value}' required"
            )

        return api_key

    return check_permission

@app.get("/api/users")
async def list_users(
    api_key: dict = Security(require_permission(Permission.READ_USERS))
):
    return {"users": [...]}

@app.delete("/api/users/{user_id}")
async def delete_user(
    user_id: int,
    api_key: dict = Security(require_permission(Permission.DELETE_USERS))
):
    return {"deleted": user_id}
```

---

## Rate Limiting

### 1. **Simple Rate Limiter**

```python
from collections import defaultdict
from datetime import timedelta
import asyncio

class RateLimiter:
    """In-memory rate limiter (use Redis in production)"""
    def __init__(self):
        self.requests = defaultdict(list)

    async def check_rate_limit(
        self,
        key: str,
        limit: int,
        window_seconds: int = 60
    ) -> bool:
        """Check if request is within rate limit"""
        now = datetime.utcnow()
        cutoff = now - timedelta(seconds=window_seconds)

        # Clean old requests
        self.requests[key] = [
            req_time for req_time in self.requests[key]
            if req_time > cutoff
        ]

        # Check limit
        if len(self.requests[key]) >= limit:
            return False

        # Record request
        self.requests[key].append(now)
        return True

rate_limiter = RateLimiter()

async def check_api_key_rate_limit(
    api_key: dict = Security(get_api_key_from_db)
):
    """Check rate limit for API key"""
    key_id = api_key["id"]
    rate_limit = api_key.get("rate_limit", 1000)

    is_allowed = await rate_limiter.check_rate_limit(
        key=f"api_key:{key_id}",
        limit=rate_limit,
        window_seconds=3600  # 1 hour
    )

    if not is_allowed:
        raise HTTPException(
            429,
            f"Rate limit exceeded: {rate_limit} requests per hour"
        )

    return api_key

@app.get("/api/data")
async def get_data(
    api_key: dict = Security(check_api_key_rate_limit)
):
    return {"data": "..."}
```

### 2. **Redis-Based Rate Limiter**

```python
import aioredis

async def get_redis():
    return await aioredis.create_redis_pool("redis://localhost")

async def check_rate_limit_redis(
    api_key: dict = Security(get_api_key_from_db),
    redis = Depends(get_redis)
):
    """Redis-based rate limiting"""
    key_id = api_key["id"]
    rate_limit = api_key["rate_limit"]

    # Redis key
    redis_key = f"rate_limit:api_key:{key_id}"

    # Increment counter
    count = await redis.incr(redis_key)

    # Set expiration on first request
    if count == 1:
        await redis.expire(redis_key, 3600)  # 1 hour

    # Check limit
    if count > rate_limit:
        # Get TTL for retry-after header
        ttl = await redis.ttl(redis_key)

        raise HTTPException(
            status_code=429,
            detail=f"Rate limit exceeded",
            headers={"Retry-After": str(ttl)}
        )

    return api_key
```

---

## Token Management

### 1. **List API Keys**

```python
@app.get("/api-keys")
async def list_api_keys(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """List user's API keys (without revealing keys)"""
    result = await db.execute(
        select(APIKeyModel)
        .where(APIKeyModel.user_id == current_user.id)
        .where(APIKeyModel.is_active == True)
    )
    keys = result.scalars().all()

    return {
        "api_keys": [
            {
                "id": key.id,
                "name": key.name,
                "permissions": key.permissions,
                "created_at": key.created_at,
                "last_used": key.last_used,
                "expires_at": key.expires_at
            }
            for key in keys
        ]
    }
```

### 2. **Revoke API Key**

```python
@app.delete("/api-keys/{key_id}")
async def revoke_api_key(
    key_id: int,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Revoke an API key"""
    result = await db.execute(
        select(APIKeyModel)
        .where(APIKeyModel.id == key_id)
        .where(APIKeyModel.user_id == current_user.id)
    )
    key = result.scalar_one_or_none()

    if not key:
        raise HTTPException(404, "API key not found")

    key.is_active = False
    key.revoked_at = datetime.utcnow()
    await db.commit()

    return {"message": "API key revoked"}
```

### 3. **Rotate API Key**

```python
@app.post("/api-keys/{key_id}/rotate")
async def rotate_api_key(
    key_id: int,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Rotate an API key (generate new key)"""
    # Get old key
    result = await db.execute(
        select(APIKeyModel)
        .where(APIKeyModel.id == key_id)
        .where(APIKeyModel.user_id == current_user.id)
    )
    old_key = result.scalar_one_or_none()

    if not old_key:
        raise HTTPException(404, "API key not found")

    # Generate new key
    new_api_key = generate_api_key()
    new_key_hash = hash_api_key(new_api_key)

    # Create new key record
    new_key = APIKeyModel(
        key_hash=new_key_hash,
        user_id=current_user.id,
        name=old_key.name,
        permissions=old_key.permissions,
        ip_whitelist=old_key.ip_whitelist,
        rate_limit=old_key.rate_limit,
        created_at=datetime.utcnow()
    )
    db.add(new_key)

    # Revoke old key
    old_key.is_active = False
    old_key.revoked_at = datetime.utcnow()

    await db.commit()

    return {
        "api_key": new_api_key,
        "message": "API key rotated. Old key has been revoked."
    }
```

---

## Security Best Practices

### 1. **Audit Logging**

```python
async def log_api_key_usage(
    api_key: dict,
    request: Request,
    db: AsyncSession
):
    """Log API key usage for audit"""
    log_entry = APIKeyLog(
        api_key_id=api_key["id"],
        endpoint=str(request.url),
        method=request.method,
        ip_address=request.client.host,
        user_agent=request.headers.get("user-agent"),
        timestamp=datetime.utcnow()
    )
    db.add(log_entry)
    await db.commit()

async def get_api_key_with_logging(
    request: Request,
    api_key_header: str = Security(api_key_header),
    db: AsyncSession = Depends(get_db)
) -> dict:
    """Validate API key and log usage"""
    key_data = await validate_api_key(api_key_header, db)

    if not key_data:
        raise HTTPException(401, "Invalid API Key")

    # Log usage
    await log_api_key_usage(key_data, request, db)

    return key_data
```

### 2. **Key Expiration**

```python
from datetime import timedelta

@app.post("/api-keys/create-temporary")
async def create_temporary_api_key(
    name: str,
    permissions: list[str],
    ttl_hours: int = 24,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Create temporary API key with expiration"""
    api_key = generate_api_key()
    key_hash = hash_api_key(api_key)

    # Set expiration
    expires_at = datetime.utcnow() + timedelta(hours=ttl_hours)

    db_key = APIKeyModel(
        key_hash=key_hash,
        name=name,
        user_id=current_user.id,
        permissions=permissions,
        expires_at=expires_at,
        created_at=datetime.utcnow()
    )
    db.add(db_key)
    await db.commit()

    return {
        "api_key": api_key,
        "expires_at": expires_at,
        "message": f"Key expires in {ttl_hours} hours"
    }
```

---

## Best Practices Summary

### ✅ Do's:

1. **Use HTTPS** always
2. **Hash API keys** before storing
3. **Generate cryptographically secure** keys
4. **Implement rate limiting**
5. **Log API key usage**
6. **Set expiration dates**
7. **Allow key rotation**
8. **Use IP whitelisting** when possible

### ❌ Don'ts:

1. **Don't store keys in plain text**
2. **Don't use predictable keys**
3. **Don't share keys** across services
4. **Don't log actual key values**
5. **Don't skip validation**
6. **Don't ignore rate limits**
7. **Don't allow unlimited key creation**

---

## Interview Questions

### Q1: How do you securely store API keys?

**Answer**: Hash the API key using SHA-256 or bcrypt before storing. Store only the hash in database. Show the actual key only once when generated.

### Q2: What's the difference between API keys and JWT?

**Answer**:

- **API Keys**: Long-lived, revocable, stored in database
- **JWT**: Short-lived, stateless, self-contained
- Use API keys for machine-to-machine, JWT for user sessions

### Q3: How do you implement API key rotation?

**Answer**: Generate new key, copy permissions/metadata, revoke old key. Provide grace period where both keys work if needed.

### Q4: What security measures should API keys have?

**Answer**:

- Rate limiting
- IP whitelisting
- Permission-based access
- Expiration dates
- Audit logging
- Ability to revoke

### Q5: How do you prevent API key leakage?

**Answer**:

- Never log keys
- Use environment variables
- Implement key scanning in repos
- Monitor for suspicious usage
- Rotate keys regularly
- Limit key permissions

---

## Summary

API key security requires:

- **Secure generation** with cryptographic randomness
- **Safe storage** using hashing
- **Access control** with permissions
- **Rate limiting** to prevent abuse
- **Audit logging** for monitoring
- **Key management** (rotation, revocation)

Properly implemented API key authentication provides a simple yet secure way to control API access! 🔐
