# Django REST Framework Permissions

## 📖 Concept Explanation

Permissions in DRF determine whether a request should be granted or denied access. They run after authentication, checking if the authenticated user (or anonymous user) has permission to perform the requested action.

### Authorization Flow

```
Request → Authentication → Permissions → Throttling → View
                           ↓
                    Allow or Deny Access
```

**Purpose**:

- Control who can access resources
- Implement fine-grained access control
- Protect sensitive operations
- Enforce business rules

### Permission vs Authentication

```
Authentication: Who are you? (request.user)
Permissions:    Can you do this? (True/False)
```

## 🧠 Built-in Permission Classes

### 1. AllowAny (Default)

```python
from rest_framework.permissions import AllowAny

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [AllowAny]  # Public access
```

### 2. IsAuthenticated

```python
from rest_framework.permissions import IsAuthenticated

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticated]  # Requires login

# Returns 401 Unauthorized if not authenticated
```

### 3. IsAuthenticatedOrReadOnly

```python
from rest_framework.permissions import IsAuthenticatedOrReadOnly

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]

    # GET, HEAD, OPTIONS: Anyone
    # POST, PUT, PATCH, DELETE: Authenticated users only
```

### 4. IsAdminUser

```python
from rest_framework.permissions import IsAdminUser

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAdminUser]  # Only staff users

    # Checks: user.is_staff == True
```

### 5. DjangoModelPermissions

```python
from rest_framework.permissions import DjangoModelPermissions

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [DjangoModelPermissions]

    # Maps to Django model permissions:
    # - view_post (GET list/detail)
    # - add_post (POST)
    # - change_post (PUT/PATCH)
    # - delete_post (DELETE)
```

### 6. DjangoObjectPermissions

```python
from rest_framework.permissions import DjangoObjectPermissions

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [DjangoObjectPermissions]

    # Requires django-guardian for object-level permissions
    # Checks per-object permissions
```

## 🎯 Custom Permissions

### 1. Simple Custom Permission

```python
from rest_framework import permissions

class IsOwner(permissions.BasePermission):
    """
    Custom permission to only allow owners to edit.
    """

    def has_object_permission(self, request, view, obj):
        """
        Check if user is the owner.
        Called on detail views (retrieve, update, destroy).
        """
        return obj.author == request.user

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticated, IsOwner]
```

### 2. Read-Only or Owner

```python
class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Allow read access to anyone, write access to owner only.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions: GET, HEAD, OPTIONS
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions: only owner
        return obj.author == request.user

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
```

### 3. Permission with Message

```python
class IsOwner(permissions.BasePermission):
    """
    Custom permission with custom error message.
    """
    message = "You must be the owner to perform this action."

    def has_object_permission(self, request, view, obj):
        return obj.author == request.user
```

### 4. View-Level Permission

```python
class IsAdminOrReadOnly(permissions.BasePermission):
    """
    Check permission at view level.
    """

    def has_permission(self, request, view):
        """
        Called before view executes.
        For list/create actions (no object).
        """
        # Read permissions
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions: admin only
        return request.user and request.user.is_staff
```

## 🏗️ Advanced Permission Patterns

### 1. Different Permissions per Action

```python
from rest_framework import viewsets

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get_permissions(self):
        """
        Return different permissions based on action.
        """
        if self.action in ['list', 'retrieve']:
            # Anyone can view
            permission_classes = [AllowAny]
        elif self.action == 'create':
            # Must be authenticated to create
            permission_classes = [IsAuthenticated]
        elif self.action in ['update', 'partial_update', 'destroy']:
            # Must be owner or admin to edit/delete
            permission_classes = [IsAuthenticated, IsOwnerOrAdmin]
        else:
            permission_classes = [IsAdminUser]

        return [permission() for permission in permission_classes]
```

### 2. Combining Permissions

```python
class IsOwnerOrAdmin(permissions.BasePermission):
    """
    Allow access to owner or admin.
    """

    def has_object_permission(self, request, view, obj):
        # Admin has full access
        if request.user.is_staff:
            return True

        # Owner has access
        return obj.author == request.user

class IsPublishedOrOwner(permissions.BasePermission):
    """
    Published posts: anyone
    Unpublished posts: owner only
    """

    def has_object_permission(self, request, view, obj):
        if obj.published:
            return True
        return obj.author == request.user

# Use both permissions (AND logic)
class PostViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwnerOrAdmin, IsPublishedOrOwner]
```

### 3. Custom OR Logic

```python
from rest_framework.permissions import BasePermission

class IsOwnerOrAdmin(BasePermission):
    """
    Custom OR logic: owner OR admin.
    """

    def has_object_permission(self, request, view, obj):
        # Check if admin
        if request.user.is_staff:
            return True

        # Check if owner
        if hasattr(obj, 'author') and obj.author == request.user:
            return True

        return False
```

### 4. Time-Based Permissions

```python
from django.utils import timezone
from datetime import timedelta

class CanEditRecent(permissions.BasePermission):
    """
    Allow editing only within 24 hours of creation.
    """

    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True

        # Check if created within 24 hours
        time_threshold = timezone.now() - timedelta(hours=24)
        return obj.created_at >= time_threshold
```

## 🎯 Object-Level Permissions

### 1. Basic Object Permission

```python
from rest_framework import viewsets, permissions

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def check_object_permissions(self, request, obj):
        """
        Override to add custom object-level checks.
        """
        # Call parent
        super().check_object_permissions(request, obj)

        # Additional checks
        if obj.status == 'locked' and not request.user.is_staff:
            self.permission_denied(
                request,
                message='This post is locked'
            )
```

### 2. Django Guardian Integration

```python
# Install: pip install django-guardian

# settings.py
INSTALLED_APPS = [
    ...
    'guardian',
]

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'guardian.backends.ObjectPermissionBackend',
]

# models.py
from django.db import models
from guardian.shortcuts import assign_perm

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(User, on_delete=models.CASCADE)

# Assign permission
post = Post.objects.create(title='My Post', author=user)
assign_perm('view_post', user, post)
assign_perm('change_post', user, post)

# Check permission
from guardian.shortcuts import get_perms
perms = get_perms(user, post)
# ['view_post', 'change_post']

# views.py
from rest_framework.permissions import DjangoObjectPermissions

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [DjangoObjectPermissions]

    # Automatically checks object permissions
```

### 3. Custom Object Permission Backend

```python
class PostObjectPermission(permissions.BasePermission):
    """
    Custom object-level permission.
    """

    def has_object_permission(self, request, view, obj):
        # Public posts: anyone can view
        if request.method in permissions.SAFE_METHODS:
            if obj.visibility == 'public':
                return True
            elif obj.visibility == 'private':
                return obj.author == request.user

        # Edit permissions
        if request.method in ['PUT', 'PATCH']:
            # Owner or collaborator
            return (obj.author == request.user or
                    request.user in obj.collaborators.all())

        # Delete permissions
        if request.method == 'DELETE':
            # Owner only
            return obj.author == request.user

        return False
```

## ✅ Best Practices

### 1. Fail Securely (Deny by Default)

```python
class MyPermission(permissions.BasePermission):
    def has_permission(self, request, view):
        # Start with deny
        allowed = False

        # Explicitly grant access
        if request.user.is_authenticated:
            allowed = True

        return allowed
```

### 2. Log Permission Denials

```python
import logging

logger = logging.getLogger(__name__)

class LoggedPermission(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        allowed = obj.author == request.user

        if not allowed:
            logger.warning(
                f"Permission denied: {request.user} tried to access {obj} "
                f"in view {view.__class__.__name__}"
            )

        return allowed
```

### 3. Separate Concerns

```python
# Good: Separate permission classes
class IsOwner(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.author == request.user

class IsPublished(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.published

# Use together
permission_classes = [IsOwner, IsPublished]
```

## 🔐 Security Best Practices

### 1. Always Check Object Ownership

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticated]

    def update(self, request, *args, **kwargs):
        """Always verify ownership before update."""
        instance = self.get_object()

        # Explicit check
        if instance.author != request.user and not request.user.is_staff:
            return Response(
                {'error': 'Permission denied'},
                status=403
            )

        return super().update(request, *args, **kwargs)
```

### 2. Rate Limit Sensitive Operations

```python
from rest_framework.throttling import UserRateThrottle

class SensitiveOperationThrottle(UserRateThrottle):
    rate = '10/hour'

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get_throttles(self):
        if self.action in ['create', 'update', 'destroy']:
            return [SensitiveOperationThrottle()]
        return []
```

### 3. Audit Trail

```python
class AuditedPermission(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        allowed = obj.author == request.user

        # Log all access attempts
        AuditLog.objects.create(
            user=request.user,
            action=request.method,
            resource=f"{obj.__class__.__name__}:{obj.pk}",
            allowed=allowed,
            ip_address=request.META.get('REMOTE_ADDR')
        )

        return allowed
```

## ⚡ Performance Optimization

### 1. Prefetch Related Data

```python
class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    permission_classes = [IsOwnerOrReadOnly]

    def get_queryset(self):
        """Prefetch author for permission checks."""
        return Post.objects.select_related('author').all()
```

### 2. Cache Permission Checks

```python
from django.core.cache import cache

class CachedPermission(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        cache_key = f"perm_{request.user.id}_{obj.pk}"

        # Check cache
        allowed = cache.get(cache_key)
        if allowed is not None:
            return allowed

        # Compute permission
        allowed = obj.author == request.user

        # Cache result
        cache.set(cache_key, allowed, 300)  # 5 minutes

        return allowed
```

## ❓ Interview Questions

### Q1: What's the difference between has_permission and has_object_permission?

**Answer**:

- **has_permission**: View-level, runs on all requests (list, create)
- **has_object_permission**: Object-level, runs on detail views (retrieve, update, destroy)

### Q2: How do you implement "owner or admin" permission?

**Answer**:

```python
def has_object_permission(self, request, view, obj):
    return request.user.is_staff or obj.author == request.user
```

### Q3: When should you use DjangoModelPermissions?

**Answer**:
When you want to leverage Django's built-in permission system (add/change/delete/view). Good for admin-like interfaces with granular permissions.

## 📚 Summary

**Key Takeaways**:

1. AllowAny for public endpoints
2. IsAuthenticated for protected endpoints
3. IsAuthenticatedOrReadOnly for read-public, write-protected
4. Custom permissions with BasePermission
5. has_permission for view-level checks
6. has_object_permission for object-level checks
7. Use get_permissions() for action-specific permissions
8. Combine multiple permissions (AND logic)
9. Prefetch related data for performance
10. Always deny by default (fail securely)

DRF permissions provide fine-grained access control!
