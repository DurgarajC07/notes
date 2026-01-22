# Django Query Optimization

## 📖 Concept Explanation

Query optimization improves database performance by reducing query count, limiting data transfer, and using efficient SQL patterns. The N+1 query problem is the most common issue.

### N+1 Query Problem

```
Without optimization:
1 query to get posts
+ N queries to get each post's author
= 1 + N queries total

With optimization:
1 query with JOIN
= 1 query total
```

**Optimization Techniques**:

- select_related() - SQL JOIN for ForeignKey/OneToOne
- prefetch_related() - Separate queries for ManyToMany/Reverse FK
- only() / defer() - Limit fetched columns
- Indexing - Speed up lookups
- Raw SQL / Database functions - Complex operations

## 🧠 select_related() - SQL JOIN

### 1. Basic select_related()

```python
# BAD: N+1 queries
posts = Post.objects.all()  # 1 query
for post in posts:
    print(post.author.username)  # N queries (one per post)
    print(post.category.name)    # N more queries

# GOOD: 1 query with JOIN
posts = Post.objects.select_related('author', 'category')  # 1 query
for post in posts:
    print(post.author.username)  # No extra queries
    print(post.category.name)    # No extra queries
```

### 2. Chaining Related Objects

```python
# Follow FK chain
posts = Post.objects.select_related(
    'author',           # User
    'author__profile',  # UserProfile
    'category',         # Category
    'category__parent'  # Parent category
)

# Now all related data is loaded in single query
for post in posts:
    print(post.author.profile.bio)
    print(post.category.parent.name)
```

### 3. select_related() with Filtering

```python
# Efficient filtering with JOIN
posts = Post.objects.select_related('author').filter(
    author__username='john',
    author__is_active=True
)
```

## 🔗 prefetch_related() - Separate Queries

### 1. Basic prefetch_related()

```python
# BAD: N+1 queries for ManyToMany
posts = Post.objects.all()  # 1 query
for post in posts:
    for tag in post.tags.all():  # N queries
        print(tag.name)

# GOOD: 2 queries total
posts = Post.objects.prefetch_related('tags')  # 2 queries total
for post in posts:
    for tag in post.tags.all():  # No extra queries
        print(tag.name)
```

### 2. Prefetch Multiple Relations

```python
posts = Post.objects.prefetch_related(
    'tags',      # ManyToMany
    'comments',  # Reverse FK
    'likes'      # Reverse FK
)

for post in posts:
    print(f"Tags: {post.tags.count()}")
    print(f"Comments: {post.comments.count()}")
    print(f"Likes: {post.likes.count()}")
```

### 3. Custom Prefetch

```python
from django.db.models import Prefetch

# Prefetch with custom queryset
posts = Post.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.filter(approved=True).select_related('author'),
        to_attr='approved_comments'
    )
)

for post in posts:
    for comment in post.approved_comments:  # Use custom attribute
        print(f"{comment.author.username}: {comment.text}")
```

### 4. Combining select_related() and prefetch_related()

```python
# Optimize complex relationships
posts = Post.objects.select_related(
    'author',           # FK (JOIN)
    'category'          # FK (JOIN)
).prefetch_related(
    'tags',             # M2M (separate query)
    'comments__author'  # Reverse FK + FK
)

for post in posts:
    print(post.author.username)      # No extra query
    print(post.category.name)        # No extra query
    for tag in post.tags.all():      # No extra query
        print(tag.name)
    for comment in post.comments.all():  # No extra query
        print(comment.author.username)   # No extra query
```

## 📊 only() and defer()

### 1. only() - Fetch Specific Fields

```python
# Only fetch needed fields
posts = Post.objects.only('id', 'title', 'created_at')

# Accessing deferred fields triggers extra queries
for post in posts:
    print(post.title)    # No extra query
    print(post.content)  # Extra query!

# With select_related
posts = Post.objects.select_related('author').only(
    'id',
    'title',
    'author__id',
    'author__username'
)
```

### 2. defer() - Exclude Specific Fields

```python
# Fetch all fields except content
posts = Post.objects.defer('content', 'metadata')

for post in posts:
    print(post.title)    # No extra query
    print(post.content)  # Extra query!

# Defer large fields
posts = Post.objects.defer('content').select_related('author')
```

## 🎯 Advanced Optimization Techniques

### 1. values() and values_list()

```python
# Get dictionary of specific fields (no model instances)
posts = Post.objects.values('id', 'title', 'author__username')
# Returns: [{'id': 1, 'title': 'My Post', 'author__username': 'john'}, ...]

# Get tuples
posts = Post.objects.values_list('id', 'title')
# Returns: [(1, 'My Post'), (2, 'Another Post'), ...]

# Flat list for single field
post_ids = Post.objects.values_list('id', flat=True)
# Returns: [1, 2, 3, 4, 5, ...]
```

### 2. iterator() for Large QuerySets

```python
# BAD: Loads all into memory
for post in Post.objects.all():
    process(post)

# GOOD: Stream results in chunks
for post in Post.objects.iterator(chunk_size=1000):
    process(post)

# Note: iterator() doesn't cache results
```

### 3. exists() vs count()

```python
# Check if exists
if Post.objects.filter(author=user).exists():  # Fast
    print("User has posts")

# Don't use count() for existence check
if Post.objects.filter(author=user).count() > 0:  # Slower
    print("User has posts")

# Use count() only when you need the actual count
total_posts = Post.objects.count()
```

### 4. Database Functions

```python
from django.db.models.functions import Length, Lower, Upper, Concat

# String operations in database
posts = Post.objects.annotate(
    title_length=Length('title'),
    title_lower=Lower('title')
).filter(title_length__gt=50)

# Concatenation
posts = Post.objects.annotate(
    full_name=Concat('author__first_name', Value(' '), 'author__last_name')
)
```

## 🏗️ Real-World Optimization Examples

### 1. Blog Post List Optimization

```python
def get_blog_posts():
    """Optimized blog post query"""
    return Post.objects.select_related(
        'author',
        'category'
    ).prefetch_related(
        'tags',
        Prefetch(
            'comments',
            queryset=Comment.objects.filter(
                approved=True
            ).select_related('author').only(
                'id', 'text', 'created_at', 'author__username'
            )
        )
    ).only(
        'id', 'title', 'slug', 'excerpt', 'featured_image',
        'created_at', 'author__username', 'category__name'
    ).filter(
        published=True
    ).order_by('-created_at')[:20]
```

### 2. Dashboard Statistics

```python
from django.db.models import Count, Sum, Avg

def get_dashboard_stats(user):
    """Optimized dashboard query"""
    # Single aggregation query
    stats = user.posts.aggregate(
        total_posts=Count('id'),
        total_views=Sum('views'),
        avg_views=Avg('views'),
        total_comments=Count('comments'),
        total_likes=Count('likes')
    )

    # Prefetch recent activity
    recent_posts = user.posts.select_related('category').prefetch_related(
        'tags'
    ).only(
        'id', 'title', 'views', 'created_at', 'category__name'
    ).order_by('-created_at')[:5]

    return {
        'stats': stats,
        'recent_posts': recent_posts
    }
```

### 3. E-commerce Product Listing

```python
def get_product_list(category=None):
    """Optimized product listing"""
    queryset = Product.objects.select_related(
        'category',
        'brand'
    ).prefetch_related(
        'images',
        'reviews'
    ).annotate(
        avg_rating=Avg('reviews__rating'),
        review_count=Count('reviews')
    ).only(
        'id', 'name', 'price', 'stock',
        'category__name', 'brand__name'
    )

    if category:
        queryset = queryset.filter(category=category)

    return queryset.filter(is_active=True).order_by('-created_at')
```

## 📈 Query Analysis

### 1. Inspecting Queries

```python
from django.db import connection
from django.db import reset_queries

# Enable query logging
reset_queries()

# Your code
posts = Post.objects.select_related('author').all()
for post in posts:
    print(post.author.username)

# Print queries
print(f"Total queries: {len(connection.queries)}")
for query in connection.queries:
    print(query['sql'])
    print(f"Time: {query['time']}s\n")
```

### 2. EXPLAIN Query Plan

```python
# Get query execution plan
queryset = Post.objects.select_related('author').filter(published=True)

# PostgreSQL
print(queryset.explain())

# With options
print(queryset.explain(verbose=True, analyze=True))
```

### 3. Django Debug Toolbar

```python
# settings.py
INSTALLED_APPS = [
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
]

INTERNAL_IPS = ['127.0.0.1']

# Shows SQL queries, execution time, duplicates
```

## 🔍 Database Indexing

### 1. Field Indexes

```python
class Post(models.Model):
    title = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(unique=True)  # Automatically indexed
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    published = models.BooleanField(default=False, db_index=True)
```

### 2. Composite Indexes

```python
class Post(models.Model):
    title = models.CharField(max_length=200)
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=['published', 'created_at']),
            models.Index(fields=['author', 'published']),
            models.Index(fields=['-created_at']),  # Descending
        ]
```

### 3. Partial Indexes (PostgreSQL)

```python
from django.db.models import Q

class Post(models.Model):
    class Meta:
        indexes = [
            models.Index(
                fields=['created_at'],
                condition=Q(published=True),
                name='published_posts_idx'
            )
        ]
```

## ✅ Best Practices

### 1. Always Use select_related() for ForeignKeys

```python
# BAD
posts = Post.objects.all()
for post in posts:
    print(post.author.username)  # N queries

# GOOD
posts = Post.objects.select_related('author')
for post in posts:
    print(post.author.username)  # 1 query
```

### 2. Use prefetch_related() for ManyToMany

```python
# BAD
posts = Post.objects.all()
for post in posts:
    for tag in post.tags.all():  # N queries
        print(tag.name)

# GOOD
posts = Post.objects.prefetch_related('tags')
for post in posts:
    for tag in post.tags.all():  # 2 queries total
        print(tag.name)
```

### 3. Limit QuerySet Early

```python
# BAD: Fetch all, then slice in Python
all_posts = Post.objects.all()
recent_posts = list(all_posts)[:10]

# GOOD: Limit in database
recent_posts = Post.objects.all()[:10]
```

## 🔐 Security & Performance

### 1. Pagination

```python
from django.core.paginator import Paginator

def paginated_posts(request):
    """Efficient pagination"""
    posts = Post.objects.select_related('author').order_by('-created_at')

    paginator = Paginator(posts, 20)  # 20 per page
    page_number = request.GET.get('page', 1)
    page_obj = paginator.get_page(page_number)

    return page_obj
```

### 2. Caching QuerySets

```python
from django.core.cache import cache

def get_popular_posts():
    """Cache expensive query"""
    cache_key = 'popular_posts'
    posts = cache.get(cache_key)

    if posts is None:
        posts = list(Post.objects.select_related('author').annotate(
            score=F('views') + F('likes') * 10
        ).order_by('-score')[:10])

        cache.set(cache_key, posts, 3600)  # Cache 1 hour

    return posts
```

## ❓ Interview Questions

### Q1: What is the N+1 query problem?

**Answer**:
Executing 1 query to fetch objects, then N additional queries to fetch related objects. Fixed with select_related() or prefetch_related().

### Q2: Difference between select_related() and prefetch_related()?

**Answer**:

- **select_related()**: SQL JOIN for ForeignKey/OneToOne (1 query)
- **prefetch_related()**: Separate queries for ManyToMany/Reverse FK (2 queries)

### Q3: When should you use only() vs defer()?

**Answer**:

- **only()**: When you need few fields
- **defer()**: When you need most fields except few large ones

## 📚 Summary

**Key Takeaways**:

1. Use select_related() for ForeignKey/OneToOne
2. Use prefetch_related() for ManyToMany/Reverse FK
3. only() to fetch specific fields
4. defer() to exclude large fields
5. values() for dict, values_list() for tuples
6. iterator() for large querysets
7. exists() over count() for boolean checks
8. Add indexes on frequently queried fields
9. Use explain() to analyze queries
10. Cache expensive queries

Django query optimization is essential for performance!
