# 🔒 CORS (Cross-Origin Resource Sharing)

> CORS is a security mechanism that allows controlled access to resources from different origins. Understanding CORS is essential for building secure APIs and preventing unauthorized cross-origin requests.

---

## 📖 1. Concept Explanation

### What is CORS?

**Cross-Origin Resource Sharing (CORS)** is a browser security feature that:

1. **Blocks cross-origin requests by default**
2. **Uses HTTP headers to allow specific origins**
3. **Requires preflight for certain requests**
4. **Protects against unauthorized API access**

**Origin:**

```
https://example.com:443
   ↓       ↓        ↓
protocol domain   port

Same origin: https://example.com:443/path (✅)
Different origin: http://example.com (❌ protocol)
Different origin: https://api.example.com (❌ subdomain)
Different origin: https://example.com:8080 (❌ port)
```

---

## 🧠 2. Why It Matters

### Without CORS

```javascript
// Malicious site: evil.com
fetch("https://bank.com/api/transfer", {
  method: "POST",
  credentials: "include", // Send cookies!
  body: JSON.stringify({
    to: "attacker",
    amount: 10000,
  }),
});

// Without CORS, this would work! 😱
// Browser would send authentication cookies to bank.com
```

---

### With CORS

```javascript
// Browser blocks request:
// Access to fetch at 'https://bank.com/api/transfer' from origin
// 'https://evil.com' has been blocked by CORS policy: No
// 'Access-Control-Allow-Origin' header is present on the requested resource.

// Only allowed origins can make requests ✅
```

---

## ⚙️ 3. CORS Headers

### 1. Access-Control-Allow-Origin

**Specifies which origins can access the resource:**

```javascript
// Express.js example
app.use((req, res, next) => {
  // ❌ Dangerous: Allow all origins
  res.header("Access-Control-Allow-Origin", "*");

  // ✅ Safe: Specific origin
  res.header("Access-Control-Allow-Origin", "https://trusted-site.com");

  // ✅ Dynamic: Whitelist
  const allowedOrigins = [
    "https://app.example.com",
    "https://admin.example.com",
  ];

  const origin = req.headers.origin;
  if (origin && allowedOrigins.includes(origin)) {
    res.header("Access-Control-Allow-Origin", origin);
  }

  next();
});
```

**Security note:**

```javascript
// ❌ Never do this with credentials!
res.header("Access-Control-Allow-Origin", "*");
res.header("Access-Control-Allow-Credentials", "true");
// This combination is rejected by browsers
```

---

### 2. Access-Control-Allow-Methods

**Specifies allowed HTTP methods:**

```javascript
res.header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");

// Or specific methods only
res.header("Access-Control-Allow-Methods", "GET, POST");
```

---

### 3. Access-Control-Allow-Headers

**Specifies allowed request headers:**

```javascript
res.header("Access-Control-Allow-Headers", "Content-Type, Authorization");

// Dynamic (echo requested headers)
const requestedHeaders = req.headers["access-control-request-headers"];
if (requestedHeaders) {
  res.header("Access-Control-Allow-Headers", requestedHeaders);
}
```

---

### 4. Access-Control-Allow-Credentials

**Allows cookies/auth headers in cross-origin requests:**

```javascript
res.header("Access-Control-Allow-Credentials", "true");

// Client-side
fetch("https://api.example.com/data", {
  credentials: "include", // Send cookies
});
```

**Requirements:**

- `Access-Control-Allow-Origin` must be specific (not `*`)
- `Access-Control-Allow-Credentials: true` on server
- `credentials: 'include'` on client

---

### 5. Access-Control-Max-Age

**Cache preflight response:**

```javascript
// Cache preflight for 1 hour (3600 seconds)
res.header("Access-Control-Max-Age", "3600");

// Browser won't send preflight again for 1 hour
```

---

### 6. Access-Control-Expose-Headers

**Expose custom headers to client:**

```javascript
// Server
res.header("X-Total-Count", "100");
res.header("Access-Control-Expose-Headers", "X-Total-Count");

// Client can access:
fetch("/api/items").then((res) => {
  const total = res.headers.get("X-Total-Count"); // Works!
});
```

**Default exposed headers:**

- Cache-Control
- Content-Language
- Content-Type
- Expires
- Last-Modified
- Pragma

---

## ⚙️ 4. Preflight Requests

### Simple Requests (No Preflight)

**Criteria:**

1. Method: `GET`, `HEAD`, or `POST`
2. Headers: Only simple headers (Content-Type, Accept, etc.)
3. Content-Type: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`

```javascript
// Simple request (no preflight)
fetch("https://api.example.com/data", {
  method: "GET",
});

// 1 request sent:
// GET /data
```

---

### Preflighted Requests

**Triggers preflight if:**

- Method: `PUT`, `DELETE`, `PATCH`
- Custom headers: `Authorization`, `X-Custom-Header`
- Content-Type: `application/json`

```javascript
// Preflighted request
fetch("https://api.example.com/data", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: "Bearer token",
  },
  body: JSON.stringify({ data: "value" }),
});

// 2 requests sent:
// 1. OPTIONS /data (preflight)
// 2. POST /data (actual request)
```

---

### Preflight Request/Response

**Preflight request:**

```http
OPTIONS /api/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```

**Preflight response:**

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600
```

**If successful, browser sends actual request:**

```http
POST /api/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Content-Type: application/json
Authorization: Bearer token

{"data": "value"}
```

---

## ✅ 5. CORS Middleware Implementation

### Express.js

```javascript
const express = require("express");
const app = express();

// Option 1: Manual middleware
app.use((req, res, next) => {
  const allowedOrigins = [
    "https://app.example.com",
    "https://admin.example.com",
    process.env.NODE_ENV === "development" && "http://localhost:3000",
  ].filter(Boolean);

  const origin = req.headers.origin;

  if (origin && allowedOrigins.includes(origin)) {
    res.header("Access-Control-Allow-Origin", origin);
    res.header("Access-Control-Allow-Credentials", "true");
    res.header(
      "Access-Control-Allow-Methods",
      "GET, POST, PUT, DELETE, OPTIONS",
    );
    res.header("Access-Control-Allow-Headers", "Content-Type, Authorization");
    res.header("Access-Control-Max-Age", "3600");
  }

  // Handle preflight
  if (req.method === "OPTIONS") {
    return res.sendStatus(204);
  }

  next();
});

// Option 2: cors package
const cors = require("cors");

app.use(
  cors({
    origin: (origin, callback) => {
      const allowedOrigins = [
        "https://app.example.com",
        "https://admin.example.com",
      ];

      if (!origin || allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error("Not allowed by CORS"));
      }
    },
    credentials: true,
    methods: ["GET", "POST", "PUT", "DELETE"],
    allowedHeaders: ["Content-Type", "Authorization"],
    maxAge: 3600,
  }),
);

app.get("/api/data", (req, res) => {
  res.json({ message: "CORS enabled" });
});
```

---

### Next.js API Routes

```typescript
// pages/api/data.ts
import type { NextApiRequest, NextApiResponse } from "next";

const allowedOrigins = ["https://app.example.com", "https://admin.example.com"];

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  const origin = req.headers.origin;

  if (origin && allowedOrigins.includes(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin);
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
    res.setHeader(
      "Access-Control-Allow-Headers",
      "Content-Type, Authorization",
    );
  }

  // Handle preflight
  if (req.method === "OPTIONS") {
    return res.status(204).end();
  }

  // Handle actual request
  if (req.method === "GET") {
    return res.status(200).json({ message: "CORS enabled" });
  }

  res.status(405).json({ error: "Method not allowed" });
}
```

---

### FastAPI (Python)

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

allowed_origins = [
    "https://app.example.com",
    "https://admin.example.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
    max_age=3600
)

@app.get("/api/data")
async def get_data():
    return {"message": "CORS enabled"}
```

---

## 🧪 6. Real-World Scenarios

### Scenario 1: API with Authentication

```javascript
// Server
app.use(
  cors({
    origin: "https://app.example.com",
    credentials: true,
  }),
);

app.post("/api/login", (req, res) => {
  // Set cookie
  res.cookie("sessionId", "abc123", {
    httpOnly: true,
    secure: true,
    sameSite: "none", // Required for cross-site cookies
  });

  res.json({ user: "john" });
});

// Client
fetch("https://api.example.com/api/login", {
  method: "POST",
  credentials: "include", // Send cookies
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ username: "john", password: "secret" }),
});
```

---

### Scenario 2: Multiple Subdomains

```javascript
// Allow all subdomains of example.com
app.use((req, res, next) => {
  const origin = req.headers.origin;

  if (origin && /^https:\/\/[\w-]+\.example\.com$/.test(origin)) {
    res.header("Access-Control-Allow-Origin", origin);
    res.header("Access-Control-Allow-Credentials", "true");
  }

  next();
});

// Matches:
// https://app.example.com ✅
// https://admin.example.com ✅
// https://api.example.com ✅
// https://evil.com ❌
```

---

### Scenario 3: Public API (No Credentials)

```javascript
// Allow all origins (no credentials)
app.use(
  cors({
    origin: "*", // Wildcard OK when no credentials
    methods: ["GET"], // Read-only
  }),
);

app.get("/api/public-data", (req, res) => {
  res.json({ data: "Public information" });
});
```

---

## ❓ 7. Interview Questions

### Q1: Explain CORS preflight request

**Answer:**

**Preflight is an OPTIONS request sent before the actual request to check if the server allows it.**

**Triggered when:**

- HTTP method: `PUT`, `DELETE`, `PATCH`
- Custom headers: `Authorization`, `X-Api-Key`
- Content-Type: `application/json`

**Flow:**

```
1. Browser: OPTIONS /api/data
   Headers:
     Access-Control-Request-Method: POST
     Access-Control-Request-Headers: Content-Type, Authorization

2. Server: 204 No Content
   Headers:
     Access-Control-Allow-Origin: https://app.com
     Access-Control-Allow-Methods: POST
     Access-Control-Allow-Headers: Content-Type, Authorization
     Access-Control-Max-Age: 3600

3. If successful, browser sends actual request:
   POST /api/data
```

**Purpose:**

- Security: Prevent unauthorized cross-origin requests
- Efficiency: Cache preflight response (max-age)

---

### Q2: Why can't you use `Access-Control-Allow-Origin: *` with credentials?

**Answer:**

**Security risk: Any website could access authenticated resources.**

```javascript
// ❌ This combination is rejected by browsers:
res.header("Access-Control-Allow-Origin", "*");
res.header("Access-Control-Allow-Credentials", "true");

// Error:
// The value of the 'Access-Control-Allow-Origin' header in the
// response must not be the wildcard '*' when the request's
// credentials mode is 'include'.
```

**Why?**

- Wildcard (`*`) allows ALL origins
- `credentials: true` sends cookies/auth headers
- Attacker could steal user data from authenticated sessions

**Solution:**

```javascript
// ✅ Specify exact origin when using credentials
res.header("Access-Control-Allow-Origin", "https://trusted-site.com");
res.header("Access-Control-Allow-Credentials", "true");
```

---

### Q3: How to handle CORS errors in development?

**Answer:**

**Options:**

**1. Proxy (Vite/Create React App):**

```javascript
// vite.config.js
export default {
  server: {
    proxy: {
      "/api": {
        target: "https://api.example.com",
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ""),
      },
    },
  },
};

// Client: Same origin (no CORS)
fetch("/api/data"); // Proxied to https://api.example.com/data
```

**2. Allow localhost in dev:**

```javascript
const allowedOrigins = [
  "https://app.example.com",
  process.env.NODE_ENV === "development" && "http://localhost:3000",
].filter(Boolean);
```

**3. Browser extension (NOT for production):**

- "Allow CORS" Chrome extension
- Disables CORS temporarily

**4. CORS middleware with wildcard (dev only):**

```javascript
if (process.env.NODE_ENV === "development") {
  app.use(cors({ origin: "*" }));
}
```

---

## 🎯 Summary

**CORS headers:**

- `Access-Control-Allow-Origin` - Allowed origins
- `Access-Control-Allow-Methods` - Allowed HTTP methods
- `Access-Control-Allow-Headers` - Allowed request headers
- `Access-Control-Allow-Credentials` - Allow cookies
- `Access-Control-Max-Age` - Cache preflight
- `Access-Control-Expose-Headers` - Expose custom headers

**Preflight:**

- OPTIONS request before actual request
- Triggered by: custom headers, JSON content-type, PUT/DELETE methods
- Cached with `max-age`

**Best practices:**

- ✅ Whitelist specific origins
- ✅ Use credentials only with specific origins
- ✅ Cache preflight responses
- ❌ Don't use `*` with credentials
- ❌ Don't disable CORS in production

Master CORS for secure cross-origin communication! 🔒
