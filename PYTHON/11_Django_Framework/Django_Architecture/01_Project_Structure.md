# Django Project Structure & Organization

## 📖 Concept Explanation

A well-organized Django project structure improves maintainability, scalability, and collaboration. As projects grow, proper structure prevents code chaos and makes features easier to find and modify.

### Standard Django Structure

```
myproject/
├── manage.py
├── myproject/           # Project package
│   ├── __init__.py
│   ├── settings.py      # ❌ Single file doesn't scale
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
└── myapp/              # Single app
    ├── migrations/
    ├── __init__.py
    ├── models.py
    ├── views.py
    └── urls.py
```

### Enterprise Django Structure

```
myproject/
├── manage.py
├── requirements/       # Split requirements
│   ├── base.txt
│   ├── development.txt
│   ├── production.txt
│   └── test.txt
├── config/            # Project configuration
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── development.py
│   │   ├── production.py
│   │   └── test.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── apps/              # All Django apps
│   ├── core/          # Core utilities
│   ├── users/         # User management
│   ├── blog/          # Blog feature
│   └── api/           # API endpoints
├── static/
├── media/
├── templates/
├── tests/             # Project-level tests
├── docs/              # Documentation
└── scripts/           # Management scripts
```

## 🧠 Settings Management

### 1. Split Settings by Environment

```python
# config/settings/base.py (Common settings)
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Third-party
    'rest_framework',
    'django_filters',

    # Local apps
    'apps.core',
    'apps.users',
    'apps.blog',
    'apps.api',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'config.urls'
WSGI_APPLICATION = 'config.wsgi.application'

# Static files
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'
STATICFILES_DIRS = [BASE_DIR / 'static']

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# Templates
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

# Internationalization
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'
USE_I18N = True
USE_TZ = True
```

```python
# config/settings/development.py
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']

# Development database (SQLite)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Email backend (console)
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# Debug toolbar
INSTALLED_APPS += ['debug_toolbar']
MIDDLEWARE += ['debug_toolbar.middleware.DebugToolbarMiddleware']
INTERNAL_IPS = ['127.0.0.1']

# Disable caching
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}
```

```python
# config/settings/production.py
from .base import *
import os

DEBUG = False
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

# Production database (PostgreSQL)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST', 'localhost'),
        'PORT': os.environ.get('DB_PORT', '5432'),
        'CONN_MAX_AGE': 600,
        'OPTIONS': {
            'sslmode': 'require',
        },
    }
}

# Security settings
SECRET_KEY = os.environ.get('SECRET_KEY')
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Redis cache
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': os.environ.get('REDIS_URL', 'redis://127.0.0.1:6379/1'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# Email (SMTP)
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = os.environ.get('EMAIL_HOST')
EMAIL_PORT = int(os.environ.get('EMAIL_PORT', 587))
EMAIL_USE_TLS = True
EMAIL_HOST_USER = os.environ.get('EMAIL_USER')
EMAIL_HOST_PASSWORD = os.environ.get('EMAIL_PASSWORD')

# Logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'ERROR',
            'class': 'logging.FileHandler',
            'filename': BASE_DIR / 'logs' / 'django.log',
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['file'],
        'level': 'ERROR',
    },
}
```

```python
# config/settings/__init__.py
import os

# Read from environment variable
environment = os.environ.get('DJANGO_ENVIRONMENT', 'development')

if environment == 'production':
    from .production import *
elif environment == 'test':
    from .test import *
else:
    from .development import *
```

### 2. Environment Variables

```bash
# .env (use python-decouple or django-environ)
DJANGO_ENVIRONMENT=production
SECRET_KEY=your-secret-key-here
DEBUG=False
ALLOWED_HOSTS=example.com,www.example.com

DB_NAME=mydb
DB_USER=myuser
DB_PASSWORD=mypassword
DB_HOST=db.example.com
DB_PORT=5432

REDIS_URL=redis://localhost:6379/1

EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your-email@gmail.com
EMAIL_PASSWORD=your-app-password
```

```python
# Using django-environ
# pip install django-environ

# config/settings/base.py
import environ

env = environ.Env(
    DEBUG=(bool, False)
)

# Read .env file
environ.Env.read_env(BASE_DIR / '.env')

SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

DATABASES = {
    'default': env.db()  # Reads DATABASE_URL
}
```

## 🎯 App Organization

### 1. Apps by Feature (Recommended)

```
apps/
├── core/              # Shared utilities
│   ├── management/
│   ├── middleware/
│   ├── models.py      # Abstract base models
│   ├── utils.py
│   └── validators.py
│
├── users/             # User management
│   ├── migrations/
│   ├── models.py
│   ├── views.py
│   ├── serializers.py
│   ├── permissions.py
│   ├── signals.py
│   └── tests.py
│
├── blog/              # Blog feature
│   ├── migrations/
│   ├── models.py      # Post, Category, Tag
│   ├── views.py
│   ├── serializers.py
│   ├── urls.py
│   └── tests.py
│
└── api/               # API endpoints
    ├── v1/
    │   ├── urls.py
    │   └── views.py
    └── v2/
        ├── urls.py
        └── views.py
```

### 2. Modular App Structure

```python
# apps/blog/models/
├── __init__.py
├── post.py            # Post model
├── category.py        # Category model
├── tag.py             # Tag model
└── comment.py         # Comment model

# apps/blog/models/__init__.py
from .post import Post
from .category import Category
from .tag import Tag
from .comment import Comment

__all__ = ['Post', 'Category', 'Tag', 'Comment']

# apps/blog/views/
├── __init__.py
├── post_views.py
├── category_views.py
└── comment_views.py

# apps/blog/serializers/
├── __init__.py
├── post_serializers.py
├── category_serializers.py
└── comment_serializers.py
```

### 3. Services Layer (Business Logic)

```python
# apps/blog/services/
├── __init__.py
├── post_service.py
└── notification_service.py

# apps/blog/services/post_service.py
from django.db import transaction
from apps.blog.models import Post
from apps.core.cache import cache_service

class PostService:
    """Business logic for posts"""

    @staticmethod
    @transaction.atomic
    def create_post(title, content, author, tags=None):
        """Create a post with tags"""
        post = Post.objects.create(
            title=title,
            content=content,
            author=author
        )

        if tags:
            post.tags.set(tags)

        # Invalidate cache
        cache_service.delete_pattern('posts:*')

        # Send notification
        from .notification_service import NotificationService
        NotificationService.notify_new_post(post)

        return post

    @staticmethod
    def get_published_posts(category=None):
        """Get published posts with caching"""
        cache_key = f'posts:published:{category or "all"}'

        posts = cache_service.get(cache_key)
        if posts is None:
            queryset = Post.objects.filter(published=True)
            if category:
                queryset = queryset.filter(category=category)

            posts = list(queryset.select_related('author', 'category'))
            cache_service.set(cache_key, posts, timeout=300)

        return posts

# Usage in views
from apps.blog.services.post_service import PostService

def create_post_view(request):
    post = PostService.create_post(
        title=request.POST['title'],
        content=request.POST['content'],
        author=request.user,
        tags=request.POST.getlist('tags')
    )
    return redirect('post-detail', pk=post.pk)
```

## ✅ Best Practices

### 1. Reusable Base Models

```python
# apps/core/models.py
from django.db import models
from django.utils import timezone

class TimestampedModel(models.Model):
    """Abstract base model with created_at and updated_at"""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

class SoftDeleteModel(models.Model):
    """Abstract base model with soft delete"""
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        abstract = True

    def delete(self, *args, **kwargs):
        self.deleted_at = timezone.now()
        self.save()

    def hard_delete(self):
        super().delete()

# Usage
from apps.core.models import TimestampedModel, SoftDeleteModel

class Post(TimestampedModel, SoftDeleteModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
```

### 2. Custom Managers

```python
# apps/blog/managers.py
from django.db import models
from django.utils import timezone

class PublishedManager(models.Manager):
    """Manager for published posts"""
    def get_queryset(self):
        return super().get_queryset().filter(
            published=True,
            deleted_at__isnull=True
        )

class Post(TimestampedModel):
    title = models.CharField(max_length=200)
    published = models.BooleanField(default=False)

    objects = models.Manager()  # Default manager
    published_objects = PublishedManager()  # Custom manager

# Usage
all_posts = Post.objects.all()  # All posts
published_posts = Post.published_objects.all()  # Only published
```

### 3. Centralized Constants

```python
# apps/core/constants.py
class PostStatus:
    DRAFT = 'draft'
    PUBLISHED = 'published'
    ARCHIVED = 'archived'

    CHOICES = [
        (DRAFT, 'Draft'),
        (PUBLISHED, 'Published'),
        (ARCHIVED, 'Archived'),
    ]

class UserRole:
    ADMIN = 'admin'
    EDITOR = 'editor'
    VIEWER = 'viewer'

    CHOICES = [
        (ADMIN, 'Administrator'),
        (EDITOR, 'Editor'),
        (VIEWER, 'Viewer'),
    ]

# Usage
from apps.core.constants import PostStatus

class Post(models.Model):
    status = models.CharField(
        max_length=20,
        choices=PostStatus.CHOICES,
        default=PostStatus.DRAFT
    )
```

## ❌ Common Mistakes

### 1. Single Large App

```python
# ❌ Bad: Everything in one app
myproject/
└── myapp/
    ├── models.py       # 50+ models
    ├── views.py        # 100+ views
    ├── serializers.py  # 50+ serializers
    └── urls.py         # 200+ URL patterns

# ✅ Good: Split by feature
myproject/
└── apps/
    ├── users/
    ├── blog/
    ├── comments/
    ├── notifications/
    └── analytics/
```

### 2. Circular Imports

```python
# ❌ Bad: Circular import
# apps/blog/models.py
from apps.users.models import User

class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)

# apps/users/models.py
from apps.blog.models import Post  # Circular!

class User(AbstractUser):
    favorite_posts = models.ManyToManyField(Post)

# ✅ Good: Use string reference
class User(AbstractUser):
    favorite_posts = models.ManyToManyField('blog.Post')
```

## ❓ Interview Questions

### Q1: How do you organize Django settings for multiple environments?

**Answer**:
Split settings into:

- `base.py`: Common settings
- `development.py`: DEBUG=True, SQLite
- `production.py`: DEBUG=False, PostgreSQL, security
- `test.py`: Fast database, disable migrations

Use environment variable to select: `DJANGO_ENVIRONMENT=production`

### Q2: Should you organize apps by feature or by layer?

**Answer**:
**By feature** (recommended): Each app is a complete feature (users, blog, payments).
**By layer**: Separate models, views, serializers into different apps (harder to maintain).

Feature-based is more maintainable as projects grow.

## 📚 Summary

**Key Takeaways**:

1. Split settings by environment (base, development, production)
2. Use environment variables for secrets
3. Organize apps by feature, not layer
4. Create core app for shared utilities
5. Use abstract base models for common fields
6. Separate business logic into services
7. Avoid circular imports with string references
8. Use consistent naming conventions
9. Keep apps small and focused
10. Document structure in README

Proper project structure is crucial for long-term maintainability!
