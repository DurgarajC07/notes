# Django Aggregation & Annotation

## 📖 Concept Explanation

Aggregation computes summary values (count, sum, average) across a QuerySet. Annotation adds computed fields to each object in a QuerySet.

### Aggregation vs Annotation

```
Aggregation: QuerySet → Single Value (e.g., total count)
Annotation:  QuerySet → QuerySet with Extra Fields (e.g., comment count per post)
```

**Key Difference**:

- **aggregate()**: Returns dictionary with summary values
- **annotate()**: Returns QuerySet with additional computed fields

### Basic Example

```python
from django.db.models import Count, Avg

# Aggregation - single value
total_posts = Post.objects.count()
avg_views = Post.objects.aggregate(Avg('views'))['views__avg']

# Annotation - per-object values
posts = Post.objects.annotate(comment_count=Count('comments'))
for post in posts:
    print(f"{post.title}: {post.comment_count} comments")
```

## 🧠 Aggregation Functions

### 1. Count

```python
from django.db.models import Count

# Total count
total = Post.objects.count()

# Aggregate count
result = Post.objects.aggregate(total=Count('id'))
# Returns: {'total': 100}

# Count distinct
author_count = Post.objects.aggregate(
    author_count=Count('author', distinct=True)
)

# Count related objects
comments_count = Comment.objects.aggregate(
    total_comments=Count('id')
)
```

### 2. Sum

```python
from django.db.models import Sum

# Total views across all posts
result = Post.objects.aggregate(total_views=Sum('views'))
# Returns: {'total_views': 150000}

# Sum with filter
published_views = Post.objects.filter(published=True).aggregate(
    total_views=Sum('views')
)

# Multiple sums
stats = Order.objects.aggregate(
    total_amount=Sum('amount'),
    total_items=Sum('quantity')
)
```

### 3. Average

```python
from django.db.models import Avg

# Average views
result = Post.objects.aggregate(avg_views=Avg('views'))
# Returns: {'avg_views': 1500.5}

# Average with decimals
avg_price = Product.objects.aggregate(
    avg_price=Avg('price')
)
```

### 4. Max & Min

```python
from django.db.models import Max, Min

# Maximum and minimum
result = Post.objects.aggregate(
    max_views=Max('views'),
    min_views=Min('views'),
    latest_date=Max('created_at'),
    earliest_date=Min('created_at')
)
# Returns: {'max_views': 10000, 'min_views': 0, ...}
```

### 5. StdDev & Variance

```python
from django.db.models import StdDev, Variance

# Standard deviation and variance
stats = Post.objects.aggregate(
    views_stddev=StdDev('views'),
    views_variance=Variance('views')
)
```

## 📊 Annotation

### 1. Basic Annotation

```python
from django.db.models import Count

# Add comment count to each post
posts = Post.objects.annotate(comment_count=Count('comments'))

for post in posts:
    print(f"{post.title}: {post.comment_count} comments")

# Filter by annotation
popular_posts = Post.objects.annotate(
    comment_count=Count('comments')
).filter(comment_count__gte=10)

# Order by annotation
posts = Post.objects.annotate(
    comment_count=Count('comments')
).order_by('-comment_count')
```

### 2. Multiple Annotations

```python
from django.db.models import Count, Sum, Avg

posts = Post.objects.annotate(
    comment_count=Count('comments'),
    like_count=Count('likes'),
    avg_rating=Avg('ratings__score'),
    total_views=Sum('views')
)

for post in posts:
    print(f"{post.title}:")
    print(f"  Comments: {post.comment_count}")
    print(f"  Likes: {post.like_count}")
    print(f"  Avg Rating: {post.avg_rating}")
```

### 3. Annotation with Distinct

```python
from django.db.models import Count

# Count distinct authors who commented
posts = Post.objects.annotate(
    commenter_count=Count('comments__author', distinct=True)
)

# Count distinct tags
posts = Post.objects.annotate(
    tag_count=Count('tags', distinct=True)
)
```

## 🎯 Advanced Aggregation

### 1. Conditional Aggregation (Case/When)

```python
from django.db.models import Count, Q, Case, When, IntegerField

# Count posts by status
stats = Post.objects.aggregate(
    published_count=Count('id', filter=Q(published=True)),
    draft_count=Count('id', filter=Q(published=False)),
    featured_count=Count('id', filter=Q(featured=True))
)

# Using Case/When
posts = Post.objects.annotate(
    status_category=Case(
        When(published=True, featured=True, then=Value('featured')),
        When(published=True, then=Value('published')),
        default=Value('draft'),
        output_field=CharField()
    )
)

# Count by category
stats = Post.objects.aggregate(
    high_views=Count('id', filter=Q(views__gte=1000)),
    medium_views=Count('id', filter=Q(views__gte=100, views__lt=1000)),
    low_views=Count('id', filter=Q(views__lt=100))
)
```

### 2. Aggregation with GroupBy

```python
from django.db.models import Count

# Group by author, count posts
author_stats = Post.objects.values('author').annotate(
    post_count=Count('id')
).order_by('-post_count')

# Returns: [{'author': 1, 'post_count': 15}, {'author': 2, 'post_count': 10}, ...]

# Group by multiple fields
stats = Post.objects.values('author', 'category').annotate(
    post_count=Count('id'),
    total_views=Sum('views')
)

# Group by date
from django.db.models.functions import TruncDate, TruncMonth

daily_posts = Post.objects.annotate(
    date=TruncDate('created_at')
).values('date').annotate(
    count=Count('id')
).order_by('date')

monthly_posts = Post.objects.annotate(
    month=TruncMonth('created_at')
).values('month').annotate(
    count=Count('id'),
    total_views=Sum('views')
)
```

### 3. Subquery Annotations

```python
from django.db.models import OuterRef, Subquery, Count

# Latest comment per post
from django.db.models import OuterRef, Subquery

latest_comment = Comment.objects.filter(
    post=OuterRef('pk')
).order_by('-created_at')

posts = Post.objects.annotate(
    latest_comment_text=Subquery(latest_comment.values('text')[:1]),
    latest_comment_date=Subquery(latest_comment.values('created_at')[:1])
)

# Count from subquery
posts = Post.objects.annotate(
    approved_comments=Count(
        'comments',
        filter=Q(comments__approved=True)
    ),
    pending_comments=Count(
        'comments',
        filter=Q(comments__approved=False)
    )
)
```

## 🔢 F Expressions

### 1. Field References

```python
from django.db.models import F

# Compare fields
high_engagement = Post.objects.filter(comments__gt=F('views') / 10)

# Arithmetic operations
posts = Post.objects.annotate(
    engagement_score=F('views') + F('likes') * 10 + F('comments') * 20
)

# Update with F expression
Post.objects.filter(published=True).update(views=F('views') + 1)

# Order by computed value
posts = Post.objects.order_by(F('views') / F('likes'))
```

### 2. F Expression with Annotations

```python
from django.db.models import F, ExpressionWrapper, DecimalField

# Calculate engagement rate
posts = Post.objects.annotate(
    comment_count=Count('comments'),
    engagement_rate=ExpressionWrapper(
        F('comment_count') * 100.0 / F('views'),
        output_field=DecimalField(max_digits=5, decimal_places=2)
    )
)

# Filter by computed value
popular = posts.filter(engagement_rate__gte=5.0)
```

## 🏗️ Real-World Examples

### 1. E-commerce Analytics

```python
from django.db.models import Sum, Count, Avg, F

# Order statistics
order_stats = Order.objects.aggregate(
    total_orders=Count('id'),
    total_revenue=Sum('total_amount'),
    avg_order_value=Avg('total_amount'),
    total_items_sold=Sum('items__quantity')
)

# Customer statistics
customer_stats = User.objects.annotate(
    order_count=Count('orders'),
    total_spent=Sum('orders__total_amount'),
    avg_order_value=Avg('orders__total_amount')
).filter(order_count__gt=0)

# Product performance
product_stats = Product.objects.annotate(
    times_ordered=Count('orderitem'),
    total_quantity_sold=Sum('orderitem__quantity'),
    total_revenue=Sum(F('orderitem__quantity') * F('orderitem__price'))
).order_by('-total_revenue')
```

### 2. Blog Analytics

```python
# Author statistics
author_stats = User.objects.annotate(
    post_count=Count('posts'),
    total_views=Sum('posts__views'),
    total_comments=Sum('posts__comments'),
    avg_views_per_post=Avg('posts__views')
).filter(post_count__gt=0).order_by('-total_views')

# Category statistics
category_stats = Category.objects.annotate(
    post_count=Count('posts'),
    published_count=Count('posts', filter=Q(posts__published=True)),
    total_views=Sum('posts__views')
)

# Tag popularity
tag_stats = Tag.objects.annotate(
    post_count=Count('posts'),
    total_views=Sum('posts__views')
).order_by('-post_count')
```

### 3. Social Network Metrics

```python
# User engagement metrics
user_metrics = User.objects.annotate(
    post_count=Count('posts'),
    follower_count=Count('followers'),
    following_count=Count('following'),
    like_count=Count('likes'),
    comment_count=Count('comments'),
    # Engagement score
    engagement_score=F('follower_count') * 10 + F('post_count') * 5 + F('like_count')
).order_by('-engagement_score')

# Post engagement
post_metrics = Post.objects.annotate(
    like_count=Count('likes'),
    comment_count=Count('comments'),
    share_count=Count('shares'),
    engagement_rate=ExpressionWrapper(
        (F('like_count') + F('comment_count') * 2 + F('share_count') * 3) * 100.0 / F('views'),
        output_field=DecimalField(max_digits=5, decimal_places=2)
    )
)
```

## ✅ Best Practices

### 1. Optimize Aggregation Queries

```python
# BAD: Multiple queries
total_posts = Post.objects.count()
total_views = Post.objects.aggregate(Sum('views'))['views__sum']
total_comments = Comment.objects.count()

# GOOD: Single aggregation
stats = Post.objects.aggregate(
    total_posts=Count('id'),
    total_views=Sum('views'),
    total_comments=Count('comments')
)
```

### 2. Use select_related with Annotations

```python
# Optimize related queries
posts = Post.objects.select_related('author', 'category') \
                    .prefetch_related('tags') \
                    .annotate(
                        comment_count=Count('comments'),
                        like_count=Count('likes')
                    )
```

### 3. Cache Expensive Aggregations

```python
from django.core.cache import cache

def get_site_stats():
    """Get cached site statistics"""
    cache_key = 'site_stats'
    stats = cache.get(cache_key)

    if stats is None:
        stats = {
            'total_posts': Post.objects.count(),
            'total_users': User.objects.count(),
            'total_comments': Comment.objects.count(),
            'total_views': Post.objects.aggregate(Sum('views'))['views__sum'],
        }
        cache.set(cache_key, stats, 3600)  # Cache for 1 hour

    return stats
```

## 🔐 Security Best Practices

### 1. Validate Aggregation Inputs

```python
def get_stats_by_date(start_date, end_date):
    """Get statistics with date validation"""
    from datetime import datetime, timedelta

    # Validate dates
    if not isinstance(start_date, datetime):
        raise ValueError("Invalid start_date")

    if not isinstance(end_date, datetime):
        raise ValueError("Invalid end_date")

    # Limit date range
    max_range = timedelta(days=365)
    if (end_date - start_date) > max_range:
        raise ValueError("Date range too large")

    return Post.objects.filter(
        created_at__range=(start_date, end_date)
    ).aggregate(
        total_posts=Count('id'),
        total_views=Sum('views')
    )
```

## ⚡ Performance Optimization

### 1. Use Database Aggregation

```python
# BAD: Python aggregation (loads all objects)
posts = Post.objects.all()
total_views = sum(post.views for post in posts)

# GOOD: Database aggregation
total_views = Post.objects.aggregate(Sum('views'))['views__sum']
```

### 2. Index Fields Used in Aggregation

```python
class Post(models.Model):
    views = models.IntegerField(default=0, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)

    class Meta:
        indexes = [
            models.Index(fields=['created_at', 'views']),
        ]
```

## ❓ Interview Questions

### Q1: What's the difference between aggregate() and annotate()?

**Answer**:

- **aggregate()**: Returns dict with summary values (single result)
- **annotate()**: Returns QuerySet with computed fields per object

### Q2: How do you count distinct values?

**Answer**:

```python
Post.objects.aggregate(
    author_count=Count('author', distinct=True)
)
```

### Q3: What is an F expression?

**Answer**:
F expression references model field values in queries:

```python
Post.objects.filter(likes__gt=F('views') * 0.1)
```

## 📚 Summary

**Key Takeaways**:

1. aggregate() returns summary dict
2. annotate() adds computed fields
3. Common functions: Count, Sum, Avg, Max, Min
4. Use filter with Q for conditional aggregation
5. F expressions reference field values
6. values() + annotate() = GROUP BY
7. Subquery for complex annotations
8. Always use DB aggregation, not Python
9. Cache expensive aggregations
10. Index fields used in aggregation

Django aggregation is powerful for analytics!
