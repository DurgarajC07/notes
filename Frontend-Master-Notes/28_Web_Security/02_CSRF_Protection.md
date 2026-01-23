# 🔐 CSRF Protection (Cross-Site Request Forgery)

> CSRF attacks trick authenticated users into performing unwanted actions. Understanding CSRF is critical for building secure web applications that handle sensitive operations.

---

## 📖 1. Concept Explanation

### What is CSRF?

**CSRF (Cross-Site Request Forgery)** exploits the browser's automatic inclusion of cookies, tricking a user's browser into making authenticated requests to a vulnerable site.

**Attack scenario:**

```
1. User logs into bank.com (gets session cookie)
2. User visits evil.com (while still logged in)
3. evil.com has hidden form:
   <form action="https://bank.com/transfer" method="POST">
     <input name="to" value="attacker">
     <input name="amount" value="10000">
   </form>
   <script>document.forms[0].submit()</script>
4. Browser automatically sends bank.com cookies!
5. Bank transfers $10,000 to attacker
```

### How It Works

```html
<!-- evil.com -->
<!DOCTYPE html>
<html>
  <body>
    <h1>Cute Cats!</h1>

    <!-- Hidden malicious form -->
    <iframe style="display:none">
      <form action="https://bank.com/api/transfer" method="POST">
        <input name="toAccount" value="attacker123" />
        <input name="amount" value="5000" />
      </form>
      <script>
        document.forms[0].submit();
      </script>
    </iframe>
  </body>
</html>

<!-- Browser automatically includes bank.com cookies! -->
```

---

## 🧠 2. Why It Matters

### Real-World Impact

**1. Financial Loss:**

- Unauthorized transfers
- Fraudulent purchases
- Account changes

**2. Data Manipulation:**

- Delete user data
- Change email/password
- Modify settings

**3. Account Takeover:**

- Change security settings
- Add attacker as admin
- Disable 2FA

### Production Scenarios

**E-commerce site:**

```javascript
// ❌ VULNERABLE: No CSRF protection
app.post("/api/checkout", (req, res) => {
  const { items, shippingAddress } = req.body;
  const userId = req.session.userId;

  // Process order - but could be CSRF attack!
  processOrder(userId, items, shippingAddress);
});
```

**Social media:**

```javascript
// ❌ VULNERABLE: No CSRF protection
app.post("/api/posts", (req, res) => {
  const { content } = req.body;
  const userId = req.session.userId;

  // Post on behalf of user - could be CSRF!
  createPost(userId, content);
});
```

---

## ⚙️ 3. CSRF Protection Techniques

### 1. CSRF Tokens (Synchronizer Token Pattern)

**How it works:**

1. Server generates unique token per session
2. Embeds token in forms/AJAX headers
3. Validates token on state-changing requests

**Server-side (Express + csurf):**

```javascript
import csrf from "csurf";
import cookieParser from "cookie-parser";

const csrfProtection = csrf({ cookie: true });

app.use(cookieParser());
app.use(csrfProtection);

// GET: Send CSRF token to client
app.get("/form", (req, res) => {
  res.render("form", { csrfToken: req.csrfToken() });
});

// POST: Validate CSRF token
app.post("/api/transfer", (req, res) => {
  // Token validated by middleware
  processTransfer(req.body);
});
```

**Client-side (HTML form):**

```html
<form action="/api/transfer" method="POST">
  <!-- Hidden CSRF token -->
  <input type="hidden" name="_csrf" value="<%= csrfToken %>" />

  <input name="toAccount" placeholder="Account" />
  <input name="amount" placeholder="Amount" />
  <button type="submit">Transfer</button>
</form>
```

**Client-side (React AJAX):**

```jsx
function TransferForm() {
  const [csrfToken, setCsrfToken] = useState("");

  useEffect(() => {
    // Fetch CSRF token
    fetch("/api/csrf-token")
      .then((res) => res.json())
      .then((data) => setCsrfToken(data.csrfToken));
  }, []);

  async function handleSubmit(e) {
    e.preventDefault();

    await fetch("/api/transfer", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-CSRF-Token": csrfToken, // Include token
      },
      body: JSON.stringify({
        toAccount,
        amount,
      }),
    });
  }

  return <form onSubmit={handleSubmit}>...</form>;
}
```

---

### 2. SameSite Cookie Attribute

**Modern, browser-level CSRF protection:**

```javascript
// Server-side: Set SameSite attribute
res.cookie("session", sessionId, {
  httpOnly: true, // Prevent XSS
  secure: true, // HTTPS only
  sameSite: "strict", // CSRF protection
  maxAge: 3600000,
});
```

**SameSite values:**

| Value    | Behavior                                       |
| -------- | ---------------------------------------------- |
| `Strict` | Cookie NEVER sent cross-site (best protection) |
| `Lax`    | Cookie sent on top-level navigation (GET only) |
| `None`   | Cookie always sent (requires `Secure`)         |

**Examples:**

```javascript
// Strict: Strongest protection
sameSite: 'strict'

// User on evil.com clicks link to bank.com
// Cookie NOT sent → user must re-login

// Use for: High-security sites (banking, admin panels)

// Lax: Balanced (default in modern browsers)
sameSite: 'lax'

// GET request from evil.com → Cookie sent
// POST request from evil.com → Cookie NOT sent

// Use for: Most web applications

// None: No protection (legacy)
sameSite: 'none', secure: true

// Cookie always sent (even cross-site)
// Use for: Third-party integrations, OAuth
```

---

### 3. Double Submit Cookie Pattern

**Alternative to synchronizer tokens:**

```javascript
// Server-side: Generate random token
app.post("/login", (req, res) => {
  const sessionId = generateSessionId();
  const csrfToken = generateToken();

  // Set both as cookies
  res.cookie("session", sessionId, { httpOnly: true });
  res.cookie("csrf-token", csrfToken); // NOT httpOnly

  res.json({ csrfToken });
});

// Middleware: Validate token
app.post("/api/*", (req, res, next) => {
  const cookieToken = req.cookies["csrf-token"];
  const headerToken = req.headers["x-csrf-token"];

  if (cookieToken !== headerToken) {
    return res.status(403).json({ error: "CSRF validation failed" });
  }

  next();
});
```

**Client-side:**

```javascript
// Read CSRF token from cookie
function getCookie(name) {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  return parts.length === 2 ? parts.pop().split(";").shift() : null;
}

// Include in requests
const csrfToken = getCookie("csrf-token");

await fetch("/api/transfer", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-CSRF-Token": csrfToken,
  },
  body: JSON.stringify(data),
});
```

**Why it works:**

- Attacker can't read cookies from another domain (same-origin policy)
- Evil.com can't get csrf-token cookie value from bank.com
- Can't set matching X-CSRF-Token header

---

### 4. Custom Request Headers

**For AJAX/SPA applications:**

```javascript
// Server: Require custom header
app.post("/api/*", (req, res, next) => {
  const customHeader = req.headers["x-requested-with"];

  if (customHeader !== "XMLHttpRequest") {
    return res.status(403).json({ error: "Missing custom header" });
  }

  next();
});

// Client: Add custom header
await fetch("/api/transfer", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-Requested-With": "XMLHttpRequest",
  },
  body: JSON.stringify(data),
});
```

**Why it works:**

- Cross-origin requests can't set custom headers (unless CORS allows)
- Attacker can't make evil.com → bank.com request with custom headers
- Simple, effective for API-only applications

---

## ✅ 4. Best Practices

### DO ✅

```javascript
// 1. Use SameSite cookies (primary defense)
res.cookie("session", sessionId, {
  httpOnly: true,
  secure: true,
  sameSite: "strict", // or 'lax'
});

// 2. CSRF tokens for state-changing operations
app.post("/api/transfer", csrfProtection, (req, res) => {
  // Token validated by middleware
});

// 3. Validate Origin/Referer headers
app.post("/api/*", (req, res, next) => {
  const origin = req.headers.origin;
  const referer = req.headers.referer;

  if (!isAllowedOrigin(origin) && !isAllowedReferer(referer)) {
    return res.status(403).json({ error: "Invalid origin" });
  }

  next();
});

// 4. Require re-authentication for sensitive actions
app.post("/api/delete-account", (req, res) => {
  // Require password confirmation
  if (!verifyPassword(req.body.password)) {
    return res.status(401).json({ error: "Password required" });
  }

  deleteAccount(req.session.userId);
});

// 5. Use POST (not GET) for state changes
// ❌ WRONG
app.get("/api/delete/:id", (req, res) => {
  deleteItem(req.params.id);
});

// ✅ CORRECT
app.post("/api/delete/:id", (req, res) => {
  deleteItem(req.params.id);
});
```

### DON'T ❌

```javascript
// ❌ Don't rely on GET requests for state changes
app.get("/transfer", (req, res) => {
  transfer(req.query.to, req.query.amount);
});

// ❌ Don't skip CSRF for AJAX requests
// Myth: "AJAX is safe from CSRF"
// Reality: Attacker can make AJAX requests!

// ❌ Don't use only Referer header validation
// Can be spoofed or stripped by proxy

// ❌ Don't store sensitive tokens in localStorage
// Vulnerable to XSS

// ❌ Don't use weak token generation
function weakToken() {
  return Math.random().toString(); // Predictable!
}

// ✅ Use crypto.randomBytes
import crypto from "crypto";

function strongToken() {
  return crypto.randomBytes(32).toString("hex");
}
```

---

## ❌ 5. Common Mistakes

### Mistake #1: GET Requests for State Changes

```javascript
// ❌ VULNERABLE: State change via GET
app.get('/api/delete/:id', (req, res) => {
  deleteItem(req.params.id);
});

// Attacker can trigger with:
<img src="https://example.com/api/delete/123">

// ✅ SECURE: Use POST
app.post('/api/delete/:id', csrfProtection, (req, res) => {
  deleteItem(req.params.id);
});
```

---

### Mistake #2: Accepting JSON Without Validation

```javascript
// ❌ VULNERABLE: Accepts any content-type
app.post("/api/transfer", (req, res) => {
  const { to, amount } = req.body;
  transfer(to, amount);
});

// Attacker can use <form> POST with JSON

// ✅ SECURE: Validate content-type
app.post("/api/transfer", (req, res) => {
  if (req.headers["content-type"] !== "application/json") {
    return res.status(400).json({ error: "Invalid content-type" });
  }

  // Also validate CSRF token
  validateCsrfToken(req);

  const { to, amount } = req.body;
  transfer(to, amount);
});
```

---

### Mistake #3: Not Validating Token on Every Request

```javascript
// ❌ WRONG: Only validates on some endpoints
app.post("/api/transfer", csrfProtection, handler);
app.post("/api/delete", handler); // Missing protection!

// ✅ CORRECT: Validate on ALL state-changing requests
app.post("/api/*", csrfProtection);
```

---

## 🧪 6. Code Examples

### Complete CSRF Protection Setup

**Server-side (Node.js/Express):**

```javascript
import express from "express";
import csrf from "csurf";
import cookieParser from "cookie-parser";

const app = express();
const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: true, // HTTPS only
    sameSite: "strict",
  },
});

app.use(express.json());
app.use(cookieParser());

// CSRF token endpoint
app.get("/api/csrf-token", csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});

// Protected endpoint
app.post("/api/transfer", csrfProtection, (req, res) => {
  try {
    const { toAccount, amount } = req.body;
    const userId = req.session.userId;

    // Process transfer
    const result = processTransfer(userId, toAccount, amount);

    res.json({ success: true, result });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Error handler for CSRF failures
app.use((err, req, res, next) => {
  if (err.code === "EBADCSRFTOKEN") {
    res.status(403).json({ error: "CSRF validation failed" });
  } else {
    next(err);
  }
});
```

**Client-side (React):**

```tsx
import { useState, useEffect } from "react";

function TransferForm() {
  const [csrfToken, setCsrfToken] = useState("");
  const [toAccount, setToAccount] = useState("");
  const [amount, setAmount] = useState("");
  const [error, setError] = useState("");

  // Fetch CSRF token on mount
  useEffect(() => {
    async function fetchToken() {
      try {
        const res = await fetch("/api/csrf-token");
        const data = await res.json();
        setCsrfToken(data.csrfToken);
      } catch (error) {
        setError("Failed to load form");
      }
    }

    fetchToken();
  }, []);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError("");

    try {
      const res = await fetch("/api/transfer", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "X-CSRF-Token": csrfToken,
        },
        body: JSON.stringify({
          toAccount,
          amount: parseFloat(amount),
        }),
      });

      if (!res.ok) {
        const data = await res.json();
        throw new Error(data.error);
      }

      alert("Transfer successful!");
      setToAccount("");
      setAmount("");
    } catch (error) {
      setError(error.message);
    }
  }

  if (!csrfToken) {
    return <div>Loading...</div>;
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={toAccount}
        onChange={(e) => setToAccount(e.target.value)}
        placeholder="To Account"
        required
      />

      <input
        type="number"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        placeholder="Amount"
        required
      />

      <button type="submit">Transfer</button>

      {error && <div className="error">{error}</div>}
    </form>
  );
}
```

---

## 🏗️ 7. Real-World Use Cases

### Banking Application

```javascript
// Multi-layer CSRF protection
app.post("/api/transfer", [
  // Layer 1: SameSite cookies
  (req, res, next) => {
    // Automatically protected by SameSite=strict
    next();
  },

  // Layer 2: CSRF token
  csrfProtection,

  // Layer 3: Origin validation
  (req, res, next) => {
    const origin = req.headers.origin;
    if (origin !== "https://bank.com") {
      return res.status(403).json({ error: "Invalid origin" });
    }
    next();
  },

  // Layer 4: Re-authentication for large amounts
  (req, res, next) => {
    if (req.body.amount > 10000) {
      if (!req.body.password || !verifyPassword(req.body.password)) {
        return res
          .status(401)
          .json({ error: "Password required for large transfers" });
      }
    }
    next();
  },

  // Handler
  (req, res) => {
    processTransfer(req.body);
    res.json({ success: true });
  },
]);
```

---

## ❓ 8. Interview Questions

### Q1: What is CSRF and how does it work?

**Answer:**

CSRF tricks a user's browser into making authenticated requests to a vulnerable site while the user is logged in.

**Attack flow:**

1. User logs into bank.com (session cookie set)
2. User visits evil.com
3. evil.com contains hidden malicious request
4. Browser automatically includes bank.com cookies
5. Bank processes request as legitimate

**Example:**

```html
<!-- evil.com -->
<img src="https://bank.com/transfer?to=attacker&amount=5000" />
<!-- Browser sends bank.com cookies automatically! -->
```

---

### Q2: How do you prevent CSRF attacks?

**Answer:**

**Defense in depth:**

1. **SameSite cookies (primary):**

```javascript
res.cookie("session", sessionId, {
  sameSite: "strict", // or 'lax'
});
```

2. **CSRF tokens:**

```javascript
// Server generates token, client includes in requests
headers: { 'X-CSRF-Token': token }
```

3. **Custom headers:**

```javascript
// Require custom header (can't be set cross-origin)
headers: { 'X-Requested-With': 'XMLHttpRequest' }
```

4. **Origin/Referer validation:**

```javascript
if (req.headers.origin !== "https://mysite.com") {
  return res.status(403).json({ error: "Invalid origin" });
}
```

5. **Re-authentication for sensitive actions:**

```javascript
if (sensitiveAction) {
  requirePasswordConfirmation();
}
```

---

## 🎯 Summary

CSRF exploits browser's automatic cookie inclusion:

- **Attacker:** Tricks user into making authenticated request
- **Impact:** Unauthorized actions (transfers, deletions, etc.)
- **Prevention:** SameSite cookies + CSRF tokens + validation

**Production checklist:**

- ✅ Set SameSite=Strict/Lax on cookies
- ✅ Use CSRF tokens for state-changing operations
- ✅ POST (not GET) for state changes
- ✅ Validate Origin/Referer headers
- ✅ Require re-authentication for sensitive actions
- ✅ Custom headers for AJAX requests
- ✅ Test with security scanner

**Defense in depth:** Multiple layers of protection! 🛡️
