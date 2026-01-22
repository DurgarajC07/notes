# 🛡️ CORS and Security Headers

## Overview

Cross-Origin Resource Sharing (CORS) and security headers are critical for protecting web applications from common vulnerabilities. FastAPI provides built-in middleware and utilities for implementing these security measures.

---

## CORS (Cross-Origin Resource Sharing)

### 1. **Basic CORS Setup**

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Configure CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allow all origins (development only!)
    allow_credentials=True,
    allow_methods=["*"],  # Allow all methods
    allow_headers=["*"],  # Allow all headers
)

@app.get("/api/data")
async def get_data():
    return {"message": "CORS enabled"}
```

### 2. **Production CORS Configuration**

```python
app = FastAPI()

# Production CORS - specific origins
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://myapp.com",
        "https://www.myapp.com",
        "https://admin.myapp.com"
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
    max_age=3600  # Cache preflight requests for 1 hour
)
```

### 3. **Environment-Based CORS**

```python
import os
from typing import List

def get_cors_origins() -> List[str]:
    """Get CORS origins based on environment"""
    env = os.getenv("ENVIRONMENT", "development")

    if env == "development":
        return [
            "http://localhost:3000",
            "http://localhost:8000",
            "http://127.0.0.1:3000"
        ]
    elif env == "staging":
        return [
            "https://staging.myapp.com",
            "https://staging-admin.myapp.com"
        ]
    else:  # production
        return [
            "https://myapp.com",
            "https://www.myapp.com",
            "https://admin.myapp.com"
        ]

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=get_cors_origins(),
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["Content-Type", "Authorization", "X-API-Key"],
    expose_headers=["X-Total-Count", "X-Page-Count"],
    max_age=600
)
```

### 4. **Dynamic CORS Validation**

```python
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware

class DynamicCORSMiddleware(BaseHTTPMiddleware):
    """Custom CORS middleware with dynamic validation"""

    def __init__(self, app, allowed_origins: List[str]):
        super().__init__(app)
        self.allowed_origins = set(allowed_origins)

    async def dispatch(self, request: Request, call_next):
        origin = request.headers.get("origin")

        # Process request
        response = await call_next(request)

        # Add CORS headers if origin is allowed
        if origin in self.allowed_origins:
            response.headers["Access-Control-Allow-Origin"] = origin
            response.headers["Access-Control-Allow-Credentials"] = "true"
            response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE"
            response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"

        return response

# Usage
app.add_middleware(
    DynamicCORSMiddleware,
    allowed_origins=[
        "https://myapp.com",
        "https://partner.com"
    ]
)
```

---

## Security Headers

### 1. **Essential Security Headers**

```python
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add security headers to all responses"""

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        # Prevent MIME type sniffing
        response.headers["X-Content-Type-Options"] = "nosniff"

        # Enable XSS protection
        response.headers["X-XSS-Protection"] = "1; mode=block"

        # Prevent clickjacking
        response.headers["X-Frame-Options"] = "DENY"

        # Strict Transport Security (HTTPS only)
        response.headers["Strict-Transport-Security"] = (
            "max-age=31536000; includeSubDomains; preload"
        )

        # Content Security Policy
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; "
            "script-src 'self' 'unsafe-inline'; "
            "style-src 'self' 'unsafe-inline'; "
            "img-src 'self' data: https:; "
            "font-src 'self' data:; "
            "connect-src 'self' https://api.myapp.com"
        )

        # Referrer Policy
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        # Permissions Policy (formerly Feature Policy)
        response.headers["Permissions-Policy"] = (
            "geolocation=(), "
            "microphone=(), "
            "camera=()"
        )

        return response

app.add_middleware(SecurityHeadersMiddleware)
```

### 2. **Content Security Policy (CSP)**

```python
class CSPMiddleware(BaseHTTPMiddleware):
    """Content Security Policy middleware"""

    def __init__(self, app, csp_directives: dict):
        super().__init__(app)
        self.csp_directives = csp_directives

    def build_csp_header(self) -> str:
        """Build CSP header from directives"""
        return "; ".join(
            f"{key} {value}"
            for key, value in self.csp_directives.items()
        )

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        response.headers["Content-Security-Policy"] = self.build_csp_header()
        return response

# Usage
app.add_middleware(
    CSPMiddleware,
    csp_directives={
        "default-src": "'self'",
        "script-src": "'self' 'unsafe-inline' https://cdn.example.com",
        "style-src": "'self' 'unsafe-inline'",
        "img-src": "'self' data: https:",
        "font-src": "'self' data:",
        "connect-src": "'self' https://api.example.com",
        "frame-ancestors": "'none'",
        "base-uri": "'self'",
        "form-action": "'self'"
    }
)
```

### 3. **Report-Only CSP**

```python
@app.post("/csp-report")
async def csp_violation_report(request: Request):
    """Endpoint to receive CSP violation reports"""
    report = await request.json()

    # Log violation
    logger.warning(f"CSP Violation: {report}")

    # Store in database for analysis
    await store_csp_violation(report)

    return {"status": "received"}

# CSP with reporting
csp_with_reporting = (
    "default-src 'self'; "
    "script-src 'self'; "
    "report-uri /csp-report; "
    "report-to csp-endpoint"
)

@app.middleware("http")
async def add_csp_report_only(request: Request, call_next):
    """Add CSP in report-only mode for testing"""
    response = await call_next(request)

    # Use report-only for testing
    response.headers["Content-Security-Policy-Report-Only"] = csp_with_reporting

    return response
```

---

## HTTPS and TLS

### 1. **Force HTTPS Redirect**

```python
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

# Redirect HTTP to HTTPS
app.add_middleware(HTTPSRedirectMiddleware)
```

### 2. **HSTS (HTTP Strict Transport Security)**

```python
class HSTSMiddleware(BaseHTTPMiddleware):
    """HTTP Strict Transport Security middleware"""

    def __init__(
        self,
        app,
        max_age: int = 31536000,  # 1 year
        include_subdomains: bool = True,
        preload: bool = False
    ):
        super().__init__(app)
        self.max_age = max_age
        self.include_subdomains = include_subdomains
        self.preload = preload

    def build_hsts_header(self) -> str:
        """Build HSTS header"""
        value = f"max-age={self.max_age}"

        if self.include_subdomains:
            value += "; includeSubDomains"

        if self.preload:
            value += "; preload"

        return value

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        # Only add HSTS on HTTPS connections
        if request.url.scheme == "https":
            response.headers["Strict-Transport-Security"] = self.build_hsts_header()

        return response

app.add_middleware(
    HSTSMiddleware,
    max_age=31536000,  # 1 year
    include_subdomains=True,
    preload=True
)
```

---

## Additional Security Measures

### 1. **Rate Limiting Headers**

```python
from datetime import datetime, timedelta

class RateLimitMiddleware(BaseHTTPMiddleware):
    """Add rate limit headers to responses"""

    async def dispatch(self, request: Request, call_next):
        # Get rate limit info (from your rate limiter)
        limit = 100
        remaining = 50
        reset_time = datetime.utcnow() + timedelta(hours=1)

        response = await call_next(request)

        # Add rate limit headers
        response.headers["X-RateLimit-Limit"] = str(limit)
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        response.headers["X-RateLimit-Reset"] = str(int(reset_time.timestamp()))

        return response
```

### 2. **Request ID Tracking**

```python
import uuid

class RequestIDMiddleware(BaseHTTPMiddleware):
    """Add unique request ID to each request"""

    async def dispatch(self, request: Request, call_next):
        # Generate or extract request ID
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))

        # Add to request state
        request.state.request_id = request_id

        # Process request
        response = await call_next(request)

        # Add request ID to response
        response.headers["X-Request-ID"] = request_id

        return response

app.add_middleware(RequestIDMiddleware)

@app.get("/test")
async def test_endpoint(request: Request):
    return {"request_id": request.state.request_id}
```

### 3. **Sensitive Data Masking**

```python
import re
from typing import Any

class SensitiveDataMiddleware(BaseHTTPMiddleware):
    """Mask sensitive data in logs and responses"""

    PATTERNS = {
        "credit_card": re.compile(r'\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}'),
        "ssn": re.compile(r'\d{3}-\d{2}-\d{4}'),
        "email": re.compile(r'[\w\.-]+@[\w\.-]+\.\w+')
    }

    def mask_data(self, text: str) -> str:
        """Mask sensitive patterns in text"""
        for pattern_name, pattern in self.PATTERNS.items():
            text = pattern.sub("****", text)
        return text

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        # Mask sensitive data in logs
        # (Implementation depends on logging setup)

        return response
```

---

## Security Testing

### 1. **Test CORS Configuration**

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_cors_allowed_origin():
    """Test CORS with allowed origin"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get(
            "/api/data",
            headers={"Origin": "https://myapp.com"}
        )

        assert response.status_code == 200
        assert response.headers["Access-Control-Allow-Origin"] == "https://myapp.com"

@pytest.mark.asyncio
async def test_cors_disallowed_origin():
    """Test CORS with disallowed origin"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get(
            "/api/data",
            headers={"Origin": "https://evil.com"}
        )

        # Response should not include CORS headers
        assert "Access-Control-Allow-Origin" not in response.headers

@pytest.mark.asyncio
async def test_cors_preflight():
    """Test CORS preflight request"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.options(
            "/api/data",
            headers={
                "Origin": "https://myapp.com",
                "Access-Control-Request-Method": "POST",
                "Access-Control-Request-Headers": "Content-Type"
            }
        )

        assert response.status_code == 200
        assert "Access-Control-Allow-Methods" in response.headers
```

### 2. **Test Security Headers**

```python
@pytest.mark.asyncio
async def test_security_headers():
    """Test security headers are present"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/")

        # Check essential security headers
        assert response.headers.get("X-Content-Type-Options") == "nosniff"
        assert response.headers.get("X-Frame-Options") == "DENY"
        assert "Content-Security-Policy" in response.headers
        assert "Strict-Transport-Security" in response.headers

@pytest.mark.asyncio
async def test_csp_header():
    """Test Content Security Policy header"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/")

        csp = response.headers.get("Content-Security-Policy")
        assert "default-src 'self'" in csp
        assert "script-src" in csp
```

---

## Common Vulnerabilities Prevention

### 1. **Clickjacking Protection**

```python
# X-Frame-Options header
response.headers["X-Frame-Options"] = "DENY"  # or "SAMEORIGIN"

# CSP frame-ancestors directive
response.headers["Content-Security-Policy"] = "frame-ancestors 'none'"
```

### 2. **MIME Type Sniffing Protection**

```python
# Prevent browsers from MIME-sniffing
response.headers["X-Content-Type-Options"] = "nosniff"
```

### 3. **XSS Protection**

```python
# Enable browser XSS filter
response.headers["X-XSS-Protection"] = "1; mode=block"

# Content Security Policy
response.headers["Content-Security-Policy"] = "script-src 'self'"
```

### 4. **Information Disclosure Prevention**

```python
class RemoveServerHeaderMiddleware(BaseHTTPMiddleware):
    """Remove server identification headers"""

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        # Remove server header
        if "Server" in response.headers:
            del response.headers["Server"]

        # Remove X-Powered-By
        if "X-Powered-By" in response.headers:
            del response.headers["X-Powered-By"]

        return response

app.add_middleware(RemoveServerHeaderMiddleware)
```

---

## Best Practices

### ✅ Do's:

1. **Whitelist specific origins** in production
2. **Enable HTTPS** everywhere
3. **Implement HSTS** with long max-age
4. **Use strict CSP** directives
5. **Add security headers** to all responses
6. **Test CORS** thoroughly
7. **Monitor CSP violations**
8. **Keep headers updated** with security best practices

### ❌ Don'ts:

1. **Don't use `allow_origins=["*"]`** in production
2. **Don't allow credentials** with wildcard origins
3. **Don't use `unsafe-eval`** in CSP
4. **Don't ignore** security header warnings
5. **Don't hardcode** origins in code
6. **Don't skip HTTPS** in production
7. **Don't expose** sensitive server information

---

## Interview Questions

### Q1: What is CORS and why is it needed?

**Answer**: CORS (Cross-Origin Resource Sharing) is a security mechanism that allows servers to specify which origins can access their resources. It's needed because browsers enforce Same-Origin Policy, which blocks cross-origin requests by default.

### Q2: What's the difference between simple and preflighted CORS requests?

**Answer**:

- **Simple requests**: GET/HEAD/POST with simple headers, executed directly
- **Preflighted requests**: Other methods or custom headers, browser sends OPTIONS request first to check if server allows the actual request

### Q3: What is Content Security Policy (CSP)?

**Answer**: CSP is a security header that specifies approved sources for content (scripts, styles, images, etc.). It prevents XSS attacks by blocking inline scripts and limiting where resources can be loaded from.

### Q4: What does the HSTS header do?

**Answer**: HTTP Strict Transport Security tells browsers to only access the site over HTTPS for a specified period. It prevents SSL stripping attacks and ensures all connections are encrypted.

### Q5: How do you prevent clickjacking?

**Answer**: Use `X-Frame-Options: DENY` or `SAMEORIGIN`, or CSP's `frame-ancestors` directive to prevent the page from being embedded in iframes on other sites.

---

## Recommended Security Headers

```python
# Complete security headers setup
SECURITY_HEADERS = {
    # Prevent MIME sniffing
    "X-Content-Type-Options": "nosniff",

    # XSS Protection
    "X-XSS-Protection": "1; mode=block",

    # Clickjacking protection
    "X-Frame-Options": "DENY",

    # HTTPS enforcement
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",

    # Content Security Policy
    "Content-Security-Policy": (
        "default-src 'self'; "
        "script-src 'self' 'unsafe-inline' https://cdn.example.com; "
        "style-src 'self' 'unsafe-inline'; "
        "img-src 'self' data: https:; "
        "font-src 'self' data:; "
        "connect-src 'self' https://api.example.com; "
        "frame-ancestors 'none'; "
        "base-uri 'self'; "
        "form-action 'self'"
    ),

    # Referrer Policy
    "Referrer-Policy": "strict-origin-when-cross-origin",

    # Permissions Policy
    "Permissions-Policy": (
        "geolocation=(), "
        "microphone=(), "
        "camera=()"
    )
}
```

---

## Summary

CORS and security headers protect against:

- **Cross-site attacks** (CORS, CSP)
- **Clickjacking** (X-Frame-Options, CSP)
- **MIME sniffing** (X-Content-Type-Options)
- **Man-in-the-middle** (HSTS)
- **XSS attacks** (CSP, X-XSS-Protection)
- **Information disclosure** (Remove server headers)

Properly configured security headers are essential for production FastAPI applications! 🛡️
