# Django REST Framework ViewSets

## 📖 Concept Explanation

ViewSets combine the logic for a set of related views (list, create, retrieve, update, destroy) into a single class. They work with routers to automatically generate URL patterns.

### ViewSet vs APIView

```
APIView:        Manual URL mapping, explicit methods
ViewSet:        Automatic URL generation via router, action-based methods
```

**Benefits**:

- Less code duplication
- Automatic URL routing
- Standard REST patterns
- Easy to extend

### Basic ViewSet

```python
from rest_framework import viewsets
from .models import Post
from .serializers import PostSerializer

class PostViewSet(viewsets.ModelViewSet):
    """
    ViewSet for Post model.
    Provides list, create, retrieve, update, destroy actions.
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer

# urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')

urlpatterns = router.urls
```

## 🧠 ViewSet Types

### 1. ModelViewSet (Full CRUD)

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated

class PostViewSet(viewsets.ModelViewSet):
    """
    Full CRUD operations:
    - list (GET /posts/)
    - create (POST /posts/)
    - retrieve (GET /posts/{id}/)
    - update (PUT /posts/{id}/)
    - partial_update (PATCH /posts/{id}/)
    - destroy (DELETE /posts/{id}/)
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        """Filter queryset based on user"""
        user = self.request.user
        if user.is_staff:
            return Post.objects.all()
        return Post.objects.filter(author=user)

    def perform_create(self, serializer):
        """Set author on creation"""
        serializer.save(author=self.request.user)
```

### 2. ReadOnlyModelViewSet

```python
class PostViewSet(viewsets.ReadOnlyModelViewSet):
    """
    Read-only operations:
    - list (GET /posts/)
    - retrieve (GET /posts/{id}/)
    """
    queryset = Post.objects.filter(published=True)
    serializer_class = PostSerializer
```

### 3. ViewSet (Generic)

```python
from rest_framework import viewsets
from rest_framework.response import Response

class PostViewSet(viewsets.ViewSet):
    """
    Manual action definition.
    No default actions provided.
    """

    def list(self, request):
        """GET /posts/"""
        queryset = Post.objects.all()
        serializer = PostSerializer(queryset, many=True)
        return Response(serializer.data)

    def create(self, request):
        """POST /posts/"""
        serializer = PostSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data, status=201)

    def retrieve(self, request, pk=None):
        """GET /posts/{pk}/"""
        post = get_object_or_404(Post, pk=pk)
        serializer = PostSerializer(post)
        return Response(serializer.data)

    def update(self, request, pk=None):
        """PUT /posts/{pk}/"""
        post = get_object_or_404(Post, pk=pk)
        serializer = PostSerializer(post, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    def destroy(self, request, pk=None):
        """DELETE /posts/{pk}/"""
        post = get_object_or_404(Post, pk=pk)
        post.delete()
        return Response(status=204)
```

### 4. GenericViewSet with Mixins

```python
from rest_framework import viewsets, mixins

class PostViewSet(mixins.CreateModelMixin,
                  mixins.RetrieveModelMixin,
                  mixins.ListModelMixin,
                  viewsets.GenericViewSet):
    """
    Custom combination: list, create, retrieve only
    (no update or delete)
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

## 🎯 Custom Actions

### 1. @action Decorator

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """
        Custom action: POST /posts/{id}/publish/
        """
        post = self.get_object()
        post.published = True
        post.save()
        serializer = self.get_serializer(post)
        return Response(serializer.data)

    @action(detail=True, methods=['post'])
    def unpublish(self, request, pk=None):
        """
        Custom action: POST /posts/{id}/unpublish/
        """
        post = self.get_object()
        post.published = False
        post.save()
        serializer = self.get_serializer(post)
        return Response(serializer.data)

    @action(detail=False, methods=['get'])
    def recent(self, request):
        """
        Collection action: GET /posts/recent/
        """
        recent_posts = Post.objects.order_by('-created_at')[:10]
        serializer = self.get_serializer(recent_posts, many=True)
        return Response(serializer.data)

    @action(detail=False, methods=['get'])
    def popular(self, request):
        """
        Collection action: GET /posts/popular/
        """
        popular_posts = Post.objects.filter(views__gte=1000).order_by('-views')
        serializer = self.get_serializer(popular_posts, many=True)
        return Response(serializer.data)
```

### 2. Custom Permissions on Actions

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.permissions import IsAuthenticated, IsAdminUser

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def like(self, request, pk=None):
        """Like a post - requires authentication"""
        post = self.get_object()
        Like.objects.get_or_create(post=post, user=request.user)
        return Response({'status': 'liked'})

    @action(detail=True, methods=['post'], permission_classes=[IsAdminUser])
    def feature(self, request, pk=None):
        """Feature a post - admin only"""
        post = self.get_object()
        post.featured = True
        post.save()
        return Response({'status': 'featured'})
```

## 🏗️ Customizing ViewSets

### 1. Different Serializers per Action

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()

    def get_serializer_class(self):
        """Use different serializers for different actions"""
        if self.action == 'list':
            return PostListSerializer
        elif self.action == 'retrieve':
            return PostDetailSerializer
        elif self.action in ['create', 'update', 'partial_update']:
            return PostWriteSerializer
        return PostSerializer
```

### 2. Custom QuerySet per Action

```python
class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer

    def get_queryset(self):
        """Customize queryset based on action"""
        if self.action == 'list':
            # Optimize list query
            return Post.objects.select_related('author').prefetch_related('tags')[:20]
        elif self.action == 'retrieve':
            # Full details with all relations
            return Post.objects.select_related('author', 'category') \
                               .prefetch_related('tags', 'comments__author')

        # Default queryset
        return Post.objects.all()
```

### 3. Filtering and Searching

```python
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    # Filtering
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['published', 'author', 'category']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'views', 'title']
    ordering = ['-created_at']

# Usage:
# GET /posts/?published=true
# GET /posts/?search=django
# GET /posts/?ordering=-views
# GET /posts/?author=1&published=true
```

## 🎯 Advanced Patterns

### 1. Nested Routes

```python
# URLs for nested resources: /posts/{post_id}/comments/

class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer

    def get_queryset(self):
        """Filter comments by post"""
        post_id = self.kwargs.get('post_pk')
        return Comment.objects.filter(post_id=post_id)

    def perform_create(self, serializer):
        """Set post on creation"""
        post_id = self.kwargs.get('post_pk')
        serializer.save(post_id=post_id, author=self.request.user)

# urls.py
from rest_framework_nested import routers

router = routers.DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')

posts_router = routers.NestedDefaultRouter(router, r'posts', lookup='post')
posts_router.register(r'comments', CommentViewSet, basename='post-comments')

urlpatterns = [
    path('', include(router.urls)),
    path('', include(posts_router.urls)),
]
```

### 2. Pagination

```python
from rest_framework.pagination import PageNumberPagination

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    pagination_class = StandardResultsSetPagination

# Response:
# {
#     "count": 100,
#     "next": "http://api.example.com/posts/?page=2",
#     "previous": null,
#     "results": [...]
# }
```

### 3. Bulk Operations

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    @action(detail=False, methods=['post'])
    def bulk_create(self, request):
        """Bulk create posts"""
        serializer = self.get_serializer(data=request.data, many=True)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data, status=201)

    @action(detail=False, methods=['post'])
    def bulk_update(self, request):
        """Bulk update posts"""
        posts_data = request.data
        posts = []

        for post_data in posts_data:
            post = Post.objects.get(pk=post_data['id'])
            serializer = self.get_serializer(post, data=post_data, partial=True)
            serializer.is_valid(raise_exception=True)
            serializer.save()
            posts.append(serializer.data)

        return Response(posts)

    @action(detail=False, methods=['post'])
    def bulk_delete(self, request):
        """Bulk delete posts"""
        post_ids = request.data.get('ids', [])
        deleted_count = Post.objects.filter(pk__in=post_ids).delete()[0]
        return Response({'deleted': deleted_count})
```

## ✅ Best Practices

### 1. Override perform\_\* Methods

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def perform_create(self, serializer):
        """Called during POST"""
        serializer.save(author=self.request.user)

    def perform_update(self, serializer):
        """Called during PUT/PATCH"""
        serializer.save(updated_by=self.request.user)

    def perform_destroy(self, instance):
        """Called during DELETE"""
        # Soft delete
        instance.deleted_at = timezone.now()
        instance.save()
```

### 2. Custom Response Format

```python
from rest_framework.response import Response

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def list(self, request, *args, **kwargs):
        """Custom list response"""
        queryset = self.filter_queryset(self.get_queryset())
        serializer = self.get_serializer(queryset, many=True)

        return Response({
            'status': 'success',
            'data': serializer.data,
            'count': queryset.count()
        })

    def create(self, request, *args, **kwargs):
        """Custom create response"""
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)

        return Response({
            'status': 'success',
            'message': 'Post created successfully',
            'data': serializer.data
        }, status=201)
```

## 🔐 Security Best Practices

### 1. Permission Classes

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated, IsAdminUser

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get_permissions(self):
        """Different permissions per action"""
        if self.action in ['list', 'retrieve']:
            return []  # Public access
        elif self.action in ['update', 'partial_update', 'destroy']:
            return [IsAuthenticated(), IsOwnerOrAdmin()]
        return [IsAuthenticated()]
```

### 2. Authorization Checks

```python
from rest_framework.exceptions import PermissionDenied

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def update(self, request, *args, **kwargs):
        """Check ownership before update"""
        instance = self.get_object()

        if instance.author != request.user and not request.user.is_staff:
            raise PermissionDenied("You don't have permission to edit this post")

        return super().update(request, *args, **kwargs)
```

## ⚡ Performance Optimization

### 1. Query Optimization

```python
class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer

    def get_queryset(self):
        """Optimize with select_related/prefetch_related"""
        return Post.objects.select_related('author', 'category') \
                           .prefetch_related('tags', 'comments__author') \
                           .all()
```

### 2. Caching

```python
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    @method_decorator(cache_page(60 * 15))  # Cache 15 minutes
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

## ❓ Interview Questions

### Q1: What's the difference between ViewSet and APIView?

**Answer**:

- **APIView**: Manual URL mapping, explicit HTTP methods (get, post, etc.)
- **ViewSet**: Automatic routing, action-based methods (list, create, retrieve, etc.)

ViewSet reduces boilerplate for standard CRUD operations.

### Q2: What is @action decorator used for?

**Answer**:
Define custom endpoints on ViewSet beyond standard CRUD. `detail=True` for single object (`/posts/{id}/publish/`), `detail=False` for collection (`/posts/recent/`).

### Q3: How do you use different serializers for different actions?

**Answer**:
Override `get_serializer_class()`:

```python
def get_serializer_class(self):
    if self.action == 'list':
        return PostListSerializer
    return PostDetailSerializer
```

## 📚 Summary

**Key Takeaways**:

1. ModelViewSet provides full CRUD
2. ReadOnlyModelViewSet for read-only APIs
3. @action for custom endpoints
4. get_queryset() for custom filtering
5. get_serializer_class() for different serializers
6. perform_create/update/destroy for hooks
7. Router automatically generates URLs
8. filter_backends for filtering/search
9. pagination_class for pagination
10. Override get_permissions() for action-specific permissions

DRF ViewSets simplify API development!
