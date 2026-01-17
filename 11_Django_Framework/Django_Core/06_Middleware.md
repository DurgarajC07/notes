# Django Middleware

## 📖 Concept Explanation

Middleware is a framework of hooks into Django's request/response processing. It's a lightweight plugin system for globally altering Django's input or output.

### Middleware Flow

```
Browser Request
    ↓
MIDDLEWARE (process_request)
    ↓
URL Routing
    ↓
View
    ↓
MIDDLEWARE (process_response)
    ↓
Browser Response
```

**Process**:

1. **Request Phase**: Middleware processes request before view (top to bottom)
2. **View Execution**: View processes request
3. **Response Phase**: Middleware processes response after view (bottom to top)

### Middleware Order (Django Settings)

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    # Custom middleware
    'myapp.middleware.RequestLoggingMiddleware',
]
```

## 🧠 Built-in Middleware

### 1. SecurityMiddleware

```python
# Adds security-related headers
# - X-Content-Type-Options: nosniff
# - X-Frame-Options: DENY
# - Strict-Transport-Security (HSTS)
# - Redirects HTTP to HTTPS

# settings.py
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_CONTENT_TYPE_NOSNIFF = True
```

### 2. SessionMiddleware

```python
# Enables session support
# Stores session data in database, cache, or cookies

# Usage in view
def my_view(request):
    request.session['user_id'] = 123
    request.session.set_expiry(3600)  # Expire in 1 hour
```

### 3. AuthenticationMiddleware

```python
# Adds `request.user` attribute
# Associates users with requests using sessions

# Usage in view
def my_view(request):
    if request.user.is_authenticated:
        return HttpResponse(f"Hello, {request.user.username}")
```

### 4. CsrfViewMiddleware

```python
# Protects against Cross-Site Request Forgery attacks
# Requires {% csrf_token %} in POST forms
```

## 🏗️ Custom Middleware

### 1. Class-Based Middleware (Recommended)

```python
# myapp/middleware.py
import time
import logging

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware:
    """Log all incoming requests"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Code executed before view
        start_time = time.time()

        # Log request
        logger.info(f"Request: {request.method} {request.path}")

        # Call the next middleware or view
        response = self.get_response(request)

        # Code executed after view
        duration = time.time() - start_time
        logger.info(f"Response: {response.status_code} ({duration:.2f}s)")

        return response

    def process_exception(self, request, exception):
        """Handle exceptions"""
        logger.error(f"Exception: {exception} on {request.path}")
        return None  # Let Django handle it
```

### 2. Request/Response Modification

```python
class CustomHeaderMiddleware:
    """Add custom headers to response"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Modify request
        request.custom_attribute = "value"

        response = self.get_response(request)

        # Add custom headers
        response['X-Custom-Header'] = 'MyValue'
        response['X-Request-ID'] = request.META.get('HTTP_X_REQUEST_ID', 'N/A')

        return response
```

### 3. Authentication Middleware

```python
from django.http import JsonResponse
from django.contrib.auth.models import AnonymousUser
import jwt

class JWTAuthenticationMiddleware:
    """Custom JWT authentication middleware"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Extract token from Authorization header
        auth_header = request.META.get('HTTP_AUTHORIZATION', '')

        if auth_header.startswith('Bearer '):
            token = auth_header.split(' ')[1]

            try:
                # Decode JWT token
                payload = jwt.decode(token, settings.SECRET_KEY, algorithms=['HS256'])
                user_id = payload.get('user_id')

                # Attach user to request
                from django.contrib.auth import get_user_model
                User = get_user_model()
                request.user = User.objects.get(pk=user_id)

            except jwt.ExpiredSignatureError:
                return JsonResponse({'error': 'Token expired'}, status=401)
            except jwt.InvalidTokenError:
                return JsonResponse({'error': 'Invalid token'}, status=401)
            except User.DoesNotExist:
                request.user = AnonymousUser()
        else:
            request.user = AnonymousUser()

        response = self.get_response(request)
        return response
```

### 4. Rate Limiting Middleware

```python
from django.core.cache import cache
from django.http import HttpResponse

class RateLimitMiddleware:
    """Rate limit requests per IP"""

    def __init__(self, get_response):
        self.get_response = get_response
        self.rate_limit = 100  # requests
        self.period = 3600     # per hour

    def __call__(self, request):
        # Get client IP
        ip = self.get_client_ip(request)

        # Create cache key
        cache_key = f'rate_limit:{ip}'

        # Get current count
        count = cache.get(cache_key, 0)

        if count >= self.rate_limit:
            return HttpResponse('Rate limit exceeded', status=429)

        # Increment count
        cache.set(cache_key, count + 1, self.period)

        response = self.get_response(request)

        # Add rate limit headers
        response['X-RateLimit-Limit'] = str(self.rate_limit)
        response['X-RateLimit-Remaining'] = str(self.rate_limit - count - 1)

        return response

    def get_client_ip(self, request):
        """Extract client IP from request"""
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            ip = x_forwarded_for.split(',')[0]
        else:
            ip = request.META.get('REMOTE_ADDR')
        return ip
```

### 5. Request Timing Middleware

```python
import time

class RequestTimingMiddleware:
    """Measure request processing time"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Start timer
        request._start_time = time.time()

        response = self.get_response(request)

        # Calculate duration
        duration = time.time() - request._start_time

        # Add timing header
        response['X-Request-Duration'] = f'{duration:.3f}s'

        # Log slow requests
        if duration > 1.0:  # More than 1 second
            logger.warning(f'Slow request: {request.path} took {duration:.2f}s')

        return response
```

## 🎯 Advanced Middleware Patterns

### 1. Conditional Middleware

```python
class MaintenanceModeMiddleware:
    """Block access during maintenance"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Check maintenance mode
        if settings.MAINTENANCE_MODE:
            # Allow staff to access
            if request.user.is_staff:
                response = self.get_response(request)
                response['X-Maintenance-Mode'] = 'Staff Access'
                return response

            # Show maintenance page to others
            return HttpResponse(
                '<h1>Site Under Maintenance</h1>',
                status=503
            )

        return self.get_response(request)
```

### 2. CORS Middleware

```python
class CORSMiddleware:
    """Handle Cross-Origin Resource Sharing"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Handle preflight OPTIONS request
        if request.method == 'OPTIONS':
            response = HttpResponse()
            response['Access-Control-Allow-Origin'] = '*'
            response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
            response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
            response['Access-Control-Max-Age'] = '3600'
            return response

        response = self.get_response(request)

        # Add CORS headers to response
        response['Access-Control-Allow-Origin'] = '*'
        response['Access-Control-Allow-Credentials'] = 'true'

        return response
```

### 3. Database Router Middleware

```python
class DatabaseRouterMiddleware:
    """Route requests to different databases"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Use read replica for GET requests
        if request.method == 'GET':
            request._db_hint = 'read_replica'
        else:
            request._db_hint = 'default'

        response = self.get_response(request)
        return response
```

## ✅ Best Practices

### 1. Middleware Order Matters

```python
# Correct order
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',  # First: security
    'django.contrib.sessions.middleware.SessionMiddleware',  # Enable sessions
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',  # After sessions
    'django.contrib.auth.middleware.AuthenticationMiddleware',  # After sessions
    'django.contrib.messages.middleware.MessageMiddleware',  # After sessions
    'myapp.middleware.RequestLoggingMiddleware',  # Custom logging
]
```

### 2. Avoid Heavy Processing

```python
# BAD: Heavy database query in middleware
class BadMiddleware:
    def __call__(self, request):
        users = User.objects.all()  # Runs for EVERY request!
        response = self.get_response(request)
        return response

# GOOD: Only when needed
class GoodMiddleware:
    def __call__(self, request):
        if request.path.startswith('/admin/'):
            # Only for admin pages
            users = User.objects.filter(is_staff=True).only('id', 'username')

        response = self.get_response(request)
        return response
```

## 🔐 Security Middleware

### 1. IP Whitelist Middleware

```python
from django.http import HttpResponseForbidden

class IPWhitelistMiddleware:
    """Allow only whitelisted IPs"""

    def __init__(self, get_response):
        self.get_response = get_response
        self.whitelist = settings.IP_WHITELIST

    def __call__(self, request):
        ip = self.get_client_ip(request)

        # Check if IP is whitelisted
        if ip not in self.whitelist and not request.user.is_staff:
            return HttpResponseForbidden('Access denied')

        return self.get_response(request)

    def get_client_ip(self, request):
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            return x_forwarded_for.split(',')[0]
        return request.META.get('REMOTE_ADDR')
```

### 2. Request Validation Middleware

```python
import json

class RequestValidationMiddleware:
    """Validate incoming requests"""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Validate Content-Type for POST/PUT
        if request.method in ['POST', 'PUT']:
            content_type = request.META.get('CONTENT_TYPE', '')

            if request.path.startswith('/api/'):
                if 'application/json' not in content_type:
                    return JsonResponse(
                        {'error': 'Content-Type must be application/json'},
                        status=400
                    )

        response = self.get_response(request)
        return response
```

## ⚡ Performance Optimization

### 1. Caching Middleware

```python
from django.core.cache import cache

class CacheMiddleware:
    """Cache GET responses"""

    def __init__(self, get_response):
        self.get_response = get_response
        self.cache_timeout = 300  # 5 minutes

    def __call__(self, request):
        # Only cache GET requests
        if request.method != 'GET':
            return self.get_response(request)

        # Create cache key
        cache_key = f'page:{request.path}:{request.GET.urlencode()}'

        # Try to get from cache
        cached_response = cache.get(cache_key)
        if cached_response:
            cached_response['X-Cache'] = 'HIT'
            return cached_response

        # Generate response
        response = self.get_response(request)

        # Cache successful responses
        if response.status_code == 200:
            cache.set(cache_key, response, self.cache_timeout)
            response['X-Cache'] = 'MISS'

        return response
```

## ❓ Interview Questions

### Q1: What is middleware in Django?

**Answer**:
Middleware is a hook system that processes requests before views and responses after views. It runs globally for all requests.

### Q2: What is the difference between process_request and **call**?

**Answer**:
Modern Django uses `__call__()` method. Old-style middleware used separate `process_request()` and `process_response()` methods.

### Q3: Why does middleware order matter?

**Answer**:
Middleware runs top-to-bottom for requests, bottom-to-top for responses. Order affects behavior (e.g., SessionMiddleware must run before AuthenticationMiddleware).

## 📚 Summary

**Key Takeaways**:

1. Middleware processes all requests/responses globally
2. Order matters: top-to-bottom (request), bottom-to-top (response)
3. Use middleware for cross-cutting concerns (logging, auth, headers)
4. Avoid heavy processing in middleware
5. Built-in middleware handles security, sessions, auth, CSRF
6. Custom middleware: logging, rate limiting, CORS, caching
7. Return None to pass control to next middleware
8. Return HttpResponse to short-circuit request

Middleware is powerful for global request/response processing!
