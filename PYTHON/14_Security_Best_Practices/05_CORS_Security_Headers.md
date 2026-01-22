# 🔐 CORS and Security Headers

## Overview

Comprehensive guide to Cross-Origin Resource Sharing (CORS) and HTTP security headers for protecting web applications and APIs.

---

## CORS (Cross-Origin Resource Sharing)

### Understanding Same-Origin Policy

```python
"""
Same-Origin Policy (SOP):
- Browser security feature
- Restricts scripts from one origin accessing resources from another origin
- Origin = protocol + domain + port

Examples:
✅ Same origin:
   https://example.com/page1
   https://example.com/page2

❌ Different origins:
   https://example.com       vs  http://example.com       (protocol)
   https://example.com       vs  https://api.example.com  (subdomain)
   https://example.com:443   vs  https://example.com:8080 (port)

Without CORS:
- JavaScript from https://frontend.com cannot make AJAX requests to https://api.backend.com
- Prevents malicious scripts from accessing your data
"""
```

### CORS Headers

```python
"""
CORS Request Headers (Browser sends):
- Origin: https://frontend.com
- Access-Control-Request-Method: POST
- Access-Control-Request-Headers: content-type, authorization

CORS Response Headers (Server sends):
- Access-Control-Allow-Origin: https://frontend.com
- Access-Control-Allow-Methods: GET, POST, PUT, DELETE
- Access-Control-Allow-Headers: Content-Type, Authorization
- Access-Control-Allow-Credentials: true
- Access-Control-Max-Age: 3600
- Access-Control-Expose-Headers: X-Custom-Header
"""
```

### Django CORS Configuration

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # Must be first
    'django.middleware.common.CommonMiddleware',
    # ...
]

# Development - Allow all origins (NOT for production!)
CORS_ALLOW_ALL_ORIGINS = True

# Production - Specific origins only
CORS_ALLOWED_ORIGINS = [
    "https://frontend.example.com",
    "https://app.example.com",
    "https://admin.example.com",
]

# Or use regex for subdomains
CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://\w+\.example\.com$",
]

# Allow credentials (cookies, authorization headers)
CORS_ALLOW_CREDENTIALS = True

# Allowed HTTP methods
CORS_ALLOW_METHODS = [
    'DELETE',
    'GET',
    'OPTIONS',
    'PATCH',
    'POST',
    'PUT',
]

# Allowed request headers
CORS_ALLOW_HEADERS = [
    'accept',
    'accept-encoding',
    'authorization',
    'content-type',
    'dnt',
    'origin',
    'user-agent',
    'x-csrftoken',
    'x-requested-with',
]

# Headers exposed to JavaScript
CORS_EXPOSE_HEADERS = [
    'content-length',
    'x-pagination-count',
    'x-pagination-page',
]

# Preflight request cache time
CORS_PREFLIGHT_MAX_AGE = 86400  # 24 hours

# Custom CORS middleware
class CustomCorsMiddleware:
    """Custom CORS handling with dynamic origins"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Get origin from request
        origin = request.META.get('HTTP_ORIGIN')

        # Check if origin is allowed
        if self.is_origin_allowed(origin):
            # Handle preflight request
            if request.method == 'OPTIONS':
                response = HttpResponse()
                response.status_code = 200
            else:
                response = self.get_response(request)

            # Add CORS headers
            response['Access-Control-Allow-Origin'] = origin
            response['Access-Control-Allow-Credentials'] = 'true'
            response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
            response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
            response['Access-Control-Max-Age'] = '3600'

            return response

        return self.get_response(request)

    def is_origin_allowed(self, origin):
        """Check if origin is in allowed list"""
        if not origin:
            return False

        # Check against database or cache
        from django.core.cache import cache

        allowed_origins = cache.get('allowed_origins')
        if allowed_origins is None:
            # Load from database
            allowed_origins = list(
                AllowedOrigin.objects.filter(is_active=True)
                .values_list('origin', flat=True)
            )
            cache.set('allowed_origins', allowed_origins, 3600)

        return origin in allowed_origins
```

### FastAPI CORS Configuration

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Development - Allow all
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Production - Specific origins
origins = [
    "https://frontend.example.com",
    "https://app.example.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
    expose_headers=["X-Total-Count"],
    max_age=3600,
)

# Dynamic origin validation
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response

class DynamicCORSMiddleware(BaseHTTPMiddleware):
    """CORS middleware with dynamic origin validation"""

    async def dispatch(self, request, call_next):
        origin = request.headers.get('origin')

        # Validate origin
        if origin and await self.is_allowed_origin(origin):
            # Handle preflight
            if request.method == 'OPTIONS':
                response = Response()
                response.status_code = 200
            else:
                response = await call_next(request)

            # Add CORS headers
            response.headers['Access-Control-Allow-Origin'] = origin
            response.headers['Access-Control-Allow-Credentials'] = 'true'
            response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE'
            response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'

            return response

        return await call_next(request)

    async def is_allowed_origin(self, origin: str) -> bool:
        """Check if origin is allowed"""
        # Check against database, Redis, etc.
        return origin in ['https://frontend.example.com']

app.add_middleware(DynamicCORSMiddleware)
```

---

## Security Headers

### Content Security Policy (CSP)

```python
"""
Content Security Policy:
- Prevents XSS attacks
- Controls which resources can be loaded
- Defines trusted sources for scripts, styles, images, etc.
"""

# Django middleware for CSP
class CSPMiddleware:
    """Add Content Security Policy headers"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)

        # CSP directives
        csp = {
            "default-src": ["'self'"],
            "script-src": ["'self'", "https://cdn.example.com"],
            "style-src": ["'self'", "'unsafe-inline'"],  # Allow inline styles
            "img-src": ["'self'", "data:", "https:"],
            "font-src": ["'self'", "https://fonts.gstatic.com"],
            "connect-src": ["'self'", "https://api.example.com"],
            "frame-ancestors": ["'none'"],  # Prevent clickjacking
            "base-uri": ["'self'"],
            "form-action": ["'self'"],
        }

        # Build CSP string
        csp_string = "; ".join(
            f"{key} {' '.join(values)}"
            for key, values in csp.items()
        )

        response['Content-Security-Policy'] = csp_string

        # Report-only mode for testing
        # response['Content-Security-Policy-Report-Only'] = csp_string

        return response

# Using django-csp package
# settings.py
CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", "https://cdn.example.com")
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'")
CSP_IMG_SRC = ("'self'", "data:", "https:")
CSP_FONT_SRC = ("'self'", "https://fonts.gstatic.com")
CSP_CONNECT_SRC = ("'self'", "https://api.example.com")
CSP_FRAME_ANCESTORS = ("'none'",)

# Report violations
CSP_REPORT_URI = '/csp-report/'

# Report-only mode
CSP_REPORT_ONLY = False
```

### Strict-Transport-Security (HSTS)

```python
"""
HTTP Strict Transport Security:
- Forces HTTPS connections
- Prevents protocol downgrade attacks
- Prevents man-in-the-middle attacks
"""

# Django
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Custom middleware
class HSTSMiddleware:
    """Add HSTS header"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)

        # Only add HSTS on HTTPS
        if request.is_secure():
            max_age = 31536000  # 1 year
            response['Strict-Transport-Security'] = (
                f'max-age={max_age}; includeSubDomains; preload'
            )

        return response
```

### X-Frame-Options

```python
"""
X-Frame-Options:
- Prevents clickjacking attacks
- Controls if page can be framed
"""

# Django
X_FRAME_OPTIONS = 'DENY'  # or 'SAMEORIGIN'

# Custom middleware
class XFrameOptionsMiddleware:
    """Add X-Frame-Options header"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)

        # DENY: Cannot be framed by anyone
        # SAMEORIGIN: Can be framed by same origin only
        # ALLOW-FROM uri: Can be framed by specific URI

        response['X-Frame-Options'] = 'DENY'

        # Or use CSP instead (more flexible)
        # response['Content-Security-Policy'] = "frame-ancestors 'none'"

        return response
```

### X-Content-Type-Options

```python
"""
X-Content-Type-Options:
- Prevents MIME type sniffing
- Forces browser to respect Content-Type
"""

# Django
SECURE_CONTENT_TYPE_NOSNIFF = True

# Custom middleware
class XContentTypeOptionsMiddleware:
    """Add X-Content-Type-Options header"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)

        # Prevent MIME sniffing
        response['X-Content-Type-Options'] = 'nosniff'

        return response
```

### X-XSS-Protection

```python
"""
X-XSS-Protection:
- Enables browser XSS filter
- Legacy header (CSP preferred)
"""

# Middleware
class XXSSProtectionMiddleware:
    """Add X-XSS-Protection header"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)

        # 0: Disable filter
        # 1: Enable filter
        # 1; mode=block: Enable and block page if XSS detected

        response['X-XSS-Protection'] = '1; mode=block'

        return response
```

### Referrer-Policy

```python
"""
Referrer-Policy:
- Controls referrer information sent
- Prevents leaking sensitive URLs
"""

# Middleware
class ReferrerPolicyMiddleware:
    """Add Referrer-Policy header"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)

        # Options:
        # no-referrer: Never send referrer
        # no-referrer-when-downgrade: Don't send on HTTPS->HTTP
        # same-origin: Only send to same origin
        # origin: Send only origin, not full URL
        # strict-origin: Origin only, not on HTTPS->HTTP
        # strict-origin-when-cross-origin: Full URL same-origin, origin cross-origin

        response['Referrer-Policy'] = 'strict-origin-when-cross-origin'

        return response
```

### Permissions-Policy

```python
"""
Permissions-Policy (formerly Feature-Policy):
- Controls browser features and APIs
- Prevents unauthorized access to camera, microphone, etc.
"""

# Middleware
class PermissionsPolicyMiddleware:
    """Add Permissions-Policy header"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)

        # Disable all features
        # response['Permissions-Policy'] = 'geolocation=(), microphone=(), camera=()'

        # Allow specific features for same origin
        policies = [
            'geolocation=(self)',
            'microphone=()',
            'camera=()',
            'payment=(self)',
            'usb=()',
        ]

        response['Permissions-Policy'] = ', '.join(policies)

        return response
```

---

## Complete Security Headers Setup

### Django Production Settings

```python
# settings.py

# HTTPS/SSL
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# HSTS
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Security headers
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
X_FRAME_OPTIONS = 'DENY'

# Referrer policy
SECURE_REFERRER_POLICY = 'strict-origin-when-cross-origin'

# Custom security middleware
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'myapp.middleware.CSPMiddleware',
    'myapp.middleware.PermissionsPolicyMiddleware',
    # ...
]
```

### FastAPI Security Headers Middleware

```python
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add all security headers"""

    async def dispatch(self, request, call_next):
        response = await call_next(request)

        # HSTS
        if request.url.scheme == 'https':
            response.headers['Strict-Transport-Security'] = (
                'max-age=31536000; includeSubDomains; preload'
            )

        # Content Security Policy
        csp = (
            "default-src 'self'; "
            "script-src 'self' https://cdn.example.com; "
            "style-src 'self' 'unsafe-inline'; "
            "img-src 'self' data: https:; "
            "font-src 'self' https://fonts.gstatic.com; "
            "connect-src 'self' https://api.example.com; "
            "frame-ancestors 'none'"
        )
        response.headers['Content-Security-Policy'] = csp

        # Other security headers
        response.headers['X-Content-Type-Options'] = 'nosniff'
        response.headers['X-Frame-Options'] = 'DENY'
        response.headers['X-XSS-Protection'] = '1; mode=block'
        response.headers['Referrer-Policy'] = 'strict-origin-when-cross-origin'

        # Permissions policy
        response.headers['Permissions-Policy'] = (
            'geolocation=(), microphone=(), camera=()'
        )

        return response

app.add_middleware(SecurityHeadersMiddleware)
```

---

## Best Practices

### ✅ Do's:

1. **Use HTTPS** everywhere in production
2. **Enable HSTS** with long max-age
3. **Set CSP** to prevent XSS
4. **Configure CORS** with specific origins
5. **Add X-Frame-Options** to prevent clickjacking
6. **Use nosniff** for Content-Type
7. **Test headers** with security scanners
8. **Monitor violations** with CSP reporting
9. **Update headers** as threats evolve
10. **Document** allowed origins

### ❌ Don'ts:

1. **Don't allow** `*` for CORS in production
2. **Don't use** `unsafe-inline` in CSP unless necessary
3. **Don't disable** XSS protection
4. **Don't forget** to set secure flags on cookies
5. **Don't expose** internal headers to clients
6. **Don't skip** HSTS on HTTPS sites

---

## Interview Questions

### Q1: What is CORS and why is it needed?

**Answer**: Cross-Origin Resource Sharing allows controlled access to resources from different origins:

- **Same-Origin Policy**: Browsers block cross-origin requests by default
- **CORS headers**: Server explicitly allows specific origins
- **Preflight**: OPTIONS request checks if actual request allowed
- **Use case**: Frontend (frontend.com) calling API (api.backend.com)
  Security feature preventing unauthorized access to resources.

### Q2: Difference between CSP and X-XSS-Protection?

**Answer**:

- **CSP**: Modern, flexible, controls all resource loading
- **X-XSS-Protection**: Legacy, browser-specific XSS filter
- **CSP advantages**: Fine-grained control, violation reporting
- **X-XSS-Protection**: Simple but limited
  Use CSP for new applications, X-XSS-Protection for legacy support.

### Q3: What is HSTS and why use it?

**Answer**: HTTP Strict Transport Security forces HTTPS:

- **Forces HTTPS**: Browser automatically upgrades HTTP to HTTPS
- **Prevents downgrade**: Blocks man-in-the-middle protocol downgrade
- **Preload list**: Can submit to browsers' HSTS preload list
- **Subdomains**: Include all subdomains with includeSubDomains
  Essential for HTTPS-only sites.

### Q4: How to allow multiple origins in CORS?

**Answer**:

- **Not allowed**: Access-Control-Allow-Origin doesn't accept multiple values
- **Option 1**: Echo back request Origin if in allowed list
- **Option 2**: Use regex patterns for subdomains
- **Don't**: Use \* with credentials
  Validate origin dynamically and echo back if allowed.

### Q5: What headers prevent clickjacking?

**Answer**:

- **X-Frame-Options**: DENY or SAMEORIGIN (legacy)
- **CSP frame-ancestors**: More flexible, modern approach
- **Use both**: For maximum compatibility
- **DENY**: Cannot be framed at all
- **SAMEORIGIN**: Only same origin can frame
  Prefer CSP frame-ancestors for new apps.

---

## Summary

CORS and security headers essentials:

- **CORS**: Control cross-origin access explicitly
- **HSTS**: Force HTTPS connections
- **CSP**: Prevent XSS and control resources
- **X-Frame-Options**: Prevent clickjacking
- **nosniff**: Prevent MIME sniffing
- **Referrer-Policy**: Control referrer info
- **Test**: Use security scanners

Secure your app from day one! 🔐
