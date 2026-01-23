# 🔐 JWT Authentication Deep Dive

> JSON Web Tokens (JWT) are the standard for stateless authentication in modern web applications. Understanding JWTs is essential for implementing secure authentication and authorization.

---

## 📖 1. Concept Explanation

### What is JWT?

**JWT is a self-contained token that securely transmits information between parties as a JSON object:**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

         HEADER              .          PAYLOAD           .       SIGNATURE
```

---

### JWT Structure

**1. Header (Algorithm + Type):**

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**2. Payload (Claims):**

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622
}
```

**3. Signature:**

```javascript
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret);
```

---

## 🧠 2. Why It Matters

### Session-based vs JWT

**Session-based (Stateful):**

```
Client → Server (login) → Server stores session in DB
Client stores session ID in cookie
Every request: Client sends session ID → Server queries DB
```

**JWT (Stateless):**

```
Client → Server (login) → Server creates JWT
Client stores JWT (localStorage/cookie)
Every request: Client sends JWT → Server verifies signature (no DB lookup!)
```

**JWT benefits:**

- ✅ Stateless (no server-side session storage)
- ✅ Scalable (no session DB lookup)
- ✅ Cross-domain (CORS-friendly)
- ✅ Mobile-friendly (no cookies required)

---

## ⚙️ 3. JWT Implementation

### Server-side (Node.js/Express)

```javascript
const jwt = require("jsonwebtoken");
const bcrypt = require("bcrypt");

const ACCESS_TOKEN_SECRET = process.env.ACCESS_TOKEN_SECRET;
const REFRESH_TOKEN_SECRET = process.env.REFRESH_TOKEN_SECRET;

// Login endpoint
app.post("/api/login", async (req, res) => {
  const { email, password } = req.body;

  // 1. Find user
  const user = await User.findOne({ email });
  if (!user) {
    return res.status(401).json({ error: "Invalid credentials" });
  }

  // 2. Verify password
  const isValidPassword = await bcrypt.compare(password, user.password);
  if (!isValidPassword) {
    return res.status(401).json({ error: "Invalid credentials" });
  }

  // 3. Create access token (short-lived)
  const accessToken = jwt.sign(
    {
      userId: user.id,
      email: user.email,
      role: user.role,
    },
    ACCESS_TOKEN_SECRET,
    { expiresIn: "15m" }, // 15 minutes
  );

  // 4. Create refresh token (long-lived)
  const refreshToken = jwt.sign(
    { userId: user.id },
    REFRESH_TOKEN_SECRET,
    { expiresIn: "7d" }, // 7 days
  );

  // 5. Store refresh token in DB (for revocation)
  await RefreshToken.create({
    userId: user.id,
    token: refreshToken,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
  });

  // 6. Send tokens
  res.json({
    accessToken,
    refreshToken,
    user: {
      id: user.id,
      email: user.email,
      name: user.name,
    },
  });
});

// Refresh token endpoint
app.post("/api/refresh", async (req, res) => {
  const { refreshToken } = req.body;

  if (!refreshToken) {
    return res.status(401).json({ error: "Refresh token required" });
  }

  try {
    // 1. Verify refresh token
    const decoded = jwt.verify(refreshToken, REFRESH_TOKEN_SECRET);

    // 2. Check if token exists in DB (not revoked)
    const storedToken = await RefreshToken.findOne({
      userId: decoded.userId,
      token: refreshToken,
    });

    if (!storedToken) {
      return res.status(401).json({ error: "Invalid refresh token" });
    }

    // 3. Create new access token
    const user = await User.findById(decoded.userId);
    const newAccessToken = jwt.sign(
      {
        userId: user.id,
        email: user.email,
        role: user.role,
      },
      ACCESS_TOKEN_SECRET,
      { expiresIn: "15m" },
    );

    res.json({ accessToken: newAccessToken });
  } catch (error) {
    res.status(401).json({ error: "Invalid refresh token" });
  }
});

// Logout endpoint
app.post("/api/logout", async (req, res) => {
  const { refreshToken } = req.body;

  // Revoke refresh token
  await RefreshToken.deleteOne({ token: refreshToken });

  res.json({ message: "Logged out" });
});
```

---

### Authentication Middleware

```javascript
function authenticateToken(req, res, next) {
  // 1. Get token from Authorization header
  const authHeader = req.headers["authorization"];
  const token = authHeader && authHeader.split(" ")[1]; // "Bearer TOKEN"

  if (!token) {
    return res.status(401).json({ error: "Access token required" });
  }

  // 2. Verify token
  jwt.verify(token, ACCESS_TOKEN_SECRET, (err, decoded) => {
    if (err) {
      // Token expired or invalid
      return res.status(403).json({ error: "Invalid or expired token" });
    }

    // 3. Attach user to request
    req.user = decoded;
    next();
  });
}

// Protected route
app.get("/api/profile", authenticateToken, async (req, res) => {
  const user = await User.findById(req.user.userId);
  res.json(user);
});

// Role-based authorization
function authorize(...roles) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: "Insufficient permissions" });
    }
    next();
  };
}

// Admin-only route
app.delete(
  "/api/users/:id",
  authenticateToken,
  authorize("admin"),
  async (req, res) => {
    await User.deleteOne({ _id: req.params.id });
    res.json({ message: "User deleted" });
  },
);
```

---

### Client-side (React)

```typescript
// API client with interceptors
import axios from 'axios';

const api = axios.create({
  baseURL: 'https://api.example.com'
});

// Request interceptor: Add access token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor: Refresh token on 401
let isRefreshing = false;
let failedQueue: any[] = [];

const processQueue = (error: any, token: string | null = null) => {
  failedQueue.forEach((prom) => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token);
    }
  });

  failedQueue = [];
};

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Wait for token refresh
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        })
          .then((token) => {
            originalRequest.headers.Authorization = `Bearer ${token}`;
            return api(originalRequest);
          })
          .catch((err) => Promise.reject(err));
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        // Refresh token
        const refreshToken = localStorage.getItem('refreshToken');
        const { data } = await axios.post('/api/refresh', { refreshToken });

        const newAccessToken = data.accessToken;
        localStorage.setItem('accessToken', newAccessToken);

        api.defaults.headers.common.Authorization = `Bearer ${newAccessToken}`;
        originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;

        processQueue(null, newAccessToken);

        return api(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);

        // Refresh failed → logout
        localStorage.removeItem('accessToken');
        localStorage.removeItem('refreshToken');
        window.location.href = '/login';

        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);

// Auth context
interface AuthContextType {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    // Load user on mount
    const loadUser = async () => {
      const token = localStorage.getItem('accessToken');
      if (token) {
        try {
          const { data } = await api.get('/api/profile');
          setUser(data);
        } catch (error) {
          localStorage.removeItem('accessToken');
          localStorage.removeItem('refreshToken');
        }
      }
    };

    loadUser();
  }, []);

  const login = async (email: string, password: string) => {
    const { data } = await api.post('/api/login', { email, password });

    localStorage.setItem('accessToken', data.accessToken);
    localStorage.setItem('refreshToken', data.refreshToken);
    setUser(data.user);
  };

  const logout = async () => {
    const refreshToken = localStorage.getItem('refreshToken');

    try {
      await api.post('/api/logout', { refreshToken });
    } finally {
      localStorage.removeItem('accessToken');
      localStorage.removeItem('refreshToken');
      setUser(null);
    }
  };

  return (
    <AuthContext.Provider value={{
      user,
      login,
      logout,
      isAuthenticated: !!user
    }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

// Protected route component
function ProtectedRoute({ children }: { children: ReactNode }) {
  const { isAuthenticated } = useAuth();

  if (!isAuthenticated) {
    return <Navigate to="/login" />;
  }

  return <>{children}</>;
}
```

---

## 🔒 4. Security Best Practices

### 1. Token Storage

```typescript
// ❌ localStorage (vulnerable to XSS)
localStorage.setItem("accessToken", token);

// ✅ httpOnly cookie (protected from XSS)
// Server sets cookie:
res.cookie("accessToken", token, {
  httpOnly: true, // Not accessible via JavaScript
  secure: true, // HTTPS only
  sameSite: "strict", // CSRF protection
  maxAge: 15 * 60 * 1000, // 15 minutes
});

// Client: Cookie sent automatically
fetch("/api/profile", {
  credentials: "include",
});
```

**Comparison:**

| Storage             | XSS Vulnerable? | CSRF Vulnerable?          | Cross-domain?     |
| ------------------- | --------------- | ------------------------- | ----------------- |
| **localStorage**    | ✅ Yes          | ❌ No                     | ✅ Yes            |
| **httpOnly cookie** | ❌ No           | ✅ Yes (without SameSite) | ❌ No (same-site) |

**Best practice:**

- **Access token:** httpOnly cookie with SameSite
- **Refresh token:** httpOnly cookie, separate domain

---

### 2. Token Expiration

```javascript
// ❌ Long-lived access tokens (security risk)
const token = jwt.sign(payload, secret, { expiresIn: "30d" });

// ✅ Short-lived access + long-lived refresh
const accessToken = jwt.sign(payload, secret, { expiresIn: "15m" });
const refreshToken = jwt.sign(payload, secret, { expiresIn: "7d" });
```

**Strategy:**

- **Access token:** 15 minutes (minimize damage if stolen)
- **Refresh token:** 7 days (stored in DB for revocation)

---

### 3. Token Revocation

```javascript
// Blacklist approach (not recommended for scale)
const blacklist = new Set();

function revokeToken(token) {
  blacklist.add(token);
}

function isRevoked(token) {
  return blacklist.has(token);
}

// ✅ Refresh token rotation (better)
app.post("/api/refresh", async (req, res) => {
  const { refreshToken } = req.body;

  // 1. Verify and delete old refresh token
  await RefreshToken.deleteOne({ token: refreshToken });

  // 2. Create NEW refresh token
  const newRefreshToken = jwt.sign({ userId }, secret, { expiresIn: "7d" });

  // 3. Store new refresh token
  await RefreshToken.create({
    userId,
    token: newRefreshToken,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
  });

  res.json({
    accessToken: newAccessToken,
    refreshToken: newRefreshToken,
  });
});

// Logout revokes all user's refresh tokens
app.post("/api/logout-all", async (req, res) => {
  await RefreshToken.deleteMany({ userId: req.user.userId });
  res.json({ message: "All sessions logged out" });
});
```

---

### 4. Secure Secrets

```javascript
// ❌ Hardcoded secrets
const SECRET = "mysecret";

// ✅ Environment variables
const SECRET = process.env.JWT_SECRET;

// ✅ Different secrets for access/refresh
const ACCESS_SECRET = process.env.ACCESS_TOKEN_SECRET;
const REFRESH_SECRET = process.env.REFRESH_TOKEN_SECRET;

// ✅ Rotate secrets periodically
// Use key versioning:
const SECRETS = {
  v1: process.env.JWT_SECRET_V1,
  v2: process.env.JWT_SECRET_V2, // New secret
};

function signToken(payload) {
  return jwt.sign({ ...payload, keyVersion: "v2" }, SECRETS.v2, {
    expiresIn: "15m",
  });
}

function verifyToken(token) {
  const decoded = jwt.decode(token);
  const secret = SECRETS[decoded.keyVersion] || SECRETS.v1;
  return jwt.verify(token, secret);
}
```

---

## ❓ 5. Interview Questions

### Q1: JWT vs Session-based authentication?

**Answer:**

| Feature          | JWT                        | Session               |
| ---------------- | -------------------------- | --------------------- |
| **Storage**      | Client (token)             | Server (session DB)   |
| **Scalability**  | High (stateless)           | Lower (DB lookups)    |
| **Revocation**   | Hard (requires DB)         | Easy (delete session) |
| **Size**         | Larger (all data in token) | Smaller (session ID)  |
| **Cross-domain** | Easy                       | Hard (cookies)        |

**JWT pros:**

- Stateless (no session DB)
- Scalable (no server-side storage)
- Cross-domain friendly

**JWT cons:**

- Hard to revoke (must blacklist)
- Larger payload
- XSS risk (if stored in localStorage)

**Session pros:**

- Easy revocation
- Smaller payload
- More control

**Session cons:**

- Requires session storage (DB/Redis)
- DB lookup every request
- Cross-domain issues

---

### Q2: How to securely store JWTs?

**Answer:**

**Options:**

**1. httpOnly Cookie (BEST):**

```javascript
// Server
res.cookie("token", jwt, {
  httpOnly: true, // Not accessible via JS (XSS protection)
  secure: true, // HTTPS only
  sameSite: "strict", // CSRF protection
});

// Client: Automatic
fetch("/api/data", { credentials: "include" });
```

**Pros:** XSS-safe, automatic  
**Cons:** CSRF risk (mitigate with SameSite)

**2. localStorage (VULNERABLE):**

```javascript
localStorage.setItem("token", jwt);
```

**Pros:** Easy, cross-domain  
**Cons:** XSS vulnerability!

**3. Memory Only (SECURE but inconvenient):**

```javascript
let token = null; // In-memory variable

// Lost on page refresh!
```

**Best practice:**

- Use **httpOnly cookie** with **SameSite=strict**
- Use **HTTPS only** (secure flag)
- Implement **CSRF tokens** for extra protection

---

### Q3: How does JWT refresh token flow work?

**Answer:**

**Flow:**

```
1. Login:
   Client → POST /login { email, password }
   Server → { accessToken (15m), refreshToken (7d) }
   Client stores both

2. API request:
   Client → GET /api/data { Authorization: "Bearer accessToken" }
   Server verifies accessToken → Response

3. Access token expires:
   Client → GET /api/data
   Server → 401 Unauthorized

4. Refresh:
   Client → POST /refresh { refreshToken }
   Server verifies refreshToken → { newAccessToken }
   Client stores newAccessToken

5. Retry API request:
   Client → GET /api/data { Authorization: "Bearer newAccessToken" }
   Server → Response
```

**Refresh token rotation:**

```javascript
// Each refresh returns NEW refresh token
POST /refresh { refreshToken: "old" }
Response: { accessToken: "new", refreshToken: "new_refresh" }

// Old refresh token invalidated
```

**Benefits:**

- Short-lived access tokens (15min)
- Compromise limited damage
- Long-lived refresh tokens (7d)
- Can be revoked

---

## 🎯 Summary

**JWT structure:**

- Header (algorithm + type)
- Payload (claims)
- Signature (verification)

**Token strategy:**

- Access token: 15min (short-lived)
- Refresh token: 7 days (long-lived, revocable)

**Security:**

- ✅ httpOnly cookies (XSS protection)
- ✅ SameSite cookies (CSRF protection)
- ✅ HTTPS only (secure flag)
- ✅ Short expiration
- ✅ Refresh token rotation
- ❌ Don't use localStorage (XSS risk)

**Best practices:**

- Separate access/refresh secrets
- Store refresh tokens in DB
- Implement token rotation
- Use environment variables for secrets

Master JWT for secure stateless authentication! 🔐
