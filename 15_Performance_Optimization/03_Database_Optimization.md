# 🗄️ Database Query Optimization

## Overview

Optimizing database queries for performance, understanding indexes, query execution plans, and common optimization patterns.

---

## Understanding Query Execution

### Query Execution Plan (EXPLAIN)

```python
from sqlalchemy import create_engine, text
from django.db import connection

# PostgreSQL - View execution plan
def explain_query(query_sql: str):
    """Analyze query execution plan"""
    with connection.cursor() as cursor:
        # EXPLAIN shows execution plan
        cursor.execute(f"EXPLAIN {query_sql}")
        plan = cursor.fetchall()

        # EXPLAIN ANALYZE actually runs query
        cursor.execute(f"EXPLAIN ANALYZE {query_sql}")
        analysis = cursor.fetchall()

        for row in analysis:
            print(row[0])

# Example usage
query = """
SELECT u.id, u.username, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.is_active = true
GROUP BY u.id
ORDER BY post_count DESC
LIMIT 10
"""

explain_query(query)

"""
Output might show:
- Seq Scan: Full table scan (slow)
- Index Scan: Using index (fast)
- Nested Loop: Join algorithm
- Hash Join: Better for larger datasets
- Cost estimates: startup cost..total cost
- Actual time: real execution time
"""
```

---

## N+1 Query Problem

### The Problem

```python
# ❌ BAD: N+1 queries
def get_users_with_posts():
    """Inefficient - causes N+1 queries"""
    users = User.objects.all()  # 1 query

    result = []
    for user in users:
        result.append({
            'username': user.username,
            'posts': list(user.posts.all())  # N queries (one per user)
        })

    return result

# If 100 users: 1 + 100 = 101 queries!
```

### The Solution

```python
from django.db.models import Prefetch

# ✅ GOOD: Use select_related for foreign keys
def get_posts_with_authors():
    """Efficient - single JOIN query"""
    # select_related: For foreign key relationships (one-to-one, many-to-one)
    posts = Post.objects.select_related('author').all()

    # Single query with JOIN:
    # SELECT * FROM posts INNER JOIN users ON posts.author_id = users.id

    for post in posts:
        print(post.author.username)  # No additional query!

    return posts

# ✅ GOOD: Use prefetch_related for reverse foreign keys
def get_users_with_posts_optimized():
    """Efficient - 2 queries total"""
    # prefetch_related: For reverse foreign keys, many-to-many
    users = User.objects.prefetch_related('posts').all()

    # Query 1: SELECT * FROM users
    # Query 2: SELECT * FROM posts WHERE user_id IN (user_ids)

    for user in users:
        for post in user.posts.all():  # No additional query!
            print(post.title)

    return users

# ✅ BEST: Custom prefetch with filtering
def get_active_users_with_recent_posts():
    """Optimized prefetch with filtering"""
    from datetime import datetime, timedelta

    recent_posts = Prefetch(
        'posts',
        queryset=Post.objects.filter(
            created_at__gte=datetime.now() - timedelta(days=30)
        ).order_by('-created_at')
    )

    users = User.objects.filter(
        is_active=True
    ).prefetch_related(recent_posts)

    return users

# Complex nested prefetching
def get_users_with_posts_and_comments():
    """Prefetch nested relationships"""
    users = User.objects.prefetch_related(
        'posts',                    # User's posts
        'posts__comments',          # Posts' comments
        'posts__comments__author'   # Comment authors
    ).all()

    # Total: 4 queries instead of potentially thousands
    return users
```

---

## Indexing Strategies

### Creating Effective Indexes

```python
from django.db import models

class User(models.Model):
    """User model with optimized indexes"""
    username = models.CharField(max_length=50, unique=True, db_index=True)
    email = models.EmailField(unique=True, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    is_active = models.BooleanField(default=True, db_index=True)

    class Meta:
        # Composite index for common query pattern
        indexes = [
            models.Index(fields=['is_active', 'created_at']),
            models.Index(fields=['username', 'email']),
        ]

        # Partial index (PostgreSQL)
        # Only index active users
        # More efficient for queries filtering by is_active=True

class Post(models.Model):
    """Post model with strategic indexes"""
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    created_at = models.DateTimeField(auto_now_add=True)
    view_count = models.IntegerField(default=0)
    is_published = models.BooleanField(default=False)

    class Meta:
        indexes = [
            # Index for filtering published posts by date
            models.Index(fields=['is_published', '-created_at']),

            # Index for author's posts
            models.Index(fields=['author', '-created_at']),

            # Index for popular posts
            models.Index(fields=['-view_count']),
        ]

# Raw SQL index creation
"""
-- B-tree index (default, good for equality and range)
CREATE INDEX idx_users_created_at ON users(created_at);

-- Partial index (only index subset)
CREATE INDEX idx_active_users ON users(is_active) WHERE is_active = true;

-- Composite index (multiple columns)
CREATE INDEX idx_user_activity ON users(is_active, created_at);

-- Text search index (PostgreSQL)
CREATE INDEX idx_posts_content_gin ON posts USING gin(to_tsvector('english', content));

-- Unique index
CREATE UNIQUE INDEX idx_users_email ON users(email);
"""

# When to add indexes
"""
✅ Add index when:
- Column in WHERE clause frequently
- Column in JOIN condition
- Column in ORDER BY
- Column has high cardinality (many unique values)
- Foreign keys (Django does this automatically)

❌ Don't add index when:
- Table has few rows (<1000)
- Column has low cardinality (few unique values like boolean)
- Table has many writes (indexes slow down INSERT/UPDATE)
- Column is rarely queried
"""
```

### Analyzing Index Usage

```python
# PostgreSQL - Check index usage
def analyze_index_usage():
    """Check which indexes are actually used"""
    query = """
    SELECT
        schemaname,
        tablename,
        indexname,
        idx_scan as times_used,
        idx_tup_read as tuples_read,
        idx_tup_fetch as tuples_fetched
    FROM pg_stat_user_indexes
    WHERE idx_scan = 0  -- Unused indexes
    ORDER BY schemaname, tablename;
    """

    with connection.cursor() as cursor:
        cursor.execute(query)
        unused_indexes = cursor.fetchall()

        for index in unused_indexes:
            print(f"Unused index: {index}")

# Check missing indexes
def find_missing_indexes():
    """Identify tables that might benefit from indexes"""
    query = """
    SELECT
        schemaname,
        tablename,
        seq_scan,
        seq_tup_read,
        idx_scan,
        seq_tup_read / seq_scan as avg_seq_tup_read
    FROM pg_stat_user_tables
    WHERE seq_scan > 0
    ORDER BY seq_tup_read DESC
    LIMIT 10;
    """

    # Tables with high seq_scan and seq_tup_read might need indexes
```

---

## Query Optimization Patterns

### Only Select What You Need

```python
# ❌ BAD: Select all columns
def get_user_names():
    """Inefficient - loads all columns"""
    users = User.objects.all()  # SELECT * FROM users
    return [user.username for user in users]

# ✅ GOOD: Select only needed columns
def get_user_names_optimized():
    """Efficient - only loads username"""
    users = User.objects.values_list('username', flat=True)
    # SELECT username FROM users
    return list(users)

# ✅ GOOD: Use only() for model instances
def get_users_minimal():
    """Load only specific fields as model instances"""
    users = User.objects.only('id', 'username', 'email')
    # Deferred loading for other fields
    return users

# ✅ GOOD: Use defer() to exclude large fields
def get_users_without_bio():
    """Exclude large text fields"""
    users = User.objects.defer('bio', 'profile_picture')
    return users
```

### Aggregation and Annotation

```python
from django.db.models import Count, Avg, Sum, Max, Min, F, Q

# ✅ GOOD: Database-level aggregation
def get_user_post_counts():
    """Count posts at database level"""
    users = User.objects.annotate(
        post_count=Count('posts')
    ).filter(
        post_count__gt=0
    ).order_by('-post_count')

    # Single query:
    # SELECT users.*, COUNT(posts.id) as post_count
    # FROM users LEFT JOIN posts ON users.id = posts.user_id
    # GROUP BY users.id
    # HAVING COUNT(posts.id) > 0
    # ORDER BY post_count DESC

    return users

# Complex aggregations
def get_user_statistics():
    """Multiple aggregations in single query"""
    from django.db.models import Avg, Count, Max

    stats = User.objects.aggregate(
        total_users=Count('id'),
        avg_posts=Avg('posts__view_count'),
        max_posts=Max('posts__view_count'),
        active_users=Count('id', filter=Q(is_active=True))
    )

    return stats

# F() expressions for field comparisons
def get_popular_posts():
    """Posts with more likes than views (using F expressions)"""
    posts = Post.objects.filter(
        like_count__gt=F('view_count') * 0.1
    )

    return posts

# Update with F() to avoid race conditions
def increment_view_count(post_id):
    """Atomic increment"""
    Post.objects.filter(id=post_id).update(
        view_count=F('view_count') + 1
    )
    # UPDATE posts SET view_count = view_count + 1 WHERE id = post_id
```

### Batch Operations

```python
# ❌ BAD: Loop with individual queries
def update_users_individually(user_ids):
    """Inefficient - N queries"""
    for user_id in user_ids:
        user = User.objects.get(id=user_id)
        user.is_active = False
        user.save()
    # 2N queries (N selects + N updates)

# ✅ GOOD: Bulk update
def update_users_bulk(user_ids):
    """Efficient - single query"""
    User.objects.filter(id__in=user_ids).update(is_active=False)
    # Single query: UPDATE users SET is_active = false WHERE id IN (...)

# ✅ GOOD: Bulk create
def create_users_bulk(user_data_list):
    """Efficient - single query"""
    users = [
        User(username=data['username'], email=data['email'])
        for data in user_data_list
    ]

    User.objects.bulk_create(users)
    # Single query: INSERT INTO users (...) VALUES (...), (...), (...)

# ✅ GOOD: Bulk update with different values
def bulk_update_different_values(users):
    """Update multiple objects with different values"""
    User.objects.bulk_update(
        users,
        ['is_active', 'updated_at'],
        batch_size=100
    )
```

### Pagination Optimization

```python
# ❌ BAD: Using OFFSET for large offsets
def get_page_offset(page, page_size=20):
    """Inefficient for large page numbers"""
    offset = (page - 1) * page_size
    users = User.objects.all()[offset:offset + page_size]
    # SELECT * FROM users LIMIT 20 OFFSET 10000
    # Database must scan and skip 10000 rows!
    return users

# ✅ GOOD: Cursor-based pagination
def get_page_cursor(last_id=None, page_size=20):
    """Efficient for any page"""
    query = User.objects.order_by('id')

    if last_id:
        query = query.filter(id__gt=last_id)

    users = query[:page_size]
    # SELECT * FROM users WHERE id > last_id ORDER BY id LIMIT 20
    # Uses index, no need to skip rows

    return users

# Django Pagination
from django.core.paginator import Paginator

def paginate_queryset(queryset, page_number, page_size=20):
    """Django's built-in pagination"""
    paginator = Paginator(queryset, page_size)
    page = paginator.get_page(page_number)

    return {
        'results': page.object_list,
        'count': paginator.count,
        'has_next': page.has_next(),
        'has_previous': page.has_previous()
    }

# FastAPI with cursor pagination
from fastapi import Query

@app.get("/users")
async def list_users(
    cursor: Optional[int] = Query(None),
    limit: int = Query(20, le=100)
):
    """API endpoint with cursor pagination"""
    query = db.query(User).order_by(User.id)

    if cursor:
        query = query.filter(User.id > cursor)

    users = query.limit(limit + 1).all()

    has_more = len(users) > limit
    if has_more:
        users = users[:limit]

    next_cursor = users[-1].id if has_more and users else None

    return {
        'users': users,
        'next_cursor': next_cursor,
        'has_more': has_more
    }
```

---

## Caching Query Results

### Query-Level Caching

```python
from django.core.cache import cache
import hashlib
import json

def cache_query_result(timeout=300):
    """Decorator to cache query results"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            # Generate cache key from function and arguments
            key_data = f"{func.__name__}:{args}:{kwargs}"
            cache_key = hashlib.md5(key_data.encode()).hexdigest()

            # Try cache first
            result = cache.get(cache_key)

            if result is None:
                # Cache miss - execute query
                result = func(*args, **kwargs)

                # Cache result
                cache.set(cache_key, result, timeout)

            return result

        return wrapper
    return decorator

@cache_query_result(timeout=600)
def get_popular_posts(limit=10):
    """Get popular posts (cached for 10 minutes)"""
    return Post.objects.filter(
        is_published=True
    ).order_by('-view_count')[:limit]

# Manual caching with invalidation
class PostService:
    """Service with cache invalidation"""

    @staticmethod
    def get_post(post_id):
        """Get post with caching"""
        cache_key = f'post:{post_id}'
        post = cache.get(cache_key)

        if post is None:
            post = Post.objects.get(id=post_id)
            cache.set(cache_key, post, 3600)

        return post

    @staticmethod
    def update_post(post_id, data):
        """Update post and invalidate cache"""
        post = Post.objects.get(id=post_id)

        for key, value in data.items():
            setattr(post, key, value)

        post.save()

        # Invalidate cache
        cache.delete(f'post:{post_id}')

        return post
```

---

## Database Connection Pooling

### Optimizing Connections

```python
# settings.py (Django)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
        'OPTIONS': {
            'connect_timeout': 10,
        },
        'CONN_MAX_AGE': 600,  # Connection pooling (10 minutes)
    }
}

# Use pgbouncer for better connection pooling
"""
pgbouncer configuration:

[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 100
default_pool_size = 20
"""

# SQLAlchemy connection pool
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql://user:pass@localhost/dbname',
    poolclass=QueuePool,
    pool_size=20,          # Number of persistent connections
    max_overflow=10,       # Additional connections when pool full
    pool_timeout=30,       # Wait time for available connection
    pool_recycle=3600,     # Recycle connections after 1 hour
    pool_pre_ping=True,    # Verify connection before using
)
```

---

## Best Practices

### ✅ Do's:

1. **Use EXPLAIN ANALYZE** to understand queries
2. **Add indexes** on frequently queried columns
3. **Use select_related** for foreign keys
4. **Use prefetch_related** for reverse FKs
5. **Select only needed** columns (only/values)
6. **Use bulk operations** for multiple objects
7. **Implement caching** for expensive queries
8. **Use cursor pagination** for large datasets
9. **Monitor slow queries** in production
10. **Use connection pooling**

### ❌ Don'ts:

1. **Don't fetch all** then filter in Python
2. **Don't create** N+1 query problems
3. **Don't over-index** (slows writes)
4. **Don't use OFFSET** for large offsets
5. **Don't ignore** query execution plans
6. **Don't fetch unused** columns
7. **Don't loop** individual queries

---

## Interview Questions

### Q1: What is the N+1 query problem?

**Answer**: Executing N additional queries in a loop:

- **Problem**: 1 query to get users, N queries for each user's posts
- **Solution**: Use select_related or prefetch_related
- **Impact**: 100 users = 101 queries vs 2 queries
- **Detection**: Django Debug Toolbar, query logging

### Q2: select_related vs prefetch_related?

**Answer**:

- **select_related**: JOIN in single query (foreign key, one-to-one)
- **prefetch_related**: Separate queries (reverse FK, many-to-many)
- **select_related**: Better for small related data
- **prefetch_related**: Better for large related sets
  Use based on relationship type.

### Q3: How to optimize slow queries?

**Answer**:

1. **EXPLAIN ANALYZE**: See execution plan
2. **Add indexes**: On WHERE/JOIN/ORDER BY columns
3. **Reduce columns**: only(), values_list()
4. **Fix N+1**: Use select/prefetch_related
5. **Cache results**: Redis for frequent queries
6. **Paginate**: Don't fetch all at once

### Q4: When to add database indexes?

**Answer**:

- **Add when**: Frequent WHERE/JOIN/ORDER BY, high cardinality
- **Don't add when**: Low cardinality, many writes, small table
- **Types**: B-tree (default), GIN (text search), partial
- **Monitor**: Check unused indexes, add for slow queries

### Q5: How does cursor pagination work?

**Answer**:

- **Offset-based**: LIMIT 20 OFFSET 10000 (scans 10000 rows)
- **Cursor-based**: WHERE id > last_id (uses index)
- **Benefit**: Constant time regardless of page
- **Trade-off**: Can't jump to arbitrary page
  Use cursor for large datasets.

---

## Summary

Database optimization essentials:

- **EXPLAIN ANALYZE**: Understand query execution
- **Indexes**: Speed up WHERE/JOIN/ORDER BY
- **select/prefetch_related**: Avoid N+1 queries
- **Bulk operations**: Reduce query count
- **Caching**: Redis for expensive queries
- **Pagination**: Cursor-based for large data
- **Connection pooling**: Reuse connections

Optimize queries, not hardware! 🗄️
