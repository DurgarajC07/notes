# Django Templates

## 📖 Concept Explanation

Django templates are text files that define the structure and layout of web pages. They use Django Template Language (DTL) to generate HTML dynamically.

### Template Rendering Flow

```
View → Context → Template → HTML
```

**Process**:

1. View prepares context data (dict)
2. Template engine loads template file
3. DTL processes variables and tags
4. Returns rendered HTML string
5. Browser receives HTML

### Basic Template

```html
<!-- templates/blog/post_detail.html -->
<!DOCTYPE html>
<html>
  <head>
    <title>{{ post.title }}</title>
  </head>
  <body>
    <h1>{{ post.title }}</h1>
    <p>By {{ post.author.username }} on {{ post.created_at|date:"F j, Y" }}</p>
    <div>{{ post.content|linebreaks }}</div>
  </body>
</html>
```

### Rendering in View

```python
from django.shortcuts import render

def post_detail(request, pk):
    post = Post.objects.get(pk=pk)
    context = {'post': post}
    return render(request, 'blog/post_detail.html', context)
```

## 🧠 Template Language (DTL)

### 1. Variables

```html
<!-- Accessing variables -->
{{ variable }} {{ user.username }} {{ post.author.email }}

<!-- Dictionary access -->
{{ mydict.key }}

<!-- List access -->
{{ mylist.0 }}

<!-- Method calls (no parentheses) -->
{{ post.get_absolute_url }} {{ user.is_authenticated }}

<!-- Default value if variable is False/None/empty -->
{{ post.title|default:"No Title" }}
```

### 2. Filters

```html
<!-- String filters -->
{{ name|lower }} {{ name|upper }} {{ name|title }} {{ name|capfirst }}

<!-- Truncation -->
{{ text|truncatewords:30 }} {{ text|truncatechars:100 }}

<!-- Date formatting -->
{{ post.created_at|date:"F j, Y" }}
<!-- January 1, 2024 -->
{{ post.created_at|date:"Y-m-d" }}
<!-- 2024-01-01 -->
{{ post.created_at|timesince }}
<!-- "2 days ago" -->

<!-- Number formatting -->
{{ value|floatformat:2 }} {{ number|filesizeformat }}
<!-- "100.5 MB" -->

<!-- Lists -->
{{ my_list|join:", " }} {{ my_list|length }} {{ my_list|first }} {{ my_list|last
}}

<!-- Safe HTML (be careful!) -->
{{ html_content|safe }}
<!-- Renders HTML without escaping -->

<!-- URL encoding -->
{{ url|urlencode }}

<!-- Chaining filters -->
{{ post.title|lower|truncatewords:5 }}
```

### 3. Tags

```html
<!-- Control flow: if -->
{% if user.is_authenticated %}
<p>Welcome, {{ user.username }}!</p>
{% elif user.is_anonymous %}
<p>Please log in.</p>
{% else %}
<p>Something went wrong.</p>
{% endif %}

<!-- Comparison operators -->
{% if post.views > 1000 %}
<span>Popular!</span>
{% endif %} {% if user == post.author %}
<a href="{% url 'post_edit' post.pk %}">Edit</a>
{% endif %}

<!-- Loops: for -->
{% for post in posts %}
<h2>{{ post.title }}</h2>
<p>{{ post.content|truncatewords:50 }}</p>
{% empty %}
<p>No posts available.</p>
{% endfor %}

<!-- Loop variables -->
{% for post in posts %}
<div class="post {% if forloop.first %}first{% endif %}">
  {{ forloop.counter }}. {{ post.title }}
  <!-- forloop.counter: 1-indexed counter -->
  <!-- forloop.counter0: 0-indexed counter -->
  <!-- forloop.first: True if first iteration -->
  <!-- forloop.last: True if last iteration -->
  <!-- forloop.parentloop: Parent loop object (nested loops) -->
</div>
{% endfor %}

<!-- URL generation -->
<a href="{% url 'post_detail' post.pk %}">View Post</a>
<a href="{% url 'post_edit' pk=post.pk %}">Edit</a>

<!-- Static files -->
{% load static %}
<img src="{% static 'images/logo.png' %}" alt="Logo" />
<link rel="stylesheet" href="{% static 'css/style.css' %}" />

<!-- Comments -->
{# Single-line comment #} {% comment %} Multi-line comment Can span multiple
lines {% endcomment %}

<!-- CSRF token (required for POST forms) -->
<form method="post">
  {% csrf_token %}
  <!-- form fields -->
</form>

<!-- Include other templates -->
{% include 'blog/post_snippet.html' %} {% include 'blog/post_snippet.html' with
post=featured_post %}

<!-- Block definition (for inheritance) -->
{% block content %} Default content {% endblock %}
```

## 🏗️ Template Inheritance

### Base Template

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    {% load static %}
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>{% block title %}My Site{% endblock %}</title>

    <link rel="stylesheet" href="{% static 'css/bootstrap.min.css' %}" />
    <link rel="stylesheet" href="{% static 'css/custom.css' %}" />

    {% block extra_css %}{% endblock %}
  </head>
  <body>
    <nav class="navbar">
      <a href="{% url 'home' %}">Home</a>
      {% if user.is_authenticated %}
      <a href="{% url 'profile' %}">Profile</a>
      <a href="{% url 'logout' %}">Logout</a>
      {% else %}
      <a href="{% url 'login' %}">Login</a>
      <a href="{% url 'register' %}">Register</a>
      {% endif %}
    </nav>

    <main>
      {% if messages %}
      <div class="messages">
        {% for message in messages %}
        <div class="alert alert-{{ message.tags }}">{{ message }}</div>
        {% endfor %}
      </div>
      {% endif %} {% block content %} {% endblock %}
    </main>

    <footer>
      <p>&copy; 2024 My Site</p>
    </footer>

    <script src="{% static 'js/jquery.min.js' %}"></script>
    <script src="{% static 'js/bootstrap.min.js' %}"></script>

    {% block extra_js %}{% endblock %}
  </body>
</html>
```

### Child Template

```html
<!-- templates/blog/post_detail.html -->
{% extends 'base.html' %} {% load static %} {% block title %}{{ post.title }} -
Blog{% endblock %} {% block extra_css %}
<link rel="stylesheet" href="{% static 'css/blog.css' %}" />
{% endblock %} {% block content %}
<article class="post">
  <h1>{{ post.title }}</h1>
  <p class="meta">
    By <strong>{{ post.author.username }}</strong>
    on {{ post.created_at|date:"F j, Y" }}
  </p>

  <div class="content">{{ post.content|linebreaks|safe }}</div>

  {% if post.tags.exists %}
  <div class="tags">
    {% for tag in post.tags.all %}
    <span class="tag">{{ tag.name }}</span>
    {% endfor %}
  </div>
  {% endif %} {% if user == post.author %}
  <div class="actions">
    <a href="{% url 'post_edit' post.pk %}">Edit</a>
    <form
      method="post"
      action="{% url 'post_delete' post.pk %}"
      style="display:inline;"
    >
      {% csrf_token %}
      <button type="submit" onclick="return confirm('Are you sure?')">
        Delete
      </button>
    </form>
  </div>
  {% endif %}
</article>

<!-- Comments section -->
<section class="comments">
  <h2>Comments ({{ post.comments.count }})</h2>
  {% for comment in post.comments.all %} {% include 'blog/comment.html' %} {%
  empty %}
  <p>No comments yet.</p>
  {% endfor %}
</section>
{% endblock %} {% block extra_js %}
<script src="{% static 'js/post-detail.js' %}"></script>
{% endblock %}
```

## 🎯 Custom Template Tags & Filters

### 1. Custom Filter

```python
# blog/templatetags/blog_extras.py
from django import template
from django.utils.html import format_html
import markdown

register = template.Library()

@register.filter(name='markdown')
def markdown_format(text):
    """Convert markdown to HTML"""
    return format_html(markdown.markdown(text))

@register.filter
def split(value, arg):
    """Split string by delimiter"""
    return value.split(arg)

# Usage in template:
# {{ post.content|markdown }}
# {{ tags|split:"," }}
```

### 2. Custom Template Tag

```python
# blog/templatetags/blog_extras.py
from django import template

register = template.Library()

@register.simple_tag
def current_time(format_string):
    """Display current time"""
    from datetime import datetime
    return datetime.now().strftime(format_string)

@register.simple_tag(takes_context=True)
def user_posts_count(context):
    """Get current user's post count"""
    user = context['request'].user
    if user.is_authenticated:
        return user.posts.count()
    return 0

@register.inclusion_tag('blog/recent_posts.html')
def show_recent_posts(count=5):
    """Render recent posts widget"""
    posts = Post.objects.order_by('-created_at')[:count]
    return {'posts': posts}

# Usage:
# {% load blog_extras %}
# {% current_time "%Y-%m-%d %H:%M" %}
# {% user_posts_count %}
# {% show_recent_posts 10 %}
```

## ✅ Best Practices

### 1. Keep Logic Out of Templates

```html
<!-- BAD: Complex logic in template -->
{% if user.is_authenticated and user.posts.count > 0 and user.is_active %}
<p>Welcome back!</p>
{% endif %}

<!-- GOOD: Move logic to view/property -->
<!-- In model -->
class User(AbstractUser): @property def can_post(self): return
self.is_authenticated and self.posts.count() > 0 and self.is_active

<!-- In template -->
{% if user.can_post %}
<p>Welcome back!</p>
{% endif %}
```

### 2. Use Template Fragments

```html
<!-- templates/blog/post_snippet.html -->
<div class="post-snippet">
  <h3><a href="{% url 'post_detail' post.pk %}">{{ post.title }}</a></h3>
  <p>{{ post.content|truncatewords:30 }}</p>
</div>

<!-- Include in other templates -->
{% for post in posts %} {% include 'blog/post_snippet.html' %} {% endfor %}
```

## 🔐 Security Best Practices

### 1. Auto-Escaping

```html
<!-- Django auto-escapes by default -->
{{ user_input }}
<!-- <script>alert('XSS')</script> → escaped -->

<!-- Only use |safe if you trust the content -->
{{ trusted_html|safe }}

<!-- For user-generated content, use filters -->
{{ comment.text|linebreaks }}
<!-- Converts \n to <br>, escapes HTML -->
```

### 2. CSRF Protection

```html
<!-- Always include {% csrf_token %} in POST forms -->
<form method="post" action="{% url 'post_create' %}">
  {% csrf_token %} {{ form.as_p }}
  <button type="submit">Submit</button>
</form>
```

## ⚡ Performance Optimization

### 1. Query Optimization in Templates

```python
# views.py
def post_list(request):
    # BAD: Will cause N+1 queries
    posts = Post.objects.all()

    # GOOD: Prefetch related data
    posts = Post.objects.select_related('author') \
                        .prefetch_related('tags', 'comments') \
                        .all()

    return render(request, 'blog/post_list.html', {'posts': posts})
```

```html
<!-- template -->
{% for post in posts %}
<h2>{{ post.title }}</h2>
<p>By {{ post.author.username }}</p>
{# No extra query #}
<p>{{ post.comments.count }} comments</p>
{# No extra query #} {% endfor %}
```

### 2. Template Fragment Caching

```html
{% load cache %} {% cache 600 sidebar %}
<!-- Cached for 10 minutes -->
<div class="sidebar">{% show_recent_posts 5 %}</div>
{% endcache %}

<!-- Cache with key based on variable -->
{% cache 600 post_detail post.pk %}
<article>{{ post.content }}</article>
{% endcache %}
```

## ❓ Interview Questions

### Q1: What is the difference between {{ }} and {% %}?

**Answer**:

- `{{ variable }}`: Output variable value
- `{% tag %}`: Execute logic (if, for, url, etc.)

### Q2: How do you prevent XSS in Django templates?

**Answer**:
Django auto-escapes all variables. Avoid using `|safe` on user input. Use built-in filters like `linebreaks`, `urlize`.

### Q3: What is template inheritance?

**Answer**:
Child templates extend base template using `{% extends 'base.html' %}` and override blocks with `{% block %}`. Promotes DRY principle.

## 📚 Summary

**Key Takeaways**:

1. DTL uses {{ }} for variables, {% %} for tags
2. Filters transform data: |lower, |date, |truncatewords
3. Template inheritance reduces duplication
4. Auto-escaping prevents XSS
5. Always use {% csrf_token %} in POST forms
6. Optimize queries before rendering
7. Cache expensive template fragments
8. Keep logic in views/models, not templates

Django templates are powerful and secure by default!
