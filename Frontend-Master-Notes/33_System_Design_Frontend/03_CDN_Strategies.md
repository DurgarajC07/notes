# CDN Strategies - Complete Guide

## Core Concepts

A Content Delivery Network (CDN) is a geographically distributed network of servers that delivers web content to users based on their location, ensuring low latency and high availability.

### Key Benefits

- **Reduced Latency**: Content served from nearest edge server
- **High Availability**: Redundancy across multiple servers
- **DDoS Protection**: Distributed architecture absorbs attacks
- **Bandwidth Optimization**: Offload origin server traffic
- **Global Reach**: Serve users worldwide efficiently

## 1. Edge Caching

```typescript
// CDN Configuration Manager
class CDNManager {
  private config: CDNConfig;

  constructor(config: CDNConfig) {
    this.config = config;
  }

  // Generate CDN URL with cache control
  getCDNUrl(asset: string, options: CacheOptions = {}): string {
    const { version = "latest", cacheBust = false, queryParams = {} } = options;

    let url = `${this.config.cdnBaseUrl}/${asset}`;

    // Add version for cache busting
    if (version !== "latest") {
      url = `${this.config.cdnBaseUrl}/v${version}/${asset}`;
    }

    // Add cache bust timestamp
    if (cacheBust) {
      queryParams.t = Date.now().toString();
    }

    // Add query parameters
    const params = new URLSearchParams(queryParams);
    if (params.toString()) {
      url += `?${params.toString()}`;
    }

    return url;
  }

  // Set cache headers for assets
  getCacheHeaders(assetType: AssetType): Record<string, string> {
    const headers: Record<string, string> = {};

    switch (assetType) {
      case "static":
        // Static assets (fonts, icons): 1 year
        headers["Cache-Control"] = "public, max-age=31536000, immutable";
        break;

      case "versioned":
        // Versioned JS/CSS: 1 year with immutable
        headers["Cache-Control"] = "public, max-age=31536000, immutable";
        break;

      case "image":
        // Images: 30 days
        headers["Cache-Control"] = "public, max-age=2592000";
        break;

      case "api":
        // API responses: 5 minutes, must revalidate
        headers["Cache-Control"] = "public, max-age=300, must-revalidate";
        break;

      case "dynamic":
        // Dynamic content: no cache
        headers["Cache-Control"] = "no-cache, no-store, must-revalidate";
        headers["Pragma"] = "no-cache";
        headers["Expires"] = "0";
        break;
    }

    return headers;
  }

  // Generate ETags for conditional requests
  generateETag(content: string): string {
    // Simple hash function for demo (use crypto.subtle.digest in production)
    let hash = 0;
    for (let i = 0; i < content.length; i++) {
      const char = content.charCodeAt(i);
      hash = (hash << 5) - hash + char;
      hash = hash & hash;
    }
    return `"${Math.abs(hash).toString(36)}"`;
  }
}

interface CDNConfig {
  cdnBaseUrl: string;
  fallbackUrl: string;
  regions: string[];
}

interface CacheOptions {
  version?: string;
  cacheBust?: boolean;
  queryParams?: Record<string, string>;
}

type AssetType = "static" | "versioned" | "image" | "api" | "dynamic";

// Usage
const cdnManager = new CDNManager({
  cdnBaseUrl: "https://cdn.example.com",
  fallbackUrl: "https://www.example.com",
  regions: ["us-east", "us-west", "eu-west", "ap-southeast"],
});

// Get CDN URL for assets
const jsUrl = cdnManager.getCDNUrl("app.js", {
  version: "1.2.3",
});

const imageUrl = cdnManager.getCDNUrl("images/logo.png", {
  cacheBust: true,
});
```

## 2. Geo-Distribution

```typescript
// Geo-Location Based CDN Selector
class GeoCDNSelector {
  private cdnEndpoints: CDNEndpoint[];

  constructor(endpoints: CDNEndpoint[]) {
    this.cdnEndpoints = endpoints;
  }

  async getUserLocation(): Promise<UserLocation> {
    try {
      // Try geolocation API
      if ("geolocation" in navigator) {
        const position = await new Promise<GeolocationPosition>(
          (resolve, reject) => {
            navigator.geolocation.getCurrentPosition(resolve, reject, {
              timeout: 5000,
            });
          },
        );

        return {
          latitude: position.coords.latitude,
          longitude: position.coords.longitude,
          accuracy: position.coords.accuracy,
        };
      }

      // Fallback to IP-based geolocation
      const response = await fetch("https://ipapi.co/json/");
      const data = await response.json();

      return {
        latitude: data.latitude,
        longitude: data.longitude,
        country: data.country_code,
        region: data.region,
      };
    } catch (error) {
      // Default to primary CDN
      return {
        latitude: 0,
        longitude: 0,
      };
    }
  }

  async getNearestCDN(): Promise<string> {
    const userLocation = await this.getUserLocation();

    // Calculate distances to all CDN endpoints
    const distances = this.cdnEndpoints.map((endpoint) => ({
      url: endpoint.url,
      distance: this.calculateDistance(
        userLocation.latitude,
        userLocation.longitude,
        endpoint.latitude,
        endpoint.longitude,
      ),
      priority: endpoint.priority,
    }));

    // Sort by distance and priority
    distances.sort((a, b) => {
      if (a.priority !== b.priority) {
        return a.priority - b.priority; // Lower priority first
      }
      return a.distance - b.distance;
    });

    return distances[0].url;
  }

  private calculateDistance(
    lat1: number,
    lon1: number,
    lat2: number,
    lon2: number,
  ): number {
    // Haversine formula
    const R = 6371; // Earth's radius in km
    const dLat = this.toRad(lat2 - lat1);
    const dLon = this.toRad(lon2 - lon1);

    const a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(this.toRad(lat1)) *
        Math.cos(this.toRad(lat2)) *
        Math.sin(dLon / 2) *
        Math.sin(dLon / 2);

    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return R * c;
  }

  private toRad(degrees: number): number {
    return degrees * (Math.PI / 180);
  }
}

interface CDNEndpoint {
  url: string;
  latitude: number;
  longitude: number;
  priority: number;
  region: string;
}

interface UserLocation {
  latitude: number;
  longitude: number;
  accuracy?: number;
  country?: string;
  region?: string;
}

// Usage
const cdnSelector = new GeoCDNSelector([
  {
    url: "https://cdn-us-east.example.com",
    latitude: 40.7128,
    longitude: -74.006,
    priority: 1,
    region: "us-east",
  },
  {
    url: "https://cdn-us-west.example.com",
    latitude: 37.7749,
    longitude: -122.4194,
    priority: 1,
    region: "us-west",
  },
  {
    url: "https://cdn-eu-west.example.com",
    latitude: 51.5074,
    longitude: -0.1278,
    priority: 1,
    region: "eu-west",
  },
]);

const nearestCDN = await cdnSelector.getNearestCDN();
console.log("Using CDN:", nearestCDN);
```

## 3. Origin Shield

```typescript
// Origin Shield Implementation
class OriginShield {
  private shieldServers: Map<string, ShieldServer> = new Map();
  private originUrl: string;

  constructor(originUrl: string, shieldConfigs: ShieldServerConfig[]) {
    this.originUrl = originUrl;

    shieldConfigs.forEach((config) => {
      this.shieldServers.set(config.region, {
        url: config.url,
        region: config.region,
        healthStatus: "healthy",
        lastCheck: Date.now(),
      });
    });

    this.startHealthChecks();
  }

  async fetchThroughShield(path: string, region: string): Promise<Response> {
    const shieldServer = this.shieldServers.get(region);

    if (!shieldServer || shieldServer.healthStatus !== "healthy") {
      // Fallback to origin
      return this.fetchFromOrigin(path);
    }

    try {
      // Try to fetch from shield
      const response = await fetch(`${shieldServer.url}${path}`, {
        headers: {
          "X-Shield-Region": region,
          "X-Origin-Url": this.originUrl,
        },
      });

      if (response.ok) {
        return response;
      }

      // Shield failed, fetch from origin
      return this.fetchFromOrigin(path);
    } catch (error) {
      console.error("Shield fetch failed:", error);
      return this.fetchFromOrigin(path);
    }
  }

  private async fetchFromOrigin(path: string): Promise<Response> {
    return fetch(`${this.originUrl}${path}`);
  }

  private startHealthChecks(): void {
    setInterval(() => {
      this.shieldServers.forEach(async (server, region) => {
        try {
          const response = await fetch(`${server.url}/health`, {
            method: "HEAD",
            timeout: 5000,
          } as any);

          server.healthStatus = response.ok ? "healthy" : "unhealthy";
          server.lastCheck = Date.now();
        } catch (error) {
          server.healthStatus = "unhealthy";
          server.lastCheck = Date.now();
        }
      });
    }, 30000); // Check every 30 seconds
  }

  getShieldStatus(region: string): ShieldServer | undefined {
    return this.shieldServers.get(region);
  }
}

interface ShieldServerConfig {
  url: string;
  region: string;
}

interface ShieldServer extends ShieldServerConfig {
  healthStatus: "healthy" | "unhealthy";
  lastCheck: number;
}

// Usage
const originShield = new OriginShield("https://origin.example.com", [
  {
    url: "https://shield-us-east.example.com",
    region: "us-east",
  },
  {
    url: "https://shield-eu-west.example.com",
    region: "eu-west",
  },
]);

const response = await originShield.fetchThroughShield("/api/data", "us-east");
```

## 4. Cache Invalidation

```typescript
// Cache Invalidation Manager
class CacheInvalidationManager {
  private cdnProviders: CDNProvider[];

  constructor(providers: CDNProvider[]) {
    this.cdnProviders = providers;
  }

  // Purge specific URLs
  async purgeUrls(urls: string[]): Promise<PurgeResult> {
    const results: PurgeResult = {
      successful: [],
      failed: [],
    };

    for (const provider of this.cdnProviders) {
      try {
        await this.purgeFromProvider(provider, urls);
        results.successful.push({
          provider: provider.name,
          urls,
        });
      } catch (error) {
        results.failed.push({
          provider: provider.name,
          urls,
          error: (error as Error).message,
        });
      }
    }

    return results;
  }

  // Purge by cache tag
  async purgeByTag(tag: string): Promise<PurgeResult> {
    const results: PurgeResult = {
      successful: [],
      failed: [],
    };

    for (const provider of this.cdnProviders) {
      try {
        await fetch(`${provider.apiUrl}/purge/tag`, {
          method: "POST",
          headers: {
            Authorization: `Bearer ${provider.apiKey}`,
            "Content-Type": "application/json",
          },
          body: JSON.stringify({ tag }),
        });

        results.successful.push({
          provider: provider.name,
          urls: [tag],
        });
      } catch (error) {
        results.failed.push({
          provider: provider.name,
          urls: [tag],
          error: (error as Error).message,
        });
      }
    }

    return results;
  }

  // Purge entire cache
  async purgeAll(): Promise<PurgeResult> {
    const results: PurgeResult = {
      successful: [],
      failed: [],
    };

    for (const provider of this.cdnProviders) {
      try {
        await fetch(`${provider.apiUrl}/purge/all`, {
          method: "POST",
          headers: {
            Authorization: `Bearer ${provider.apiKey}`,
          },
        });

        results.successful.push({
          provider: provider.name,
          urls: ["*"],
        });
      } catch (error) {
        results.failed.push({
          provider: provider.name,
          urls: ["*"],
          error: (error as Error).message,
        });
      }
    }

    return results;
  }

  private async purgeFromProvider(
    provider: CDNProvider,
    urls: string[],
  ): Promise<void> {
    const response = await fetch(`${provider.apiUrl}/purge/urls`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${provider.apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ urls }),
    });

    if (!response.ok) {
      throw new Error(`Purge failed: ${response.statusText}`);
    }
  }

  // Smart invalidation with versioning
  async invalidateWithVersion(
    asset: string,
    oldVersion: string,
    newVersion: string,
  ): Promise<void> {
    const oldUrl = this.getVersionedUrl(asset, oldVersion);
    const newUrl = this.getVersionedUrl(asset, newVersion);

    // Purge old version
    await this.purgeUrls([oldUrl]);

    // Warm cache for new version
    await this.warmCache([newUrl]);
  }

  private getVersionedUrl(asset: string, version: string): string {
    return `https://cdn.example.com/v${version}/${asset}`;
  }

  private async warmCache(urls: string[]): Promise<void> {
    // Pre-fetch URLs to warm CDN cache
    const promises = urls.map((url) =>
      fetch(url, { method: "HEAD" }).catch(() => {}),
    );

    await Promise.all(promises);
  }
}

interface CDNProvider {
  name: string;
  apiUrl: string;
  apiKey: string;
}

interface PurgeResult {
  successful: Array<{
    provider: string;
    urls: string[];
  }>;
  failed: Array<{
    provider: string;
    urls: string[];
    error: string;
  }>;
}

// Usage
const cacheManager = new CacheInvalidationManager([
  {
    name: "Cloudflare",
    apiUrl: "https://api.cloudflare.com/client/v4/zones/ZONE_ID",
    apiKey: "your-api-key",
  },
  {
    name: "AWS CloudFront",
    apiUrl: "https://cloudfront.amazonaws.com",
    apiKey: "your-api-key",
  },
]);

// Purge specific URLs
await cacheManager.purgeUrls([
  "https://cdn.example.com/js/app.js",
  "https://cdn.example.com/css/styles.css",
]);

// Purge by tag
await cacheManager.purgeByTag("product-images");

// Smart version-based invalidation
await cacheManager.invalidateWithVersion("app.js", "1.0.0", "1.0.1");
```

## 5. Cache Warming

```typescript
// Cache Warming Strategy
class CacheWarmingManager {
  private urls: string[];
  private workers: number;

  constructor(urls: string[], workers: number = 5) {
    this.urls = urls;
    this.workers = workers;
  }

  async warmCache(): Promise<WarmingResult> {
    const results: WarmingResult = {
      total: this.urls.length,
      successful: 0,
      failed: 0,
      duration: 0,
    };

    const startTime = Date.now();

    // Create worker pool
    const chunks = this.chunkArray(this.urls, this.workers);

    const promises = chunks.map((chunk) => this.warmChunk(chunk, results));

    await Promise.all(promises);

    results.duration = Date.now() - startTime;

    return results;
  }

  private async warmChunk(
    urls: string[],
    results: WarmingResult,
  ): Promise<void> {
    for (const url of urls) {
      try {
        const response = await fetch(url, {
          method: "HEAD",
          headers: {
            "X-Cache-Warming": "true",
          },
        });

        if (response.ok) {
          results.successful++;
        } else {
          results.failed++;
        }
      } catch (error) {
        results.failed++;
        console.error(`Failed to warm ${url}:`, error);
      }
    }
  }

  private chunkArray<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }

  // Progressive cache warming
  async progressiveWarm(
    priority: "high" | "medium" | "low",
    onProgress?: (progress: number) => void,
  ): Promise<void> {
    const priorityUrls = this.urls.filter(
      (url) => this.getUrlPriority(url) === priority,
    );

    let completed = 0;

    for (const url of priorityUrls) {
      await fetch(url, { method: "HEAD" });
      completed++;

      if (onProgress) {
        onProgress((completed / priorityUrls.length) * 100);
      }
    }
  }

  private getUrlPriority(url: string): "high" | "medium" | "low" {
    // Determine priority based on URL patterns
    if (url.includes("/critical/") || url.includes("app.js")) {
      return "high";
    }
    if (url.includes("/images/")) {
      return "medium";
    }
    return "low";
  }
}

interface WarmingResult {
  total: number;
  successful: number;
  failed: number;
  duration: number;
}

// Usage
const warmer = new CacheWarmingManager(
  [
    "https://cdn.example.com/js/app.js",
    "https://cdn.example.com/css/styles.css",
    "https://cdn.example.com/images/hero.jpg",
    // ... more URLs
  ],
  10,
);

const result = await warmer.warmCache();
console.log(
  `Warmed ${result.successful}/${result.total} URLs in ${result.duration}ms`,
);

// Progressive warming with priority
await warmer.progressiveWarm("high", (progress) => {
  console.log(`High priority: ${progress}% complete`);
});
```

## 6. Multi-CDN Strategies

```typescript
// Multi-CDN Manager
class MultiCDNManager {
  private cdns: CDNConfiguration[];
  private failoverAttempts: number = 3;

  constructor(cdns: CDNConfiguration[]) {
    this.cdns = cdns.sort((a, b) => a.priority - b.priority);
  }

  async fetchAsset(path: string): Promise<Response> {
    for (const cdn of this.cdns) {
      if (!cdn.enabled) continue;

      for (let attempt = 0; attempt < this.failoverAttempts; attempt++) {
        try {
          const url = `${cdn.baseUrl}${path}`;
          const response = await this.fetchWithTimeout(url, cdn.timeout);

          if (response.ok) {
            // Update CDN performance metrics
            this.updateMetrics(cdn.name, "success");
            return response;
          }
        } catch (error) {
          console.warn(
            `CDN ${cdn.name} failed (attempt ${attempt + 1}):`,
            error,
          );
          this.updateMetrics(cdn.name, "failure");
        }
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
      return response;
    } finally {
      clearTimeout(timeoutId);
    }
  }

  private updateMetrics(cdnName: string, result: "success" | "failure"): void {
    const cdn = this.cdns.find((c) => c.name === cdnName);

    if (!cdn) return;

    if (result === "success") {
      cdn.successCount++;
    } else {
      cdn.failureCount++;
    }

    // Disable CDN if failure rate is too high
    const failureRate =
      cdn.failureCount / (cdn.successCount + cdn.failureCount);

    if (failureRate > 0.5 && cdn.successCount + cdn.failureCount > 10) {
      cdn.enabled = false;
      console.warn(`Disabled CDN ${cdnName} due to high failure rate`);
    }
  }

  // Get CDN performance report
  getPerformanceReport(): CDNPerformanceReport[] {
    return this.cdns.map((cdn) => ({
      name: cdn.name,
      successCount: cdn.successCount,
      failureCount: cdn.failureCount,
      successRate:
        (cdn.successCount / (cdn.successCount + cdn.failureCount)) * 100 || 0,
      enabled: cdn.enabled,
    }));
  }

  // Enable/disable specific CDN
  setCDNStatus(name: string, enabled: boolean): void {
    const cdn = this.cdns.find((c) => c.name === name);
    if (cdn) {
      cdn.enabled = enabled;
    }
  }
}

interface CDNConfiguration {
  name: string;
  baseUrl: string;
  priority: number;
  timeout: number;
  enabled: boolean;
  successCount: number;
  failureCount: number;
}

interface CDNPerformanceReport {
  name: string;
  successCount: number;
  failureCount: number;
  successRate: number;
  enabled: boolean;
}

// Usage
const multiCDN = new MultiCDNManager([
  {
    name: "Cloudflare",
    baseUrl: "https://cdn-cf.example.com",
    priority: 1,
    timeout: 5000,
    enabled: true,
    successCount: 0,
    failureCount: 0,
  },
  {
    name: "AWS CloudFront",
    baseUrl: "https://cdn-aws.example.com",
    priority: 2,
    timeout: 5000,
    enabled: true,
    successCount: 0,
    failureCount: 0,
  },
  {
    name: "Fastly",
    baseUrl: "https://cdn-fastly.example.com",
    priority: 3,
    timeout: 5000,
    enabled: true,
    successCount: 0,
    failureCount: 0,
  },
]);

// Fetch asset with automatic failover
const response = await multiCDN.fetchAsset("/js/app.js");

// Get performance report
const report = multiCDN.getPerformanceReport();
console.log("CDN Performance:", report);
```

## 7. Content Optimization

```typescript
// Content Optimization Manager
class ContentOptimizer {
  // Image optimization
  optimizeImage(imageUrl: string, options: ImageOptimizationOptions): string {
    const params = new URLSearchParams();

    if (options.width) {
      params.append("w", options.width.toString());
    }

    if (options.height) {
      params.append("h", options.height.toString());
    }

    if (options.quality) {
      params.append("q", options.quality.toString());
    }

    if (options.format) {
      params.append("f", options.format);
    }

    if (options.dpr) {
      params.append("dpr", options.dpr.toString());
    }

    return `${imageUrl}?${params.toString()}`;
  }

  // Responsive image srcset
  generateSrcSet(
    imageUrl: string,
    widths: number[],
    format?: ImageFormat,
  ): string {
    return widths
      .map((width) => {
        const optimizedUrl = this.optimizeImage(imageUrl, {
          width,
          format,
          quality: 80,
        });
        return `${optimizedUrl} ${width}w`;
      })
      .join(", ");
  }

  // WebP support detection and fallback
  async supportsWebP(): Promise<boolean> {
    return new Promise((resolve) => {
      const webP = new Image();
      webP.onload = webP.onerror = () => {
        resolve(webP.height === 2);
      };
      webP.src =
        "data:image/webp;base64,UklGRjoAAABXRUJQVlA4IC4AAACyAgCdASoCAAIALmk0mk0iIiIiIgBoSygABc6WWgAA/veff/0PP8bA//LwYAAA";
    });
  }

  async getOptimalImageFormat(): Promise<ImageFormat> {
    if (await this.supportsWebP()) {
      return "webp";
    }
    return "jpg";
  }

  // CSS optimization
  optimizeCSS(css: string): string {
    return css
      .replace(/\/\*[\s\S]*?\*\//g, "") // Remove comments
      .replace(/\s+/g, " ") // Collapse whitespace
      .replace(/\s*([{}:;,])\s*/g, "$1") // Remove spaces around punctuation
      .trim();
  }

  // JavaScript optimization
  async optimizeJavaScript(code: string): Promise<string> {
    // In production, use a proper minifier like Terser
    // This is a simplified example
    return code
      .replace(/\/\/.*/g, "") // Remove single-line comments
      .replace(/\/\*[\s\S]*?\*\//g, "") // Remove multi-line comments
      .replace(/\s+/g, " ") // Collapse whitespace
      .trim();
  }

  // Resource hints
  generateResourceHints(resources: ResourceHint[]): string {
    return resources
      .map((resource) => {
        const attrs = [`rel="${resource.type}"`];

        if (resource.href) {
          attrs.push(`href="${resource.href}"`);
        }

        if (resource.as) {
          attrs.push(`as="${resource.as}"`);
        }

        if (resource.type === "preconnect" && resource.crossorigin) {
          attrs.push("crossorigin");
        }

        return `<link ${attrs.join(" ")} />`;
      })
      .join("\n");
  }
}

interface ImageOptimizationOptions {
  width?: number;
  height?: number;
  quality?: number;
  format?: ImageFormat;
  dpr?: number;
}

type ImageFormat = "webp" | "jpg" | "png" | "avif";

interface ResourceHint {
  type: "preconnect" | "dns-prefetch" | "preload" | "prefetch";
  href?: string;
  as?: string;
  crossorigin?: boolean;
}

// Usage
const optimizer = new ContentOptimizer();

// Optimize image
const optimizedImage = optimizer.optimizeImage(
  "https://cdn.example.com/image.jpg",
  {
    width: 800,
    quality: 80,
    format: "webp",
    dpr: 2,
  },
);

// Generate responsive srcset
const srcSet = optimizer.generateSrcSet(
  "https://cdn.example.com/image.jpg",
  [400, 800, 1200, 1600],
  "webp",
);

// Generate resource hints
const hints = optimizer.generateResourceHints([
  {
    type: "preconnect",
    href: "https://cdn.example.com",
    crossorigin: true,
  },
  {
    type: "dns-prefetch",
    href: "https://api.example.com",
  },
  {
    type: "preload",
    href: "/fonts/main.woff2",
    as: "font",
  },
]);
```

## Real-World Production Example: Netflix-Scale CDN

```typescript
// Netflix-Style CDN Architecture
class NetflixCDNStrategy {
  private cdnManager: MultiCDNManager;
  private cacheManager: CacheInvalidationManager;
  private geoSelector: GeoCDNSelector;
  private optimizer: ContentOptimizer;

  constructor() {
    this.cdnManager = new MultiCDNManager([
      {
        name: "Primary-US",
        baseUrl: "https://cdn-us.netflix.example.com",
        priority: 1,
        timeout: 3000,
        enabled: true,
        successCount: 0,
        failureCount: 0,
      },
      {
        name: "Primary-EU",
        baseUrl: "https://cdn-eu.netflix.example.com",
        priority: 1,
        timeout: 3000,
        enabled: true,
        successCount: 0,
        failureCount: 0,
      },
      {
        name: "Backup-Global",
        baseUrl: "https://cdn-backup.netflix.example.com",
        priority: 2,
        timeout: 5000,
        enabled: true,
        successCount: 0,
        failureCount: 0,
      },
    ]);

    this.cacheManager = new CacheInvalidationManager([
      {
        name: "Primary",
        apiUrl: "https://api-cdn.netflix.example.com",
        apiKey: process.env.CDN_API_KEY!,
      },
    ]);

    this.geoSelector = new GeoCDNSelector([
      {
        url: "https://cdn-us-east.netflix.example.com",
        latitude: 40.7128,
        longitude: -74.006,
        priority: 1,
        region: "us-east",
      },
      {
        url: "https://cdn-eu-west.netflix.example.com",
        latitude: 51.5074,
        longitude: -0.1278,
        priority: 1,
        region: "eu-west",
      },
    ]);

    this.optimizer = new ContentOptimizer();
  }

  async loadVideo(videoId: string, quality: VideoQuality): Promise<string> {
    // Get user's nearest CDN
    const cdnUrl = await this.geoSelector.getNearestCDN();

    // Generate video URL with optimal quality
    const videoPath = `/videos/${videoId}/${quality}.mp4`;

    // Fetch through multi-CDN with failover
    const response = await this.cdnManager.fetchAsset(videoPath);

    return response.url;
  }

  async loadThumbnail(videoId: string): Promise<string> {
    const thumbnailUrl = `https://cdn.netflix.example.com/thumbnails/${videoId}.jpg`;

    // Optimize for user's device
    const format = await this.optimizer.getOptimalImageFormat();

    return this.optimizer.optimizeImage(thumbnailUrl, {
      width: 400,
      quality: 85,
      format,
    });
  }

  async preloadContent(videoIds: string[]): Promise<void> {
    // Warm cache for upcoming content
    const urls = videoIds.map(
      (id) => `https://cdn.netflix.example.com/videos/${id}/720p.mp4`,
    );

    const warmer = new CacheWarmingManager(urls, 5);
    await warmer.warmCache();
  }

  async handleContentUpdate(videoId: string, version: string): Promise<void> {
    // Invalidate old version
    await this.cacheManager.purgeByTag(`video-${videoId}`);

    // Warm cache for new version
    await this.preloadContent([videoId]);
  }
}

type VideoQuality = "360p" | "480p" | "720p" | "1080p" | "4k";

// Usage
const netflix = new NetflixCDNStrategy();

// Load video with optimal quality
const videoUrl = await netflix.loadVideo("abc123", "1080p");

// Preload upcoming episodes
await netflix.preloadContent(["abc124", "abc125", "abc126"]);
```

## Best Practices

1. **Use immutable URLs** - Version assets for efficient caching
2. **Implement proper cache headers** - Max-age, immutable, must-revalidate
3. **Enable compression** - Gzip, Brotli for text assets
4. **Optimize images** - WebP, AVIF, responsive sizes
5. **Use resource hints** - preconnect, dns-prefetch, preload
6. **Implement failover** - Multi-CDN strategy for reliability
7. **Monitor CDN performance** - Track latency, cache hit rates
8. **Use cache tags** - Efficient bulk invalidation
9. **Warm critical cache** - Pre-populate after deployments
10. **Implement origin shield** - Reduce origin load

## Key Takeaways

1. **Edge caching reduces latency** - Serve content from nearest location, use Cache-Control headers strategically

2. **Geo-distribution is critical** - Select CDN based on user location, calculate distances using Haversine formula

3. **Origin shield protects backends** - Single CDN layer between edge and origin, reduces origin requests by 80%+

4. **Cache invalidation strategies matter** - Purge by URL, tag, or wildcard, version-based invalidation is safest

5. **Cache warming prevents cold starts** - Pre-populate cache after deployments, prioritize critical assets first

6. **Multi-CDN provides resilience** - Automatic failover between providers, monitor and disable unhealthy CDNs

7. **Content optimization saves bandwidth** - WebP/AVIF for images, responsive srcsets, minify CSS/JS

8. **Resource hints improve performance** - preconnect for CDN origins, preload for critical assets, dns-prefetch for third-parties

9. **Smart routing maximizes speed** - Use Anycast or GeoDNS, measure real user metrics (RUM) for optimization

10. **Netflix-scale requires sophistication** - Multi-region, multi-CDN, adaptive streaming, predictive cache warming for next episodes
