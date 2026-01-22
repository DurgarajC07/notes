# Django REST Framework Pagination & Filtering

## 📖 Pagination Concept

Pagination breaks large result sets into manageable pages. Essential for performance and user experience when dealing with thousands of records.

### Why Pagination Matters

```
Without Pagination:
GET /api/posts/ → Returns 10,000 posts (slow, memory-heavy)

With Pagination:
GET /api/posts/?page=1 → Returns 20 posts (fast, efficient)
```

**Benefits**:

- Reduced server memory usage
- Faster response times
- Better client-side rendering
- Lower database load

## 🧠 Pagination Classes

### 1. PageNumberPagination

Standard page-based pagination with page numbers.

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20
}

# Custom pagination class
from rest_framework.pagination import PageNumberPagination

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'  # Allow client to override
    max_page_size = 100
    page_query_param = 'page'

# views.py
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    pagination_class = StandardResultsSetPagination

# Request:
# GET /api/posts/?page=2&page_size=50

# Response:
# {
#     "count": 1000,
#     "next": "http://api.example.com/api/posts/?page=3&page_size=50",
#     "previous": "http://api.example.com/api/posts/?page=1&page_size=50",
#     "results": [
#         {...},
#         {...}
#     ]
# }
```

**Pros**:

- Simple and intuitive
- Jump to any page
- Shows total count

**Cons**:

- Inefficient for large datasets (OFFSET queries)
- Results can shift between pages if data changes

### 2. LimitOffsetPagination

Database-style pagination with offset and limit.

```python
from rest_framework.pagination import LimitOffsetPagination

class CustomLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 20
    limit_query_param = 'limit'
    offset_query_param = 'offset'
    max_limit = 100

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    pagination_class = CustomLimitOffsetPagination

# Request:
# GET /api/posts/?limit=20&offset=40
# (Skip 40 records, return next 20)

# Response:
# {
#     "count": 1000,
#     "next": "http://api.example.com/api/posts/?limit=20&offset=60",
#     "previous": "http://api.example.com/api/posts/?limit=20&offset=20",
#     "results": [...]
# }
```

**Pros**:

- Flexible offset control
- Familiar to SQL users

**Cons**:

- Slow for large offsets
- Same shifting issue as PageNumber

### 3. CursorPagination (Recommended for Large Datasets)

Cursor-based pagination using opaque cursor tokens. Most efficient for large datasets.

```python
from rest_framework.pagination import CursorPagination

class PostCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # Must specify ordering
    cursor_query_param = 'cursor'

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    pagination_class = PostCursorPagination

# Request:
# GET /api/posts/

# Response:
# {
#     "next": "http://api.example.com/api/posts/?cursor=cD0yMDIw...",
#     "previous": null,
#     "results": [...]
# }

# Next page:
# GET /api/posts/?cursor=cD0yMDIw...
```

**Pros**:

- Constant-time lookups (no OFFSET)
- No result shifting
- Best for infinite scroll

**Cons**:

- Can't jump to specific page
- No total count
- Must have consistent ordering

### 4. Custom Pagination

```python
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response

class CustomPagination(PageNumberPagination):
    page_size = 20

    def get_paginated_response(self, data):
        """Custom response format"""
        return Response({
            'links': {
                'next': self.get_next_link(),
                'previous': self.get_previous_link()
            },
            'count': self.page.paginator.count,
            'total_pages': self.page.paginator.num_pages,
            'current_page': self.page.number,
            'results': data
        })

# Response:
# {
#     "links": {
#         "next": "...",
#         "previous": "..."
#     },
#     "count": 1000,
#     "total_pages": 50,
#     "current_page": 2,
#     "results": [...]
# }
```

## 🔍 Filtering

### 1. Basic Filtering with DjangoFilterBackend

```bash
# Install
pip install django-filter
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'django_filters',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend'
    ]
}

# views.py
from django_filters.rest_framework import DjangoFilterBackend

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['author', 'category', 'published', 'created_at']

# Usage:
# GET /api/posts/?author=1
# GET /api/posts/?published=true
# GET /api/posts/?category=tech&published=true
```

### 2. Advanced Filtering with FilterSet

```python
import django_filters
from django_filters import rest_framework as filters

class PostFilter(filters.FilterSet):
    """Custom filter for posts"""

    # Exact match
    author = filters.NumberFilter(field_name='author__id')

    # Case-insensitive contains
    title = filters.CharFilter(field_name='title', lookup_expr='icontains')

    # Date range
    created_after = filters.DateTimeFilter(field_name='created_at', lookup_expr='gte')
    created_before = filters.DateTimeFilter(field_name='created_at', lookup_expr='lte')

    # Range
    views_min = filters.NumberFilter(field_name='views', lookup_expr='gte')
    views_max = filters.NumberFilter(field_name='views', lookup_expr='lte')

    # Choice field
    status = filters.ChoiceFilter(choices=[('draft', 'Draft'), ('published', 'Published')])

    # Boolean
    is_featured = filters.BooleanFilter(field_name='featured')

    # Related field
    category = filters.CharFilter(field_name='category__slug', lookup_expr='iexact')

    # Multiple choice
    tags = filters.ModelMultipleChoiceFilter(
        field_name='tags__slug',
        to_field_name='slug',
        queryset=Tag.objects.all(),
    )

    class Meta:
        model = Post
        fields = ['author', 'title', 'status', 'category', 'tags']

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = PostFilter

# Usage:
# GET /api/posts/?title=django
# GET /api/posts/?created_after=2024-01-01&created_before=2024-12-31
# GET /api/posts/?views_min=1000&views_max=5000
# GET /api/posts/?tags=python&tags=django
```

### 3. Custom Filter Method

```python
class PostFilter(filters.FilterSet):
    """Filter with custom method"""

    search = filters.CharFilter(method='filter_search')

    def filter_search(self, queryset, name, value):
        """Search in title and content"""
        return queryset.filter(
            Q(title__icontains=value) | Q(content__icontains=value)
        )

    author_name = filters.CharFilter(method='filter_author_name')

    def filter_author_name(self, queryset, name, value):
        """Filter by author username"""
        return queryset.filter(author__username__icontains=value)

    class Meta:
        model = Post
        fields = ['search', 'author_name']

# Usage:
# GET /api/posts/?search=python
# GET /api/posts/?author_name=john
```

## 🔎 Searching

### 1. SearchFilter

```python
from rest_framework import filters

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [filters.SearchFilter]
    search_fields = ['title', 'content', 'author__username']

# Usage:
# GET /api/posts/?search=django
# Searches in title, content, and author username
```

### 2. Advanced Search

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [filters.SearchFilter]

    # Search operators:
    search_fields = [
        'title',              # Partial match (default)
        '=email',             # Exact match
        '^title',             # Starts with
        '@content',           # Full-text search (PostgreSQL)
        '$slug',              # Regex search
    ]

# Usage:
# GET /api/posts/?search=django
```

### 3. Multiple Filter Backends

```python
from rest_framework import filters
from django_filters.rest_framework import DjangoFilterBackend

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    # Combine multiple backends
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
        filters.OrderingFilter,
    ]

    # Filtering
    filterset_fields = ['author', 'category', 'published']

    # Searching
    search_fields = ['title', 'content']

    # Ordering
    ordering_fields = ['created_at', 'views', 'title']
    ordering = ['-created_at']  # Default ordering

# Usage:
# GET /api/posts/?author=1&search=django&ordering=-views
```

## 📊 Ordering

### 1. OrderingFilter

```python
from rest_framework import filters

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['created_at', 'views', 'title', 'author__username']
    ordering = ['-created_at']  # Default

# Usage:
# GET /api/posts/?ordering=views        # Ascending
# GET /api/posts/?ordering=-views       # Descending
# GET /api/posts/?ordering=-views,title # Multiple fields
```

### 2. Custom Ordering

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get_queryset(self):
        queryset = Post.objects.all()

        # Custom ordering logic
        ordering = self.request.query_params.get('ordering')

        if ordering == 'popularity':
            # Custom calculated ordering
            queryset = queryset.annotate(
                popularity=Count('likes') + Count('comments')
            ).order_by('-popularity')
        elif ordering == 'trending':
            # Last 7 days activity
            week_ago = timezone.now() - timedelta(days=7)
            queryset = queryset.filter(created_at__gte=week_ago) \
                               .order_by('-views')

        return queryset

# Usage:
# GET /api/posts/?ordering=popularity
# GET /api/posts/?ordering=trending
```

## 🎯 Advanced Patterns

### 1. Dynamic Fields with Filtering

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [DjangoFilterBackend]

    def get_serializer(self, *args, **kwargs):
        """Allow client to specify fields"""
        fields = self.request.query_params.get('fields')

        if fields:
            kwargs['fields'] = fields.split(',')

        return super().get_serializer(*args, **kwargs)

# Usage:
# GET /api/posts/?fields=id,title,author
```

### 2. Conditional Filtering

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get_queryset(self):
        queryset = Post.objects.all()
        user = self.request.user

        # Filter based on user
        if not user.is_authenticated:
            # Anonymous: only published
            queryset = queryset.filter(published=True)
        elif not user.is_staff:
            # Regular users: published or own posts
            queryset = queryset.filter(
                Q(published=True) | Q(author=user)
            )
        # Staff: see everything

        return queryset
```

### 3. Aggregation with Filtering

```python
from django.db.models import Count, Avg
from rest_framework.decorators import action
from rest_framework.response import Response

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['author', 'category']

    @action(detail=False, methods=['get'])
    def statistics(self, request):
        """Get statistics with current filters"""
        # Apply filters
        queryset = self.filter_queryset(self.get_queryset())

        # Aggregate
        stats = queryset.aggregate(
            total=Count('id'),
            avg_views=Avg('views'),
            published=Count('id', filter=Q(published=True)),
            drafts=Count('id', filter=Q(published=False))
        )

        return Response(stats)

# Usage:
# GET /api/posts/statistics/?author=1
# {
#     "total": 50,
#     "avg_views": 1234.5,
#     "published": 45,
#     "drafts": 5
# }
```

## ✅ Best Practices

### 1. Always Use Pagination

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# Prevent unpaginated responses by default
```

### 2. Index Filtered Fields

```python
# models.py
class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE, db_index=True)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL,
                                 null=True, db_index=True)
    published = models.BooleanField(default=False, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)

    class Meta:
        indexes = [
            models.Index(fields=['published', 'created_at']),
            models.Index(fields=['author', 'published']),
        ]
```

### 3. Limit Filterset Fields

```python
# Don't expose all fields for filtering
class PostFilter(filters.FilterSet):
    class Meta:
        model = Post
        fields = ['author', 'category', 'published']  # Limited set
        # Don't use: fields = '__all__'  # Security risk!
```

## 🔐 Security Best Practices

### 1. Validate Filter Values

```python
class PostFilter(filters.FilterSet):
    author = filters.NumberFilter(method='filter_author')

    def filter_author(self, queryset, name, value):
        """Validate author ID"""
        if value < 1:
            return queryset.none()

        return queryset.filter(author_id=value)
```

### 2. Limit Max Page Size

```python
class StandardPagination(PageNumberPagination):
    page_size = 20
    max_page_size = 100  # Prevent client from requesting too many
```

## ⚡ Performance Optimization

### 1. Optimize Queryset for Filters

```python
class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer

    def get_queryset(self):
        """Optimize based on filters"""
        queryset = Post.objects.all()

        # If filtering by author, prefetch
        if 'author' in self.request.query_params:
            queryset = queryset.select_related('author')

        # If searching, optimize text search
        if 'search' in self.request.query_params:
            queryset = queryset.select_related('author').only(
                'id', 'title', 'content', 'author__username'
            )

        return queryset
```

## ❓ Interview Questions

### Q1: CursorPagination vs PageNumberPagination - when to use each?

**Answer**:

- **CursorPagination**: Large datasets, infinite scroll, constant-time performance
- **PageNumberPagination**: Small/medium datasets, need page jumping, total count

### Q2: How do you prevent N+1 queries with filtering?

**Answer**:
Use select_related/prefetch_related in get_queryset() based on active filters.

### Q3: What's the security risk of exposing all model fields for filtering?

**Answer**:
Allows filtering on sensitive fields (e.g., internal_status, deleted_at), potential data leakage, and performance issues from complex queries.

## 📚 Summary

**Key Takeaways**:

1. Always use pagination for APIs
2. CursorPagination for large datasets
3. PageNumberPagination for general use
4. DjangoFilterBackend for field filtering
5. SearchFilter for text search
6. OrderingFilter for sorting
7. Index all filtered fields
8. Limit max_page_size
9. Validate filter inputs
10. Optimize querysets for filters

DRF pagination and filtering are essential for scalable APIs!
