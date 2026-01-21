# Django Framework Interview Questions

## 📖 Overview

Comprehensive Django interview questions covering beginner to senior level topics, with detailed explanations and code examples.

## 🎯 Django Core Concepts

### Q1: Explain Django's MTV (Model-Template-View) architecture

**Answer**:

Django follows MTV pattern (variation of MVC):

- **Model**: Data layer, database interaction (ORM)
- **Template**: Presentation layer (HTML with Django template language)
- **View**: Business logic, handles requests and returns responses
- **URLconf**: Maps URLs to views (Controller equivalent)

```python
# Model
class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()

# View
def article_list(request):
    articles = Article.objects.all()
    return render(request, 'articles/list.html', {'articles': articles})

# Template (articles/list.html)
# {% for article in articles %}
#   <h2>{{ article.title }}</h2>
# {% endfor %}

# URL
urlpatterns = [
    path('articles/', article_list, name='article-list'),
]
```

**Key Difference from MVC**: In Django, the View contains business logic (like Controller in MVC), and Template is the presentation (like View in MVC).

### Q2: What is Django ORM and how does it work?

**Answer**:

Django ORM (Object-Relational Mapping) translates Python code to SQL queries:

```python
# Python ORM query
users = User.objects.filter(age__gte=18, is_active=True)

# Translates to SQL:
# SELECT * FROM users WHERE age >= 18 AND is_active = TRUE;

# Complex query
articles = (Article.objects
           .filter(published=True)
           .select_related('author')  # JOIN
           .prefetch_related('tags')   # Separate query
           .order_by('-created_at'))

# Translates to optimized SQL with JOIN and additional query for M2M
```

**Benefits**:

- Database agnostic (PostgreSQL, MySQL, SQLite)
- Protection against SQL injection
- Pythonic API
- Lazy evaluation

**Limitations**:

- Complex queries may be inefficient
- Learning curve for query optimization
- Raw SQL sometimes necessary

### Q3: Explain Django's request-response cycle

**Answer**:

```
1. Browser sends HTTP request
   ↓
2. WSGI/ASGI server receives request
   ↓
3. Middleware (Request phase) - Authentication, CORS, etc.
   ↓
4. URL Resolver - Matches URL to view
   ↓
5. View - Business logic, queries database
   ↓
6. Template (if used) - Renders HTML
   ↓
7. Middleware (Response phase) - Headers, logging
   ↓
8. HTTP Response returned to browser
```

**Code example**:

```python
# Custom middleware example
class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Before view (Request phase)
        print(f"Request: {request.method} {request.path}")

        response = self.get_response(request)

        # After view (Response phase)
        print(f"Response: {response.status_code}")
        return response

# settings.py
MIDDLEWARE = [
    'myapp.middleware.RequestLoggingMiddleware',
    # ... other middleware
]
```

### Q4: What are Django signals and when should you use them?

**Answer**:

Signals allow decoupled applications to get notified when actions occur elsewhere in the framework.

**Common signals**:

- `pre_save` / `post_save` - Before/after model save
- `pre_delete` / `post_delete` - Before/after model delete
- `m2m_changed` - When ManyToMany relationship changes
- `request_started` / `request_finished` - Request lifecycle

```python
# Example: Create user profile automatically
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User
from .models import UserProfile

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        UserProfile.objects.create(user=instance)

@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    instance.profile.save()
```

**When to use**:
✅ Logging/auditing
✅ Cache invalidation
✅ Sending notifications
✅ Creating related objects

**When NOT to use**:
❌ Core business logic (use model methods instead)
❌ Complex workflows (use Celery tasks)
❌ Can be tested with regular code
❌ Performance-critical paths (signals add overhead)

### Q5: What is the difference between select_related and prefetch_related?

**Answer**:

Both optimize database queries but work differently:

**select_related** - SQL JOIN (ForeignKey, OneToOne):

```python
# Without select_related (N+1 problem)
articles = Article.objects.all()
for article in articles:  # 1 query
    print(article.author.name)  # N queries (one per article)

# With select_related (1 query with JOIN)
articles = Article.objects.select_related('author').all()
for article in articles:  # 1 query total
    print(article.author.name)  # No additional query

# SQL: SELECT * FROM article JOIN author ON article.author_id = author.id
```

**prefetch_related** - Separate queries (ManyToMany, reverse ForeignKey):

```python
# Without prefetch_related
articles = Article.objects.all()
for article in articles:  # 1 query
    print(article.tags.all())  # N queries

# With prefetch_related (2 queries total)
articles = Article.objects.prefetch_related('tags').all()
for article in articles:  # 1 query for articles
    print(article.tags.all())  # 1 additional query for all tags

# Query 1: SELECT * FROM article
# Query 2: SELECT * FROM tag WHERE article_id IN (1,2,3...)
```

**Key differences**:
| Feature | select_related | prefetch_related |
|---------|---------------|------------------|
| Relationship | ForeignKey, OneToOne | ManyToMany, Reverse FK |
| SQL | JOIN | Separate queries |
| Queries | 1 | 2 (or more) |
| Use case | Simple relations | Complex relations |

## 🔐 Django Security

### Q6: Explain Django's CSRF protection

**Answer**:

CSRF (Cross-Site Request Forgery) protection prevents malicious sites from performing actions on behalf of authenticated users.

**How it works**:

1. Django generates unique CSRF token per session
2. Token included in forms via `{% csrf_token %}`
3. On POST, Django verifies token matches session
4. If mismatch, request rejected (403 Forbidden)

```python
# Template
<form method="post">
    {% csrf_token %}
    <input type="text" name="username">
    <button type="submit">Submit</button>
</form>

# View
def update_profile(request):
    if request.method == 'POST':
        # CSRF token automatically validated
        # by CsrfViewMiddleware
        username = request.POST.get('username')
        # ... process form

# AJAX requests
// JavaScript
fetch('/api/data/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': getCookie('csrftoken'),
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data)
})

# Exempt specific view (use carefully!)
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def webhook(request):
    # External webhooks don't have CSRF tokens
    # Verify with signature/API key instead
    pass
```

**Best practices**:

- Always include `{% csrf_token %}` in forms
- Use `@csrf_exempt` sparingly
- Verify AJAX includes CSRF token
- Don't disable CSRF globally

### Q7: How do you prevent SQL injection in Django?

**Answer**:

Django ORM automatically prevents SQL injection by:

1. Using parameterized queries
2. Escaping user input

```python
# ✅ Safe: ORM automatically parameterizes
username = request.GET.get('username')
users = User.objects.filter(username=username)
# SQL: SELECT * FROM user WHERE username = %s
# Parameter: 'username_value'

# ✅ Safe: Using params
from django.db import connection
cursor = connection.cursor()
cursor.execute("SELECT * FROM user WHERE username = %s", [username])

# ❌ DANGEROUS: String formatting
cursor.execute(f"SELECT * FROM user WHERE username = '{username}'")
# Vulnerable to: ' OR '1'='1

# ❌ DANGEROUS: Using .raw() incorrectly
User.objects.raw(f"SELECT * FROM user WHERE username = '{username}'")

# ✅ Safe: .raw() with params
User.objects.raw("SELECT * FROM user WHERE username = %s", [username])

# ✅ Safe: .extra() with params
User.objects.extra(
    where=["username = %s"],
    params=[username]
)
```

**Additional protections**:

- Validate user input
- Use form validation
- Limit queryset access
- Use `get_object_or_404()` for safe lookups

### Q8: What is XSS and how does Django prevent it?

**Answer**:

XSS (Cross-Site Scripting) allows attackers to inject malicious scripts into web pages.

**Django's protections**:

```python
# Template auto-escaping (enabled by default)
{{ user_input }}  # Automatically escapes HTML
# <script>alert('XSS')</script> becomes:
# &lt;script&gt;alert('XSS')&lt;/script&gt;

# Explicitly mark safe (use carefully!)
{{ html_content|safe }}

# In Python code
from django.utils.html import escape
safe_text = escape(user_input)

# In JSON responses
from django.utils.html import escapejs
safe_js = escapejs(user_input)
```

**Best practices**:

```python
# ✅ Good: Let Django handle escaping
def user_profile(request):
    bio = request.user.bio
    return render(request, 'profile.html', {'bio': bio})

# Template: {{ bio }} - automatically escaped

# ❌ Bad: Marking untrusted input as safe
{{ user.bio|safe }}  # Don't do this!

# ✅ Good: Sanitize HTML if needed
from bleach import clean
allowed_tags = ['p', 'br', 'strong', 'em']
safe_html = clean(user_input, tags=allowed_tags)
```

**Content Security Policy (CSP)**:

```python
# settings.py
CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", "https://cdn.example.com")
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'")
```

## 🚀 Django Performance

### Q9: How do you optimize Django queries?

**Answer**:

**1. Use select_related and prefetch_related**:

```python
# ❌ N+1 problem
articles = Article.objects.all()
for article in articles:
    print(article.author.name)  # N queries

# ✅ Optimized
articles = Article.objects.select_related('author').all()
```

**2. Use only() and defer()**:

```python
# ✅ Load only needed fields
users = User.objects.only('id', 'username')

# ✅ Defer large fields
articles = Article.objects.defer('content')
```

**3. Use values() and values_list()**:

```python
# ✅ Return dicts instead of model instances
user_names = User.objects.values('id', 'username')

# ✅ Return tuples
user_ids = User.objects.values_list('id', flat=True)
```

**4. Use queryset caching**:

```python
# ✅ Queryset is cached after evaluation
users = User.objects.filter(is_active=True)
list(users)  # Evaluates and caches
print(len(users))  # Uses cache
for user in users:  # Uses cache
    print(user)
```

**5. Use database indexes**:

```python
class Article(models.Model):
    title = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(unique=True)  # Automatic index

    class Meta:
        indexes = [
            models.Index(fields=['author', '-created_at']),
            models.Index(fields=['published', 'created_at']),
        ]
```

**6. Use annotate and aggregate**:

```python
from django.db.models import Count, Avg

# ✅ Aggregate in database
stats = User.objects.aggregate(
    total=Count('id'),
    avg_age=Avg('age')
)

# ✅ Annotate each object
authors = Author.objects.annotate(
    article_count=Count('articles')
).filter(article_count__gt=5)
```

**7. Use bulk operations**:

```python
# ❌ Slow: N queries
for user in users:
    user.is_active = True
    user.save()

# ✅ Fast: 1 query
User.objects.filter(id__in=user_ids).update(is_active=True)

# ✅ Bulk create
User.objects.bulk_create([
    User(username='user1'),
    User(username='user2'),
])
```

### Q10: Explain Django's caching strategies

**Answer**:

Django provides multiple caching levels:

**1. Per-site cache**:

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.cache.UpdateCacheMiddleware',  # First
    # ... other middleware
    'django.middleware.cache.FetchFromCacheMiddleware',  # Last
]

CACHE_MIDDLEWARE_SECONDS = 600
```

**2. Per-view cache**:

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # 15 minutes
def article_list(request):
    articles = Article.objects.all()
    return render(request, 'articles.html', {'articles': articles})

# Cache with vary
@cache_page(60 * 15, key_prefix="site1")
@vary_on_headers('User-Agent')
def my_view(request):
    ...
```

**3. Template fragment caching**:

```django
{% load cache %}

{% cache 500 sidebar %}
    {# Expensive sidebar #}
    {% for item in items %}
        ...
    {% endfor %}
{% endcache %}

{# With variables #}
{% cache 500 sidebar request.user.username %}
    User-specific sidebar
{% endcache %}
```

**4. Low-level cache API**:

```python
from django.core.cache import cache

# Set cache
cache.set('my_key', 'my_value', 300)  # 5 minutes

# Get cache
value = cache.get('my_key')
if value is None:
    value = expensive_computation()
    cache.set('my_key', value, 300)

# Get or set
value = cache.get_or_set('my_key', expensive_computation, 300)

# Delete
cache.delete('my_key')

# Clear all
cache.clear()

# Multiple keys
cache.set_many({'key1': 'val1', 'key2': 'val2'}, 300)
values = cache.get_many(['key1', 'key2'])
```

**5. Cache backends**:

```python
# settings.py

# Memcached
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}

# Redis (recommended)
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# Database cache
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.db.DatabaseCache',
        'LOCATION': 'my_cache_table',
    }
}
```

**6. Cache invalidation**:

```python
from django.db.models.signals import post_save
from django.core.cache import cache

@receiver(post_save, sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    cache.delete(f'article_{instance.id}')
    cache.delete('article_list')
```

## 🏗️ Django REST Framework

### Q11: Explain DRF serializers and their types

**Answer**:

Serializers convert complex data types (models, querysets) to Python native datatypes and vice versa.

**Types of serializers**:

**1. Serializer (base)**:

```python
from rest_framework import serializers

class UserSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    username = serializers.CharField(max_length=100)
    email = serializers.EmailField()

    def create(self, validated_data):
        return User.objects.create(**validated_data)

    def update(self, instance, validated_data):
        instance.username = validated_data.get('username', instance.username)
        instance.save()
        return instance
```

**2. ModelSerializer (most common)**:

```python
class UserSerializer(serializers.ModelSerializer):
    article_count = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'article_count']
        read_only_fields = ['id']

    def get_article_count(self, obj):
        return obj.articles.count()
```

**3. HyperlinkedModelSerializer**:

```python
class ArticleSerializer(serializers.HyperlinkedModelSerializer):
    author = serializers.HyperlinkedRelatedField(
        view_name='user-detail',
        read_only=True
    )

    class Meta:
        model = Article
        fields = ['url', 'title', 'author', 'content']
```

**4. Nested serializers**:

```python
class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username']

class ArticleSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)

    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'content']
```

**5. SerializerMethodField**:

```python
class ArticleSerializer(serializers.ModelSerializer):
    author_name = serializers.SerializerMethodField()
    is_recent = serializers.SerializerMethodField()

    class Meta:
        model = Article
        fields = ['id', 'title', 'author_name', 'is_recent']

    def get_author_name(self, obj):
        return obj.author.get_full_name()

    def get_is_recent(self, obj):
        from datetime import timedelta
        from django.utils import timezone
        return obj.created_at > timezone.now() - timedelta(days=7)
```

**Common patterns**:

```python
# Validation
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['username', 'email', 'age']

    def validate_age(self, value):
        if value < 18:
            raise serializers.ValidationError("Must be 18+")
        return value

    def validate(self, data):
        if data['username'] == data['email'].split('@')[0]:
            raise serializers.ValidationError("Username can't match email")
        return data

# Context access
class ArticleSerializer(serializers.ModelSerializer):
    is_author = serializers.SerializerMethodField()

    def get_is_author(self, obj):
        request = self.context.get('request')
        return obj.author == request.user
```

### Q12: What are ViewSets and when should you use them?

**Answer**:

ViewSets combine logic for multiple related views into a single class.

**Types of ViewSets**:

**1. ModelViewSet (full CRUD)**:

```python
from rest_framework import viewsets

class ArticleViewSet(viewsets.ModelViewSet):
    """
    Provides: list, create, retrieve, update, partial_update, destroy
    """
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

# URLs (automatic routing)
from rest_framework.routers import DefaultRouter
router = DefaultRouter()
router.register(r'articles', ArticleViewSet)

# Generates URLs:
# GET /articles/ - list
# POST /articles/ - create
# GET /articles/{id}/ - retrieve
# PUT /articles/{id}/ - update
# PATCH /articles/{id}/ - partial_update
# DELETE /articles/{id}/ - destroy
```

**2. ReadOnlyModelViewSet**:

```python
class ArticleViewSet(viewsets.ReadOnlyModelViewSet):
    """
    Provides only: list, retrieve
    """
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
```

**3. GenericViewSet + Mixins**:

```python
from rest_framework import viewsets, mixins

class ArticleViewSet(mixins.CreateModelMixin,
                     mixins.RetrieveModelMixin,
                     mixins.ListModelMixin,
                     viewsets.GenericViewSet):
    """
    Custom combination: list, retrieve, create only
    No update or delete
    """
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
```

**4. Custom actions**:

```python
from rest_framework.decorators import action
from rest_framework.response import Response

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer

    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """POST /articles/{id}/publish/"""
        article = self.get_object()
        article.published = True
        article.save()
        return Response({'status': 'published'})

    @action(detail=False, methods=['get'])
    def recent(self, request):
        """GET /articles/recent/"""
        recent_articles = self.queryset.order_by('-created_at')[:10]
        serializer = self.get_serializer(recent_articles, many=True)
        return Response(serializer.data)
```

**When to use ViewSets vs APIViews**:

✅ **Use ViewSets when**:

- Standard CRUD operations
- RESTful resource-based API
- Want automatic URL routing
- Consistent patterns across endpoints

✅ **Use APIView when**:

- Non-standard operations
- Complex business logic
- Fine-grained control needed
- Non-resource endpoints

```python
# APIView for custom logic
from rest_framework.views import APIView

class UserStatsView(APIView):
    def get(self, request):
        stats = {
            'total_users': User.objects.count(),
            'active_today': User.objects.filter(
                last_login__date=timezone.now().date()
            ).count()
        }
        return Response(stats)
```

## 📚 Summary

Key Django interview topics:

1. **Architecture**: MTV pattern, request-response cycle
2. **ORM**: Query optimization, select_related, prefetch_related
3. **Security**: CSRF, XSS, SQL injection prevention
4. **Performance**: Caching, query optimization, indexing
5. **DRF**: Serializers, ViewSets, authentication
6. **Best Practices**: Signals, middleware, testing

Prepare for:

- Code examples and implementations
- Performance optimization scenarios
- Security vulnerability discussions
- Design pattern applications
- Real-world problem solving
