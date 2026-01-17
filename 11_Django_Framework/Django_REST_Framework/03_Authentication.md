# Django REST Framework Authentication

## 📖 Concept Explanation

Authentication in DRF identifies who is making the request. It runs before permissions and throttling, attaching `request.user` and `request.auth` to the request object.

### Authentication Flow

```
Request → Authentication → Permissions → View
         ↓
   Sets request.user
   Sets request.auth
```

**Purpose**:

- Identify the user making the request
- Verify credentials
- Set user context
- Enable authorization decisions

### Authentication vs Authorization

```
Authentication: Who are you? (Identity)
Authorization:  What can you do? (Permissions)
```

## 🧠 Built-in Authentication Classes

### 1. SessionAuthentication

Uses Django's session framework. Best for browser-based clients (same-origin).

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ]
}

# views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    authentication_classes = [SessionAuthentication]
    permission_classes = [IsAuthenticated]

# Usage: Browser sends session cookie automatically
# GET /api/posts/
# Cookie: sessionid=abc123
```

**Pros**:

- Automatic with Django sessions
- No token management
- Works with Django admin

**Cons**:

- CSRF protection required
- Not ideal for mobile/SPA apps
- Stateful (server-side sessions)

### 2. BasicAuthentication

HTTP Basic Auth with username:password encoded in Base64. Use only over HTTPS.

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',
    ]
}

# Usage:
# GET /api/posts/
# Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
# (username:password in Base64)
```

**Pros**:

- Simple to implement
- Supported by all HTTP clients

**Cons**:

- Credentials sent with every request
- Must use HTTPS
- No logout mechanism
- Not recommended for production APIs

### 3. TokenAuthentication

Simple token-based auth. Token stored in database.

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework.authtoken',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ]
}

# Run migrations
# python manage.py migrate

# views.py
from rest_framework.authtoken.views import obtain_auth_token
from rest_framework.authtoken.models import Token

# URL for obtaining token
urlpatterns = [
    path('api/token/', obtain_auth_token, name='api_token_auth'),
]

# Usage:
# 1. Obtain token
# POST /api/token/
# {"username": "john", "password": "secret"}
# Response: {"token": "9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"}

# 2. Use token
# GET /api/posts/
# Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```

**Token Generation**:

```python
# Create token for user
from rest_framework.authtoken.models import Token

# On user creation
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(user=instance)

# Manual token creation
token = Token.objects.create(user=user)
print(token.key)

# Get or create token
token, created = Token.objects.get_or_create(user=user)
```

**Pros**:

- Simple and fast
- Stateless
- Easy to implement

**Cons**:

- Single token per user
- No expiration
- No refresh mechanism
- Token stored in database (query overhead)

## 🔐 JWT Authentication (Recommended)

JSON Web Tokens are self-contained, stateless tokens with expiration.

### 1. Installation

```bash
pip install djangorestframework-simplejwt
```

### 2. Configuration

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework_simplejwt',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'UPDATE_LAST_LOGIN': True,

    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'VERIFYING_KEY': None,
    'AUDIENCE': None,
    'ISSUER': None,

    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',

    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),
    'TOKEN_TYPE_CLAIM': 'token_type',

    'JTI_CLAIM': 'jti',
}
```

### 3. URL Configuration

```python
# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

### 4. Usage Flow

```python
# 1. Obtain tokens
# POST /api/token/
# {"username": "john", "password": "secret"}
# Response:
# {
#     "access": "eyJ0eXAiOiJKV1QiLCJhbGc...",
#     "refresh": "eyJ0eXAiOiJKV1QiLCJhbGc..."
# }

# 2. Use access token
# GET /api/posts/
# Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGc...

# 3. Refresh access token when expired
# POST /api/token/refresh/
# {"refresh": "eyJ0eXAiOiJKV1QiLCJhbGc..."}
# Response:
# {
#     "access": "eyJ0eXAiOiJKV1QiLCJhbGc..."
# }
```

### 5. Custom JWT Claims

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)

        # Add custom claims
        token['username'] = user.username
        token['email'] = user.email
        token['is_staff'] = user.is_staff

        return token

class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer

# urls.py
urlpatterns = [
    path('api/token/', CustomTokenObtainPairView.as_view(), name='token_obtain_pair'),
]
```

## 🎯 OAuth2 Authentication

OAuth2 for third-party authentication (Google, Facebook, GitHub).

### 1. Installation

```bash
pip install django-oauth-toolkit
```

### 2. Configuration

```python
# settings.py
INSTALLED_APPS = [
    ...
    'oauth2_provider',
    'rest_framework',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
    ],
}

OAUTH2_PROVIDER = {
    'SCOPES': {
        'read': 'Read scope',
        'write': 'Write scope',
    },
    'ACCESS_TOKEN_EXPIRE_SECONDS': 3600,  # 1 hour
    'REFRESH_TOKEN_EXPIRE_SECONDS': 86400,  # 1 day
}

# urls.py
urlpatterns = [
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
]
```

### 3. Application Registration

```python
# Create OAuth2 application
from oauth2_provider.models import Application

app = Application.objects.create(
    name='My API Client',
    user=user,
    client_type=Application.CLIENT_CONFIDENTIAL,
    authorization_grant_type=Application.GRANT_PASSWORD,
)

print(f"Client ID: {app.client_id}")
print(f"Client Secret: {app.client_secret}")
```

### 4. Usage

```python
# 1. Obtain token
# POST /o/token/
# {
#     "grant_type": "password",
#     "username": "john",
#     "password": "secret",
#     "client_id": "YOUR_CLIENT_ID",
#     "client_secret": "YOUR_CLIENT_SECRET"
# }
# Response:
# {
#     "access_token": "abc123",
#     "token_type": "Bearer",
#     "expires_in": 3600,
#     "refresh_token": "def456",
#     "scope": "read write"
# }

# 2. Use access token
# GET /api/posts/
# Authorization: Bearer abc123

# 3. Refresh token
# POST /o/token/
# {
#     "grant_type": "refresh_token",
#     "refresh_token": "def456",
#     "client_id": "YOUR_CLIENT_ID",
#     "client_secret": "YOUR_CLIENT_SECRET"
# }
```

## 🏗️ Custom Authentication

### 1. Custom Authentication Class

```python
from rest_framework import authentication
from rest_framework.exceptions import AuthenticationFailed
from django.contrib.auth.models import User

class APIKeyAuthentication(authentication.BaseAuthentication):
    """
    Custom API Key authentication.
    Header: X-API-Key: abc123
    """

    def authenticate(self, request):
        api_key = request.META.get('HTTP_X_API_KEY')

        if not api_key:
            return None  # No authentication attempted

        try:
            # Lookup user by API key
            profile = UserProfile.objects.select_related('user').get(api_key=api_key)
            return (profile.user, None)  # (user, auth)
        except UserProfile.DoesNotExist:
            raise AuthenticationFailed('Invalid API Key')

    def authenticate_header(self, request):
        """
        Return WWW-Authenticate header for 401 responses.
        """
        return 'X-API-Key'

# Usage
class PostViewSet(viewsets.ModelViewSet):
    authentication_classes = [APIKeyAuthentication]
    permission_classes = [IsAuthenticated]
```

### 2. Multiple Authentication

```python
from rest_framework.authentication import SessionAuthentication, BasicAuthentication
from rest_framework_simplejwt.authentication import JWTAuthentication

class PostViewSet(viewsets.ModelViewSet):
    """
    Try multiple authentication methods.
    First successful auth is used.
    """
    authentication_classes = [
        JWTAuthentication,
        SessionAuthentication,
        APIKeyAuthentication,
    ]
    permission_classes = [IsAuthenticated]
```

### 3. Token with Expiration

```python
import datetime
from django.utils import timezone
from rest_framework import authentication
from rest_framework.exceptions import AuthenticationFailed

class ExpiringTokenAuthentication(authentication.TokenAuthentication):
    """
    Token authentication with expiration.
    """

    def authenticate_credentials(self, key):
        try:
            token = self.get_model().objects.select_related('user').get(key=key)
        except self.get_model().DoesNotExist:
            raise AuthenticationFailed('Invalid token')

        if not token.user.is_active:
            raise AuthenticationFailed('User inactive or deleted')

        # Check if token expired
        utc_now = timezone.now()
        if token.created < utc_now - datetime.timedelta(hours=24):
            raise AuthenticationFailed('Token has expired')

        return (token.user, token)
```

## ✅ Best Practices

### 1. Use HTTPS in Production

```python
# settings.py (production)
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

### 2. Short Access Token Lifetime

```python
# JWT settings
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),  # Short-lived
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),     # Longer-lived
    'ROTATE_REFRESH_TOKENS': True,
}
```

### 3. Token Blacklisting

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework_simplejwt.token_blacklist',
]

# Logout view
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework_simplejwt.tokens import RefreshToken

class LogoutView(APIView):
    def post(self, request):
        try:
            refresh_token = request.data["refresh"]
            token = RefreshToken(refresh_token)
            token.blacklist()  # Add to blacklist
            return Response({"message": "Logout successful"})
        except Exception as e:
            return Response({"error": str(e)}, status=400)
```

## 🔐 Security Best Practices

### 1. Rate Limiting Login Attempts

```python
from rest_framework.throttling import AnonRateThrottle

class LoginRateThrottle(AnonRateThrottle):
    rate = '5/hour'  # 5 attempts per hour

class LoginView(TokenObtainPairView):
    throttle_classes = [LoginRateThrottle]
```

### 2. Strong Password Requirements

```python
# settings.py
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 12}
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]
```

### 3. Secure Token Storage

```python
# Client-side (JavaScript)
// Store in httpOnly cookie (backend-controlled)
// OR
// Store in memory (lost on refresh)
const token = await getToken();

// DON'T store in localStorage (XSS vulnerable)
// localStorage.setItem('token', token);  // BAD!
```

## ⚡ Performance Optimization

### 1. Cache User Lookups

```python
from django.core.cache import cache

class CachedJWTAuthentication(JWTAuthentication):
    def get_user(self, validated_token):
        user_id = validated_token['user_id']

        # Check cache
        cache_key = f'user_{user_id}'
        user = cache.get(cache_key)

        if user is None:
            # Fetch from database
            user = super().get_user(validated_token)
            cache.set(cache_key, user, 300)  # Cache 5 minutes

        return user
```

## ❓ Interview Questions

### Q1: JWT vs Token Authentication - when to use each?

**Answer**:

- **JWT**: Stateless, self-contained, no database lookup, shorter lifespan, refresh tokens
- **Token Auth**: Stateful, database lookup, single token, no expiration

Use JWT for microservices, mobile apps, high-scale APIs.

### Q2: How do you handle token refresh in JWT?

**Answer**:
Use refresh tokens. Access token (15 min), refresh token (7 days). When access expires, use refresh to get new access. Rotate refresh tokens for security.

### Q3: What are the security risks of storing JWT in localStorage?

**Answer**:
XSS attacks can steal tokens. Prefer httpOnly cookies or memory storage with refresh flow.

## 📚 Summary

**Key Takeaways**:

1. SessionAuthentication for browser apps
2. JWT for mobile/SPA apps
3. OAuth2 for third-party auth
4. Short access token lifetime (15 min)
5. Use refresh tokens for renewals
6. Always use HTTPS in production
7. Rate limit login endpoints
8. Blacklist tokens on logout
9. Cache user lookups for performance
10. Never store tokens in localStorage

DRF authentication is flexible and powerful!
