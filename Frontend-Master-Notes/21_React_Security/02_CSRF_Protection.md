# CSRF Protection in React

## Core Concept

Cross-Site Request Forgery (CSRF) tricks authenticated users into performing unwanted actions. Attackers exploit the browser's automatic cookie sending to make unauthorized requests.

---

## How CSRF Works

```typescript
// User is logged into bank.com (has auth cookie)
// User visits evil.com which contains:

<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker" />
  <input name="amount" value="10000" />
</form>
<script>document.forms[0].submit();</script>

// Browser automatically sends bank.com cookies!
// Transfer executes with user's credentials 💀
```

---

## CSRF Tokens

Server generates unique token for each session:

```typescript
// Backend (Express)
import csrf from 'csurf';

const csrfProtection = csrf({ cookie: true });

app.get('/api/form', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});

app.post('/api/transfer', csrfProtection, (req, res) => {
  // Token validated automatically
  // If invalid, returns 403
  performTransfer(req.body);
  res.json({ success: true });
});

// React frontend
function TransferForm() {
  const [csrfToken, setCsrfToken] = useState('');

  useEffect(() => {
    fetch('/api/form')
      .then(res => res.json())
      .then(data => setCsrfToken(data.csrfToken));
  }, []);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    await fetch('/api/transfer', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'CSRF-Token': csrfToken // Include token
      },
      credentials: 'include', // Send cookies
      body: JSON.stringify({ to: 'alice', amount: 100 })
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* Hidden field alternative */}
      <input type="hidden" name="_csrf" value={csrfToken} />
      <input name="amount" />
      <button>Transfer</button>
    </form>
  );
}
```

---

## SameSite Cookies

Modern browsers support SameSite attribute:

```typescript
// Backend - Set SameSite cookie
app.use(
  session({
    secret: "secret",
    cookie: {
      httpOnly: true,
      secure: true, // HTTPS only
      sameSite: "strict", // or 'lax'
      maxAge: 3600000,
    },
  }),
);

// SameSite values:
// - 'strict': Never sent on cross-site requests
// - 'lax': Sent on top-level navigation (GET only)
// - 'none': Always sent (requires secure: true)
```

### **SameSite=Strict**

```typescript
// User clicks link from external site to your app
// Session cookie NOT sent on first request
// User appears logged out initially
// Good for: Banking, sensitive operations
```

### **SameSite=Lax** (Recommended)

```typescript
// User clicks link from external site
// Cookie sent on top-level GET requests
// NOT sent on POST/PUT/DELETE
// Good for: Most applications
```

---

## Double Submit Cookie

Store token in both cookie and request header:

```typescript
// Backend
app.get('/api/data', (req, res) => {
  const token = generateToken();

  // Set in cookie
  res.cookie('XSRF-TOKEN', token, {
    httpOnly: false, // JavaScript can read
    sameSite: 'strict'
  });

  res.json({ data: '...' });
});

app.post('/api/action', (req, res) => {
  const cookieToken = req.cookies['XSRF-TOKEN'];
  const headerToken = req.headers['x-xsrf-token'];

  if (cookieToken !== headerToken) {
    return res.status(403).json({ error: 'Invalid CSRF token' });
  }

  // Process request
});

// React frontend
import Cookies from 'js-cookie';

function Component() {
  const handleAction = async () => {
    const token = Cookies.get('XSRF-TOKEN');

    await fetch('/api/action', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-XSRF-TOKEN': token // Match cookie value
      },
      body: JSON.stringify({ ... })
    });
  };

  return <button onClick={handleAction}>Action</button>;
}
```

---

## Axios with CSRF

Axios automatically handles XSRF tokens:

```typescript
import axios from 'axios';

// Configure Axios
const api = axios.create({
  baseURL: '/api',
  withCredentials: true, // Send cookies
  xsrfCookieName: 'XSRF-TOKEN', // Cookie name
  xsrfHeaderName: 'X-XSRF-TOKEN' // Header name
});

// Axios automatically:
// 1. Reads XSRF-TOKEN cookie
// 2. Sends it in X-XSRF-TOKEN header

function Component() {
  const handleSubmit = async (data: FormData) => {
    await api.post('/transfer', data);
    // Token sent automatically!
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

---

## Custom Hooks for CSRF

```typescript
import { useState, useEffect } from 'react';

interface CSRFHook {
  token: string | null;
  loading: boolean;
  error: Error | null;
}

function useCSRFToken(): CSRFHook {
  const [token, setToken] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    fetch('/api/csrf-token', { credentials: 'include' })
      .then(res => res.json())
      .then(data => {
        setToken(data.token);
        setLoading(false);
      })
      .catch(err => {
        setError(err);
        setLoading(false);
      });
  }, []);

  return { token, loading, error };
}

// Usage
function TransferForm() {
  const { token, loading } = useCSRFToken();

  if (loading) return <Spinner />;

  const handleSubmit = async (data: FormData) => {
    await fetch('/api/transfer', {
      method: 'POST',
      headers: { 'CSRF-Token': token! },
      body: data
    });
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

---

## Origin Header Validation

```typescript
// Backend - Validate Origin header
app.use((req, res, next) => {
  const origin = req.headers.origin;
  const allowedOrigins = ["https://example.com", "https://app.example.com"];

  if (req.method !== "GET" && req.method !== "HEAD") {
    if (!origin || !allowedOrigins.includes(origin)) {
      return res.status(403).json({ error: "Invalid origin" });
    }
  }

  next();
});
```

---

## Referer Header Validation

```typescript
// Backend
app.post("/api/transfer", (req, res) => {
  const referer = req.headers.referer;

  if (!referer || !referer.startsWith("https://example.com")) {
    return res.status(403).json({ error: "Invalid referer" });
  }

  // Process request
});
```

---

## Real-World Example: Banking App

```typescript
// Backend
import express from 'express';
import csrf from 'csurf';
import cookieParser from 'cookie-parser';

const app = express();
app.use(cookieParser());

const csrfProtection = csrf({
  cookie: {
    httpOnly: false,
    sameSite: 'strict',
    secure: true
  }
});

// Get CSRF token
app.get('/api/csrf', csrfProtection, (req, res) => {
  res.json({ token: req.csrfToken() });
});

// Protected route
app.post('/api/transfer', csrfProtection, (req, res) => {
  const { to, amount } = req.body;

  // Validate
  if (!to || !amount) {
    return res.status(400).json({ error: 'Missing fields' });
  }

  // Perform transfer
  bankService.transfer(req.user.id, to, amount);

  res.json({ success: true });
});

// React frontend
import { useState, useEffect } from 'react';

function BankTransfer() {
  const [csrf, setCsrf] = useState('');
  const [to, setTo] = useState('');
  const [amount, setAmount] = useState('');

  // Fetch CSRF token on mount
  useEffect(() => {
    fetch('/api/csrf', { credentials: 'include' })
      .then(res => res.json())
      .then(data => setCsrf(data.token));
  }, []);

  const handleTransfer = async (e: React.FormEvent) => {
    e.preventDefault();

    const response = await fetch('/api/transfer', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'CSRF-Token': csrf // Send token
      },
      credentials: 'include', // Send cookies
      body: JSON.stringify({ to, amount })
    });

    if (response.ok) {
      alert('Transfer successful!');
    } else {
      alert('Transfer failed - CSRF validation error');
    }
  };

  return (
    <form onSubmit={handleTransfer}>
      <input
        placeholder="Recipient"
        value={to}
        onChange={e => setTo(e.target.value)}
      />
      <input
        type="number"
        placeholder="Amount"
        value={amount}
        onChange={e => setAmount(e.target.value)}
      />
      <button type="submit">Transfer</button>
    </form>
  );
}
```

---

## REST API Pattern

```typescript
// API client with CSRF
class APIClient {
  private csrfToken: string | null = null;

  async init() {
    const res = await fetch('/api/csrf-token');
    const data = await res.json();
    this.csrfToken = data.token;
  }

  async post(url: string, body: any) {
    if (!this.csrfToken) {
      await this.init();
    }

    return fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': this.csrfToken!
      },
      credentials: 'include',
      body: JSON.stringify(body)
    });
  }
}

const api = new APIClient();

// Usage
function Component() {
  const handleAction = async () => {
    await api.post('/api/action', { data: '...' });
  };

  return <button onClick={handleAction}>Action</button>;
}
```

---

## Best Practices

✅ **Use SameSite=Lax** for cookies (default in modern browsers)  
✅ **Implement CSRF tokens** for state-changing operations  
✅ **Use HTTPS only** - secure cookies  
✅ **Validate Origin/Referer** headers  
✅ **Use POST/PUT/DELETE** for mutations (not GET)  
✅ **Set httpOnly** for session cookies  
❌ **Don't rely on CSRF tokens alone** - defense in depth  
❌ **Don't use GET** for state-changing operations  
❌ **Don't skip CSRF** for "internal" APIs

---

## Security Checklist

- [ ] SameSite cookies enabled
- [ ] CSRF tokens for POST/PUT/DELETE
- [ ] Cookies set to httpOnly and secure
- [ ] Origin/Referer validation
- [ ] No state changes in GET requests
- [ ] HTTPS enforced
- [ ] Token rotation on sensitive actions

---

## Key Takeaways

1. **CSRF exploits automatic cookie sending**
2. **SameSite=Lax** prevents most CSRF attacks
3. **CSRF tokens** validate request origin
4. **Double submit cookie** is simpler alternative
5. **Axios handles XSRF** automatically
6. **Always use HTTPS** for secure cookies
7. **Validate Origin/Referer** as additional layer
