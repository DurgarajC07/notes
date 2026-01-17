# Django Custom Managers

## 📖 Concept Explanation

Managers are interfaces through which database query operations are provided to Django models. Custom managers allow you to add custom query methods or modify the default queryset.

### Manager Basics

```
Model.objects → Default Manager
Model.custom_manager → Custom Manager
```

**Default Manager**:

- Every model has at least one manager
- `objects` is the default manager name
- Returns all instances by default

### Basic Custom Manager

```python
from django.db import models

class PublishedManager(models.Manager):
    """Manager that returns only published posts"""
    def get_queryset(self):
        return super().get_queryset().filter(published=True)

class Post(models.Model):
    title = models.CharField(max_length=200)
    published = models.BooleanField(default=False)

    objects = models.Manager()  # Default manager
    published_objects = PublishedManager()  # Custom manager

# Usage
all_posts = Post.objects.all()  # All posts
published_posts = Post.published_objects.all()  # Only published
```

## 🧠 Creating Custom Managers

### 1. Modifying Default QuerySet

```python
class PublishedManager(models.Manager):
    """Return only published posts by default"""
    def get_queryset(self):
        return super().get_queryset().filter(published=True)

class Post(models.Model):
    title = models.CharField(max_length=200)
    published = models.BooleanField(default=False)
    featured = models.BooleanField(default=False)

    published_objects = PublishedManager()

    # Still have access to original manager
    objects = models.Manager()

# Usage
Post.published_objects.all()  # Only published posts
Post.published_objects.filter(featured=True)  # Published + featured
```

### 2. Adding Custom Methods

```python
class PostManager(models.Manager):
    """Manager with custom query methods"""

    def published(self):
        """Get published posts"""
        return self.filter(published=True)

    def featured(self):
        """Get featured posts"""
        return self.filter(featured=True)

    def by_author(self, author):
        """Get posts by author"""
        return self.filter(author=author)

    def recent(self, days=7):
        """Get recent posts"""
        from datetime import datetime, timedelta
        cutoff = datetime.now() - timedelta(days=days)
        return self.filter(created_at__gte=cutoff)

    def popular(self, min_views=1000):
        """Get popular posts"""
        return self.filter(views__gte=min_views)

class Post(models.Model):
    title = models.CharField(max_length=200)
    published = models.BooleanField(default=False)
    featured = models.BooleanField(default=False)
    views = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)

    objects = PostManager()

# Usage - chainable
Post.objects.published().featured().recent(days=30).order_by('-views')
Post.objects.by_author(user).recent(days=7)
Post.objects.popular(min_views=5000).published()
```

### 3. Custom QuerySet + Manager

```python
from django.db import models

class PostQuerySet(models.QuerySet):
    """Custom QuerySet with chainable methods"""

    def published(self):
        return self.filter(published=True)

    def featured(self):
        return self.filter(featured=True)

    def by_category(self, category):
        return self.filter(category=category)

    def search(self, query):
        return self.filter(
            models.Q(title__icontains=query) |
            models.Q(content__icontains=query)
        )

    def with_author(self):
        """Optimize with select_related"""
        return self.select_related('author', 'category')

    def with_stats(self):
        """Annotate with computed fields"""
        return self.annotate(
            comment_count=models.Count('comments'),
            like_count=models.Count('likes')
        )

class PostManager(models.Manager):
    """Manager using custom QuerySet"""
    def get_queryset(self):
        return PostQuerySet(self.model, using=self._db)

    # Proxy methods to QuerySet
    def published(self):
        return self.get_queryset().published()

    def featured(self):
        return self.get_queryset().featured()

    def by_category(self, category):
        return self.get_queryset().by_category(category)

    def search(self, query):
        return self.get_queryset().search(query)

# Or use from_queryset (Django 1.7+)
class PostManager(models.Manager.from_queryset(PostQuerySet)):
    pass

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    published = models.BooleanField(default=False)
    featured = models.BooleanField(default=False)
    category = models.ForeignKey('Category', on_delete=models.CASCADE)

    objects = PostManager()

# Usage - all methods are chainable
posts = Post.objects.published() \
                    .featured() \
                    .by_category(tech_category) \
                    .search('django') \
                    .with_author() \
                    .with_stats() \
                    .order_by('-created_at')[:10]
```

## 🎯 Advanced Manager Patterns

### 1. Soft Delete Manager

```python
class SoftDeleteQuerySet(models.QuerySet):
    """QuerySet that excludes deleted objects"""

    def delete(self):
        """Soft delete - mark as deleted"""
        return self.update(deleted_at=timezone.now())

    def hard_delete(self):
        """Actually delete from database"""
        return super().delete()

    def alive(self):
        """Get non-deleted objects"""
        return self.filter(deleted_at__isnull=True)

    def deleted(self):
        """Get deleted objects"""
        return self.filter(deleted_at__isnull=False)

class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return SoftDeleteQuerySet(self.model, using=self._db).alive()

    def all_with_deleted(self):
        return SoftDeleteQuerySet(self.model, using=self._db)

    def deleted_only(self):
        return SoftDeleteQuerySet(self.model, using=self._db).deleted()

class Post(models.Model):
    title = models.CharField(max_length=200)
    deleted_at = models.DateTimeField(null=True, blank=True)

    objects = SoftDeleteManager()

# Usage
Post.objects.all()  # Only alive posts
Post.objects.all_with_deleted()  # Include deleted
Post.objects.deleted_only()  # Only deleted
post.delete()  # Soft delete
```

### 2. Polymorphic Manager

```python
class PersonQuerySet(models.QuerySet):
    """QuerySet for polymorphic queries"""

    def employees(self):
        """Get only employees"""
        return self.filter(employee__isnull=False).select_related('employee')

    def customers(self):
        """Get only customers"""
        return self.filter(customer__isnull=False).select_related('customer')

class PersonManager(models.Manager.from_queryset(PersonQuerySet)):
    pass

class Person(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()

    objects = PersonManager()

class Employee(models.Model):
    person = models.OneToOneField(Person, on_delete=models.CASCADE)
    employee_id = models.CharField(max_length=20)
    department = models.CharField(max_length=100)

class Customer(models.Model):
    person = models.OneToOneField(Person, on_delete=models.CASCADE)
    customer_id = models.CharField(max_length=20)
    loyalty_points = models.IntegerField(default=0)

# Usage
Person.objects.employees()  # Get all employees
Person.objects.customers()  # Get all customers
```

### 3. Cached Manager

```python
from django.core.cache import cache

class CachedManager(models.Manager):
    """Manager with built-in caching"""

    def get_cached(self, cache_key, timeout=3600):
        """Get from cache or database"""
        data = cache.get(cache_key)

        if data is None:
            data = list(self.get_queryset())
            cache.set(cache_key, data, timeout)

        return data

    def invalidate_cache(self, cache_key):
        """Invalidate cache"""
        cache.delete(cache_key)

class Post(models.Model):
    title = models.CharField(max_length=200)

    objects = CachedManager()

# Usage
posts = Post.objects.get_cached('all_posts')
Post.objects.invalidate_cache('all_posts')
```

### 4. Tenant-Aware Manager

```python
class TenantManager(models.Manager):
    """Manager that filters by tenant"""

    def __init__(self, *args, **kwargs):
        self.tenant_id = kwargs.pop('tenant_id', None)
        super().__init__(*args, **kwargs)

    def get_queryset(self):
        queryset = super().get_queryset()
        if self.tenant_id:
            queryset = queryset.filter(tenant_id=self.tenant_id)
        return queryset

class Post(models.Model):
    title = models.CharField(max_length=200)
    tenant = models.ForeignKey('Tenant', on_delete=models.CASCADE)

    objects = models.Manager()  # All posts
    tenant_objects = TenantManager()  # Tenant-specific

# Usage (set tenant in middleware)
current_tenant = get_current_tenant(request)
posts = Post.tenant_objects.filter(tenant_id=current_tenant.id)
```

## 🏗️ Real-World Examples

### 1. E-commerce Order Manager

```python
class OrderQuerySet(models.QuerySet):
    def pending(self):
        return self.filter(status='pending')

    def confirmed(self):
        return self.filter(status='confirmed')

    def shipped(self):
        return self.filter(status='shipped')

    def delivered(self):
        return self.filter(status='delivered')

    def cancelled(self):
        return self.filter(status='cancelled')

    def by_customer(self, customer):
        return self.filter(customer=customer)

    def recent(self, days=30):
        from datetime import datetime, timedelta
        cutoff = datetime.now() - timedelta(days=days)
        return self.filter(created_at__gte=cutoff)

    def with_items(self):
        return self.prefetch_related('items__product')

    def with_totals(self):
        from django.db.models import Sum, Count
        return self.annotate(
            total_amount=Sum('items__price'),
            item_count=Count('items')
        )

class OrderManager(models.Manager.from_queryset(OrderQuerySet)):
    pass

class Order(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('confirmed', 'Confirmed'),
        ('shipped', 'Shipped'),
        ('delivered', 'Delivered'),
        ('cancelled', 'Cancelled'),
    ]

    customer = models.ForeignKey('Customer', on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES)
    created_at = models.DateTimeField(auto_now_add=True)

    objects = OrderManager()

# Usage
orders = Order.objects.confirmed() \
                      .by_customer(customer) \
                      .recent(days=7) \
                      .with_items() \
                      .with_totals()
```

### 2. Blog Post Manager

```python
class PostQuerySet(models.QuerySet):
    def published(self):
        return self.filter(published=True, published_date__lte=timezone.now())

    def drafts(self):
        return self.filter(published=False)

    def scheduled(self):
        return self.filter(published=True, published_date__gt=timezone.now())

    def by_tag(self, tag):
        return self.filter(tags=tag)

    def popular(self):
        return self.filter(views__gte=1000).order_by('-views')

    def trending(self, days=7):
        from datetime import datetime, timedelta
        cutoff = datetime.now() - timedelta(days=days)
        return self.filter(created_at__gte=cutoff).order_by('-views')

    def optimized(self):
        return self.select_related('author', 'category') \
                   .prefetch_related('tags') \
                   .annotate(
                       comment_count=models.Count('comments')
                   )

class PostManager(models.Manager.from_queryset(PostQuerySet)):
    def get_for_user(self, user):
        """Get posts visible to user"""
        if user.is_staff:
            return self.get_queryset()
        return self.published()

class Post(models.Model):
    title = models.CharField(max_length=200)
    published = models.BooleanField(default=False)
    published_date = models.DateTimeField(null=True, blank=True)
    views = models.IntegerField(default=0)

    objects = PostManager()
```

## ✅ Best Practices

### 1. Use QuerySet Methods for Reusability

```python
# GOOD: Chainable QuerySet methods
class PostQuerySet(models.QuerySet):
    def published(self):
        return self.filter(published=True)

    def featured(self):
        return self.filter(featured=True)

# Usage
Post.objects.published().featured()  # Works
Post.objects.featured().published()  # Also works
```

### 2. Keep Managers Focused

```python
# GOOD: Separate concerns
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(published=True)

class FeaturedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(featured=True)

class Post(models.Model):
    objects = models.Manager()
    published_objects = PublishedManager()
    featured_objects = FeaturedManager()
```

### 3. Document Custom Managers

```python
class PostManager(models.Manager):
    """
    Custom manager for Post model.

    Methods:
        published(): Get published posts
        featured(): Get featured posts
        recent(days=7): Get recent posts
    """

    def published(self):
        """Get all published posts."""
        return self.filter(published=True)
```

## ❓ Interview Questions

### Q1: What is a Django Manager?

**Answer**:
Interface through which database query operations are provided to models. Default is `Model.objects`.

### Q2: How do you create a custom manager?

**Answer**:
Subclass `models.Manager` and override `get_queryset()`:

```python
class CustomManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(active=True)
```

### Q3: What's the difference between Manager and QuerySet?

**Answer**:

- **Manager**: Model-level interface (Model.objects)
- **QuerySet**: Chainable query API (Model.objects.filter())

Use `from_queryset()` to combine both.

## 📚 Summary

**Key Takeaways**:

1. Managers provide query interface to models
2. Override get_queryset() to change default behavior
3. Add custom methods for reusable queries
4. Use QuerySet for chainable methods
5. from_queryset() combines Manager + QuerySet
6. Soft delete with custom managers
7. Tenant-aware managers for multi-tenancy
8. Cache expensive queries in managers
9. Keep managers focused and documented
10. Multiple managers per model allowed

Django custom managers promote DRY principle!
