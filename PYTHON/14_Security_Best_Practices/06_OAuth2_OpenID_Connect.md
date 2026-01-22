# 🔐 OAuth2 and OpenID Connect

## Overview

Modern authentication and authorization protocols for securing APIs and enabling single sign-on (SSO) across applications.

---

## OAuth2 Fundamentals

### What is OAuth2?

```python
"""
OAuth2 = Authorization Framework (RFC 6749)

Purpose:
- Delegate access to resources
- Third-party apps access user data WITHOUT password
- "Allow App X to access your Google Drive"

NOT for:
❌ Authentication (use OpenID Connect)
❌ Password management
❌ User identity verification

Key Concepts:
1. Resource Owner: User who owns the data
2. Client: Application requesting access (your app)
3. Resource Server: API hosting protected resources (e.g., Google Drive API)
4. Authorization Server: Issues access tokens (e.g., Google OAuth)

Tokens:
- Access Token: Short-lived (1 hour), grants access to resources
- Refresh Token: Long-lived (days/months), gets new access tokens
"""
```

---

## OAuth2 Grant Types (Flows)

### 1. Authorization Code Flow (Most Secure)

```python
"""
Best for: Web apps, mobile apps (with PKCE)

Steps:
1. User clicks "Login with Google"
2. Redirect to authorization server
3. User logs in and grants permission
4. Server redirects back with authorization code
5. App exchanges code for access token (backend)
6. App uses access token to call APIs

Security:
✅ Access token never exposed to browser
✅ Code is single-use, expires quickly
✅ Requires client secret (for web apps)

Flow Diagram:
User -> Client: Click login
Client -> Auth Server: Redirect to /authorize
Auth Server -> User: Show consent screen
User -> Auth Server: Grant permission
Auth Server -> Client: Redirect with code
Client -> Auth Server: POST /token with code + secret
Auth Server -> Client: Return access token
Client -> Resource Server: API call with token
"""

from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import RedirectResponse
import httpx
import secrets
from urllib.parse import urlencode

app = FastAPI()

# Configuration
OAUTH_CONFIG = {
    'client_id': 'your-client-id',
    'client_secret': 'your-client-secret',
    'authorization_endpoint': 'https://accounts.google.com/o/oauth2/v2/auth',
    'token_endpoint': 'https://oauth2.googleapis.com/token',
    'redirect_uri': 'http://localhost:8000/callback',
    'scope': 'openid email profile'
}

# Store state to prevent CSRF (in production, use Redis)
oauth_states = {}

@app.get("/login")
async def login():
    """
    Step 1: Redirect to authorization server

    Generate state parameter to prevent CSRF attacks.
    """
    # Generate random state
    state = secrets.token_urlsafe(32)
    oauth_states[state] = True

    # Build authorization URL
    params = {
        'client_id': OAUTH_CONFIG['client_id'],
        'redirect_uri': OAUTH_CONFIG['redirect_uri'],
        'response_type': 'code',
        'scope': OAUTH_CONFIG['scope'],
        'state': state,
        'access_type': 'offline',  # Request refresh token
        'prompt': 'consent'  # Force consent screen
    }

    auth_url = f"{OAUTH_CONFIG['authorization_endpoint']}?{urlencode(params)}"

    return RedirectResponse(auth_url)

@app.get("/callback")
async def callback(code: str, state: str):
    """
    Step 2: Handle callback with authorization code

    Exchange code for access token.
    """
    # Verify state to prevent CSRF
    if state not in oauth_states:
        raise HTTPException(status_code=400, detail="Invalid state")

    del oauth_states[state]

    # Exchange code for token
    async with httpx.AsyncClient() as client:
        response = await client.post(
            OAUTH_CONFIG['token_endpoint'],
            data={
                'code': code,
                'client_id': OAUTH_CONFIG['client_id'],
                'client_secret': OAUTH_CONFIG['client_secret'],
                'redirect_uri': OAUTH_CONFIG['redirect_uri'],
                'grant_type': 'authorization_code'
            },
            headers={'Content-Type': 'application/x-www-form-urlencoded'}
        )

    if response.status_code != 200:
        raise HTTPException(status_code=400, detail="Token exchange failed")

    token_data = response.json()

    # Token response contains:
    # {
    #     "access_token": "ya29.a0AfH6...",
    #     "expires_in": 3600,
    #     "refresh_token": "1//0gK8...",  # Only if offline access
    #     "scope": "openid email profile",
    #     "token_type": "Bearer"
    # }

    # Store tokens securely (in production, encrypt and store in DB)
    access_token = token_data['access_token']
    refresh_token = token_data.get('refresh_token')

    # Fetch user info
    user_info = await get_user_info(access_token)

    return {
        'message': 'Login successful',
        'user': user_info,
        'access_token': access_token
    }

async def get_user_info(access_token: str) -> dict:
    """Fetch user info from resource server"""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            'https://www.googleapis.com/oauth2/v2/userinfo',
            headers={'Authorization': f'Bearer {access_token}'}
        )

    return response.json()
```

### 2. PKCE (Authorization Code + PKCE for Mobile/SPA)

```python
"""
PKCE = Proof Key for Code Exchange (RFC 7636)

Problem: Mobile/SPA apps can't keep client_secret secure
Solution: Use code challenge/verifier instead

Steps:
1. Generate code_verifier (random string)
2. Generate code_challenge = SHA256(code_verifier)
3. Send code_challenge with authorization request
4. Send code_verifier with token request
5. Server verifies: SHA256(verifier) == challenge

Security:
✅ No client secret needed
✅ Protects against authorization code interception
✅ Recommended for all OAuth2 flows now
"""

import hashlib
import base64

def generate_pkce_pair():
    """
    Generate PKCE code verifier and challenge

    Returns:
        (code_verifier, code_challenge)
    """
    # Generate code verifier (43-128 chars)
    code_verifier = secrets.token_urlsafe(64)

    # Generate code challenge: base64url(SHA256(verifier))
    challenge_bytes = hashlib.sha256(code_verifier.encode()).digest()
    code_challenge = base64.urlsafe_b64encode(challenge_bytes).decode().rstrip('=')

    return code_verifier, code_challenge

@app.get("/login/pkce")
async def login_pkce():
    """Login with PKCE (for mobile/SPA)"""
    state = secrets.token_urlsafe(32)
    code_verifier, code_challenge = generate_pkce_pair()

    # Store verifier for later (associated with state)
    oauth_states[state] = {
        'code_verifier': code_verifier,
        'timestamp': time.time()
    }

    params = {
        'client_id': OAUTH_CONFIG['client_id'],
        'redirect_uri': OAUTH_CONFIG['redirect_uri'],
        'response_type': 'code',
        'scope': OAUTH_CONFIG['scope'],
        'state': state,
        'code_challenge': code_challenge,
        'code_challenge_method': 'S256'  # SHA256
    }

    auth_url = f"{OAUTH_CONFIG['authorization_endpoint']}?{urlencode(params)}"
    return RedirectResponse(auth_url)

@app.get("/callback/pkce")
async def callback_pkce(code: str, state: str):
    """Callback with PKCE"""
    if state not in oauth_states:
        raise HTTPException(status_code=400, detail="Invalid state")

    state_data = oauth_states[state]
    code_verifier = state_data['code_verifier']
    del oauth_states[state]

    # Exchange code for token (with verifier instead of secret)
    async with httpx.AsyncClient() as client:
        response = await client.post(
            OAUTH_CONFIG['token_endpoint'],
            data={
                'code': code,
                'client_id': OAUTH_CONFIG['client_id'],
                'redirect_uri': OAUTH_CONFIG['redirect_uri'],
                'grant_type': 'authorization_code',
                'code_verifier': code_verifier  # PKCE
            }
        )

    return response.json()
```

### 3. Client Credentials Flow

```python
"""
Client Credentials Flow

Use case: Server-to-server authentication (no user involved)
Example: Microservice calling another microservice

Steps:
1. Service sends client_id + client_secret
2. Auth server returns access token
3. Service uses token to call API

Security:
⚠️ Only for confidential clients
⚠️ Client secret must be secure
"""

async def get_service_token() -> str:
    """
    Get access token for service-to-service auth

    No user involved, just client credentials.
    """
    async with httpx.AsyncClient() as client:
        response = await client.post(
            OAUTH_CONFIG['token_endpoint'],
            data={
                'grant_type': 'client_credentials',
                'client_id': OAUTH_CONFIG['client_id'],
                'client_secret': OAUTH_CONFIG['client_secret'],
                'scope': 'api.read api.write'
            },
            headers={'Content-Type': 'application/x-www-form-urlencoded'}
        )

    token_data = response.json()
    return token_data['access_token']

# Usage
@app.get("/service/data")
async def call_external_service():
    """Call external API with service token"""
    # Get token
    access_token = await get_service_token()

    # Call API
    async with httpx.AsyncClient() as client:
        response = await client.get(
            'https://api.example.com/data',
            headers={'Authorization': f'Bearer {access_token}'}
        )

    return response.json()
```

### 4. Refresh Token Flow

```python
"""
Refresh Token Flow

Purpose: Get new access token without user re-authentication

When:
- Access token expired
- Want to reduce login frequency

Security:
✅ Refresh token long-lived but can be revoked
✅ Access token short-lived (limits damage if leaked)
✅ Refresh token stored securely (encrypted DB)
"""

async def refresh_access_token(refresh_token: str) -> dict:
    """
    Refresh access token

    Returns new access token (and possibly new refresh token).
    """
    async with httpx.AsyncClient() as client:
        response = await client.post(
            OAUTH_CONFIG['token_endpoint'],
            data={
                'grant_type': 'refresh_token',
                'refresh_token': refresh_token,
                'client_id': OAUTH_CONFIG['client_id'],
                'client_secret': OAUTH_CONFIG['client_secret']
            }
        )

    if response.status_code != 200:
        raise HTTPException(status_code=401, detail="Refresh token invalid or expired")

    return response.json()

# Middleware to handle token refresh
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_valid_token(credentials: HTTPAuthorizationCredentials = Depends(security)) -> str:
    """
    Validate token and refresh if needed

    If token expired, try to refresh automatically.
    """
    access_token = credentials.credentials

    # Check if token valid (call token introspection or try API)
    try:
        # Validate token
        await validate_token(access_token)
        return access_token
    except HTTPException as e:
        if e.status_code == 401:
            # Token expired, try to refresh
            refresh_token = get_stored_refresh_token()  # From DB/session

            if refresh_token:
                token_data = await refresh_access_token(refresh_token)
                new_access_token = token_data['access_token']

                # Update stored tokens
                store_tokens(new_access_token, token_data.get('refresh_token'))

                return new_access_token

        raise

async def validate_token(token: str):
    """Validate access token (example)"""
    # Option 1: Call userinfo endpoint
    async with httpx.AsyncClient() as client:
        response = await client.get(
            'https://www.googleapis.com/oauth2/v2/userinfo',
            headers={'Authorization': f'Bearer {token}'}
        )

    if response.status_code != 200:
        raise HTTPException(status_code=401, detail="Invalid token")
```

---

## OpenID Connect (OIDC)

### What is OpenID Connect?

```python
"""
OpenID Connect = Authentication layer on top of OAuth2

OAuth2: Authorization (access delegation)
OIDC: Authentication (who is the user?)

Additions:
- ID Token (JWT): Contains user identity
- UserInfo endpoint: Get user profile
- Standard claims: sub, name, email, picture

ID Token Structure (JWT):
{
  "iss": "https://accounts.google.com",  # Issuer
  "sub": "10769150350006150715113082367",  # Subject (user ID)
  "aud": "your-client-id",  # Audience
  "exp": 1661234567,  # Expiration
  "iat": 1661230967,  # Issued at
  "email": "user@example.com",
  "email_verified": true,
  "name": "John Doe",
  "picture": "https://..."
}
"""

import jwt
from jwt import PyJWKClient

def verify_id_token(id_token: str, client_id: str) -> dict:
    """
    Verify and decode ID token (JWT)

    Steps:
    1. Fetch public keys from provider
    2. Verify signature
    3. Verify claims (aud, exp, iss)
    4. Return payload
    """
    # Fetch public keys from Google
    jwks_client = PyJWKClient('https://www.googleapis.com/oauth2/v3/certs')
    signing_key = jwks_client.get_signing_key_from_jwt(id_token)

    # Verify token
    try:
        payload = jwt.decode(
            id_token,
            signing_key.key,
            algorithms=['RS256'],
            audience=client_id,
            issuer='https://accounts.google.com'
        )

        return payload

    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {str(e)}")

@app.get("/callback/oidc")
async def callback_oidc(code: str, state: str):
    """
    Callback with OpenID Connect

    Exchange code for tokens (access + ID token).
    """
    # Exchange code (same as OAuth2)
    async with httpx.AsyncClient() as client:
        response = await client.post(
            OAUTH_CONFIG['token_endpoint'],
            data={
                'code': code,
                'client_id': OAUTH_CONFIG['client_id'],
                'client_secret': OAUTH_CONFIG['client_secret'],
                'redirect_uri': OAUTH_CONFIG['redirect_uri'],
                'grant_type': 'authorization_code'
            }
        )

    token_data = response.json()

    # Verify ID token
    id_token = token_data['id_token']
    user_info = verify_id_token(id_token, OAUTH_CONFIG['client_id'])

    return {
        'message': 'Login successful',
        'user': {
            'id': user_info['sub'],
            'email': user_info['email'],
            'name': user_info['name'],
            'picture': user_info.get('picture')
        },
        'access_token': token_data['access_token']
    }
```

---

## Token Storage and Security

### Secure Token Storage

```python
"""
Token Storage Best Practices:

Access Token:
❌ LocalStorage (XSS vulnerable)
❌ SessionStorage (XSS vulnerable)
✅ HTTP-only cookie (backend sets, JS can't access)
✅ Memory (SPA, lost on refresh)

Refresh Token:
❌ Browser storage (NEVER)
✅ HTTP-only, Secure, SameSite cookie
✅ Encrypted database
✅ Rotate on use

Security Measures:
1. Short-lived access tokens (15 min - 1 hour)
2. Long-lived refresh tokens (days - months)
3. Refresh token rotation
4. Token revocation
5. HTTPS only
6. CSRF protection
"""

from fastapi import Response
from cryptography.fernet import Fernet

# Token encryption key (store in environment)
ENCRYPTION_KEY = Fernet.generate_key()
cipher = Fernet(ENCRYPTION_KEY)

def encrypt_token(token: str) -> str:
    """Encrypt token for storage"""
    return cipher.encrypt(token.encode()).decode()

def decrypt_token(encrypted_token: str) -> str:
    """Decrypt token from storage"""
    return cipher.decrypt(encrypted_token.encode()).decode()

@app.get("/secure-login")
async def secure_login_callback(code: str, response: Response):
    """
    Secure token handling

    - Access token in HTTP-only cookie
    - Refresh token encrypted in DB
    - No tokens in response body
    """
    # Exchange code for tokens
    token_data = await exchange_code_for_tokens(code)

    access_token = token_data['access_token']
    refresh_token = token_data['refresh_token']

    # Store refresh token in DB (encrypted)
    user_id = await create_or_update_user(token_data)
    encrypted_refresh = encrypt_token(refresh_token)
    await store_refresh_token(user_id, encrypted_refresh)

    # Set access token in HTTP-only cookie
    response.set_cookie(
        key="access_token",
        value=access_token,
        httponly=True,  # JS can't access
        secure=True,  # HTTPS only
        samesite="strict",  # CSRF protection
        max_age=3600  # 1 hour
    )

    # Set refresh token in HTTP-only cookie
    response.set_cookie(
        key="refresh_token",
        value=refresh_token,
        httponly=True,
        secure=True,
        samesite="strict",
        max_age=604800,  # 7 days
        path="/auth/refresh"  # Limited scope
    )

    return {
        'message': 'Login successful',
        'user_id': user_id
    }

@app.post("/auth/refresh")
async def refresh_endpoint(request: Request, response: Response):
    """
    Refresh access token

    - Read refresh token from HTTP-only cookie
    - Rotate refresh token (issue new one)
    - Return new tokens in cookies
    """
    # Get refresh token from cookie
    refresh_token = request.cookies.get('refresh_token')

    if not refresh_token:
        raise HTTPException(status_code=401, detail="No refresh token")

    # Validate and use refresh token
    token_data = await refresh_access_token(refresh_token)

    new_access_token = token_data['access_token']
    new_refresh_token = token_data.get('refresh_token', refresh_token)

    # Rotate refresh token in DB
    if new_refresh_token != refresh_token:
        await rotate_refresh_token(refresh_token, new_refresh_token)

    # Update cookies
    response.set_cookie(
        key="access_token",
        value=new_access_token,
        httponly=True,
        secure=True,
        samesite="strict",
        max_age=3600
    )

    if new_refresh_token != refresh_token:
        response.set_cookie(
            key="refresh_token",
            value=new_refresh_token,
            httponly=True,
            secure=True,
            samesite="strict",
            max_age=604800,
            path="/auth/refresh"
        )

    return {'message': 'Token refreshed'}

@app.post("/logout")
async def logout(request: Request, response: Response):
    """
    Logout user

    - Revoke tokens
    - Clear cookies
    """
    access_token = request.cookies.get('access_token')
    refresh_token = request.cookies.get('refresh_token')

    # Revoke tokens on auth server
    if refresh_token:
        await revoke_token(refresh_token)

    # Clear cookies
    response.delete_cookie('access_token')
    response.delete_cookie('refresh_token', path="/auth/refresh")

    return {'message': 'Logged out'}
```

---

## Best Practices

### ✅ Do's:

1. **Use PKCE** - For all authorization code flows (mobile/SPA/web)
2. **Short-lived access tokens** - 15 min to 1 hour
3. **Rotate refresh tokens** - Issue new on each use
4. **HTTPS only** - Never transmit tokens over HTTP
5. **Validate tokens** - Verify signature, expiry, audience
6. **Store securely** - HTTP-only cookies, encrypted DB
7. **Implement token revocation** - Allow users to logout

### ❌ Don'ts:

1. **Don't use implicit flow** - Deprecated (insecure)
2. **Don't store tokens in localStorage** - XSS vulnerable
3. **Don't log tokens** - Sensitive data
4. **Don't share client secrets** - Keep confidential
5. **Don't trust tokens blindly** - Always validate

---

## Interview Questions

### Q1: OAuth2 vs OpenID Connect?

**Answer**:

- **OAuth2**: Authorization framework (delegate access)
- **OIDC**: Authentication layer on OAuth2 (identity)
- **OAuth2**: Access token (opaque), access resources
- **OIDC**: ID token (JWT), user identity + access token
- **Use OAuth2**: API access delegation
- **Use OIDC**: User login, SSO
  OIDC = OAuth2 + identity.

### Q2: Why use PKCE?

**Answer**:

- **Problem**: Mobile/SPA can't securely store client_secret
- **Solution**: Dynamic secret per authorization (code_verifier)
- **Flow**: Generate verifier → hash to challenge → verify on token exchange
- **Security**: Prevents authorization code interception
- **Recommendation**: Use for ALL OAuth2 flows now
  PKCE eliminates need for client secret in public clients.

### Q3: Where to store access tokens?

**Answer**:

- **Backend**: Encrypted database, HTTP-only cookies
- **Frontend (SPA)**: Memory (lost on refresh), or backend manages
- **Never**: localStorage (XSS), sessionStorage (XSS)
- **Best**: HTTP-only, Secure, SameSite cookies set by backend
- **Trade-off**: Cookies vs bearer token (CSRF vs XSS)
  Security over convenience - use HTTP-only cookies.

### Q4: How does refresh token work?

**Answer**:

- **Purpose**: Get new access token without re-login
- **Lifetime**: Access short (1 hour), refresh long (7-30 days)
- **Flow**: Access expires → use refresh to get new access → continue
- **Security**: Refresh can be revoked, rotate on use
- **Storage**: Encrypted, HTTP-only cookie or secure DB
  Balances security and UX.

### Q5: What's authorization code vs client credentials?

**Answer**:

- **Authorization code**: User grants access to their resources (3-legged)
- **Client credentials**: App accesses its own resources (2-legged)
- **Auth code**: User involved, consent screen, delegated access
- **Client creds**: No user, service-to-service, own resources
- **Example auth code**: "Allow App X to read your email"
- **Example client creds**: Microservice calling internal API
  Choose based on whether user is involved.

---

## Summary

OAuth2 & OIDC essentials:

- **OAuth2**: Authorization (delegate access), 4 grant types
- **OIDC**: Authentication (identity), ID token (JWT)
- **PKCE**: Dynamic secret, use for all flows
- **Tokens**: Access (short), refresh (long), ID (identity)
- **Security**: HTTPS, HTTP-only cookies, validate tokens
- **Storage**: Encrypt, never localStorage

Modern auth for secure applications! 🔐
