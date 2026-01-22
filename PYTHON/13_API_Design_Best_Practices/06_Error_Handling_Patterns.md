# 🎯 API Error Handling Patterns

## Overview

Designing robust error handling for APIs with proper status codes, error responses, and client-friendly messaging.

---

## HTTP Status Codes

### Choosing the Right Status Code

```python
from fastapi import FastAPI, HTTPException, status
from enum import Enum

class ErrorCode(Enum):
    """Application error codes"""
    # Client errors (4xx)
    INVALID_INPUT = "INVALID_INPUT"
    UNAUTHORIZED = "UNAUTHORIZED"
    FORBIDDEN = "FORBIDDEN"
    NOT_FOUND = "NOT_FOUND"
    CONFLICT = "CONFLICT"
    VALIDATION_ERROR = "VALIDATION_ERROR"
    RATE_LIMIT_EXCEEDED = "RATE_LIMIT_EXCEEDED"

    # Server errors (5xx)
    INTERNAL_ERROR = "INTERNAL_ERROR"
    DATABASE_ERROR = "DATABASE_ERROR"
    EXTERNAL_SERVICE_ERROR = "EXTERNAL_SERVICE_ERROR"

# Status code mapping
STATUS_CODE_MAP = {
    ErrorCode.INVALID_INPUT: status.HTTP_400_BAD_REQUEST,
    ErrorCode.UNAUTHORIZED: status.HTTP_401_UNAUTHORIZED,
    ErrorCode.FORBIDDEN: status.HTTP_403_FORBIDDEN,
    ErrorCode.NOT_FOUND: status.HTTP_404_NOT_FOUND,
    ErrorCode.CONFLICT: status.HTTP_409_CONFLICT,
    ErrorCode.VALIDATION_ERROR: status.HTTP_422_UNPROCESSABLE_ENTITY,
    ErrorCode.RATE_LIMIT_EXCEEDED: status.HTTP_429_TOO_MANY_REQUESTS,
    ErrorCode.INTERNAL_ERROR: status.HTTP_500_INTERNAL_SERVER_ERROR,
    ErrorCode.DATABASE_ERROR: status.HTTP_503_SERVICE_UNAVAILABLE,
    ErrorCode.EXTERNAL_SERVICE_ERROR: status.HTTP_502_BAD_GATEWAY,
}

app = FastAPI()

# Common status codes explained:
"""
2xx Success:
- 200 OK: Standard success
- 201 Created: Resource created
- 202 Accepted: Async processing started
- 204 No Content: Success but no data to return

4xx Client Errors:
- 400 Bad Request: Invalid request format
- 401 Unauthorized: Authentication required
- 403 Forbidden: Authenticated but no permission
- 404 Not Found: Resource doesn't exist
- 405 Method Not Allowed: HTTP method not supported
- 409 Conflict: Request conflicts with current state
- 422 Unprocessable Entity: Validation failed
- 429 Too Many Requests: Rate limit exceeded

5xx Server Errors:
- 500 Internal Server Error: Unexpected server error
- 502 Bad Gateway: Upstream service error
- 503 Service Unavailable: Temporary unavailability
- 504 Gateway Timeout: Upstream timeout
"""

# Usage examples
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """Get user - 200 or 404"""
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )

    return user  # 200 OK

@app.post("/users", status_code=status.HTTP_201_CREATED)
async def create_user(user_data: UserCreate):
    """Create user - 201 or 409"""
    # Check if exists
    existing = db.query(User)\
        .filter(User.email == user_data.email)\
        .first()

    if existing:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="User with this email already exists"
        )

    user = User(**user_data.dict())
    db.add(user)
    db.commit()

    return user  # 201 Created

@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    """Delete user - 204 or 404"""
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )

    db.delete(user)
    db.commit()

    # 204 No Content - no return value
```

---

## Structured Error Responses

### Standard Error Format

```python
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any
from datetime import datetime

class ErrorDetail(BaseModel):
    """Single error detail"""
    field: Optional[str] = None
    message: str
    code: str

class ErrorResponse(BaseModel):
    """Standard error response format"""
    error: str
    message: str
    status_code: int
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    path: Optional[str] = None
    details: Optional[List[ErrorDetail]] = None
    trace_id: Optional[str] = None

class APIError(Exception):
    """Base API error"""

    def __init__(
        self,
        message: str,
        error_code: ErrorCode,
        details: List[ErrorDetail] = None
    ):
        self.message = message
        self.error_code = error_code
        self.details = details or []
        super().__init__(message)

# Custom exceptions
class ValidationError(APIError):
    """Validation error"""

    def __init__(self, message: str, details: List[ErrorDetail]):
        super().__init__(
            message=message,
            error_code=ErrorCode.VALIDATION_ERROR,
            details=details
        )

class NotFoundError(APIError):
    """Resource not found"""

    def __init__(self, resource: str, identifier: Any):
        super().__init__(
            message=f"{resource} with id {identifier} not found",
            error_code=ErrorCode.NOT_FOUND
        )

class UnauthorizedError(APIError):
    """Unauthorized access"""

    def __init__(self, message: str = "Authentication required"):
        super().__init__(
            message=message,
            error_code=ErrorCode.UNAUTHORIZED
        )

# Error handler
from fastapi import Request
from fastapi.responses import JSONResponse
import uuid

@app.exception_handler(APIError)
async def api_error_handler(request: Request, exc: APIError):
    """Handle custom API errors"""
    return JSONResponse(
        status_code=STATUS_CODE_MAP[exc.error_code],
        content=ErrorResponse(
            error=exc.error_code.value,
            message=exc.message,
            status_code=STATUS_CODE_MAP[exc.error_code],
            path=request.url.path,
            details=exc.details,
            trace_id=str(uuid.uuid4())
        ).dict()
    )

# Validation error handler
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_error_handler(
    request: Request,
    exc: RequestValidationError
):
    """Handle Pydantic validation errors"""
    details = []

    for error in exc.errors():
        details.append(ErrorDetail(
            field='.'.join(str(loc) for loc in error['loc']),
            message=error['msg'],
            code='validation_error'
        ))

    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content=ErrorResponse(
            error='VALIDATION_ERROR',
            message='Request validation failed',
            status_code=422,
            path=request.url.path,
            details=details,
            trace_id=str(uuid.uuid4())
        ).dict()
    )

# Generic error handler
@app.exception_handler(Exception)
async def generic_error_handler(request: Request, exc: Exception):
    """Handle unexpected errors"""
    logger.exception("Unexpected error", exc_info=exc)

    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content=ErrorResponse(
            error='INTERNAL_ERROR',
            message='An unexpected error occurred',
            status_code=500,
            path=request.url.path,
            trace_id=str(uuid.uuid4())
        ).dict()
    )

# Usage
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    """Get user with proper error handling"""
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        raise NotFoundError("User", user_id)

    return user

# Example error response:
"""
{
  "error": "NOT_FOUND",
  "message": "User with id 123 not found",
  "status_code": 404,
  "timestamp": "2026-01-20T10:30:00Z",
  "path": "/users/123",
  "details": null,
  "trace_id": "abc-123-def-456"
}
"""
```

---

## Field-Level Validation Errors

### Detailed Validation Feedback

```python
from pydantic import BaseModel, validator, root_validator
from typing import List

class UserCreate(BaseModel):
    """User creation with validation"""
    username: str
    email: str
    password: str
    age: Optional[int] = None

    @validator('username')
    def username_valid(cls, v):
        """Validate username"""
        if len(v) < 3:
            raise ValueError('Username must be at least 3 characters')

        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')

        return v

    @validator('email')
    def email_valid(cls, v):
        """Validate email"""
        import re
        pattern = r'^[\w\.-]+@[\w\.-]+\.\w+$'

        if not re.match(pattern, v):
            raise ValueError('Invalid email format')

        return v.lower()

    @validator('password')
    def password_strong(cls, v):
        """Validate password strength"""
        if len(v) < 8:
            raise ValueError('Password must be at least 8 characters')

        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')

        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')

        return v

    @validator('age')
    def age_valid(cls, v):
        """Validate age"""
        if v is not None and (v < 0 or v > 150):
            raise ValueError('Age must be between 0 and 150')

        return v

# Example validation error response:
"""
{
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "status_code": 422,
  "timestamp": "2026-01-20T10:30:00Z",
  "path": "/users",
  "details": [
    {
      "field": "username",
      "message": "Username must be at least 3 characters",
      "code": "validation_error"
    },
    {
      "field": "password",
      "message": "Password must contain uppercase letter",
      "code": "validation_error"
    }
  ],
  "trace_id": "xyz-789"
}
"""

# Custom validation function
def validate_user_data(data: dict) -> List[ErrorDetail]:
    """Custom validation logic"""
    errors = []

    # Check username uniqueness
    if db.query(User).filter(User.username == data.get('username')).first():
        errors.append(ErrorDetail(
            field='username',
            message='Username already taken',
            code='duplicate'
        ))

    # Check email uniqueness
    if db.query(User).filter(User.email == data.get('email')).first():
        errors.append(ErrorDetail(
            field='email',
            message='Email already registered',
            code='duplicate'
        ))

    return errors

@app.post("/users")
async def create_user(user_data: UserCreate):
    """Create user with custom validation"""
    # Additional validation
    validation_errors = validate_user_data(user_data.dict())

    if validation_errors:
        raise ValidationError(
            message='User data validation failed',
            details=validation_errors
        )

    # Create user
    user = User(**user_data.dict())
    db.add(user)
    db.commit()

    return user
```

---

## Rate Limiting Errors

### Communicating Rate Limits

```python
from fastapi import Header
from datetime import datetime, timedelta
import time

class RateLimitExceeded(APIError):
    """Rate limit exceeded error"""

    def __init__(
        self,
        retry_after: int,
        limit: int,
        window: str
    ):
        super().__init__(
            message=f'Rate limit exceeded. Try again in {retry_after} seconds',
            error_code=ErrorCode.RATE_LIMIT_EXCEEDED
        )
        self.retry_after = retry_after
        self.limit = limit
        self.window = window

@app.exception_handler(RateLimitExceeded)
async def rate_limit_handler(request: Request, exc: RateLimitExceeded):
    """Handle rate limit errors"""
    response = JSONResponse(
        status_code=status.HTTP_429_TOO_MANY_REQUESTS,
        content=ErrorResponse(
            error='RATE_LIMIT_EXCEEDED',
            message=exc.message,
            status_code=429,
            path=request.url.path,
            details=[ErrorDetail(
                message=f'Limit: {exc.limit} requests per {exc.window}',
                code='rate_limit'
            )],
            trace_id=str(uuid.uuid4())
        ).dict()
    )

    # Add rate limit headers
    response.headers['Retry-After'] = str(exc.retry_after)
    response.headers['X-RateLimit-Limit'] = str(exc.limit)
    response.headers['X-RateLimit-Remaining'] = '0'
    reset_time = int(time.time()) + exc.retry_after
    response.headers['X-RateLimit-Reset'] = str(reset_time)

    return response

# Middleware to add rate limit headers
@app.middleware("http")
async def rate_limit_headers(request: Request, call_next):
    """Add rate limit headers to response"""
    # Get rate limit info for user
    limit_info = get_rate_limit_info(request)

    response = await call_next(request)

    # Add headers
    response.headers['X-RateLimit-Limit'] = str(limit_info['limit'])
    response.headers['X-RateLimit-Remaining'] = str(limit_info['remaining'])
    response.headers['X-RateLimit-Reset'] = str(limit_info['reset'])

    return response
```

---

## Database Error Handling

### Handling Database Errors Gracefully

```python
from sqlalchemy.exc import (
    IntegrityError,
    OperationalError,
    DatabaseError
)

class DatabaseUnavailableError(APIError):
    """Database unavailable"""

    def __init__(self):
        super().__init__(
            message='Database temporarily unavailable',
            error_code=ErrorCode.DATABASE_ERROR
        )

@app.exception_handler(IntegrityError)
async def integrity_error_handler(request: Request, exc: IntegrityError):
    """Handle database integrity errors"""
    # Parse constraint violation
    error_msg = str(exc.orig)

    if 'unique constraint' in error_msg.lower():
        # Extract field from error message
        field = parse_constraint_field(error_msg)

        return JSONResponse(
            status_code=status.HTTP_409_CONFLICT,
            content=ErrorResponse(
                error='CONFLICT',
                message='Resource already exists',
                status_code=409,
                path=request.url.path,
                details=[ErrorDetail(
                    field=field,
                    message=f'{field} already exists',
                    code='duplicate'
                )],
                trace_id=str(uuid.uuid4())
            ).dict()
        )

    # Generic integrity error
    return JSONResponse(
        status_code=status.HTTP_400_BAD_REQUEST,
        content=ErrorResponse(
            error='INVALID_INPUT',
            message='Data integrity constraint violated',
            status_code=400,
            path=request.url.path,
            trace_id=str(uuid.uuid4())
        ).dict()
    )

@app.exception_handler(OperationalError)
async def operational_error_handler(request: Request, exc: OperationalError):
    """Handle database operational errors"""
    logger.error(f"Database operational error: {exc}")

    return JSONResponse(
        status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
        content=ErrorResponse(
            error='DATABASE_ERROR',
            message='Database temporarily unavailable',
            status_code=503,
            path=request.url.path,
            trace_id=str(uuid.uuid4())
        ).dict()
    )
```

---

## External Service Errors

### Handling Third-Party API Failures

```python
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

class ExternalServiceError(APIError):
    """External service error"""

    def __init__(self, service_name: str, original_error: str = None):
        message = f'External service {service_name} unavailable'
        if original_error:
            message += f': {original_error}'

        super().__init__(
            message=message,
            error_code=ErrorCode.EXTERNAL_SERVICE_ERROR
        )

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
async def call_external_api(url: str, **kwargs):
    """Call external API with retry"""
    try:
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.get(url, **kwargs)
            response.raise_for_status()
            return response.json()

    except httpx.TimeoutException:
        raise ExternalServiceError(
            service_name='External API',
            original_error='Request timeout'
        )

    except httpx.HTTPStatusError as e:
        if e.response.status_code >= 500:
            # Server error - might be temporary
            raise ExternalServiceError(
                service_name='External API',
                original_error=f'Server error: {e.response.status_code}'
            )
        else:
            # Client error - don't retry
            raise APIError(
                message=f'External API error: {e.response.status_code}',
                error_code=ErrorCode.EXTERNAL_SERVICE_ERROR
            )

    except Exception as e:
        raise ExternalServiceError(
            service_name='External API',
            original_error=str(e)
        )

@app.get("/external-data")
async def get_external_data():
    """Endpoint that depends on external service"""
    try:
        data = await call_external_api('https://api.example.com/data')
        return data

    except ExternalServiceError:
        # Let error handler deal with it
        raise
```

---

## Best Practices

### ✅ Do's:

1. **Use appropriate** HTTP status codes
2. **Provide helpful** error messages
3. **Include trace IDs** for debugging
4. **Return field-level** validation errors
5. **Handle database** errors gracefully
6. **Add retry headers** for rate limits
7. **Log errors** server-side
8. **Don't expose** internal details
9. **Use consistent** error format
10. **Document errors** in API docs

### ❌ Don'ts:

1. **Don't expose** stack traces to clients
2. **Don't use 200** for errors
3. **Don't return generic** "Error" messages
4. **Don't expose** internal error codes
5. **Don't leak** sensitive information
6. **Don't ignore** error logging
7. **Don't forget** error documentation

---

## Interview Questions

### Q1: When to use 404 vs 400?

**Answer**:

- **404**: Resource doesn't exist (GET /users/999)
- **400**: Request is malformed (invalid JSON)
- **404**: Path-based (resource identification)
- **400**: Body/params based (validation)

### Q2: What's the difference between 401 and 403?

**Answer**:

- **401 Unauthorized**: Not authenticated (no credentials)
- **403 Forbidden**: Authenticated but no permission
- **401**: "Who are you?" - login required
- **403**: "I know who you are, but you can't do this"

### Q3: How to handle validation errors?

**Answer**:

- **422 Unprocessable Entity**: Validation failed
- **Field-level errors**: Which field, what's wrong
- **Error codes**: Machine-readable codes
- **Helpful messages**: Human-readable description
  Return structured errors, not just "Invalid input".

### Q4: What should error responses include?

**Answer**:

- **Error code**: Machine-readable identifier
- **Message**: Human-readable description
- **Status code**: HTTP status
- **Trace ID**: For tracking
- **Details**: Field-specific errors
- **Timestamp**: When it occurred

### Q5: How to handle external service failures?

**Answer**:

- **Retry**: With exponential backoff
- **Circuit breaker**: Stop calling if down
- **Fallback**: Default/cached response
- **Timeout**: Don't wait forever
- **502/503**: Indicate upstream issue
  Graceful degradation essential.

---

## Summary

Error handling essentials:

- **Status codes**: Use appropriate HTTP codes
- **Error format**: Consistent structured responses
- **Validation**: Field-level error details
- **Rate limits**: Retry-After headers
- **Database**: Handle constraints gracefully
- **External**: Retry and fallback strategies
- **Logging**: Track errors server-side

Errors are part of the API contract! 🎯
