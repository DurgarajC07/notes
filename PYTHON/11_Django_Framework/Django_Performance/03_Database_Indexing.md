# Django Database Indexing

## 📖 Concept Explanation

Database indexes are data structures that improve query performance by allowing the database to find rows quickly without scanning the entire table. Like a book index, they map values to their locations.

### Index Types

```
B-Tree Index (Default): Fast lookups, range queries, ordering
Hash Index: Fast equality lookups only
GIN/GiST (PostgreSQL): Full-text search, arrays, JSON
Partial Index: Index subset of rows
Covering Index: Index includes all needed columns
```

**Benefits**:

- Faster SELECT queries (10-1000x speedup)
- Faster ORDER BY operations
- Faster JOIN operations

**Drawbacks**:

- Slower INSERT/UPDATE/DELETE (index must be updated)
- Additional storage space
- Maintenance overhead

## 🧠 Django Index Types

### 1. Field-Level Index (db_index=True)

```python
# models.py
class Post(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Single-column index
    slug = models.SlugField(unique=True, db_index=True)  # Unique constraint + index
    author = models.ForeignKey(User, on_delete=models.CASCADE, db_index=True)
    published = models.BooleanField(default=False, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)

# Generated SQL (PostgreSQL):
# CREATE INDEX "myapp_post_title_idx" ON "myapp_post" ("title");
# CREATE INDEX "myapp_post_author_id_idx" ON "myapp_post" ("author_id");
```

### 2. Meta Indexes (Composite & Advanced)

```python
# models.py
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    views = models.IntegerField(default=0)

    class Meta:
        indexes = [
            # Composite index (multiple columns)
            models.Index(fields=['author', 'created_at'], name='author_created_idx'),

            # Composite index for common filter
            models.Index(fields=['published', 'created_at'], name='published_created_idx'),

            # Descending order index
            models.Index(fields=['-created_at'], name='created_desc_idx'),

            # Multiple columns with mixed order
            models.Index(fields=['category', '-views'], name='category_views_idx'),
        ]

# Generated SQL:
# CREATE INDEX "author_created_idx" ON "myapp_post" ("author_id", "created_at");
# CREATE INDEX "published_created_idx" ON "myapp_post" ("published", "created_at");
# CREATE INDEX "created_desc_idx" ON "myapp_post" ("created_at" DESC);
```

### 3. Unique Indexes

```python
class Post(models.Model):
    slug = models.SlugField(unique=True)  # Automatically creates unique index

    class Meta:
        # Unique together (composite unique index)
        unique_together = [['author', 'slug']]

        # Or with constraints (Django 2.2+)
        constraints = [
            models.UniqueConstraint(
                fields=['author', 'slug'],
                name='unique_author_slug'
            )
        ]
```

## 🎯 Advanced Indexing

### 1. Partial Indexes (PostgreSQL)

```python
from django.db import models
from django.db.models import Q

class Post(models.Model):
    title = models.CharField(max_length=200)
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            # Index only published posts
            models.Index(
                fields=['created_at'],
                name='published_created_idx',
                condition=Q(published=True)
            ),

            # Index only recent posts (last 30 days)
            models.Index(
                fields=['created_at'],
                name='recent_posts_idx',
                condition=Q(created_at__gte=models.functions.Now() - models.DurationField(days=30))
            ),
        ]

# Generated SQL (PostgreSQL):
# CREATE INDEX "published_created_idx" ON "myapp_post" ("created_at")
# WHERE "published" = TRUE;
```

### 2. Expression Indexes (PostgreSQL)

```python
from django.db.models import F
from django.db.models.functions import Lower

class User(models.Model):
    username = models.CharField(max_length=150)
    email = models.EmailField()

    class Meta:
        indexes = [
            # Case-insensitive search on username
            models.Index(Lower('username'), name='username_lower_idx'),

            # Case-insensitive search on email
            models.Index(Lower('email'), name='email_lower_idx'),
        ]

# Usage:
users = User.objects.filter(username__lower='john')
# Uses index: username_lower_idx
```

### 3. Covering Indexes (Include Columns)

```python
# PostgreSQL only
from django.contrib.postgres.indexes import Index

class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            # Index on created_at, includes title and slug (no table lookup needed)
            Index(
                fields=['created_at'],
                name='created_covering_idx',
                include=['title', 'slug']
            ),
        ]

# Query using covering index (no table access needed):
# SELECT created_at, title, slug FROM posts WHERE created_at > '2024-01-01';
```

### 4. GIN Indexes (Full-Text Search)

```python
from django.contrib.postgres.indexes import GinIndex
from django.contrib.postgres.search import SearchVectorField

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    search_vector = SearchVectorField(null=True)

    class Meta:
        indexes = [
            GinIndex(fields=['search_vector'], name='search_idx'),
        ]

# Update search vector
from django.contrib.postgres.search import SearchVector

Post.objects.update(search_vector=SearchVector('title', 'content'))

# Search using GIN index
from django.contrib.postgres.search import SearchQuery

results = Post.objects.filter(search_vector=SearchQuery('django performance'))
```

## 🔍 Index Strategy

### 1. When to Add Indexes

```python
# Index columns used in:

# 1. WHERE clauses
Post.objects.filter(author_id=1)  # Index: author_id
Post.objects.filter(published=True)  # Index: published

# 2. ORDER BY clauses
Post.objects.order_by('-created_at')  # Index: created_at DESC

# 3. JOIN columns (ForeignKey)
Post.objects.select_related('author')  # Index: author_id (automatic)

# 4. GROUP BY columns
Post.objects.values('category').annotate(Count('id'))  # Index: category

# 5. Composite filters (use composite index)
Post.objects.filter(author_id=1, published=True).order_by('-created_at')
# Index: (author_id, published, created_at DESC)
```

### 2. Index Column Order (Leftmost Prefix)

```python
class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            # Index: (author, category, published, created_at)
            models.Index(
                fields=['author', 'category', 'published', 'created_at'],
                name='compound_idx'
            ),
        ]

# This index supports queries:
# ✅ WHERE author = 1
# ✅ WHERE author = 1 AND category = 2
# ✅ WHERE author = 1 AND category = 2 AND published = True
# ✅ WHERE author = 1 AND category = 2 AND published = True ORDER BY created_at

# This index does NOT fully support:
# ❌ WHERE category = 2 (doesn't start with author)
# ❌ WHERE published = True (doesn't start with author)
# ⚠️ WHERE author = 1 AND published = True (skips category, less efficient)
```

### 3. Analyze Query Performance

```python
# Use EXPLAIN to check index usage
posts = Post.objects.filter(author_id=1, published=True).order_by('-created_at')
print(posts.explain(analyze=True))

# PostgreSQL EXPLAIN output shows:
# - Seq Scan (bad: full table scan)
# - Index Scan (good: using index)
# - Index Only Scan (best: covering index)
# - Bitmap Index Scan (good: multiple indexes)
```

## ✅ Index Best Practices

### 1. Don't Over-Index

```python
# Bad: Too many indexes
class Post(models.Model):
    title = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(db_index=True)
    author = models.ForeignKey(User, on_delete=models.CASCADE, db_index=True)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True, db_index=True)
    published = models.BooleanField(default=False, db_index=True)
    featured = models.BooleanField(default=False, db_index=True)
    views = models.IntegerField(default=0, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    updated_at = models.DateTimeField(auto_now=True, db_index=True)
    # 9 indexes! Slows down writes significantly

# Good: Strategic indexes based on queries
class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)  # Unique creates index
    author = models.ForeignKey(User, on_delete=models.CASCADE)  # FK creates index
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    published = models.BooleanField(default=False)
    featured = models.BooleanField(default=False)
    views = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        indexes = [
            # Common query pattern
            models.Index(fields=['published', '-created_at'], name='published_created_idx'),
            # Featured posts ordering
            models.Index(fields=['featured', '-views'], name='featured_views_idx'),
        ]
    # 4 indexes total (2 automatic + 2 strategic)
```

### 2. Index Maintenance

```python
# Monitor index usage (PostgreSQL)
from django.db import connection

def check_index_usage():
    """Check which indexes are being used"""
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT
                schemaname,
                tablename,
                indexname,
                idx_scan,
                idx_tup_read,
                idx_tup_fetch
            FROM pg_stat_user_indexes
            WHERE schemaname = 'public'
            ORDER BY idx_scan;
        """)

        rows = cursor.fetchall()
        for row in rows:
            print(f"Index: {row[2]}, Scans: {row[3]}")
            # If idx_scan = 0, index is never used (consider removing)

# Find unused indexes
def find_unused_indexes():
    """Find indexes that are never used"""
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT
                schemaname,
                tablename,
                indexname
            FROM pg_stat_user_indexes
            WHERE idx_scan = 0
            AND indexname NOT LIKE '%_pkey';
        """)

        unused = cursor.fetchall()
        return unused
```

### 3. Rebuild Indexes Periodically

```python
# Management command: python manage.py reindex
from django.core.management.base import BaseCommand
from django.db import connection

class Command(BaseCommand):
    help = 'Rebuild all indexes'

    def handle(self, *args, **options):
        with connection.cursor() as cursor:
            # PostgreSQL
            cursor.execute("REINDEX DATABASE mydb;")

            # Or reindex specific table
            cursor.execute("REINDEX TABLE myapp_post;")

        self.stdout.write(self.style.SUCCESS('Indexes rebuilt'))
```

## 🧪 Testing Index Impact

```python
# tests.py
from django.test import TestCase
from django.db import connection
from django.test.utils import override_settings

class IndexPerformanceTest(TestCase):
    def test_query_uses_index(self):
        """Verify query uses expected index"""

        # Create test data
        for i in range(1000):
            Post.objects.create(
                title=f'Post {i}',
                published=i % 2 == 0
            )

        # Query with EXPLAIN
        posts = Post.objects.filter(published=True)
        explain = posts.explain(analyze=True)

        # Check that index is used
        self.assertIn('Index Scan', explain)
        self.assertIn('published_created_idx', explain)

    def test_query_performance(self):
        """Compare query performance with and without index"""
        import time

        # Create test data
        for i in range(10000):
            Post.objects.create(title=f'Post {i}', published=i % 2 == 0)

        # Query with index
        start = time.time()
        list(Post.objects.filter(published=True))
        with_index = time.time() - start

        # This test requires dropping and recreating index
        # to compare performance (complex setup)

        self.assertLess(with_index, 0.1)  # Should be fast
```

## ❓ Interview Questions

### Q1: When should you add a database index?

**Answer**:
Add indexes on columns used in:

1. WHERE clauses (filtering)
2. ORDER BY clauses (sorting)
3. JOIN operations (ForeignKey automatically indexed)
4. GROUP BY clauses

Don't over-index: Each index slows writes and uses storage.

### Q2: What's the difference between a single-column and composite index?

**Answer**:

- **Single-column**: Index on one column (e.g., `author_id`)
- **Composite**: Index on multiple columns (e.g., `author_id, created_at`)

Composite indexes support queries using leftmost prefix. Index `(A, B, C)` supports queries on `A`, `A+B`, `A+B+C`, but NOT `B` or `C` alone.

### Q3: Why don't you index every column?

**Answer**:
Indexes have costs:

1. Slower INSERT/UPDATE/DELETE (must update index)
2. Additional storage (can double database size)
3. Maintenance overhead

Index strategically based on actual query patterns.

## 📚 Summary

**Key Takeaways**:

1. Use `db_index=True` for single-column indexes
2. Use `Meta.indexes` for composite indexes
3. ForeignKey automatically creates index
4. Index columns used in WHERE, ORDER BY, JOIN
5. Composite index order matters (leftmost prefix)
6. Use partial indexes to index subsets
7. Use covering indexes for frequently accessed columns
8. Monitor index usage and remove unused indexes
9. Don't over-index (slows writes)
10. Use EXPLAIN to verify index usage

Proper indexing is crucial for database performance!
