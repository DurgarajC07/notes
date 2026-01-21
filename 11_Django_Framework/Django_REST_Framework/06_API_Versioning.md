# API Versioning in Django REST Framework

## 📖 Concept Overview

API versioning allows you to make breaking changes to your API while maintaining backward compatibility with existing clients. It's essential for evolving APIs without disrupting current users.

## 🧠 Why It Matters

- **Backward Compatibility**: Support multiple client versions simultaneously
- **Gradual Migration**: Allow clients to upgrade at their own pace
- **Breaking Changes**: Introduce new features without breaking existing apps
- **Deprecation Strategy**: Retire old versions gracefully
- **Client Flexibility**: Different clients can use different versions

## ⚙️ DRF Versioning Schemes

Django REST Framework provides 5 built-in versioning schemes:

1. **URLPathVersioning** - Version in URL path (`/v1/users/`)
2. **NamespaceVersioning** - Version in URL namespace
3. **HostNameVersioning** - Version in hostname (`v1.api.example.com`)
4. **QueryParameterVersioning** - Version in query param (`?version=v1`)
5. **AcceptHeaderVersioning** - Version in Accept header

## 🧪 URLPathVersioning (Recommended)

### Configuration

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2', 'v3'],
    'VERSION_PARAM': 'version',
}
```

### URL Configuration

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import UserViewSet, ProductViewSet

# Router for versioned API
router = DefaultRouter()
router.register(r'users', UserViewSet, basename='user')
router.register(r'products', ProductViewSet, basename='product')

urlpatterns = [
    # Versioned endpoints
    path('api/v1/', include(router.urls)),
    path('api/v2/', include(router.urls)),
    path('api/v3/', include(router.urls)),

    # Or using a more flexible pattern
    path('api/<str:version>/', include(router.urls)),
]
```

### Accessing Version in Views

```python
# views.py
from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.decorators import action
from .models import User
from .serializers import UserSerializerV1, UserSerializerV2, UserSerializerV3

class UserViewSet(viewsets.ModelViewSet):
    """User API with versioning support"""

    queryset = User.objects.all()

    def get_serializer_class(self):
        """Return different serializer based on version"""
        if self.request.version == 'v1':
            return UserSerializerV1
        elif self.request.version == 'v2':
            return UserSerializerV2
        elif self.request.version == 'v3':
            return UserSerializerV3
        return UserSerializerV1  # Default

    def list(self, request, *args, **kwargs):
        """List users with version-specific logic"""
        queryset = self.get_queryset()

        # Version-specific filtering
        if request.version == 'v1':
            queryset = queryset.filter(is_active=True)
        elif request.version in ['v2', 'v3']:
            # v2+ includes inactive users
            queryset = queryset.all()

        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)

    @action(detail=False, methods=['get'])
    def stats(self, request):
        """Endpoint added in v2"""
        if request.version == 'v1':
            return Response(
                {'error': 'This endpoint is only available in v2+'},
                status=status.HTTP_404_NOT_FOUND
            )

        stats = {
            'total_users': User.objects.count(),
            'active_users': User.objects.filter(is_active=True).count(),
        }
        return Response(stats)
```

### Version-Specific Serializers

```python
# serializers.py
from rest_framework import serializers
from .models import User

class UserSerializerV1(serializers.ModelSerializer):
    """V1: Basic user info"""

    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'created_at']
        read_only_fields = ['id', 'created_at']

class UserSerializerV2(serializers.ModelSerializer):
    """V2: Added full_name and phone"""

    full_name = serializers.CharField(source='get_full_name', read_only=True)

    class Meta:
        model = User
        fields = [
            'id', 'username', 'email', 'full_name',
            'phone', 'is_active', 'created_at'
        ]
        read_only_fields = ['id', 'created_at']

class UserSerializerV3(serializers.ModelSerializer):
    """V3: Restructured response format"""

    profile = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = ['id', 'username', 'profile', 'created_at']
        read_only_fields = ['id', 'created_at']

    def get_profile(self, obj):
        """Nested profile structure (breaking change from v2)"""
        return {
            'email': obj.email,
            'full_name': obj.get_full_name(),
            'phone': obj.phone,
            'is_active': obj.is_active,
        }
```

## 🎯 NamespaceVersioning

```python
# urls.py
from django.urls import path, include

urlpatterns = [
    path('api/', include(('api.v1.urls', 'v1'), namespace='v1')),
    path('api/', include(('api.v2.urls', 'v2'), namespace='v2')),
]

# api/v1/urls.py
from django.urls import path
from .views import user_list, user_detail

app_name = 'api_v1'

urlpatterns = [
    path('users/', user_list, name='user-list'),
    path('users/<int:pk>/', user_detail, name='user-detail'),
]

# api/v2/urls.py (similar structure)

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.NamespaceVersioning',
}
```

## 🌐 AcceptHeaderVersioning

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.AcceptHeaderVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2', 'v3'],
}

# Client request headers:
# Accept: application/json; version=v1
# Accept: application/json; version=v2

# views.py
class UserViewSet(viewsets.ModelViewSet):
    def list(self, request):
        # Access version from Accept header
        version = request.version
        # ... version-specific logic
```

## 🔧 QueryParameterVersioning

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.QueryParameterVersioning',
    'VERSION_PARAM': 'version',
}

# Client requests:
# GET /api/users/?version=v1
# GET /api/users/?version=v2
```

## 🏗️ Real-World Example: E-commerce API

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Product(models.Model):
    name = models.CharField(max_length=200)
    description = models.TextField()
    price = models.DecimalField(max_digits=10, decimal_places=2)

    # Added in v2
    discount_price = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        null=True,
        blank=True
    )

    # Added in v3
    stock = models.IntegerField(default=0)
    categories = models.ManyToManyField('Category')

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

# serializers.py
from rest_framework import serializers
from .models import Product

class ProductSerializerV1(serializers.ModelSerializer):
    """V1: Basic product info"""

    class Meta:
        model = Product
        fields = ['id', 'name', 'description', 'price', 'created_at']

class ProductSerializerV2(serializers.ModelSerializer):
    """V2: Added discount pricing"""

    final_price = serializers.SerializerMethodField()
    has_discount = serializers.SerializerMethodField()

    class Meta:
        model = Product
        fields = [
            'id', 'name', 'description', 'price',
            'discount_price', 'final_price', 'has_discount',
            'created_at', 'updated_at'
        ]

    def get_final_price(self, obj):
        return obj.discount_price if obj.discount_price else obj.price

    def get_has_discount(self, obj):
        return obj.discount_price is not None

class ProductSerializerV3(serializers.ModelSerializer):
    """V3: Added inventory and categories"""

    pricing = serializers.SerializerMethodField()
    inventory = serializers.SerializerMethodField()
    category_names = serializers.SerializerMethodField()

    class Meta:
        model = Product
        fields = [
            'id', 'name', 'description', 'pricing',
            'inventory', 'category_names', 'created_at'
        ]

    def get_pricing(self, obj):
        """Nested pricing structure"""
        return {
            'regular_price': str(obj.price),
            'discount_price': str(obj.discount_price) if obj.discount_price else None,
            'final_price': str(obj.discount_price if obj.discount_price else obj.price),
            'has_discount': obj.discount_price is not None,
        }

    def get_inventory(self, obj):
        """Inventory information"""
        return {
            'in_stock': obj.stock > 0,
            'quantity': obj.stock,
            'status': 'available' if obj.stock > 0 else 'out_of_stock',
        }

    def get_category_names(self, obj):
        return [cat.name for cat in obj.categories.all()]

# views.py
from rest_framework import viewsets, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from django_filters.rest_framework import DjangoFilterBackend

class ProductViewSet(viewsets.ModelViewSet):
    """Product API with versioning"""

    queryset = Product.objects.all()
    filter_backends = [DjangoFilterBackend, filters.SearchFilter]
    filterset_fields = ['price']
    search_fields = ['name', 'description']

    def get_serializer_class(self):
        """Version-specific serializers"""
        version_serializers = {
            'v1': ProductSerializerV1,
            'v2': ProductSerializerV2,
            'v3': ProductSerializerV3,
        }
        return version_serializers.get(
            self.request.version,
            ProductSerializerV1
        )

    def get_queryset(self):
        """Version-specific queryset optimization"""
        queryset = super().get_queryset()

        # V3 needs categories prefetched
        if self.request.version == 'v3':
            queryset = queryset.prefetch_related('categories')

        return queryset

    @action(detail=False, methods=['get'])
    def discounted(self, request):
        """Get discounted products (v2+)"""
        if request.version == 'v1':
            return Response(
                {'error': 'This endpoint requires API v2 or higher'},
                status=404
            )

        products = self.get_queryset().filter(
            discount_price__isnull=False
        )
        serializer = self.get_serializer(products, many=True)
        return Response(serializer.data)

    @action(detail=True, methods=['get'])
    def availability(self, request, pk=None):
        """Check product availability (v3+)"""
        if request.version in ['v1', 'v2']:
            return Response(
                {'error': 'This endpoint requires API v3 or higher'},
                status=404
            )

        product = self.get_object()
        return Response({
            'product_id': product.id,
            'in_stock': product.stock > 0,
            'quantity': product.stock,
            'can_order': product.stock > 0,
        })
```

## 🔄 Deprecation Strategy

```python
# views.py
from rest_framework.response import Response
from rest_framework import status
import warnings

class DeprecatedMixin:
    """Mixin to handle deprecated API versions"""

    deprecated_versions = ['v1']
    sunset_date = '2024-12-31'

    def initial(self, request, *args, **kwargs):
        super().initial(request, *args, **kwargs)

        if request.version in self.deprecated_versions:
            # Add deprecation header
            self.headers = {
                'Warning': f'299 - "API version {request.version} is deprecated"',
                'Sunset': self.sunset_date,
                'Link': '<https://api.example.com/docs/migration>; rel="deprecation"',
            }

class UserViewSet(DeprecatedMixin, viewsets.ModelViewSet):
    """User API with deprecation warnings"""

    deprecated_versions = ['v1']
    sunset_date = '2024-12-31'

    queryset = User.objects.all()

    def list(self, request, *args, **kwargs):
        response = super().list(request, *args, **kwargs)

        # Add deprecation headers
        if hasattr(self, 'headers'):
            for key, value in self.headers.items():
                response[key] = value

        return response
```

## 📝 Migration Documentation

```python
# Create migration guide endpoint
from rest_framework.views import APIView
from rest_framework.response import Response

class MigrationGuideView(APIView):
    """API migration guide"""

    permission_classes = []  # Public

    def get(self, request):
        guide = {
            'current_version': 'v3',
            'supported_versions': ['v2', 'v3'],
            'deprecated_versions': {
                'v1': {
                    'sunset_date': '2024-12-31',
                    'replacement': 'v2',
                    'breaking_changes': [
                        'User.full_name field added',
                        'Inactive users now included in list endpoint',
                    ]
                }
            },
            'v2_to_v3_changes': [
                'Product response restructured with nested pricing and inventory',
                'Categories added to products',
                'New /availability endpoint for products',
            ],
            'migration_steps': {
                'v1_to_v2': [
                    'Update User serialization to handle full_name field',
                    'Handle inactive users in list responses',
                    'Use /stats endpoint for user statistics',
                ],
                'v2_to_v3': [
                    'Update Product response parsing for nested structure',
                    'Handle new inventory fields',
                    'Use /availability endpoint for stock checks',
                ]
            },
            'documentation_url': 'https://api.example.com/docs',
        }
        return Response(guide)

# urls.py
urlpatterns = [
    path('api/migration-guide/', MigrationGuideView.as_view()),
]
```

## ✅ Best Practices

### 1. Semantic Versioning

```python
# ✅ Good: Clear version numbers
'v1', 'v2', 'v3'

# ✅ Also good: Date-based
'2024-01-01', '2024-06-01'

# ❌ Bad: Unclear versioning
'latest', 'stable', 'beta'
```

### 2. Version in URL Path (Recommended)

```python
# ✅ Good: Clear and visible
/api/v1/users/
/api/v2/users/

# ❌ Less discoverable
/api/users/?version=v1  # Query parameter
Accept: application/json; version=v1  # Header
```

### 3. Maintain Backward Compatibility

```python
# ✅ Good: Additive changes in minor versions
class UserSerializerV1_1(UserSerializerV1):
    # Add optional field
    phone = serializers.CharField(required=False)

# ❌ Bad: Breaking changes in minor versions
# Don't remove or rename fields in v1.1
```

### 4. Clear Deprecation Timeline

```python
# ✅ Good: Clear deprecation path
DEPRECATION_POLICY = {
    'v1': {
        'deprecated_date': '2024-01-01',
        'sunset_date': '2024-06-01',  # 6 months notice
        'migration_guide': '/docs/v1-to-v2',
    }
}
```

## ❌ Common Mistakes

### 1. Too Many Versions

```python
# ❌ Bad: Version explosion
/api/v1/, /api/v1.1/, /api/v1.2/, /api/v2/, /api/v2.1/...

# ✅ Good: Major versions only
/api/v1/, /api/v2/, /api/v3/
```

### 2. Inconsistent Versioning

```python
# ❌ Bad: Different versioning for different endpoints
/api/v1/users/
/api/v2/products/
/api/v1/orders/

# ✅ Good: Consistent versioning
/api/v2/users/
/api/v2/products/
/api/v2/orders/
```

### 3. No Migration Path

```python
# ❌ Bad: Just deprecate without guidance
"v1 is deprecated, use v2"

# ✅ Good: Provide clear migration path
MIGRATION_GUIDE = {
    'breaking_changes': [...],
    'code_examples': {...},
    'timeline': '6 months',
}
```

## 🔐 Security Considerations

```python
# Prevent version confusion attacks
class VersionSecurityMixin:
    """Ensure version is validated"""

    def initial(self, request, *args, **kwargs):
        super().initial(request, *args, **kwargs)

        # Validate version
        allowed_versions = settings.REST_FRAMEWORK.get('ALLOWED_VERSIONS', [])
        if request.version not in allowed_versions:
            raise ValidationError(
                f"Invalid API version: {request.version}"
            )

        # Log version usage for monitoring
        logger.info(
            f"API request: version={request.version}, "
            f"endpoint={request.path}, user={request.user}"
        )
```

## ❓ Interview Questions

### Q1: What are the main API versioning strategies?

**Answer**:

1. **URL Path**: `/api/v1/users/` - Most visible, recommended
2. **Query Parameter**: `/api/users/?version=v1` - Less discoverable
3. **Header**: `Accept: application/json; version=v1` - Clean URLs
4. **Hostname**: `v1.api.example.com` - Infrastructure overhead

**Best**: URL Path (clear, discoverable, cache-friendly)

### Q2: When should you create a new API version?

**Answer**:
Create new version for **breaking changes**:

- Removing fields
- Renaming fields
- Changing data types
- Changing response structure
- Removing endpoints

**Don't version for**:

- Adding optional fields
- Adding new endpoints
- Bug fixes
- Performance improvements

### Q3: How do you handle API deprecation?

**Answer**:

1. **Announce early**: 6-12 months notice
2. **Add deprecation headers**: Sunset, Warning, Link
3. **Provide migration guide**: Clear documentation
4. **Monitor usage**: Track old version usage
5. **Gradual sunset**: Remove after timeline

```python
Response headers:
Warning: 299 - "API v1 deprecated"
Sunset: 2024-12-31
Link: <https://api.example.com/migration>; rel="deprecation"
```

### Q4: How do you test multiple API versions?

**Answer**:

```python
# Test both versions
class ProductAPITestCase(APITestCase):
    def test_v1_response(self):
        response = self.client.get('/api/v1/products/')
        self.assertIn('price', response.data[0])
        self.assertNotIn('discount_price', response.data[0])

    def test_v2_response(self):
        response = self.client.get('/api/v2/products/')
        self.assertIn('discount_price', response.data[0])
```

## 📚 Summary

### Key Takeaways

1. **URL Path versioning is recommended** for clarity
2. **Version only for breaking changes** not additions
3. **Maintain backward compatibility** as long as possible
4. **Provide clear deprecation timeline** (6-12 months)
5. **Version-specific serializers** keep code organized
6. **Migration guides** essential for clients
7. **Monitor usage** to know when to sunset versions

### Versioning Checklist

- ✅ Use URLPathVersioning for new APIs
- ✅ Set DEFAULT_VERSION and ALLOWED_VERSIONS
- ✅ Create version-specific serializers
- ✅ Handle version logic in views
- ✅ Add deprecation warnings early
- ✅ Provide migration documentation
- ✅ Test all supported versions
- ✅ Monitor version usage
- ✅ Have clear sunset policy

### Version Lifecycle

1. **Active**: Current production version
2. **Supported**: Previous version, still maintained
3. **Deprecated**: Warning headers, sunset date announced
4. **Sunset**: No longer available, returns 410 Gone
