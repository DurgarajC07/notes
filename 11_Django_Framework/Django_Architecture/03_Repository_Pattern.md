# Django Repository Pattern

## 📖 Concept Explanation

The Repository Pattern abstracts data access logic, providing a collection-like interface for accessing domain objects. It sits between the business logic (services) and data access (ORM), making the application more maintainable and testable.

### Traditional Django ORM Access

```python
# ❌ Direct ORM access scattered everywhere
def get_published_posts(request):
    posts = Post.objects.filter(published=True).select_related('author')
    return render(request, 'posts.html', {'posts': posts})

def get_user_posts(request):
    posts = Post.objects.filter(author=request.user, published=True).select_related('author')
    return render(request, 'my_posts.html', {'posts': posts})

# Same query logic duplicated!
```

### With Repository Pattern

```python
# ✅ Centralized data access
class PostRepository:
    def get_published_posts(self):
        return Post.objects.filter(published=True).select_related('author')

    def get_user_posts(self, user):
        return self.get_published_posts().filter(author=user)

# Usage
repo = PostRepository()
posts = repo.get_published_posts()
user_posts = repo.get_user_posts(request.user)
```

**Benefits**:

- **Centralized queries**: All data access in one place
- **Testable**: Easy to mock repositories
- **Swappable**: Change data source without affecting business logic
- **Maintainable**: Query optimization in one location

## 🧠 Basic Repository Implementation

### 1. Base Repository

```python
# apps/core/repositories/base_repository.py
from typing import List, Optional, Type, TypeVar
from django.db.models import Model, QuerySet

T = TypeVar('T', bound=Model)

class BaseRepository:
    """Base repository with common CRUD operations"""

    def __init__(self, model: Type[T]):
        self.model = model

    def get_all(self) -> QuerySet[T]:
        """Get all objects"""
        return self.model.objects.all()

    def get_by_id(self, id: int) -> Optional[T]:
        """Get object by ID"""
        try:
            return self.model.objects.get(id=id)
        except self.model.DoesNotExist:
            return None

    def filter(self, **kwargs) -> QuerySet[T]:
        """Filter objects"""
        return self.model.objects.filter(**kwargs)

    def create(self, **kwargs) -> T:
        """Create new object"""
        return self.model.objects.create(**kwargs)

    def update(self, id: int, **kwargs) -> Optional[T]:
        """Update object by ID"""
        obj = self.get_by_id(id)
        if obj:
            for key, value in kwargs.items():
                setattr(obj, key, value)
            obj.save()
        return obj

    def delete(self, id: int) -> bool:
        """Delete object by ID"""
        obj = self.get_by_id(id)
        if obj:
            obj.delete()
            return True
        return False

    def exists(self, **kwargs) -> bool:
        """Check if object exists"""
        return self.model.objects.filter(**kwargs).exists()

    def count(self, **kwargs) -> int:
        """Count objects"""
        return self.model.objects.filter(**kwargs).count()
```

### 2. Model-Specific Repository

```python
# apps/blog/repositories/post_repository.py
from typing import List, Optional
from django.db.models import Q, Count, Prefetch
from apps.core.repositories import BaseRepository
from apps.blog.models import Post, Comment

class PostRepository(BaseRepository):
    """Repository for Post model"""

    def __init__(self):
        super().__init__(Post)

    def get_published_posts(self) -> List[Post]:
        """Get all published posts with optimized queries"""
        return self.model.objects.filter(
            published=True,
            deleted_at__isnull=True
        ).select_related(
            'author',
            'category'
        ).prefetch_related(
            'tags'
        ).order_by('-created_at')

    def get_post_by_slug(self, slug: str) -> Optional[Post]:
        """Get post by slug with all relations"""
        try:
            return self.model.objects.select_related(
                'author',
                'author__profile',
                'category'
            ).prefetch_related(
                'tags',
                Prefetch(
                    'comments',
                    queryset=Comment.objects.select_related('author')
                )
            ).get(slug=slug, published=True)
        except self.model.DoesNotExist:
            return None

    def get_user_posts(self, user_id: int) -> List[Post]:
        """Get all posts by user"""
        return self.model.objects.filter(
            author_id=user_id,
            deleted_at__isnull=True
        ).select_related('category').prefetch_related('tags')

    def get_category_posts(self, category_id: int, limit: int = 10) -> List[Post]:
        """Get published posts by category"""
        return self.model.objects.filter(
            category_id=category_id,
            published=True,
            deleted_at__isnull=True
        ).select_related('author')[:limit]

    def search_posts(self, query: str) -> List[Post]:
        """Search posts by title or content"""
        return self.model.objects.filter(
            Q(title__icontains=query) | Q(content__icontains=query),
            published=True,
            deleted_at__isnull=True
        ).select_related('author')

    def get_popular_posts(self, limit: int = 10) -> List[Post]:
        """Get most popular posts by view count"""
        return self.model.objects.filter(
            published=True,
            deleted_at__isnull=True
        ).order_by('-views')[:limit]

    def get_posts_with_comment_count(self) -> List[Post]:
        """Get posts with comment count annotation"""
        return self.model.objects.filter(
            published=True
        ).annotate(
            comment_count=Count('comments')
        ).select_related('author')

    def bulk_create(self, posts_data: List[dict]) -> List[Post]:
        """Bulk create posts"""
        posts = [Post(**data) for data in posts_data]
        return self.model.objects.bulk_create(posts)

    def soft_delete(self, post_id: int) -> Optional[Post]:
        """Soft delete post"""
        from django.utils import timezone

        post = self.get_by_id(post_id)
        if post:
            post.deleted_at = timezone.now()
            post.save()
        return post
```

### 3. Using Repository in Service

```python
# apps/blog/services/post_service.py
from django.db import transaction
from apps.blog.repositories import PostRepository
from apps.core.exceptions import ValidationError

class PostService:
    """Service using repository for data access"""

    def __init__(self):
        self.post_repo = PostRepository()

    def get_published_posts(self):
        """Get all published posts"""
        return self.post_repo.get_published_posts()

    def get_post_detail(self, slug: str):
        """Get post by slug"""
        post = self.post_repo.get_post_by_slug(slug)
        if not post:
            raise ValidationError("Post not found")

        # Increment views
        post.views += 1
        post.save()

        return post

    @transaction.atomic
    def create_post(self, title: str, content: str, author, tags=None):
        """Create new post"""
        # Validation
        if self.post_repo.exists(slug=self._slugify(title)):
            raise ValidationError("Post with this title already exists")

        # Create post
        post = self.post_repo.create(
            title=title,
            content=content,
            slug=self._slugify(title),
            author=author
        )

        # Add tags
        if tags:
            post.tags.set(tags)

        return post

    def delete_post(self, post_id: int, user):
        """Delete post with permission check"""
        post = self.post_repo.get_by_id(post_id)
        if not post:
            raise ValidationError("Post not found")

        if post.author != user and not user.is_staff:
            raise ValidationError("Permission denied")

        return self.post_repo.soft_delete(post_id)

    @staticmethod
    def _slugify(title: str) -> str:
        from django.utils.text import slugify
        return slugify(title)
```

## 🎯 Advanced Repository Patterns

### 1. Repository with Specification Pattern

```python
# apps/core/repositories/specification.py
from abc import ABC, abstractmethod
from django.db.models import Q

class Specification(ABC):
    """Base specification for filtering"""

    @abstractmethod
    def to_q(self) -> Q:
        """Convert to Django Q object"""
        pass

class AndSpecification(Specification):
    """AND combination of specifications"""

    def __init__(self, *specs):
        self.specs = specs

    def to_q(self) -> Q:
        q = Q()
        for spec in self.specs:
            q &= spec.to_q()
        return q

class OrSpecification(Specification):
    """OR combination of specifications"""

    def __init__(self, *specs):
        self.specs = specs

    def to_q(self) -> Q:
        q = Q()
        for spec in self.specs:
            q |= spec.to_q()
        return q

# Concrete specifications
class PublishedPostSpecification(Specification):
    """Specification for published posts"""

    def to_q(self) -> Q:
        return Q(published=True, deleted_at__isnull=True)

class AuthorPostSpecification(Specification):
    """Specification for posts by author"""

    def __init__(self, author_id: int):
        self.author_id = author_id

    def to_q(self) -> Q:
        return Q(author_id=self.author_id)

class CategoryPostSpecification(Specification):
    """Specification for posts by category"""

    def __init__(self, category_id: int):
        self.category_id = category_id

    def to_q(self) -> Q:
        return Q(category_id=self.category_id)

# Usage in repository
class PostRepository(BaseRepository):
    def find_by_specification(self, spec: Specification):
        """Find posts matching specification"""
        return self.model.objects.filter(spec.to_q())

# Usage
repo = PostRepository()

# Published posts by author
spec = AndSpecification(
    PublishedPostSpecification(),
    AuthorPostSpecification(author_id=1)
)
posts = repo.find_by_specification(spec)

# Published posts in multiple categories
spec = AndSpecification(
    PublishedPostSpecification(),
    OrSpecification(
        CategoryPostSpecification(category_id=1),
        CategoryPostSpecification(category_id=2)
    )
)
posts = repo.find_by_specification(spec)
```

### 2. Repository with Caching

```python
# apps/blog/repositories/cached_post_repository.py
from django.core.cache import cache
from apps.blog.repositories import PostRepository

class CachedPostRepository(PostRepository):
    """Repository with caching layer"""

    CACHE_TTL = 300  # 5 minutes

    def get_post_by_slug(self, slug: str):
        """Get post with caching"""
        cache_key = f'post:slug:{slug}'

        # Try cache
        post = cache.get(cache_key)
        if post is not None:
            return post

        # Load from database
        post = super().get_post_by_slug(slug)

        # Cache result
        if post:
            cache.set(cache_key, post, self.CACHE_TTL)

        return post

    def get_published_posts(self):
        """Get published posts with caching"""
        cache_key = 'posts:published'

        posts = cache.get(cache_key)
        if posts is None:
            posts = list(super().get_published_posts())
            cache.set(cache_key, posts, self.CACHE_TTL)

        return posts

    def create(self, **kwargs):
        """Create and invalidate cache"""
        post = super().create(**kwargs)

        # Invalidate cache
        cache.delete('posts:published')

        return post

    def update(self, id: int, **kwargs):
        """Update and invalidate cache"""
        post = super().update(id, **kwargs)

        if post:
            # Invalidate specific cache
            cache.delete(f'post:slug:{post.slug}')
            cache.delete('posts:published')

        return post
```

### 3. Repository Factory

```python
# apps/core/repositories/factory.py
from apps.blog.repositories import PostRepository
from apps.users.repositories import UserRepository
from apps.ecommerce.repositories import OrderRepository

class RepositoryFactory:
    """Factory for creating repositories"""

    _instances = {}

    @classmethod
    def get_post_repository(cls):
        """Get or create PostRepository"""
        if 'post' not in cls._instances:
            cls._instances['post'] = PostRepository()
        return cls._instances['post']

    @classmethod
    def get_user_repository(cls):
        """Get or create UserRepository"""
        if 'user' not in cls._instances:
            cls._instances['user'] = UserRepository()
        return cls._instances['user']

    @classmethod
    def get_order_repository(cls):
        """Get or create OrderRepository"""
        if 'order' not in cls._instances:
            cls._instances['order'] = OrderRepository()
        return cls._instances['order']

# Usage
post_repo = RepositoryFactory.get_post_repository()
posts = post_repo.get_published_posts()
```

## ✅ Best Practices

### 1. Keep Repositories Focused

```python
# ✅ Good: Repository only handles data access
class PostRepository:
    def get_published_posts(self):
        return Post.objects.filter(published=True)

    def create(self, **kwargs):
        return Post.objects.create(**kwargs)

# ❌ Bad: Repository with business logic
class PostRepository:
    def publish_post(self, post_id):
        post = self.get_by_id(post_id)
        post.published = True
        post.save()

        # ❌ Sending email is business logic, not data access!
        send_mail(...)
```

### 2. Test Repositories

```python
# tests/test_post_repository.py
from django.test import TestCase
from apps.blog.repositories import PostRepository
from apps.blog.models import Post
from django.contrib.auth import get_user_model

User = get_user_model()

class PostRepositoryTest(TestCase):
    def setUp(self):
        self.repo = PostRepository()
        self.user = User.objects.create_user(username='testuser')

    def test_get_published_posts(self):
        """Test getting published posts"""
        # Create published post
        Post.objects.create(title='Published', published=True, author=self.user)
        # Create draft post
        Post.objects.create(title='Draft', published=False, author=self.user)

        posts = self.repo.get_published_posts()

        self.assertEqual(len(posts), 1)
        self.assertEqual(posts[0].title, 'Published')

    def test_get_post_by_slug(self):
        """Test getting post by slug"""
        post = Post.objects.create(
            title='Test',
            slug='test',
            published=True,
            author=self.user
        )

        result = self.repo.get_post_by_slug('test')

        self.assertEqual(result, post)

    def test_get_post_by_slug_not_found(self):
        """Test getting non-existent post"""
        result = self.repo.get_post_by_slug('nonexistent')

        self.assertIsNone(result)
```

### 3. Mock Repositories in Service Tests

```python
# tests/test_post_service.py
from unittest.mock import Mock, MagicMock
from django.test import TestCase
from apps.blog.services import PostService
from apps.blog.models import Post

class PostServiceTest(TestCase):
    def test_get_post_detail(self):
        """Test service with mocked repository"""
        # Create mock repository
        mock_repo = Mock()
        mock_post = MagicMock(spec=Post)
        mock_post.views = 0
        mock_repo.get_post_by_slug.return_value = mock_post

        # Inject mock
        service = PostService()
        service.post_repo = mock_repo

        # Test
        result = service.get_post_detail('test-slug')

        # Verify
        mock_repo.get_post_by_slug.assert_called_once_with('test-slug')
        self.assertEqual(result.views, 1)
```

## ❌ Common Mistakes

### 1. Repositories with Business Logic

```python
# ❌ Bad: Business logic in repository
class PostRepository:
    def publish_post(self, post_id):
        post = self.get_by_id(post_id)
        post.published = True
        post.save()

        # Business logic!
        send_notification_email(post)
        invalidate_cache()
        log_event()

# ✅ Good: Business logic in service
class PostService:
    def __init__(self):
        self.repo = PostRepository()

    def publish_post(self, post_id):
        post = self.repo.get_by_id(post_id)
        post.published = True
        post.save()

        # Business logic in service
        send_notification_email(post)
        invalidate_cache()
        log_event()
```

### 2. Leaky Abstractions

```python
# ❌ Bad: Exposes ORM QuerySet
class PostRepository:
    def get_posts(self):
        return Post.objects.filter(published=True)  # Returns QuerySet

# View can modify queryset, defeating purpose of repository
posts = repo.get_posts().filter(author=user)

# ✅ Good: Returns list/tuple
class PostRepository:
    def get_posts(self):
        return list(Post.objects.filter(published=True))
```

## ❓ Interview Questions

### Q1: What is the Repository Pattern?

**Answer**:
A pattern that abstracts data access logic, providing a collection-like interface. It sits between business logic and data source, making code more testable and maintainable.

### Q2: When should you use Repository Pattern in Django?

**Answer**:
Use when:

- Complex queries repeated across codebase
- Need to swap data sources (ORM to API)
- Want highly testable code
- Large team needs consistent data access

Don't use for simple CRUD apps (over-engineering).

## 📚 Summary

**Key Takeaways**:

1. Repositories abstract data access from business logic
2. Use BaseRepository for common CRUD operations
3. Create model-specific repositories for complex queries
4. Repositories make services more testable
5. Keep repositories focused on data access only
6. Use specifications for complex filtering
7. Add caching layer in repositories
8. Test repositories separately from services
9. Mock repositories in service tests
10. Avoid leaky abstractions (don't expose QuerySets)

The Repository Pattern improves code organization and testability!
