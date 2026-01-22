# Django Settings & Configuration

## 📖 Concept Explanation

Django settings module is the central configuration file for your Django project. It contains all application settings: database configuration, installed apps, middleware, security settings, static files, and more.

### Why Settings Matter

- **Single source of truth** for application configuration
- **Environment-specific** settings (dev, staging, prod)
- **Security-critical** settings (SECRET_KEY, ALLOWED_HOSTS)
- **Performance tuning** (caching, database connections)
- **Third-party integration** configuration

### Default Settings Location

```
myproject/
    myproject/
        __init__.py
        settings.py    # Main settings file
        urls.py
        wsgi.py
    manage.py
```

## 🧠 Key Settings Categories

### 1. Core Settings

```python
# settings.py
import os
from pathlib import Path

# Build paths inside the project
BASE_DIR = Path(__file__).resolve().parent.parent

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = os.environ.get('SECRET_KEY', 'dev-secret-key-change-in-production')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = os.environ.get('DEBUG', 'False') == 'True'

# Hosts allowed to serve the application
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', 'localhost,127.0.0.1').split(',')

# Application definition
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Third-party apps
    'rest_framework',
    'corsheaders',
    'django_celery_beat',

    # Local apps
    'accounts',
    'products',
    'orders',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'myproject.urls'

WSGI_APPLICATION = 'myproject.wsgi.application'
```

### 2. Database Configuration

```python
# Single database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', 'mydb'),
        'USER': os.environ.get('DB_USER', 'postgres'),
        'PASSWORD': os.environ.get('DB_PASSWORD', ''),
        'HOST': os.environ.get('DB_HOST', 'localhost'),
        'PORT': os.environ.get('DB_PORT', '5432'),
        'CONN_MAX_AGE': 600,  # Connection pooling
        'OPTIONS': {
            'connect_timeout': 10,
        }
    }
}

# Multiple databases
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'primary_db',
        # ... other settings
    },
    'analytics': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'analytics_db',
        # ... other settings
    },
    'cache_db': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'cache.sqlite3',
    }
}

# Database router for multiple databases
DATABASE_ROUTERS = ['myproject.routers.DatabaseRouter']
```

### 3. Cache Configuration

```python
# Redis cache
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': os.environ.get('REDIS_URL', 'redis://127.0.0.1:6379/1'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,  # 5 minutes default
    },
    'sessions': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': os.environ.get('REDIS_URL', 'redis://127.0.0.1:6379/2'),
        'TIMEOUT': 86400,  # 24 hours
    }
}

# Use Redis for sessions
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'sessions'
```

### 4. Static & Media Files

```python
# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'
STATICFILES_DIRS = [
    BASE_DIR / 'static',
]

STATICFILES_FINDERS = [
    'django.contrib.staticfiles.finders.FileSystemFinder',
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
]

# Media files (User uploads)
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# For production with S3
if not DEBUG:
    # AWS S3 Settings
    AWS_ACCESS_KEY_ID = os.environ.get('AWS_ACCESS_KEY_ID')
    AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_SECRET_ACCESS_KEY')
    AWS_STORAGE_BUCKET_NAME = os.environ.get('AWS_STORAGE_BUCKET_NAME')
    AWS_S3_REGION_NAME = 'us-east-1'
    AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'

    # Static and media files
    STATIC_URL = f'https://{AWS_S3_CUSTOM_DOMAIN}/static/'
    MEDIA_URL = f'https://{AWS_S3_CUSTOM_DOMAIN}/media/'

    DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
    STATICFILES_STORAGE = 'storages.backends.s3boto3.S3StaticStorage'
```

### 5. Security Settings

```python
# Security settings
SECURE_SSL_REDIRECT = not DEBUG
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True

# CSRF
CSRF_COOKIE_SECURE = not DEBUG
CSRF_COOKIE_HTTPONLY = True
CSRF_TRUSTED_ORIGINS = [
    'https://yourdomain.com',
    'https://www.yourdomain.com',
]

# Session
SESSION_COOKIE_SECURE = not DEBUG
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = 'Lax'
SESSION_COOKIE_AGE = 86400  # 24 hours

# CORS
CORS_ALLOWED_ORIGINS = [
    'https://yourdomain.com',
    'https://www.yourdomain.com',
]
CORS_ALLOW_CREDENTIALS = True
```

### 6. Email Configuration

```python
# Email backend
if DEBUG:
    EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
else:
    EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
    EMAIL_HOST = os.environ.get('EMAIL_HOST', 'smtp.gmail.com')
    EMAIL_PORT = int(os.environ.get('EMAIL_PORT', 587))
    EMAIL_USE_TLS = True
    EMAIL_HOST_USER = os.environ.get('EMAIL_HOST_USER')
    EMAIL_HOST_PASSWORD = os.environ.get('EMAIL_HOST_PASSWORD')
    DEFAULT_FROM_EMAIL = os.environ.get('DEFAULT_FROM_EMAIL', 'noreply@example.com')

# For production with SendGrid/Mailgun
# EMAIL_BACKEND = 'sendgrid_backend.SendgridBackend'
# SENDGRID_API_KEY = os.environ.get('SENDGRID_API_KEY')
```

### 7. Logging Configuration

```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
        'simple': {
            'format': '{levelname} {message}',
            'style': '{',
        },
    },
    'filters': {
        'require_debug_false': {
            '()': 'django.utils.log.RequireDebugFalse',
        },
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
    },
    'handlers': {
        'console': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'simple',
        },
        'file': {
            'level': 'WARNING',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': BASE_DIR / 'logs' / 'django.log',
            'maxBytes': 1024 * 1024 * 10,  # 10 MB
            'backupCount': 5,
            'formatter': 'verbose',
        },
        'mail_admins': {
            'level': 'ERROR',
            'class': 'django.utils.log.AdminEmailHandler',
            'filters': ['require_debug_false'],
        },
    },
    'root': {
        'handlers': ['console', 'file'],
        'level': 'INFO',
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': False,
        },
        'django.request': {
            'handlers': ['mail_admins', 'file'],
            'level': 'ERROR',
            'propagate': False,
        },
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG' if DEBUG else 'INFO',
            'propagate': False,
        },
    },
}
```

## ✅ Best Practices

### 1. Environment-Based Settings

```python
# settings/__init__.py
import os

environment = os.environ.get('DJANGO_ENV', 'development')

if environment == 'production':
    from .production import *
elif environment == 'staging':
    from .staging import *
else:
    from .development import *

# settings/base.py - Shared settings
# settings/development.py - Dev-specific
# settings/staging.py - Staging-specific
# settings/production.py - Production-specific
```

### 2. Environment Variables with python-decouple

```python
# pip install python-decouple

from decouple import config, Csv

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=Csv())
DATABASE_URL = config('DATABASE_URL')

# .env file
# SECRET_KEY=your-secret-key
# DEBUG=True
# ALLOWED_HOSTS=localhost,127.0.0.1
# DATABASE_URL=postgresql://user:pass@localhost/dbname
```

### 3. django-environ for Settings

```python
# pip install django-environ

import environ

env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, []),
)

# Read .env file
environ.Env.read_env(BASE_DIR / '.env')

SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
ALLOWED_HOSTS = env('ALLOWED_HOSTS')

DATABASES = {
    'default': env.db()  # Parses DATABASE_URL automatically
}

CACHES = {
    'default': env.cache()  # Parses CACHE_URL automatically
}
```

### 4. Custom Settings Manager

```python
# config/settings.py
from typing import Optional

class Settings:
    """Centralized settings manager"""

    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        from django.conf import settings
        self.settings = settings

    def get(self, key: str, default: Optional[any] = None):
        return getattr(self.settings, key, default)

    def is_production(self) -> bool:
        return not self.settings.DEBUG

    def is_debug(self) -> bool:
        return self.settings.DEBUG

    def get_cache_timeout(self, key: str = 'default') -> int:
        timeouts = {
            'default': 300,
            'long': 3600,
            'short': 60,
        }
        return timeouts.get(key, 300)

# Usage
settings_manager = Settings()
if settings_manager.is_production():
    # Production-specific code
    pass
```

### 5. Dynamic Settings Based on Hostname

```python
import socket

HOSTNAME = socket.gethostname()

if 'prod' in HOSTNAME:
    DEBUG = False
    ALLOWED_HOSTS = ['yourdomain.com', 'www.yourdomain.com']
elif 'staging' in HOSTNAME:
    DEBUG = False
    ALLOWED_HOSTS = ['staging.yourdomain.com']
else:
    DEBUG = True
    ALLOWED_HOSTS = ['*']
```

## 🔐 Security Best Practices

### 1. Secure SECRET_KEY Generation

```python
from django.core.management.utils import get_random_secret_key

# Generate new secret key
new_secret_key = get_random_secret_key()
print(new_secret_key)

# In production, store in environment variable
SECRET_KEY = os.environ.get('SECRET_KEY')
if not SECRET_KEY:
    raise ValueError("SECRET_KEY environment variable not set")
```

### 2. Security Checklist Settings

```python
# Run: python manage.py check --deploy

# Production security settings
if not DEBUG:
    # SSL/HTTPS
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True

    # HSTS
    SECURE_HSTS_SECONDS = 31536000
    SECURE_HSTS_INCLUDE_SUBDOMAINS = True
    SECURE_HSTS_PRELOAD = True

    # Security headers
    SECURE_CONTENT_TYPE_NOSNIFF = True
    SECURE_BROWSER_XSS_FILTER = True
    X_FRAME_OPTIONS = 'DENY'

    # Admin URL protection
    ADMIN_URL = os.environ.get('ADMIN_URL', 'admin/')
```

### 3. Database Connection Security

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST'),
        'PORT': os.environ.get('DB_PORT', '5432'),
        'OPTIONS': {
            'sslmode': 'require',  # Require SSL
            'connect_timeout': 10,
        },
        'CONN_MAX_AGE': 600,
    }
}
```

## 🚀 Performance Optimization Settings

### 1. Database Optimization

```python
# Connection pooling
DATABASES = {
    'default': {
        # ... other settings
        'CONN_MAX_AGE': 600,  # Keep connections for 10 minutes
        'OPTIONS': {
            'MAX_CONNS': 20,  # Maximum connections
        }
    }
}

# Query optimization
DEBUG = False  # Disables query logging overhead
```

### 2. Template Caching

```python
if not DEBUG:
    TEMPLATES = [{
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'OPTIONS': {
            'context_processors': [
                # ... processors
            ],
            'loaders': [
                ('django.template.loaders.cached.Loader', [
                    'django.template.loaders.filesystem.Loader',
                    'django.template.loaders.app_directories.Loader',
                ]),
            ],
        },
    }]
```

### 3. Session Optimization

```python
# Use cached_db sessions for better performance
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
SESSION_CACHE_ALIAS = 'default'
SESSION_COOKIE_AGE = 86400  # 24 hours
```

## ❌ Common Mistakes

### 1. Hardcoding Secrets

```python
# WRONG
SECRET_KEY = 'django-insecure-hardcoded-key'
DEBUG = True

# CORRECT
SECRET_KEY = os.environ.get('SECRET_KEY')
DEBUG = os.environ.get('DEBUG', 'False') == 'True'
```

### 2. Using DEBUG=True in Production

```python
# WRONG - Never in production
DEBUG = True

# CORRECT - Environment-based
DEBUG = os.environ.get('DJANGO_ENV') != 'production'
```

### 3. Allowing All Hosts

```python
# WRONG
ALLOWED_HOSTS = ['*']

# CORRECT
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')
```

## ❓ Interview Questions

### Q1: What is the purpose of SECRET_KEY in Django?

**Answer**: SECRET_KEY is used for:

- Cryptographic signing (sessions, cookies)
- Password reset tokens
- CSRF protection
- Generating secure hashes

It must be:

- Unique per project
- Kept secret (never in version control)
- At least 50 characters long
- Random and unpredictable

### Q2: How do you manage different settings for dev/staging/prod?

**Answer**: Multiple approaches:

1. **Separate files**: `settings/development.py`, `settings/production.py`
2. **Environment variables**: Use `python-decouple` or `django-environ`
3. **Environment detection**: Check `DJANGO_ENV` variable
4. **Hostname-based**: Detect based on server hostname

Best practice: Combine environment variables with separate settings files.

### Q3: What happens if ALLOWED_HOSTS is misconfigured?

**Answer**:

- Django returns 400 Bad Request for unrecognized hosts
- Prevents Host header attacks
- Production requires explicit host list
- Never use `['*']` in production

### Q4: How does CONN_MAX_AGE improve performance?

**Answer**:

- Keeps database connections alive for reuse
- Reduces connection overhead (handshake, authentication)
- Value in seconds (0 = close after each request)
- Typical values: 600 (10 min) or None (persistent)
- Must match database's max connection limit

### Q5: What security headers should be enabled in production?

**Answer**:

```python
SECURE_SSL_REDIRECT = True           # Force HTTPS
SECURE_HSTS_SECONDS = 31536000       # HSTS for 1 year
SECURE_CONTENT_TYPE_NOSNIFF = True   # Prevent MIME sniffing
SECURE_BROWSER_XSS_FILTER = True     # XSS protection
X_FRAME_OPTIONS = 'DENY'             # Clickjacking protection
CSRF_COOKIE_SECURE = True            # CSRF over HTTPS only
SESSION_COOKIE_SECURE = True         # Session over HTTPS only
```

## 📚 Summary

**Key Takeaways**:

1. Never hardcode secrets - use environment variables
2. Separate settings for different environments
3. Enable security headers in production
4. Use connection pooling for database performance
5. Configure caching appropriately
6. Set up proper logging
7. Run `python manage.py check --deploy` before deployment

**Essential Settings**:

- SECRET_KEY, DEBUG, ALLOWED_HOSTS
- DATABASES with SSL and connection pooling
- CACHES for Redis/Memcached
- Security headers (HSTS, CSP, etc.)
- Static/Media files configuration
- Email backend
- Logging

Django settings are the foundation of your application's configuration, security, and performance!
