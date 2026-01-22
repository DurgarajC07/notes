# 🌐 Critical Rendering Path

> The Critical Rendering Path is the sequence of steps the browser takes to convert HTML, CSS, and JavaScript into pixels on the screen. Optimizing this path is essential for fast page loads and better Core Web Vitals.

---

## 📖 1. Concept Explanation

### What is the Critical Rendering Path?

The **Critical Rendering Path (CRP)** is the sequence of steps browsers must complete before rendering content:

```
HTML → DOM
CSS  → CSSOM
DOM + CSSOM → Render Tree → Layout → Paint → Composite
```

**The 6 Steps:**

1. **Parse HTML** → Build DOM (Document Object Model)
2. **Parse CSS** → Build CSSOM (CSS Object Model)
3. **Combine** → Create Render Tree
4. **Layout** → Calculate positions and sizes
5. **Paint** → Fill in pixels
6. **Composite** → Layer composition to screen

```javascript
// Browser's internal process
function renderPage() {
  const dom = parseHTML(htmlString);
  const cssom = parseCSS(cssString);
  const renderTree = combineTreesFilterHidden(dom, cssom);
  const layout = calculatePositions(renderTree);
  const painted = paintPixels(layout);
  composite(painted);
}
```

### DOM Construction

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Page</title>
  </head>
  <body>
    <div class="container">
      <h1>Hello</h1>
    </div>
  </body>
</html>
```

**DOM Tree:**

```
html
├── head
│   └── title → "Page"
└── body
    └── div.container
        └── h1 → "Hello"
```

### CSSOM Construction

```css
body {
  font-size: 16px;
}

.container {
  width: 100%;
}

.container h1 {
  color: blue;
}
```

**CSSOM Tree:**

```
body (font-size: 16px)
└── .container (width: 100%)
    └── h1 (color: blue, inherited: font-size: 16px)
```

### Render Tree

Combines DOM + CSSOM, excludes hidden elements:

```
body
└── div.container
    └── h1 (computed styles: font-size: 16px, color: blue)

// Excluded: <head>, <script>, display:none elements
```

---

## 🧠 2. Why It Matters in Real Projects

### Performance Impact

**Blocking Resources:**

```html
<!-- ❌ BAD: Blocks rendering -->
<head>
  <link rel="stylesheet" href="styles.css" />
  <!-- Browser WAITS for CSS before rendering -->

  <script src="app.js"></script>
  <!-- Parser STOPS, downloads & executes, THEN continues -->
</head>

<!-- ✅ GOOD: Optimized loading -->
<head>
  <!-- Critical CSS inline -->
  <style>
    /* Above-the-fold styles */
    .hero {
      /* ... */
    }
  </style>

  <!-- Non-critical CSS async -->
  <link
    rel="preload"
    href="styles.css"
    as="style"
    onload="this.onload=null;this.rel='stylesheet'"
  />

  <!-- Scripts deferred -->
  <script defer src="app.js"></script>
</head>
```

### Real-World Metrics

**Impact on LCP (Largest Contentful Paint):**

```javascript
// BEFORE optimization
// CRP: HTML (200ms) + CSS (300ms) + JS (500ms) = 1000ms
// LCP: 2.5s ❌

// AFTER optimization
// CRP: HTML (200ms) + Inline Critical CSS (0ms) = 200ms
// LCP: 1.2s ✅

// Business impact: 52% faster LCP = 15% increase in conversions
```

### Production Scenarios

**1. E-commerce Product Page:**

```html
<!-- Critical: Product image, title, price -->
<style>
  .product-hero {
    /* inline critical CSS */
  }
</style>

<!-- Deferred: Reviews, recommendations -->
<link rel="preload" href="secondary.css" as="style" onload="..." />
```

**2. News Article:**

```html
<!-- Critical: Headline, first paragraph -->
<!-- Deferred: Comments, related articles -->
```

---

## ⚙️ 3. Internal Working

### Step 1: HTML Parsing (Incremental)

```javascript
// Browser's HTML parser (simplified)
class HTMLParser {
  parse(html) {
    const tokens = tokenize(html);
    const dom = buildDOM(tokens);

    // Incremental: Can render partial DOM
    while (moreTokens) {
      processToken();
      if (canRender) {
        scheduleRender(); // Don't wait for entire HTML
      }
    }

    return dom;
  }
}
```

**Blocking Points:**

```html
<html>
  <head>
    <link rel="stylesheet" href="styles.css" />
    <!-- ⏸️ PARSER CONTINUES but RENDERING BLOCKED -->
  </head>
  <body>
    <h1>Hello</h1>
    <script src="app.js"></script>
    <!-- ⏸️ PARSER STOPS, waits for script -->
    <p>World</p>
  </body>
</html>
```

### Step 2: CSSOM Construction (Blocking)

```javascript
// CSSOM must be complete before rendering
function parseCSS(css) {
  // Parse all CSS rules
  const rules = parseCSSRules(css);

  // Build CSSOM tree
  const cssom = buildCSSOSOM(rules);

  // Compute cascaded values
  computeCascade(cssom);

  return cssom; // MUST complete before rendering
}
```

**Why CSSOM blocks rendering:**

```css
/* Browser can't know final styles without all CSS */
.button {
  color: red;
}
.button {
  color: blue;
} /* Overrides! */

/* Without complete CSSOM, would show wrong styles */
```

### Step 3: Render Tree Construction

```javascript
function buildRenderTree(dom, cssom) {
  const renderTree = [];

  function traverse(domNode) {
    // Skip non-visual nodes
    if (domNode.tagName === "SCRIPT") return;
    if (domNode.tagName === "META") return;

    // Get computed styles
    const styles = cssom.getComputedStyle(domNode);

    // Skip display:none
    if (styles.display === "none") return;

    // Create render object
    const renderObject = {
      domNode,
      styles,
      children: [],
    };

    // Recurse children
    domNode.children.forEach((child) => {
      const childRO = traverse(child);
      if (childRO) renderObject.children.push(childRO);
    });

    return renderObject;
  }

  return traverse(dom.documentElement);
}
```

### Step 4: Layout (Reflow)

```javascript
function layout(renderTree, viewport) {
  // Calculate geometry for all elements
  function calculateLayout(node, parentBox) {
    // Box model calculation
    const box = {
      x: calculateX(node.styles, parentBox),
      y: calculateY(node.styles, parentBox),
      width: calculateWidth(node.styles, parentBox),
      height: calculateHeight(node.styles, parentBox),
      margin: calculateMargin(node.styles),
      padding: calculatePadding(node.styles),
    };

    // Store in node
    node.box = box;

    // Layout children
    node.children.forEach((child) => {
      calculateLayout(child, box);
    });
  }

  calculateLayout(renderTree, {
    width: viewport.width,
    height: viewport.height,
  });
}
```

**Layout triggers:**

- Window resize
- Content changes (DOM manipulation)
- Style changes affecting geometry
- Font loading

### Step 5: Paint

```javascript
function paint(renderTree) {
  const layers = [];

  // Create paint layers
  function createLayers(node, currentLayer) {
    // Create new layer if needed
    if (needsNewLayer(node)) {
      currentLayer = createNewLayer();
      layers.push(currentLayer);
    }

    // Add paint commands to layer
    currentLayer.commands.push(
      { op: "fillRect", box: node.box, color: node.styles.backgroundColor },
      {
        op: "drawText",
        box: node.box,
        text: node.text,
        color: node.styles.color,
      },
      { op: "drawBorder", box: node.box, border: node.styles.border },
    );

    // Recurse children
    node.children.forEach((child) => createLayers(child, currentLayer));
  }

  createLayers(renderTree, createNewLayer());
  return layers;
}
```

**Paint triggers:**

- Color changes
- Background changes
- Box shadow changes
- Visibility changes

### Step 6: Composite

```javascript
function composite(layers) {
  // Sort layers by z-index
  layers.sort((a, b) => a.zIndex - b.zIndex);

  // Composite on GPU
  layers.forEach((layer) => {
    // Upload to GPU texture
    const texture = uploadToGPU(layer);

    // Apply transforms (CSS transform)
    applyTransforms(texture, layer.transform);

    // Blend with previous layers
    blendTexture(texture, layer.blendMode);
  });

  // Display on screen
  swapBuffers();
}
```

**Composite-only operations (fastest):**

- `transform: translate/scale/rotate`
- `opacity`
- `filter` (some)

---

## ✅ 4. Best Practices

### DO ✅

```html
<!-- 1. Inline critical CSS -->
<head>
  <style>
    /* Above-the-fold styles */
    .hero {
      width: 100%;
      height: 600px;
      background: #f0f0f0;
    }
  </style>

  <!-- Load non-critical async -->
  <link
    rel="preload"
    href="styles.css"
    as="style"
    onload="this.onload=null;this.rel='stylesheet'"
  />
  <noscript><link rel="stylesheet" href="styles.css" /></noscript>
</head>

<!-- 2. Use preconnect for external resources -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://cdn.example.com" />

<!-- 3. Defer non-critical JavaScript -->
<script defer src="app.js"></script>
<script async src="analytics.js"></script>

<!-- 4. Optimize images for LCP -->
<img src="hero.jpg" fetchpriority="high" decoding="sync" alt="Hero" />

<!-- 5. Use resource hints -->
<link rel="preload" href="hero.jpg" as="image" />
<link rel="prefetch" href="next-page.html" />
<link rel="dns-prefetch" href="https://api.example.com" />
```

### DON'T ❌

```html
<!-- ❌ Blocking CSS at bottom -->
<body>
  <div>Content</div>
  <link rel="stylesheet" href="styles.css" />
  <!-- Late! -->
</body>

<!-- ❌ Synchronous scripts in head -->
<head>
  <script src="large-lib.js"></script>
  <!-- BLOCKS parsing! -->
</head>

<!-- ❌ Large inline styles -->
<style>
  /* 500KB of CSS inline... */
</style>

<!-- ❌ Multiple stylesheets -->
<link rel="stylesheet" href="reset.css" />
<link rel="stylesheet" href="grid.css" />
<link rel="stylesheet" href="components.css" />
<link rel="stylesheet" href="utilities.css" />
<!-- Combine into one! -->

<!-- ❌ Loading all fonts upfront -->
<link
  href="https://fonts.googleapis.com/css2?family=Roboto:wght@100;300;400;500;700;900&display=swap"
/>
<!-- Only load weights you need -->
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Render-Blocking CSS

```html
<!-- ❌ WRONG: All CSS blocks rendering -->
<head>
  <link rel="stylesheet" href="styles.css" />
  <!-- 300KB -->
</head>

<!-- ✅ CORRECT: Split critical/non-critical -->
<head>
  <!-- Inline critical CSS (< 14KB) -->
  <style>
    .hero {
      /* ... */
    }
    .nav {
      /* ... */
    }
  </style>

  <!-- Async load rest -->
  <link
    rel="preload"
    href="styles.css"
    as="style"
    onload="this.rel='stylesheet'"
  />
</head>
```

**Tool: Critical CSS extraction**

```javascript
// Using critical package
const critical = require("critical");

critical.generate({
  inline: true,
  base: "dist/",
  src: "index.html",
  target: {
    html: "index-critical.html",
    css: "critical.css",
  },
  width: 1300,
  height: 900,
});
```

### Mistake #2: Parser-Blocking JavaScript

```html
<!-- ❌ WRONG: Blocks HTML parsing -->
<head>
  <script src="app.js"></script>
  <!-- Parser stops here -->
</head>
<body>
  <h1>Hello</h1>
  <!-- Not parsed until script loads & executes -->
</body>

<!-- ✅ CORRECT: defer or async -->
<head>
  <script defer src="app.js"></script>
  <!-- Parallel download, executes after DOM ready -->
</head>
<body>
  <h1>Hello</h1>
  <!-- Parses immediately -->
</body>
```

**defer vs async:**

```html
<!-- defer: Execute in order, after DOM ready -->
<script defer src="lib.js"></script>
<script defer src="app.js"></script>
<!-- Runs AFTER lib.js -->

<!-- async: Execute as soon as downloaded, out of order -->
<script async src="analytics.js"></script>
<!-- Runs whenever ready -->
```

### Mistake #3: Not Optimizing Web Fonts

```css
/* ❌ WRONG: Blocks rendering while font loads */
@font-face {
  font-family: "CustomFont";
  src: url("font.woff2");
  /* No font-display */
}

body {
  font-family: "CustomFont"; /* FOIT: Flash of Invisible Text */
}

/* ✅ CORRECT: Use font-display */
@font-face {
  font-family: "CustomFont";
  src: url("font.woff2");
  font-display: swap; /* Show fallback immediately */
}
```

**Font loading strategies:**

```html
<!-- Preload critical fonts -->
<link rel="preload" href="font.woff2" as="font" type="font/woff2" crossorigin />

<!-- Optional: Font Loading API -->
<script>
  if ("fonts" in document) {
    document.fonts.load("16px CustomFont").then(() => {
      document.body.classList.add("fonts-loaded");
    });
  }
</script>
```

---

## 🔐 6. Security Considerations

### Content Security Policy (CSP)

```html
<!-- Prevent inline script execution -->
<meta
  http-equiv="Content-Security-Policy"
  content="
  default-src 'self';
  script-src 'self' https://cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
"
/>
```

**Impact on CRP:**

- Blocks inline scripts (use external files with defer/async)
- May block external resources
- Use nonce or hash for trusted inline scripts

### Subresource Integrity (SRI)

```html
<!-- Verify external resources -->
<link
  rel="stylesheet"
  href="https://cdn.example.com/style.css"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxOCVSCq+Xvjn/Yjn/xP9b"
  crossorigin="anonymous"
/>
```

---

## 🚀 7. Performance Optimization

### 1. Critical CSS Inline

```html
<head>
  <style>
    /* Extract above-the-fold CSS */
    .hero {
      /* ... */
    }
    .header {
      /* ... */
    }
  </style>

  <link
    rel="preload"
    href="full.css"
    as="style"
    onload="this.rel='stylesheet'"
  />
</head>
```

**Tool automation:**

```javascript
// webpack.config.js
const CriticalCssPlugin = require("critical-css-webpack-plugin");

module.exports = {
  plugins: [
    new CriticalCssPlugin({
      base: "dist/",
      src: "index.html",
      inline: true,
      minify: true,
      width: 1300,
      height: 900,
    }),
  ],
};
```

### 2. Resource Hints

```html
<!-- Preconnect: Early connection to origin -->
<link rel="preconnect" href="https://cdn.example.com" />

<!-- DNS-prefetch: Resolve DNS early -->
<link rel="dns-prefetch" href="https://api.example.com" />

<!-- Preload: High-priority fetch -->
<link rel="preload" href="hero.jpg" as="image" />
<link rel="preload" href="font.woff2" as="font" crossorigin />

<!-- Prefetch: Low-priority fetch for next navigation -->
<link rel="prefetch" href="next-page.html" />
```

### 3. Async CSS Loading

```html
<link
  rel="preload"
  href="styles.css"
  as="style"
  onload="this.onload=null;this.rel='stylesheet'"
/>
<noscript>
  <link rel="stylesheet" href="styles.css" />
</noscript>
```

**JavaScript alternative:**

```javascript
function loadCSS(href) {
  const link = document.createElement("link");
  link.rel = "stylesheet";
  link.href = href;
  document.head.appendChild(link);
}

// Load non-critical CSS
if ("requestIdleCallback" in window) {
  requestIdleCallback(() => loadCSS("non-critical.css"));
} else {
  setTimeout(() => loadCSS("non-critical.css"), 1000);
}
```

### 4. Optimize Images for LCP

```html
<!-- Priority hint for LCP image -->
<img
  src="hero.jpg"
  alt="Hero"
  fetchpriority="high"
  decoding="sync"
  width="1200"
  height="600"
/>

<!-- Lazy load below-fold images -->
<img src="image.jpg" alt="Below fold" loading="lazy" fetchpriority="low" />
```

---

## 🧪 8. Code Examples

### Example 1: Optimized HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <!-- Preconnect to external origins -->
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://cdn.example.com" />

    <!-- Critical CSS inline -->
    <style>
      body {
        margin: 0;
        font-family: system-ui;
      }
      .hero {
        width: 100%;
        height: 600px;
        background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      }
      .nav {
        display: flex;
        justify-content: space-between;
        padding: 1rem 2rem;
        background: white;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
      }
    </style>

    <!-- Preload LCP image -->
    <link rel="preload" href="hero.jpg" as="image" />

    <!-- Preload critical font -->
    <link
      rel="preload"
      href="font.woff2"
      as="font"
      type="font/woff2"
      crossorigin
    />

    <!-- Load non-critical CSS async -->
    <link
      rel="preload"
      href="styles.css"
      as="style"
      onload="this.rel='stylesheet'"
    />
    <noscript><link rel="stylesheet" href="styles.css" /></noscript>

    <title>Optimized Page</title>
  </head>
  <body>
    <nav class="nav">
      <!-- Navigation -->
    </nav>

    <div class="hero">
      <img src="hero.jpg" alt="Hero" fetchpriority="high" decoding="sync" />
    </div>

    <main>
      <!-- Content -->
    </main>

    <!-- Scripts at bottom with defer -->
    <script defer src="app.js"></script>
    <script async src="analytics.js"></script>
  </body>
</html>
```

### Example 2: Measuring CRP Metrics

```javascript
// Measure CRP performance
window.addEventListener("load", () => {
  const perfData = performance.getEntriesByType("navigation")[0];

  const metrics = {
    // Time to DOM Complete
    domComplete: perfData.domComplete,

    // Time to DOM Content Loaded (DOM + CSSOM ready)
    domContentLoaded: perfData.domContentLoadedEventEnd,

    // Time to Load (all resources)
    loadComplete: perfData.loadEventEnd,

    // Critical rendering path time
    crpTime: perfData.domContentLoadedEventEnd - perfData.fetchStart,
  };

  console.table(metrics);

  // Send to analytics
  sendToAnalytics("CRP", metrics);
});

// Measure resource timing
function analyzeResources() {
  const resources = performance.getEntriesByType("resource");

  const blockingResources = resources.filter((r) => {
    return (
      (r.initiatorType === "link" && r.name.endsWith(".css")) ||
      (r.initiatorType === "script" && !r.name.includes("async"))
    );
  });

  console.log("Blocking resources:", blockingResources.length);
  console.table(
    blockingResources.map((r) => ({
      name: r.name.split("/").pop(),
      duration: r.duration.toFixed(2),
      size: r.transferSize,
    })),
  );
}

analyzeResources();
```

### Example 3: Dynamic Critical CSS

```javascript
// Generate critical CSS dynamically
async function extractCriticalCSS() {
  const viewportHeight = window.innerHeight;
  const elements = document.querySelectorAll("*");
  const criticalSelectors = new Set();

  // Find above-the-fold elements
  elements.forEach((el) => {
    const rect = el.getBoundingClientRect();
    if (rect.top < viewportHeight) {
      // Get all CSS rules for this element
      const rules = getMatchedCSSRules(el);
      rules.forEach((rule) => {
        criticalSelectors.add(rule.selectorText);
      });
    }
  });

  // Extract critical CSS
  const criticalCSS = Array.from(criticalSelectors)
    .map((selector) => {
      const rules = [...document.styleSheets]
        .flatMap((sheet) => [...sheet.cssRules])
        .filter((rule) => rule.selectorText === selector);

      return rules.map((rule) => rule.cssText).join("\n");
    })
    .join("\n");

  return criticalCSS;
}

// Usage
extractCriticalCSS().then((css) => {
  console.log("Critical CSS:", css);
});
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: E-commerce Product Page

```html
<!DOCTYPE html>
<html>
  <head>
    <!-- Preconnect to CDN -->
    <link rel="preconnect" href="https://cdn.shopify.com" />

    <!-- Critical CSS: Product card -->
    <style>
      .product {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 2rem;
        max-width: 1200px;
        margin: 0 auto;
      }
      .product__image {
        width: 100%;
        height: auto;
      }
      .product__price {
        font-size: 2rem;
        font-weight: bold;
        color: #e63946;
      }
      .product__button {
        width: 100%;
        padding: 1rem;
        background: #2a9d8f;
        color: white;
        border: none;
        font-size: 1.125rem;
        cursor: pointer;
      }
    </style>

    <!-- Preload product image (LCP) -->
    <link rel="preload" href="product.jpg" as="image" fetchpriority="high" />

    <title>Product Name | Store</title>
  </head>
  <body>
    <div class="product">
      <img
        src="product.jpg"
        alt="Product"
        class="product__image"
        fetchpriority="high"
        decoding="sync"
        width="600"
        height="600"
      />
      <div class="product__info">
        <h1>Product Name</h1>
        <p class="product__price">$99.99</p>
        <button class="product__button">Add to Cart</button>
      </div>
    </div>

    <!-- Defer non-critical scripts -->
    <script defer src="cart.js"></script>
    <script defer src="reviews.js"></script>
    <script async src="analytics.js"></script>
  </body>
</html>
```

**Results:**

- LCP: 1.2s (product image loads fast)
- FCP: 0.8s (critical CSS renders immediately)
- CRP: Optimized (minimal blocking resources)

---

## ❓ 10. Interview Questions

### Q1: What is the Critical Rendering Path?

**Answer:**

The Critical Rendering Path is the sequence of steps the browser takes to render a page:

1. **Parse HTML** → Build DOM
2. **Parse CSS** → Build CSSOM (blocks rendering)
3. **Combine** → Create Render Tree (DOM + CSSOM)
4. **Layout** → Calculate positions/sizes
5. **Paint** → Fill pixels
6. **Composite** → Combine layers, display on screen

**Key points:**

- CSS blocks rendering (must have complete CSSOM)
- JavaScript blocks parsing (unless async/defer)
- Optimize by reducing blocking resources

**Follow-up:** What blocks rendering?

- External CSS (render-blocking)
- Synchronous JavaScript in `<head>` (parser-blocking)
- Web fonts without `font-display` (FOIT/FOUT)

---

### Q2: How do you optimize the Critical Rendering Path?

**Answer:**

**1. Minimize Blocking Resources:**

```html
<!-- Inline critical CSS -->
<style>
  /* above-the-fold styles */
</style>

<!-- Async non-critical CSS -->
<link
  rel="preload"
  href="styles.css"
  as="style"
  onload="this.rel='stylesheet'"
/>

<!-- Defer JavaScript -->
<script defer src="app.js"></script>
```

**2. Optimize Resource Loading:**

```html
<!-- Preconnect to external origins -->
<link rel="preconnect" href="https://cdn.example.com" />

<!-- Preload LCP image -->
<link rel="preload" href="hero.jpg" as="image" />

<!-- Priority hints -->
<img src="hero.jpg" fetchpriority="high" />
```

**3. Reduce Resource Size:**

- Minify CSS/JS
- Compress with gzip/brotli
- Optimize images (WebP, AVIF)
- Tree-shake unused code

**4. Use Resource Hints:**

- `preconnect` - Early connection
- `dns-prefetch` - Resolve DNS early
- `preload` - High-priority fetch
- `prefetch` - Next-page resources

**Metrics to track:**

- Time to First Byte (TTFB)
- First Contentful Paint (FCP)
- Largest Contentful Paint (LCP)
- DOM Content Loaded
- Load Complete

---

### Q3: What's the difference between async and defer?

**Answer:**

| Attribute | Download | Execute   | Blocks Parser            | Order        |
| --------- | -------- | --------- | ------------------------ | ------------ |
| None      | Blocking | Immediate | ✅ Yes                   | In order     |
| `async`   | Parallel | ASAP      | ❌ No (during execution) | Out of order |
| `defer`   | Parallel | After DOM | ❌ No                    | In order     |

**Visual:**

```
Normal:
HTML parsing ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                ↓
              <script> ■■■■■■■■■■ (blocks parsing)
                                ↓
HTML parsing ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Async:
HTML parsing ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
<script>         ↓↓↓↓↓↓↓↓■■ (executes when ready)

Defer:
HTML parsing ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
<script>         ↓↓↓↓↓↓↓↓          ■■ (after DOM)
```

**When to use:**

- `async` - Independent scripts (analytics, ads)
- `defer` - Scripts that depend on DOM or each other
- Module scripts - `type="module"` (deferred by default)

---

### Q4: Why does CSS block rendering?

**Answer:**

CSS blocks rendering to prevent **Flash of Unstyled Content (FOUC)**:

**Without blocking:**

```
1. Show unstyled HTML (FOUC)
2. CSS loads
3. Re-render with styles (jarring visual shift)
```

**With blocking:**

```
1. Wait for CSS
2. Show styled HTML (smooth experience)
```

**How it works:**

```javascript
// Browser's render engine
function render() {
  const dom = parseHTML(); // Non-blocking
  const cssom = parseCSS(); // BLOCKS!

  // Can't render until CSSOM ready
  if (!cssom.ready) {
    return; // Wait...
  }

  const renderTree = combine(dom, cssom);
  layout(renderTree);
  paint(renderTree);
}
```

**Optimization:**

```html
<!-- Critical CSS inline (no blocking fetch) -->
<style>
  .hero {
    /* ... */
  }
</style>

<!-- Non-critical async -->
<link
  rel="preload"
  href="styles.css"
  as="style"
  onload="this.rel='stylesheet'"
/>
```

---

### Q5: How do you measure Critical Rendering Path performance?

**Answer:**

**1. Performance API:**

```javascript
const perfData = performance.getEntriesByType("navigation")[0];

const metrics = {
  // DOM parsing complete
  domInteractive: perfData.domInteractive,

  // DOM + CSSOM ready (CRP complete)
  domContentLoaded: perfData.domContentLoadedEventEnd,

  // All resources loaded
  loadComplete: perfData.loadEventEnd,

  // CRP duration
  crpTime: perfData.domContentLoadedEventEnd - perfData.fetchStart,
};

console.table(metrics);
```

**2. Chrome DevTools:**

- Performance tab → Recording
- Look for "DCL" (DOM Content Loaded) marker
- Analyze blocking resources in waterfall
- Check "Coverage" tab for unused CSS

**3. Lighthouse:**

```bash
lighthouse https://example.com --only-categories=performance
```

**Metrics to optimize:**

- **TTFB** < 600ms (server response)
- **FCP** < 1.8s (first contentful paint)
- **LCP** < 2.5s (largest contentful paint)
- **DCL** < 2s (DOM + CSSOM ready)

**4. Web Vitals:**

```javascript
import { onLCP, onFCP, onCLS } from "web-vitals";

onLCP(console.log); // Largest Contentful Paint
onFCP(console.log); // First Contentful Paint
onCLS(console.log); // Cumulative Layout Shift
```

---

## 🧩 11. Design Patterns

### Pattern 1: Progressive Enhancement

```html
<!-- Basic HTML (works without CSS/JS) -->
<nav>
  <a href="#home">Home</a>
  <a href="#about">About</a>
</nav>

<!-- Enhanced with CSS -->
<style>
  nav {
    display: flex;
    gap: 1rem;
  }
</style>

<!-- Enhanced with JS -->
<script defer>
  // Add interactivity
  document.querySelector("nav").addEventListener("click", handleNav);
</script>
```

### Pattern 2: Resource Loading Strategy

```javascript
class ResourceLoader {
  constructor() {
    this.queue = [];
  }

  // Load critical resources immediately
  loadCritical(resources) {
    resources.forEach((resource) => {
      this.load(resource, "high");
    });
  }

  // Load non-critical when idle
  loadNonCritical(resources) {
    if ("requestIdleCallback" in window) {
      requestIdleCallback(() => {
        resources.forEach((resource) => this.load(resource, "low"));
      });
    } else {
      setTimeout(() => {
        resources.forEach((resource) => this.load(resource, "low"));
      }, 1000);
    }
  }

  load(resource, priority) {
    const link = document.createElement("link");
    link.rel = resource.type === "style" ? "stylesheet" : "preload";
    link.href = resource.url;
    link.as = resource.type;
    link.fetchpriority = priority;
    document.head.appendChild(link);
  }
}

// Usage
const loader = new ResourceLoader();
loader.loadCritical([
  { url: "critical.css", type: "style" },
  { url: "hero.jpg", type: "image" },
]);
loader.loadNonCritical([
  { url: "comments.css", type: "style" },
  { url: "social.js", type: "script" },
]);
```

---

## 🎯 Summary

The Critical Rendering Path is the foundation of web performance:

- **6 steps:** Parse HTML → Parse CSS → Render Tree → Layout → Paint → Composite
- **CSS blocks rendering** (prevents FOUC)
- **JavaScript blocks parsing** (unless async/defer)
- **Optimize by:** Inline critical CSS, defer JS, preload resources

**Production checklist:**

- ✅ Inline critical CSS (< 14KB)
- ✅ Defer non-critical JavaScript
- ✅ Preload LCP image
- ✅ Use resource hints (preconnect, dns-prefetch)
- ✅ Optimize web fonts (font-display: swap)
- ✅ Measure with Lighthouse/WebPageTest
- ✅ Monitor Core Web Vitals

Master the CRP to build blazing-fast web applications! 🚀
