# Performance Checklist

## Table of Contents

- [Introduction](#introduction)
- [Initial Load Performance](#initial-load-performance)
- [Runtime Performance](#runtime-performance)
- [Network Optimization](#network-optimization)
- [Rendering Performance](#rendering-performance)
- [Memory Management](#memory-management)
- [Asset Optimization](#asset-optimization)
- [Caching Strategies](#caching-strategies)
- [Monitoring and Metrics](#monitoring-and-metrics)
- [Mobile Performance](#mobile-performance)
- [Advanced Optimizations](#advanced-optimizations)
- [Performance Testing](#performance-testing)
- [Key Takeaways](#key-takeaways)

## Introduction

Performance directly impacts user experience, conversion rates, and SEO rankings. This checklist ensures your application is optimized before launch.

### Performance Budget

```typescript
interface PerformanceBudget {
  metrics: {
    FCP: "< 1.8s"; // First Contentful Paint
    LCP: "< 2.5s"; // Largest Contentful Paint
    FID: "< 100ms"; // First Input Delay
    CLS: "< 0.1"; // Cumulative Layout Shift
    TTI: "< 3.8s"; // Time to Interactive
    TBT: "< 300ms"; // Total Blocking Time
  };
  resources: {
    javascript: "< 200KB gzipped";
    css: "< 50KB gzipped";
    images: "< 1MB total";
    fonts: "< 100KB total";
    totalPageSize: "< 1.5MB";
  };
  timing: {
    serverResponse: "< 600ms";
    domContentLoaded: "< 2s";
    fullyLoaded: "< 5s";
  };
}

const performanceThresholds = {
  good: { LCP: 2.5, FID: 100, CLS: 0.1 },
  needsImprovement: { LCP: 4.0, FID: 300, CLS: 0.25 },
  poor: { LCP: Infinity, FID: Infinity, CLS: Infinity },
};
```

## Initial Load Performance

### ✅ Code Splitting

```typescript
// Route-based code splitting
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// Lazy load route components
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}

// Component-based code splitting
const HeavyChart = lazy(() =>
  import('./components/HeavyChart').then(module => ({
    default: module.HeavyChart
  }))
);

// Conditional loading
function DataVisualization({ type }: { type: 'chart' | 'table' }) {
  if (type === 'chart') {
    return (
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart />
      </Suspense>
    );
  }
  return <DataTable />;
}
```

### ✅ Bundle Optimization

```typescript
// vite.config.ts - Advanced bundle optimization
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true,
      filename: "dist/stats.html",
    }),
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          // Vendor chunks
          if (id.includes("node_modules")) {
            // React ecosystem
            if (id.includes("react") || id.includes("react-dom")) {
              return "react-vendor";
            }
            // UI libraries
            if (id.includes("@radix-ui") || id.includes("@headlessui")) {
              return "ui-vendor";
            }
            // Data fetching
            if (id.includes("@tanstack")) {
              return "query-vendor";
            }
            // Other vendors
            return "vendor";
          }
        },
        // Optimize chunk names
        chunkFileNames: "assets/[name]-[hash].js",
        entryFileNames: "assets/[name]-[hash].js",
        assetFileNames: "assets/[name]-[hash].[ext]",
      },
    },
    // Minification options
    minify: "terser",
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
        pure_funcs: ["console.log", "console.info"],
      },
    },
    // Source maps for production debugging
    sourcemap: "hidden",
    // Chunk size warnings
    chunkSizeWarningLimit: 500,
  },
  // Optimize dependencies
  optimizeDeps: {
    include: ["react", "react-dom", "react-router-dom"],
    exclude: ["@tanstack/react-query-devtools"],
  },
});
```

### ✅ Critical CSS

```typescript
// Extract critical CSS for above-the-fold content
import { createApp } from './app';
import { renderToString } from 'react-dom/server';

async function generateCriticalCSS(component: React.ReactElement) {
  const html = renderToString(component);

  // Use critical CSS extraction tool
  const { css } = await critical.generate({
    html,
    width: 1300,
    height: 900,
    inline: true
  });

  return css;
}

// Inline critical CSS in index.html
const criticalCSS = await generateCriticalCSS(<App />);

// HTML template
const html = `
<!DOCTYPE html>
<html>
  <head>
    <style>${criticalCSS}</style>
    <link rel="preload" href="/assets/main.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
    <noscript><link rel="stylesheet" href="/assets/main.css"></noscript>
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
`;
```

### ✅ Resource Hints

```html
<!-- index.html - Resource hints for better loading -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <!-- DNS Prefetch for external domains -->
    <link rel="dns-prefetch" href="https://api.example.com" />
    <link rel="dns-prefetch" href="https://cdn.example.com" />

    <!-- Preconnect for critical origins -->
    <link rel="preconnect" href="https://api.example.com" crossorigin />
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

    <!-- Preload critical resources -->
    <link
      rel="preload"
      href="/fonts/inter-var.woff2"
      as="font"
      type="font/woff2"
      crossorigin
    />
    <link rel="preload" href="/assets/hero-image.webp" as="image" />

    <!-- Prefetch for next page -->
    <link rel="prefetch" href="/dashboard" as="document" />

    <!-- Preload modules -->
    <link rel="modulepreload" href="/src/main.tsx" />

    <title>Your App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

## Runtime Performance

### ✅ React Performance Optimization

```typescript
// 1. Memoization
import { memo, useMemo, useCallback } from 'react';

interface ExpensiveComponentProps {
  data: ComplexData[];
  onItemClick: (id: string) => void;
}

const ExpensiveComponent = memo(({ data, onItemClick }: ExpensiveComponentProps) => {
  // Memoize expensive computations
  const sortedData = useMemo(() => {
    return [...data].sort((a, b) => a.priority - b.priority);
  }, [data]);

  // Memoize callbacks
  const handleClick = useCallback((id: string) => {
    onItemClick(id);
  }, [onItemClick]);

  return (
    <div>
      {sortedData.map(item => (
        <ExpensiveItem
          key={item.id}
          item={item}
          onClick={handleClick}
        />
      ))}
    </div>
  );
});

// 2. Virtual scrolling for large lists
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }: { items: string[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative'
        }}
      >
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`
            }}
          >
            {items[virtualRow.index]}
          </div>
        ))}
      </div>
    </div>
  );
}

// 3. Debouncing user input
import { useDebouncedValue } from './hooks/useDebouncedValue';

function SearchComponent() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebouncedValue(query, 500);

  useEffect(() => {
    if (debouncedQuery) {
      performSearch(debouncedQuery);
    }
  }, [debouncedQuery]);

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}

// 4. Windowing expensive operations
function useThrottle<T>(value: T, limit: number): T {
  const [throttledValue, setThrottledValue] = useState(value);
  const lastRan = useRef(Date.now());

  useEffect(() => {
    const handler = setTimeout(() => {
      if (Date.now() - lastRan.current >= limit) {
        setThrottledValue(value);
        lastRan.current = Date.now();
      }
    }, limit - (Date.now() - lastRan.current));

    return () => clearTimeout(handler);
  }, [value, limit]);

  return throttledValue;
}
```

### ✅ Lazy Loading Components

```typescript
// Progressive image loading
import { useState, useEffect } from 'react';

interface ProgressiveImageProps {
  lowQualitySrc: string;
  highQualitySrc: string;
  alt: string;
}

function ProgressiveImage({ lowQualitySrc, highQualitySrc, alt }: ProgressiveImageProps) {
  const [src, setSrc] = useState(lowQualitySrc);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const img = new Image();
    img.src = highQualitySrc;
    img.onload = () => {
      setSrc(highQualitySrc);
      setIsLoading(false);
    };
  }, [highQualitySrc]);

  return (
    <img
      src={src}
      alt={alt}
      style={{
        filter: isLoading ? 'blur(10px)' : 'none',
        transition: 'filter 0.3s'
      }}
    />
  );
}

// Intersection Observer for lazy loading
function useLazyLoad() {
  const [ref, setRef] = useState<HTMLElement | null>(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    if (!ref) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect();
        }
      },
      { rootMargin: '50px' }
    );

    observer.observe(ref);

    return () => observer.disconnect();
  }, [ref]);

  return [setRef, isVisible] as const;
}

function LazyComponent() {
  const [ref, isVisible] = useLazyLoad();

  return (
    <div ref={ref}>
      {isVisible ? <HeavyComponent /> : <Placeholder />}
    </div>
  );
}
```

## Network Optimization

### ✅ API Request Optimization

```typescript
// Request batching
class RequestBatcher {
  private queue: Array<{ id: string; resolve: Function }> = [];
  private timeoutId: NodeJS.Timeout | null = null;

  constructor(private batchTime: number = 50) {}

  add(id: string): Promise<any> {
    return new Promise((resolve) => {
      this.queue.push({ id, resolve });

      if (!this.timeoutId) {
        this.timeoutId = setTimeout(() => this.flush(), this.batchTime);
      }
    });
  }

  private async flush() {
    const batch = this.queue.splice(0);
    this.timeoutId = null;

    const ids = batch.map((item) => item.id);
    const results = await this.fetchBatch(ids);

    batch.forEach((item, index) => {
      item.resolve(results[index]);
    });
  }

  private async fetchBatch(ids: string[]) {
    const response = await fetch("/api/batch", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ ids }),
    });
    return response.json();
  }
}

const batcher = new RequestBatcher();

// Usage
async function getUserData(id: string) {
  return batcher.add(id);
}

// Request deduplication
class RequestCache {
  private cache = new Map<string, Promise<any>>();

  async fetch(url: string, options?: RequestInit): Promise<any> {
    const key = `${url}:${JSON.stringify(options)}`;

    if (this.cache.has(key)) {
      return this.cache.get(key);
    }

    const promise = fetch(url, options)
      .then((res) => res.json())
      .finally(() => {
        // Remove from cache after completion
        setTimeout(() => this.cache.delete(key), 5000);
      });

    this.cache.set(key, promise);
    return promise;
  }
}

const requestCache = new RequestCache();

// Optimistic updates
import { useMutation, useQueryClient } from "@tanstack/react-query";

function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updateUser,
    onMutate: async (newUser) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ["user", newUser.id] });

      // Snapshot previous value
      const previousUser = queryClient.getQueryData(["user", newUser.id]);

      // Optimistically update
      queryClient.setQueryData(["user", newUser.id], newUser);

      return { previousUser };
    },
    onError: (err, newUser, context) => {
      // Rollback on error
      queryClient.setQueryData(["user", newUser.id], context?.previousUser);
    },
    onSettled: (newUser) => {
      // Refetch to ensure consistency
      queryClient.invalidateQueries({ queryKey: ["user", newUser?.id] });
    },
  });
}
```

### ✅ GraphQL Optimization

```typescript
// Query batching and caching
import { ApolloClient, InMemoryCache, HttpLink } from "@apollo/client";
import { BatchHttpLink } from "@apollo/client/link/batch-http";

const client = new ApolloClient({
  link: new BatchHttpLink({
    uri: "/graphql",
    batchMax: 5,
    batchInterval: 20,
  }),
  cache: new InMemoryCache({
    typePolicies: {
      Query: {
        fields: {
          user: {
            read(existing, { args, toReference }) {
              return (
                existing || toReference({ __typename: "User", id: args?.id })
              );
            },
          },
        },
      },
    },
  }),
  defaultOptions: {
    watchQuery: {
      fetchPolicy: "cache-and-network",
    },
  },
});

// Persisted queries
const link = createPersistedQueryLink({ sha256 }).concat(httpLink);
```

## Rendering Performance

### ✅ Avoid Layout Thrashing

```typescript
// Bad: Causes multiple reflows
function badExample() {
  const elements = document.querySelectorAll('.item');
  elements.forEach(el => {
    const height = el.offsetHeight; // Read
    el.style.height = `${height * 2}px`; // Write
  });
}

// Good: Batch reads and writes
function goodExample() {
  const elements = Array.from(document.querySelectorAll('.item'));

  // Batch all reads
  const heights = elements.map(el => el.offsetHeight);

  // Batch all writes
  requestAnimationFrame(() => {
    elements.forEach((el, i) => {
      el.style.height = `${heights[i] * 2}px`;
    });
  });
}

// React: Use refs to batch DOM operations
function Component() {
  const ref = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    if (!ref.current) return;

    const { width } = ref.current.getBoundingClientRect();
    ref.current.style.transform = `translateX(${width}px)`;
  }, []);

  return <div ref={ref}>Content</div>;
}
```

### ✅ CSS Performance

```css
/* Use contain for isolated components */
.card {
  contain: layout style paint;
}

/* Use will-change for animated properties */
.animated {
  will-change: transform;
  /* Remove after animation */
}

/* Avoid expensive properties */
.box {
  /* Expensive - causes reflow */
  /* width: calc(100% - 20px); */

  /* Better - use transforms */
  transform: translateX(-10px);
}

/* Use CSS containment */
.list-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 500px;
}

/* Optimize animations */
@keyframes slide {
  from {
    /* Avoid animating layout properties */
    /* left: 0; */
    /* margin-left: 0; */

    /* Use transform instead */
    transform: translateX(0);
  }
  to {
    transform: translateX(100px);
  }
}
```

### ✅ Web Workers for Heavy Computation

```typescript
// worker.ts
self.onmessage = (e: MessageEvent) => {
  const { data } = e;

  // Perform heavy computation
  const result = processLargeDataset(data);

  self.postMessage(result);
};

function processLargeDataset(data: any[]) {
  return data
    .map(item => expensiveTransform(item))
    .filter(item => complexFilter(item))
    .sort((a, b) => a.score - b.score);
}

// main.ts - Using the worker
function useWorker<T, R>(workerFn: () => Worker) {
  const [result, setResult] = useState<R | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const workerRef = useRef<Worker | null>(null);

  useEffect(() => {
    workerRef.current = workerFn();

    workerRef.current.onmessage = (e: MessageEvent<R>) => {
      setResult(e.data);
      setIsLoading(false);
    };

    return () => {
      workerRef.current?.terminate();
    };
  }, [workerFn]);

  const execute = useCallback((data: T) => {
    setIsLoading(true);
    workerRef.current?.postMessage(data);
  }, []);

  return { result, isLoading, execute };
}

// Usage
function DataProcessor() {
  const { result, isLoading, execute } = useWorker<DataItem[], ProcessedData>(
    () => new Worker(new URL('./worker.ts', import.meta.url))
  );

  const handleProcess = () => {
    execute(largeDataset);
  };

  return (
    <div>
      <button onClick={handleProcess}>Process Data</button>
      {isLoading && <Spinner />}
      {result && <Results data={result} />}
    </div>
  );
}
```

## Memory Management

### ✅ Memory Leak Prevention

```typescript
// 1. Cleanup event listeners
useEffect(() => {
  const handler = (e: Event) => console.log(e);
  window.addEventListener("resize", handler);

  return () => {
    window.removeEventListener("resize", handler);
  };
}, []);

// 2. Cancel pending requests
useEffect(() => {
  const abortController = new AbortController();

  fetch("/api/data", { signal: abortController.signal })
    .then((res) => res.json())
    .then(setData)
    .catch((err) => {
      if (err.name !== "AbortError") {
        console.error(err);
      }
    });

  return () => {
    abortController.abort();
  };
}, []);

// 3. Clear timers
useEffect(() => {
  const intervalId = setInterval(() => {
    console.log("tick");
  }, 1000);

  return () => {
    clearInterval(intervalId);
  };
}, []);

// 4. Unsubscribe from observables
useEffect(() => {
  const subscription = observable$.subscribe((value) => {
    setValue(value);
  });

  return () => {
    subscription.unsubscribe();
  };
}, []);

// 5. Clear caches
const cache = new Map();

function ComponentWithCache() {
  useEffect(() => {
    return () => {
      cache.clear();
    };
  }, []);
}
```

### ✅ Memory Profiling

```typescript
// Monitor memory usage
class MemoryMonitor {
  private measurements: number[] = [];

  measure() {
    if ("memory" in performance) {
      const memory = (performance as any).memory;
      this.measurements.push(memory.usedJSHeapSize);
    }
  }

  getAverage() {
    const sum = this.measurements.reduce((a, b) => a + b, 0);
    return sum / this.measurements.length;
  }

  detectLeak(threshold: number = 1.5) {
    if (this.measurements.length < 10) return false;

    const recent = this.measurements.slice(-5);
    const recentAvg = recent.reduce((a, b) => a + b, 0) / recent.length;

    const initial = this.measurements.slice(0, 5);
    const initialAvg = initial.reduce((a, b) => a + b, 0) / initial.length;

    return recentAvg / initialAvg > threshold;
  }
}

// Usage
const monitor = new MemoryMonitor();
setInterval(() => {
  monitor.measure();
  if (monitor.detectLeak()) {
    console.warn("Potential memory leak detected");
  }
}, 5000);
```

## Asset Optimization

### ✅ Image Optimization

```typescript
// Next.js Image component equivalent
import { useState, useEffect } from 'react';

interface OptimizedImageProps {
  src: string;
  alt: string;
  width: number;
  height: number;
  priority?: boolean;
  quality?: number;
}

function OptimizedImage({
  src,
  alt,
  width,
  height,
  priority = false,
  quality = 75
}: OptimizedImageProps) {
  const [imageSrc, setImageSrc] = useState(
    priority ? src : '/placeholder.jpg'
  );

  useEffect(() => {
    if (priority) return;

    const img = new Image();
    img.src = src;
    img.onload = () => setImageSrc(src);
  }, [src, priority]);

  return (
    <img
      src={imageSrc}
      alt={alt}
      width={width}
      height={height}
      loading={priority ? 'eager' : 'lazy'}
      decoding="async"
      style={{
        aspectRatio: `${width} / ${height}`,
        objectFit: 'cover'
      }}
    />
  );
}

// Responsive images
function ResponsiveImage({ src, alt }: { src: string; alt: string }) {
  return (
    <picture>
      <source
        srcSet={`${src}?w=320&format=webp 320w,
                 ${src}?w=640&format=webp 640w,
                 ${src}?w=1024&format=webp 1024w`}
        type="image/webp"
        sizes="(max-width: 320px) 320px,
               (max-width: 640px) 640px,
               1024px"
      />
      <source
        srcSet={`${src}?w=320 320w,
                 ${src}?w=640 640w,
                 ${src}?w=1024 1024w`}
        type="image/jpeg"
        sizes="(max-width: 320px) 320px,
               (max-width: 640px) 640px,
               1024px"
      />
      <img src={`${src}?w=640`} alt={alt} loading="lazy" />
    </picture>
  );
}
```

### ✅ Font Optimization

```css
/* fonts.css */
@font-face {
  font-family: "Inter";
  src: url("/fonts/inter-var.woff2") format("woff2");
  font-weight: 100 900;
  font-style: normal;
  font-display: swap; /* Avoid FOIT */
  unicode-range: U+0000-00FF, U+0131, U+0152-0153;
}

/* Subset fonts */
@font-face {
  font-family: "Inter";
  src: url("/fonts/inter-latin.woff2") format("woff2");
  font-weight: 400;
  unicode-range: U+0000-00FF;
}
```

```html
<!-- Preload critical fonts -->
<link
  rel="preload"
  href="/fonts/inter-var.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

## Caching Strategies

### ✅ Service Worker Caching

```typescript
// service-worker.ts
const CACHE_NAME = "app-v1";
const RUNTIME_CACHE = "runtime-v1";

const PRECACHE_URLS = [
  "/",
  "/index.html",
  "/assets/main.js",
  "/assets/main.css",
];

// Install event - precache assets
self.addEventListener("install", (event: ExtendableEvent) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(PRECACHE_URLS);
    }),
  );
});

// Fetch event - cache-first strategy
self.addEventListener("fetch", (event: FetchEvent) => {
  const { request } = event;

  // Network-first for API calls
  if (request.url.includes("/api/")) {
    event.respondWith(
      fetch(request)
        .then((response) => {
          const responseClone = response.clone();
          caches.open(RUNTIME_CACHE).then((cache) => {
            cache.put(request, responseClone);
          });
          return response;
        })
        .catch(() => caches.match(request)),
    );
    return;
  }

  // Cache-first for assets
  event.respondWith(
    caches.match(request).then((cached) => {
      return (
        cached ||
        fetch(request).then((response) => {
          return caches.open(RUNTIME_CACHE).then((cache) => {
            cache.put(request, response.clone());
            return response;
          });
        })
      );
    }),
  );
});

// Activate event - cleanup old caches
self.addEventListener("activate", (event: ExtendableEvent) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME && name !== RUNTIME_CACHE)
          .map((name) => caches.delete(name)),
      );
    }),
  );
});
```

### ✅ HTTP Caching Headers

```typescript
// Express server example
app.use((req, res, next) => {
  // Static assets - long cache
  if (req.url.match(/\.(js|css|png|jpg|jpeg|gif|ico|woff2)$/)) {
    res.setHeader("Cache-Control", "public, max-age=31536000, immutable");
  }
  // HTML - no cache
  else if (req.url.endsWith(".html")) {
    res.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");
  }
  // API responses - short cache with revalidation
  else if (req.url.startsWith("/api/")) {
    res.setHeader(
      "Cache-Control",
      "public, max-age=60, stale-while-revalidate=30",
    );
  }

  next();
});
```

## Monitoring and Metrics

### ✅ Web Vitals Tracking

```typescript
// track-vitals.ts
import { onCLS, onFID, onFCP, onLCP, onTTFB } from "web-vitals";

interface AnalyticsEvent {
  name: string;
  value: number;
  rating: "good" | "needs-improvement" | "poor";
  delta: number;
}

function sendToAnalytics(metric: AnalyticsEvent) {
  // Send to your analytics service
  fetch("/api/analytics", {
    method: "POST",
    body: JSON.stringify(metric),
    headers: { "Content-Type": "application/json" },
  });
}

export function initVitalsTracking() {
  onCLS((metric) => sendToAnalytics({ ...metric, name: "CLS" }));
  onFID((metric) => sendToAnalytics({ ...metric, name: "FID" }));
  onFCP((metric) => sendToAnalytics({ ...metric, name: "FCP" }));
  onLCP((metric) => sendToAnalytics({ ...metric, name: "LCP" }));
  onTTFB((metric) => sendToAnalytics({ ...metric, name: "TTFB" }));
}

// Custom performance marks
export function measureFeature(name: string, fn: () => void) {
  performance.mark(`${name}-start`);
  fn();
  performance.mark(`${name}-end`);
  performance.measure(name, `${name}-start`, `${name}-end`);

  const measure = performance.getEntriesByName(name)[0];
  sendToAnalytics({
    name,
    value: measure.duration,
    rating: measure.duration < 100 ? "good" : "needs-improvement",
    delta: 0,
  });
}
```

### ✅ Performance Observer

```typescript
// Observe long tasks
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      console.warn("Long task detected:", entry);
      sendToAnalytics({
        name: "long-task",
        value: entry.duration,
        rating: "poor",
        delta: 0,
      });
    }
  }
});

observer.observe({ entryTypes: ["longtask"] });

// Monitor resource loading
const resourceObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 1000) {
      console.warn("Slow resource:", entry.name, entry.duration);
    }
  }
});

resourceObserver.observe({ entryTypes: ["resource"] });
```

## Mobile Performance

### ✅ Mobile-Specific Optimizations

```typescript
// Detect device capabilities
function getDeviceCapabilities() {
  const connection = (navigator as any).connection;

  return {
    effectiveType: connection?.effectiveType || '4g',
    saveData: connection?.saveData || false,
    deviceMemory: (navigator as any).deviceMemory || 8,
    hardwareConcurrency: navigator.hardwareConcurrency || 4
  };
}

// Adaptive loading
function useAdaptiveLoading() {
  const capabilities = getDeviceCapabilities();

  const shouldLoadHighQuality =
    capabilities.effectiveType === '4g' &&
    !capabilities.saveData &&
    capabilities.deviceMemory >= 4;

  return { shouldLoadHighQuality };
}

// Usage
function ImageGallery({ images }: { images: string[] }) {
  const { shouldLoadHighQuality } = useAdaptiveLoading();

  return (
    <div>
      {images.map((src, i) => (
        <OptimizedImage
          key={i}
          src={src}
          quality={shouldLoadHighQuality ? 90 : 60}
          width={shouldLoadHighQuality ? 1200 : 600}
        />
      ))}
    </div>
  );
}

// Touch optimization
function useTouchOptimization() {
  useEffect(() => {
    // Prevent double-tap zoom
    let lastTouchEnd = 0;

    const handler = (e: TouchEvent) => {
      const now = Date.now();
      if (now - lastTouchEnd <= 300) {
        e.preventDefault();
      }
      lastTouchEnd = now;
    };

    document.addEventListener('touchend', handler, { passive: false });

    return () => {
      document.removeEventListener('touchend', handler);
    };
  }, []);
}
```

## Advanced Optimizations

### ✅ Preloading and Prefetching

```typescript
// Router-based prefetching
import { useEffect } from "react";
import { useLocation } from "react-router-dom";

const routePrefetchMap: Record<string, string[]> = {
  "/": ["/dashboard", "/products"],
  "/dashboard": ["/profile", "/settings"],
  "/products": ["/products/[id]"],
};

function usePrefetch() {
  const location = useLocation();

  useEffect(() => {
    const routesToPrefetch = routePrefetchMap[location.pathname] || [];

    routesToPrefetch.forEach((route) => {
      const link = document.createElement("link");
      link.rel = "prefetch";
      link.href = route;
      link.as = "document";
      document.head.appendChild(link);
    });
  }, [location.pathname]);
}

// Predictive prefetching based on user behavior
class PredictivePrefetcher {
  private linkHoverTime = new Map<string, number>();
  private prefetchThreshold = 300; // ms

  observe() {
    document.addEventListener("mouseover", (e) => {
      const link = (e.target as HTMLElement).closest("a");
      if (!link) return;

      const href = link.getAttribute("href");
      if (!href) return;

      this.linkHoverTime.set(href, Date.now());

      setTimeout(() => {
        const hoverTime = this.linkHoverTime.get(href);
        if (hoverTime && Date.now() - hoverTime >= this.prefetchThreshold) {
          this.prefetch(href);
        }
      }, this.prefetchThreshold);
    });

    document.addEventListener("mouseout", (e) => {
      const link = (e.target as HTMLElement).closest("a");
      if (!link) return;

      const href = link.getAttribute("href");
      if (href) {
        this.linkHoverTime.delete(href);
      }
    });
  }

  private prefetch(href: string) {
    const link = document.createElement("link");
    link.rel = "prefetch";
    link.href = href;
    document.head.appendChild(link);
  }
}

// Initialize
const prefetcher = new PredictivePrefetcher();
prefetcher.observe();
```

### ✅ Critical Path Optimization

```typescript
// Inline critical CSS
const criticalCSS = `
  body { margin: 0; font-family: system-ui; }
  .header { height: 60px; background: #000; }
  .hero { min-height: 400px; }
`;

// Defer non-critical CSS
function loadCSS(href: string) {
  const link = document.createElement("link");
  link.rel = "stylesheet";
  link.href = href;
  link.media = "print";
  link.onload = () => {
    link.media = "all";
  };
  document.head.appendChild(link);
}

// Defer non-critical JavaScript
const script = document.createElement("script");
script.src = "/non-critical.js";
script.defer = true;
document.body.appendChild(script);
```

## Performance Testing

### ✅ Automated Performance Tests

```typescript
// performance.test.ts
import { test, expect } from "@playwright/test";

test.describe("Performance", () => {
  test("should load home page within budget", async ({ page }) => {
    await page.goto("/");

    const metrics = await page.evaluate(() => {
      const navigation = performance.getEntriesByType(
        "navigation",
      )[0] as PerformanceNavigationTiming;
      return {
        fcp:
          performance.getEntriesByName("first-contentful-paint")[0]
            ?.startTime || 0,
        lcp: 0, // Would need observer
        domContentLoaded:
          navigation.domContentLoadedEventEnd - navigation.fetchStart,
        loadComplete: navigation.loadEventEnd - navigation.fetchStart,
      };
    });

    expect(metrics.fcp).toBeLessThan(1800);
    expect(metrics.domContentLoaded).toBeLessThan(2000);
    expect(metrics.loadComplete).toBeLessThan(5000);
  });

  test("should have acceptable bundle size", async ({ page }) => {
    const response = await page.goto("/");
    const resources = await page.evaluate(() => {
      return performance
        .getEntriesByType("resource")
        .filter((r: any) => r.name.includes(".js"))
        .map((r: any) => ({
          name: r.name,
          size: r.transferSize,
        }));
    });

    const totalJSSize = resources.reduce((sum, r) => sum + r.size, 0);
    expect(totalJSSize).toBeLessThan(200000); // 200KB
  });
});
```

### ✅ Lighthouse CI

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI

on:
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "18"

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          urls: |
            http://localhost:3000
            http://localhost:3000/dashboard
          budgetPath: ./budget.json
          uploadArtifacts: true
```

```json
// budget.json - Performance budgets
{
  "budget": [
    {
      "resourceSizes": [
        {
          "resourceType": "script",
          "budget": 200
        },
        {
          "resourceType": "stylesheet",
          "budget": 50
        },
        {
          "resourceType": "image",
          "budget": 300
        }
      ],
      "resourceCounts": [
        {
          "resourceType": "third-party",
          "budget": 10
        }
      ],
      "timings": [
        {
          "metric": "first-contentful-paint",
          "budget": 1800
        },
        {
          "metric": "largest-contentful-paint",
          "budget": 2500
        },
        {
          "metric": "interactive",
          "budget": 3800
        }
      ]
    }
  ]
}
```

## Key Takeaways

1. **Measure First**: Use tools like Lighthouse, WebPageTest, and Chrome DevTools to establish baseline metrics before optimizing

2. **Set Performance Budgets**: Define clear thresholds for Core Web Vitals (LCP < 2.5s, FID < 100ms, CLS < 0.1) and enforce them in CI/CD

3. **Optimize Critical Path**: Inline critical CSS, defer non-critical resources, and use resource hints (preload, prefetch, preconnect)

4. **Code Splitting**: Split code by routes and features, lazy load non-critical components, use dynamic imports

5. **Image Optimization**: Use WebP/AVIF formats, responsive images, lazy loading, and appropriate quality settings (75-85%)

6. **Minimize JavaScript**: Tree shake unused code, minimize bundle size, use compression (Brotli > gzip), remove console.logs in production

7. **Cache Aggressively**: Implement service workers, use HTTP caching headers, leverage browser cache, implement stale-while-revalidate

8. **Monitor Real Users**: Track Core Web Vitals in production, use RUM (Real User Monitoring), alert on performance degradation

9. **React Optimizations**: Use React.memo, useMemo, useCallback, virtual scrolling for large lists, debounce user input

10. **Test Performance**: Automate performance tests in CI, use Lighthouse CI, load test critical flows, monitor performance over time

---

**Pre-Launch Performance Checklist**:

```bash
✅ Performance Metrics
□ LCP < 2.5s
□ FID < 100ms
□ CLS < 0.1
□ FCP < 1.8s
□ TTI < 3.8s

✅ Bundle Size
□ JavaScript < 200KB (gzipped)
□ CSS < 50KB (gzipped)
□ Images optimized (WebP/AVIF)
□ Fonts < 100KB total

✅ Network
□ Enable HTTP/2
□ Enable Brotli compression
□ CDN configured
□ API response times < 200ms

✅ Caching
□ Service worker registered
□ Cache headers configured
□ Immutable assets have long cache
□ HTML has no-cache

✅ Rendering
□ No layout shifts
□ Smooth animations (60fps)
□ No blocking scripts
□ Critical CSS inlined

✅ Monitoring
□ Web Vitals tracking enabled
□ Performance alerts configured
□ Real User Monitoring active
□ Error tracking enabled

# Ready to launch! 🚀
```
