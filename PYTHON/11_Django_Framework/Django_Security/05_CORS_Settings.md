# Django CORS Settings

## 📖 Concept Explanation

Cross-Origin Resource Sharing (CORS) is a security mechanism that allows or restricts resources requested from another domain outside the domain from which the resource originated.

### Same-Origin Policy

```
Same Origin:
https://example.com/page1 → https://example.com/page2 ✅

Different Origin (Blocked by default):
https://example.com → https://api.example.com ❌ (different subdomain)
https://example.com → http://example.com ❌ (different protocol)
https://example.com:443 → https://example.com:8000 ❌ (different port)
```

**CORS Purpose**:

- Relax same-origin policy selectively
- Allow frontend (example.com) to call API (api.example.com)
- Protect users from malicious cross-origin requests

## 🧠 Django CORS Headers

### 1. Installation

```bash
pip install django-cors-headers
```

### 2. Basic Configuration

```python
# settings.py
INSTALLED_APPS = [
    #...
    'corsheaders',
    #...
]

MIDDLEWARE = [
    # CORS middleware should be before CommonMiddleware
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    #...
]

# Allow all origins (Development only!)
CORS_ALLOW_ALL_ORIGINS = True  # DON'T USE IN PRODUCTION!

# Production: Whitelist specific origins
CORS_ALLOWED_ORIGINS = [
    "https://example.com",
    "https://www.example.com",
    "https://app.example.com",
]

# Alternative: Allow origins matching regex
CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://\w+\.example\.com$",  # All subdomains
]
```

### 3. CORS Headers Configuration

```python
# settings.py

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

# Allowed headers
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

# Expose headers to JavaScript
CORS_EXPOSE_HEADERS = [
    'content-type',
    'x-csrf-token',
]

# Cache preflight requests (seconds)
CORS_PREFLIGHT_MAX_AGE = 86400  # 24 hours
```

## 🎯 CORS Request Flow

### 1. Simple Request

```javascript
// Frontend (https://app.example.com)
fetch('https://api.example.com/posts/', {
    method: 'GET',
    headers: {
        'Content-Type': 'application/json'
    }
})
.then(response => response.json())
.then(data => console.log(data));

// Browser sends:
GET /posts/ HTTP/1.1
Host: api.example.com
Origin: https://app.example.com

// Django responds:
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Content-Type: application/json

// Browser: Origin allowed, request succeeds
```

### 2. Preflight Request

```javascript
// Frontend: Request with custom header (triggers preflight)
fetch('https://api.example.com/posts/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-Custom-Header': 'value'
    },
    body: JSON.stringify({title: 'Test'})
});

// Step 1: Browser sends OPTIONS request (preflight)
OPTIONS /posts/ HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type, x-custom-header

// Step 2: Django responds
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: content-type, x-custom-header
Access-Control-Max-Age: 86400

// Step 3: Browser sends actual POST request (if preflight succeeds)
POST /posts/ HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Content-Type: application/json

{"title": "Test"}
```

## 🔐 Security Configurations

### 1. Production CORS Settings

```python
# settings.py (production)

# Never allow all origins in production!
CORS_ALLOW_ALL_ORIGINS = False

# Whitelist specific origins
CORS_ALLOWED_ORIGINS = [
    "https://example.com",
    "https://www.example.com",
    "https://app.example.com",
]

# Allow credentials only if needed
CORS_ALLOW_CREDENTIALS = True

# Restrict methods
CORS_ALLOW_METHODS = [
    'GET',
    'POST',
    'PUT',
    'PATCH',
    'DELETE',
    'OPTIONS',
]

# Only allow necessary headers
CORS_ALLOW_HEADERS = [
    'accept',
    'authorization',
    'content-type',
    'x-csrftoken',
]
```

### 2. Environment-Based Configuration

```python
# settings.py
import os

# Development
if DEBUG:
    CORS_ALLOWED_ORIGINS = [
        "http://localhost:3000",
        "http://localhost:8080",
        "http://127.0.0.1:3000",
    ]
else:
    # Production
    CORS_ALLOWED_ORIGINS = os.environ.get('CORS_ORIGINS', '').split(',')
    # Set in environment: CORS_ORIGINS=https://example.com,https://www.example.com
```

### 3. Per-View CORS

```python
# Whitelist CORS for specific views
from corsheaders.signals import check_request_enabled

def cors_allow_api_to_everyone(sender, request, **kwargs):
    """Allow CORS for public API endpoints"""
    return request.path.startswith('/api/public/')

check_request_enabled.connect(cors_allow_api_to_everyone)
```

## 🏗️ CORS with Authentication

### 1. Token Authentication (No Credentials)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ]
}

CORS_ALLOW_CREDENTIALS = False  # Tokens in headers, no cookies needed

# JavaScript
fetch('https://api.example.com/posts/', {
    method: 'GET',
    headers: {
        'Authorization': 'Token abc123xyz789'
    }
});
```

### 2. Session Authentication (With Credentials)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ]
}

CORS_ALLOW_CREDENTIALS = True  # Allow cookies
CORS_ALLOWED_ORIGINS = [
    "https://app.example.com",  # Must be specific (not *)
]

# Session cookie settings
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = 'None'  # Required for cross-origin cookies

# CSRF settings
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_SAMESITE = 'None'  # Required for cross-origin
CSRF_TRUSTED_ORIGINS = [
    'https://app.example.com',
]

# JavaScript
fetch('https://api.example.com/posts/', {
    method: 'POST',
    credentials: 'include',  // Include cookies
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': getCookie('csrftoken')
    },
    body: JSON.stringify({title: 'Test'})
});
```

### 3. JWT Authentication (Recommended for CORS)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ]
}

CORS_ALLOW_CREDENTIALS = False  # JWT in headers, no cookies
CORS_ALLOWED_ORIGINS = [
    "https://app.example.com",
]

# JavaScript
const token = localStorage.getItem('access_token');

fetch('https://api.example.com/posts/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({title: 'Test'})
});
```

## 🎯 Advanced CORS Patterns

### 1. Custom CORS Middleware

```python
# middleware.py
class CustomCORSMiddleware:
    """
    Custom CORS middleware with additional logic.
    """
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        origin = request.META.get('HTTP_ORIGIN')

        # Check if origin is allowed
        allowed = self.is_origin_allowed(origin)

        response = self.get_response(request)

        if allowed:
            response['Access-Control-Allow-Origin'] = origin
            response['Access-Control-Allow-Credentials'] = 'true'
            response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
            response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'

        # Handle preflight
        if request.method == 'OPTIONS':
            response.status_code = 200

        return response

    def is_origin_allowed(self, origin):
        """Check if origin is allowed"""
        allowed_origins = [
            'https://example.com',
            'https://app.example.com',
        ]

        # Check exact match
        if origin in allowed_origins:
            return True

        # Check regex pattern
        import re
        if re.match(r'^https://.*\.example\.com$', origin):
            return True

        return False
```

### 2. Dynamic Origin Validation

```python
# settings.py
def cors_origin_whitelist():
    """
    Dynamic CORS origin list based on database.
    """
    from myapp.models import TrustedOrigin
    return list(TrustedOrigin.objects.values_list('origin', flat=True))

CORS_ORIGIN_WHITELIST = cors_origin_whitelist()

# Or use regex
CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://[\w-]+\.example\.com$",  # All subdomains
    r"^https://example\.com$",          # Main domain
]
```

### 3. CORS with Multiple Environments

```python
# settings.py
import os

ENVIRONMENT = os.environ.get('ENVIRONMENT', 'development')

if ENVIRONMENT == 'production':
    CORS_ALLOWED_ORIGINS = [
        "https://example.com",
        "https://www.example.com",
    ]
elif ENVIRONMENT == 'staging':
    CORS_ALLOWED_ORIGINS = [
        "https://staging.example.com",
        "https://staging-app.example.com",
    ]
else:  # development
    CORS_ALLOW_ALL_ORIGINS = True  # Only for local dev!
```

## 🛡️ Security Best Practices

### 1. Never Use Wildcard with Credentials

```python
# Bad: Security risk!
CORS_ALLOW_ALL_ORIGINS = True
CORS_ALLOW_CREDENTIALS = True
# This combination allows ANY origin to send credentials!

# Good: Specific origins only
CORS_ALLOWED_ORIGINS = ["https://app.example.com"]
CORS_ALLOW_CREDENTIALS = True
```

### 2. Validate Origin Strictly

```python
# Bad: Too permissive
CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://.*$",  # Allows ALL HTTPS sites!
]

# Good: Specific pattern
CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://[\w-]+\.example\.com$",  # Only example.com subdomains
]
```

### 3. Limit Exposed Headers

```python
# Only expose necessary headers
CORS_EXPOSE_HEADERS = [
    'Content-Type',
    'X-Total-Count',  # Pagination
]

# Don't expose sensitive headers
# Never: 'Set-Cookie', 'Authorization', 'X-API-Key'
```

## 🧪 Testing CORS

### 1. Test CORS Configuration

```python
# tests.py
from django.test import TestCase

class CORSTestCase(TestCase):
    def test_cors_allowed_origin(self):
        """Test CORS with allowed origin"""
        response = self.client.options(
            '/api/posts/',
            HTTP_ORIGIN='https://app.example.com',
            HTTP_ACCESS_CONTROL_REQUEST_METHOD='POST'
        )

        self.assertEqual(response.status_code, 200)
        self.assertEqual(
            response['Access-Control-Allow-Origin'],
            'https://app.example.com'
        )

    def test_cors_disallowed_origin(self):
        """Test CORS with disallowed origin"""
        response = self.client.options(
            '/api/posts/',
            HTTP_ORIGIN='https://malicious.com',
            HTTP_ACCESS_CONTROL_REQUEST_METHOD='POST'
        )

        # Should not have CORS headers
        self.assertNotIn('Access-Control-Allow-Origin', response)
```

### 2. Manual CORS Testing

```bash
# Test preflight request
curl -X OPTIONS \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type" \
  https://api.example.com/posts/

# Expected response headers:
# Access-Control-Allow-Origin: https://app.example.com
# Access-Control-Allow-Methods: POST, GET, OPTIONS
# Access-Control-Allow-Headers: Content-Type
```

## ⚡ Performance Optimization

### 1. Cache Preflight Requests

```python
# settings.py
CORS_PREFLIGHT_MAX_AGE = 86400  # 24 hours

# Browser caches preflight response for 24 hours
# Reduces OPTIONS requests significantly
```

### 2. Optimize CORS Middleware Order

```python
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # First
    'django.middleware.common.CommonMiddleware',  # After CORS
    # ...
]
```

## ❓ Interview Questions

### Q1: What's the difference between CORS_ALLOW_ALL_ORIGINS and CORS_ALLOWED_ORIGINS?

**Answer**:

- `CORS_ALLOW_ALL_ORIGINS=True`: Allows ANY origin (wildcard `*`)
- `CORS_ALLOWED_ORIGINS`: Whitelist of specific origins

Never use `ALLOW_ALL` with `ALLOW_CREDENTIALS=True` in production.

### Q2: Why do browsers send OPTIONS requests?

**Answer**:
Preflight requests check if the actual request is safe to send. Triggered by:

- Custom headers
- Methods other than GET/HEAD/POST
- Content-Type other than application/x-www-form-urlencoded, multipart/form-data, or text/plain

### Q3: Can you use CORS with session authentication?

**Answer**:
Yes, but requires:

1. `CORS_ALLOW_CREDENTIALS = True`
2. Specific origins (not wildcard)
3. `SESSION_COOKIE_SAMESITE = 'None'`
4. `credentials: 'include'` in fetch
5. CSRF token handling

JWT is simpler for CORS.

## 📚 Summary

**Key Takeaways**:

1. Install django-cors-headers for CORS support
2. Whitelist specific origins in production
3. Never use ALLOW_ALL with ALLOW_CREDENTIALS
4. Use JWT for cross-origin APIs (simpler than session)
5. Set PREFLIGHT_MAX_AGE for performance
6. SESSION_COOKIE_SAMESITE='None' for cross-origin sessions
7. Add CSRF_TRUSTED_ORIGINS for session auth
8. Validate origins strictly (no overly broad regex)
9. Only expose necessary headers
10. Test CORS with different origins

CORS is essential for modern API development!
