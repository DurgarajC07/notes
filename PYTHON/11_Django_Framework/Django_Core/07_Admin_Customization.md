# Django Admin Customization

## 📖 Concept Explanation

Django Admin is an automatic admin interface for managing your application data. It reads metadata from your models to provide a production-ready interface for CRUD operations.

### Admin Site Architecture

```
Model → ModelAdmin → Admin Site → Web Interface
```

**Process**:

1. Register model with admin site
2. Define ModelAdmin class for customization
3. Django generates admin interface
4. Staff users access via /admin/

### Basic Admin Registration

```python
from django.contrib import admin
from .models import Post

# Simple registration
admin.site.register(Post)

# With customization
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'author', 'created_at', 'published']
    list_filter = ['published', 'created_at']
    search_fields = ['title', 'content']
```

## 🧠 ModelAdmin Options

### 1. List Display Customization

```python
from django.contrib import admin
from django.utils.html import format_html
from .models import Post

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # Columns to display in list view
    list_display = [
        'title',
        'author',
        'status_badge',
        'view_count',
        'created_at',
        'is_published'
    ]

    # Make fields clickable links
    list_display_links = ['title']

    # Enable inline editing
    list_editable = ['is_published']

    # Items per page
    list_per_page = 50

    # Custom method for list display
    @admin.display(description='Status', ordering='published')
    def status_badge(self, obj):
        """Display colored status badge"""
        if obj.published:
            color = 'green'
            text = 'Published'
        else:
            color = 'orange'
            text = 'Draft'

        return format_html(
            '<span style="background: {}; color: white; padding: 3px 10px; border-radius: 3px;">{}</span>',
            color, text
        )

    @admin.display(description='Views')
    def view_count(self, obj):
        """Display formatted view count"""
        return f"{obj.views:,}"
```

### 2. Filtering & Search

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # Sidebar filters
    list_filter = [
        'published',
        'created_at',
        'author',
        ('created_at', admin.DateFieldListFilter),  # Date hierarchy
    ]

    # Search functionality
    search_fields = [
        'title',
        'content',
        'author__username',  # Search in related model
        'tags__name',
    ]

    # Search help text
    search_help_text = "Search by title, content, author, or tags"

    # Date hierarchy navigation
    date_hierarchy = 'created_at'
```

### 3. Form Layout

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # Fieldsets for organized form
    fieldsets = [
        ('Basic Information', {
            'fields': ['title', 'slug', 'author']
        }),
        ('Content', {
            'fields': ['content', 'excerpt'],
            'description': 'Main post content and summary'
        }),
        ('Media', {
            'fields': ['featured_image', 'video_url'],
            'classes': ['collapse']  # Collapsible section
        }),
        ('Metadata', {
            'fields': ['tags', 'category', 'published', 'featured'],
            'classes': ['wide']  # Wider fieldset
        }),
        ('Advanced Options', {
            'fields': ['allow_comments', 'seo_title', 'seo_description'],
            'classes': ['collapse']
        }),
    ]

    # Alternative: simple fields list
    # fields = ['title', 'slug', 'content']

    # Prepopulated fields (e.g., slug from title)
    prepopulated_fields = {'slug': ['title']}

    # Read-only fields
    readonly_fields = ['created_at', 'updated_at', 'view_count', 'display_thumbnail']

    # Custom read-only field
    @admin.display(description='Thumbnail')
    def display_thumbnail(self, obj):
        if obj.featured_image:
            return format_html(
                '<img src="{}" style="max-height: 100px;" />',
                obj.featured_image.url
            )
        return "No image"

    # Autocomplete for foreign keys
    autocomplete_fields = ['author', 'category']

    # Filter for many-to-many
    filter_horizontal = ['tags']  # or filter_vertical
```

## 🏗️ Inline Models

### 1. Tabular Inline

```python
from django.contrib import admin
from .models import Post, Comment

class CommentInline(admin.TabularInline):
    """Display comments as table rows"""
    model = Comment
    extra = 1  # Number of empty forms
    fields = ['author', 'content', 'approved']
    readonly_fields = ['created_at']

    # Show only approved comments by default
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        return qs.select_related('author')

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    inlines = [CommentInline]
    list_display = ['title', 'author', 'comment_count']

    @admin.display(description='Comments')
    def comment_count(self, obj):
        return obj.comments.count()
```

### 2. Stacked Inline

```python
class PostMetaInline(admin.StackedInline):
    """Display related model as stacked form"""
    model = PostMeta
    extra = 0
    fieldsets = [
        ('SEO', {
            'fields': ['seo_title', 'seo_description', 'seo_keywords']
        }),
        ('Social Media', {
            'fields': ['og_title', 'og_description', 'og_image']
        }),
    ]

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    inlines = [PostMetaInline, CommentInline]
```

## 🎯 Advanced Customization

### 1. Custom Actions

```python
from django.contrib import messages

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    actions = ['make_published', 'make_draft', 'delete_selected']

    @admin.action(description='Publish selected posts')
    def make_published(self, request, queryset):
        """Bulk publish posts"""
        updated = queryset.update(published=True)
        self.message_user(
            request,
            f'{updated} post(s) successfully published.',
            messages.SUCCESS
        )

    @admin.action(description='Mark as draft')
    def make_draft(self, request, queryset):
        """Bulk unpublish posts"""
        updated = queryset.update(published=False)
        self.message_user(request, f'{updated} post(s) marked as draft.')

    # Disable delete action
    def get_actions(self, request):
        actions = super().get_actions(request)
        if not request.user.is_superuser:
            if 'delete_selected' in actions:
                del actions['delete_selected']
        return actions
```

### 2. Custom Admin Views

```python
from django.urls import path
from django.shortcuts import render
from django.http import HttpResponseRedirect

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # Add custom view
    def get_urls(self):
        urls = super().get_urls()
        custom_urls = [
            path('import/', self.import_posts, name='post_import'),
            path('export/', self.export_posts, name='post_export'),
        ]
        return custom_urls + urls

    def import_posts(self, request):
        """Custom import view"""
        if request.method == 'POST':
            # Handle file upload
            csv_file = request.FILES['csv_file']
            # Process import...
            self.message_user(request, 'Posts imported successfully')
            return HttpResponseRedirect('../')

        context = dict(
            self.admin_site.each_context(request),
            title='Import Posts',
        )
        return render(request, 'admin/post_import.html', context)

    def export_posts(self, request):
        """Custom export view"""
        import csv
        from django.http import HttpResponse

        response = HttpResponse(content_type='text/csv')
        response['Content-Disposition'] = 'attachment; filename="posts.csv"'

        writer = csv.writer(response)
        writer.writerow(['Title', 'Author', 'Created', 'Published'])

        for post in Post.objects.all():
            writer.writerow([
                post.title,
                post.author.username,
                post.created_at,
                post.published
            ])

        return response
```

### 3. Permissions & Authorization

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    def has_add_permission(self, request):
        """Only superusers can add posts"""
        return request.user.is_superuser

    def has_change_permission(self, request, obj=None):
        """Users can only edit their own posts"""
        if obj is not None and not request.user.is_superuser:
            return obj.author == request.user
        return True

    def has_delete_permission(self, request, obj=None):
        """Only superusers can delete"""
        return request.user.is_superuser

    def get_queryset(self, request):
        """Show only user's posts (non-superusers)"""
        qs = super().get_queryset(request)
        if request.user.is_superuser:
            return qs
        return qs.filter(author=request.user)

    def save_model(self, request, obj, form, change):
        """Set author on creation"""
        if not change:  # New object
            obj.author = request.user
        super().save_model(request, obj, form, change)
```

### 4. Custom Filters

```python
from django.contrib.admin import SimpleListFilter

class PublishedFilter(SimpleListFilter):
    """Custom filter for published status"""
    title = 'publication status'
    parameter_name = 'status'

    def lookups(self, request, model_admin):
        return [
            ('published', 'Published'),
            ('draft', 'Draft'),
            ('featured', 'Featured'),
        ]

    def queryset(self, request, queryset):
        if self.value() == 'published':
            return queryset.filter(published=True, featured=False)
        elif self.value() == 'draft':
            return queryset.filter(published=False)
        elif self.value() == 'featured':
            return queryset.filter(published=True, featured=True)
        return queryset

class ViewCountFilter(SimpleListFilter):
    """Filter by view count ranges"""
    title = 'popularity'
    parameter_name = 'views'

    def lookups(self, request, model_admin):
        return [
            ('low', 'Low (< 100)'),
            ('medium', 'Medium (100-1000)'),
            ('high', 'High (> 1000)'),
        ]

    def queryset(self, request, queryset):
        if self.value() == 'low':
            return queryset.filter(views__lt=100)
        elif self.value() == 'medium':
            return queryset.filter(views__gte=100, views__lte=1000)
        elif self.value() == 'high':
            return queryset.filter(views__gt=1000)

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_filter = [PublishedFilter, ViewCountFilter, 'created_at']
```

## ✅ Best Practices

### 1. Optimize Admin Queries

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_select_related = ['author', 'category']  # Reduce N+1 queries

    def get_queryset(self, request):
        """Optimize queryset"""
        qs = super().get_queryset(request)
        return qs.select_related('author', 'category') \
                 .prefetch_related('tags', 'comments') \
                 .annotate(comment_count=Count('comments'))

    @admin.display(description='Comments', ordering='comment_count')
    def display_comment_count(self, obj):
        return obj.comment_count
```

### 2. Custom Admin Templates

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # Override templates
    change_form_template = 'admin/post_change_form.html'
    change_list_template = 'admin/post_change_list.html'
```

```html
<!-- templates/admin/post_change_form.html -->
{% extends "admin/change_form.html" %} {% block after_field_sets %}
<div style="margin-top: 20px;">
  <h2>Post Statistics</h2>
  <p>Views: {{ original.views }}</p>
  <p>Comments: {{ original.comments.count }}</p>
</div>
{% endblock %}
```

## 🔐 Security Best Practices

### 1. Secure Admin URL

```python
# urls.py
from django.contrib import admin
from django.urls import path

# Change default /admin/ to custom URL
urlpatterns = [
    path(os.environ.get('ADMIN_URL', 'secret-admin/'), admin.site.urls),
]
```

### 2. Two-Factor Authentication

```python
# Install: pip install django-otp django-otp[qr]
# settings.py
INSTALLED_APPS = [
    'django_otp',
    'django_otp.plugins.otp_totp',
]

MIDDLEWARE = [
    'django_otp.middleware.OTPMiddleware',
]

# admin.py
from django_otp.admin import OTPAdminSite
admin.site.__class__ = OTPAdminSite
```

## ⚡ Performance Optimization

### 1. Limit Queryset Size

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_per_page = 25
    list_max_show_all = 200

    show_full_result_count = False  # Don't count total (slow for large tables)
```

## ❓ Interview Questions

### Q1: How do you customize the Django admin?

**Answer**:
Use `ModelAdmin` class with options like:

- `list_display` for columns
- `list_filter` for sidebar filters
- `search_fields` for search
- `fieldsets` for form layout
- `inlines` for related models

### Q2: What is the purpose of list_select_related?

**Answer**:
Optimizes queries by using `select_related()` to reduce N+1 queries when displaying foreign key relationships in list view.

### Q3: How do you add custom actions in Django admin?

**Answer**:
Define method with `@admin.action` decorator, add to `actions` list:

```python
@admin.action(description='Publish posts')
def make_published(self, request, queryset):
    queryset.update(published=True)
```

## 📚 Summary

**Key Takeaways**:

1. Django admin auto-generates interface from models
2. ModelAdmin customizes display, filters, search
3. Inlines show related models (TabularInline, StackedInline)
4. Custom actions enable bulk operations
5. Optimize with list_select_related, prefetch_related
6. Secure admin URL with environment variable
7. Custom filters with SimpleListFilter
8. Control permissions with has\_\*\_permission methods

Django admin is powerful for data management!
