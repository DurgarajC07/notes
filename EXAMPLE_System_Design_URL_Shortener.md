# System Design: URL Shortener (Like bit.ly)

## 📖 Problem Statement

Design a URL shortening service like bit.ly, TinyURL, or goo.gl that:

- Converts long URLs to short URLs
- Redirects short URLs to original long URLs
- Tracks analytics (click counts, timestamps)
- Handles high traffic (millions of requests/day)

## 🎯 Requirements

### Functional Requirements

1. Generate unique short URL for given long URL
2. Redirect short URL to original URL
3. Custom short URLs (optional)
4. URL expiration (optional)
5. Analytics tracking

### Non-Functional Requirements

1. **High Availability**: 99.9% uptime
2. **Low Latency**: <100ms redirect time
3. **Scalability**: Handle 100M URLs, 10K requests/second
4. **Durability**: URLs never lost
5. **Security**: Prevent abuse, handle malicious URLs

## 📊 Capacity Estimation

### Traffic Estimates

- **Write requests**: 100M new URLs/month = ~40 URLs/second
- **Read requests**: 100:1 read/write ratio = 4000 reads/second
- **Peak traffic**: 3x average = 12K reads/second

### Storage Estimates

- **URL size**: Average 500 bytes/URL
- **Total URLs**: 100M new/month × 12 months × 5 years = 6B URLs
- **Storage needed**: 6B × 500 bytes = 3 TB

### Bandwidth Estimates

- **Write bandwidth**: 40 URLs/sec × 500 bytes = 20 KB/sec
- **Read bandwidth**: 4000 URLs/sec × 500 bytes = 2 MB/sec

## 🏗️ High-Level Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│   Load Balancer     │
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
┌─────────┐   ┌─────────┐
│  API    │   │  API    │
│ Server  │   │ Server  │
└────┬────┘   └────┬────┘
     │             │
     └──────┬──────┘
            │
     ┌──────┴──────┐
     ▼             ▼
┌─────────┐   ┌──────────┐
│  Cache  │   │ Database │
│ (Redis) │   │(Postgres)│
└─────────┘   └──────────┘
```

## 🔑 Core Components

### 1. Short URL Generation

#### Base62 Encoding Approach

```python
import string
import hashlib
from typing import Optional

class URLShortener:
    """URL shortening with Base62 encoding"""

    # Base62: 0-9, a-z, A-Z
    ALPHABET = string.digits + string.ascii_lowercase + string.ascii_uppercase
    BASE = len(ALPHABET)  # 62

    def __init__(self, min_length=6):
        self.min_length = min_length

    def encode(self, number: int) -> str:
        """Convert integer ID to short string"""
        if number == 0:
            return self.ALPHABET[0]

        result = []
        while number:
            number, remainder = divmod(number, self.BASE)
            result.append(self.ALPHABET[remainder])

        # Pad to minimum length
        while len(result) < self.min_length:
            result.append(self.ALPHABET[0])

        return ''.join(reversed(result))

    def decode(self, short_code: str) -> int:
        """Convert short string back to integer ID"""
        number = 0
        for char in short_code:
            number = number * self.BASE + self.ALPHABET.index(char)
        return number

# Example usage
shortener = URLShortener(min_length=6)

# URL with ID 12345 becomes short code
short_code = shortener.encode(12345)  # "3D7"
print(f"Short code: {short_code}")

# Decode back to ID
url_id = shortener.decode(short_code)  # 12345
print(f"URL ID: {url_id}")
```

#### Hash-Based Approach (Alternative)

```python
import hashlib
import base64

def generate_short_url(long_url: str, length: int = 6) -> str:
    """Generate short URL using MD5 hash"""
    # Create MD5 hash of URL
    hash_object = hashlib.md5(long_url.encode())
    hash_hex = hash_object.hexdigest()

    # Convert to Base64 and take first N characters
    hash_bytes = bytes.fromhex(hash_hex)
    base64_str = base64.urlsafe_b64encode(hash_bytes).decode('utf-8')

    return base64_str[:length]

# Example
short_code = generate_short_url("https://example.com/very/long/url")
print(f"Short code: {short_code}")  # "Xy7K2p"
```

### 2. Database Schema

```sql
-- URLs table
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);

-- Analytics table
CREATE TABLE url_analytics (
    id BIGSERIAL PRIMARY KEY,
    url_id BIGINT REFERENCES urls(id),
    accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET,
    user_agent TEXT,
    referer TEXT
);

-- Indexes for performance
CREATE INDEX idx_urls_short_code ON urls(short_code);
CREATE INDEX idx_urls_user_id ON urls(user_id);
CREATE INDEX idx_analytics_url_id ON url_analytics(url_id);
CREATE INDEX idx_analytics_accessed_at ON url_analytics(accessed_at);
```

### 3. Django Implementation

```python
# models.py
from django.db import models
from django.contrib.auth.models import User
from django.utils import timezone
import string
import random

class URL(models.Model):
    """URL model with short code generation"""

    short_code = models.CharField(max_length=10, unique=True, db_index=True)
    long_url = models.URLField(max_length=2048)
    user = models.ForeignKey(User, on_delete=models.CASCADE, null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    expires_at = models.DateTimeField(null=True, blank=True)
    is_active = models.BooleanField(default=True)
    click_count = models.PositiveIntegerField(default=0)

    class Meta:
        db_table = 'urls'
        indexes = [
            models.Index(fields=['short_code']),
            models.Index(fields=['user', 'created_at']),
        ]

    def __str__(self):
        return f"{self.short_code} -> {self.long_url}"

    @classmethod
    def generate_short_code(cls, length=6):
        """Generate unique short code"""
        chars = string.ascii_letters + string.digits
        while True:
            code = ''.join(random.choices(chars, k=length))
            if not cls.objects.filter(short_code=code).exists():
                return code

    def is_expired(self):
        """Check if URL has expired"""
        if self.expires_at:
            return timezone.now() > self.expires_at
        return False

    def increment_clicks(self):
        """Increment click counter"""
        self.click_count = models.F('click_count') + 1
        self.save(update_fields=['click_count'])

class URLAnalytics(models.Model):
    """Track URL access analytics"""

    url = models.ForeignKey(URL, on_delete=models.CASCADE, related_name='analytics')
    accessed_at = models.DateTimeField(auto_now_add=True)
    ip_address = models.GenericIPAddressField()
    user_agent = models.TextField(blank=True)
    referer = models.URLField(max_length=2048, blank=True)
    country = models.CharField(max_length=2, blank=True)

    class Meta:
        db_table = 'url_analytics'
        indexes = [
            models.Index(fields=['url', 'accessed_at']),
        ]

# services.py
from django.core.cache import cache
from django.db import transaction
from typing import Optional
import logging

logger = logging.getLogger(__name__)

class URLShortenerService:
    """Business logic for URL shortening"""

    CACHE_TTL = 3600  # 1 hour

    @staticmethod
    def create_short_url(
        long_url: str,
        user: Optional[User] = None,
        custom_code: Optional[str] = None,
        expires_at: Optional[timezone.datetime] = None
    ) -> URL:
        """Create new short URL"""

        # Check if long URL already shortened by this user
        existing = URL.objects.filter(
            long_url=long_url,
            user=user,
            is_active=True
        ).first()

        if existing and not existing.is_expired():
            return existing

        # Generate or validate custom short code
        if custom_code:
            if URL.objects.filter(short_code=custom_code).exists():
                raise ValueError("Custom code already taken")
            short_code = custom_code
        else:
            short_code = URL.generate_short_code()

        # Create URL
        with transaction.atomic():
            url = URL.objects.create(
                short_code=short_code,
                long_url=long_url,
                user=user,
                expires_at=expires_at
            )

        # Cache the mapping
        cache_key = f"short_url:{short_code}"
        cache.set(cache_key, url.long_url, URLShortenerService.CACHE_TTL)

        logger.info(f"Created short URL: {short_code} for {long_url}")
        return url

    @staticmethod
    def get_long_url(short_code: str) -> Optional[str]:
        """Get long URL from short code (with caching)"""

        # Try cache first
        cache_key = f"short_url:{short_code}"
        long_url = cache.get(cache_key)

        if long_url:
            logger.debug(f"Cache hit for {short_code}")
            return long_url

        # Cache miss - query database
        try:
            url = URL.objects.get(short_code=short_code, is_active=True)

            # Check expiration
            if url.is_expired():
                url.is_active = False
                url.save(update_fields=['is_active'])
                return None

            # Cache for next time
            cache.set(cache_key, url.long_url, URLShortenerService.CACHE_TTL)

            logger.debug(f"Database hit for {short_code}")
            return url.long_url

        except URL.DoesNotExist:
            logger.warning(f"Short code not found: {short_code}")
            return None

    @staticmethod
    def track_access(
        short_code: str,
        ip_address: str,
        user_agent: str = "",
        referer: str = ""
    ):
        """Track URL access asynchronously"""
        try:
            url = URL.objects.get(short_code=short_code)

            # Increment counter
            url.increment_clicks()

            # Create analytics record
            URLAnalytics.objects.create(
                url=url,
                ip_address=ip_address,
                user_agent=user_agent,
                referer=referer
            )
        except URL.DoesNotExist:
            logger.error(f"Cannot track access for unknown code: {short_code}")

# views.py (Django REST Framework)
from rest_framework import status
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import AllowAny, IsAuthenticated
from django.shortcuts import redirect
from django.http import HttpResponseNotFound

class ShortenURLView(APIView):
    """Create short URL"""
    permission_classes = [AllowAny]

    def post(self, request):
        long_url = request.data.get('long_url')
        custom_code = request.data.get('custom_code')

        if not long_url:
            return Response(
                {"error": "long_url is required"},
                status=status.HTTP_400_BAD_REQUEST
            )

        try:
            url = URLShortenerService.create_short_url(
                long_url=long_url,
                user=request.user if request.user.is_authenticated else None,
                custom_code=custom_code
            )

            short_url = request.build_absolute_uri(f'/{url.short_code}')

            return Response({
                "short_url": short_url,
                "short_code": url.short_code,
                "long_url": url.long_url,
                "created_at": url.created_at
            }, status=status.HTTP_201_CREATED)

        except ValueError as e:
            return Response(
                {"error": str(e)},
                status=status.HTTP_400_BAD_REQUEST
            )

class RedirectView(APIView):
    """Redirect short URL to long URL"""
    permission_classes = [AllowAny]

    def get(self, request, short_code):
        long_url = URLShortenerService.get_long_url(short_code)

        if not long_url:
            return HttpResponseNotFound("URL not found or expired")

        # Track access asynchronously (use Celery in production)
        URLShortenerService.track_access(
            short_code=short_code,
            ip_address=self.get_client_ip(request),
            user_agent=request.META.get('HTTP_USER_AGENT', ''),
            referer=request.META.get('HTTP_REFERER', '')
        )

        return redirect(long_url, permanent=False)  # 302 redirect

    @staticmethod
    def get_client_ip(request):
        """Extract client IP address"""
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            return x_forwarded_for.split(',')[0]
        return request.META.get('REMOTE_ADDR')

class URLAnalyticsView(APIView):
    """Get analytics for short URL"""
    permission_classes = [IsAuthenticated]

    def get(self, request, short_code):
        try:
            url = URL.objects.get(short_code=short_code)

            # Check ownership
            if url.user != request.user:
                return Response(
                    {"error": "Not authorized"},
                    status=status.HTTP_403_FORBIDDEN
                )

            # Aggregate analytics
            from django.db.models import Count
            from django.db.models.functions import TruncDate

            daily_clicks = url.analytics.annotate(
                date=TruncDate('accessed_at')
            ).values('date').annotate(
                clicks=Count('id')
            ).order_by('-date')[:30]

            return Response({
                "short_code": url.short_code,
                "long_url": url.long_url,
                "total_clicks": url.click_count,
                "daily_clicks": list(daily_clicks),
                "created_at": url.created_at
            })

        except URL.DoesNotExist:
            return Response(
                {"error": "URL not found"},
                status=status.HTTP_404_NOT_FOUND
            )

# urls.py
from django.urls import path

urlpatterns = [
    path('api/shorten/', ShortenURLView.as_view(), name='shorten'),
    path('api/analytics/<str:short_code>/', URLAnalyticsView.as_view(), name='analytics'),
    path('<str:short_code>/', RedirectView.as_view(), name='redirect'),
]
```

### 4. FastAPI Implementation

```python
# main.py
from fastapi import FastAPI, HTTPException, Request, Depends
from fastapi.responses import RedirectResponse
from pydantic import BaseModel, HttpUrl
from sqlalchemy.orm import Session
from typing import Optional
import logging

app = FastAPI(title="URL Shortener API")

# Models
class URLCreate(BaseModel):
    long_url: HttpUrl
    custom_code: Optional[str] = None
    expires_in_days: Optional[int] = None

class URLResponse(BaseModel):
    short_url: str
    short_code: str
    long_url: str
    created_at: datetime
    expires_at: Optional[datetime]

class AnalyticsResponse(BaseModel):
    short_code: str
    long_url: str
    total_clicks: int
    daily_clicks: list

# Routes
@app.post("/api/shorten", response_model=URLResponse)
async def shorten_url(
    url_data: URLCreate,
    request: Request,
    db: Session = Depends(get_db)
):
    """Create short URL"""
    try:
        url = await URLShortenerService.create_short_url(
            db=db,
            long_url=str(url_data.long_url),
            custom_code=url_data.custom_code
        )

        short_url = f"{request.base_url}{url.short_code}"

        return URLResponse(
            short_url=short_url,
            short_code=url.short_code,
            long_url=url.long_url,
            created_at=url.created_at,
            expires_at=url.expires_at
        )

    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@app.get("/{short_code}")
async def redirect_url(
    short_code: str,
    request: Request,
    background_tasks: BackgroundTasks,
    db: Session = Depends(get_db)
):
    """Redirect to long URL"""
    long_url = await URLShortenerService.get_long_url(db, short_code)

    if not long_url:
        raise HTTPException(status_code=404, detail="URL not found")

    # Track analytics in background
    background_tasks.add_task(
        URLShortenerService.track_access,
        db=db,
        short_code=short_code,
        ip_address=request.client.host,
        user_agent=request.headers.get("user-agent", "")
    )

    return RedirectResponse(url=long_url, status_code=302)

@app.get("/api/analytics/{short_code}", response_model=AnalyticsResponse)
async def get_analytics(
    short_code: str,
    db: Session = Depends(get_db)
):
    """Get URL analytics"""
    analytics = await URLShortenerService.get_analytics(db, short_code)
    return analytics
```

## 🚀 Scalability Optimizations

### 1. Caching Strategy

```python
# Redis caching
import redis
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def cache_short_url(ttl=3600):
    """Cache decorator for short URL lookups"""
    def decorator(func):
        @wraps(func)
        async def wrapper(short_code: str, *args, **kwargs):
            # Try cache
            cache_key = f"url:{short_code}"
            cached = redis_client.get(cache_key)

            if cached:
                return cached

            # Cache miss - query database
            result = await func(short_code, *args, **kwargs)

            if result:
                redis_client.setex(cache_key, ttl, result)

            return result
        return wrapper
    return decorator

@cache_short_url(ttl=3600)
async def get_long_url(short_code: str, db: Session):
    url = db.query(URL).filter(URL.short_code == short_code).first()
    return url.long_url if url else None
```

### 2. Database Sharding

```python
# Shard URLs by hash of short_code
def get_shard_id(short_code: str, num_shards: int = 4) -> int:
    """Determine which database shard to use"""
    return hash(short_code) % num_shards

# Use routing middleware
class DatabaseRouter:
    def db_for_read(self, model, **hints):
        if model == URL:
            short_code = hints.get('short_code')
            if short_code:
                shard_id = get_shard_id(short_code)
                return f'shard_{shard_id}'
        return 'default'
```

### 3. Rate Limiting

```python
from fastapi import HTTPException
from collections import defaultdict
import time

class RateLimiter:
    def __init__(self, max_requests=100, window=60):
        self.max_requests = max_requests
        self.window = window
        self.requests = defaultdict(list)

    def is_allowed(self, client_id: str) -> bool:
        now = time.time()

        # Remove old requests outside window
        self.requests[client_id] = [
            req_time for req_time in self.requests[client_id]
            if now - req_time < self.window
        ]

        # Check if limit exceeded
        if len(self.requests[client_id]) >= self.max_requests:
            return False

        # Record new request
        self.requests[client_id].append(now)
        return True

rate_limiter = RateLimiter(max_requests=100, window=60)

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    client_ip = request.client.host

    if not rate_limiter.is_allowed(client_ip):
        raise HTTPException(status_code=429, detail="Rate limit exceeded")

    response = await call_next(request)
    return response
```

## 🔐 Security Considerations

```python
# URL validation
from urllib.parse import urlparse
import re

BLACKLISTED_DOMAINS = ['malicious.com', 'phishing.net']

def validate_url(url: str) -> bool:
    """Validate URL for security"""
    try:
        parsed = urlparse(url)

        # Check protocol
        if parsed.scheme not in ['http', 'https']:
            return False

        # Check blacklisted domains
        if any(domain in parsed.netloc for domain in BLACKLISTED_DOMAINS):
            return False

        # Check for suspicious patterns
        suspicious = ['javascript:', 'data:', 'vbscript:']
        if any(pattern in url.lower() for pattern in suspicious):
            return False

        return True

    except:
        return False
```

## ❓ Interview Questions

### Q1: How do you generate unique short codes?

**Answer**: Two main approaches:

1. **Auto-increment ID + Base62 encoding**: Guaranteed unique, sequential
2. **Hash-based**: Take hash of URL, use first N characters, handle collisions

Base62 is preferred for predictable uniqueness.

### Q2: How do you handle collisions in hash-based approach?

**Answer**:

- Try different hash functions (MD5, SHA256)
- Append counter and rehash
- Fall back to random generation
- Check database before committing

### Q3: How would you scale to 10K requests/second?

**Answer**:

- **Caching**: Redis for hot URLs (90% hit rate)
- **CDN**: Cache redirects at edge
- **Load balancing**: Multiple API servers
- **Database**: Read replicas, sharding
- **Async processing**: Queue analytics tracking

### Q4: How do you prevent abuse?

**Answer**:

- Rate limiting per IP/user
- CAPTCHA for anonymous users
- URL validation and blacklisting
- Monitor for spam patterns
- Require authentication for custom codes

### Q5: How do you make the system highly available?

**Answer**:

- Multi-region deployment
- Database replication (master-slave)
- Cache replication (Redis cluster)
- Health checks and auto-scaling
- Circuit breakers for dependencies
- Graceful degradation (serve from cache even if DB down)

---

## 📚 Summary

URL Shortener demonstrates:

- **Database design**: Proper indexing, relationships
- **Caching strategies**: Redis for performance
- **Scalability patterns**: Sharding, load balancing
- **API design**: RESTful, versioning
- **Security**: Validation, rate limiting
- **Analytics**: Tracking, aggregation

**Key Takeaways**:

1. Use Base62 encoding for unique short codes
2. Cache aggressively (90%+ hit rate)
3. Track analytics asynchronously
4. Implement rate limiting
5. Validate and sanitize URLs
6. Plan for scale from the start

This pattern applies to many system design problems: short URLs, file sharing, image uploads, etc.
