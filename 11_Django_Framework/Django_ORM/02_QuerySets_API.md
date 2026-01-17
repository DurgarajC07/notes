# Django QuerySet API

## 📖 Concept Explanation

QuerySets are Django's way of retrieving objects from the database. They are lazy, chainable, and represent database queries that haven't been executed yet.

### QuerySet Execution Flow

```
Model.objects.filter() → QuerySet (lazy) → Iteration/Evaluation → SQL Query → Results
```

**Lazy Evaluation**:

- QuerySets don't hit the database until evaluated
- Allows chaining multiple filters
- Executes single optimized SQL query

### Basic QuerySet

```python
from blog.models import Post

# Create QuerySet (no database hit yet)
posts = Post.objects.filter(published=True)

# Still no database hit
posts = posts.filter(created_at__gte='2024-01-01')

# NOW database is hit (during iteration)
for post in posts:
    print(post.title)
```

## 🧠 Retrieving Objects

### 1. Get All Objects

```python
# Get all posts
all_posts = Post.objects.all()

# Iterate (executes query)
for post in all_posts:
    print(post.title)

# Count (executes COUNT query)
count = Post.objects.count()

# Check existence (efficient)
exists = Post.objects.filter(published=True).exists()
```

### 2. Filtering

```python
# Filter by field value
published = Post.objects.filter(published=True)

# Multiple conditions (AND)
recent = Post.objects.filter(published=True, created_at__gte='2024-01-01')

# Exclude
drafts = Post.objects.exclude(published=True)

# Chaining filters
posts = Post.objects.filter(published=True) \
                    .filter(category__name='Technology') \
                    .exclude(views__lt=100)
```

### 3. Get Single Object

```python
# Get single object (raises DoesNotExist if not found, MultipleObjectsReturned if multiple)
post = Post.objects.get(pk=1)
post = Post.objects.get(slug='my-post')

# Get or create
post, created = Post.objects.get_or_create(
    slug='my-post',
    defaults={'title': 'My Post', 'content': 'Content here'}
)

# Update or create
post, created = Post.objects.update_or_create(
    slug='my-post',
    defaults={'title': 'Updated Title', 'views': 100}
)

# Get first/last
first = Post.objects.filter(published=True).first()  # Returns None if empty
last = Post.objects.filter(published=True).last()
```

## 🔍 Field Lookups

### 1. Exact Matches

```python
# Exact match (case-sensitive)
Post.objects.filter(title='My Title')
Post.objects.filter(title__exact='My Title')  # Same as above

# Case-insensitive
Post.objects.filter(title__iexact='my title')

# In list
Post.objects.filter(id__in=[1, 2, 3, 4, 5])
Post.objects.filter(category__name__in=['Tech', 'Science'])
```

### 2. Comparisons

```python
# Greater than
Post.objects.filter(views__gt=1000)

# Greater than or equal
Post.objects.filter(views__gte=1000)

# Less than
Post.objects.filter(views__lt=100)

# Less than or equal
Post.objects.filter(views__lte=100)

# Between (range)
from datetime import datetime, timedelta
last_week = datetime.now() - timedelta(days=7)
Post.objects.filter(created_at__range=(last_week, datetime.now()))
```

### 3. String Matching

```python
# Contains (case-sensitive)
Post.objects.filter(title__contains='Django')

# Case-insensitive contains
Post.objects.filter(title__icontains='django')

# Starts with
Post.objects.filter(title__startswith='How to')
Post.objects.filter(title__istartswith='how to')

# Ends with
Post.objects.filter(title__endswith='Tutorial')
Post.objects.filter(title__iendswith='tutorial')

# Regex (case-sensitive)
Post.objects.filter(title__regex=r'^(An?|The) ')

# Case-insensitive regex
Post.objects.filter(title__iregex=r'^(an?|the) ')
```

### 4. Date Lookups

```python
from datetime import date

# Year
Post.objects.filter(created_at__year=2024)

# Month
Post.objects.filter(created_at__month=1)

# Day
Post.objects.filter(created_at__day=15)

# Week day (1=Sunday, 7=Saturday)
Post.objects.filter(created_at__week_day=2)

# Date (ignore time)
Post.objects.filter(created_at__date=date.today())

# Date comparisons
Post.objects.filter(created_at__date__gte=date(2024, 1, 1))
```

### 5. Null/Boolean Checks

```python
# Is null
Post.objects.filter(published_date__isnull=True)

# Is not null
Post.objects.filter(published_date__isnull=False)

# Boolean
Post.objects.filter(published=True)
Post.objects.filter(featured=False)
```

## 🏗️ Complex Queries with Q Objects

### 1. OR Queries

```python
from django.db.models import Q

# OR condition
Post.objects.filter(Q(published=True) | Q(featured=True))

# Complex OR with AND
Post.objects.filter(
    Q(published=True) & (Q(category__name='Tech') | Q(category__name='Science'))
)

# NOT
Post.objects.filter(~Q(status='draft'))

# Multiple OR
Post.objects.filter(
    Q(title__icontains='django') |
    Q(title__icontains='python') |
    Q(title__icontains='tutorial')
)
```

### 2. Dynamic Filters

```python
def search_posts(keyword=None, category=None, published=None):
    """Build dynamic query"""
    query = Q()

    if keyword:
        query &= Q(title__icontains=keyword) | Q(content__icontains=keyword)

    if category:
        query &= Q(category=category)

    if published is not None:
        query &= Q(published=published)

    return Post.objects.filter(query)

# Usage
posts = search_posts(keyword='django', category=tech_category, published=True)
```

## 🎯 Ordering & Limiting

### 1. Ordering

```python
# Order by field (ascending)
Post.objects.order_by('created_at')

# Descending
Post.objects.order_by('-created_at')

# Multiple fields
Post.objects.order_by('-created_at', 'title')

# Random order
Post.objects.order_by('?')

# Reverse order
posts = Post.objects.order_by('-created_at')
reversed_posts = posts.reverse()

# Clear ordering (use model's Meta ordering)
Post.objects.order_by('title').order_by()
```

### 2. Limiting & Slicing

```python
# First 5 posts
Post.objects.all()[:5]

# Skip first 5, get next 5 (pagination)
Post.objects.all()[5:10]

# Get 10th post
Post.objects.all()[9:10]

# Negative indexing (NOT SUPPORTED)
# Post.objects.all()[-1]  # Error!

# Latest/Earliest
latest = Post.objects.latest('created_at')
earliest = Post.objects.earliest('created_at')
```

## 📊 Aggregation & Annotation

### 1. Aggregation

```python
from django.db.models import Count, Sum, Avg, Max, Min

# Count all posts
total = Post.objects.count()

# Aggregate functions
stats = Post.objects.aggregate(
    total_posts=Count('id'),
    total_views=Sum('views'),
    avg_views=Avg('views'),
    max_views=Max('views'),
    min_views=Min('views')
)
# Returns: {'total_posts': 100, 'total_views': 50000, ...}
```

### 2. Annotation

```python
from django.db.models import Count, F

# Add computed field to each object
posts = Post.objects.annotate(comment_count=Count('comments'))

for post in posts:
    print(f"{post.title}: {post.comment_count} comments")

# Filter by annotation
popular = Post.objects.annotate(
    comment_count=Count('comments')
).filter(comment_count__gte=10)

# Multiple annotations
posts = Post.objects.annotate(
    comment_count=Count('comments'),
    like_count=Count('likes'),
    engagement=F('views') + F('comment_count') * 10
)
```

## 🔗 Related Object Queries

### 1. Following Relationships

```python
# Forward FK: post.author
posts = Post.objects.filter(author__username='john')

# Reverse FK: author.posts (related_name)
from django.contrib.auth.models import User
users = User.objects.filter(posts__published=True)

# Through multiple relationships
Post.objects.filter(author__profile__country='US')

# Many-to-many
Post.objects.filter(tags__name='django')
```

### 2. Select Related (JOIN)

```python
# Without select_related (N+1 queries)
posts = Post.objects.all()
for post in posts:
    print(post.author.username)  # Separate query for each author

# With select_related (1 query with JOIN)
posts = Post.objects.select_related('author', 'category')
for post in posts:
    print(post.author.username)  # No extra queries

# Following FK chain
posts = Post.objects.select_related('author__profile')
```

### 3. Prefetch Related (Separate Queries)

```python
# Without prefetch_related (N+1 queries)
posts = Post.objects.all()
for post in posts:
    for tag in post.tags.all():  # Separate query for each post
        print(tag.name)

# With prefetch_related (2 queries total)
posts = Post.objects.prefetch_related('tags', 'comments')
for post in posts:
    for tag in post.tags.all():  # No extra queries
        print(tag.name)

# Custom prefetch
from django.db.models import Prefetch

posts = Post.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.filter(approved=True).select_related('author')
    )
)
```

## ✅ Best Practices

### 1. Efficient Queries

```python
# BAD: N+1 queries
posts = Post.objects.all()
for post in posts:
    print(post.author.username)  # Extra query per post
    for tag in post.tags.all():  # Extra query per post
        print(tag.name)

# GOOD: Optimized
posts = Post.objects.select_related('author').prefetch_related('tags')
for post in posts:
    print(post.author.username)  # No extra query
    for tag in post.tags.all():  # No extra query
        print(tag.name)
```

### 2. Use only() and defer()

```python
# Only fetch specific fields
posts = Post.objects.only('id', 'title', 'created_at')

# Fetch all except specific fields
posts = Post.objects.defer('content', 'metadata')

# Combined with select_related
posts = Post.objects.select_related('author').only(
    'id', 'title', 'author__username'
)
```

### 3. Iterator for Large QuerySets

```python
# BAD: Loads all into memory
for post in Post.objects.all():
    process(post)

# GOOD: Iterator (chunks)
for post in Post.objects.iterator(chunk_size=1000):
    process(post)
```

## 🔐 Security Best Practices

### 1. Avoid SQL Injection

```python
# BAD: SQL injection risk
Post.objects.raw(f"SELECT * FROM post WHERE title = '{user_input}'")

# GOOD: Parameterized query
Post.objects.raw("SELECT * FROM post WHERE title = %s", [user_input])

# BETTER: Use ORM
Post.objects.filter(title=user_input)
```

### 2. Validate User Input

```python
def search_posts(request):
    # Validate input
    query = request.GET.get('q', '').strip()

    if not query or len(query) < 3:
        return Post.objects.none()  # Empty QuerySet

    # Limit results
    return Post.objects.filter(title__icontains=query)[:100]
```

## ⚡ Performance Optimization

### 1. Query Inspection

```python
# Print SQL query
queryset = Post.objects.filter(published=True)
print(queryset.query)

# Explain query plan
print(queryset.explain())

# Count queries
from django.db import connection
from django.db import reset_queries

reset_queries()
# Your code here
print(len(connection.queries))
print(connection.queries)
```

### 2. Bulk Operations

```python
# Bulk create (single INSERT)
posts = [
    Post(title=f'Post {i}', content='Content')
    for i in range(1000)
]
Post.objects.bulk_create(posts, batch_size=500)

# Bulk update
posts = Post.objects.filter(published=False)
posts.update(published=True)  # Single UPDATE query

# Bulk update with different values (Django 4.2+)
Post.objects.bulk_update(posts, ['title', 'views'], batch_size=500)
```

### 3. Database Functions

```python
from django.db.models.functions import Length, Lower, Upper, Concat
from django.db.models import Value

# String functions
posts = Post.objects.annotate(
    title_length=Length('title'),
    title_lower=Lower('title'),
    full_title=Concat('title', Value(' - '), 'category__name')
)

# Filter by computed value
long_titles = Post.objects.annotate(
    title_length=Length('title')
).filter(title_length__gt=50)
```

## ❓ Interview Questions

### Q1: What is the difference between filter() and get()?

**Answer**:

- **filter()**: Returns QuerySet (can be empty, can have multiple results)
- **get()**: Returns single object (raises DoesNotExist or MultipleObjectsReturned)

Use `get()` for unique lookups (pk, slug), `filter()` otherwise.

### Q2: When should you use select_related vs prefetch_related?

**Answer**:

- **select_related**: ForeignKey, OneToOne (SQL JOIN)
- **prefetch_related**: ManyToMany, reverse FK (separate queries)

### Q3: What is lazy evaluation in QuerySets?

**Answer**:
QuerySets don't execute SQL until needed (iteration, len(), list(), etc.). Allows chaining filters efficiently.

## 📚 Summary

**Key Takeaways**:

1. QuerySets are lazy - no DB hit until evaluation
2. Use filter() for multiple results, get() for single
3. Field lookups: **exact, **icontains, **gte, **date, etc.
4. Q objects for complex OR/NOT queries
5. select_related for FK/OneToOne (JOIN)
6. prefetch_related for M2M/reverse FK
7. Use only()/defer() to limit fields
8. Bulk operations for better performance
9. Avoid N+1 queries
10. Always use ORM (not raw SQL)

Django QuerySet API is powerful and efficient!
