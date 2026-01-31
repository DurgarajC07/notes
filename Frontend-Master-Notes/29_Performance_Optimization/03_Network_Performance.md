# Network Performance Optimization

## Core Concepts

Network performance is critical for web application speed and user experience. Modern web apps rely on efficient data transfer, intelligent caching, and optimized resource delivery. Key areas include HTTP/2 multiplexing, compression algorithms, caching strategies, resource hints, service workers, and CDN optimization.

### Key Network Metrics

- **Time to First Byte (TTFB)**: Time from request to first byte received
- **Connection Time**: DNS lookup + TCP handshake + TLS negotiation
- **Download Time**: Time to download complete resource
- **Round Trip Time (RTT)**: Time for request-response cycle
- **Bandwidth**: Maximum data transfer rate
- **Latency**: Delay in network communication

## TypeScript/JavaScript Code Examples

### Example 1: HTTP/2 Server Push Configuration

```typescript
// Express server with HTTP/2 push
import http2 from "http2";
import fs from "fs";
import path from "path";

interface PushResource {
  path: string;
  mimeType: string;
  priority?: number;
}

class HTTP2Server {
  private server: http2.Http2SecureServer;
  private pushResources: Map<string, PushResource[]> = new Map();

  constructor(options: http2.SecureServerOptions) {
    this.server = http2.createSecureServer(options);
    this.setupRoutes();
  }

  // Configure resources to push for specific routes
  configurePush(route: string, resources: PushResource[]): void {
    this.pushResources.set(route, resources);
  }

  private setupRoutes(): void {
    this.server.on("stream", (stream, headers) => {
      const path = headers[":path"] as string;

      // Push associated resources
      if (this.pushResources.has(path)) {
        this.pushResources.get(path)?.forEach((resource) => {
          this.pushResource(stream, resource);
        });
      }

      // Serve main content
      this.serveContent(stream, path);
    });
  }

  private pushResource(
    stream: http2.ServerHttp2Stream,
    resource: PushResource,
  ): void {
    try {
      stream.pushStream({ ":path": resource.path }, (err, pushStream) => {
        if (err) {
          console.error("Push failed:", err);
          return;
        }

        pushStream.respond({
          ":status": 200,
          "content-type": resource.mimeType,
          "cache-control": "public, max-age=31536000, immutable",
        });

        const fileStream = fs.createReadStream(
          path.join(__dirname, resource.path),
        );
        fileStream.pipe(pushStream);
      });
    } catch (error) {
      console.error("Push stream error:", error);
    }
  }

  private serveContent(
    stream: http2.ServerHttp2Stream,
    requestPath: string,
  ): void {
    const filePath = path.join(__dirname, "public", requestPath);

    stream.respondWithFile(filePath, {
      "content-type": this.getMimeType(filePath),
      "cache-control": "public, max-age=3600",
    });
  }

  private getMimeType(filePath: string): string {
    const ext = path.extname(filePath);
    const mimeTypes: Record<string, string> = {
      ".html": "text/html",
      ".js": "application/javascript",
      ".css": "text/css",
      ".json": "application/json",
    };
    return mimeTypes[ext] || "application/octet-stream";
  }

  listen(port: number): void {
    this.server.listen(port);
    console.log(`HTTP/2 server listening on port ${port}`);
  }
}

// Usage
const server = new HTTP2Server({
  key: fs.readFileSync("key.pem"),
  cert: fs.readFileSync("cert.pem"),
});

// Configure critical resources to push
server.configurePush("/", [
  { path: "/styles/critical.css", mimeType: "text/css" },
  { path: "/scripts/main.js", mimeType: "application/javascript" },
  { path: "/fonts/inter-var.woff2", mimeType: "font/woff2" },
]);

server.listen(443);
```

### Example 2: Advanced Compression Strategy

```typescript
// Compression middleware with intelligent selection
import { Request, Response, NextFunction } from "express";
import zlib from "zlib";
import { promisify } from "util";

const brotliCompress = promisify(zlib.brotliCompress);
const gzipCompress = promisify(zlib.gzip);

interface CompressionOptions {
  threshold: number; // Minimum size to compress
  level: number; // Compression level (0-11 for Brotli, 0-9 for gzip)
  mimeTypes: RegExp;
}

class CompressionMiddleware {
  private options: CompressionOptions;
  private cache: Map<string, { br: Buffer; gzip: Buffer }> = new Map();

  constructor(options: Partial<CompressionOptions> = {}) {
    this.options = {
      threshold: options.threshold ?? 1024, // 1KB
      level: options.level ?? 6,
      mimeTypes: options.mimeTypes ?? /text|javascript|json|xml|svg/i,
    };
  }

  middleware() {
    return async (req: Request, res: Response, next: NextFunction) => {
      const originalSend = res.send.bind(res);
      const originalJson = res.json.bind(res);

      // Override send method
      res.send = (body: any) => {
        return this.compressAndSend(req, res, body, originalSend);
      };

      // Override json method
      res.json = (body: any) => {
        const jsonString = JSON.stringify(body);
        return this.compressAndSend(req, res, jsonString, originalSend);
      };

      next();
    };
  }

  private compressAndSend(
    req: Request,
    res: Response,
    body: any,
    originalSend: Function,
  ): Response {
    // Check if compression is appropriate
    if (!this.shouldCompress(req, res, body)) {
      return originalSend(body);
    }

    const buffer = Buffer.isBuffer(body) ? body : Buffer.from(body);

    // Use cached compression if available
    const cacheKey = this.getCacheKey(buffer);
    if (this.cache.has(cacheKey)) {
      return this.sendCompressed(req, res, this.cache.get(cacheKey)!);
    }

    // Compress in parallel
    this.compressContent(buffer)
      .then((compressed) => {
        this.cache.set(cacheKey, compressed);
        return this.sendCompressed(req, res, compressed);
      })
      .catch((err) => {
        console.error("Compression failed:", err);
        return originalSend(body);
      });

    return res;
  }

  private shouldCompress(req: Request, res: Response, body: any): boolean {
    const contentType = res.getHeader("content-type") as string;

    // Check mime type
    if (!this.options.mimeTypes.test(contentType)) {
      return false;
    }

    // Check size threshold
    const bodySize = Buffer.byteLength(body);
    if (bodySize < this.options.threshold) {
      return false;
    }

    // Check if client accepts compression
    const acceptEncoding = req.headers["accept-encoding"] || "";
    return /br|gzip/.test(acceptEncoding);
  }

  private async compressContent(
    buffer: Buffer,
  ): Promise<{ br: Buffer; gzip: Buffer }> {
    const [br, gzip] = await Promise.all([
      brotliCompress(buffer, {
        params: {
          [zlib.constants.BROTLI_PARAM_MODE]: zlib.constants.BROTLI_MODE_TEXT,
          [zlib.constants.BROTLI_PARAM_QUALITY]: this.options.level,
        },
      }),
      gzipCompress(buffer, { level: Math.min(this.options.level, 9) }),
    ]);

    return { br, gzip };
  }

  private sendCompressed(
    req: Request,
    res: Response,
    compressed: { br: Buffer; gzip: Buffer },
  ): Response {
    const acceptEncoding = req.headers["accept-encoding"] || "";

    if (acceptEncoding.includes("br")) {
      res.setHeader("Content-Encoding", "br");
      res.setHeader("Content-Length", compressed.br.length);
      return res.end(compressed.br);
    } else if (acceptEncoding.includes("gzip")) {
      res.setHeader("Content-Encoding", "gzip");
      res.setHeader("Content-Length", compressed.gzip.length);
      return res.end(compressed.gzip);
    }

    return res;
  }

  private getCacheKey(buffer: Buffer): string {
    return require("crypto").createHash("sha256").update(buffer).digest("hex");
  }

  clearCache(): void {
    this.cache.clear();
  }
}

// Usage
const compression = new CompressionMiddleware({
  threshold: 2048,
  level: 8,
  mimeTypes: /text|javascript|json|xml|svg|font/i,
});

app.use(compression.middleware());
```

### Example 3: Smart Caching Headers Manager

```typescript
// Advanced cache control with versioning
interface CacheStrategy {
  maxAge: number;
  sMaxAge?: number;
  staleWhileRevalidate?: number;
  staleIfError?: number;
  mustRevalidate?: boolean;
  immutable?: boolean;
  private?: boolean;
}

class CacheHeaderManager {
  private strategies: Map<RegExp, CacheStrategy> = new Map();
  private etags: Map<string, string> = new Map();

  // Define caching strategy for resource patterns
  defineStrategy(pattern: RegExp, strategy: CacheStrategy): void {
    this.strategies.set(pattern, strategy);
  }

  // Generate cache control header
  generateCacheControl(strategy: CacheStrategy): string {
    const directives: string[] = [];

    if (strategy.private) {
      directives.push("private");
    } else {
      directives.push("public");
    }

    directives.push(`max-age=${strategy.maxAge}`);

    if (strategy.sMaxAge !== undefined) {
      directives.push(`s-maxage=${strategy.sMaxAge}`);
    }

    if (strategy.staleWhileRevalidate !== undefined) {
      directives.push(
        `stale-while-revalidate=${strategy.staleWhileRevalidate}`,
      );
    }

    if (strategy.staleIfError !== undefined) {
      directives.push(`stale-if-error=${strategy.staleIfError}`);
    }

    if (strategy.mustRevalidate) {
      directives.push("must-revalidate");
    }

    if (strategy.immutable) {
      directives.push("immutable");
    }

    return directives.join(", ");
  }

  // Generate ETag for content
  generateETag(content: string | Buffer, weak: boolean = false): string {
    const hash = require("crypto")
      .createHash("sha256")
      .update(content)
      .digest("hex")
      .substring(0, 16);

    return weak ? `W/"${hash}"` : `"${hash}"`;
  }

  // Middleware for automatic cache headers
  middleware() {
    return (req: Request, res: Response, next: NextFunction) => {
      const originalSend = res.send.bind(res);

      res.send = (body: any) => {
        const path = req.path;
        const strategy = this.getStrategy(path);

        if (strategy) {
          // Set Cache-Control
          res.setHeader("Cache-Control", this.generateCacheControl(strategy));

          // Generate and check ETag
          const etag = this.generateETag(body, !strategy.mustRevalidate);
          res.setHeader("ETag", etag);

          // Handle If-None-Match
          const ifNoneMatch = req.headers["if-none-match"];
          if (ifNoneMatch === etag) {
            res.status(304).end();
            return res;
          }

          // Set Vary header for proper caching
          res.setHeader("Vary", "Accept-Encoding");
        }

        return originalSend(body);
      };

      next();
    };
  }

  private getStrategy(path: string): CacheStrategy | undefined {
    for (const [pattern, strategy] of this.strategies) {
      if (pattern.test(path)) {
        return strategy;
      }
    }
    return undefined;
  }
}

// Usage with predefined strategies
const cacheManager = new CacheHeaderManager();

// Static assets with hash (immutable, long cache)
cacheManager.defineStrategy(/\.(js|css|woff2|png)\.\w{8}\./, {
  maxAge: 31536000, // 1 year
  sMaxAge: 31536000,
  immutable: true,
});

// API responses (short cache with revalidation)
cacheManager.defineStrategy(/^\/api\//, {
  maxAge: 300, // 5 minutes
  sMaxAge: 600, // 10 minutes for CDN
  staleWhileRevalidate: 86400, // 1 day
  mustRevalidate: true,
  private: false,
});

// User-specific data (no shared cache)
cacheManager.defineStrategy(/^\/api\/user\//, {
  maxAge: 0,
  mustRevalidate: true,
  private: true,
});

// HTML pages (revalidate but serve stale on error)
cacheManager.defineStrategy(/\.html$/, {
  maxAge: 3600, // 1 hour
  staleWhileRevalidate: 86400, // 1 day
  staleIfError: 604800, // 1 week
});

app.use(cacheManager.middleware());
```

### Example 4: Resource Hints Manager

```typescript
// Intelligent resource hints for optimal loading
interface ResourceHint {
  href: string;
  as?: string;
  type?: string;
  crossOrigin?: "anonymous" | "use-credentials";
  importance?: "high" | "low" | "auto";
  fetchPriority?: "high" | "low" | "auto";
}

class ResourceHintManager {
  private preconnects: ResourceHint[] = [];
  private preloads: ResourceHint[] = [];
  private prefetches: ResourceHint[] = [];
  private dnsPrefetches: string[] = [];

  // Add DNS prefetch for early domain resolution
  addDNSPrefetch(domain: string): void {
    if (!this.dnsPrefetches.includes(domain)) {
      this.dnsPrefetches.push(domain);
    }
  }

  // Add preconnect for warm connection
  addPreconnect(hint: ResourceHint): void {
    this.preconnects.push(hint);
  }

  // Add preload for critical resources
  addPreload(hint: ResourceHint): void {
    this.preloads.push(hint);
  }

  // Add prefetch for future navigation
  addPrefetch(hint: ResourceHint): void {
    this.prefetches.push(hint);
  }

  // Generate HTML link tags
  generateLinks(): string[] {
    const links: string[] = [];

    // DNS Prefetch
    this.dnsPrefetches.forEach((domain) => {
      links.push(`<link rel="dns-prefetch" href="${domain}">`);
    });

    // Preconnect
    this.preconnects.forEach((hint) => {
      const attrs = [`rel="preconnect"`, `href="${hint.href}"`];
      if (hint.crossOrigin) {
        attrs.push(`crossorigin="${hint.crossOrigin}"`);
      }
      links.push(`<link ${attrs.join(" ")}>`);
    });

    // Preload
    this.preloads.forEach((hint) => {
      const attrs = [`rel="preload"`, `href="${hint.href}"`, `as="${hint.as}"`];
      if (hint.type) attrs.push(`type="${hint.type}"`);
      if (hint.crossOrigin) attrs.push(`crossorigin="${hint.crossOrigin}"`);
      if (hint.importance) attrs.push(`importance="${hint.importance}"`);
      if (hint.fetchPriority)
        attrs.push(`fetchpriority="${hint.fetchPriority}"`);
      links.push(`<link ${attrs.join(" ")}>`);
    });

    // Prefetch
    this.prefetches.forEach((hint) => {
      links.push(`<link rel="prefetch" href="${hint.href}">`);
    });

    return links;
  }

  // Generate HTTP headers
  generateHeaders(): Record<string, string> {
    const headers: Record<string, string> = {};
    const linkHeaders: string[] = [];

    // Combine all hints into Link header
    this.preconnects.forEach((hint) => {
      linkHeaders.push(`<${hint.href}>; rel=preconnect`);
    });

    this.preloads.forEach((hint) => {
      let header = `<${hint.href}>; rel=preload; as=${hint.as}`;
      if (hint.crossOrigin) header += `; crossorigin=${hint.crossOrigin}`;
      linkHeaders.push(header);
    });

    if (linkHeaders.length > 0) {
      headers["Link"] = linkHeaders.join(", ");
    }

    return headers;
  }

  // Inject hints into HTML
  injectIntoHTML(html: string): string {
    const links = this.generateLinks();
    const linksHTML = links.join("\n    ");
    return html.replace("</head>", `    ${linksHTML}\n  </head>`);
  }
}

// Usage for e-commerce site
const hintManager = new ResourceHintManager();

// DNS Prefetch for third-party domains
hintManager.addDNSPrefetch("https://cdn.example.com");
hintManager.addDNSPrefetch("https://analytics.google.com");
hintManager.addDNSPrefetch("https://fonts.googleapis.com");

// Preconnect for critical third-party resources
hintManager.addPreconnect({
  href: "https://cdn.example.com",
  crossOrigin: "anonymous",
});

// Preload critical resources
hintManager.addPreload({
  href: "/fonts/inter-var.woff2",
  as: "font",
  type: "font/woff2",
  crossOrigin: "anonymous",
  importance: "high",
});

hintManager.addPreload({
  href: "/styles/critical.css",
  as: "style",
  importance: "high",
});

hintManager.addPreload({
  href: "/scripts/main.js",
  as: "script",
  fetchPriority: "high",
});

// Prefetch for likely next navigation
hintManager.addPrefetch({
  href: "/product-details.html",
});

hintManager.addPrefetch({
  href: "/api/recommended-products",
});

// Apply to response
app.get("/", (req, res) => {
  let html = fs.readFileSync("index.html", "utf-8");
  html = hintManager.injectIntoHTML(html);

  const headers = hintManager.generateHeaders();
  Object.entries(headers).forEach(([key, value]) => {
    res.setHeader(key, value);
  });

  res.send(html);
});
```

### Example 5: Service Worker with Advanced Caching

```typescript
// Service worker with network-first and cache-first strategies
/// <reference lib="webworker" />

interface CacheConfig {
  name: string;
  version: number;
  maxAge: number;
  maxEntries: number;
}

interface CacheStrategy {
  cacheName: string;
  strategy: "network-first" | "cache-first" | "stale-while-revalidate";
  networkTimeout?: number;
}

class ServiceWorkerCache {
  private configs: Map<string, CacheConfig> = new Map();
  private strategies: Map<RegExp, CacheStrategy> = new Map();

  constructor() {
    this.initializeConfigs();
    this.setupEventListeners();
  }

  private initializeConfigs(): void {
    // Static assets cache
    this.configs.set("static-v1", {
      name: "static-v1",
      version: 1,
      maxAge: 30 * 24 * 60 * 60 * 1000, // 30 days
      maxEntries: 50,
    });

    // API cache
    this.configs.set("api-v1", {
      name: "api-v1",
      version: 1,
      maxAge: 5 * 60 * 1000, // 5 minutes
      maxEntries: 100,
    });

    // Images cache
    this.configs.set("images-v1", {
      name: "images-v1",
      version: 1,
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
      maxEntries: 200,
    });
  }

  private setupEventListeners(): void {
    self.addEventListener("install", (event: ExtendableEvent) => {
      event.waitUntil(this.handleInstall());
    });

    self.addEventListener("activate", (event: ExtendableEvent) => {
      event.waitUntil(this.handleActivate());
    });

    self.addEventListener("fetch", (event: FetchEvent) => {
      event.respondWith(this.handleFetch(event.request));
    });
  }

  private async handleInstall(): Promise<void> {
    console.log("Service Worker installing...");

    // Pre-cache critical resources
    const cache = await caches.open("static-v1");
    await cache.addAll([
      "/",
      "/styles/critical.css",
      "/scripts/main.js",
      "/offline.html",
    ]);

    await self.skipWaiting();
  }

  private async handleActivate(): Promise<void> {
    console.log("Service Worker activating...");

    // Clean up old caches
    const cacheNames = await caches.keys();
    const validCaches = Array.from(this.configs.keys());

    await Promise.all(
      cacheNames.map((cacheName) => {
        if (!validCaches.includes(cacheName)) {
          console.log("Deleting old cache:", cacheName);
          return caches.delete(cacheName);
        }
      }),
    );

    await self.clients.claim();
  }

  private async handleFetch(request: Request): Promise<Response> {
    const strategy = this.getStrategy(request.url);

    if (!strategy) {
      return fetch(request);
    }

    switch (strategy.strategy) {
      case "network-first":
        return this.networkFirst(request, strategy);
      case "cache-first":
        return this.cacheFirst(request, strategy);
      case "stale-while-revalidate":
        return this.staleWhileRevalidate(request, strategy);
      default:
        return fetch(request);
    }
  }

  private async networkFirst(
    request: Request,
    strategy: CacheStrategy,
  ): Promise<Response> {
    try {
      const fetchPromise = fetch(request);
      const timeoutPromise = strategy.networkTimeout
        ? this.timeout(strategy.networkTimeout)
        : null;

      const response = timeoutPromise
        ? await Promise.race([fetchPromise, timeoutPromise])
        : await fetchPromise;

      if (response && response.ok) {
        const cache = await caches.open(strategy.cacheName);
        await cache.put(request, response.clone());
      }

      return response;
    } catch (error) {
      console.log("Network failed, trying cache:", error);
      const cachedResponse = await caches.match(request);

      if (cachedResponse) {
        return cachedResponse;
      }

      // Return offline page for navigation requests
      if (request.mode === "navigate") {
        const offlineResponse = await caches.match("/offline.html");
        return offlineResponse || new Response("Offline");
      }

      throw error;
    }
  }

  private async cacheFirst(
    request: Request,
    strategy: CacheStrategy,
  ): Promise<Response> {
    const cachedResponse = await caches.match(request);

    if (cachedResponse) {
      // Check if cached response is still fresh
      const cachedTime = new Date(
        cachedResponse.headers.get("date") || "",
      ).getTime();
      const config = this.configs.get(strategy.cacheName);

      if (config && Date.now() - cachedTime < config.maxAge) {
        return cachedResponse;
      }
    }

    try {
      const response = await fetch(request);

      if (response && response.ok) {
        const cache = await caches.open(strategy.cacheName);
        await cache.put(request, response.clone());
      }

      return response;
    } catch (error) {
      if (cachedResponse) {
        return cachedResponse;
      }
      throw error;
    }
  }

  private async staleWhileRevalidate(
    request: Request,
    strategy: CacheStrategy,
  ): Promise<Response> {
    const cachedResponse = await caches.match(request);

    const fetchPromise = fetch(request).then(async (response) => {
      if (response && response.ok) {
        const cache = await caches.open(strategy.cacheName);
        await cache.put(request, response.clone());
      }
      return response;
    });

    return cachedResponse || fetchPromise;
  }

  private timeout(ms: number): Promise<Response> {
    return new Promise((_, reject) => {
      setTimeout(() => reject(new Error("Network timeout")), ms);
    });
  }

  private getStrategy(url: string): CacheStrategy | undefined {
    for (const [pattern, strategy] of this.strategies) {
      if (pattern.test(url)) {
        return strategy;
      }
    }
    return undefined;
  }

  // Configure strategies
  addStrategy(pattern: RegExp, strategy: CacheStrategy): void {
    this.strategies.set(pattern, strategy);
  }
}

// Initialize service worker
const swCache = new ServiceWorkerCache();

// Configure caching strategies
swCache.addStrategy(/\.(js|css|woff2)$/, {
  cacheName: "static-v1",
  strategy: "cache-first",
});

swCache.addStrategy(/^https:\/\/api\.example\.com\//, {
  cacheName: "api-v1",
  strategy: "network-first",
  networkTimeout: 3000,
});

swCache.addStrategy(/\.(png|jpg|jpeg|svg|webp)$/, {
  cacheName: "images-v1",
  strategy: "stale-while-revalidate",
});

swCache.addStrategy(/^https:\/\/example\.com\//, {
  cacheName: "static-v1",
  strategy: "network-first",
  networkTimeout: 5000,
});
```

### Example 6: CDN Strategy with Fallback

```typescript
// Multi-CDN strategy with automatic fallback
interface CDNConfig {
  name: string;
  baseUrl: string;
  priority: number;
  timeout: number;
  healthCheck?: string;
}

class CDNManager {
  private cdns: CDNConfig[] = [];
  private healthStatus: Map<string, boolean> = new Map();
  private failureCount: Map<string, number> = new Map();
  private readonly MAX_FAILURES = 3;

  constructor(cdns: CDNConfig[]) {
    this.cdns = cdns.sort((a, b) => a.priority - b.priority);
    this.initializeHealthChecks();
  }

  private async initializeHealthChecks(): Promise<void> {
    // Check CDN health periodically
    setInterval(() => this.checkAllCDNs(), 60000); // Every minute
    await this.checkAllCDNs();
  }

  private async checkAllCDNs(): Promise<void> {
    await Promise.all(this.cdns.map((cdn) => this.checkCDNHealth(cdn)));
  }

  private async checkCDNHealth(cdn: CDNConfig): Promise<void> {
    if (!cdn.healthCheck) {
      this.healthStatus.set(cdn.name, true);
      return;
    }

    try {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), cdn.timeout);

      const response = await fetch(`${cdn.baseUrl}${cdn.healthCheck}`, {
        method: "HEAD",
        signal: controller.signal,
      });

      clearTimeout(timeoutId);

      const isHealthy = response.ok;
      this.healthStatus.set(cdn.name, isHealthy);

      if (isHealthy) {
        this.failureCount.set(cdn.name, 0);
      }
    } catch (error) {
      console.error(`CDN health check failed for ${cdn.name}:`, error);
      this.healthStatus.set(cdn.name, false);
    }
  }

  async fetchResource(path: string): Promise<Response> {
    for (const cdn of this.cdns) {
      // Skip unhealthy CDNs
      if (!this.isHealthy(cdn)) {
        continue;
      }

      try {
        const url = `${cdn.baseUrl}${path}`;
        const response = await this.fetchWithTimeout(url, cdn.timeout);

        if (response.ok) {
          this.recordSuccess(cdn);
          return response;
        }

        this.recordFailure(cdn);
      } catch (error) {
        console.error(`CDN fetch failed for ${cdn.name}:`, error);
        this.recordFailure(cdn);
      }
    }

    throw new Error("All CDNs failed");
  }

  private async fetchWithTimeout(
    url: string,
    timeout: number,
  ): Promise<Response> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeout);

    try {
      const response = await fetch(url, {
        signal: controller.signal,
      });
      clearTimeout(timeoutId);
      return response;
    } catch (error) {
      clearTimeout(timeoutId);
      throw error;
    }
  }

  private isHealthy(cdn: CDNConfig): boolean {
    const failures = this.failureCount.get(cdn.name) || 0;
    return (
      failures < this.MAX_FAILURES && this.healthStatus.get(cdn.name) !== false
    );
  }

  private recordSuccess(cdn: CDNConfig): void {
    this.failureCount.set(cdn.name, 0);
    this.healthStatus.set(cdn.name, true);
  }

  private recordFailure(cdn: CDNConfig): void {
    const failures = (this.failureCount.get(cdn.name) || 0) + 1;
    this.failureCount.set(cdn.name, failures);

    if (failures >= this.MAX_FAILURES) {
      this.healthStatus.set(cdn.name, false);
    }
  }

  // Get best available CDN
  getBestCDN(): CDNConfig | undefined {
    return this.cdns.find((cdn) => this.isHealthy(cdn));
  }

  // Get CDN status report
  getStatus(): Record<string, any> {
    return this.cdns.reduce(
      (status, cdn) => {
        status[cdn.name] = {
          healthy: this.isHealthy(cdn),
          failures: this.failureCount.get(cdn.name) || 0,
        };
        return status;
      },
      {} as Record<string, any>,
    );
  }
}

// Usage
const cdnManager = new CDNManager([
  {
    name: "Primary CDN",
    baseUrl: "https://cdn1.example.com",
    priority: 1,
    timeout: 3000,
    healthCheck: "/health",
  },
  {
    name: "Secondary CDN",
    baseUrl: "https://cdn2.example.com",
    priority: 2,
    timeout: 3000,
    healthCheck: "/health",
  },
  {
    name: "Origin Server",
    baseUrl: "https://origin.example.com",
    priority: 3,
    timeout: 5000,
  },
]);

// Fetch resource with automatic fallback
async function loadAsset(path: string): Promise<Response> {
  try {
    return await cdnManager.fetchResource(path);
  } catch (error) {
    console.error("Failed to load asset:", error);
    throw error;
  }
}

// Example: Load critical CSS
loadAsset("/styles/main.css")
  .then((response) => response.text())
  .then((css) => {
    const style = document.createElement("style");
    style.textContent = css;
    document.head.appendChild(style);
  })
  .catch((error) => console.error("CSS load failed:", error));
```

### Example 7: Request Batching and Deduplication

```typescript
// Intelligent request batching to reduce network calls
interface BatchConfig {
  maxBatchSize: number;
  maxWaitTime: number;
  endpoint: string;
}

class RequestBatcher<T = any, R = any> {
  private queue: Array<{
    data: T;
    resolve: (value: R) => void;
    reject: (error: Error) => void;
  }> = [];
  private timer: NodeJS.Timeout | null = null;
  private inFlightRequests: Map<string, Promise<R>> = new Map();
  private config: BatchConfig;

  constructor(config: BatchConfig) {
    this.config = config;
  }

  // Add request to batch
  async request(data: T): Promise<R> {
    // Check for duplicate in-flight request
    const requestKey = this.getRequestKey(data);
    if (this.inFlightRequests.has(requestKey)) {
      return this.inFlightRequests.get(requestKey)!;
    }

    return new Promise<R>((resolve, reject) => {
      this.queue.push({ data, resolve, reject });

      // Start timer if not already running
      if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), this.config.maxWaitTime);
      }

      // Flush if batch is full
      if (this.queue.length >= this.config.maxBatchSize) {
        this.flush();
      }
    });
  }

  // Flush pending requests
  private async flush(): Promise<void> {
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }

    if (this.queue.length === 0) {
      return;
    }

    const batch = this.queue.splice(0, this.config.maxBatchSize);
    const requestData = batch.map((item) => item.data);

    try {
      const response = await fetch(this.config.endpoint, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(requestData),
      });

      if (!response.ok) {
        throw new Error(`Batch request failed: ${response.statusText}`);
      }

      const results: R[] = await response.json();

      // Resolve individual promises
      batch.forEach((item, index) => {
        const requestKey = this.getRequestKey(item.data);
        this.inFlightRequests.delete(requestKey);
        item.resolve(results[index]);
      });
    } catch (error) {
      // Reject all promises in batch
      batch.forEach((item) => {
        const requestKey = this.getRequestKey(item.data);
        this.inFlightRequests.delete(requestKey);
        item.reject(error as Error);
      });
    }

    // Continue flushing if queue has more items
    if (this.queue.length > 0) {
      setTimeout(() => this.flush(), 0);
    }
  }

  private getRequestKey(data: T): string {
    return JSON.stringify(data);
  }

  // Manual flush
  async forceFlush(): Promise<void> {
    await this.flush();
  }
}

// Usage for analytics events
interface AnalyticsEvent {
  type: string;
  userId: string;
  timestamp: number;
  data: Record<string, any>;
}

const analyticsBatcher = new RequestBatcher<AnalyticsEvent, void>({
  maxBatchSize: 10,
  maxWaitTime: 5000, // 5 seconds
  endpoint: "/api/analytics/batch",
});

// Track events
function trackEvent(type: string, data: Record<string, any>): Promise<void> {
  return analyticsBatcher.request({
    type,
    userId: getCurrentUserId(),
    timestamp: Date.now(),
    data,
  });
}

// Example usage
trackEvent("page_view", { path: "/products" });
trackEvent("button_click", { buttonId: "checkout" });
trackEvent("form_submit", { formId: "contact" });

// Usage for GraphQL query batching
interface GraphQLQuery {
  query: string;
  variables?: Record<string, any>;
}

const graphqlBatcher = new RequestBatcher<GraphQLQuery, any>({
  maxBatchSize: 5,
  maxWaitTime: 100, // 100ms
  endpoint: "/graphql/batch",
});

async function executeQuery(
  query: string,
  variables?: Record<string, any>,
): Promise<any> {
  return graphqlBatcher.request({ query, variables });
}
```

### Example 8: Adaptive Loading Strategy

```typescript
// Adjust loading strategy based on network conditions
interface NetworkInfo {
  effectiveType: "slow-2g" | "2g" | "3g" | "4g";
  downlink: number; // Mbps
  rtt: number; // ms
  saveData: boolean;
}

class AdaptiveLoader {
  private networkInfo: NetworkInfo;
  private strategies: Map<string, LoadStrategy> = new Map();

  constructor() {
    this.networkInfo = this.getNetworkInfo();
    this.setupNetworkListeners();
  }

  private getNetworkInfo(): NetworkInfo {
    const connection =
      (navigator as any).connection ||
      (navigator as any).mozConnection ||
      (navigator as any).webkitConnection;

    if (!connection) {
      return {
        effectiveType: "4g",
        downlink: 10,
        rtt: 50,
        saveData: false,
      };
    }

    return {
      effectiveType: connection.effectiveType,
      downlink: connection.downlink,
      rtt: connection.rtt,
      saveData: connection.saveData || false,
    };
  }

  private setupNetworkListeners(): void {
    const connection = (navigator as any).connection;
    if (connection) {
      connection.addEventListener("change", () => {
        this.networkInfo = this.getNetworkInfo();
        console.log("Network changed:", this.networkInfo);
      });
    }
  }

  // Determine if we should load resource
  shouldLoad(resourceType: string, priority: "high" | "low"): boolean {
    if (this.networkInfo.saveData) {
      return priority === "high";
    }

    const qualityScore = this.getNetworkQuality();

    if (qualityScore >= 0.8) {
      return true; // Good network, load everything
    }

    if (qualityScore >= 0.5) {
      return priority === "high"; // Medium network, load high priority only
    }

    return resourceType === "critical"; // Poor network, critical only
  }

  private getNetworkQuality(): number {
    const typeScores = {
      "slow-2g": 0.1,
      "2g": 0.3,
      "3g": 0.6,
      "4g": 1.0,
    };

    const typeScore = typeScores[this.networkInfo.effectiveType];
    const rttScore = Math.max(0, 1 - this.networkInfo.rtt / 1000);
    const downlinkScore = Math.min(1, this.networkInfo.downlink / 10);

    return (typeScore + rttScore + downlinkScore) / 3;
  }

  // Get image quality based on network
  getImageQuality(): "low" | "medium" | "high" {
    const quality = this.getNetworkQuality();

    if (quality >= 0.8) return "high";
    if (quality >= 0.5) return "medium";
    return "low";
  }

  // Get video quality
  getVideoQuality(): { resolution: string; bitrate: number } {
    const quality = this.getNetworkQuality();

    if (quality >= 0.8) {
      return { resolution: "1080p", bitrate: 5000000 };
    } else if (quality >= 0.6) {
      return { resolution: "720p", bitrate: 2500000 };
    } else if (quality >= 0.4) {
      return { resolution: "480p", bitrate: 1000000 };
    } else {
      return { resolution: "360p", bitrate: 500000 };
    }
  }

  // Adaptive fetch with retry
  async adaptiveFetch(
    url: string,
    options: RequestInit = {},
  ): Promise<Response> {
    const quality = this.getNetworkQuality();
    const timeout = quality >= 0.6 ? 10000 : 5000;
    const maxRetries = quality >= 0.5 ? 3 : 1;

    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        const controller = new AbortController();
        const timeoutId = setTimeout(() => controller.abort(), timeout);

        const response = await fetch(url, {
          ...options,
          signal: controller.signal,
        });

        clearTimeout(timeoutId);
        return response;
      } catch (error) {
        if (attempt === maxRetries - 1) {
          throw error;
        }
        // Exponential backoff
        await new Promise((resolve) =>
          setTimeout(resolve, Math.pow(2, attempt) * 1000),
        );
      }
    }

    throw new Error("Max retries exceeded");
  }

  // Get network info for debugging
  getNetworkStatus(): NetworkInfo {
    return { ...this.networkInfo };
  }
}

// Usage
const adaptiveLoader = new AdaptiveLoader();

// Load images adaptively
function loadImage(src: string, alt: string): HTMLImageElement {
  const img = document.createElement("img");
  img.alt = alt;

  const quality = adaptiveLoader.getImageQuality();

  if (quality === "low") {
    img.src = src.replace(".jpg", "-small.jpg");
  } else if (quality === "medium") {
    img.src = src.replace(".jpg", "-medium.jpg");
  } else {
    img.src = src;
  }

  // Add loading attribute based on network
  if (adaptiveLoader.shouldLoad("image", "low")) {
    img.loading = "lazy";
  }

  return img;
}

// Load video adaptively
function setupVideoPlayer(videoElement: HTMLVideoElement): void {
  const { resolution, bitrate } = adaptiveLoader.getVideoQuality();

  // Set appropriate video source
  const source = videoElement.querySelector("source");
  if (source) {
    source.src = `/videos/video-${resolution}.mp4`;
  }

  // Configure player settings
  videoElement.preload = adaptiveLoader.shouldLoad("video", "low")
    ? "metadata"
    : "none";
}

// Conditional feature loading
async function loadEnhancedFeatures(): Promise<void> {
  if (adaptiveLoader.shouldLoad("feature", "low")) {
    const module = await import("./enhanced-features");
    module.initialize();
  }
}
```

### Example 9: Request Priority Queue

```typescript
// Priority-based request scheduling
interface QueuedRequest {
  url: string;
  options: RequestInit;
  priority: number;
  resolve: (response: Response) => void;
  reject: (error: Error) => void;
  timestamp: number;
}

class RequestPriorityQueue {
  private queue: QueuedRequest[] = [];
  private inFlight: number = 0;
  private readonly MAX_CONCURRENT = 6; // Browser limit
  private readonly TIMEOUT = 30000; // 30 seconds

  // Add request to queue
  async fetch(
    url: string,
    options: RequestInit = {},
    priority: number = 0,
  ): Promise<Response> {
    return new Promise<Response>((resolve, reject) => {
      this.queue.push({
        url,
        options,
        priority,
        resolve,
        reject,
        timestamp: Date.now(),
      });

      this.queue.sort((a, b) => b.priority - a.priority);
      this.processQueue();
    });
  }

  private async processQueue(): Promise<void> {
    while (this.queue.length > 0 && this.inFlight < this.MAX_CONCURRENT) {
      const request = this.queue.shift();
      if (!request) break;

      // Check if request has timed out while in queue
      if (Date.now() - request.timestamp > this.TIMEOUT) {
        request.reject(new Error("Request timeout in queue"));
        continue;
      }

      this.inFlight++;
      this.executeRequest(request);
    }
  }

  private async executeRequest(request: QueuedRequest): Promise<void> {
    try {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), this.TIMEOUT);

      const response = await fetch(request.url, {
        ...request.options,
        signal: controller.signal,
      });

      clearTimeout(timeoutId);
      request.resolve(response);
    } catch (error) {
      request.reject(error as Error);
    } finally {
      this.inFlight--;
      this.processQueue();
    }
  }

  // Get queue status
  getStatus(): { queued: number; inFlight: number } {
    return {
      queued: this.queue.length,
      inFlight: this.inFlight,
    };
  }

  // Clear queue
  clear(): void {
    this.queue.forEach((request) => {
      request.reject(new Error("Queue cleared"));
    });
    this.queue = [];
  }
}

// Usage
const requestQueue = new RequestPriorityQueue();

// High priority - critical API calls
async function fetchUserData(): Promise<any> {
  const response = await requestQueue.fetch(
    "/api/user/profile",
    { method: "GET" },
    10, // High priority
  );
  return response.json();
}

// Medium priority - important data
async function fetchProductData(id: string): Promise<any> {
  const response = await requestQueue.fetch(
    `/api/products/${id}`,
    { method: "GET" },
    5, // Medium priority
  );
  return response.json();
}

// Low priority - analytics, tracking
async function trackAnalytics(event: string): Promise<void> {
  await requestQueue.fetch(
    "/api/analytics",
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ event }),
    },
    1, // Low priority
  );
}

// Example: Load page data with prioritization
async function loadPageData(): Promise<void> {
  // Critical data loads first
  const userData = fetchUserData(); // Priority 10

  // Important data loads next
  const products = Promise.all([
    fetchProductData("1"), // Priority 5
    fetchProductData("2"),
    fetchProductData("3"),
  ]);

  // Analytics last
  trackAnalytics("page_view"); // Priority 1

  await Promise.all([userData, products]);
}
```

### Example 10: Connection Pooling for WebSockets

```typescript
// WebSocket connection pool with reconnection
interface WSPoolConfig {
  maxConnections: number;
  reconnectInterval: number;
  maxReconnectAttempts: number;
  heartbeatInterval: number;
}

class WebSocketPool {
  private connections: Map<string, WebSocket> = new Map();
  private connectionQueues: Map<string, Function[]> = new Map();
  private reconnectAttempts: Map<string, number> = new Map();
  private config: WSPoolConfig;

  constructor(config: Partial<WSPoolConfig> = {}) {
    this.config = {
      maxConnections: config.maxConnections ?? 10,
      reconnectInterval: config.reconnectInterval ?? 5000,
      maxReconnectAttempts: config.maxReconnectAttempts ?? 5,
      heartbeatInterval: config.heartbeatInterval ?? 30000,
    };
  }

  // Get or create connection
  async getConnection(url: string): Promise<WebSocket> {
    if (this.connections.has(url)) {
      const ws = this.connections.get(url)!;
      if (ws.readyState === WebSocket.OPEN) {
        return ws;
      }
    }

    return new Promise((resolve, reject) => {
      if (!this.connectionQueues.has(url)) {
        this.connectionQueues.set(url, []);
        this.createConnection(url);
      }

      this.connectionQueues.get(url)!.push((ws: WebSocket) => {
        if (ws) {
          resolve(ws);
        } else {
          reject(new Error("Failed to create connection"));
        }
      });
    });
  }

  private createConnection(url: string): void {
    try {
      const ws = new WebSocket(url);

      ws.addEventListener("open", () => {
        console.log("WebSocket connected:", url);
        this.connections.set(url, ws);
        this.reconnectAttempts.set(url, 0);
        this.setupHeartbeat(url, ws);
        this.resolveQueue(url, ws);
      });

      ws.addEventListener("close", () => {
        console.log("WebSocket closed:", url);
        this.connections.delete(url);
        this.attemptReconnect(url);
      });

      ws.addEventListener("error", (error) => {
        console.error("WebSocket error:", error);
        this.rejectQueue(url);
      });
    } catch (error) {
      console.error("Failed to create WebSocket:", error);
      this.rejectQueue(url);
    }
  }

  private setupHeartbeat(url: string, ws: WebSocket): void {
    const interval = setInterval(() => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ type: "ping" }));
      } else {
        clearInterval(interval);
      }
    }, this.config.heartbeatInterval);
  }

  private attemptReconnect(url: string): void {
    const attempts = this.reconnectAttempts.get(url) || 0;

    if (attempts < this.config.maxReconnectAttempts) {
      this.reconnectAttempts.set(url, attempts + 1);

      console.log(
        `Reconnecting (${attempts + 1}/${this.config.maxReconnectAttempts})...`,
      );

      setTimeout(
        () => {
          this.createConnection(url);
        },
        this.config.reconnectInterval * Math.pow(2, attempts),
      );
    } else {
      console.error("Max reconnect attempts reached");
      this.reconnectAttempts.delete(url);
    }
  }

  private resolveQueue(url: string, ws: WebSocket): void {
    const queue = this.connectionQueues.get(url) || [];
    queue.forEach((callback) => callback(ws));
    this.connectionQueues.delete(url);
  }

  private rejectQueue(url: string): void {
    const queue = this.connectionQueues.get(url) || [];
    queue.forEach((callback) => callback(null));
    this.connectionQueues.delete(url);
  }

  // Send message with automatic connection handling
  async send(url: string, data: any): Promise<void> {
    const ws = await this.getConnection(url);

    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(data));
    } else {
      throw new Error("WebSocket not open");
    }
  }

  // Close specific connection
  close(url: string): void {
    const ws = this.connections.get(url);
    if (ws) {
      ws.close();
      this.connections.delete(url);
    }
  }

  // Close all connections
  closeAll(): void {
    this.connections.forEach((ws, url) => {
      ws.close();
    });
    this.connections.clear();
  }
}

// Usage
const wsPool = new WebSocketPool({
  maxConnections: 5,
  reconnectInterval: 3000,
  maxReconnectAttempts: 10,
  heartbeatInterval: 30000,
});

// Real-time chat application
class ChatClient {
  private wsUrl = "wss://chat.example.com";

  async sendMessage(message: string): Promise<void> {
    await wsPool.send(this.wsUrl, {
      type: "message",
      content: message,
      timestamp: Date.now(),
    });
  }

  async subscribeToChannel(channel: string): Promise<void> {
    const ws = await wsPool.getConnection(this.wsUrl);

    ws.addEventListener("message", (event) => {
      const data = JSON.parse(event.data);
      if (data.channel === channel) {
        this.handleMessage(data);
      }
    });

    await wsPool.send(this.wsUrl, {
      type: "subscribe",
      channel,
    });
  }

  private handleMessage(data: any): void {
    console.log("Received message:", data);
    // Handle incoming message
  }
}
```

## Real-World Usage

### E-Commerce Platform

```typescript
// Complete network optimization for e-commerce
class EcommerceNetworkOptimizer {
  private cdnManager: CDNManager;
  private cacheManager: CacheHeaderManager;
  private hintManager: ResourceHintManager;
  private adaptiveLoader: AdaptiveLoader;
  private requestQueue: RequestPriorityQueue;

  constructor() {
    this.setupCDN();
    this.setupCaching();
    this.setupResourceHints();
    this.adaptiveLoader = new AdaptiveLoader();
    this.requestQueue = new RequestPriorityQueue();
  }

  private setupCDN(): void {
    this.cdnManager = new CDNManager([
      {
        name: "Cloudflare",
        baseUrl: "https://cdn.example.com",
        priority: 1,
        timeout: 3000,
        healthCheck: "/health",
      },
      {
        name: "AWS CloudFront",
        baseUrl: "https://d1234.cloudfront.net",
        priority: 2,
        timeout: 3000,
      },
    ]);
  }

  private setupCaching(): void {
    this.cacheManager = new CacheHeaderManager();

    // Product images - long cache
    this.cacheManager.defineStrategy(/\/products\/.*\.(webp|jpg)/, {
      maxAge: 2592000, // 30 days
      sMaxAge: 31536000, // 1 year on CDN
      immutable: true,
    });

    // Product API - short cache with revalidation
    this.cacheManager.defineStrategy(/\/api\/products/, {
      maxAge: 300, // 5 minutes
      staleWhileRevalidate: 3600,
      mustRevalidate: true,
    });

    // Cart API - no cache
    this.cacheManager.defineStrategy(/\/api\/cart/, {
      maxAge: 0,
      mustRevalidate: true,
      private: true,
    });
  }

  private setupResourceHints(): void {
    this.hintManager = new ResourceHintManager();

    // Critical resources
    this.hintManager.addPreload({
      href: "/fonts/product-sans.woff2",
      as: "font",
      type: "font/woff2",
      crossOrigin: "anonymous",
      importance: "high",
    });

    this.hintManager.addPreconnect({
      href: "https://api.example.com",
      crossOrigin: "anonymous",
    });

    // Prefetch likely next pages
    this.hintManager.addPrefetch({
      href: "/checkout",
    });
  }

  // Load product page with optimization
  async loadProductPage(productId: string): Promise<void> {
    // Priority 10 - Critical product data
    const productPromise = this.requestQueue.fetch(
      `/api/products/${productId}`,
      { method: "GET" },
      10,
    );

    // Priority 8 - Product images
    const imagesPromise = this.requestQueue.fetch(
      `/api/products/${productId}/images`,
      { method: "GET" },
      8,
    );

    // Priority 5 - Related products
    const relatedPromise = this.requestQueue.fetch(
      `/api/products/${productId}/related`,
      { method: "GET" },
      5,
    );

    // Priority 2 - Reviews (can load later)
    const reviewsPromise = this.requestQueue.fetch(
      `/api/products/${productId}/reviews`,
      { method: "GET" },
      2,
    );

    // Wait for critical data
    const [product, images] = await Promise.all([
      productPromise.then((r) => r.json()),
      imagesPromise.then((r) => r.json()),
    ]);

    this.renderProduct(product, images);

    // Load non-critical data
    Promise.all([relatedPromise, reviewsPromise]).then(([related, reviews]) => {
      this.renderRelated(related);
      this.renderReviews(reviews);
    });
  }

  private renderProduct(product: any, images: any): void {
    // Render product with adaptive image loading
    const quality = this.adaptiveLoader.getImageQuality();
    // Implementation...
  }

  private renderRelated(related: any): void {
    // Render related products
  }

  private renderReviews(reviews: any): void {
    // Render reviews
  }
}
```

### Real-Time Dashboard

```typescript
// Network optimization for real-time data dashboard
class DashboardNetworkManager {
  private wsPool: WebSocketPool;
  private requestBatcher: RequestBatcher<any, any>;
  private updateInterval: number = 5000;

  constructor() {
    this.wsPool = new WebSocketPool({
      maxConnections: 3,
      heartbeatInterval: 15000,
    });

    this.requestBatcher = new RequestBatcher({
      maxBatchSize: 20,
      maxWaitTime: 1000,
      endpoint: "/api/metrics/batch",
    });

    this.setupWebSocketStreams();
  }

  private async setupWebSocketStreams(): Promise<void> {
    // Connect to real-time metrics stream
    const metricsWS = await this.wsPool.getConnection(
      "wss://api.example.com/metrics",
    );

    metricsWS.addEventListener("message", (event) => {
      const data = JSON.parse(event.data);
      this.handleMetricUpdate(data);
    });

    // Connect to alerts stream
    const alertsWS = await this.wsPool.getConnection(
      "wss://api.example.com/alerts",
    );

    alertsWS.addEventListener("message", (event) => {
      const alert = JSON.parse(event.data);
      this.handleAlert(alert);
    });
  }

  async loadHistoricalData(
    metricIds: string[],
    timeRange: { start: number; end: number },
  ): Promise<void> {
    // Batch requests for historical data
    const promises = metricIds.map((id) =>
      this.requestBatcher.request({
        metricId: id,
        timeRange,
      }),
    );

    const results = await Promise.all(promises);
    this.renderHistoricalData(results);
  }

  private handleMetricUpdate(data: any): void {
    // Update dashboard with real-time data
    console.log("Metric update:", data);
  }

  private handleAlert(alert: any): void {
    // Show alert notification
    console.log("Alert:", alert);
  }

  private renderHistoricalData(data: any[]): void {
    // Render charts and graphs
  }
}
```

## Production Patterns

### 1. **Progressive Enhancement Pattern**

```typescript
// Load features progressively based on network
async function initializeApp(): Promise<void> {
  const adaptiveLoader = new AdaptiveLoader();

  // Always load core features
  await loadCoreFeatures();

  // Load enhanced features based on network
  if (adaptiveLoader.shouldLoad("feature", "low")) {
    await loadEnhancedFeatures();
  }

  if (adaptiveLoader.shouldLoad("feature", "low")) {
    await loadAnalytics();
  }
}
```

### 2. **Graceful Degradation Pattern**

```typescript
// Fallback chain for resource loading
async function loadResource(url: string): Promise<Response> {
  try {
    return await cdnManager.fetchResource(url);
  } catch (cdnError) {
    console.warn("CDN failed, trying origin");
    try {
      return await fetch(url);
    } catch (originError) {
      // Return cached version if available
      const cached = await caches.match(url);
      if (cached) return cached;
      throw originError;
    }
  }
}
```

### 3. **Request Coalescing Pattern**

```typescript
// Prevent duplicate simultaneous requests
const pendingRequests = new Map<string, Promise<any>>();

async function fetchWithCoalescing(url: string): Promise<any> {
  if (pendingRequests.has(url)) {
    return pendingRequests.get(url);
  }

  const promise = fetch(url)
    .then((r) => r.json())
    .finally(() => pendingRequests.delete(url));

  pendingRequests.set(url, promise);
  return promise;
}
```

## Best Practices

1. **Enable HTTP/2 or HTTP/3** for multiplexing and reduced latency
2. **Use Brotli compression** for text resources (better than gzip)
3. **Implement smart caching** with appropriate Cache-Control headers
4. **Use resource hints** (preload, prefetch, preconnect) strategically
5. **Deploy service workers** for offline support and intelligent caching
6. **Leverage CDN** with multiple providers for redundancy
7. **Batch API requests** to reduce network overhead
8. **Implement adaptive loading** based on network conditions
9. **Use request prioritization** for critical resources
10. **Monitor and optimize** Time to First Byte (TTFB)
11. **Implement connection pooling** for WebSocket applications
12. **Use ETags** for efficient cache revalidation
13. **Enable keep-alive** for persistent connections
14. **Minimize redirects** and DNS lookups
15. **Use modern image formats** (WebP, AVIF) with proper fallbacks

## 10 Key Takeaways

1. **HTTP/2 multiplexing** eliminates head-of-line blocking and enables parallel resource loading over a single connection
2. **Brotli compression** provides 15-20% better compression than gzip for text-based resources
3. **Cache-Control headers** with stale-while-revalidate provide instant responses while updating in background
4. **Resource hints** (preload/prefetch/preconnect) reduce latency by starting connections and downloads early
5. **Service workers** enable offline functionality and sophisticated caching strategies for PWAs
6. **CDN failover** with health checks ensures high availability and optimal performance across regions
7. **Request batching** reduces network overhead by combining multiple small requests into single calls
8. **Adaptive loading** optimizes user experience by adjusting content quality based on network conditions
9. **Request prioritization** ensures critical resources load first, improving perceived performance
10. **ETag validation** minimizes bandwidth usage by sending only changed resources to clients
