# Django Authentication Security

## 📖 Concept Explanation

Authentication security encompasses protecting user credentials, sessions, and account access from unauthorized use. Weak authentication is one of the OWASP Top 10 vulnerabilities.

### Common Authentication Attacks

```
1. Brute Force: Automated password guessing
2. Credential Stuffing: Using leaked credentials from other breaches
3. Session Hijacking: Stealing session tokens
4. Password Spraying: Trying common passwords across many accounts
5. Account Enumeration: Identifying valid usernames
6. Session Fixation: Forcing user to use attacker's session ID
```

## 🧠 Django Password Security

### 1. Password Hashing (PBKDF2 by Default)

```python
# settings.py
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',        # Default
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
    'django.contrib.auth.hashers.Argon2PasswordHasher',        # Recommended
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',  # Alternative
]

# Django automatically:
# 1. Salts passwords (unique salt per password)
# 2. Uses many iterations (260,000+ for PBKDF2)
# 3. Stores: algorithm$iterations$salt$hash

# Example stored password:
# pbkdf2_sha256$260000$salt$hash
```

### 2. Argon2 (Recommended for Production)

```bash
# Install
pip install django[argon2]
```

```python
# settings.py
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',  # Use first
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',  # Fallback
]

# Argon2 benefits:
# - Winner of Password Hashing Competition (2015)
# - Resistant to GPU/ASIC attacks
# - Memory-hard algorithm
# - Better than PBKDF2/BCrypt
```

### 3. Password Validation

```python
# settings.py
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
        # Prevents passwords similar to username, email, etc.
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {
            'min_length': 12,  # Increased from default 8
        }
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
        # Blocks 20,000 most common passwords
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
        # Prevents all-numeric passwords
    },
]

# Usage in forms
from django.contrib.auth.password_validation import validate_password

class SignupForm(forms.Form):
    password = forms.CharField(widget=forms.PasswordInput)

    def clean_password(self):
        password = self.cleaned_data['password']
        validate_password(password)  # Raises ValidationError if weak
        return password
```

### 4. Custom Password Validator

```python
# validators.py
from django.core.exceptions import ValidationError
from django.utils.translation import gettext as _
import re

class SpecialCharacterValidator:
    """Require at least one special character"""

    def validate(self, password, user=None):
        if not re.search(r'[!@#$%^&*(),.?":{}|<>]', password):
            raise ValidationError(
                _("Password must contain at least one special character."),
                code='password_no_special',
            )

    def get_help_text(self):
        return _("Your password must contain at least one special character: !@#$%^&*(),.?\":{}|<>")

class UppercaseLowercaseValidator:
    """Require both uppercase and lowercase"""

    def validate(self, password, user=None):
        if not re.search(r'[A-Z]', password):
            raise ValidationError(
                _("Password must contain at least one uppercase letter."),
                code='password_no_upper',
            )
        if not re.search(r'[a-z]', password):
            raise ValidationError(
                _("Password must contain at least one lowercase letter."),
                code='password_no_lower',
            )

    def get_help_text(self):
        return _("Your password must contain both uppercase and lowercase letters.")

# settings.py
AUTH_PASSWORD_VALIDATORS = [
    # ... Django's built-in validators ...
    {
        'NAME': 'myapp.validators.SpecialCharacterValidator',
    },
    {
        'NAME': 'myapp.validators.UppercaseLowercaseValidator',
    },
]
```

## 🔐 Session Security

### 1. Secure Session Settings

```python
# settings.py

# Session security
SESSION_COOKIE_SECURE = True        # HTTPS only
SESSION_COOKIE_HTTPONLY = True      # No JavaScript access
SESSION_COOKIE_SAMESITE = 'Strict'  # Prevent CSRF
SESSION_COOKIE_AGE = 3600           # 1 hour (shorter = more secure)

# Session engine
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
# Options:
# - 'db': Database (persistent)
# - 'cache': Redis/Memcached (fast, volatile)
# - 'cached_db': Hybrid (fast + persistent)
# - 'file': File-based (local dev)
# - 'signed_cookies': Client-side (no server storage)

# Clear sessions on logout
SESSION_SAVE_EVERY_REQUEST = False
SESSION_EXPIRE_AT_BROWSER_CLOSE = True  # Session ends when browser closes
```

### 2. Rotate Session on Login

```python
from django.contrib.auth import authenticate, login

def login_view(request):
    """Login with session rotation"""
    if request.method == 'POST':
        username = request.POST['username']
        password = request.POST['password']

        user = authenticate(request, username=username, password=password)

        if user is not None:
            # Rotate session to prevent fixation
            login(request, user)  # Django automatically rotates session key

            # Additional security: regenerate session
            request.session.cycle_key()

            return redirect('dashboard')
        else:
            messages.error(request, 'Invalid credentials')

    return render(request, 'login.html')
```

### 3. Logout and Clear Session

```python
from django.contrib.auth import logout

def logout_view(request):
    """Secure logout"""
    # Clear session data
    logout(request)

    # Force session deletion
    request.session.flush()

    # Redirect to login
    return redirect('login')
```

## 🛡️ Rate Limiting and Brute Force Protection

### 1. Django Axes (Recommended)

```bash
pip install django-axes
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'axes',
]

MIDDLEWARE = [
    # AxesMiddleware should be last
    # ...
    'axes.middleware.AxesMiddleware',
]

AUTHENTICATION_BACKENDS = [
    'axes.backends.AxesBackend',  # Axes backend first
    'django.contrib.auth.backends.ModelBackend',
]

# Axes configuration
AXES_FAILURE_LIMIT = 5              # Lock after 5 failed attempts
AXES_COOLOFF_TIME = 1               # Lock for 1 hour
AXES_LOCK_OUT_BY_COMBINATION_USER_AND_IP = True
AXES_RESET_ON_SUCCESS = True
AXES_LOCKOUT_TEMPLATE = 'lockout.html'
AXES_LOCKOUT_URL = '/locked/'

# Logging
AXES_LOGGER = 'axes.watch_login'
```

### 2. Manual Rate Limiting

```python
from django.core.cache import cache
from django.http import HttpResponseForbidden

def rate_limit_login(request):
    """Manual rate limiting for login"""
    ip = request.META.get('REMOTE_ADDR')
    cache_key = f'login_attempts_{ip}'

    # Get attempt count
    attempts = cache.get(cache_key, 0)

    if attempts >= 5:
        return HttpResponseForbidden('Too many login attempts. Try again later.')

    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')

        user = authenticate(request, username=username, password=password)

        if user is not None:
            # Success: reset counter
            cache.delete(cache_key)
            login(request, user)
            return redirect('dashboard')
        else:
            # Failed: increment counter
            cache.set(cache_key, attempts + 1, timeout=3600)  # 1 hour
            return render(request, 'login.html', {'error': 'Invalid credentials'})

    return render(request, 'login.html')
```

### 3. CAPTCHA After Failed Attempts

```bash
pip install django-recaptcha
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'django_recaptcha',
]

RECAPTCHA_PUBLIC_KEY = os.environ.get('RECAPTCHA_PUBLIC_KEY')
RECAPTCHA_PRIVATE_KEY = os.environ.get('RECAPTCHA_PRIVATE_KEY')

# forms.py
from django_recaptcha.fields import ReCaptchaField

class LoginForm(forms.Form):
    username = forms.CharField()
    password = forms.CharField(widget=forms.PasswordInput)
    captcha = ReCaptchaField()  # Show after N failed attempts

    def __init__(self, *args, show_captcha=False, **kwargs):
        super().__init__(*args, **kwargs)

        if not show_captcha:
            # Hide CAPTCHA initially
            del self.fields['captcha']

# views.py
def login_view(request):
    ip = request.META.get('REMOTE_ADDR')
    attempts = cache.get(f'login_attempts_{ip}', 0)

    # Show CAPTCHA after 3 failed attempts
    show_captcha = attempts >= 3

    if request.method == 'POST':
        form = LoginForm(request.POST, show_captcha=show_captcha)

        if form.is_valid():
            # Process login
            pass
    else:
        form = LoginForm(show_captcha=show_captcha)

    return render(request, 'login.html', {'form': form})
```

## 🎯 Two-Factor Authentication (2FA)

### 1. Django OTP

```bash
pip install django-otp qrcode
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'django_otp',
    'django_otp.plugins.otp_totp',
    'django_otp.plugins.otp_static',
]

MIDDLEWARE = [
    # ...
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django_otp.middleware.OTPMiddleware',  # After AuthMiddleware
]

# views.py
from django_otp.decorators import otp_required
from django_otp.plugins.otp_totp.models import TOTPDevice
import qrcode
import io
import base64

@login_required
def setup_2fa(request):
    """Setup TOTP 2FA"""
    user = request.user

    # Create TOTP device
    device = TOTPDevice.objects.create(
        user=user,
        name='default',
        confirmed=False
    )

    # Generate QR code
    url = device.config_url
    qr = qrcode.make(url)

    # Convert to base64 for display
    buffer = io.BytesIO()
    qr.save(buffer, format='PNG')
    qr_image = base64.b64encode(buffer.getvalue()).decode()

    return render(request, 'setup_2fa.html', {
        'qr_image': qr_image,
        'secret': device.key
    })

@login_required
def verify_2fa(request):
    """Verify 2FA token"""
    if request.method == 'POST':
        token = request.POST.get('token')
        device = TOTPDevice.objects.get(user=request.user, name='default')

        if device.verify_token(token):
            device.confirmed = True
            device.save()

            # Mark session as verified
            request.session['2fa_verified'] = True

            return redirect('dashboard')
        else:
            messages.error(request, 'Invalid token')

    return render(request, 'verify_2fa.html')

@otp_required
def sensitive_view(request):
    """View requiring 2FA"""
    # Only accessible with valid 2FA
    return render(request, 'sensitive.html')
```

### 2. SMS-Based 2FA

```python
# Send SMS code
from twilio.rest import Client

def send_sms_code(phone_number):
    """Send 2FA code via SMS"""
    code = ''.join(random.choices('0123456789', k=6))

    # Store code in cache
    cache.set(f'sms_code_{phone_number}', code, timeout=300)  # 5 minutes

    # Send via Twilio
    client = Client(settings.TWILIO_ACCOUNT_SID, settings.TWILIO_AUTH_TOKEN)
    client.messages.create(
        to=phone_number,
        from_=settings.TWILIO_PHONE_NUMBER,
        body=f'Your verification code is: {code}'
    )

def verify_sms_code(phone_number, code):
    """Verify SMS code"""
    stored_code = cache.get(f'sms_code_{phone_number}')

    if stored_code and stored_code == code:
        cache.delete(f'sms_code_{phone_number}')
        return True

    return False
```

## 🔒 Account Security Features

### 1. Password Reset Security

```python
# settings.py
PASSWORD_RESET_TIMEOUT = 3600  # 1 hour (default: 3 days)

# views.py
from django.contrib.auth.views import PasswordResetView
from django.contrib.auth.tokens import default_token_generator

class SecurePasswordResetView(PasswordResetView):
    """Secure password reset"""

    def form_valid(self, form):
        """Don't reveal if email exists"""
        # Always show success message (prevents email enumeration)
        messages.success(
            self.request,
            'If an account exists with this email, '
            'password reset instructions have been sent.'
        )

        # Only send email if user exists
        email = form.cleaned_data['email']
        if User.objects.filter(email=email).exists():
            return super().form_valid(form)

        return redirect('password_reset_done')
```

### 2. Account Lockout

```python
# models.py
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    failed_login_attempts = models.IntegerField(default=0)
    account_locked_until = models.DateTimeField(null=True, blank=True)

    def is_locked(self):
        """Check if account is locked"""
        if self.account_locked_until:
            if timezone.now() < self.account_locked_until:
                return True
            else:
                # Lock expired, reset
                self.account_locked_until = None
                self.failed_login_attempts = 0
                self.save()
        return False

    def lock_account(self):
        """Lock account for 1 hour"""
        self.account_locked_until = timezone.now() + timedelta(hours=1)
        self.save()

# views.py
def login_view(request):
    if request.method == 'POST':
        username = request.POST['username']
        password = request.POST['password']

        try:
            user = User.objects.get(username=username)

            if user.is_locked():
                return render(request, 'login.html', {
                    'error': 'Account is locked. Try again later.'
                })

            authenticated_user = authenticate(request, username=username, password=password)

            if authenticated_user:
                # Success: reset counter
                user.failed_login_attempts = 0
                user.save()
                login(request, authenticated_user)
                return redirect('dashboard')
            else:
                # Failed: increment counter
                user.failed_login_attempts += 1

                if user.failed_login_attempts >= 5:
                    user.lock_account()
                    messages.error(request, 'Too many failed attempts. Account locked.')

                user.save()

        except User.DoesNotExist:
            # Don't reveal that user doesn't exist
            pass

        messages.error(request, 'Invalid credentials')

    return render(request, 'login.html')
```

## ✅ Best Practices

### 1. Secure Password Storage

```python
# Never store plaintext passwords!
# Bad:
user.password = password  # DON'T DO THIS!

# Good:
user.set_password(password)  # Uses Django's hashers
user.save()

# Check password
if user.check_password(password):
    # Password is correct
    pass
```

### 2. Session Management

```python
# Logout all other sessions
from django.contrib.sessions.models import Session

def logout_all_sessions(user):
    """Logout user from all devices"""
    # Get all sessions
    for session in Session.objects.all():
        if session.get_decoded().get('_auth_user_id') == str(user.id):
            session.delete()
```

### 3. Security Logging

```python
# Log authentication events
import logging

logger = logging.getLogger('security')

def log_login_attempt(username, ip, success):
    """Log login attempts"""
    if success:
        logger.info(f"Successful login: {username} from {ip}")
    else:
        logger.warning(f"Failed login attempt: {username} from {ip}")

def log_password_change(user):
    """Log password changes"""
    logger.info(f"Password changed: {user.username}")
```

## ❓ Interview Questions

### Q1: Why use Argon2 over PBKDF2?

**Answer**:
Argon2 is memory-hard (requires significant RAM), making GPU/ASIC attacks much more expensive. It won the Password Hashing Competition. PBKDF2 is CPU-bound and more vulnerable to specialized hardware attacks.

### Q2: How do you prevent account enumeration?

**Answer**:

1. Same response for valid/invalid usernames
2. Same response time (use sleep if needed)
3. Generic error messages
4. Don't reveal if email exists in password reset

### Q3: What's session fixation and how does Django prevent it?

**Answer**:
Attack where attacker forces victim to use attacker's session ID. Django prevents it by rotating session key on login (calling `login()` generates new session ID).

## 📚 Summary

**Key Takeaways**:

1. Use Argon2 for password hashing
2. Enforce strong password requirements (12+ chars, complexity)
3. Secure sessions: HTTPS-only, HttpOnly, SameSite
4. Implement rate limiting (django-axes)
5. Add 2FA for sensitive accounts
6. Rotate sessions on login/logout
7. Lock accounts after failed attempts
8. Log all authentication events
9. Don't reveal account existence
10. Set short session timeouts

Authentication security is critical for protecting user accounts!
