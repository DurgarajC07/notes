# 🔄 API Versioning Strategies

## Overview

API versioning allows you to evolve your API while maintaining backwards compatibility for existing clients. Choosing the right versioning strategy is crucial for long-term API maintainability.

---

## Versioning Approaches

### 1. **URL Path Versioning**

Most common and recommended for RESTful APIs.

```python
from fastapi import FastAPI, APIRouter

# V1 Router
router_v1 = APIRouter(prefix="/api/v1", tags=["v1"])

@router_v1.get("/users")
async def get_users_v1():
    """V1: Returns simple user list"""
    return {"users": [{"id": 1, "name": "Alice"}]}

@router_v1.get("/users/{user_id}")
async def get_user_v1(user_id: int):
    """V1: Basic user details"""
    return {"id": user_id, "name": "User"}

# V2 Router with enhancements
router_v2 = APIRouter(prefix="/api/v2", tags=["v2"])

@router_v2.get("/users")
async def get_users_v2():
    """V2: Returns users with pagination"""
    return {
        "users": [{"id": 1, "name": "Alice", "email": "alice@example.com"}],
        "meta": {"page": 1, "total": 1}
    }

@router_v2.get("/users/{user_id}")
async def get_user_v2(user_id: int):
    """V2: Enhanced user details with metadata"""
    return {
        "id": user_id,
        "name": "User",
        "email": "user@example.com",
        "created_at": "2024-01-01T00:00:00Z"
    }

app = FastAPI()
app.include_router(router_v1)
app.include_router(router_v2)

# URLs:
# GET /api/v1/users
# GET /api/v2/users
```

**Pros:**

- Clear and explicit
- Easy to route
- Cacheable
- Works with all clients

**Cons:**

- URL pollution
- Duplicated code
- Hard to maintain many versions

### 2. **Header Versioning**

Version specified in HTTP header.

```python
from fastapi import Header, HTTPException

@app.get("/users")
async def get_users(api_version: str = Header("1.0", alias="API-Version")):
    """
    Get users with version in header.

    Headers:
        API-Version: 1.0 or 2.0
    """
    if api_version == "1.0":
        return {"users": [{"id": 1, "name": "Alice"}]}
    elif api_version == "2.0":
        return {
            "users": [{"id": 1, "name": "Alice", "email": "alice@example.com"}],
            "meta": {"page": 1}
        }
    else:
        raise HTTPException(400, f"Unsupported API version: {api_version}")

# Request:
# GET /users
# Headers:
#   API-Version: 2.0
```

**Pros:**

- Clean URLs
- Version negotiation
- No URL changes

**Cons:**

- Hidden from URL
- Not cacheable by default
- Harder to test

### 3. **Query Parameter Versioning**

Version as query string parameter.

```python
@app.get("/users")
async def get_users(version: str = Query("1.0", enum=["1.0", "2.0"])):
    """
    Get users with version parameter.

    Query params:
        version: API version (1.0 or 2.0)
    """
    if version == "1.0":
        return {"users": [{"id": 1, "name": "Alice"}]}
    else:  # version == "2.0"
        return {
            "users": [{"id": 1, "name": "Alice", "email": "alice@example.com"}],
            "meta": {"page": 1}
        }

# Request:
# GET /users?version=2.0
```

**Pros:**

- Easy to test
- No code duplication
- Default version possible

**Cons:**

- Mixes with other params
- Can be confusing
- Version in every URL

### 4. **Content Negotiation (Accept Header)**

Use Accept header with custom media types.

```python
@app.get("/users")
async def get_users(accept: str = Header("application/vnd.myapi.v1+json")):
    """
    Content negotiation versioning.

    Headers:
        Accept: application/vnd.myapi.v1+json
        Accept: application/vnd.myapi.v2+json
    """
    if "v1" in accept:
        return Response(
            content='{"users":[{"id":1,"name":"Alice"}]}',
            media_type="application/vnd.myapi.v1+json"
        )
    elif "v2" in accept:
        return Response(
            content='{"users":[{"id":1,"name":"Alice","email":"alice@example.com"}]}',
            media_type="application/vnd.myapi.v2+json"
        )
    else:
        raise HTTPException(406, "Not Acceptable")

# Request:
# GET /users
# Headers:
#   Accept: application/vnd.myapi.v2+json
```

**Pros:**

- RESTful
- Clean URLs
- Standard HTTP

**Cons:**

- Complex
- Not intuitive
- Client support needed

---

## Version Management Patterns

### 1. **Shared Code with Version-Specific Logic**

```python
from typing import Literal

class UserServiceV1:
    """User service version 1"""

    @staticmethod
    def get_user(user_id: int):
        return {"id": user_id, "name": "User"}

    @staticmethod
    def format_response(user):
        """V1 format"""
        return {"id": user["id"], "name": user["name"]}

class UserServiceV2:
    """User service version 2"""

    @staticmethod
    def get_user(user_id: int):
        return {
            "id": user_id,
            "name": "User",
            "email": "user@example.com",
            "created_at": "2024-01-01T00:00:00Z"
        }

    @staticmethod
    def format_response(user):
        """V2 format with additional fields"""
        return {
            "id": user["id"],
            "name": user["name"],
            "email": user["email"],
            "created_at": user["created_at"]
        }

def get_user_service(version: Literal["v1", "v2"]):
    """Factory for version-specific services"""
    if version == "v1":
        return UserServiceV1()
    else:
        return UserServiceV2()

@app.get("/api/{version}/users/{user_id}")
async def get_user(version: Literal["v1", "v2"], user_id: int):
    """Get user with version-specific logic"""
    service = get_user_service(version)
    user = service.get_user(user_id)
    return service.format_response(user)
```

### 2. **Deprecation Warnings**

```python
from fastapi import Response
import warnings

@app.get("/api/v1/users", deprecated=True)
async def get_users_v1_deprecated(response: Response):
    """
    V1 endpoint (deprecated).

    **DEPRECATED**: This endpoint is deprecated. Use /api/v2/users instead.
    Will be removed in version 3.0 (2025-06-01).
    """
    # Add deprecation header
    response.headers["Warning"] = '299 - "Deprecated API. Use /api/v2/users"'
    response.headers["Sunset"] = "2025-06-01"  # RFC 8594
    response.headers["Link"] = '</api/v2/users>; rel="successor-version"'

    warnings.warn("V1 API is deprecated", DeprecationWarning)

    return {"users": []}
```

### 3. **Version Routing Middleware**

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class APIVersionMiddleware(BaseHTTPMiddleware):
    """Middleware to handle API versioning"""

    async def dispatch(self, request: Request, call_next):
        # Extract version from header
        version = request.headers.get("API-Version", "1.0")

        # Store in request state
        request.state.api_version = version

        # Add version to response headers
        response = await call_next(request)
        response.headers["API-Version"] = version

        return response

app.add_middleware(APIVersionMiddleware)

def get_api_version(request: Request) -> str:
    """Dependency to get API version"""
    return request.state.api_version

@app.get("/users")
async def get_users(version: str = Depends(get_api_version)):
    """Get users based on API version"""
    if version == "1.0":
        return {"users": []}
    else:
        return {"users": [], "meta": {}}
```

---

## Breaking Changes

### 1. **Identifying Breaking Changes**

```python
# ❌ Breaking changes (require new version):
# - Removing fields
# - Renaming fields
# - Changing field types
# - Removing endpoints
# - Changing HTTP methods
# - Changing error formats

# V1
class UserV1(BaseModel):
    id: int
    name: str
    email: str

# V2 - BREAKING: removed email field
class UserV2(BaseModel):
    id: int
    name: str
    # email removed - BREAKING!

# ✅ Non-breaking changes (same version):
# - Adding optional fields
# - Adding new endpoints
# - Making required fields optional
# - Adding query parameters

# V1
class ProductV1(BaseModel):
    id: int
    name: str
    price: float

# V1.1 - Non-breaking: added optional field
class ProductV1_1(BaseModel):
    id: int
    name: str
    price: float
    description: Optional[str] = None  # New optional field
```

### 2. **Migration Helpers**

```python
class V1ToV2Adapter:
    """Adapter for migrating V1 responses to V2 format"""

    @staticmethod
    def adapt_user(v1_user: dict) -> dict:
        """Convert V1 user to V2 format"""
        return {
            **v1_user,
            "email": v1_user.get("email", "unknown@example.com"),
            "created_at": "2024-01-01T00:00:00Z"
        }

    @staticmethod
    def adapt_users_list(v1_users: list) -> dict:
        """Convert V1 user list to V2 paginated format"""
        return {
            "users": [V1ToV2Adapter.adapt_user(u) for u in v1_users],
            "meta": {
                "page": 1,
                "total": len(v1_users)
            }
        }

@app.get("/api/v2/users")
async def get_users_v2_adapted():
    """V2 endpoint using adapter for legacy data"""
    v1_users = get_legacy_users()  # Returns V1 format
    return V1ToV2Adapter.adapt_users_list(v1_users)
```

---

## Version Documentation

### 1. **OpenAPI Multiple Versions**

```python
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

# V1 App
app_v1 = FastAPI(title="My API V1", version="1.0.0")

@app_v1.get("/users")
async def get_users_v1():
    return {"users": []}

# V2 App
app_v2 = FastAPI(title="My API V2", version="2.0.0")

@app_v2.get("/users")
async def get_users_v2():
    return {"users": [], "meta": {}}

# Main app
app = FastAPI()
app.mount("/v1", app_v1)
app.mount("/v2", app_v2)

# Separate OpenAPI docs:
# /v1/docs - V1 documentation
# /v2/docs - V2 documentation
```

### 2. **Changelog Endpoint**

```python
@app.get("/api/changelog")
async def get_changelog():
    """
    API changelog showing version history.
    """
    return {
        "versions": [
            {
                "version": "2.0.0",
                "release_date": "2024-01-01",
                "changes": [
                    "Added pagination to /users endpoint",
                    "Added email field to user response",
                    "Added /users/{id}/posts endpoint"
                ],
                "breaking_changes": [
                    "Removed /legacy-users endpoint"
                ]
            },
            {
                "version": "1.0.0",
                "release_date": "2023-06-01",
                "changes": [
                    "Initial release"
                ],
                "breaking_changes": []
            }
        ],
        "current_version": "2.0.0",
        "supported_versions": ["1.0.0", "2.0.0"],
        "deprecated_versions": []
    }
```

---

## Version Sunset Policy

```python
from datetime import datetime, timedelta

VERSION_SUNSET_DATES = {
    "v1": datetime(2025, 6, 1),  # V1 sunset date
    "v2": None,  # Current version, no sunset
}

@app.middleware("http")
async def version_sunset_middleware(request: Request, call_next):
    """Add sunset headers for deprecated versions"""
    version = extract_version_from_path(request.url.path)

    if version and version in VERSION_SUNSET_DATES:
        sunset_date = VERSION_SUNSET_DATES[version]

        if sunset_date:
            response = await call_next(request)

            # Add Sunset header (RFC 8594)
            response.headers["Sunset"] = sunset_date.strftime("%a, %d %b %Y %H:%M:%S GMT")

            # Warning if sunset is soon
            days_until_sunset = (sunset_date - datetime.now()).days
            if days_until_sunset < 90:
                response.headers["Warning"] = (
                    f'299 - "API version {version} will be sunset on {sunset_date.date()}. '
                    f'Please upgrade to latest version."'
                )

            return response

    return await call_next(request)
```

---

## Best Practices

### ✅ Do's:

1. **Use URL path versioning** for clarity
2. **Support multiple versions** simultaneously
3. **Document breaking changes** clearly
4. **Provide migration guides**
5. **Use semantic versioning** (MAJOR.MINOR.PATCH)
6. **Set sunset dates** for old versions
7. **Maintain backwards compatibility** when possible
8. **Version from day one**

### ❌ Don'ts:

1. **Don't break existing clients** without warning
2. **Don't support too many versions** (3 max recommended)
3. **Don't change versions** for non-breaking changes
4. **Don't forget to document** version differences
5. **Don't remove versions** without deprecation period
6. **Don't version every endpoint** individually

---

## Interview Questions

### Q1: What's the difference between versioning strategies?

**Answer**:

- **URL path**: `/v1/users` - Most common, clear, cacheable
- **Header**: `API-Version: 1.0` - Clean URLs, harder to test
- **Query param**: `/users?version=1.0` - Easy to test, URL pollution
- **Accept header**: `application/vnd.api.v1+json` - RESTful but complex

### Q2: When should you create a new API version?

**Answer**: Create new version for breaking changes:

- Removing/renaming fields
- Changing response structure
- Removing endpoints
- Changing status codes
- Changing authentication
  Non-breaking changes (adding optional fields) don't need new version.

### Q3: How do you handle API deprecation?

**Answer**:

1. Announce deprecation with timeline
2. Add `Deprecated: true` in OpenAPI
3. Return `Warning` and `Sunset` headers
4. Provide migration documentation
5. Support old version for 6-12 months
6. Finally remove after sunset date

### Q4: What is semantic versioning for APIs?

**Answer**: MAJOR.MINOR.PATCH format:

- **MAJOR**: Breaking changes (increment on incompatible API changes)
- **MINOR**: New features (backward-compatible)
- **PATCH**: Bug fixes (backward-compatible)
  Example: 2.1.3 → Breaking change → 3.0.0

### Q5: How do you maintain multiple API versions?

**Answer**: Strategies:

- Separate routers for each version
- Shared business logic, version-specific adapters
- Feature flags for version-specific behavior
- Automated tests for all versions
- Documentation for each version

---

## Summary

API versioning strategies:

- **URL path versioning** (recommended for REST)
- **Header/query parameter** alternatives
- **Semantic versioning** for version numbers
- **Deprecation policy** with sunset dates
- **Migration guides** for clients
- **Documentation** for all versions

Proper versioning enables API evolution without breaking clients! 🔄
