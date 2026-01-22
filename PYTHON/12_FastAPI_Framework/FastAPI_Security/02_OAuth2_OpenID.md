# 🔒 OAuth2 and OpenID Connect

## Overview

OAuth2 is an authorization framework that enables applications to obtain limited access to user accounts. OpenID Connect (OIDC) is an identity layer built on top of OAuth2 for authentication. FastAPI provides excellent support for implementing OAuth2 flows.

---

## OAuth2 Fundamentals

### Key Concepts

**OAuth2 Roles:**

- **Resource Owner**: User who owns the data
- **Client**: Application requesting access
- **Authorization Server**: Issues access tokens
- **Resource Server**: Hosts protected resources

**Grant Types:**

1. **Authorization Code**: Most secure, for server-side apps
2. **Password**: Direct username/password exchange
3. **Client Credentials**: Machine-to-machine
4. **Implicit**: Deprecated, for browser-only apps
5. **Refresh Token**: Get new access token

---

## Password Grant Flow

### 1. **Basic Implementation**

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext

app = FastAPI()

# Configuration
SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# Security
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Models
class User(BaseModel):
    username: str
    email: str
    full_name: str = None
    disabled: bool = False

class UserInDB(User):
    hashed_password: str

class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: str = None

# Fake database
fake_users_db = {
    "johndoe": {
        "username": "johndoe",
        "full_name": "John Doe",
        "email": "johndoe@example.com",
        "hashed_password": "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW",
        "disabled": False,
    }
}

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_user(db, username: str):
    if username in db:
        user_dict = db[username]
        return UserInDB(**user_dict)

def authenticate_user(fake_db, username: str, password: str):
    user = get_user(fake_db, username)
    if not user:
        return False
    if not verify_password(password, user.hashed_password):
        return False
    return user

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username)
    except JWTError:
        raise credentials_exception
    user = get_user(fake_users_db, username=token_data.username)
    if user is None:
        raise credentials_exception
    return user

async def get_current_active_user(current_user: User = Depends(get_current_user)):
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(fake_users_db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username}, expires_delta=access_token_expires
    )
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/users/me", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_active_user)):
    return current_user
```

---

## Authorization Code Flow

### 1. **Authorization Code Grant**

```python
from fastapi import FastAPI, Request
from fastapi.responses import RedirectResponse
import secrets
import hashlib
import base64

app = FastAPI()

# Storage for authorization codes (use Redis in production)
authorization_codes = {}
access_tokens = {}

# OAuth2 Configuration
AUTHORIZATION_ENDPOINT = "/oauth/authorize"
TOKEN_ENDPOINT = "/oauth/token"
CLIENT_ID = "my-client-id"
CLIENT_SECRET = "my-client-secret"
REDIRECT_URI = "http://localhost:8000/callback"

def generate_authorization_code() -> str:
    """Generate random authorization code"""
    return secrets.token_urlsafe(32)

def generate_access_token() -> str:
    """Generate access token"""
    return secrets.token_urlsafe(32)

@app.get("/oauth/authorize")
async def authorize(
    client_id: str,
    redirect_uri: str,
    response_type: str,
    scope: str = None,
    state: str = None
):
    """Authorization endpoint"""
    # Validate client
    if client_id != CLIENT_ID:
        raise HTTPException(400, "Invalid client_id")

    if response_type != "code":
        raise HTTPException(400, "Unsupported response_type")

    # In real app: Show login page and consent screen
    # For demo: Directly generate code

    # Generate authorization code
    code = generate_authorization_code()

    # Store code with user info
    authorization_codes[code] = {
        "client_id": client_id,
        "redirect_uri": redirect_uri,
        "scope": scope,
        "user_id": "user123",  # From authenticated session
        "expires_at": datetime.utcnow() + timedelta(minutes=10)
    }

    # Redirect back to client with code
    redirect_url = f"{redirect_uri}?code={code}"
    if state:
        redirect_url += f"&state={state}"

    return RedirectResponse(redirect_url)

@app.post("/oauth/token")
async def token(
    grant_type: str,
    code: str = None,
    redirect_uri: str = None,
    client_id: str = None,
    client_secret: str = None,
    refresh_token: str = None
):
    """Token endpoint"""

    if grant_type == "authorization_code":
        # Validate client credentials
        if client_id != CLIENT_ID or client_secret != CLIENT_SECRET:
            raise HTTPException(401, "Invalid client credentials")

        # Validate authorization code
        if code not in authorization_codes:
            raise HTTPException(400, "Invalid authorization code")

        code_data = authorization_codes[code]

        # Validate redirect_uri
        if redirect_uri != code_data["redirect_uri"]:
            raise HTTPException(400, "Invalid redirect_uri")

        # Check expiration
        if datetime.utcnow() > code_data["expires_at"]:
            del authorization_codes[code]
            raise HTTPException(400, "Authorization code expired")

        # Generate tokens
        access_token = generate_access_token()
        refresh_token_value = generate_access_token()

        # Store tokens
        access_tokens[access_token] = {
            "user_id": code_data["user_id"],
            "scope": code_data["scope"],
            "expires_at": datetime.utcnow() + timedelta(hours=1)
        }

        # Delete used code
        del authorization_codes[code]

        return {
            "access_token": access_token,
            "token_type": "Bearer",
            "expires_in": 3600,
            "refresh_token": refresh_token_value,
            "scope": code_data["scope"]
        }

    elif grant_type == "refresh_token":
        # Handle refresh token
        # Implementation similar to above
        pass

    else:
        raise HTTPException(400, "Unsupported grant_type")

@app.get("/api/user")
async def get_user_info(authorization: str = Header(None)):
    """Protected API endpoint"""
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(401, "Missing or invalid authorization header")

    token = authorization.split(" ")[1]

    if token not in access_tokens:
        raise HTTPException(401, "Invalid access token")

    token_data = access_tokens[token]

    # Check expiration
    if datetime.utcnow() > token_data["expires_at"]:
        del access_tokens[token]
        raise HTTPException(401, "Access token expired")

    return {
        "user_id": token_data["user_id"],
        "scope": token_data["scope"]
    }
```

### 2. **PKCE (Proof Key for Code Exchange)**

```python
import hashlib
import base64

def generate_code_verifier() -> str:
    """Generate code verifier for PKCE"""
    return base64.urlsafe_b64encode(secrets.token_bytes(32)).decode('utf-8').rstrip('=')

def generate_code_challenge(verifier: str) -> str:
    """Generate code challenge from verifier"""
    digest = hashlib.sha256(verifier.encode('utf-8')).digest()
    return base64.urlsafe_b64encode(digest).decode('utf-8').rstrip('=')

# Client-side code
code_verifier = generate_code_verifier()
code_challenge = generate_code_challenge(code_verifier)

# Store code_verifier for later use

@app.get("/oauth/authorize/pkce")
async def authorize_pkce(
    client_id: str,
    redirect_uri: str,
    code_challenge: str,
    code_challenge_method: str = "S256",
    state: str = None
):
    """Authorization endpoint with PKCE"""
    if code_challenge_method != "S256":
        raise HTTPException(400, "Only S256 supported")

    # Generate code
    code = generate_authorization_code()

    # Store with code_challenge
    authorization_codes[code] = {
        "client_id": client_id,
        "redirect_uri": redirect_uri,
        "code_challenge": code_challenge,
        "code_challenge_method": code_challenge_method,
        "user_id": "user123",
        "expires_at": datetime.utcnow() + timedelta(minutes=10)
    }

    redirect_url = f"{redirect_uri}?code={code}"
    if state:
        redirect_url += f"&state={state}"

    return RedirectResponse(redirect_url)

@app.post("/oauth/token/pkce")
async def token_pkce(
    grant_type: str,
    code: str,
    redirect_uri: str,
    client_id: str,
    code_verifier: str
):
    """Token endpoint with PKCE verification"""
    if grant_type != "authorization_code":
        raise HTTPException(400, "Unsupported grant_type")

    if code not in authorization_codes:
        raise HTTPException(400, "Invalid authorization code")

    code_data = authorization_codes[code]

    # Verify code_challenge
    challenge = generate_code_challenge(code_verifier)
    if challenge != code_data["code_challenge"]:
        raise HTTPException(400, "Invalid code_verifier")

    # Generate access token
    access_token = generate_access_token()

    access_tokens[access_token] = {
        "user_id": code_data["user_id"],
        "expires_at": datetime.utcnow() + timedelta(hours=1)
    }

    del authorization_codes[code]

    return {
        "access_token": access_token,
        "token_type": "Bearer",
        "expires_in": 3600
    }
```

---

## Client Credentials Flow

### 1. **Machine-to-Machine Authentication**

```python
from typing import List

# Client credentials storage
clients = {
    "service-a": {
        "client_secret": "secret-a",
        "scopes": ["read:data", "write:data"]
    },
    "service-b": {
        "client_secret": "secret-b",
        "scopes": ["read:data"]
    }
}

@app.post("/oauth/token/client-credentials")
async def client_credentials_token(
    grant_type: str,
    client_id: str,
    client_secret: str,
    scope: str = None
):
    """Client credentials grant"""
    if grant_type != "client_credentials":
        raise HTTPException(400, "Unsupported grant_type")

    # Validate client
    if client_id not in clients:
        raise HTTPException(401, "Invalid client_id")

    client = clients[client_id]

    if client["client_secret"] != client_secret:
        raise HTTPException(401, "Invalid client_secret")

    # Validate scope
    requested_scopes = scope.split() if scope else []
    allowed_scopes = client["scopes"]

    if not all(s in allowed_scopes for s in requested_scopes):
        raise HTTPException(403, "Invalid scope")

    # Generate token
    access_token = generate_access_token()

    access_tokens[access_token] = {
        "client_id": client_id,
        "scope": requested_scopes,
        "token_type": "client_credentials",
        "expires_at": datetime.utcnow() + timedelta(hours=1)
    }

    return {
        "access_token": access_token,
        "token_type": "Bearer",
        "expires_in": 3600,
        "scope": " ".join(requested_scopes)
    }

def require_scope(required_scope: str):
    """Dependency to check scope"""
    async def check_scope(authorization: str = Header(None)):
        if not authorization or not authorization.startswith("Bearer "):
            raise HTTPException(401, "Missing authorization")

        token = authorization.split(" ")[1]

        if token not in access_tokens:
            raise HTTPException(401, "Invalid token")

        token_data = access_tokens[token]

        if required_scope not in token_data.get("scope", []):
            raise HTTPException(403, f"Scope '{required_scope}' required")

        return token_data

    return check_scope

@app.get("/api/data")
async def get_data(token_data: dict = Depends(require_scope("read:data"))):
    return {"data": "sensitive information", "client": token_data["client_id"]}
```

---

## OpenID Connect

### 1. **ID Token**

```python
from jose import jwt

def create_id_token(
    user_id: str,
    email: str,
    name: str,
    client_id: str
) -> str:
    """Create OpenID Connect ID token"""
    payload = {
        "iss": "https://your-domain.com",  # Issuer
        "sub": user_id,                     # Subject (user ID)
        "aud": client_id,                   # Audience (client ID)
        "exp": datetime.utcnow() + timedelta(hours=1),
        "iat": datetime.utcnow(),           # Issued at
        "auth_time": datetime.utcnow(),     # Authentication time
        "nonce": secrets.token_urlsafe(16), # Nonce from auth request
        "email": email,
        "email_verified": True,
        "name": name,
        "preferred_username": email.split("@")[0]
    }

    id_token = jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
    return id_token

@app.post("/oauth/token/oidc")
async def oidc_token(
    grant_type: str,
    code: str,
    redirect_uri: str,
    client_id: str,
    client_secret: str
):
    """Token endpoint with OpenID Connect"""
    # Validate authorization code (similar to above)
    if code not in authorization_codes:
        raise HTTPException(400, "Invalid code")

    code_data = authorization_codes[code]

    # Validate client
    if client_id != CLIENT_ID or client_secret != CLIENT_SECRET:
        raise HTTPException(401, "Invalid client")

    # Generate tokens
    access_token = generate_access_token()
    refresh_token = generate_access_token()

    # Create ID token
    id_token = create_id_token(
        user_id=code_data["user_id"],
        email="user@example.com",
        name="John Doe",
        client_id=client_id
    )

    # Store access token
    access_tokens[access_token] = {
        "user_id": code_data["user_id"],
        "expires_at": datetime.utcnow() + timedelta(hours=1)
    }

    del authorization_codes[code]

    return {
        "access_token": access_token,
        "token_type": "Bearer",
        "expires_in": 3600,
        "refresh_token": refresh_token,
        "id_token": id_token  # OpenID Connect addition
    }

@app.get("/.well-known/openid-configuration")
async def openid_configuration():
    """OpenID Connect discovery endpoint"""
    return {
        "issuer": "https://your-domain.com",
        "authorization_endpoint": "https://your-domain.com/oauth/authorize",
        "token_endpoint": "https://your-domain.com/oauth/token",
        "userinfo_endpoint": "https://your-domain.com/oauth/userinfo",
        "jwks_uri": "https://your-domain.com/.well-known/jwks.json",
        "response_types_supported": ["code", "token", "id_token"],
        "subject_types_supported": ["public"],
        "id_token_signing_alg_values_supported": ["RS256"],
        "scopes_supported": ["openid", "profile", "email"]
    }

@app.get("/oauth/userinfo")
async def userinfo(authorization: str = Header(None)):
    """UserInfo endpoint"""
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(401, "Missing authorization")

    token = authorization.split(" ")[1]

    if token not in access_tokens:
        raise HTTPException(401, "Invalid token")

    token_data = access_tokens[token]
    user_id = token_data["user_id"]

    # Fetch user info
    return {
        "sub": user_id,
        "name": "John Doe",
        "email": "john@example.com",
        "email_verified": True,
        "picture": "https://example.com/avatar.jpg"
    }
```

---

## Social OAuth2 Integration

### 1. **Google OAuth2**

```python
import httpx

GOOGLE_CLIENT_ID = "your-google-client-id"
GOOGLE_CLIENT_SECRET = "your-google-client-secret"
GOOGLE_REDIRECT_URI = "http://localhost:8000/auth/google/callback"

@app.get("/auth/google/login")
async def google_login():
    """Redirect to Google OAuth"""
    google_auth_url = (
        "https://accounts.google.com/o/oauth2/v2/auth"
        f"?client_id={GOOGLE_CLIENT_ID}"
        f"&redirect_uri={GOOGLE_REDIRECT_URI}"
        "&response_type=code"
        "&scope=openid%20email%20profile"
    )
    return RedirectResponse(google_auth_url)

@app.get("/auth/google/callback")
async def google_callback(code: str):
    """Handle Google OAuth callback"""
    # Exchange code for token
    async with httpx.AsyncClient() as client:
        token_response = await client.post(
            "https://oauth2.googleapis.com/token",
            data={
                "code": code,
                "client_id": GOOGLE_CLIENT_ID,
                "client_secret": GOOGLE_CLIENT_SECRET,
                "redirect_uri": GOOGLE_REDIRECT_URI,
                "grant_type": "authorization_code"
            }
        )

        if token_response.status_code != 200:
            raise HTTPException(400, "Failed to get token")

        tokens = token_response.json()
        access_token = tokens["access_token"]
        id_token = tokens["id_token"]

        # Verify ID token
        user_info = jwt.decode(
            id_token,
            options={"verify_signature": False}  # Verify in production!
        )

        # Create or update user in your database
        user = await create_or_update_user(
            email=user_info["email"],
            name=user_info.get("name"),
            picture=user_info.get("picture")
        )

        # Create your own access token
        your_access_token = create_access_token(
            data={"sub": user.email, "user_id": user.id}
        )

        return {
            "access_token": your_access_token,
            "token_type": "bearer"
        }
```

### 2. **GitHub OAuth2**

```python
GITHUB_CLIENT_ID = "your-github-client-id"
GITHUB_CLIENT_SECRET = "your-github-client-secret"

@app.get("/auth/github/login")
async def github_login():
    """Redirect to GitHub OAuth"""
    github_auth_url = (
        "https://github.com/login/oauth/authorize"
        f"?client_id={GITHUB_CLIENT_ID}"
        "&scope=user:email"
    )
    return RedirectResponse(github_auth_url)

@app.get("/auth/github/callback")
async def github_callback(code: str):
    """Handle GitHub OAuth callback"""
    async with httpx.AsyncClient() as client:
        # Exchange code for token
        token_response = await client.post(
            "https://github.com/login/oauth/access_token",
            json={
                "client_id": GITHUB_CLIENT_ID,
                "client_secret": GITHUB_CLIENT_SECRET,
                "code": code
            },
            headers={"Accept": "application/json"}
        )

        tokens = token_response.json()
        access_token = tokens["access_token"]

        # Get user info
        user_response = await client.get(
            "https://api.github.com/user",
            headers={"Authorization": f"Bearer {access_token}"}
        )

        user_data = user_response.json()

        # Create user in your system
        your_token = create_access_token(
            data={
                "sub": user_data["login"],
                "user_id": user_data["id"]
            }
        )

        return {
            "access_token": your_token,
            "token_type": "bearer",
            "github_user": user_data["login"]
        }
```

---

## Best Practices

### ✅ Do's:

1. **Use HTTPS** always in production
2. **Validate redirect_uri** strictly
3. **Implement PKCE** for public clients
4. **Use short-lived tokens**
5. **Implement refresh tokens**
6. **Validate client credentials**
7. **Use state parameter** for CSRF protection
8. **Store tokens securely**

### ❌ Don'ts:

1. **Don't use implicit flow** (deprecated)
2. **Don't expose client_secret** in public clients
3. **Don't skip redirect_uri validation**
4. **Don't reuse authorization codes**
5. **Don't ignore token expiration**
6. **Don't log tokens**

---

## Interview Questions

### Q1: What's the difference between OAuth2 and OpenID Connect?

**Answer**:

- **OAuth2**: Authorization framework for delegated access
- **OpenID Connect**: Authentication layer on top of OAuth2, adds ID token with user identity information

### Q2: When should you use which OAuth2 grant type?

**Answer**:

- **Authorization Code**: Web apps with backend server (most secure)
- **Authorization Code + PKCE**: Mobile/SPA apps (public clients)
- **Client Credentials**: Server-to-server (no user involved)
- **Password**: Legacy, avoid if possible
- **Implicit**: Deprecated, don't use

### Q3: What is PKCE and why is it important?

**Answer**: Proof Key for Code Exchange prevents authorization code interception attacks in public clients (mobile/SPA). Uses code_verifier/code_challenge to ensure the client exchanging the code is the same one that requested it.

### Q4: How do you secure OAuth2 redirect_uri?

**Answer**:

- Store allowed redirect URIs in database
- Exact match validation (no wildcards)
- Validate on both authorization and token endpoints
- Use HTTPS only
- Implement state parameter for CSRF protection

### Q5: What's in an OpenID Connect ID token?

**Answer**: Standard claims include:

- `iss` (issuer)
- `sub` (subject/user ID)
- `aud` (audience/client ID)
- `exp` (expiration)
- `iat` (issued at)
- User info: email, name, picture, etc.

---

## Summary

OAuth2 and OpenID Connect provide:

- **Secure authorization** with multiple grant types
- **Social login** integration (Google, GitHub, etc.)
- **Identity verification** with OpenID Connect
- **Scalable architecture** for distributed systems
- **Industry standard** protocols

Properly implemented OAuth2/OIDC ensures your FastAPI application has enterprise-grade authentication and authorization! 🔒
