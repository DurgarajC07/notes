# Django Models & Fields

## 📖 Concept Explanation

Django Models define the structure of your database tables. Each model is a Python class that subclasses `django.db.models.Model`. Each attribute represents a database field.

### Model → Database Mapping

```
Python Model → Django ORM → SQL → Database Table
```

**Process**:

1. Define model class with fields
2. Run `makemigrations` to create migration files
3. Run `migrate` to apply to database
4. Django creates table with columns

### Basic Model

```python
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title
```

## 🧠 Common Field Types

### 1. Text Fields

```python
from django.db import models

class Article(models.Model):
    # CharField - Short text (requires max_length)
    title = models.CharField(max_length=200)

    # TextField - Long text (unlimited length)
    content = models.TextField()

    # SlugField - URL-friendly string
    slug = models.SlugField(max_length=200, unique=True)

    # EmailField - Email validation
    author_email = models.EmailField()

    # URLField - URL validation
    source_url = models.URLField(max_length=500, blank=True)

    # TextField with choices
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('published', 'Published'),
        ('archived', 'Archived'),
    ]
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='draft')
```

### 2. Numeric Fields

```python
class Product(models.Model):
    # IntegerField - Integers
    quantity = models.IntegerField(default=0)

    # PositiveIntegerField - Only positive integers
    stock = models.PositiveIntegerField(default=0)

    # SmallIntegerField - Smaller range (-32768 to 32767)
    rating = models.SmallIntegerField(default=0)

    # DecimalField - Precise decimal numbers (for money)
    price = models.DecimalField(max_digits=10, decimal_places=2)

    # FloatField - Floating point (less precise, avoid for money)
    weight = models.FloatField()

    # BigIntegerField - Very large integers
    views = models.BigIntegerField(default=0)
```

### 3. Date & Time Fields

```python
class Event(models.Model):
    # DateField - Date only
    event_date = models.DateField()

    # TimeField - Time only
    start_time = models.TimeField()

    # DateTimeField - Date and time
    created_at = models.DateTimeField(auto_now_add=True)  # Set once on creation
    updated_at = models.DateTimeField(auto_now=True)      # Update on every save

    # DurationField - Time duration
    duration = models.DurationField()  # e.g., timedelta(hours=2)
```

### 4. Boolean & Binary Fields

```python
class User(models.Model):
    # BooleanField - True/False
    is_active = models.BooleanField(default=True)
    is_verified = models.BooleanField(default=False)

    # NullBooleanField (deprecated, use BooleanField with null=True)
    newsletter_subscription = models.BooleanField(null=True, blank=True)

    # BinaryField - Binary data
    signature = models.BinaryField()
```

### 5. File Fields

```python
class Document(models.Model):
    # FileField - Any file
    document = models.FileField(upload_to='documents/%Y/%m/%d/')

    # ImageField - Images (requires Pillow)
    thumbnail = models.ImageField(
        upload_to='images/',
        height_field='thumbnail_height',
        width_field='thumbnail_width',
        blank=True
    )
    thumbnail_height = models.IntegerField(null=True, blank=True)
    thumbnail_width = models.IntegerField(null=True, blank=True)

    # Custom upload path
    def user_directory_path(instance, filename):
        return f'user_{instance.user.id}/{filename}'

    avatar = models.ImageField(upload_to=user_directory_path)
```

### 6. Special Fields

```python
from django.contrib.postgres.fields import ArrayField, JSONField
import uuid

class AdvancedModel(models.Model):
    # UUIDField - Universally unique identifier
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    # JSONField - JSON data (Postgres, MySQL 5.7+, SQLite 3.9+)
    metadata = models.JSONField(default=dict, blank=True)

    # ArrayField - Array of values (PostgreSQL only)
    tags = ArrayField(models.CharField(max_length=50), default=list, blank=True)

    # IPAddressField
    ip_address = models.GenericIPAddressField()
```

## 🏗️ Field Options

### 1. Common Options

```python
class Post(models.Model):
    # null - Database allows NULL
    middle_name = models.CharField(max_length=100, null=True)

    # blank - Form validation allows empty
    bio = models.TextField(blank=True)

    # default - Default value
    status = models.CharField(max_length=20, default='draft')

    # unique - Must be unique
    slug = models.SlugField(unique=True)

    # db_index - Create database index
    email = models.EmailField(db_index=True)

    # editable - Show in forms/admin
    created_at = models.DateTimeField(auto_now_add=True, editable=False)

    # help_text - Help text in forms/admin
    title = models.CharField(max_length=200, help_text="Enter a catchy title")

    # verbose_name - Human-readable name
    created_at = models.DateTimeField(auto_now_add=True, verbose_name="Creation Date")

    # validators - Custom validation
    from django.core.validators import MinValueValidator, MaxValueValidator
    rating = models.IntegerField(
        validators=[MinValueValidator(1), MaxValueValidator(5)]
    )
```

### 2. Choices

```python
class Order(models.Model):
    # Old-style choices (Django < 3.0)
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('processing', 'Processing'),
        ('shipped', 'Shipped'),
        ('delivered', 'Delivered'),
        ('cancelled', 'Cancelled'),
    ]
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')

    # New-style choices (Django 3.0+) - Recommended
    class Status(models.TextChoices):
        PENDING = 'pending', 'Pending'
        PROCESSING = 'processing', 'Processing'
        SHIPPED = 'shipped', 'Shipped'
        DELIVERED = 'delivered', 'Delivered'
        CANCELLED = 'cancelled', 'Cancelled'

    status_new = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDING
    )

    # Integer choices
    class Priority(models.IntegerChoices):
        LOW = 1, 'Low'
        MEDIUM = 2, 'Medium'
        HIGH = 3, 'High'
        URGENT = 4, 'Urgent'

    priority = models.IntegerField(
        choices=Priority.choices,
        default=Priority.MEDIUM
    )
```

## 🎯 Model Meta Options

```python
class Post(models.Model):
    title = models.CharField(max_length=200)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        # Table name
        db_table = 'blog_posts'

        # Ordering (default queryset order)
        ordering = ['-created_at', 'title']

        # Verbose names
        verbose_name = 'Blog Post'
        verbose_name_plural = 'Blog Posts'

        # Unique constraints
        unique_together = [['title', 'author']]  # Deprecated, use constraints

        # Indexes
        indexes = [
            models.Index(fields=['title']),
            models.Index(fields=['created_at', 'published']),
        ]

        # Constraints (Django 2.2+)
        constraints = [
            models.UniqueConstraint(
                fields=['title', 'author'],
                name='unique_title_author'
            ),
            models.CheckConstraint(
                check=models.Q(rating__gte=0) & models.Q(rating__lte=5),
                name='valid_rating'
            ),
        ]

        # Permissions
        permissions = [
            ('can_publish', 'Can publish posts'),
            ('can_feature', 'Can feature posts'),
        ]

        # Abstract model
        abstract = False

        # Proxy model
        proxy = False

        # Managed by Django
        managed = True
```

## 🔗 Relationships

### 1. ForeignKey (Many-to-One)

```python
class Post(models.Model):
    # Many posts → One author
    author = models.ForeignKey(
        'auth.User',
        on_delete=models.CASCADE,  # Delete posts when user is deleted
        related_name='posts',       # Reverse relation: user.posts.all()
        related_query_name='post',  # Filter: User.objects.filter(post__title='...')
    )

    # on_delete options:
    # CASCADE - Delete related objects
    # PROTECT - Prevent deletion (raise ProtectedError)
    # SET_NULL - Set to NULL (requires null=True)
    # SET_DEFAULT - Set to default value (requires default)
    # SET(...) - Set to specific value
    # DO_NOTHING - Do nothing (can break integrity)

    category = models.ForeignKey(
        'Category',
        on_delete=models.SET_NULL,
        null=True,
        blank=True
    )
```

### 2. OneToOneField

```python
class UserProfile(models.Model):
    # One profile → One user
    user = models.OneToOneField(
        'auth.User',
        on_delete=models.CASCADE,
        related_name='profile'
    )
    bio = models.TextField()
    avatar = models.ImageField(upload_to='avatars/')

# Usage:
# user.profile.bio
# profile.user.username
```

### 3. ManyToManyField

```python
class Post(models.Model):
    title = models.CharField(max_length=200)

    # Many posts ↔ Many tags
    tags = models.ManyToManyField('Tag', related_name='posts', blank=True)

    # With intermediate model (for extra fields)
    collaborators = models.ManyToManyField(
        'auth.User',
        through='PostCollaborator',
        related_name='collaborated_posts'
    )

class PostCollaborator(models.Model):
    """Intermediate model for M2M with extra fields"""
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    role = models.CharField(max_length=50)
    joined_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ['post', 'user']
```

## ✅ Best Practices

### 1. Model Methods

```python
from django.urls import reverse

class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    published = models.BooleanField(default=False)

    def __str__(self):
        """String representation"""
        return self.title

    def get_absolute_url(self):
        """Canonical URL for object"""
        return reverse('post_detail', kwargs={'slug': self.slug})

    @property
    def is_new(self):
        """Check if post is new (< 7 days old)"""
        from datetime.datetime import datetime, timedelta
        return self.created_at >= datetime.now() - timedelta(days=7)

    def publish(self):
        """Publish post"""
        self.published = True
        self.save(update_fields=['published'])

    class Meta:
        ordering = ['-created_at']
```

### 2. Custom Managers

```python
from django.db import models

class PublishedManager(models.Manager):
    """Custom manager for published posts"""
    def get_queryset(self):
        return super().get_queryset().filter(published=True)

class Post(models.Model):
    title = models.CharField(max_length=200)
    published = models.BooleanField(default=False)

    objects = models.Manager()  # Default manager
    published_objects = PublishedManager()  # Custom manager

# Usage:
# Post.objects.all()  # All posts
# Post.published_objects.all()  # Only published posts
```

## 🔐 Security Best Practices

### 1. Sensitive Data

```python
from django.contrib.auth.hashers import make_password

class User(models.Model):
    # Never store plain passwords
    password = models.CharField(max_length=128)

    def set_password(self, raw_password):
        self.password = make_password(raw_password)
        self.save()
```

### 2. Input Validation

```python
from django.core.validators import EmailValidator, URLValidator, RegexValidator

class Contact(models.Model):
    email = models.EmailField(validators=[EmailValidator()])

    phone = models.CharField(
        max_length=20,
        validators=[RegexValidator(r'^\+?1?\d{9,15}$', 'Invalid phone')]
    )
```

## ❓ Interview Questions

### Q1: What is the difference between null=True and blank=True?

**Answer**:

- **null=True**: Database allows NULL values
- **blank=True**: Form validation allows empty values

Use both for optional fields: `models.CharField(null=True, blank=True)`

### Q2: When should you use ForeignKey vs OneToOneField?

**Answer**:

- **ForeignKey**: Many-to-one (many posts → one author)
- **OneToOneField**: One-to-one (one user → one profile)

### Q3: What is the purpose of related_name?

**Answer**:
Defines reverse relation name:

```python
author = models.ForeignKey(User, related_name='posts')
# Access: user.posts.all()
```

## 📚 Summary

**Key Takeaways**:

1. Models define database structure
2. CharField for short text, TextField for long text
3. Use DecimalField for money (not FloatField)
4. auto_now_add sets once, auto_now updates always
5. ForeignKey for many-to-one, ManyToManyField for many-to-many
6. null=True (DB), blank=True (forms)
7. Use Meta class for table name, ordering, indexes
8. Custom managers filter default querysets
9. Always validate user input
10. Use TextChoices/IntegerChoices for enums

Django models are the foundation of your application!
