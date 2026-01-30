# Secure Communication

## Core Concept

Secure communication protects data in transit between client and server using HTTPS, Content Security Policy (CSP), Cross-Origin Resource Sharing (CORS), and secure WebSocket connections. These mechanisms prevent man-in-the-middle attacks, XSS, and unauthorized API access.

---

## HTTPS Enforcement

```typescript
// Redirect HTTP to HTTPS
if (
  window.location.protocol !== "https:" &&
  window.location.hostname !== "localhost"
) {
  window.location.href =
    "https:" + window.location.href.substring(window.location.protocol.length);
}

// Check for secure context
if (window.isSecureContext) {
  // Use sensitive APIs only in HTTPS
  navigator.geolocation.getCurrentPosition(/* ... */);
} else {
  console.warn("Not in secure context");
}
```

---

## Strict-Transport-Security (HSTS)

```typescript
// Server-side header (Express)
app.use((req, res, next) => {
  res.setHeader(
    "Strict-Transport-Security",
    "max-age=31536000; includeSubDomains; preload",
  );
  next();
});

// HSTS tells browsers to:
// - Always use HTTPS for this domain
// - For 1 year (31536000 seconds)
// - Including all subdomains
// - Can be added to browser preload list
```

---

## Content Security Policy (CSP)

```typescript
// Basic CSP
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "default-src 'self'; " +
    "script-src 'self' https://cdn.example.com; " +
    "style-src 'self' 'unsafe-inline'; " +
    "img-src 'self' data: https:; " +
    "font-src 'self' https://fonts.gstatic.com; " +
    "connect-src 'self' https://api.example.com; " +
    "frame-ancestors 'none'; " +
    "base-uri 'self'; " +
    "form-action 'self'"
  );
  next();
});

// Strict CSP with nonces
function generateNonce() {
  return crypto.randomBytes(16).toString('base64');
}

app.use((req, res, next) => {
  const nonce = generateNonce();
  res.locals.nonce = nonce;

  res.setHeader(
    'Content-Security-Policy',
    `script-src 'nonce-${nonce}' 'strict-dynamic'; ` +
    "object-src 'none'; " +
    "base-uri 'none'"
  );
  next();
});

// Use nonce in HTML
<script nonce="${nonce}">
  console.log('Allowed script');
</script>
```

---

## CSP in React

```typescript
// Next.js CSP configuration
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: "/:path*",
        headers: [
          {
            key: "Content-Security-Policy",
            value: [
              "default-src 'self'",
              "script-src 'self' 'unsafe-eval' 'unsafe-inline'",
              "style-src 'self' 'unsafe-inline'",
              "img-src 'self' data: https:",
              "font-src 'self' data:",
              "connect-src 'self' https://api.example.com",
            ].join("; "),
          },
        ],
      },
    ];
  },
};

// Report CSP violations
const cspConfig = {
  "Content-Security-Policy-Report-Only": [
    "default-src 'self'",
    "report-uri /api/csp-report",
  ].join("; "),
};

// Handle reports
app.post("/api/csp-report", express.json(), (req, res) => {
  console.error("CSP Violation:", req.body);
  // Log to monitoring service
  res.status(204).end();
});
```

---

## CORS Configuration

```typescript
// Basic CORS (Express)
import cors from "cors";

app.use(
  cors({
    origin: "https://example.com",
    credentials: true,
    methods: ["GET", "POST", "PUT", "DELETE"],
    allowedHeaders: ["Content-Type", "Authorization"],
  }),
);

// Dynamic origin validation
const allowedOrigins = [
  "https://example.com",
  "https://app.example.com",
  "https://staging.example.com",
];

app.use(
  cors({
    origin: (origin, callback) => {
      if (!origin || allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error("Not allowed by CORS"));
      }
    },
    credentials: true,
  }),
);

// Preflight cache
app.use(
  cors({
    origin: "https://example.com",
    credentials: true,
    maxAge: 86400, // 24 hours
  }),
);
```

---

## Secure Fetch Requests

```typescript
// Always use HTTPS endpoints
const API_BASE = "https://api.example.com";

// Include credentials for cookies
async function secureFetch<T>(
  endpoint: string,
  options: RequestInit = {},
): Promise<T> {
  const response = await fetch(`${API_BASE}${endpoint}`, {
    ...options,
    credentials: "include", // Send cookies
    headers: {
      "Content-Type": "application/json",
      ...options.headers,
    },
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  return response.json();
}

// Usage
const data = await secureFetch<User[]>("/users");
```

---

## Secure WebSockets

```typescript
// Use wss:// instead of ws://
const socket = new WebSocket("wss://api.example.com/ws");

// With authentication
class SecureWebSocket {
  private socket: WebSocket | null = null;
  private token: string;

  constructor(token: string) {
    this.token = token;
  }

  connect() {
    // Include token in URL or send after connection
    this.socket = new WebSocket(`wss://api.example.com/ws?token=${this.token}`);

    this.socket.onopen = () => {
      console.log("Connected securely");
    };

    this.socket.onmessage = (event) => {
      // Validate message origin
      if (this.socket?.url.startsWith("wss://api.example.com")) {
        this.handleMessage(JSON.parse(event.data));
      }
    };

    this.socket.onerror = (error) => {
      console.error("WebSocket error:", error);
    };
  }

  private handleMessage(data: any) {
    // Process validated message
  }

  send(data: any) {
    if (this.socket?.readyState === WebSocket.OPEN) {
      this.socket.send(JSON.stringify(data));
    }
  }

  disconnect() {
    this.socket?.close();
  }
}
```

---

## Certificate Pinning (Advanced)

```typescript
// Service Worker certificate validation
self.addEventListener('fetch', (event) => {
  if (event.request.url.startsWith('https://api.example.com')) {
    event.respondWith(
      fetch(event.request).then(response => {
        // Validate certificate (requires custom implementation)
        // Note: Not directly supported in browsers
        return response;
      })
    );
  }
});

// Use Subresource Integrity for CDN scripts
<script
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/ux..."
  crossorigin="anonymous"
></script>
```

---

## Subresource Integrity (SRI)

```typescript
// Generate SRI hash
import crypto from 'crypto';
import fs from 'fs';

function generateSRI(filePath: string): string {
  const content = fs.readFileSync(filePath);
  const hash = crypto.createHash('sha384').update(content).digest('base64');
  return `sha384-${hash}`;
}

const integrity = generateSRI('./public/app.js');
console.log(integrity);

// Use in HTML
<script
  src="/app.js"
  integrity={integrity}
  crossorigin="anonymous"
></script>

// React component for secure external scripts
interface SecureScriptProps {
  src: string;
  integrity: string;
}

function SecureScript({ src, integrity }: SecureScriptProps) {
  useEffect(() => {
    const script = document.createElement('script');
    script.src = src;
    script.integrity = integrity;
    script.crossOrigin = 'anonymous';

    script.onerror = () => {
      console.error('Script failed integrity check');
    };

    document.body.appendChild(script);

    return () => {
      document.body.removeChild(script);
    };
  }, [src, integrity]);

  return null;
}
```

---

## Secure Headers Bundle

```typescript
// Express security headers
import helmet from "helmet";

app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'unsafe-inline'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", "data:", "https:"],
        connectSrc: ["'self'", "https://api.example.com"],
        fontSrc: ["'self'", "https:", "data:"],
        objectSrc: ["'none'"],
        mediaSrc: ["'self'"],
        frameSrc: ["'none'"],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
    noSniff: true,
    xssFilter: true,
    referrerPolicy: { policy: "strict-origin-when-cross-origin" },
  }),
);

// Additional headers
app.use((req, res, next) => {
  res.setHeader("X-Frame-Options", "DENY");
  res.setHeader("X-Content-Type-Options", "nosniff");
  res.setHeader("X-XSS-Protection", "1; mode=block");
  res.setHeader("Permissions-Policy", "geolocation=(), microphone=()");
  next();
});
```

---

## Real-World: Secure API Client

```typescript
class SecureAPIClient {
  private baseURL: string;
  private csrfToken: string | null = null;

  constructor(baseURL: string) {
    if (!baseURL.startsWith("https://")) {
      throw new Error("API must use HTTPS");
    }
    this.baseURL = baseURL;
  }

  async initialize() {
    // Fetch CSRF token
    const response = await fetch(`${this.baseURL}/csrf-token`, {
      credentials: "include",
    });
    const { token } = await response.json();
    this.csrfToken = token;
  }

  async request<T>(endpoint: string, options: RequestInit = {}): Promise<T> {
    const headers: HeadersInit = {
      "Content-Type": "application/json",
      ...options.headers,
    };

    // Add CSRF token for mutations
    if (["POST", "PUT", "DELETE"].includes(options.method || "GET")) {
      if (!this.csrfToken) {
        await this.initialize();
      }
      headers["X-CSRF-Token"] = this.csrfToken!;
    }

    const response = await fetch(`${this.baseURL}${endpoint}`, {
      ...options,
      headers,
      credentials: "include",
      mode: "cors",
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    return response.json();
  }

  async get<T>(endpoint: string): Promise<T> {
    return this.request<T>(endpoint, { method: "GET" });
  }

  async post<T>(endpoint: string, data: any): Promise<T> {
    return this.request<T>(endpoint, {
      method: "POST",
      body: JSON.stringify(data),
    });
  }
}

// Usage
const api = new SecureAPIClient("https://api.example.com");
await api.initialize();

const users = await api.get<User[]>("/users");
const created = await api.post<User>("/users", { name: "John" });
```

---

## Best Practices

✅ **Always use HTTPS** in production  
✅ **Implement CSP** to prevent XSS  
✅ **Configure CORS** properly  
✅ **Use SRI for CDN resources**  
✅ **Enable HSTS** for HTTPS enforcement  
✅ **Validate certificate chains**  
✅ **Use secure WebSocket (wss://)**  
❌ **Don't disable HTTPS** certificate validation  
❌ **Don't allow `unsafe-inline`** in CSP without nonces  
❌ **Don't use `*` for CORS origin**

---

## Key Takeaways

1. **HTTPS is mandatory** for secure communication
2. **CSP prevents XSS** and other injection attacks
3. **CORS controls** cross-origin requests
4. **SRI validates** external resources
5. **HSTS enforces** HTTPS automatically
6. **Secure WebSockets** use wss:// protocol
7. **Defense in depth** - use multiple layers
