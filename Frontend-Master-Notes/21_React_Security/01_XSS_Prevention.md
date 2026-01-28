# XSS Prevention in React

## Core Concept

Cross-Site Scripting (XSS) occurs when attackers inject malicious scripts into web pages. React provides automatic escaping, but developers must understand when protection is bypassed.

---

## React's Built-in Protection

React automatically escapes JSX values:

```typescript
// ✅ SAFE - React escapes HTML
function UserProfile({ username }: { username: string }) {
  // If username = "<script>alert('XSS')</script>"
  // React renders: &lt;script&gt;alert('XSS')&lt;/script&gt;
  return <div>{username}</div>;
}

// ✅ SAFE - React escapes attributes
function Link({ url }: { url: string }) {
  // React prevents javascript: URLs in href
  return <a href={url}>Click</a>;
}
```

---

## dangerouslySetInnerHTML

**The most common XSS vulnerability:**

```typescript
// ❌ DANGEROUS - No escaping!
function Article({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// If html contains: "<img src=x onerror='alert(1)'>"
// Script executes! 💀

// ✅ SAFE - Sanitize first
import DOMPurify from 'dompurify';

function SafeArticle({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

---

## DOMPurify

Industry-standard HTML sanitizer:

```typescript
import DOMPurify from "dompurify";

// Basic sanitization
const clean = DOMPurify.sanitize(dirtyHTML);

// Custom config
const clean = DOMPurify.sanitize(dirtyHTML, {
  ALLOWED_TAGS: ["b", "i", "em", "strong", "a", "p"],
  ALLOWED_ATTR: ["href", "title"],
  ALLOW_DATA_ATTR: false,
});

// Remove all HTML
const text = DOMPurify.sanitize(input, { ALLOWED_TAGS: [] });
```

---

## URL Sanitization

Prevent `javascript:` URLs:

```typescript
// ❌ DANGEROUS - Allows javascript: URLs
function UnsafeLink({ url }: { url: string }) {
  return <a href={url}>Click</a>;
}
// If url = "javascript:alert(1)" → executes!

// ✅ SAFE - Validate URL protocol
function SafeLink({ url }: { url: string }) {
  const isValid = (url: string): boolean => {
    try {
      const parsed = new URL(url, window.location.origin);
      return ['http:', 'https:', 'mailto:'].includes(parsed.protocol);
    } catch {
      return false;
    }
  };

  if (!isValid(url)) {
    return <span>Invalid link</span>;
  }

  return <a href={url}>Click</a>;
}
```

---

## Event Handler Injection

```typescript
// ❌ DANGEROUS - User input in event handler
function BadButton({ onClick }: { onClick: string }) {
  return <button onClick={eval(onClick)}>Click</button>;
}
// Never use eval()!

// ✅ SAFE - Use proper event handlers
function GoodButton({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Click</button>;
}
```

---

## CSS Injection

```typescript
// ❌ DANGEROUS - User input in style
function UnsafeDiv({ color }: { color: string }) {
  return <div style={{ color }}>Text</div>;
}
// If color = "red; background: url('javascript:alert(1)')"

// ✅ SAFE - Validate CSS values
function SafeDiv({ color }: { color: string }) {
  const validColor = /^#[0-9A-Fa-f]{6}$|^rgb\(|^hsl\(/;

  if (!validColor.test(color)) {
    return <div>Text</div>;
  }

  return <div style={{ color }}>Text</div>;
}
```

---

## Markdown to HTML

```typescript
import { marked } from 'marked';
import DOMPurify from 'dompurify';

// ❌ DANGEROUS - Raw markdown conversion
function UnsafeMarkdown({ content }: { content: string }) {
  const html = marked(content);
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// ✅ SAFE - Sanitize after conversion
function SafeMarkdown({ content }: { content: string }) {
  const html = marked(content);
  const clean = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// ✅ BETTER - Use React markdown library
import ReactMarkdown from 'react-markdown';

function BetterMarkdown({ content }: { content: string }) {
  return <ReactMarkdown>{content}</ReactMarkdown>;
}
```

---

## Server-Side Rendering (SSR)

```typescript
// ❌ DANGEROUS - Injecting state with user data
function ServerApp({ user }: { user: User }) {
  return (
    <html>
      <body>
        <div id="root" />
        <script
          dangerouslySetInnerHTML={{
            __html: `window.__USER__ = ${JSON.stringify(user)}`
          }}
        />
      </body>
    </html>
  );
}
// If user.name = "</script><script>alert(1)</script>"
// Breaks out of script tag!

// ✅ SAFE - Properly escape
import serialize from 'serialize-javascript';

function SafeServerApp({ user }: { user: User }) {
  return (
    <html>
      <body>
        <div id="root" />
        <script
          dangerouslySetInnerHTML={{
            __html: `window.__USER__ = ${serialize(user, { isJSON: true })}`
          }}
        />
      </body>
    </html>
  );
}
```

---

## Content Security Policy (CSP)

HTTP header that restricts resource loading:

```typescript
// Next.js example
export default function handler(req, res) {
  res.setHeader(
    "Content-Security-Policy",
    [
      "default-src 'self'",
      "script-src 'self' 'nonce-{RANDOM}'",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self'",
      "frame-ancestors 'none'",
    ].join("; "),
  );

  res.send("...");
}
```

---

## Real-World Example: Rich Text Editor

```typescript
import DOMPurify from 'dompurify';
import { useState } from 'react';

function RichTextEditor() {
  const [content, setContent] = useState('');
  const [preview, setPreview] = useState('');

  const handleSave = () => {
    // Sanitize before saving
    const clean = DOMPurify.sanitize(content, {
      ALLOWED_TAGS: [
        'p', 'br', 'strong', 'em', 'u', 'h1', 'h2', 'h3',
        'ul', 'ol', 'li', 'a', 'img'
      ],
      ALLOWED_ATTR: ['href', 'src', 'alt', 'title'],
      ALLOW_DATA_ATTR: false
    });

    // Save to backend
    api.save({ content: clean });
    setPreview(clean);
  };

  return (
    <div>
      <textarea
        value={content}
        onChange={e => setContent(e.target.value)}
      />
      <button onClick={handleSave}>Save</button>

      {preview && (
        <div
          className="preview"
          dangerouslySetInnerHTML={{ __html: preview }}
        />
      )}
    </div>
  );
}
```

---

## Input Validation

```typescript
// Validate and sanitize user input
function validateInput(input: string): string {
  // Remove scripts
  let clean = input.replace(
    /<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi,
    "",
  );

  // Remove event handlers
  clean = clean.replace(/on\w+\s*=\s*["'][^"']*["']/gi, "");

  // Remove javascript: URLs
  clean = clean.replace(/javascript:/gi, "");

  return clean;
}

// Better: Use a library
import validator from "validator";

function safeEmail(email: string): boolean {
  return validator.isEmail(email);
}

function safeURL(url: string): boolean {
  return validator.isURL(url, {
    protocols: ["http", "https"],
    require_protocol: true,
  });
}
```

---

## User-Generated Content

```typescript
interface Comment {
  id: string;
  author: string;
  text: string;
  createdAt: Date;
}

function CommentList({ comments }: { comments: Comment[] }) {
  return (
    <div>
      {comments.map(comment => (
        <div key={comment.id}>
          {/* ✅ SAFE - React escapes automatically */}
          <strong>{comment.author}</strong>
          <p>{comment.text}</p>
          <time>{comment.createdAt.toLocaleString()}</time>
        </div>
      ))}
    </div>
  );
}

// If you need HTML formatting:
function CommentWithHTML({ comment }: { comment: Comment }) {
  const sanitized = DOMPurify.sanitize(comment.text, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong'],
    ALLOWED_ATTR: []
  });

  return (
    <div>
      <strong>{comment.author}</strong>
      <div dangerouslySetInnerHTML={{ __html: sanitized }} />
    </div>
  );
}
```

---

## Best Practices

✅ **Never use eval()** or Function() with user input  
✅ **Sanitize HTML** before dangerouslySetInnerHTML  
✅ **Validate URLs** before using in href/src  
✅ **Use DOMPurify** for HTML sanitization  
✅ **Implement CSP headers**  
✅ **Escape JSON** in SSR properly  
✅ **Use React markdown libraries** instead of raw HTML  
❌ **Don't trust user input** - always validate  
❌ **Don't disable React's escaping** without sanitization  
❌ **Don't use innerHTML** in vanilla JS

---

## Security Checklist

- [ ] All user input is validated
- [ ] HTML is sanitized with DOMPurify
- [ ] URLs are validated before use
- [ ] CSP headers are implemented
- [ ] No eval() or Function() with user data
- [ ] SSR state is properly escaped
- [ ] Event handlers don't use user input
- [ ] CSS values are validated
- [ ] Markdown is sanitized or use React library

---

## Key Takeaways

1. **React escapes JSX** automatically - safe by default
2. **dangerouslySetInnerHTML** bypasses protection
3. **Always sanitize** with DOMPurify before innerHTML
4. **Validate URLs** to prevent javascript: attacks
5. **Never use eval()** with user input
6. **Implement CSP headers** for defense-in-depth
7. **Trust no user input** - validate everything
