# ⚡ Web Vitals & Performance Metrics

> Core Web Vitals are Google's performance metrics that directly impact SEO, user experience, and business metrics. Senior engineers must understand, measure, and optimize these.

---

## 📖 1. Concept Explanation

### What are Core Web Vitals?

Core Web Vitals are three key metrics that measure user experience:

1. **LCP (Largest Contentful Paint)** - Loading performance
2. **INP (Interaction to Next Paint)** - Interactivity (replaced FID in 2024)
3. **CLS (Cumulative Layout Shift)** - Visual stability

### LCP - Largest Contentful Paint

**What it measures:** Time until the largest content element is rendered.

**Target:** < 2.5 seconds (good), 2.5-4s (needs improvement), > 4s (poor)

**What counts as "largest content":**

- `<img>` elements
- `<image>` inside `<svg>`
- `<video>` poster images
- Background images (CSS)
- Block-level text elements

**Does NOT count:**

- Elements with opacity 0
- Elements that cover full viewport (likely background)
- Placeholder images

```html
<!-- LCP element candidates -->
<img src="hero.jpg" alt="Hero" />
<!-- Often LCP -->
<h1>Main Headline</h1>
<!-- Could be LCP -->
<div style="background-image: url(...)"><!-- Could be LCP --></div>
```

### INP - Interaction to Next Paint

**What it measures:** Responsiveness to user interactions (clicks, taps, key presses).

**Target:** < 200ms (good), 200-500ms (needs improvement), > 500ms (poor)

**How it works:**

- Measures time from user interaction to next paint
- Reports the worst interaction (or 98th percentile with many interactions)
- Includes: input delay + processing time + rendering delay

```
User clicks → Input delay → Event handler runs → Render → Paint
              ←────────────── INP ──────────────────────→
```

### CLS - Cumulative Layout Shift

**What it measures:** Visual stability (how much content shifts unexpectedly).

**Target:** < 0.1 (good), 0.1-0.25 (needs improvement), > 0.25 (poor)

**Calculation:**

```
CLS = Impact Fraction × Distance Fraction
```

**Example:**

- Element takes 50% of viewport (impact fraction = 0.5)
- Shifts down by 25% of viewport (distance fraction = 0.25)
- CLS = 0.5 × 0.25 = 0.125

---

## 🧠 2. Why It Matters in Real Projects

### Business Impact

**SEO Rankings:**

- Google uses Web Vitals as ranking signals
- Poor vitals = lower search rankings = less organic traffic

**Conversion Rates:**

- Amazon: 100ms delay = 1% revenue loss
- Pinterest: 40% faster LCP = 15% increase in signups
- BBC: Each additional second load time = 10% more users leave

**User Experience:**

- Poor INP: Users think app is broken (click multiple times)
- High CLS: Users click wrong buttons (ads, accidental purchases)
- Slow LCP: Users leave before seeing content

### Real-World Example: E-Commerce

```javascript
// ❌ POOR VITALS (before optimization)
// LCP: 4.5s - Hero image loads slowly
// INP: 350ms - Heavy JavaScript blocks interaction
// CLS: 0.35 - Ads shift content

// Impact:
// - 25% bounce rate increase
// - 18% conversion rate decrease
// - Page 3 in Google search

// ✅ OPTIMIZED (after optimization)
// LCP: 1.8s - Hero image optimized, preloaded
// INP: 120ms - Code split, lazy loaded
// CLS: 0.05 - Reserved space for ads

// Impact:
// - 15% bounce rate decrease
// - 22% conversion rate increase
// - Page 1 in Google search
```

---

## ⚙️ 3. Internal Working

### How LCP is Measured

Browser determines LCP through these steps:

1. **Parse HTML** - Identify content elements
2. **Track render times** - When each element paints
3. **Determine largest** - Which element takes most viewport space
4. **Update LCP** - As larger elements render, update the metric
5. **Finalize** - When user interacts or page hidden

```javascript
// Browser's internal LCP calculation (simplified)
let largestContentfulPaint = 0;
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1];
  largestContentfulPaint = lastEntry.renderTime || lastEntry.loadTime;
});

observer.observe({ type: "largest-contentful-paint", buffered: true });
```

### How INP is Calculated

```
INP = max(interaction_delays) or 98th_percentile(interaction_delays)

interaction_delay = presentation_time - event_time

Where:
- event_time = when user interacted
- presentation_time = when visual update appeared
```

**Phases:**

1. **Input Delay** - Time from interaction to event handler start
2. **Processing Time** - Event handler execution
3. **Presentation Delay** - Time from handler end to paint

```javascript
// What happens during a click:
button.addEventListener("click", () => {
  // Processing time starts here
  const data = heavyComputation(); // Blocks main thread
  updateUI(data); // Triggers re-render
  // Processing time ends, paint happens
});

// Good INP: < 200ms total
// Bad INP: > 500ms total (user feels lag)
```

### How CLS is Tracked

Browser tracks all layout shifts and sums their scores:

```javascript
// Browser's internal CLS tracking (simplified)
let cumulativeLayoutShift = 0;

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      // Ignore user-initiated shifts
      cumulativeLayoutShift += entry.value;
    }
  }
});

observer.observe({ type: "layout-shift", buffered: true });
```

---

## ✅ 4. Best Practices

### Optimize LCP ✅

```javascript
// 1. Preload critical resources
<link rel="preload" as="image" href="hero.jpg" />
<link rel="preload" as="font" href="font.woff2" crossorigin />

// 2. Use modern image formats
<picture>
  <source srcset="hero.webp" type="image/webp" />
  <source srcset="hero.avif" type="image/avif" />
  <img src="hero.jpg" alt="Hero" />
</picture>

// 3. Optimize images (responsive)
<img
  srcset="hero-320.jpg 320w, hero-640.jpg 640w, hero-1280.jpg 1280w"
  sizes="(max-width: 640px) 100vw, 640px"
  src="hero-640.jpg"
  alt="Hero"
/>

// 4. Inline critical CSS
<style>
  /* Critical above-the-fold styles */
  .hero { background: #000; height: 400px; }
</style>

// 5. Lazy load below-the-fold images
<img src="below-fold.jpg" loading="lazy" alt="..." />

// 6. Use CDN for assets
<img src="https://cdn.example.com/hero.jpg" alt="Hero" />

// 7. Reduce server response time (TTFB)
// - Use edge caching
// - Optimize database queries
// - Enable HTTP/3
```

### Optimize INP ✅

```javascript
// 1. Debounce expensive handlers
const handleSearch = debounce((query) => {
  const results = expensiveSearch(query);
  setResults(results);
}, 300);

<input onChange={(e) => handleSearch(e.target.value)} />;

// 2. Use Web Workers for heavy computation
// main.js
const worker = new Worker("worker.js");
worker.postMessage({ data: largeDataset });
worker.onmessage = (e) => {
  updateUI(e.data); // UI stays responsive
};

// worker.js
self.onmessage = (e) => {
  const result = heavyComputation(e.data);
  self.postMessage(result);
};

// 3. Code split and lazy load
const HeavyComponent = lazy(() => import("./HeavyComponent"));

// 4. Use React concurrent features
import { useTransition } from "react";

function SearchPage() {
  const [isPending, startTransition] = useTransition();

  const handleChange = (value) => {
    startTransition(() => {
      setResults(filterData(value)); // Non-urgent
    });
  };
}

// 5. Minimize main thread work
// ❌ BAD: Blocks main thread
function processData(items) {
  return items.map((item) => expensiveTransform(item));
}

// ✅ GOOD: Chunk processing
async function processData(items) {
  const CHUNK_SIZE = 50;
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE);
    await new Promise((resolve) => {
      setTimeout(() => {
        processChunk(chunk);
        resolve();
      }, 0); // Yield to browser
    });
  }
}

// 6. Use requestIdleCallback for non-urgent work
requestIdleCallback(() => {
  logAnalytics(); // Runs when browser is idle
});
```

### Optimize CLS ✅

```javascript
// 1. Reserve space for images
<img
  src="image.jpg"
  width="800"
  height="600"  // Aspect ratio preserved
  alt="..."
/>

// Or with CSS aspect ratio
<img
  src="image.jpg"
  style="aspect-ratio: 16/9; width: 100%;"
  alt="..."
/>

// 2. Reserve space for ads
<div class="ad-slot" style="min-height: 250px;">
  <!-- Ad loads here -->
</div>

// 3. Use font-display to prevent layout shift
@font-face {
  font-family: 'MyFont';
  src: url('font.woff2');
  font-display: swap; // Show fallback immediately
}

// 4. Avoid inserting content above existing content
// ❌ BAD: Prepends content
const newItem = createItem();
container.insertBefore(newItem, container.firstChild); // Shifts down

// ✅ GOOD: Append or use transform
container.appendChild(newItem);
// Or use CSS transform (doesn't trigger layout shift)

// 5. Use CSS containment
<div style="contain: layout;">
  <!-- Changes inside don't affect outside layout -->
</div>

// 6. Preload fonts
<link rel="preload" as="font" href="font.woff2" crossorigin />
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Not Reserving Space for Dynamic Content

```javascript
// ❌ CAUSES CLS
function ProductCard() {
  const [image, setImage] = useState(null);

  useEffect(() => {
    loadImage().then(setImage);
  }, []);

  return (
    <div>
      {image && <img src={image} />} {/* No height specified! */}
      <h3>Product Name</h3>
    </div>
  );
}

// ✅ FIXED
function ProductCard() {
  const [image, setImage] = useState(null);

  return (
    <div>
      <div style={{ aspectRatio: "1/1", background: "#eee" }}>
        {image && <img src={image} style={{ width: "100%" }} />}
      </div>
      <h3>Product Name</h3>
    </div>
  );
}
```

### Mistake #2: Heavy JavaScript in Event Handlers

```javascript
// ❌ POOR INP
<button onClick={() => {
  const data = fetchHugeDataset(); // Sync, blocks thread
  const processed = processData(data); // Blocks thread
  updateUI(processed); // Blocks thread
}}>
  Click Me
</button>

// ✅ OPTIMIZED
<button onClick={async () => {
  // Chunk work and use async
  startTransition(async () => {
    const data = await fetchHugeDataset();
    const processed = await processDataInChunks(data);
    updateUI(processed);
  });
}}>
  Click Me
</button>
```

### Mistake #3: Loading All Images Eagerly

```javascript
// ❌ SLOW LCP (loads all images immediately)
{
  products.map((product) => <img src={product.image} alt={product.name} />);
}

// ✅ OPTIMIZED (lazy load below fold)
{
  products.map((product, index) => (
    <img
      src={product.image}
      loading={index < 6 ? "eager" : "lazy"} // First 6 eager, rest lazy
      alt={product.name}
    />
  ));
}
```

### Mistake #4: Font Loading Causing Shift

```javascript
// ❌ CAUSES CLS
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2');
  font-display: block; // Invisible text, then flash
}

// ✅ PREVENTS CLS
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2');
  font-display: swap; // Fallback immediately, swap when loaded
}

// And preload:
<link rel="preload" as="font" href="font.woff2" crossorigin />
```

---

## 🔐 6. Security Considerations

### Resource Hints and Security

```html
<!-- ⚠️ Be careful with preconnect to third parties -->
<link rel="preconnect" href="https://third-party.com" />
<!-- Opens connection, could leak info -->

<!-- ✅ Use dns-prefetch for less trust -->
<link rel="dns-prefetch" href="https://third-party.com" />
<!-- Only resolves DNS, safer -->
```

### Content Security Policy (CSP) Impact

```javascript
// CSP can block inline resources, affecting LCP

// ❌ CSP blocks inline styles
<style>
  .hero { background-image: url(hero.jpg); }
</style>

// ✅ Use nonce or hash
<meta http-equiv="Content-Security-Policy"
      content="style-src 'nonce-2726c7f26c'">
<style nonce="2726c7f26c">
  .hero { background-image: url(hero.jpg); }
</style>
```

---

## 🚀 7. Performance Optimization

### Measuring Web Vitals

```javascript
// 1. Using web-vitals library (recommended)
import { onCLS, onINP, onLCP } from "web-vitals";

onCLS(console.log);
onINP(console.log);
onLCP(console.log);

// 2. Send to analytics
onLCP((metric) => {
  gtag("event", "web_vitals", {
    event_category: "Web Vitals",
    event_label: metric.name,
    value: Math.round(metric.value),
    non_interaction: true,
  });
});

// 3. Manual measurement
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log("LCP:", entry.renderTime || entry.loadTime);
  }
});
observer.observe({ type: "largest-contentful-paint", buffered: true });
```

### Optimizing with Performance API

```javascript
// Monitor long tasks (INP indicators)
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      // >50ms = long task
      console.warn("Long task detected:", entry.duration, entry.name);
      // Send to monitoring service
    }
  }
});
observer.observe({ type: "longtask", buffered: true });

// Monitor layout shifts
const clsObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      console.log("Layout shift:", entry.value, entry.sources);
      // Debug which elements caused shift
    }
  }
});
clsObserver.observe({ type: "layout-shift", buffered: true });
```

### Image Optimization Strategies

```javascript
// 1. Responsive images with modern formats
<picture>
  <source
    type="image/avif"
    srcset="hero-320.avif 320w, hero-640.avif 640w, hero-1280.avif 1280w"
  />
  <source
    type="image/webp"
    srcset="hero-320.webp 320w, hero-640.webp 640w, hero-1280.webp 1280w"
  />
  <img
    src="hero-640.jpg"
    srcset="hero-320.jpg 320w, hero-640.jpg 640w, hero-1280.jpg 1280w"
    sizes="(max-width: 640px) 100vw, 640px"
    alt="Hero"
    width="640"
    height="360"
  />
</picture>;

// 2. Lazy loading with intersection observer
const images = document.querySelectorAll("img[data-src]");
const imageObserver = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      imageObserver.unobserve(img);
    }
  });
});

images.forEach((img) => imageObserver.observe(img));

// 3. Blur-up technique
<div style="position: relative;">
  <img src="tiny-blur.jpg" style="filter: blur(10px); position: absolute;" />
  <img
    src="full-quality.jpg"
    loading="lazy"
    onLoad={(e) => e.target.previousSibling.remove()}
  />
</div>;
```

---

## 🧪 8. Code Examples

### Example 1: LCP Optimization

```javascript
// Before: LCP = 4.2s
function Hero() {
  const [image, setImage] = useState(null);

  useEffect(() => {
    import("./hero.jpg").then(setImage);
  }, []);

  return <div>{image && <img src={image} alt="Hero" />}</div>;
}

// After: LCP = 1.5s
function Hero() {
  return (
    <picture>
      <source srcset="/hero.avif" type="image/avif" />
      <source srcset="/hero.webp" type="image/webp" />
      <img
        src="/hero.jpg"
        alt="Hero"
        width="1200"
        height="600"
        fetchpriority="high" // Prioritize loading
      />
    </picture>
  );
}

// And in HTML head:
<link rel="preload" as="image" href="/hero.avif" />;
```

### Example 2: INP Optimization

```javascript
// Before: INP = 450ms
function SearchResults({ query }) {
  const [results, setResults] = useState([]);

  const handleSearch = (newQuery) => {
    // Heavy sync computation
    const filtered = items.filter((item) =>
      item.name.toLowerCase().includes(newQuery.toLowerCase()),
    );
    const sorted = filtered.sort((a, b) => a.score - b.score);
    const enriched = sorted.map((item) => enrichItem(item));
    setResults(enriched);
  };

  return <input onChange={(e) => handleSearch(e.target.value)} />;
}

// After: INP = 120ms
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (newQuery) => {
    startTransition(() => {
      // Mark as non-urgent, allow yielding
      const filtered = items.filter((item) =>
        item.name.toLowerCase().includes(newQuery.toLowerCase()),
      );
      const sorted = filtered.sort((a, b) => a.score - b.score);
      const enriched = sorted.map((item) => enrichItem(item));
      setResults(enriched);
    });
  };

  return (
    <>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  );
}
```

### Example 3: CLS Prevention

```javascript
// Before: CLS = 0.28
function ProductGrid() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    fetchProducts().then(setProducts);
  }, []);

  return (
    <div className="grid">
      {products.map((product) => (
        <div key={product.id}>
          <img src={product.image} alt={product.name} />
          <h3>{product.name}</h3>
        </div>
      ))}
    </div>
  );
}

// After: CLS = 0.02
function ProductGrid() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    fetchProducts().then(setProducts);
  }, []);

  return (
    <div className="grid">
      {products.length === 0 &&
        // Skeleton with same dimensions
        Array(12)
          .fill(0)
          .map((_, i) => (
            <div key={i} className="skeleton">
              <div style={{ aspectRatio: "1/1", background: "#eee" }} />
              <div
                style={{ height: "20px", background: "#eee", marginTop: "8px" }}
              />
            </div>
          ))}

      {products.map((product) => (
        <div key={product.id}>
          <img
            src={product.image}
            alt={product.name}
            width="300"
            height="300"
            style={{ aspectRatio: "1/1" }}
          />
          <h3>{product.name}</h3>
        </div>
      ))}
    </div>
  );
}
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: News Website

```javascript
// Challenge: Fast LCP for hero article image

// Solution:
// 1. Preload hero image
<link rel="preload" as="image" href="/hero.webp" />

// 2. Use picture element
<picture>
  <source
    media="(max-width: 640px)"
    srcset="/hero-mobile.webp"
  />
  <source
    media="(min-width: 641px)"
    srcset="/hero-desktop.webp"
  />
  <img src="/hero-desktop.jpg" alt="..." fetchpriority="high" />
</picture>

// 3. Inline critical CSS
<style>
  .hero-container { height: 400px; background: #f0f0f0; }
</style>

// Result: LCP improved from 3.8s to 1.4s
```

### Use Case 2: E-Commerce Product Listing

```javascript
// Challenge: Good INP while filtering 10,000 products

// Solution:
function ProductListing() {
  const [filters, setFilters] = useState({});
  const [isPending, startTransition] = useTransition();
  const deferredFilters = useDeferredValue(filters);

  // Filtering uses deferred value
  const filteredProducts = useMemo(() => {
    return products.filter((p) => matchesFilters(p, deferredFilters));
  }, [products, deferredFilters]);

  const handleFilterChange = (newFilters) => {
    // Immediate: Update filter UI
    setFilters(newFilters);

    // Deferred: Heavy filtering
    // React can pause this if user interacts again
  };

  return (
    <>
      <Filters onChange={handleFilterChange} />
      <div className={isPending ? "opacity-50" : ""}>
        <ProductGrid products={filteredProducts} />
      </div>
    </>
  );
}

// Result: INP improved from 520ms to 180ms
```

### Use Case 3: Dashboard with Dynamic Charts

```javascript
// Challenge: Prevent CLS when charts load

// Solution:
function Dashboard() {
  const [chartData, setChartData] = useState(null);

  return (
    <div className="dashboard">
      {/* Reserve space with min-height */}
      <div style={{ minHeight: '400px' }}>
        {chartData ? (
          <Chart data={chartData} />
        ) : (
          <Skeleton height="400px" /> {/* Same size as chart */}
        )}
      </div>
    </div>
  );
}

// Result: CLS improved from 0.32 to 0.05
```

---

## ❓ 10. Interview Questions

### Q1: What are Core Web Vitals and why do they matter?

**Answer:**

Core Web Vitals are three metrics that measure real user experience:

1. **LCP (Largest Contentful Paint)** - Loading speed (< 2.5s)
2. **INP (Interaction to Next Paint)** - Responsiveness (< 200ms)
3. **CLS (Cumulative Layout Shift)** - Visual stability (< 0.1)

**Why they matter:**

**Business:**

- Google ranking factor (SEO)
- Direct correlation with conversion rates
- User retention and engagement

**Technical:**

- Objective measures of UX
- Identify performance bottlenecks
- Track improvements over time

**Example impact:**

- Vodafone: Improved LCP by 31% → 8% more sales
- Yahoo Japan: Improved CLS → 15.1% more page views

**Follow-up:** How do you measure them?

- RUM (Real User Monitoring): web-vitals library + analytics
- Lab data: Lighthouse, PageSpeed Insights
- Field data: Chrome User Experience Report

---

### Q2: How would you optimize LCP for a hero image?

**Answer:**

**Strategies:**

1. **Preload the image:**

```html
<link rel="preload" as="image" href="hero.webp" />
```

2. **Use modern formats:**

```html
<picture>
  <source srcset="hero.avif" type="image/avif" />
  <source srcset="hero.webp" type="image/webp" />
  <img src="hero.jpg" alt="Hero" />
</picture>
```

3. **Set fetchpriority:**

```html
<img src="hero.jpg" fetchpriority="high" />
```

4. **Optimize image size:**

- Compress (TinyPNG, ImageOptim)
- Responsive sizes (srcset)
- Serve from CDN

5. **Reduce TTFB:**

- Edge caching
- CDN
- HTTP/3

6. **Avoid render-blocking resources:**

```html
<link
  rel="stylesheet"
  href="styles.css"
  media="print"
  onload="this.media='all'"
/>
```

**Result:** Typically see 40-60% LCP improvement.

---

### Q3: What causes high INP and how do you fix it?

**Answer:**

**Causes:**

1. **Heavy JavaScript execution:**

```javascript
// ❌ Blocks main thread
button.onclick = () => {
  const result = processLargeDataset(); // 300ms
  updateUI(result);
};
```

2. **Large DOM updates:**

```javascript
// ❌ Updates 1000 nodes
items.forEach((item) => {
  container.appendChild(createNode(item));
});
```

3. **Layout thrashing:**

```javascript
// ❌ Forces multiple reflows
elements.forEach((el) => {
  el.style.width = el.offsetWidth + 10 + "px"; // Read then write
});
```

**Fixes:**

1. **Use Web Workers:**

```javascript
const worker = new Worker("process.js");
worker.postMessage(data);
worker.onmessage = (e) => updateUI(e.data);
```

2. **Chunk heavy work:**

```javascript
startTransition(() => {
  // React can pause this
  updateResults(filteredData);
});
```

3. **Debounce/throttle:**

```javascript
const handleInput = debounce(processInput, 300);
```

4. **Virtual scrolling:**

```javascript
<VirtualList
  items={10000}
  renderItem={(index) => <Item data={items[index]} />}
/>
```

---

### Q4: How do you prevent CLS?

**Answer:**

**Key strategies:**

1. **Reserve space for images:**

```html
<img src="..." width="800" height="600" alt="..." />
<!-- Or -->
<img src="..." style="aspect-ratio: 16/9" alt="..." />
```

2. **Use font-display: swap:**

```css
@font-face {
  font-family: "MyFont";
  font-display: swap; /* Show fallback immediately */
}
```

3. **Reserve space for ads:**

```html
<div class="ad-container" style="min-height: 250px;">
  <!-- Ad loads here -->
</div>
```

4. **Avoid inserting content above existing:**

```javascript
// ❌ Shifts content down
container.prepend(newElement);

// ✅ Append or use absolute positioning
container.append(newElement);
```

5. **Preload fonts:**

```html
<link rel="preload" as="font" href="font.woff2" crossorigin />
```

6. **Use transform instead of top/left:**

```css
/* ❌ Triggers layout */
.slide {
  left: 0;
  transition: left 0.3s;
}

/* ✅ Doesn't trigger layout */
.slide {
  transform: translateX(0);
  transition: transform 0.3s;
}
```

---

### Q5: How do you monitor Web Vitals in production?

**Answer:**

**Implementation:**

```javascript
// 1. Install web-vitals
import { onCLS, onINP, onLCP } from "web-vitals";

// 2. Send to analytics
function sendToAnalytics(metric) {
  const body = JSON.stringify(metric);

  // Use sendBeacon (doesn't block unload)
  if (navigator.sendBeacon) {
    navigator.sendBeacon("/analytics", body);
  } else {
    fetch("/analytics", { body, method: "POST", keepalive: true });
  }
}

onCLS(sendToAnalytics);
onINP(sendToAnalytics);
onLCP(sendToAnalytics);

// 3. Attribution (find what caused issues)
import { onCLS } from "web-vitals/attribution";

onCLS((metric) => {
  console.log("CLS:", metric.value);
  console.log("Largest shift:", metric.attribution.largestShiftEntry);
  console.log("Element:", metric.attribution.largestShiftTarget);
});
```

**Tools:**

- Google Analytics
- Datadog RUM
- New Relic
- Sentry Performance
- Custom solution

**Dashboard:**

- P75 values (75th percentile)
- Trends over time
- By page/route
- By device type
- By geography

---

## 🧩 11. Design Patterns

### Pattern 1: Progressive Enhancement

```javascript
// Start with core content, enhance with features
function ProductPage() {
  return (
    <>
      {/* Core content loads fast (good LCP) */}
      <img src="product.jpg" width="600" height="600" alt="Product" />
      <h1>Product Name</h1>
      <p>Price: $99</p>

      {/* Enhanced features load later */}
      <Suspense fallback={<Skeleton />}>
        <RelatedProducts />
      </Suspense>

      <Suspense fallback={<Skeleton />}>
        <Reviews />
      </Suspense>
    </>
  );
}
```

### Pattern 2: Skeleton Screens

```javascript
// Prevent CLS with skeleton placeholders
function UserList() {
  const { data, loading } = useQuery(GET_USERS);

  if (loading) {
    return Array(5)
      .fill(0)
      .map((_, i) => (
        <div key={i} className="user-skeleton">
          <div className="avatar-skeleton" /> {/* Same size as real avatar */}
          <div className="name-skeleton" /> {/* Same size as real name */}
        </div>
      ));
  }

  return data.map((user) => <UserCard key={user.id} user={user} />);
}
```

---

## 🎯 Summary

Core Web Vitals directly impact:

- **SEO rankings** (Google search)
- **User experience** (speed, responsiveness, stability)
- **Business metrics** (conversions, revenue, engagement)

**Key metrics:**

- **LCP** < 2.5s - Optimize images, preload, CDN
- **INP** < 200ms - Chunk work, Web Workers, lazy load
- **CLS** < 0.1 - Reserve space, font-display, avoid shifts

**Production checklist:**

- ✅ Measure with web-vitals library
- ✅ Monitor in real-time (RUM)
- ✅ Set performance budgets
- ✅ Test on real devices
- ✅ Optimize critical path
- ✅ Use modern formats (WebP, AVIF)
- ✅ Implement progressive enhancement

**Interview tip:** Always tie performance metrics to business outcomes. Don't just say "we improved LCP by 2s"—say "we improved LCP by 2s, which increased conversions by 15%."
