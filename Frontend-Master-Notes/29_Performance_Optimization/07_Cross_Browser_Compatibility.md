# 🌐 Cross-Browser Compatibility

> **Building Web Applications That Work Consistently Across All Browsers**

---

## 📋 Table of Contents

- [Introduction](#introduction)
- [Browser Rendering Engines](#browser-rendering-engines)
- [Feature Detection](#feature-detection)
- [Polyfills & Transpilation](#polyfills--transpilation)
- [CSS Cross-Browser Strategies](#css-cross-browser-strategies)
- [JavaScript Cross-Browser Patterns](#javascript-cross-browser-patterns)
- [Testing Strategies](#testing-strategies)
- [Common Browser Issues](#common-browser-issues)
- [Progressive Enhancement](#progressive-enhancement)
- [Graceful Degradation](#graceful-degradation)
- [Browser-Specific Workarounds](#browser-specific-workarounds)
- [Best Practices](#best-practices)
- [Testing Tools](#testing-tools)
- [Interview Questions](#interview-questions)

---

## Introduction

**Cross-browser compatibility** ensures your web application works consistently across different browsers, versions, and platforms. With diverse rendering engines and feature support, developers must write defensive code and implement fallbacks.

### Why Cross-Browser Compatibility Matters

```typescript
interface BrowserMarketShare {
  browser: string;
  engine: string;
  marketShare: number;
  versions: string[];
}

const browserLandscape2024: BrowserMarketShare[] = [
  {
    browser: "Chrome",
    engine: "Blink",
    marketShare: 64.5,
    versions: ["120+", "119", "118"],
  },
  {
    browser: "Safari",
    engine: "WebKit",
    marketShare: 18.2,
    versions: ["17+", "16", "15"],
  },
  {
    browser: "Edge",
    engine: "Blink",
    marketShare: 5.1,
    versions: ["120+", "119", "118"],
  },
  {
    browser: "Firefox",
    engine: "Gecko",
    marketShare: 3.2,
    versions: ["121+", "120", "119"],
  },
  {
    browser: "Samsung Internet",
    engine: "Blink",
    marketShare: 2.8,
    versions: ["23+", "22", "21"],
  },
  {
    browser: "Opera",
    engine: "Blink",
    marketShare: 2.1,
    versions: ["105+", "104", "103"],
  },
];
```

---

## Browser Rendering Engines

### Major Rendering Engines

```typescript
interface RenderingEngine {
  name: string;
  browsers: string[];
  cssPrefix: string;
  features: string[];
  quirks: string[];
}

const renderingEngines: RenderingEngine[] = [
  {
    name: "Blink",
    browsers: ["Chrome", "Edge", "Opera", "Brave", "Samsung Internet"],
    cssPrefix: "-webkit-",
    features: [
      "Modern CSS Grid",
      "Container Queries",
      "CSS Cascade Layers",
      "View Transitions API",
    ],
    quirks: ["Aggressive caching", "Different date parsing"],
  },
  {
    name: "WebKit",
    browsers: ["Safari", "iOS Safari"],
    cssPrefix: "-webkit-",
    features: ["CSS Grid (with gaps)", "Modern Flexbox", "CSS Variables"],
    quirks: [
      "No support for some modern features",
      "IndexedDB limitations",
      "Service Worker restrictions",
      "Date input handling differences",
    ],
  },
  {
    name: "Gecko",
    browsers: ["Firefox"],
    cssPrefix: "-moz-",
    features: [
      "Excellent standards compliance",
      "Privacy features",
      "Container Queries",
    ],
    quirks: ["Different scrollbar styling", "Unique input autofill behavior"],
  },
];

// Check rendering engine
function detectEngine(): string {
  const ua = navigator.userAgent;

  if (ua.includes("Chrome") && !ua.includes("Edge")) return "Blink";
  if (ua.includes("Safari") && !ua.includes("Chrome")) return "WebKit";
  if (ua.includes("Firefox")) return "Gecko";
  if (ua.includes("Edg")) return "Blink"; // New Edge

  return "Unknown";
}
```

---

## Feature Detection

### Modern Feature Detection

```typescript
/**
 * Feature detection utility
 * NEVER use browser sniffing - always detect features
 */
class FeatureDetector {
  // CSS Feature Detection
  static supportsCSS(property: string, value: string): boolean {
    if (typeof CSS !== "undefined" && CSS.supports) {
      return CSS.supports(property, value);
    }

    // Fallback
    const element = document.createElement("div");
    element.style[property as any] = value;
    return element.style[property as any] === value;
  }

  // Check for CSS Grid support
  static get hasGrid(): boolean {
    return this.supportsCSS("display", "grid");
  }

  // Check for container queries
  static get hasContainerQueries(): boolean {
    return this.supportsCSS("container-type", "inline-size");
  }

  // Check for backdrop-filter
  static get hasBackdropFilter(): boolean {
    return (
      this.supportsCSS("backdrop-filter", "blur(10px)") ||
      this.supportsCSS("-webkit-backdrop-filter", "blur(10px)")
    );
  }

  // JavaScript API Detection
  static hasAPI(apiName: string): boolean {
    const apis: Record<string, () => boolean> = {
      IntersectionObserver: () => "IntersectionObserver" in window,
      ResizeObserver: () => "ResizeObserver" in window,
      fetch: () => "fetch" in window,
      localStorage: () => {
        try {
          const test = "__test__";
          localStorage.setItem(test, test);
          localStorage.removeItem(test);
          return true;
        } catch {
          return false;
        }
      },
      serviceWorker: () => "serviceWorker" in navigator,
      webGL: () => {
        try {
          const canvas = document.createElement("canvas");
          return !!(
            canvas.getContext("webgl") ||
            canvas.getContext("experimental-webgl")
          );
        } catch {
          return false;
        }
      },
      webAssembly: () => typeof WebAssembly !== "undefined",
      indexedDB: () => "indexedDB" in window,
      webRTC: () =>
        "RTCPeerConnection" in window || "webkitRTCPeerConnection" in window,
    };

    return apis[apiName]?.() ?? false;
  }

  // Touch support
  static get hasTouch(): boolean {
    return (
      "ontouchstart" in window ||
      navigator.maxTouchPoints > 0 ||
      (navigator as any).msMaxTouchPoints > 0
    );
  }

  // Check for modern JavaScript features
  static get hasModernJS(): boolean {
    try {
      // Test for ES6+ features
      eval("const x = () => {}; class Test {}; [...[1]]");
      return true;
    } catch {
      return false;
    }
  }
}

// Usage
if (FeatureDetector.hasGrid) {
  // Use CSS Grid
} else {
  // Use Flexbox fallback
}

if (FeatureDetector.hasAPI("IntersectionObserver")) {
  // Use Intersection Observer
} else {
  // Use scroll event fallback
}
```

### Feature Detection with @supports

```css
/* ✅ Feature query for CSS Grid */
@supports (display: grid) {
  .container {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  }
}

/* ✅ Fallback for older browsers */
@supports not (display: grid) {
  .container {
    display: flex;
    flex-wrap: wrap;
  }

  .container > * {
    flex: 1 1 250px;
  }
}

/* ✅ Multiple condition checks */
@supports (backdrop-filter: blur(10px)) or (-webkit-backdrop-filter: blur(10px)) {
  .glassmorphism {
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
  }
}

/* ✅ Combine conditions */
@supports (display: grid) and (gap: 1rem) {
  .modern-grid {
    display: grid;
    gap: 1rem;
  }
}
```

---

## Polyfills & Transpilation

### Polyfill Strategy

```typescript
/**
 * Smart polyfill loader
 * Only load polyfills if needed
 */
class PolyfillLoader {
  // Check what needs to be polyfilled
  static async loadIfNeeded(): Promise<void> {
    const polyfills: Array<{
      test: boolean;
      url: string;
      name: string;
    }> = [];

    // Check Promise support
    if (typeof Promise === "undefined") {
      polyfills.push({
        test: true,
        url: "https://cdn.jsdelivr.net/npm/promise-polyfill",
        name: "Promise",
      });
    }

    // Check fetch support
    if (!("fetch" in window)) {
      polyfills.push({
        test: true,
        url: "https://cdn.jsdelivr.net/npm/whatwg-fetch",
        name: "fetch",
      });
    }

    // Check IntersectionObserver
    if (!("IntersectionObserver" in window)) {
      polyfills.push({
        test: true,
        url: "https://cdn.jsdelivr.net/npm/intersection-observer",
        name: "IntersectionObserver",
      });
    }

    // Load all required polyfills
    await Promise.all(polyfills.map((p) => this.loadScript(p.url, p.name)));
  }

  private static loadScript(url: string, name: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const script = document.createElement("script");
      script.src = url;
      script.async = true;

      script.onload = () => {
        console.log(`✅ Loaded polyfill: ${name}`);
        resolve();
      };

      script.onerror = () => {
        console.error(`❌ Failed to load polyfill: ${name}`);
        reject();
      };

      document.head.appendChild(script);
    });
  }
}

// Initialize polyfills before app starts
PolyfillLoader.loadIfNeeded().then(() => {
  // Start your application
  initApp();
});
```

### Babel Configuration for Cross-Browser Support

```javascript
// babel.config.js
module.exports = {
  presets: [
    [
      "@babel/preset-env",
      {
        // Target browsers
        targets: {
          browsers: [
            "> 0.5%", // Browsers with >0.5% market share
            "last 2 versions", // Last 2 versions of each browser
            "not dead", // Not discontinued browsers
            "not ie <= 11", // No IE11 or below
          ],
        },
        // Only include polyfills that are needed
        useBuiltIns: "usage",
        corejs: 3,
        // Debug to see what's being transpiled
        debug: false,
      },
    ],
    "@babel/preset-typescript",
    "@babel/preset-react",
  ],
  plugins: [
    "@babel/plugin-proposal-class-properties",
    "@babel/plugin-proposal-optional-chaining",
    "@babel/plugin-proposal-nullish-coalescing-operator",
  ],
};
```

### Core-js Polyfills

```typescript
// polyfills.ts - Selective polyfill imports
import "core-js/stable/array/from";
import "core-js/stable/array/find";
import "core-js/stable/array/includes";
import "core-js/stable/object/assign";
import "core-js/stable/object/entries";
import "core-js/stable/object/values";
import "core-js/stable/promise";
import "core-js/stable/string/includes";
import "core-js/stable/string/starts-with";
import "core-js/stable/string/ends-with";
import "core-js/stable/symbol";

// Web APIs
import "whatwg-fetch"; // fetch API
import "intersection-observer"; // IntersectionObserver

// Only for older browsers
if (!Element.prototype.matches) {
  Element.prototype.matches =
    (Element.prototype as any).msMatchesSelector ||
    Element.prototype.webkitMatchesSelector;
}

if (!Element.prototype.closest) {
  Element.prototype.closest = function (selector: string): Element | null {
    let el: Element | null = this;
    while (el) {
      if (el.matches(selector)) return el;
      el = el.parentElement;
    }
    return null;
  };
}
```

---

## CSS Cross-Browser Strategies

### Vendor Prefixes

```css
/* ✅ Autoprefixer handles this automatically */
.element {
  /* Modern syntax */
  display: flex;
  user-select: none;
  backdrop-filter: blur(10px);
}

/* ✅ After Autoprefixer processing */
.element {
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;

  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;

  -webkit-backdrop-filter: blur(10px);
  backdrop-filter: blur(10px);
}
```

### CSS Reset for Consistency

```css
/* ✅ Modern CSS Reset */
*,
*::before,
*::after {
  box-sizing: border-box;
}

* {
  margin: 0;
  padding: 0;
}

html {
  /* Prevent font size inflation on mobile */
  -webkit-text-size-adjust: 100%;
  -moz-text-size-adjust: 100%;
  text-size-adjust: 100%;
}

body {
  min-height: 100vh;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

img,
picture,
video,
canvas,
svg {
  display: block;
  max-width: 100%;
}

input,
button,
textarea,
select {
  font: inherit;
}

p,
h1,
h2,
h3,
h4,
h5,
h6 {
  overflow-wrap: break-word;
}
```

### Browser-Specific CSS

```css
/* ✅ Safari-specific fixes */
@supports (-webkit-appearance: none) {
  input[type="date"]::-webkit-inner-spin-button,
  input[type="date"]::-webkit-calendar-picker-indicator {
    display: none;
    -webkit-appearance: none;
  }
}

/* ✅ Firefox-specific */
@-moz-document url-prefix() {
  .firefox-only {
    /* Firefox-specific styles */
  }
}

/* ✅ IE/Edge specific (old Edge) */
@supports (-ms-ime-align: auto) {
  .edge-only {
    /* Edge-specific styles */
  }
}

/* ✅ Chromium-specific */
@media screen and (-webkit-min-device-pixel-ratio: 0) {
  .chrome-only {
    /* Chrome/Edge/Opera styles */
  }
}
```

---

## JavaScript Cross-Browser Patterns

### Safe DOM Manipulation

```typescript
/**
 * Cross-browser DOM utilities
 */
class DOMUtils {
  // Safe event listener
  static addEventListener(
    element: Element,
    event: string,
    handler: EventListener,
    options?: AddEventListenerOptions,
  ): void {
    if (element.addEventListener) {
      element.addEventListener(event, handler, options);
    } else if ((element as any).attachEvent) {
      // IE8 and below
      (element as any).attachEvent(`on${event}`, handler);
    }
  }

  // Safe event removal
  static removeEventListener(
    element: Element,
    event: string,
    handler: EventListener,
  ): void {
    if (element.removeEventListener) {
      element.removeEventListener(event, handler);
    } else if ((element as any).detachEvent) {
      (element as any).detachEvent(`on${event}`, handler);
    }
  }

  // Get computed style safely
  static getComputedStyle(element: Element): CSSStyleDeclaration {
    if (window.getComputedStyle) {
      return window.getComputedStyle(element);
    } else if ((element as any).currentStyle) {
      // IE8 and below
      return (element as any).currentStyle;
    }
    return {} as CSSStyleDeclaration;
  }

  // Safe classList operations
  static addClass(element: Element, className: string): void {
    if (element.classList) {
      element.classList.add(className);
    } else {
      const classes = element.className.split(" ");
      if (classes.indexOf(className) === -1) {
        element.className += ` ${className}`;
      }
    }
  }

  static removeClass(element: Element, className: string): void {
    if (element.classList) {
      element.classList.remove(className);
    } else {
      const classes = element.className.split(" ");
      const index = classes.indexOf(className);
      if (index > -1) {
        classes.splice(index, 1);
        element.className = classes.join(" ");
      }
    }
  }

  // Safe query selector
  static querySelector(
    selector: string,
    context: Document | Element = document,
  ): Element | null {
    if (context.querySelector) {
      return context.querySelector(selector);
    } else if ((context as any).querySelectorAll) {
      const elements = context.querySelectorAll(selector);
      return elements.length > 0 ? elements[0] : null;
    }
    return null;
  }
}
```

### Date Handling Cross-Browser

```typescript
/**
 * Safe date parsing across browsers
 * Safari and Firefox parse dates differently!
 */
class DateUtils {
  // ❌ DON'T: Different results in different browsers
  static unsafeParse(dateString: string): Date {
    return new Date(dateString); // Unpredictable!
  }

  // ✅ DO: Safe date parsing
  static safeParse(dateString: string): Date | null {
    // Parse ISO 8601 format: YYYY-MM-DD or YYYY-MM-DDTHH:mm:ss
    const isoRegex = /^(\d{4})-(\d{2})-(\d{2})(T(\d{2}):(\d{2}):(\d{2}))?/;
    const match = dateString.match(isoRegex);

    if (match) {
      const [, year, month, day, , hour, minute, second] = match;
      return new Date(
        parseInt(year),
        parseInt(month) - 1, // Months are 0-indexed
        parseInt(day),
        parseInt(hour || "0"),
        parseInt(minute || "0"),
        parseInt(second || "0"),
      );
    }

    // Fallback to Date.parse with validation
    const timestamp = Date.parse(dateString);
    return isNaN(timestamp) ? null : new Date(timestamp);
  }

  // Format date safely
  static format(date: Date, format: string = "YYYY-MM-DD"): string {
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, "0");
    const day = String(date.getDate()).padStart(2, "0");
    const hours = String(date.getHours()).padStart(2, "0");
    const minutes = String(date.getMinutes()).padStart(2, "0");
    const seconds = String(date.getSeconds()).padStart(2, "0");

    return format
      .replace("YYYY", String(year))
      .replace("MM", month)
      .replace("DD", day)
      .replace("HH", hours)
      .replace("mm", minutes)
      .replace("ss", seconds);
  }
}

// Usage
const date = DateUtils.safeParse("2024-12-25");
console.log(DateUtils.format(date!, "YYYY-MM-DD HH:mm:ss"));
```

### Number Formatting

```typescript
/**
 * Cross-browser number formatting
 */
class NumberUtils {
  // Use Intl API with fallback
  static format(value: number, options: Intl.NumberFormatOptions = {}): string {
    if (typeof Intl !== "undefined" && Intl.NumberFormat) {
      return new Intl.NumberFormat("en-US", options).format(value);
    }

    // Fallback for older browsers
    return value.toLocaleString();
  }

  // Currency formatting
  static formatCurrency(value: number, currency: string = "USD"): string {
    return this.format(value, {
      style: "currency",
      currency,
    });
  }

  // Percentage formatting
  static formatPercent(value: number, decimals: number = 0): string {
    return this.format(value / 100, {
      style: "percent",
      minimumFractionDigits: decimals,
      maximumFractionDigits: decimals,
    });
  }
}

// Usage
console.log(NumberUtils.formatCurrency(1234.56)); // "$1,234.56"
console.log(NumberUtils.formatPercent(75.5, 1)); // "75.5%"
```

---

## Testing Strategies

### Browser Testing Matrix

```typescript
interface BrowserTestConfig {
  browser: string;
  versions: string[];
  platforms: string[];
  priority: "critical" | "high" | "medium" | "low";
  testCases: string[];
}

const browserTestMatrix: BrowserTestConfig[] = [
  {
    browser: "Chrome",
    versions: ["latest", "latest-1"],
    platforms: ["Windows", "macOS", "Android"],
    priority: "critical",
    testCases: [
      "Core functionality",
      "Responsive layout",
      "Form validation",
      "Animation performance",
    ],
  },
  {
    browser: "Safari",
    versions: ["latest", "latest-1"],
    platforms: ["macOS", "iOS"],
    priority: "critical",
    testCases: [
      "Core functionality",
      "Touch interactions",
      "Date picker behavior",
      "Video playback",
    ],
  },
  {
    browser: "Firefox",
    versions: ["latest", "latest-1"],
    platforms: ["Windows", "macOS"],
    priority: "high",
    testCases: ["Core functionality", "CSS Grid layout", "WebGL rendering"],
  },
  {
    browser: "Edge",
    versions: ["latest"],
    platforms: ["Windows"],
    priority: "high",
    testCases: ["Core functionality", "Accessibility features"],
  },
  {
    browser: "Samsung Internet",
    versions: ["latest"],
    platforms: ["Android"],
    priority: "medium",
    testCases: ["Mobile layout", "Touch gestures"],
  },
];
```

### Automated Cross-Browser Testing

```typescript
// playwright.config.ts
import { PlaywrightTestConfig } from "@playwright/test";

const config: PlaywrightTestConfig = {
  testDir: "./tests",
  timeout: 30000,
  retries: 2,

  use: {
    baseURL: "http://localhost:3000",
    screenshot: "only-on-failure",
    video: "retain-on-failure",
  },

  // Test across multiple browsers
  projects: [
    {
      name: "Chrome",
      use: { browserName: "chromium" },
    },
    {
      name: "Firefox",
      use: { browserName: "firefox" },
    },
    {
      name: "Safari",
      use: { browserName: "webkit" },
    },
    {
      name: "Mobile Chrome",
      use: {
        browserName: "chromium",
        ...devices["Pixel 5"],
      },
    },
    {
      name: "Mobile Safari",
      use: {
        browserName: "webkit",
        ...devices["iPhone 12"],
      },
    },
  ],
};

export default config;
```

```typescript
// Example test
import { test, expect } from "@playwright/test";

test.describe("Cross-browser compatibility", () => {
  test("renders correctly on all browsers", async ({ page }) => {
    await page.goto("/");

    // Check critical elements exist
    await expect(page.locator("header")).toBeVisible();
    await expect(page.locator("main")).toBeVisible();
    await expect(page.locator("footer")).toBeVisible();

    // Check layout doesn't break
    const header = await page.locator("header").boundingBox();
    expect(header?.width).toBeGreaterThan(0);

    // Take screenshot for visual comparison
    await expect(page).toHaveScreenshot("homepage.png", {
      maxDiffPixels: 100,
    });
  });

  test("form works across browsers", async ({ page }) => {
    await page.goto("/contact");

    await page.fill('input[name="email"]', "test@example.com");
    await page.fill('textarea[name="message"]', "Test message");
    await page.click('button[type="submit"]');

    await expect(page.locator(".success-message")).toBeVisible();
  });

  test("date picker works on Safari", async ({ page, browserName }) => {
    test.skip(browserName !== "webkit", "Safari-specific test");

    await page.goto("/booking");
    await page.click('input[type="date"]');

    // Safari has different date picker behavior
    await page.keyboard.type("12/25/2024");
    const value = await page.inputValue('input[type="date"]');
    expect(value).toBe("2024-12-25");
  });
});
```

---

## Common Browser Issues

### Issue 1: Flexbox Inconsistencies

```css
/* ❌ Problem: Different flex behavior in Safari */
.container {
  display: flex;
}

.item {
  flex: 1; /* Safari may not calculate correctly */
}

/* ✅ Solution: Be explicit with flex properties */
.container {
  display: flex;
}

.item {
  flex: 1 1 0%; /* flex-grow flex-shrink flex-basis */
  min-width: 0; /* Prevent overflow in Safari/Firefox */
}
```

### Issue 2: Date Input Differences

```typescript
// ❌ Problem: Date input behavior varies
function DatePicker() {
  return <input type="date" />; // Different UI in each browser!
}

// ✅ Solution: Use a cross-browser date picker library
import DatePicker from "react-datepicker";

function UniformDatePicker() {
  const [date, setDate] = useState<Date | null>(null);

  return (
    <DatePicker
      selected={date}
      onChange={(date) => setDate(date)}
      dateFormat="yyyy-MM-dd"
      // Consistent UI across all browsers
    />
  );
}
```

### Issue 3: Scroll Behavior

```css
/* ❌ Problem: Smooth scroll not supported everywhere */
html {
  scroll-behavior: smooth; /* Not in Safari < 15.4 */
}

/* ✅ Solution: JavaScript fallback */
```

```typescript
function smoothScrollTo(elementId: string): void {
  const element = document.getElementById(elementId);
  if (!element) return;

  // Check for native support
  if ("scrollBehavior" in document.documentElement.style) {
    element.scrollIntoView({ behavior: "smooth" });
  } else {
    // Polyfill for older browsers
    const start = window.pageYOffset;
    const target = element.getBoundingClientRect().top + start;
    const distance = target - start;
    const duration = 1000;
    let startTime: number | null = null;

    function animation(currentTime: number) {
      if (startTime === null) startTime = currentTime;
      const timeElapsed = currentTime - startTime;
      const progress = Math.min(timeElapsed / duration, 1);

      window.scrollTo(0, start + distance * easeInOutCubic(progress));

      if (timeElapsed < duration) {
        requestAnimationFrame(animation);
      }
    }

    function easeInOutCubic(t: number): number {
      return t < 0.5 ? 4 * t * t * t : (t - 1) * (2 * t - 2) * (2 * t - 2) + 1;
    }

    requestAnimationFrame(animation);
  }
}
```

### Issue 4: localStorage in Private Mode

```typescript
/**
 * Safari throws in private mode
 * Other browsers return null or throw
 */
class SafeStorage {
  private static isAvailable(): boolean {
    try {
      const test = "__storage_test__";
      localStorage.setItem(test, test);
      localStorage.removeItem(test);
      return true;
    } catch {
      return false;
    }
  }

  private static memoryStorage: Map<string, string> = new Map();

  static getItem(key: string): string | null {
    if (this.isAvailable()) {
      return localStorage.getItem(key);
    }
    return this.memoryStorage.get(key) || null;
  }

  static setItem(key: string, value: string): void {
    if (this.isAvailable()) {
      localStorage.setItem(key, value);
    } else {
      this.memoryStorage.set(key, value);
    }
  }

  static removeItem(key: string): void {
    if (this.isAvailable()) {
      localStorage.removeItem(key);
    } else {
      this.memoryStorage.delete(key);
    }
  }
}
```

---

## Progressive Enhancement

### Strategy

```typescript
/**
 * Progressive Enhancement Layers
 * 1. Core HTML content (works everywhere)
 * 2. Enhanced with CSS (better presentation)
 * 3. Enhanced with JavaScript (interactive features)
 */

// Layer 1: Basic HTML (functional without CSS/JS)
const BasicHTML = `
<article class="card">
  <h2>Product Title</h2>
  <img src="product.jpg" alt="Product">
  <p>Description</p>
  <a href="/product/123">View Details</a>
</article>
`;

// Layer 2: Enhanced with CSS
const EnhancedCSS = `
.card {
  /* Fallback styles first */
  border: 1px solid #ccc;
  padding: 1rem;

  /* Progressive enhancement */
  @supports (display: grid) {
    display: grid;
    gap: 1rem;
  }

  @supports (backdrop-filter: blur(10px)) {
    backdrop-filter: blur(10px);
    background: rgba(255, 255, 255, 0.8);
  }
}
`;

// Layer 3: Enhanced with JavaScript
class ProgressiveCard {
  constructor(element: HTMLElement) {
    // Basic functionality works without JS
    this.enhanceWithJS(element);
  }

  private enhanceWithJS(element: HTMLElement): void {
    // Only add advanced features if supported
    if (FeatureDetector.hasAPI("IntersectionObserver")) {
      this.lazyLoadImage(element);
    }

    if (FeatureDetector.hasAPI("fetch")) {
      this.enableQuickView(element);
    }
  }

  private lazyLoadImage(element: HTMLElement): void {
    const img = element.querySelector("img");
    if (!img) return;

    const observer = new IntersectionObserver((entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          const img = entry.target as HTMLImageElement;
          img.src = img.dataset.src || "";
          observer.unobserve(img);
        }
      });
    });

    observer.observe(img);
  }

  private enableQuickView(element: HTMLElement): void {
    const link = element.querySelector("a");
    if (!link) return;

    link.addEventListener("click", async (e) => {
      e.preventDefault();
      // Fetch and show details without navigation
      // Falls back to normal link if fetch fails
    });
  }
}
```

---

## Graceful Degradation

```typescript
/**
 * Graceful degradation - Start with full experience
 * Remove features for browsers that can't handle them
 */
class GracefulFeatures {
  static initializeApp(): void {
    // Start with advanced features
    this.setupModernFeatures();

    // Remove features that aren't supported
    if (!FeatureDetector.hasAPI("serviceWorker")) {
      this.disableOfflineMode();
    }

    if (!FeatureDetector.hasAPI("webGL")) {
      this.use2DGraphics();
    }

    if (!FeatureDetector.hasBackdropFilter) {
      this.useSolidBackgrounds();
    }
  }

  private static setupModernFeatures(): void {
    // Attempt to use all modern features
    this.enableServiceWorker();
    this.enable3DGraphics();
    this.enableGlassmorphism();
  }

  private static disableOfflineMode(): void {
    console.warn("Offline mode not available");
    // Show online-only UI
  }

  private static use2DGraphics(): void {
    // Fallback to Canvas 2D instead of WebGL
  }

  private static useSolidBackgrounds(): void {
    document.documentElement.classList.add("no-backdrop-filter");
  }

  // Implement other methods...
  private static enableServiceWorker(): void {}
  private static enable3DGraphics(): void {}
  private static enableGlassmorphism(): void {}
}
```

---

## Browser-Specific Workarounds

### Safari-Specific Fixes

```typescript
/**
 * Safari has many unique behaviors
 */
class SafariWorkarounds {
  static isSafari(): boolean {
    return (
      /^((?!chrome|android).)*safari/i.test(navigator.userAgent) &&
      !navigator.userAgent.includes("Chrome")
    );
  }

  static applyFixes(): void {
    if (!this.isSafari()) return;

    this.fixDateInputs();
    this.fixVideoAutoplay();
    this.fixFlexboxBugs();
    this.fixScrolling();
  }

  // Safari doesn't show date picker consistently
  private static fixDateInputs(): void {
    const dateInputs = document.querySelectorAll('input[type="date"]');
    dateInputs.forEach((input) => {
      // Add custom date picker or format hint
      const hint = document.createElement("span");
      hint.textContent = "(YYYY-MM-DD)";
      hint.className = "date-hint";
      input.parentElement?.appendChild(hint);
    });
  }

  // Safari requires user interaction for autoplay
  private static fixVideoAutoplay(): void {
    const videos = document.querySelectorAll("video[autoplay]");
    videos.forEach((video) => {
      (video as HTMLVideoElement).muted = true; // Must be muted
      (video as HTMLVideoElement).playsInline = true; // Required for iOS
    });
  }

  // Safari flexbox requires explicit flex-basis
  private static fixFlexboxBugs(): void {
    document.documentElement.classList.add("safari");
  }

  // Fix momentum scrolling on iOS
  private static fixScrolling(): void {
    const scrollContainers = document.querySelectorAll(".scroll-container");
    scrollContainers.forEach((container) => {
      (container as HTMLElement).style.webkitOverflowScrolling = "touch";
    });
  }
}

// Apply on load
if (document.readyState === "loading") {
  document.addEventListener("DOMContentLoaded", () =>
    SafariWorkarounds.applyFixes(),
  );
} else {
  SafariWorkarounds.applyFixes();
}
```

### Internet Explorer Fixes (Legacy)

```typescript
/**
 * IE11 and below fixes (if still supporting)
 * Consider dropping support for IE
 */
class IEWorkarounds {
  static isIE(): boolean {
    return (
      /MSIE|Trident/.test(navigator.userAgent) ||
      !!(document as any).documentMode
    );
  }

  static applyFixes(): void {
    if (!this.isIE()) return;

    this.addCustomEventPolyfill();
    this.addObjectFitPolyfill();
    this.fixFlexbox();
  }

  private static addCustomEventPolyfill(): void {
    if (typeof (window as any).CustomEvent !== "function") {
      function CustomEvent(
        event: string,
        params: CustomEventInit = { bubbles: false, cancelable: false },
      ) {
        const evt = document.createEvent("CustomEvent");
        evt.initCustomEvent(
          event,
          params.bubbles ?? false,
          params.cancelable ?? false,
          params.detail,
        );
        return evt;
      }
      (window as any).CustomEvent = CustomEvent;
    }
  }

  private static addObjectFitPolyfill(): void {
    // Use object-fit-images library or implement polyfill
    const images = document.querySelectorAll("img[style*='object-fit']");
    images.forEach((img) => {
      // Implement polyfill logic
    });
  }

  private static fixFlexbox(): void {
    // IE11 flexbox bugs require explicit sizing
    document.documentElement.classList.add("ie");
  }
}
```

---

## Best Practices

### 1. Feature Detection Over Browser Detection

```typescript
// ❌ DON'T: Browser sniffing
if (navigator.userAgent.includes("Safari")) {
  // Fragile - user agents can be spoofed
}

// ✅ DO: Feature detection
if ("IntersectionObserver" in window) {
  // Reliable - tests actual capability
}
```

### 2. Use Autoprefixer for CSS

```javascript
// postcss.config.js
module.exports = {
  plugins: {
    autoprefixer: {
      overrideBrowserslist: ["> 1%", "last 2 versions", "not dead"],
    },
  },
};
```

### 3. Progressive Enhancement Strategy

```typescript
// ✅ Build from the ground up
// 1. Works without JavaScript
// 2. Enhanced with CSS
// 3. Enhanced with JavaScript

// Example: Form submission
class Form {
  constructor(form: HTMLFormElement) {
    // Works even if JS fails to load
    form.action = "/api/submit";
    form.method = "POST";

    // Enhance with JS if available
    this.enhanceWithAjax(form);
  }

  private enhanceWithAjax(form: HTMLFormElement): void {
    if (!("fetch" in window)) return;

    form.addEventListener("submit", async (e) => {
      e.preventDefault();
      // AJAX submission
      // Falls back to normal form submission if fetch fails
    });
  }
}
```

### 4. Test Early and Often

```typescript
// Continuous cross-browser testing
const testingSchedule = {
  "During Development": [
    "Test in 2-3 browsers constantly",
    "Use browser DevTools device emulation",
  ],
  "Before Commit": [
    "Run automated tests across all browsers",
    "Visual regression testing",
  ],
  "Before Release": [
    "Manual testing on real devices",
    "Performance testing across browsers",
    "Accessibility audit in all browsers",
  ],
};
```

### 5. Document Browser Support

```typescript
// README.md or docs
interface BrowserSupport {
  browser: string;
  minVersion: string;
  notes?: string;
}

const supportedBrowsers: BrowserSupport[] = [
  { browser: "Chrome", minVersion: "90+", notes: "Full support" },
  { browser: "Firefox", minVersion: "88+", notes: "Full support" },
  {
    browser: "Safari",
    minVersion: "14+",
    notes: "Some features degraded",
  },
  { browser: "Edge", minVersion: "90+", notes: "Full support" },
  {
    browser: "IE",
    minVersion: "Not supported",
    notes: "Please use a modern browser",
  },
];
```

---

## Testing Tools

### Recommended Tools

```typescript
interface CrossBrowserTool {
  name: string;
  type: "automation" | "manual" | "emulation" | "cloud";
  features: string[];
  url: string;
}

const testingTools: CrossBrowserTool[] = [
  {
    name: "Playwright",
    type: "automation",
    features: [
      "Chrome, Firefox, Safari testing",
      "Mobile emulation",
      "Screenshots & videos",
      "Network mocking",
    ],
    url: "https://playwright.dev",
  },
  {
    name: "BrowserStack",
    type: "cloud",
    features: [
      "Real device testing",
      "2000+ browser/OS combinations",
      "Live testing",
      "Automated testing",
    ],
    url: "https://browserstack.com",
  },
  {
    name: "LambdaTest",
    type: "cloud",
    features: [
      "Cross-browser testing",
      "Responsive testing",
      "Screenshot testing",
      "Geolocation testing",
    ],
    url: "https://lambdatest.com",
  },
  {
    name: "Can I Use",
    type: "manual",
    features: [
      "Feature support tables",
      "Browser compatibility data",
      "Usage statistics",
    ],
    url: "https://caniuse.com",
  },
  {
    name: "Browserling",
    type: "cloud",
    features: [
      "Interactive browser testing",
      "Screenshot tool",
      "Responsive testing",
    ],
    url: "https://browserling.com",
  },
];
```

---

## Interview Questions

### Conceptual Questions

1. **What is cross-browser compatibility and why is it important?**
   - Ensures consistent experience across different browsers
   - Different rendering engines interpret code differently
   - Impacts user experience, accessibility, and SEO

2. **Explain the difference between progressive enhancement and graceful degradation?**
   - Progressive enhancement: Start basic, add features
   - Graceful degradation: Start full-featured, remove features
   - Progressive enhancement is generally preferred

3. **Why should you use feature detection instead of browser detection?**
   - User agents can be spoofed
   - More reliable - tests actual capability
   - Easier to maintain
   - Doesn't break with new browser versions

4. **What are vendor prefixes and when should you use them?**
   - Browser-specific CSS properties (-webkit-, -moz-, -ms-)
   - Needed for experimental features
   - Use Autoprefixer to add automatically
   - Write standard syntax, let tools add prefixes

### Technical Questions

5. **How do you detect if a CSS feature is supported?**

```typescript
if (CSS.supports("display", "grid")) {
  // Use grid
} else {
  // Use flexbox fallback
}
```

6. **What are polyfills? Give examples.**
   - Code that provides modern functionality in older browsers
   - Examples: fetch, Promise, IntersectionObserver
   - Use conditionally to reduce bundle size

7. **How would you handle localStorage in Safari private mode?**
   - Safari throws QuotaExceededError
   - Try/catch and fallback to memory storage
   - Check availability before use

8. **What are common CSS differences between browsers?**
   - Flexbox implementation variations
   - Default form styling
   - Scrollbar appearance
   - Date input rendering
   - Vendor-prefixed properties

9. **How do you test across multiple browsers efficiently?**
   - Automated testing (Playwright, Selenium)
   - Cloud testing services (BrowserStack)
   - Local VMs for specific browsers
   - Continuous integration with browser matrix

10. **What is Babel and how does it help with cross-browser support?**
    - Transpiles modern JavaScript to ES5
    - Configured with browserslist
    - Add polyfills automatically with useBuiltIns
    - Reduces need for manual compatibility code

### Scenario-Based Questions

11. **A feature works in Chrome but not Safari. How do you debug?**
    - Check browser console for errors
    - Use Safari Web Inspector
    - Check Can I Use for feature support
    - Test with feature detection
    - Look for Safari-specific bugs

12. **How would you implement smooth scrolling cross-browser?**
    - Use CSS scroll-behavior with JS fallback
    - Detect support before implementing
    - Polyfill with requestAnimationFrame
    - Easing functions for smooth animation

13. **A client requires IE11 support. How do you approach this?**
    - Evaluate if truly necessary (cost vs benefit)
    - Use Babel with IE11 preset
    - Add extensive polyfills
    - Test thoroughly - IE11 has many quirks
    - Consider separate stylesheet

14. **How do you handle responsive images across browsers?**
    - Use srcset and sizes attributes
    - Provide fallback src
    - Use picture element for art direction
    - Test on actual devices
    - Consider WebP with fallbacks

---

## Summary

### Key Takeaways

1. **Feature Detection**: Always detect features, never sniff browsers
2. **Polyfills**: Load conditionally based on need
3. **Testing**: Automate cross-browser tests in CI/CD
4. **Progressive Enhancement**: Build from basic to advanced
5. **Vendor Prefixes**: Use Autoprefixer automatically
6. **Documentation**: Clearly specify supported browsers
7. **User Experience**: Prioritize consistency over pixel-perfection
8. **Performance**: Consider impact of polyfills on bundle size

### Compatibility Checklist

- [ ] Define browser support policy
- [ ] Set up Babel with browserslist
- [ ] Configure Autoprefixer
- [ ] Implement feature detection
- [ ] Add necessary polyfills
- [ ] Write cross-browser CSS
- [ ] Test on real devices
- [ ] Set up automated testing
- [ ] Monitor browser usage analytics
- [ ] Document known issues

### Resources

- [Can I Use](https://caniuse.com) - Feature support tables
- [MDN Browser Compatibility](https://developer.mozilla.org/en-US/docs/MDN/Writing_guidelines/Page_structures/Compatibility_tables)
- [Browserslist](https://browsersl.ist) - Target browser configuration
- [Autoprefixer](https://autoprefixer.github.io) - CSS vendor prefixing
- [Core-js](https://github.com/zloirock/core-js) - JavaScript polyfills

---

**Next Steps**: Practice implementing feature detection, write cross-browser tests, and experiment with different polyfill strategies.
