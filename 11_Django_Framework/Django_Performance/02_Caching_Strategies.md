# Django Caching Strategies

## 📖 Concept Explanation

Caching stores frequently accessed data in fast storage (memory) to avoid expensive database queries or computations. A well-implemented caching strategy can improve response times by 10-100x.

### Cache Hierarchy

```
1. Browser Cache (Client-side)
2. CDN Cache (Edge servers)
3. Django Cache Framework (Application-level)
   - Template fragments
   - View output
   - QuerySet results
4. Database Query Cache
5. ORM Query Cache (select_related/prefetch_related)
```

## 🧠 Django Cache Backends

### 1. Redis Cache (Recommended for Production)

```bash
pip install django-redis
```

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'PASSWORD': os.environ.get('REDIS_PASSWORD'),
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
            'CONNECTION_POOL_KWARGS': {
                'max_connections': 50,
                'retry_on_timeout': True
            },
            'COMPRESSOR': 'django_redis.compressors.zlib.ZlibCompressor',
            'SERIALIZER': 'django_redis.serializers.json.JSONSerializer',
        },
        'KEY_PREFIX': 'myapp',
        'VERSION': 1,
        'TIMEOUT': 300,  # 5 minutes default
    }
}
```

### 2. Memcached Cache

```bash
pip install python-memcached
```

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
        'TIMEOUT': 300,
        'OPTIONS': {
            'MAX_ENTRIES': 1000,
        }
    }
}
```

### 3. Multiple Cache Backends

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    },
    'sessions': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/2',
    },
    'staticfiles': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'staticfiles',
    }
}

# Use specific cache
from django.core.cache import caches
session_cache = caches['sessions']
```

## 🎯 Cache Levels

### 1. Low-Level Cache API

```python
from django.core.cache import cache

# Set cache
cache.set('my_key', 'my_value', timeout=300)  # 5 minutes

# Get cache
value = cache.get('my_key')
if value is None:
    value = expensive_computation()
    cache.set('my_key', value, timeout=300)

# Get with default
value = cache.get('my_key', 'default_value')

# Get or set (atomic operation)
value = cache.get_or_set('my_key', expensive_computation, timeout=300)

# Delete
cache.delete('my_key')

# Clear all
cache.clear()

# Multiple operations
cache.set_many({'key1': 'value1', 'key2': 'value2'}, timeout=300)
values = cache.get_many(['key1', 'key2'])
# {'key1': 'value1', 'key2': 'value2'}
```

### 2. Per-View Caching

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
def my_view(request):
    """Entire view output is cached"""
    posts = Post.objects.all()
    return render(request, 'posts.html', {'posts': posts})

# Cache with specific cache backend
@cache_page(60 * 15, cache='special_cache')
def special_view(request):
    return render(request, 'special.html')

# URL-based caching
from django.views.decorators.cache import cache_page

urlpatterns = [
    path('posts/', cache_page(60 * 15)(PostListView.as_view())),
]
```

### 3. Template Fragment Caching

```django
{% load cache %}

<!-- Cache for 5 minutes -->
{% cache 300 sidebar %}
    <div class="sidebar">
        {% for item in sidebar_items %}
            <div>{{ item.title }}</div>
        {% endfor %}
    </div>
{% endcache %}

<!-- Cache with variable key -->
{% cache 300 post_detail post.id %}
    <div class="post">
        <h1>{{ post.title }}</h1>
        <p>{{ post.content }}</p>
    </div>
{% endcache %}

<!-- Conditional caching -->
{% cache 300 user_sidebar request.user.id %}
    <!-- User-specific sidebar -->
{% endcache %}
```

### 4. QuerySet Caching

```python
from django.core.cache import cache

def get_posts():
    """Cache expensive queryset"""
    cache_key = 'all_posts'
    posts = cache.get(cache_key)

    if posts is None:
        posts = list(Post.objects.select_related('author')
                                 .prefetch_related('tags')
                                 .filter(published=True))
        cache.set(cache_key, posts, timeout=300)

    return posts

# Cache with parameters
def get_posts_by_category(category_id):
    cache_key = f'posts_category_{category_id}'
    posts = cache.get(cache_key)

    if posts is None:
        posts = list(Post.objects.filter(category_id=category_id))
        cache.set(cache_key, posts, timeout=600)

    return posts
```

## 🔄 Cache Invalidation

### 1. Manual Invalidation

```python
from django.core.cache import cache

# Invalidate on save
class Post(models.Model):
    title = models.CharField(max_length=200)

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        # Clear related caches
        cache.delete('all_posts')
        cache.delete(f'post_{self.id}')
        cache.delete(f'posts_category_{self.category_id}')
```

### 2. Signal-Based Invalidation

```python
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver

@receiver(post_save, sender=Post)
def invalidate_post_cache(sender, instance, **kwargs):
    """Invalidate caches when post is saved"""
    cache.delete('all_posts')
    cache.delete(f'post_{instance.id}')
    cache.delete(f'posts_author_{instance.author_id}')

@receiver(post_delete, sender=Post)
def invalidate_post_cache_on_delete(sender, instance, **kwargs):
    """Invalidate caches when post is deleted"""
    cache.delete('all_posts')
    cache.delete(f'post_{instance.id}')
```

### 3. Time-Based Invalidation

```python
# Short TTL for frequently changing data
cache.set('trending_posts', posts, timeout=60)  # 1 minute

# Long TTL for rarely changing data
cache.set('site_config', config, timeout=3600 * 24)  # 1 day

# Very long TTL with versioning
cache.set('static_content_v2', content, timeout=3600 * 24 * 7)  # 1 week
```

### 4. Cache Versioning

```python
# Cache with version key
VERSION = 'v1'
cache_key = f'posts_{VERSION}'
posts = cache.get(cache_key)

# When data structure changes, increment version
VERSION = 'v2'  # Old cache automatically ignored
```

## 🚀 Advanced Caching Patterns

### 1. Cache Warming

```python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Warm up cache with frequently accessed data'

    def handle(self, *args, **options):
        """Pre-populate cache"""

        # Cache homepage data
        posts = list(Post.objects.filter(published=True)[:10])
        cache.set('homepage_posts', posts, timeout=600)

        # Cache popular posts
        popular = list(Post.objects.order_by('-views')[:20])
        cache.set('popular_posts', popular, timeout=300)

        # Cache categories
        categories = list(Category.objects.all())
        cache.set('all_categories', categories, timeout=3600)

        self.stdout.write(self.style.SUCCESS('Cache warmed successfully'))

# Run: python manage.py warm_cache
```

### 2. Cache Aside Pattern

```python
def get_post(post_id):
    """
    Cache Aside: Try cache first, then database
    """
    cache_key = f'post_{post_id}'

    # Try cache
    post = cache.get(cache_key)

    if post is None:
        # Cache miss: Load from database
        try:
            post = Post.objects.select_related('author').get(pk=post_id)
            # Store in cache
            cache.set(cache_key, post, timeout=600)
        except Post.DoesNotExist:
            # Cache negative result to prevent repeated DB queries
            cache.set(cache_key, 'NOT_FOUND', timeout=60)
            return None

    if post == 'NOT_FOUND':
        return None

    return post
```

### 3. Write-Through Cache

```python
def update_post(post_id, data):
    """
    Write-Through: Update database and cache simultaneously
    """
    post = Post.objects.get(pk=post_id)

    # Update database
    for key, value in data.items():
        setattr(post, key, value)
    post.save()

    # Update cache immediately
    cache_key = f'post_{post_id}'
    cache.set(cache_key, post, timeout=600)

    return post
```

### 4. Cache Stampede Prevention

```python
import random
from django.core.cache import cache

def get_with_jitter(cache_key, compute_func, base_timeout=300):
    """
    Prevent cache stampede with jittered TTL
    """
    result = cache.get(cache_key)

    if result is None:
        # Add jitter (±10%)
        jitter = random.randint(-30, 30)
        timeout = base_timeout + jitter

        result = compute_func()
        cache.set(cache_key, result, timeout=timeout)

    return result
```

## 🔐 Session Caching

```python
# settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'sessions'

# Or cached_db (hybrid: cache + database fallback)
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
```

## ⚡ Performance Best Practices

### 1. Cache Key Design

```python
# Good: Specific, versioned keys
cache_key = f'post_{post_id}_v2'
cache_key = f'user_{user_id}_posts_page_{page}_v1'

# Bad: Generic keys prone to collisions
cache_key = 'posts'  # Too generic
cache_key = f'{post_id}'  # No context
```

### 2. Appropriate Timeouts

```python
# Very frequent updates: Short TTL
cache.set('active_users', users, timeout=30)  # 30 seconds

# Moderate updates: Medium TTL
cache.set('post_list', posts, timeout=300)  # 5 minutes

# Rare updates: Long TTL
cache.set('site_settings', settings, timeout=3600)  # 1 hour

# Static data: Very long TTL
cache.set('country_list', countries, timeout=86400)  # 24 hours
```

### 3. Cache Size Management

```python
# Limit cached data size
def cache_post(post):
    """Cache only essential fields"""
    cache_data = {
        'id': post.id,
        'title': post.title,
        'slug': post.slug,
        'created_at': post.created_at.isoformat(),
    }
    cache.set(f'post_{post.id}', cache_data, timeout=600)

# Don't cache large objects
# Bad:
# cache.set('post_with_comments', post)  # Includes all related data!

# Good:
# Cache post and comments separately
```

## 🧪 Testing Cache

```python
from django.test import TestCase
from django.core.cache import cache

class CacheTestCase(TestCase):
    def setUp(self):
        cache.clear()

    def test_cache_set_get(self):
        """Test basic cache operations"""
        cache.set('test_key', 'test_value', timeout=300)
        value = cache.get('test_key')
        self.assertEqual(value, 'test_value')

    def test_cache_invalidation(self):
        """Test cache invalidation on model save"""
        post = Post.objects.create(title='Test')

        # Cache should be invalidated
        cached_posts = cache.get('all_posts')
        self.assertIsNone(cached_posts)
```

## ❓ Interview Questions

### Q1: When should you use Redis vs Memcached?

**Answer**:

- **Redis**: Persistence, complex data structures, pub/sub, TTL per key, better for sessions
- **Memcached**: Pure memory, simpler, slightly faster for simple key-value

Choose Redis for production Django apps.

### Q2: What's cache stampede and how do you prevent it?

**Answer**:
Multiple requests simultaneously hit expired cache, all query database. Prevent with:

1. Jittered TTL (add randomness to timeout)
2. Soft expiration (return stale data while refreshing)
3. Lock-based updates

### Q3: How do you invalidate cache when related model changes?

**Answer**:
Use signals (`post_save`, `post_delete`) to invalidate related cache keys. Consider cache dependencies and version-based invalidation for complex relationships.

## 📚 Summary

**Key Takeaways**:

1. Use Redis for production caching
2. Cache expensive queries and computations
3. Use template fragment caching for reusable components
4. Implement cache invalidation on data changes
5. Add jitter to prevent cache stampede
6. Use appropriate TTLs for different data types
7. Design specific cache keys with versioning
8. Monitor cache hit/miss ratios
9. Don't cache everything (overhead vs benefit)
10. Test cache behavior in tests

Effective caching is key to Django performance!
