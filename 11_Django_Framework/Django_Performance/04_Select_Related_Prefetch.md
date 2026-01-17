# Django select_related & prefetch_related Deep Dive

## 📖 Concept Explanation

`select_related` and `prefetch_related` are Django ORM methods that optimize database queries by reducing the number of queries needed to access related objects. They solve the N+1 query problem.

### The N+1 Problem

```python
# ❌ N+1 Problem: 1 query + N queries = 101 total queries
posts = Post.objects.all()  # 1 query to get 100 posts
for post in posts:
    print(post.author.username)  # N queries (1 per post)

# Total: 1 + 100 = 101 queries
```

### Solution

```python
# ✅ select_related: 1 query with JOIN
posts = Post.objects.select_related('author').all()
for post in posts:
    print(post.author.username)  # No additional queries

# Total: 1 query
```

## 🧠 select_related (SQL JOIN)

Used for ForeignKey and OneToOneField (forward lookups). Creates SQL JOIN.

### Basic Usage

```python
# models.py
class Author(models.Model):
    username = models.CharField(max_length=150)
    email = models.EmailField()

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)

# Without select_related
posts = Post.objects.all()
for post in posts:
    print(post.author.username)
# Query 1: SELECT * FROM post
# Query 2-101: SELECT * FROM author WHERE id = ?

# With select_related
posts = Post.objects.select_related('author').all()
for post in posts:
    print(post.author.username)
# Query 1: SELECT * FROM post INNER JOIN author ON post.author_id = author.id
```

### Multiple Relations

```python
# models.py
class Category(models.Model):
    name = models.CharField(max_length=100)

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)

# Select multiple relations
posts = Post.objects.select_related('author', 'category').all()

# Generated SQL:
# SELECT post.*, author.*, category.*
# FROM post
# INNER JOIN author ON post.author_id = author.id
# LEFT OUTER JOIN category ON post.category_id = category.id
```

### Nested Relations (Double Underscore)

```python
# models.py
class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    bio = models.TextField()
    avatar = models.ImageField()

class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    title = models.CharField(max_length=200)

# Select nested relations
posts = Post.objects.select_related('author__profile').all()
for post in posts:
    print(post.author.profile.bio)  # No additional queries

# Generated SQL joins 3 tables:
# SELECT post.*, user.*, profile.*
# FROM post
# INNER JOIN user ON post.author_id = user.id
# INNER JOIN profile ON user.id = profile.user_id
```

## 🧠 prefetch_related (Separate Queries)

Used for ManyToManyField and reverse ForeignKey. Creates separate queries and joins in Python.

### Basic Usage

```python
# models.py
class Tag(models.Model):
    name = models.CharField(max_length=50)

class Post(models.Model):
    title = models.CharField(max_length=200)
    tags = models.ManyToManyField(Tag)

# Without prefetch_related
posts = Post.objects.all()
for post in posts:
    print(post.tags.all())
# Query 1: SELECT * FROM post (gets 100 posts)
# Query 2-101: SELECT * FROM tag WHERE post_tag.post_id = ? (1 per post)

# With prefetch_related
posts = Post.objects.prefetch_related('tags').all()
for post in posts:
    print(post.tags.all())
# Query 1: SELECT * FROM post
# Query 2: SELECT * FROM tag WHERE id IN (1, 2, 3, ..., 500)
# Query 3: SELECT * FROM post_tag WHERE post_id IN (1, 2, 3, ..., 100)
# Total: 3 queries regardless of post count
```

### Reverse ForeignKey

```python
# models.py
class Author(models.Model):
    username = models.CharField(max_length=150)

class Post(models.Model):
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    title = models.CharField(max_length=200)

# Get authors with their posts (reverse FK)
authors = Author.objects.prefetch_related('post_set').all()
for author in authors:
    for post in author.post_set.all():  # No additional queries
        print(post.title)

# Query 1: SELECT * FROM author
# Query 2: SELECT * FROM post WHERE author_id IN (1, 2, 3, ...)
```

### Custom Related Name

```python
# models.py
class Post(models.Model):
    author = models.ForeignKey(
        Author,
        on_delete=models.CASCADE,
        related_name='posts'  # Custom related name
    )

# Use custom related name
authors = Author.objects.prefetch_related('posts').all()
for author in authors:
    for post in author.posts.all():  # Use 'posts' instead of 'post_set'
        print(post.title)
```

## 🎯 Advanced Techniques

### 1. Prefetch() Object with Custom QuerySet

```python
from django.db.models import Prefetch

# Only prefetch published posts
authors = Author.objects.prefetch_related(
    Prefetch(
        'posts',
        queryset=Post.objects.filter(published=True).order_by('-created_at')
    )
).all()

for author in authors:
    for post in author.posts.all():  # Only published posts, no query
        print(post.title)
```

### 2. Prefetch with to_attr

```python
from django.db.models import Prefetch

# Store prefetched results in custom attribute
authors = Author.objects.prefetch_related(
    Prefetch(
        'posts',
        queryset=Post.objects.filter(published=True),
        to_attr='published_posts'  # Custom attribute name
    )
).all()

for author in authors:
    for post in author.published_posts:  # Access via custom attribute
        print(post.title)

    # Can still access all posts (with additional query)
    all_posts = author.posts.all()  # New query
```

### 3. Nested Prefetching

```python
# models.py
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    content = models.TextField()

# Prefetch posts with comments and comment authors
authors = Author.objects.prefetch_related(
    'posts__comments__author'  # Double underscore for nested
).all()

for author in authors:
    for post in author.posts.all():
        for comment in post.comments.all():
            print(comment.author.username)  # No queries

# Query 1: SELECT * FROM author
# Query 2: SELECT * FROM post WHERE author_id IN (...)
# Query 3: SELECT * FROM comment WHERE post_id IN (...)
# Query 4: SELECT * FROM user WHERE id IN (...)
```

### 4. Combining select_related and prefetch_related

```python
# Get posts with authors (FK) and tags (M2M)
posts = Post.objects.select_related('author').prefetch_related('tags').all()

for post in posts:
    print(post.author.username)  # select_related (JOIN)
    for tag in post.tags.all():  # prefetch_related (separate queries)
        print(tag.name)

# Query 1: SELECT post.*, author.* FROM post INNER JOIN author
# Query 2: SELECT * FROM tag WHERE id IN (...)
# Query 3: SELECT * FROM post_tag WHERE post_id IN (...)
```

### 5. Complex Prefetch with Custom QuerySet

```python
from django.db.models import Prefetch, Count

# Prefetch comments with reply count
posts = Post.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.annotate(
            reply_count=Count('replies')
        ).select_related('author'),
        to_attr='annotated_comments'
    )
).all()

for post in posts:
    for comment in post.annotated_comments:
        print(f"{comment.author.username}: {comment.reply_count} replies")
```

## ⚡ Performance Comparison

### Benchmark Setup

```python
# Create test data
def setup_data():
    # 100 authors
    authors = [Author.objects.create(username=f'author{i}') for i in range(100)]

    # 1000 posts (10 per author)
    posts = []
    for author in authors:
        for i in range(10):
            posts.append(Post.objects.create(
                title=f'Post {i}',
                author=author
            ))

    # 5000 comments (5 per post)
    for post in posts:
        for i in range(5):
            Comment.objects.create(
                post=post,
                author=post.author,
                content=f'Comment {i}'
            )
```

### Benchmark Results

```python
import time
from django.db import connection, reset_queries
from django.test.utils import override_settings

@override_settings(DEBUG=True)  # Enable query logging
def benchmark():
    reset_queries()

    # Test 1: No optimization
    start = time.time()
    posts = Post.objects.all()
    for post in posts:
        _ = post.author.username
    time1 = time.time() - start
    queries1 = len(connection.queries)

    # Test 2: select_related
    reset_queries()
    start = time.time()
    posts = Post.objects.select_related('author').all()
    for post in posts:
        _ = post.author.username
    time2 = time.time() - start
    queries2 = len(connection.queries)

    # Test 3: prefetch_related for comments
    reset_queries()
    start = time.time()
    posts = Post.objects.prefetch_related('comments').all()
    for post in posts:
        _ = list(post.comments.all())
    time3 = time.time() - start
    queries3 = len(connection.queries)

    print(f"No optimization: {time1:.3f}s, {queries1} queries")
    print(f"select_related: {time2:.3f}s, {queries2} queries")
    print(f"prefetch_related: {time3:.3f}s, {queries3} queries")

    # Output:
    # No optimization: 2.450s, 1001 queries
    # select_related: 0.050s, 1 query (49x faster)
    # prefetch_related: 0.100s, 2 queries (24x faster)
```

## ✅ Best Practices

### 1. When to Use select_related vs prefetch_related

```python
# Use select_related for:
# - ForeignKey (forward lookup)
# - OneToOneField
# - Relationships that result in a single JOIN

posts = Post.objects.select_related('author', 'category')

# Use prefetch_related for:
# - ManyToManyField
# - Reverse ForeignKey (e.g., author.posts)
# - GenericForeignKey
# - When you need custom filtering

authors = Author.objects.prefetch_related('posts', 'followers')
```

### 2. Avoid Over-fetching

```python
# Bad: Fetches all related data unnecessarily
posts = Post.objects.select_related(
    'author',
    'author__profile',
    'author__profile__country',
    'category',
    'category__parent'
).prefetch_related(
    'tags',
    'comments',
    'comments__author'
).all()

# Good: Only fetch what you need
if need_author_info:
    posts = posts.select_related('author')
if need_tags:
    posts = posts.prefetch_related('tags')
```

### 3. Reusing Prefetch Objects

```python
from django.db.models import Prefetch

# Define reusable prefetch
published_posts = Prefetch(
    'posts',
    queryset=Post.objects.filter(published=True).order_by('-created_at')
)

# Use in multiple queries
authors = Author.objects.prefetch_related(published_posts)
categories = Category.objects.prefetch_related(published_posts)
```

### 4. Debugging with django-debug-toolbar

```python
# Install: pip install django-debug-toolbar

# settings.py
INSTALLED_APPS = [
    ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    ...
]

INTERNAL_IPS = ['127.0.0.1']

# Now visit your page - toolbar shows queries, includes duplicate detection
```

### 5. Measure Query Count

```python
from django.test import TestCase
from django.test.utils import override_settings

class QueryCountTest(TestCase):
    @override_settings(DEBUG=True)
    def test_post_list_queries(self):
        """Ensure post list uses efficient queries"""
        from django.db import connection

        # Create test data
        author = Author.objects.create(username='testuser')
        for i in range(10):
            Post.objects.create(title=f'Post {i}', author=author)

        # Reset query log
        from django.db import reset_queries
        reset_queries()

        # Execute query
        posts = Post.objects.select_related('author').all()
        list(posts)  # Force evaluation

        # Check query count
        self.assertEqual(len(connection.queries), 1)
```

## ❌ Common Mistakes

### 1. Breaking Prefetch Chain

```python
# ❌ Bad: Breaks prefetch
authors = Author.objects.prefetch_related('posts').all()
for author in authors:
    # filter() creates new query, discards prefetch
    published_posts = author.posts.filter(published=True)

# ✅ Good: Use Prefetch with custom queryset
authors = Author.objects.prefetch_related(
    Prefetch(
        'posts',
        queryset=Post.objects.filter(published=True),
        to_attr='published_posts'
    )
).all()

for author in authors:
    # Uses prefetched data
    for post in author.published_posts:
        print(post.title)
```

### 2. Using select_related on Many-to-Many

```python
# ❌ Wrong: select_related doesn't work with M2M
posts = Post.objects.select_related('tags').all()  # No effect

# ✅ Correct: Use prefetch_related
posts = Post.objects.prefetch_related('tags').all()
```

### 3. Not Forcing Evaluation

```python
# ❌ Misleading: QuerySet not evaluated
posts = Post.objects.select_related('author')
# No queries yet!

# ✅ Force evaluation
posts = Post.objects.select_related('author').all()
list(posts)  # Forces query execution
```

## ❓ Interview Questions

### Q1: Explain the difference between select_related and prefetch_related

**Answer**:

- **select_related**: Uses SQL JOIN for ForeignKey/OneToOne. Single query. Best for "to-one" relationships.
- **prefetch_related**: Uses separate queries for ManyToMany/reverse FK. Multiple queries joined in Python. Best for "to-many" relationships.

### Q2: How does prefetch_related reduce queries from N+1 to 2?

**Answer**:
It collects all primary keys from the first query, then fetches all related objects in a second query using `WHERE id IN (...)`, joining results in Python memory.

### Q3: What's the Prefetch object used for?

**Answer**:
Customizes prefetch_related behavior:

- Apply filters to prefetched data
- Use `to_attr` to store results in custom attribute
- Combine with select_related for nested optimization

## 📚 Summary

**Key Takeaways**:

1. Use `select_related()` for ForeignKey/OneToOne (SQL JOIN)
2. Use `prefetch_related()` for ManyToMany/reverse FK (separate queries)
3. Solve N+1 problem: 101 queries → 1-3 queries
4. Use `Prefetch()` for custom filtering
5. Combine both for complex relationships
6. Nested lookups: `'author__profile'` or `'posts__comments'`
7. Use `to_attr` to store prefetched data
8. Don't break prefetch chain with filters
9. Measure with django-debug-toolbar
10. Test query counts in unit tests

Proper use of these methods is essential for performant Django applications!
