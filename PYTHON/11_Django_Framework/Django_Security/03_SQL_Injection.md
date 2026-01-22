# Django SQL Injection Prevention

## 📖 Concept Explanation

SQL Injection is an attack where malicious SQL code is inserted into application queries, allowing attackers to manipulate database operations.

### SQL Injection Attack Example

```sql
-- Vulnerable query (DO NOT USE!)
query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"

-- If attacker enters username: admin' OR '1'='1
-- Query becomes:
SELECT * FROM users WHERE username = 'admin' OR '1'='1' AND password = 'anything'
-- Returns all users (bypasses authentication!)

-- If attacker enters: '; DROP TABLE users; --
-- Query becomes:
SELECT * FROM users WHERE username = ''; DROP TABLE users; --' AND password = ''
-- Deletes entire users table!
```

**Impact**:

- Data theft (dump entire database)
- Data modification/deletion
- Authentication bypass
- Privilege escalation
- Server compromise

## 🧠 Django ORM Protection (Safe by Default)

### 1. Parameterized Queries

```python
# Good: Django ORM uses parameterized queries
username = request.POST.get('username')
password = request.POST.get('password')

# Safe: Parameters are properly escaped
user = User.objects.filter(username=username, password=password).first()

# Generated SQL (parameterized):
# SELECT * FROM users WHERE username = %s AND password = %s
# Parameters: ['admin', 'password123']
# Database driver handles escaping
```

### 2. QuerySet API (Always Safe)

```python
# All QuerySet methods are safe from SQL injection

# Filter
Post.objects.filter(title=user_input)
# Safe: SELECT * FROM posts WHERE title = %s

# Exclude
Post.objects.exclude(author__username=user_input)
# Safe: SELECT * FROM posts WHERE NOT author.username = %s

# Get
Post.objects.get(pk=user_input)
# Safe: SELECT * FROM posts WHERE id = %s

# Complex queries with Q objects
from django.db.models import Q
Post.objects.filter(
    Q(title__icontains=search_term) | Q(content__icontains=search_term)
)
# Safe: Parameterized OR query

# Aggregation
from django.db.models import Count
Post.objects.filter(author__username=username).aggregate(Count('id'))
# Safe: Parameterized aggregation
```

### 3. F() Expressions (Safe)

```python
from django.db.models import F

# Safe: F() expressions use SQL identifiers, not parameters
Post.objects.filter(views__gt=F('likes'))
# Safe: SELECT * FROM posts WHERE views > likes

# Increment field
Post.objects.filter(pk=post_id).update(views=F('views') + 1)
# Safe: UPDATE posts SET views = views + 1 WHERE id = %s
```

## ⚠️ Dangerous Patterns to Avoid

### 1. String Formatting in Queries (NEVER DO THIS!)

```python
# DANGEROUS: String interpolation
username = request.GET.get('username')
query = f"SELECT * FROM users WHERE username = '{username}'"
# Vulnerable to SQL injection!

# DANGEROUS: % formatting
query = "SELECT * FROM users WHERE username = '%s'" % username
# Still vulnerable!

# DANGEROUS: .format()
query = "SELECT * FROM users WHERE username = '{}'".format(username)
# Still vulnerable!
```

### 2. Raw SQL (Use Carefully)

```python
# DANGEROUS: Raw SQL with string formatting
username = request.GET.get('username')
query = f"SELECT * FROM users WHERE username = '{username}'"
users = User.objects.raw(query)  # VULNERABLE!

# SAFE: Raw SQL with parameterization
username = request.GET.get('username')
query = "SELECT * FROM users WHERE username = %s"
users = User.objects.raw(query, [username])  # Safe: parameterized
```

### 3. extra() Method (Deprecated, Avoid)

```python
# DANGEROUS: extra() with string formatting
search = request.GET.get('search')
Post.objects.extra(
    where=[f"title LIKE '%{search}%'"]  # VULNERABLE!
)

# SAFER: extra() with params (but use ORM instead)
Post.objects.extra(
    where=["title LIKE %s"],
    params=[f'%{search}%']
)

# BEST: Use ORM methods
Post.objects.filter(title__icontains=search)  # Safe and cleaner
```

## ✅ Safe Practices

### 1. Use Django ORM

```python
# Always prefer ORM over raw SQL

# Good: ORM filter
posts = Post.objects.filter(
    author__username=username,
    published=True,
    created_at__gte=start_date
)

# Avoid: Raw SQL (unless absolutely necessary)
cursor.execute("SELECT * FROM posts WHERE ...")
```

### 2. Parameterized Raw Queries

```python
from django.db import connection

def get_posts_by_author_raw(username):
    """Raw SQL with proper parameterization"""

    # Safe: Use %s placeholders and pass params separately
    with connection.cursor() as cursor:
        cursor.execute(
            "SELECT * FROM posts WHERE author_id IN "
            "(SELECT id FROM users WHERE username = %s)",
            [username]  # Parameters passed separately
        )
        rows = cursor.fetchall()

    return rows

# NEVER do this:
# cursor.execute(f"SELECT * FROM posts WHERE author = '{username}'")
```

### 3. Validate Input

```python
# forms.py
from django import forms

class SearchForm(forms.Form):
    query = forms.CharField(max_length=200, required=True)
    category = forms.ChoiceField(choices=[
        ('tech', 'Technology'),
        ('science', 'Science'),
        ('news', 'News')
    ])

    def clean_query(self):
        query = self.cleaned_data['query']

        # Validate input
        if len(query) < 3:
            raise forms.ValidationError('Query too short')

        # Remove potentially dangerous characters (defense in depth)
        # Though Django ORM handles this, validation is good practice

        return query

# views.py
def search(request):
    form = SearchForm(request.GET)

    if form.is_valid():
        query = form.cleaned_data['query']
        category = form.cleaned_data['category']

        # Safe: Django ORM with validated input
        results = Post.objects.filter(
            title__icontains=query,
            category=category
        )

    return render(request, 'search.html', {'results': results})
```

## 🎯 Advanced Safe Patterns

### 1. Dynamic Field Names (Whitelist Approach)

```python
def get_posts_ordered(order_by):
    """Safe dynamic ordering with whitelist"""

    # Whitelist allowed fields
    ALLOWED_FIELDS = ['created_at', 'views', 'title', 'author__username']

    # Validate order_by parameter
    if order_by not in ALLOWED_FIELDS:
        order_by = 'created_at'  # Default

    # Safe: Field name from whitelist
    return Post.objects.order_by(order_by)

# NEVER do this:
# order_by = request.GET.get('order_by')
# Post.objects.order_by(order_by)  # User could inject SQL!
```

### 2. Dynamic Table Names (Avoid if Possible)

```python
from django.apps import apps

def get_model_count(model_name):
    """Safe dynamic model access with whitelist"""

    # Whitelist allowed models
    ALLOWED_MODELS = ['Post', 'Comment', 'User']

    if model_name not in ALLOWED_MODELS:
        raise ValueError('Invalid model')

    # Safe: Get model from Django registry
    model = apps.get_model('myapp', model_name)
    return model.objects.count()

# NEVER construct table names from user input:
# table_name = f"myapp_{request.GET.get('table')}"
```

### 3. Bulk Operations

```python
# Safe: Bulk create with ORM
posts_to_create = [
    Post(title=title, content=content, author=user)
    for title, content in user_data
]
Post.objects.bulk_create(posts_to_create)
# Safe: All parameters properly escaped

# Safe: Bulk update
posts = Post.objects.filter(author=user)
for post in posts:
    post.views += 1
Post.objects.bulk_update(posts, ['views'])
# Safe: Parameters properly escaped
```

## 🛡️ Defense in Depth

### 1. Least Privilege Database User

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'django_app',  # Limited privileges
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': 'localhost',
        'PORT': '5432',
        'OPTIONS': {
            'options': '-c default_transaction_read_only=on'  # Read-only by default
        }
    }
}

# Database user should only have:
# - SELECT on required tables
# - INSERT, UPDATE, DELETE on specific tables
# - NO DROP, CREATE, ALTER permissions
# - NO access to system tables
```

### 2. Database Activity Monitoring

```python
# middleware.py
import logging
from django.db import connection

logger = logging.getLogger('sql')

class SQLMonitoringMiddleware:
    """Log suspicious SQL patterns"""

    SUSPICIOUS_PATTERNS = [
        'DROP ', 'DELETE FROM', 'TRUNCATE', '-- ',
        'UNION SELECT', 'OR 1=1', "OR '1'='1",
        'EXEC ', 'EXECUTE ', 'xp_'
    ]

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)

        # Check executed queries
        for query in connection.queries:
            sql = query['sql'].upper()

            for pattern in self.SUSPICIOUS_PATTERNS:
                if pattern.upper() in sql:
                    logger.warning(
                        f"Suspicious SQL detected: {sql[:200]} "
                        f"Path: {request.path} User: {request.user}"
                    )

        return response
```

### 3. Input Validation Layers

```python
# 1. Form validation
class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'content', 'category']

    def clean_title(self):
        title = self.cleaned_data['title']
        # Validate length, characters, etc.
        return title

# 2. Model validation
class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()

    def clean(self):
        # Additional validation
        if 'script' in self.title.lower():
            raise ValidationError('Invalid title')

    def save(self, *args, **kwargs):
        self.full_clean()  # Run validation
        super().save(*args, **kwargs)

# 3. View validation
@require_http_methods(['POST'])
def create_post(request):
    form = PostForm(request.POST)

    if form.is_valid():
        # Additional checks
        if not request.user.has_perm('myapp.add_post'):
            return HttpResponseForbidden()

        post = form.save(commit=False)
        post.author = request.user
        post.save()

        return redirect('post_detail', pk=post.pk)

    return render(request, 'create_post.html', {'form': form})
```

## 🧪 Testing SQL Injection Resistance

```python
# tests.py
from django.test import TestCase

class SQLInjectionTestCase(TestCase):
    def test_username_sql_injection(self):
        """Test SQL injection in username field"""

        # SQL injection payloads
        payloads = [
            "admin' OR '1'='1",
            "admin'; DROP TABLE users; --",
            "admin' UNION SELECT * FROM users --",
            "admin' AND 1=1 --",
        ]

        for payload in payloads:
            # Should not cause SQL injection
            user = User.objects.filter(username=payload).first()

            # Should return None (no user found)
            self.assertIsNone(user)

            # Database should still be intact
            self.assertTrue(User.objects.exists())

    def test_search_sql_injection(self):
        """Test SQL injection in search"""

        payload = "test' OR '1'='1"

        # Should not return all posts
        results = Post.objects.filter(title__icontains=payload)

        # Should only match literal string
        self.assertEqual(results.count(), 0)
```

## 🔐 Security Checklist

### 1. Code Review Checklist

```python
# ✅ Safe patterns:
Post.objects.filter(title=user_input)
User.objects.raw("SELECT * FROM users WHERE id = %s", [user_id])
Post.objects.filter(Q(title__contains=term) | Q(content__contains=term))

# ❌ Dangerous patterns:
f"SELECT * FROM posts WHERE title = '{user_input}'"
Post.objects.raw(f"SELECT * FROM posts WHERE id = {post_id}")
Post.objects.extra(where=[f"title LIKE '%{search}%'"])
```

### 2. Database Configuration

```python
# settings.py (production)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),  # Limited privileges
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST'),
        'PORT': os.environ.get('DB_PORT', '5432'),
        'OPTIONS': {
            'sslmode': 'require',  # Encrypted connection
        },
        'CONN_MAX_AGE': 600,
    }
}
```

## ❓ Interview Questions

### Q1: Why is Django ORM safe from SQL injection?

**Answer**:
Django ORM uses parameterized queries. User input is passed as parameters (not concatenated into SQL string), and the database driver handles escaping. The SQL structure is fixed, and user input can't change it.

### Q2: When would you use raw SQL, and how do you make it safe?

**Answer**:
Use raw SQL only for complex queries that ORM can't express efficiently. Make it safe by:

1. Use parameterized queries with `%s` placeholders
2. Pass parameters separately in list/tuple
3. Never use string formatting (f-strings, %, .format())
4. Whitelist dynamic parts (field names, table names)

### Q3: What's wrong with using .extra() in Django?

**Answer**:
`.extra()` is deprecated and error-prone. Easy to introduce SQL injection if not careful with string formatting. Modern Django provides better alternatives: F() expressions, Q() objects, annotate(), aggregate(), subqueries.

## 📚 Summary

**Key Takeaways**:

1. Django ORM uses parameterized queries (safe by default)
2. Never use string formatting in SQL queries
3. Use `raw()` with parameters: `[param1, param2]`
4. Whitelist dynamic field/table names
5. Prefer ORM over raw SQL
6. Validate and sanitize all user input
7. Use least privilege database user
8. Monitor for suspicious SQL patterns
9. Test with SQL injection payloads
10. Avoid deprecated `.extra()` method

Django ORM is your best defense against SQL injection!
