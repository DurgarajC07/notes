# Django Query Optimization

## 📖 Concept Explanation

Query optimization in Django focuses on reducing database queries, minimizing data transfer, and improving query execution time. Poorly optimized queries are the primary cause of slow Django applications.

### The N+1 Query Problem

```python
# Bad: N+1 queries (1 + N queries)
posts = Post.objects.all()  # 1 query
for post in posts:
    print(post.author.username)  # N queries (one per post)

# If 100 posts: 1 + 100 = 101 database queries!

# Good: 2 queries total
posts = Post.objects.select_related('author').all()  # 2 queries (JOIN)
for post in posts:
    print(post.author.username)  # No additional queries

# Fixed: 2 queries regardless of post count
```

## 🧠 select_related() - SQL JOIN

### 1. Forward ForeignKey (One-to-One)

```python
# models.py
class Author(models.Model):
    username = models.CharField(max_length=100)
    email = models.EmailField()

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)

# Bad: N+1 queries
posts = Post.objects.all()
for post in posts:
    print(f"{post.title} by {post.author.username}")
# Queries: SELECT * FROM posts + SELECT * FROM authors WHERE id=? (x N)

# Good: Single JOIN query
posts = Post.objects.select_related('author').all()
for post in posts:
    print(f"{post.title} by {post.author.username}")
# Query: SELECT * FROM posts INNER JOIN authors ON posts.author_id = authors.id
```

### 2. Multiple Related Objects

```python
# models.py
class Category(models.Model):
    name = models.CharField(max_length=100)

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)

# Select multiple related objects
posts = Post.objects.select_related('author', 'category').all()
# Query: JOIN with both authors and categories tables

# Nested relations with double underscore
posts = Post.objects.select_related('author__profile').all()
# Query: JOIN authors and profiles tables
```

### 3. OneToOneField

```python
# models.py
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    bio = models.TextField()
    avatar = models.ImageField()

# Optimize OneToOne access
users = User.objects.select_related('profile').all()
for user in users:
    print(user.profile.bio)  # No extra query
```

## 🔗 prefetch_related() - Separate Queries

### 1. Reverse ForeignKey (One-to-Many)

```python
# models.py
class Author(models.Model):
    username = models.CharField(max_length=100)

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='posts')

# Bad: N+1 queries
authors = Author.objects.all()
for author in authors:
    for post in author.posts.all():  # Query per author!
        print(post.title)

# Good: 2 queries total
authors = Author.objects.prefetch_related('posts').all()
# Query 1: SELECT * FROM authors
# Query 2: SELECT * FROM posts WHERE author_id IN (1, 2, 3, ...)
for author in authors:
    for post in author.posts.all():  # No additional queries
        print(post.title)
```

### 2. ManyToManyField

```python
# models.py
class Tag(models.Model):
    name = models.CharField(max_length=50)

class Post(models.Model):
    title = models.CharField(max_length=200)
    tags = models.ManyToManyField(Tag, related_name='posts')

# Bad: N+1 queries
posts = Post.objects.all()
for post in posts:
    print(list(post.tags.all()))  # Query per post!

# Good: 2 queries
posts = Post.objects.prefetch_related('tags').all()
# Query 1: SELECT * FROM posts
# Query 2: SELECT * FROM posts_tags JOIN tags (WHERE post_id IN ...)
for post in posts:
    print(list(post.tags.all()))  # No additional queries
```

### 3. Prefetch with Custom Queryset

```python
from django.db.models import Prefetch

# Prefetch only published posts
authors = Author.objects.prefetch_related(
    Prefetch('posts', queryset=Post.objects.filter(published=True))
).all()

# Prefetch with select_related (combined)
authors = Author.objects.prefetch_related(
    Prefetch(
        'posts',
        queryset=Post.objects.select_related('category').filter(published=True)
    )
).all()

# Multiple prefetches
posts = Post.objects.prefetch_related(
    'tags',
    'comments__author',  # Nested prefetch
    Prefetch('comments', queryset=Comment.objects.filter(approved=True))
).all()
```

## 🎯 only() and defer()

### 1. only() - Load Specific Fields

```python
# Load only specific fields
posts = Post.objects.only('id', 'title', 'published').all()
# Query: SELECT id, title, published FROM posts

# Accessing other fields triggers additional queries
for post in posts:
    print(post.title)  # No query (loaded)
    print(post.content)  # Additional query! (not loaded)

# With select_related
posts = Post.objects.select_related('author').only(
    'id', 'title', 'author__username'
).all()
```

### 2. defer() - Exclude Specific Fields

```python
# Load everything except specific fields
posts = Post.objects.defer('content', 'metadata').all()
# Query: SELECT id, title, author_id, ... FROM posts (excludes content, metadata)

# Useful for large text/binary fields
posts = Post.objects.defer('content').all()
for post in posts:
    print(post.title)  # No issue
    # print(post.content)  # Would trigger additional query
```

## 📊 Aggregation and Annotation

### 1. Basic Aggregation

```python
from django.db.models import Count, Avg, Sum, Max, Min

# Count posts per author
authors = Author.objects.annotate(post_count=Count('posts'))
for author in authors:
    print(f"{author.username}: {author.post_count} posts")
# Query: SELECT *, COUNT(posts.id) FROM authors LEFT JOIN posts GROUP BY authors.id

# Average views
avg_views = Post.objects.aggregate(avg_views=Avg('views'))
# {'avg_views': 1234.56}

# Multiple aggregations
stats = Post.objects.aggregate(
    total=Count('id'),
    avg_views=Avg('views'),
    max_views=Max('views'),
    total_views=Sum('views')
)
```

### 2. Conditional Annotation

```python
from django.db.models import Q, Case, When, IntegerField

# Count published vs draft posts
authors = Author.objects.annotate(
    published_count=Count('posts', filter=Q(posts__published=True)),
    draft_count=Count('posts', filter=Q(posts__published=False))
)

# Conditional aggregation with Case/When
authors = Author.objects.annotate(
    post_status=Case(
        When(posts__count__gte=10, then=Value('prolific')),
        When(posts__count__gte=5, then=Value('active')),
        default=Value('beginner'),
        output_field=CharField()
    )
)
```

### 3. Subquery and OuterRef

```python
from django.db.models import OuterRef, Subquery

# Get latest comment for each post
latest_comment = Comment.objects.filter(
    post=OuterRef('pk')
).order_by('-created_at')

posts = Post.objects.annotate(
    latest_comment_text=Subquery(latest_comment.values('text')[:1])
)

# Efficient nested data
posts = Post.objects.annotate(
    comment_count=Count('comments'),
    latest_comment_date=Max('comments__created_at')
)
```

## 🚀 Advanced Optimization

### 1. values() and values_list()

```python
# Return dictionaries instead of model instances (faster)
posts = Post.objects.values('id', 'title', 'author__username')
# [{'id': 1, 'title': 'Post 1', 'author__username': 'john'}, ...]

# Return tuples
posts = Post.objects.values_list('id', 'title')
# [(1, 'Post 1'), (2, 'Post 2'), ...]

# Flat list (single field)
post_ids = Post.objects.values_list('id', flat=True)
# [1, 2, 3, 4, 5]

# Use when you only need specific fields (no need for model methods)
```

### 2. iterator() - Memory Efficient

```python
# Bad: Loads all 1 million posts into memory
posts = Post.objects.all()
for post in posts:
    process(post)  # Memory exhausted!

# Good: Stream results in chunks
for post in Post.objects.iterator(chunk_size=1000):
    process(post)  # Memory efficient

# Combine with only() for maximum efficiency
for post in Post.objects.only('id', 'title').iterator(chunk_size=1000):
    process(post)
```

### 3. exists() vs count() vs len()

```python
# Check if any posts exist

# Bad: Loads all posts into memory
if len(Post.objects.all()) > 0:  # DON'T DO THIS!
    pass

# Better: Database count
if Post.objects.count() > 0:  # SELECT COUNT(*) FROM posts
    pass

# Best: Existence check
if Post.objects.exists():  # SELECT 1 FROM posts LIMIT 1
    pass

# exists() is faster when you only need to know if records exist
```

## 🔍 Query Analysis Tools

### 1. QuerySet.explain()

```python
# Analyze query execution plan
posts = Post.objects.filter(author__username='john').select_related('author')
print(posts.explain())

# PostgreSQL EXPLAIN output
print(posts.explain(analyze=True, verbose=True))

# Shows:
# - Query plan
# - Index usage
# - Cost estimates
# - Actual execution time
```

### 2. Django Debug Toolbar

```bash
pip install django-debug-toolbar
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ...
]

INTERNAL_IPS = ['127.0.0.1']

# URLs
from django.urls import include

urlpatterns = [
    # ...
    path('__debug__/', include('debug_toolbar.urls')),
]
```

### 3. Connection Queries Inspection

```python
from django.db import connection
from django.test.utils import override_settings

@override_settings(DEBUG=True)
def my_view(request):
    # Execute queries
    posts = Post.objects.select_related('author').all()

    # Check executed queries
    queries = connection.queries
    print(f"Total queries: {len(queries)}")
    for query in queries:
        print(f"SQL: {query['sql']}")
        print(f"Time: {query['time']}")

    # Reset query log
    from django.db import reset_queries
    reset_queries()
```

## ✅ Best Practices

### 1. Optimization Checklist

```python
# 1. Use select_related for ForeignKey/OneToOne
posts = Post.objects.select_related('author', 'category')

# 2. Use prefetch_related for ManyToMany/Reverse FK
posts = Post.objects.prefetch_related('tags', 'comments')

# 3. Combine both when needed
posts = Post.objects.select_related('author').prefetch_related('tags')

# 4. Use only() for specific fields
posts = Post.objects.only('id', 'title')

# 5. Use values() when model instances not needed
post_data = Post.objects.values('id', 'title')

# 6. Use iterator() for large datasets
for post in Post.objects.iterator(chunk_size=1000):
    process(post)

# 7. Use exists() instead of count() for existence checks
if Post.objects.filter(author=user).exists():
    pass

# 8. Annotate instead of Python loops
authors = Author.objects.annotate(post_count=Count('posts'))
```

### 2. Common Patterns

```python
# Pattern: List view with related data
def post_list(request):
    posts = Post.objects.select_related('author', 'category') \
                        .prefetch_related('tags') \
                        .only('id', 'title', 'created_at', 'author__username', 'category__name')
    return render(request, 'posts.html', {'posts': posts})

# Pattern: Detail view with all related data
def post_detail(request, pk):
    post = Post.objects.select_related('author', 'category') \
                       .prefetch_related('tags', 'comments__author') \
                       .get(pk=pk)
    return render(request, 'post_detail.html', {'post': post})
```

## ❓ Interview Questions

### Q1: When do you use select_related vs prefetch_related?

**Answer**:

- **select_related**: ForeignKey and OneToOne (SQL JOIN, single query)
- **prefetch_related**: ManyToMany and reverse ForeignKey (separate queries, Python JOIN)

### Q2: What's the N+1 query problem and how do you fix it?

**Answer**:
N+1 occurs when you query a list of objects (1 query) then access related objects in a loop (N queries). Fix with `select_related()` or `prefetch_related()`.

### Q3: How do you optimize a query that loads 1 million records?

**Answer**:

1. Use `iterator(chunk_size=1000)` to stream results
2. Use `only()` to load minimal fields
3. Use `values()` if model instances not needed
4. Consider pagination
5. Add database indexes

## 📚 Summary

**Key Takeaways**:

1. select_related for ForeignKey/OneToOne (JOIN)
2. prefetch_related for ManyToMany/Reverse FK (separate queries)
3. Use only() to load specific fields
4. Use values() when model instances not needed
5. Use iterator() for large datasets
6. Use exists() instead of count() > 0
7. Annotate/aggregate at database level
8. Use Django Debug Toolbar to identify N+1 queries
9. Use explain() to analyze query plans
10. Always test query count in production-like data

Query optimization is critical for Django performance!
