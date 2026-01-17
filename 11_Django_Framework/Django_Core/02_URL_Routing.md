# Django URL Routing & Configuration

## 📖 Concept Explanation

Django's URL dispatcher (URLconf) maps URL patterns to view functions/classes. It's the routing system that determines which code executes for each incoming request.

### How URL Routing Works

```
Browser Request → URL Pattern Match → View Function → Response
```

**Flow**:

1. Django receives HTTP request
2. Loads ROOT_URLCONF from settings
3. Compiles URL patterns
4. Matches request path against patterns (top to bottom)
5. Calls matched view with HttpRequest
6. Returns HttpResponse

### Basic URL Configuration

```python
# myproject/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
    path('users/', include('users.urls')),
]
```

## 🧠 URL Pattern Types

### 1. path() - Simple Patterns

```python
from django.urls import path
from . import views

urlpatterns = [
    # Exact match
    path('', views.index, name='index'),
    path('about/', views.about, name='about'),
    path('contact/', views.contact, name='contact'),

    # Path converters
    path('users/<int:user_id>/', views.user_detail, name='user_detail'),
    path('posts/<slug:slug>/', views.post_detail, name='post_detail'),
    path('articles/<uuid:article_id>/', views.article_detail, name='article_detail'),
]

# Path converters:
# <int:id> - matches integer
# <str:name> - matches any string (default)
# <slug:slug> - matches slugs (letters, numbers, hyphens, underscores)
# <uuid:uuid> - matches UUID
# <path:path> - matches any string including slashes
```

### 2. re_path() - Regex Patterns

```python
from django.urls import re_path
from . import views

urlpatterns = [
    # Regex patterns for complex matching
    re_path(r'^articles/(?P<year>[0-9]{4})/$', views.year_archive),
    re_path(r'^articles/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/$', views.month_archive),
    re_path(r'^users/(?P<username>[\w.@+-]+)/$', views.user_profile),

    # Regex with optional group
    re_path(r'^blog/(?P<page>\d+)?/$', views.blog_list),  # page is optional
]
```

### 3. include() - Including Other URLconfs

```python
# myproject/urls.py
from django.urls import path, include

urlpatterns = [
    # Include app URLs
    path('blog/', include('blog.urls')),
    path('api/v1/', include('api.v1.urls')),
    path('api/v2/', include('api.v2.urls')),

    # Include with namespace
    path('dashboard/', include(('dashboard.urls', 'dashboard'), namespace='dashboard')),
]

# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'  # Application namespace

urlpatterns = [
    path('', views.post_list, name='list'),
    path('<int:pk>/', views.post_detail, name='detail'),
    path('create/', views.post_create, name='create'),
]

# Usage: reverse('blog:detail', kwargs={'pk': 1})
```

## 🏗️ Real-World URL Patterns

### 1. RESTful API URLs

```python
# api/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

# Router for ViewSets
router = DefaultRouter()
router.register(r'users', views.UserViewSet, basename='user')
router.register(r'posts', views.PostViewSet, basename='post')
router.register(r'comments', views.CommentViewSet, basename='comment')

# Manual API routes
urlpatterns = [
    # Router URLs
    path('', include(router.urls)),

    # Custom endpoints
    path('auth/login/', views.LoginView.as_view(), name='login'),
    path('auth/logout/', views.LogoutView.as_view(), name='logout'),
    path('auth/refresh/', views.RefreshTokenView.as_view(), name='refresh'),

    # Nested resources
    path('posts/<int:post_id>/comments/', views.PostCommentsList.as_view()),
    path('users/<int:user_id>/posts/', views.UserPostsList.as_view()),

    # Actions
    path('posts/<int:pk>/publish/', views.publish_post, name='publish_post'),
    path('posts/<int:pk>/archive/', views.archive_post, name='archive_post'),
]
```

### 2. Versioned API URLs

```python
# api/urls.py
from django.urls import path, include

urlpatterns = [
    # API versioning
    path('v1/', include(('api.v1.urls', 'api_v1'), namespace='v1')),
    path('v2/', include(('api.v2.urls', 'api_v2'), namespace='v2')),
]

# api/v1/urls.py
from django.urls import path
from . import views

app_name = 'api_v1'

urlpatterns = [
    path('users/', views.UserListV1.as_view(), name='user_list'),
    path('users/<int:pk>/', views.UserDetailV1.as_view(), name='user_detail'),
]

# api/v2/urls.py
from django.urls import path
from . import views

app_name = 'api_v2'

urlpatterns = [
    path('users/', views.UserListV2.as_view(), name='user_list'),
    path('users/<int:pk>/', views.UserDetailV2.as_view(), name='user_detail'),
]
```

### 3. Multi-Language URLs

```python
# urls.py
from django.conf.urls.i18n import i18n_patterns
from django.urls import path, include

urlpatterns = [
    # Non-translated URLs (API, admin)
    path('api/', include('api.urls')),
    path('admin/', admin.site.urls),
]

# Translated URLs with language prefix
urlpatterns += i18n_patterns(
    path('', include('website.urls')),
    path('blog/', include('blog.urls')),
    prefix_default_language=False,
)

# URLs will be: /en/blog/, /es/blog/, /fr/blog/
```

## ✅ Best Practices

### 1. Named URLs

```python
# urls.py
urlpatterns = [
    path('users/', views.user_list, name='user_list'),
    path('users/<int:pk>/', views.user_detail, name='user_detail'),
    path('users/<int:pk>/edit/', views.user_edit, name='user_edit'),
]

# views.py
from django.shortcuts import redirect
from django.urls import reverse

def my_view(request):
    # Use reverse() instead of hardcoding URLs
    url = reverse('user_detail', kwargs={'pk': 1})
    return redirect(url)

# templates
# {% url 'user_detail' pk=user.id %}
```

### 2. URL Organization

```python
# Good organization
myproject/
    myproject/
        urls.py           # Root URLconf
    apps/
        blog/
            urls.py       # Blog URLs
            api/
                urls.py   # Blog API URLs
        users/
            urls.py       # User URLs
            api/
                urls.py   # User API URLs

# myproject/urls.py
urlpatterns = [
    path('', include('website.urls')),
    path('blog/', include('blog.urls')),
    path('users/', include('users.urls')),
    path('api/blog/', include('blog.api.urls')),
    path('api/users/', include('users.api.urls')),
]
```

## 🔐 Security Best Practices

### 1. Protected Admin URL

```python
# Don't use default /admin/
from django.conf import settings

admin_url = settings.ADMIN_URL if hasattr(settings, 'ADMIN_URL') else 'admin/'

urlpatterns = [
    path(admin_url, admin.site.urls),  # Use custom URL like 'secret-admin/'
]

# settings.py
ADMIN_URL = os.environ.get('ADMIN_URL', 'secret-admin-panel/')
```

### 2. Rate-Limited URLs

```python
from django.views.decorators.cache import cache_page

urlpatterns = [
    # Cache expensive views
    path('stats/', cache_page(60 * 15)(views.stats_view)),  # Cache 15 minutes
]
```

## ❌ Common Mistakes

### 1. Forgetting Trailing Slashes

```python
# INCONSISTENT
urlpatterns = [
    path('users', views.user_list),     # No slash
    path('posts/', views.post_list),    # With slash
]

# CORRECT - Always use trailing slash
urlpatterns = [
    path('users/', views.user_list),
    path('posts/', views.post_list),
]
```

### 2. Hardcoding URLs

```python
# WRONG
def my_view(request):
    return redirect('/users/1/')

# CORRECT
def my_view(request):
    return redirect('user_detail', pk=1)
```

## ❓ Interview Questions

### Q1: What is the difference between path() and re_path()?

**Answer**:

- **path()**: Simple pattern matching with converters (`<int:id>`)
  - Faster, easier to read
- **re_path()**: Full regex support
  - More flexible but slower

Use path() unless you need regex.

### Q2: How does Django resolve URLs?

**Answer**:

1. Load ROOT_URLCONF from settings
2. Compile all patterns
3. Match request path top-to-bottom
4. First match wins
5. Call matched view

Order matters!

### Q3: What are URL namespaces?

**Answer**:
Prevent naming conflicts:

```python
reverse('blog:list')  # Clear
reverse('products:list')  # No conflict
```

## 📚 Summary

**Key Takeaways**:

1. Use path() for simple patterns
2. Always use named URLs
3. Organize with include() and namespaces
4. Keep trailing slashes consistent
5. Order matters - specific before generic
6. Secure admin URL
7. Use caching for performance

Django URL routing is powerful and essential for clean architecture!
