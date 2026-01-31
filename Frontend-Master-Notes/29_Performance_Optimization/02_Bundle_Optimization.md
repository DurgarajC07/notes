# Bundle Optimization

> **Master bundle size reduction techniques for faster load times and better user experience**

---

## Core Concept

Bundle optimization reduces the amount of JavaScript, CSS, and assets shipped to users, directly impacting load time, parse time, and Time to Interactive (TTI). Every kilobyte saved improves performance, especially on slow networks and low-end devices.

**Key Metrics:**

- **Bundle Size**: Total JavaScript/CSS shipped
- **Parse Time**: Time to compile JavaScript
- **Load Time**: Network transfer duration
- **Cache Hit Rate**: Percentage of assets served from cache

---

## Tree Shaking

### **Webpack Tree Shaking**

```javascript
// webpack.config.js
module.exports = {
  mode: 'production', // Enables tree shaking
  optimization: {
    usedExports: true, // Mark unused exports
    sideEffects: false, // Assume no side effects (unless specified in package.json)
  },
};

// package.json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js"
  ]
}
```

### **Writing Tree-Shakeable Code**

```typescript
// ❌ BAD - Default export prevents tree shaking
export default {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
  multiply: (a, b) => a * b,
  divide: (a, b) => a / b,
};

// ✅ GOOD - Named exports enable tree shaking
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
export const multiply = (a, b) => a * b;
export const divide = (a, b) => a / b;

// Consumer only imports what's needed
import { add, multiply } from "./math"; // subtract and divide removed
```

### **Lodash Tree Shaking**

```typescript
// ❌ BAD - Imports entire library (70KB+)
import _ from "lodash";
const result = _.debounce(fn, 300);

// ❌ Still bad - imports all of lodash
import { debounce } from "lodash";

// ✅ GOOD - Import specific module
import debounce from "lodash/debounce";

// ✅ BETTER - Use lodash-es for ESM tree shaking
import { debounce } from "lodash-es";
```

---

## Code Splitting

### **Route-Based Code Splitting**

```typescript
// React Router lazy loading
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

### **Component-Level Code Splitting**

```typescript
// Split heavy components
const HeavyChart = lazy(() => import('./components/HeavyChart'));
const VideoPlayer = lazy(() => import('./components/VideoPlayer'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <h1>Dashboard</h1>
      <button onClick={() => setShowChart(true)}>Show Chart</button>

      {showChart && (
        <Suspense fallback={<Skeleton height={400} />}>
          <HeavyChart data={chartData} />
        </Suspense>
      )}
    </div>
  );
}
```

### **Named Chunks with Webpack Magic Comments**

```typescript
// Control chunk names and loading behavior
const AdminPanel = lazy(
  () =>
    import(
      /* webpackChunkName: "admin-panel" */
      /* webpackPrefetch: true */
      "./pages/AdminPanel"
    ),
);

const Reports = lazy(
  () =>
    import(
      /* webpackChunkName: "reports" */
      /* webpackPreload: true */
      "./pages/Reports"
    ),
);

// Prefetch: Load during idle time (low priority)
// Preload: Load in parallel with parent chunk (high priority)
```

---

## Dynamic Imports

### **Conditional Loading**

```typescript
// Load library only when needed
async function exportToExcel(data: any[]) {
  const ExcelJS = await import("exceljs");
  const workbook = new ExcelJS.Workbook();
  // ... export logic
}

// Load polyfills conditionally
if (!("IntersectionObserver" in window)) {
  await import("intersection-observer");
}
```

### **Feature-Based Splitting**

```typescript
// Split by user role
async function loadUserFeatures(userRole: string) {
  if (userRole === "admin") {
    const { AdminFeatures } = await import("./features/admin");
    return AdminFeatures;
  } else if (userRole === "editor") {
    const { EditorFeatures } = await import("./features/editor");
    return EditorFeatures;
  } else {
    const { ViewerFeatures } = await import("./features/viewer");
    return ViewerFeatures;
  }
}
```

---

## Dependency Analysis

### **Bundle Analyzer**

```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static', // Generates HTML report
      openAnalyzer: false,
      reportFilename: 'bundle-report.html',
    }),
  ],
};

// package.json scripts
{
  "scripts": {
    "analyze": "webpack --config webpack.config.js --profile --json > stats.json && webpack-bundle-analyzer stats.json"
  }
}
```

### **Source Map Explorer**

```bash
npm install -g source-map-explorer

# Analyze bundle
source-map-explorer bundle.js bundle.js.map

# Multiple bundles
source-map-explorer 'dist/*.js'
```

### **Finding Large Dependencies**

```typescript
// Use import cost VS Code extension to see sizes inline

// Check package.json dependencies
import { getPackageSize } from "package-size";

const sizes = await getPackageSize(["react", "lodash", "moment"]);
// { react: '6.4 KB', lodash: '72 KB', moment: '68 KB' }
```

---

## Minification & Compression

### **Terser Configuration**

```javascript
// webpack.config.js
const TerserPlugin = require("terser-webpack-plugin");

module.exports = {
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true, // Remove console.log in production
            drop_debugger: true,
            pure_funcs: ["console.info", "console.debug"], // Remove specific calls
          },
          mangle: {
            safari10: true, // Fix Safari 10 bug
          },
          format: {
            comments: false, // Remove comments
          },
        },
        extractComments: false,
      }),
    ],
  },
};
```

### **CSS Minification**

```javascript
const CssMinimizerPlugin = require("css-minimizer-webpack-plugin");

module.exports = {
  optimization: {
    minimizer: [
      new CssMinimizerPlugin({
        minimizerOptions: {
          preset: [
            "default",
            {
              discardComments: { removeAll: true },
              normalizeWhitespace: true,
            },
          ],
        },
      }),
    ],
  },
};
```

### **Compression (Gzip & Brotli)**

```javascript
const CompressionPlugin = require("compression-webpack-plugin");

module.exports = {
  plugins: [
    // Gzip compression
    new CompressionPlugin({
      filename: "[path][base].gz",
      algorithm: "gzip",
      test: /\.(js|css|html|svg)$/,
      threshold: 10240, // Only assets > 10KB
      minRatio: 0.8,
    }),

    // Brotli compression (better than gzip)
    new CompressionPlugin({
      filename: "[path][base].br",
      algorithm: "brotliCompress",
      test: /\.(js|css|html|svg)$/,
      compressionOptions: {
        level: 11, // Max compression
      },
      threshold: 10240,
      minRatio: 0.8,
    }),
  ],
};
```

---

## Module Federation

### **Sharing Dependencies Across Micro-Frontends**

```javascript
// host/webpack.config.js
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "host",
      remotes: {
        app1: "app1@http://localhost:3001/remoteEntry.js",
        app2: "app2@http://localhost:3002/remoteEntry.js",
      },
      shared: {
        react: { singleton: true, requiredVersion: "^18.0.0" },
        "react-dom": { singleton: true, requiredVersion: "^18.0.0" },
        "react-router-dom": { singleton: true },
      },
    }),
  ],
};

// remote/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "app1",
      filename: "remoteEntry.js",
      exposes: {
        "./UserProfile": "./src/components/UserProfile",
        "./Dashboard": "./src/components/Dashboard",
      },
      shared: {
        react: { singleton: true },
        "react-dom": { singleton: true },
      },
    }),
  ],
};
```

---

## Dead Code Elimination

### **Detecting Unused Code**

```typescript
// Use ESLint to find unused variables
// .eslintrc.js
module.exports = {
  rules: {
    "no-unused-vars": "error",
    "@typescript-eslint/no-unused-vars": "error",
  },
};

// Use ts-prune to find unused exports
// npm install -g ts-prune
// ts-prune | grep -v test
```

### **Removing Dead Code**

```typescript
// ❌ BAD - Dead code that can't be tree-shaken
export function getData() {
  const legacyTransform = (data) => {
    /* never used */
  };
  const unusedHelper = () => {
    /* never called */
  };

  return fetch("/api/data").then((r) => r.json());
}

// ✅ GOOD - Clean, minimal code
export function getData() {
  return fetch("/api/data").then((r) => r.json());
}
```

---

## Optimizing Third-Party Libraries

### **Replace Heavy Libraries**

```typescript
// ❌ moment.js (68KB minified + gzipped)
import moment from "moment";
const formatted = moment().format("YYYY-MM-DD");

// ✅ date-fns (2KB per function, tree-shakeable)
import { format } from "date-fns";
const formatted = format(new Date(), "yyyy-MM-dd");

// ❌ lodash (24KB minified + gzipped)
import _ from "lodash";

// ✅ Native alternatives
const unique = [...new Set(array)];
const sorted = [...array].sort();
const grouped = Object.groupBy(array, (item) => item.category);
```

### **Lazy Loading Third-Party Scripts**

```typescript
// Load analytics only after interaction
let analyticsLoaded = false;

function loadAnalytics() {
  if (analyticsLoaded) return;

  const script = document.createElement("script");
  script.src = "https://www.google-analytics.com/analytics.js";
  script.async = true;
  document.head.appendChild(script);

  analyticsLoaded = true;
}

// Load on first interaction
document.addEventListener("click", loadAnalytics, { once: true });
```

---

## Production Build Optimization

### **Environment Variables**

```javascript
// webpack.config.js
const webpack = require("webpack");

module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      "process.env.NODE_ENV": JSON.stringify("production"),
      __DEV__: JSON.stringify(false),
    }),
  ],
};

// Code that gets eliminated
if (__DEV__) {
  // This entire block removed in production
  console.log("Debug info");
  enableDevTools();
}
```

### **Scope Hoisting**

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    concatenateModules: true, // Enable scope hoisting
  },
};

// Before scope hoisting (multiple module wrappers):
// (function(module, exports) { /* module A */ })
// (function(module, exports) { /* module B */ })

// After scope hoisting (single wrapper):
// (function() { /* module A + B inlined */ })
```

---

## Webpack Performance Tips

### **Build Performance**

```javascript
module.exports = {
  cache: {
    type: "filesystem", // Cache to disk
    cacheDirectory: path.resolve(__dirname, ".webpack_cache"),
  },

  optimization: {
    moduleIds: "deterministic", // Stable IDs for caching
    runtimeChunk: "single", // Extract runtime to separate chunk
    splitChunks: {
      chunks: "all",
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendors",
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },
  },

  performance: {
    hints: "warning",
    maxEntrypointSize: 250000, // 250KB
    maxAssetSize: 250000,
  },
};
```

---

## Real-World Example: E-Commerce App

```typescript
// App structure optimized for bundle size
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// Eager load: Critical for first paint
import Header from './components/Header'; // 5KB
import Footer from './components/Footer'; // 3KB

// Lazy load: Non-critical routes
const Home = lazy(() => import(/* webpackChunkName: "home" */ './pages/Home')); // 45KB
const ProductList = lazy(() => import(/* webpackChunkName: "products" */ './pages/ProductList')); // 60KB
const ProductDetail = lazy(() => import(/* webpackChunkName: "product-detail" */ './pages/ProductDetail')); // 35KB
const Cart = lazy(() => import(/* webpackChunkName: "cart" */ './pages/Cart')); // 25KB
const Checkout = lazy(() => import(/* webpackChunkName: "checkout" */ './pages/Checkout')); // 70KB
const AdminPanel = lazy(() => import(/* webpackChunkName: "admin" */ './pages/Admin')); // 120KB

function App() {
  return (
    <BrowserRouter>
      <Header />
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/products" element={<ProductList />} />
          <Route path="/product/:id" element={<ProductDetail />} />
          <Route path="/cart" element={<Cart />} />
          <Route path="/checkout" element={<Checkout />} />
          <Route path="/admin/*" element={<AdminPanel />} />
        </Routes>
      </Suspense>
      <Footer />
    </BrowserRouter>
  );
}

// Initial bundle: ~100KB (header, footer, routing)
// Each route loads on demand: 25-120KB
// Without code splitting: 480KB initial bundle!
```

---

## Best Practices

1. **Analyze before optimizing** - Use bundle analyzer to identify large dependencies
2. **Code split by route** - Load only what's needed for current page
3. **Use tree-shakeable libraries** - Prefer ESM over CommonJS
4. **Lazy load non-critical code** - Defer heavy components until needed
5. **Replace heavy dependencies** - date-fns instead of moment, lodash-es instead of lodash
6. **Enable compression** - Serve Brotli (fallback to gzip)
7. **Remove dead code** - Use ESLint and ts-prune
8. **Optimize images** - Use WebP, lazy load below fold
9. **Monitor bundle size** - Set budgets in CI/CD
10. **Cache aggressively** - Use content hashing for long-term caching

---

## Key Takeaways

1. **Tree shaking removes unused code** - but only works with ESM and side-effect-free code
2. **Code splitting reduces initial load** - lazy load routes and heavy components
3. **Bundle analyzer reveals optimization opportunities** - always measure before optimizing
4. **Third-party libraries often bloat bundles** - replace with lighter alternatives or native APIs
5. **Compression is free performance** - Brotli saves 20-30% over gzip
6. **Dynamic imports enable conditional loading** - load features based on user role or interaction
7. **Scope hoisting reduces wrapper overhead** - inline modules into single scope
8. **Dead code elimination requires aggressive minification** - use Terser with proper settings
9. **Module federation shares dependencies** - avoids duplication in micro-frontends
10. **Every KB matters on slow networks** - target <200KB initial bundle for good performance

---

**Remember:** Bundle optimization is ongoing work, not a one-time task. Continuously monitor, analyze, and optimize as you add features.
