# Django XSS Prevention

## 📖 Concept Explanation

Cross-Site Scripting (XSS) is an attack where malicious scripts are injected into web pages viewed by other users. XSS exploits the trust a user has for a particular site.

### XSS Attack Types

```
1. Stored XSS (Persistent):
   - Attacker stores malicious script in database
   - Example: <script>steal_cookies()</script> in comment
   - Executes when any user views the comment

2. Reflected XSS (Non-Persistent):
   - Malicious script in URL parameter
   - Example: /search?q=<script>alert('XSS')</script>
   - Executes when victim clicks malicious link

3. DOM-Based XSS:
   - JavaScript directly manipulates DOM with user input
   - Example: document.write(location.hash)
   - Executes client-side without server involvement
```

**Impact**:

- Session hijacking (cookie theft)
- Credential theft
- Defacement
- Malware distribution
- Phishing

## 🧠 Django's Auto-Escaping

### 1. Template Auto-Escaping (Default)

```python
# views.py
def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    return render(request, 'post_detail.html', {'post': post})

# post_detail.html
<h1>{{ post.title }}</h1>  <!-- Auto-escaped -->
<div>{{ post.content }}</div>  <!-- Auto-escaped -->

# If post.title = "<script>alert('XSS')</script>"
# Rendered as: &lt;script&gt;alert(&#x27;XSS&#x27;)&lt;/script&gt;
# Displayed as: <script>alert('XSS')</script> (safe text)
```

### 2. Characters Escaped

```python
Django automatically escapes:
< → &lt;
> → &gt;
' → &#x27;
" → &quot;
& → &amp;

# Example
user_input = '<img src=x onerror="alert(\'XSS\')">'
# Rendered as: &lt;img src=x onerror=&quot;alert(&#x27;XSS&#x27;)&quot;&gt;
# Safe: No script execution
```

### 3. Safe Filter (Use with Caution!)

```django
<!-- Bypass auto-escaping -->
{{ post.content|safe }}

<!-- DANGEROUS if content contains user input! -->
<!-- Use only for trusted content -->
```

## ✅ Safe HTML Rendering

### 1. Bleach Library (Recommended)

```bash
pip install bleach
```

```python
# utils.py
import bleach

ALLOWED_TAGS = [
    'p', 'br', 'strong', 'em', 'u', 'a', 'ul', 'ol', 'li',
    'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
    'blockquote', 'code', 'pre'
]

ALLOWED_ATTRIBUTES = {
    'a': ['href', 'title', 'rel'],
    'img': ['src', 'alt', 'width', 'height'],
}

ALLOWED_PROTOCOLS = ['http', 'https', 'mailto']

def sanitize_html(content):
    """
    Clean HTML content, allowing only safe tags.
    """
    cleaned = bleach.clean(
        content,
        tags=ALLOWED_TAGS,
        attributes=ALLOWED_ATTRIBUTES,
        protocols=ALLOWED_PROTOCOLS,
        strip=True  # Remove disallowed tags instead of escaping
    )

    # Add rel="noopener noreferrer" to all links
    cleaned = bleach.linkify(
        cleaned,
        callbacks=[lambda attrs, new: attrs]
    )

    return cleaned

# models.py
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    content_html = models.TextField(blank=True)

    def save(self, *args, **kwargs):
        """Sanitize HTML before saving"""
        self.content_html = sanitize_html(self.content)
        super().save(*args, **kwargs)

# template.html
{{ post.content_html|safe }}  <!-- Now safe to use |safe -->
```

### 2. Django's mark_safe (Use Carefully)

```python
from django.utils.safestring import mark_safe

def format_text(text):
    """
    Format text with safe HTML.
    Only use when you control the HTML generation.
    """
    # You control the HTML - this is safe
    formatted = f"<strong>{escape(text)}</strong>"
    return mark_safe(formatted)

# template.html
{{ formatted_text }}  <!-- Already marked safe -->
```

### 3. Custom Template Filter

```python
# templatetags/safe_html.py
from django import template
from django.utils.safestring import mark_safe
import bleach

register = template.Library()

@register.filter(name='safe_html')
def safe_html(value):
    """
    Custom filter to safely render HTML.
    """
    if not value:
        return ''

    cleaned = bleach.clean(
        value,
        tags=['p', 'br', 'strong', 'em', 'a', 'ul', 'ol', 'li'],
        attributes={'a': ['href', 'title']},
        strip=True
    )

    return mark_safe(cleaned)

# template.html
{% load safe_html %}
{{ post.content|safe_html }}
```

## 🎯 Preventing Different XSS Types

### 1. Stored XSS Prevention

```python
# forms.py
from django import forms
import bleach

class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'content']

    def clean_title(self):
        """Strip all HTML from title"""
        title = self.cleaned_data['title']
        # Remove ALL HTML tags
        return bleach.clean(title, tags=[], strip=True)

    def clean_content(self):
        """Sanitize HTML in content"""
        content = self.cleaned_data['content']
        return bleach.clean(
            content,
            tags=['p', 'br', 'strong', 'em', 'ul', 'ol', 'li'],
            strip=True
        )

# views.py
def create_post(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            return redirect('post_detail', pk=post.pk)
    else:
        form = PostForm()

    return render(request, 'create_post.html', {'form': form})
```

### 2. Reflected XSS Prevention

```python
# views.py
from django.utils.html import escape

def search(request):
    """Search with safe query display"""
    query = request.GET.get('q', '')

    # Django templates auto-escape, but be explicit for clarity
    safe_query = escape(query)

    results = Post.objects.filter(
        Q(title__icontains=query) | Q(content__icontains=query)
    )

    return render(request, 'search.html', {
        'query': query,  # Auto-escaped in template
        'results': results
    })

# search.html
<h2>Search results for: {{ query }}</h2>  <!-- Auto-escaped -->

<!-- NEVER do this: -->
<!-- <h2>Search results for: <script>document.write(location.search)</script></h2> -->
```

### 3. DOM-Based XSS Prevention

```javascript
// Bad: Direct DOM manipulation with user input
// NEVER DO THIS!
document.getElementById("output").innerHTML = userInput;
document.write(userInput);
element.outerHTML = userInput;

// Good: Use textContent or safe methods
document.getElementById("output").textContent = userInput; // Safe

// Good: Create elements programmatically
const p = document.createElement("p");
p.textContent = userInput; // Auto-escaped
document.getElementById("output").appendChild(p);

// Good: DOMPurify for HTML (if you must)
const cleanHTML = DOMPurify.sanitize(userInput);
document.getElementById("output").innerHTML = cleanHTML;
```

## 🛡️ Content Security Policy (CSP)

### 1. CSP Headers

```python
# settings.py
MIDDLEWARE = [
    # ...
    'csp.middleware.CSPMiddleware',
]

# Install: pip install django-csp

CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", "'unsafe-inline'", "cdn.example.com")
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'", "fonts.googleapis.com")
CSP_IMG_SRC = ("'self'", "data:", "https:")
CSP_FONT_SRC = ("'self'", "fonts.gstatic.com")
CSP_CONNECT_SRC = ("'self'", "api.example.com")

# Report violations
CSP_REPORT_URI = "/csp-report/"
CSP_REPORT_ONLY = False  # True for testing, False for enforcement

# Response header:
# Content-Security-Policy: default-src 'self'; script-src 'self' cdn.example.com; ...
```

### 2. CSP Report Handler

```python
# views.py
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
import json
import logging

logger = logging.getLogger(__name__)

@csrf_exempt
def csp_report(request):
    """
    Handle CSP violation reports.
    """
    if request.method == 'POST':
        try:
            report = json.loads(request.body)
            logger.warning(f"CSP Violation: {report}")

            # Store in database for analysis
            CSPViolation.objects.create(
                document_uri=report.get('csp-report', {}).get('document-uri'),
                violated_directive=report.get('csp-report', {}).get('violated-directive'),
                blocked_uri=report.get('csp-report', {}).get('blocked-uri'),
                source_file=report.get('csp-report', {}).get('source-file'),
                line_number=report.get('csp-report', {}).get('line-number'),
            )
        except Exception as e:
            logger.error(f"Error processing CSP report: {e}")

    return JsonResponse({'status': 'ok'})
```

### 3. Nonce-Based CSP (More Secure)

```python
# middleware.py
import secrets
from django.utils.deprecation import MiddlewareMixin

class CSPNonceMiddleware(MiddlewareMixin):
    """
    Add nonce to CSP and make it available in templates.
    """
    def process_request(self, request):
        nonce = secrets.token_urlsafe(16)
        request.csp_nonce = nonce

    def process_response(self, request, response):
        if hasattr(request, 'csp_nonce'):
            nonce = request.csp_nonce
            csp = f"script-src 'self' 'nonce-{nonce}'; object-src 'none';"
            response['Content-Security-Policy'] = csp
        return response

# template.html
<script nonce="{{ request.csp_nonce }}">
    // Inline script allowed with nonce
    console.log('This is safe');
</script>
```

## 🔐 Advanced XSS Prevention

### 1. HTTP-Only Cookies

```python
# settings.py
SESSION_COOKIE_HTTPONLY = True  # JavaScript can't access session cookie
CSRF_COOKIE_HTTPONLY = False    # CSRF token needs to be readable
SESSION_COOKIE_SECURE = True     # Only over HTTPS
CSRF_COOKIE_SECURE = True        # Only over HTTPS

# Prevents: document.cookie access in XSS attacks
```

### 2. X-XSS-Protection Header

```python
# settings.py
SECURE_BROWSER_XSS_FILTER = True

# Response header:
# X-XSS-Protection: 1; mode=block
# Enables browser's built-in XSS filter
```

### 3. X-Content-Type-Options

```python
# settings.py
SECURE_CONTENT_TYPE_NOSNIFF = True

# Response header:
# X-Content-Type-Options: nosniff
# Prevents MIME-type sniffing attacks
```

### 4. JSON Escaping

```python
# views.py
from django.http import JsonResponse
from django.core.serializers.json import DjangoJSONEncoder
import json

def api_view(request):
    """JSON responses are safe from XSS"""
    data = {
        'user_input': '<script>alert("XSS")</script>',
        'description': 'User provided content'
    }

    # JsonResponse automatically escapes special characters
    return JsonResponse(data)

    # Response: {"user_input": "<script>alert(\"XSS\")</script>", ...}
    # Safe when parsed as JSON, not as HTML
```

## ✅ Best Practices

### 1. Never Trust User Input

```python
# Bad: Direct rendering of user input
{{ user.bio|safe }}  # DANGEROUS!

# Good: Sanitize first
{{ user.bio|safe_html }}  # Custom filter with bleach

# Good: Let auto-escaping handle it
{{ user.bio }}  # Auto-escaped, displays as text
```

### 2. Validate and Sanitize Input

```python
# forms.py
class ProfileForm(forms.ModelForm):
    class Meta:
        model = Profile
        fields = ['bio', 'website']

    def clean_bio(self):
        bio = self.cleaned_data['bio']
        # Sanitize HTML
        return bleach.clean(bio, tags=['p', 'br', 'strong', 'em'], strip=True)

    def clean_website(self):
        website = self.cleaned_data['website']
        # Validate URL
        if not website.startswith(('http://', 'https://')):
            raise forms.ValidationError('Invalid URL')
        return website
```

### 3. Use Templating Engine Safely

```django
<!-- Good: Auto-escaped -->
<div>{{ user_comment }}</div>

<!-- Bad: Bypass auto-escaping -->
<div>{{ user_comment|safe }}</div>

<!-- Good: Sanitized HTML -->
<div>{{ user_comment|safe_html }}</div>

<!-- Bad: JavaScript context (always dangerous) -->
<script>
    var data = "{{ user_input }}";  // Can break out of string
</script>

<!-- Good: Use JSON filter -->
<script>
    var data = {{ user_data|json_script:"user-data" }};
</script>
```

## 🧪 Testing XSS Prevention

```python
# tests.py
from django.test import TestCase

class XSSTestCase(TestCase):
    def test_xss_in_title(self):
        """XSS in title should be escaped"""
        xss_payload = '<script>alert("XSS")</script>'

        post = Post.objects.create(
            title=xss_payload,
            content='Test content',
            author=self.user
        )

        response = self.client.get(f'/posts/{post.pk}/')

        # Should be escaped in HTML
        self.assertContains(response, '&lt;script&gt;')
        self.assertNotContains(response, '<script>alert')

    def test_xss_in_search(self):
        """XSS in search query should be escaped"""
        xss_payload = '<img src=x onerror=alert("XSS")>'

        response = self.client.get(f'/search/?q={xss_payload}')

        # Should be escaped
        self.assertContains(response, '&lt;img')
        self.assertNotContains(response, '<img src=x onerror=')
```

## ❓ Interview Questions

### Q1: What's the difference between {{ var }} and {{ var|safe }}?

**Answer**:

- `{{ var }}`: Auto-escaped (safe by default)
- `{{ var|safe }}`: Bypasses escaping (dangerous if var contains user input)

Only use `|safe` for trusted, sanitized content.

### Q2: How do you safely render user-generated HTML?

**Answer**:

1. Use bleach library to whitelist safe tags/attributes
2. Sanitize on input (in form's clean method)
3. Store sanitized HTML in separate field
4. Use `|safe` only on sanitized field

### Q3: What is Content Security Policy (CSP)?

**Answer**:
HTTP header that restricts resource loading (scripts, styles, images). Prevents XSS by blocking inline scripts and external resources from untrusted domains.

## 📚 Summary

**Key Takeaways**:

1. Django auto-escapes templates by default
2. Never use `|safe` with user input
3. Use bleach to sanitize HTML
4. Implement Content Security Policy (CSP)
5. Set HTTPONLY on session cookies
6. Enable X-XSS-Protection header
7. Validate and sanitize all user input
8. Never insert user data in JavaScript context
9. Use textContent instead of innerHTML in JS
10. Test for XSS vulnerabilities

XSS prevention is a fundamental security requirement!
