# Django CSRF Protection

## 📖 Concept Explanation

Cross-Site Request Forgery (CSRF) is an attack where a malicious site tricks a user's browser into making unwanted requests to your site using the user's authenticated session.

### CSRF Attack Flow

```
1. User logs into yoursite.com (gets session cookie)
2. User visits malicious.com (while still logged in)
3. malicious.com sends request to yoursite.com/delete-account
4. Browser includes session cookie automatically
5. yoursite.com processes request as legitimate user
```

**Django's CSRF Protection**:

- Generates unique token for each session
- Token required for state-changing requests (POST, PUT, DELETE)
- Validates token on server before processing

## 🧠 How CSRF Protection Works

### 1. CSRF Middleware

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',  # CSRF protection
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    # ...
]

# CSRF settings
CSRF_COOKIE_HTTPONLY = False  # JavaScript needs to read it
CSRF_COOKIE_SECURE = True     # Only send over HTTPS
CSRF_COOKIE_SAMESITE = 'Strict'  # Prevent cross-site requests
CSRF_COOKIE_AGE = 31449600    # 1 year
CSRF_COOKIE_NAME = 'csrftoken'
CSRF_HEADER_NAME = 'HTTP_X_CSRFTOKEN'
CSRF_TRUSTED_ORIGINS = [
    'https://yourdomain.com',
    'https://*.yourdomain.com',
]
```

### 2. Template Usage

```html
<!-- forms.html -->
<!DOCTYPE html>
<html>
  <body>
    <form method="post" action="/submit/">
      {% csrf_token %}
      <!-- Adds hidden CSRF token field -->
      <input type="text" name="username" />
      <button type="submit">Submit</button>
    </form>
  </body>
</html>

<!-- Rendered HTML -->
<form method="post" action="/submit/">
  <input type="hidden" name="csrfmiddlewaretoken" value="abc123xyz789..." />
  <input type="text" name="username" />
  <button type="submit">Submit</button>
</form>
```

### 3. AJAX Requests

```javascript
// Get CSRF token from cookie
function getCookie(name) {
  let cookieValue = null;
  if (document.cookie && document.cookie !== "") {
    const cookies = document.cookie.split(";");
    for (let i = 0; i < cookies.length; i++) {
      const cookie = cookies[i].trim();
      if (cookie.substring(0, name.length + 1) === name + "=") {
        cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
        break;
      }
    }
  }
  return cookieValue;
}

const csrftoken = getCookie("csrftoken");

// jQuery AJAX
$.ajax({
  url: "/api/posts/",
  type: "POST",
  headers: {
    "X-CSRFToken": csrftoken,
  },
  data: {
    title: "My Post",
    content: "Content here",
  },
  success: function (response) {
    console.log("Success!");
  },
});

// Fetch API
fetch("/api/posts/", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-CSRFToken": csrftoken,
  },
  body: JSON.stringify({
    title: "My Post",
    content: "Content here",
  }),
})
  .then((response) => response.json())
  .then((data) => console.log(data));

// Axios (automatically includes CSRF token if configured)
axios.defaults.xsrfCookieName = "csrftoken";
axios.defaults.xsrfHeaderName = "X-CSRFToken";

axios.post("/api/posts/", {
  title: "My Post",
  content: "Content here",
});
```

## 🎯 Exempting Views from CSRF

### 1. @csrf_exempt Decorator

```python
from django.views.decorators.csrf import csrf_exempt
from django.http import JsonResponse

@csrf_exempt
def webhook_endpoint(request):
    """
    Exempt external webhooks from CSRF.
    Use with caution! Implement alternative authentication.
    """
    if request.method == 'POST':
        # Verify webhook signature instead
        signature = request.headers.get('X-Webhook-Signature')
        if not verify_webhook_signature(request.body, signature):
            return JsonResponse({'error': 'Invalid signature'}, status=401)

        # Process webhook
        data = json.loads(request.body)
        process_webhook(data)

        return JsonResponse({'status': 'success'})

    return JsonResponse({'error': 'Method not allowed'}, status=405)

# Class-based view
from django.utils.decorators import method_decorator

@method_decorator(csrf_exempt, name='dispatch')
class WebhookView(View):
    def post(self, request):
        # Webhook handling
        return JsonResponse({'status': 'success'})
```

### 2. @csrf_protect Decorator

```python
from django.views.decorators.csrf import csrf_protect

@csrf_protect
def protected_view(request):
    """
    Explicitly require CSRF protection.
    Useful when CSRF middleware is disabled globally.
    """
    if request.method == 'POST':
        # Process form
        return JsonResponse({'status': 'success'})

    return render(request, 'form.html')
```

### 3. @requires_csrf_token Decorator

```python
from django.views.decorators.csrf import requires_csrf_token

@requires_csrf_token
def custom_error_view(request):
    """
    Ensure CSRF token is available in template.
    Used for error handlers.
    """
    return render(request, 'error.html', status=500)
```

## 🏗️ Django REST Framework CSRF

### 1. SessionAuthentication with CSRF

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ]
}

# views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated

class PostViewSet(viewsets.ModelViewSet):
    """
    SessionAuthentication requires CSRF token for POST/PUT/DELETE.
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    authentication_classes = [SessionAuthentication]
    permission_classes = [IsAuthenticated]

# AJAX request must include CSRF token
fetch('/api/posts/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': getCookie('csrftoken')
    },
    credentials: 'include',  // Include cookies
    body: JSON.stringify({title: 'Test', content: 'Content'})
});
```

### 2. Token Authentication (No CSRF Needed)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ]
}

# Token auth doesn't use cookies, so CSRF not needed
# Request with Authorization header instead
fetch('/api/posts/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Token abc123xyz789'
    },
    body: JSON.stringify({title: 'Test', content: 'Content'})
});
```

### 3. Mixed Authentication with CSRF

```python
from rest_framework.authentication import SessionAuthentication

class CsrfExemptSessionAuthentication(SessionAuthentication):
    """
    SessionAuthentication without CSRF for API.
    Use only if you have alternative protection (e.g., JWT).
    """
    def enforce_csrf(self, request):
        return  # Skip CSRF check

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    authentication_classes = [
        CsrfExemptSessionAuthentication,
        TokenAuthentication
    ]
```

## ✅ Best Practices

### 1. Always Use CSRF for Session-Based Auth

```python
# Good: CSRF enabled for session auth
MIDDLEWARE = [
    # ...
    'django.middleware.csrf.CsrfViewMiddleware',
    # ...
]

# Bad: Disabling CSRF globally
# DON'T DO THIS!
# MIDDLEWARE = [...]  # Without CsrfViewMiddleware
```

### 2. HTTPS-Only CSRF Cookies

```python
# settings.py (production)
CSRF_COOKIE_SECURE = True  # Only send over HTTPS
SESSION_COOKIE_SECURE = True
SECURE_SSL_REDIRECT = True
```

### 3. SameSite Cookie Attribute

```python
# settings.py
CSRF_COOKIE_SAMESITE = 'Strict'  # or 'Lax'
SESSION_COOKIE_SAMESITE = 'Strict'

# Strict: Never send in cross-site requests
# Lax: Send on top-level navigation (GET)
# None: Send in all contexts (requires Secure)
```

### 4. Trusted Origins for CORS

```python
# settings.py
CSRF_TRUSTED_ORIGINS = [
    'https://yourdomain.com',
    'https://www.yourdomain.com',
    'https://api.yourdomain.com',
    'https://*.yourdomain.com',  # Wildcard subdomain
]

# For local development
if DEBUG:
    CSRF_TRUSTED_ORIGINS += [
        'http://localhost:3000',
        'http://127.0.0.1:3000',
    ]
```

## 🔐 Advanced CSRF Protection

### 1. Custom CSRF Failure View

```python
# views.py
from django.views.decorators.csrf import requires_csrf_token
from django.http import JsonResponse

@requires_csrf_token
def csrf_failure(request, reason=""):
    """
    Custom CSRF failure handler.
    """
    return JsonResponse({
        'error': 'CSRF verification failed',
        'reason': reason,
        'detail': 'CSRF token missing or incorrect'
    }, status=403)

# settings.py
CSRF_FAILURE_VIEW = 'myapp.views.csrf_failure'
```

### 2. Rotating CSRF Tokens

```python
from django.middleware.csrf import rotate_token

def sensitive_operation(request):
    """
    Rotate CSRF token after sensitive operation.
    Prevents token fixation attacks.
    """
    if request.method == 'POST':
        # Process sensitive operation
        change_password(request.user, request.POST['new_password'])

        # Rotate CSRF token
        rotate_token(request)

        return JsonResponse({'status': 'success'})

    return render(request, 'change_password.html')
```

### 3. CSRF Token in Meta Tag

```html
<!-- base.html -->
<html>
  <head>
    <meta name="csrf-token" content="{{ csrf_token }}" />
  </head>
  <body>
    <!-- Content -->

    <script>
      // Get CSRF token from meta tag
      const csrftoken = document.querySelector(
        'meta[name="csrf-token"]',
      ).content;

      // Use in AJAX requests
      fetch("/api/endpoint/", {
        method: "POST",
        headers: {
          "X-CSRFToken": csrftoken,
        },
        body: formData,
      });
    </script>
  </body>
</html>
```

## 🛡️ Security Considerations

### 1. Don't Disable CSRF Without Good Reason

```python
# Bad: Disabling CSRF for convenience
@csrf_exempt
def my_view(request):
    # This removes important security!
    pass

# Good: Use proper authentication instead
from rest_framework.authentication import TokenAuthentication

class MyAPIView(APIView):
    authentication_classes = [TokenAuthentication]
    # Token auth doesn't need CSRF
```

### 2. Validate Referer Header (Defense in Depth)

```python
# settings.py
# CSRF middleware checks Referer/Origin headers
# Ensure CSRF_TRUSTED_ORIGINS is properly configured

CSRF_TRUSTED_ORIGINS = [
    'https://yourdomain.com'
]

# Middleware validates:
# 1. CSRF token matches
# 2. Referer/Origin header matches trusted origins
```

### 3. Use SameSite Cookies

```python
# settings.py
CSRF_COOKIE_SAMESITE = 'Strict'
SESSION_COOKIE_SAMESITE = 'Strict'

# Strict: Maximum protection
# - Blocks all cross-site requests
# - May break legitimate cross-site navigation

# Lax: Balanced protection
# - Allows top-level navigation (GET)
# - Blocks cross-site POST/PUT/DELETE

# None: No protection (requires Secure flag)
# - Use only if you need cross-site requests
# - Must be HTTPS
```

## ⚡ Performance Optimization

### 1. Cache CSRF Token

```javascript
// Cache CSRF token to avoid repeated cookie lookups
let csrfToken = null;

function getCSRFToken() {
  if (csrfToken === null) {
    csrfToken = getCookie("csrftoken");
  }
  return csrfToken;
}

// Use cached token
fetch("/api/posts/", {
  method: "POST",
  headers: {
    "X-CSRFToken": getCSRFToken(),
  },
  body: JSON.stringify(data),
});
```

### 2. Single-Page Application (SPA) Pattern

```javascript
// Get CSRF token once on page load
const csrftoken = getCookie("csrftoken");

// Configure axios globally
axios.defaults.headers.common["X-CSRFToken"] = csrftoken;

// All subsequent requests include token automatically
axios.post("/api/posts/", data);
axios.put("/api/posts/1/", data);
axios.delete("/api/posts/1/");
```

## 🧪 Testing CSRF Protection

### 1. Test with CSRF Token

```python
from django.test import TestCase, Client

class CSRFTestCase(TestCase):
    def setUp(self):
        self.client = Client(enforce_csrf_checks=True)
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')

    def test_post_without_csrf_fails(self):
        """POST without CSRF token should fail"""
        self.client.login(username='testuser', password='password')
        response = self.client.post('/api/posts/', {
            'title': 'Test Post',
            'content': 'Content'
        })
        self.assertEqual(response.status_code, 403)

    def test_post_with_csrf_succeeds(self):
        """POST with CSRF token should succeed"""
        self.client.login(username='testuser', password='password')

        # Get CSRF token
        response = self.client.get('/form/')
        csrftoken = response.cookies['csrftoken'].value

        # POST with token
        response = self.client.post('/api/posts/', {
            'title': 'Test Post',
            'content': 'Content'
        }, HTTP_X_CSRFTOKEN=csrftoken)

        self.assertEqual(response.status_code, 201)
```

### 2. Test CSRF Exempt View

```python
def test_webhook_csrf_exempt(self):
    """Webhook should work without CSRF token"""
    signature = generate_webhook_signature(payload)

    response = self.client.post('/webhooks/stripe/',
        data=payload,
        content_type='application/json',
        HTTP_X_WEBHOOK_SIGNATURE=signature
    )

    self.assertEqual(response.status_code, 200)
```

## ❓ Interview Questions

### Q1: How does Django's CSRF protection work?

**Answer**:
Django generates a random token stored in cookie and session. For POST/PUT/DELETE requests, middleware validates that the token in the request (form field or header) matches the cookie token.

### Q2: When should you use @csrf_exempt?

**Answer**:
Only for:

1. External webhooks with signature verification
2. API endpoints using token authentication (not session)
3. Public endpoints with no state changes

Never exempt authenticated endpoints using session auth.

### Q3: What's the difference between CSRF_COOKIE_HTTPONLY and CSRF_COOKIE_SECURE?

**Answer**:

- **HTTPONLY**: If True, JavaScript can't read cookie (blocks client-side CSRF token access)
- **SECURE**: If True, cookie only sent over HTTPS (prevents interception)

For CSRF, HTTPONLY should be False (JS needs to read token). SECURE should be True in production.

## 📚 Summary

**Key Takeaways**:

1. Always use {% csrf_token %} in forms
2. Include X-CSRFToken header in AJAX requests
3. Use CSRF_COOKIE_SECURE=True in production
4. Set CSRF_COOKIE_SAMESITE='Strict' or 'Lax'
5. Configure CSRF_TRUSTED_ORIGINS for CORS
6. Token auth doesn't need CSRF protection
7. Session auth requires CSRF protection
8. Never disable CSRF for authenticated endpoints
9. Rotate tokens after sensitive operations
10. Test CSRF protection in test suite

CSRF protection is essential for web security!
