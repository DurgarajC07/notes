# Caching Strategies - Complete Guide

## Core Concepts

Caching is the process of storing copies of files or computation results to serve future requests faster. Effective caching strategies dramatically improve performance, reduce server load, and enhance user experience.

### Caching Layers

1. **Browser Cache**: Local storage in user's browser
2. **Service Worker Cache**: Programmable cache for offline support
3. **CDN Cache**: Edge servers close to users
4. **Server Cache**: Backend caching (Redis, Memcached)
5. **Database Cache**: Query results caching

## 1. HTTP Caching

### Cache-Control Headers

```typescript
// HTTP Cache Manager
class HTTPCacheManager {
  // Generate cache headers based on resource type
  getCacheHeaders(resourceType: ResourceType): Headers {
    const headers = new Headers();

    switch (resourceType) {
      case "static-asset":
        // Immutable static assets (fonts, icons with hash in filename)
        headers.set("Cache-Control", "public, max-age=31536000, immutable");
        break;

      case "versioned-bundle":
        // Versioned JS/CSS bundles
        headers.set("Cache-Control", "public, max-age=31536000, immutable");
        break;

      case "image":
        // Images: 30 days, can be revalidated
        headers.set("Cache-Control", "public, max-age=2592000");
        break;

      case "api-data":
        // API responses: 5 minutes, must revalidate
        headers.set("Cache-Control", "public, max-age=300, must-revalidate");
        break;

      case "html":
        // HTML pages: no cache, always revalidate
        headers.set("Cache-Control", "no-cache, must-revalidate");
        break;

      case "sensitive":
        // Sensitive data: no cache at all
        headers.set(
          "Cache-Control",
          "no-store, no-cache, must-revalidate, private",
        );
        headers.set("Pragma", "no-cache");
        headers.set("Expires", "0");
        break;
    }

    return headers;
  }

  // ETag generation
  generateETag(content: string | ArrayBuffer): string {
    if (typeof content === "string") {
      return this.hashString(content);
    }
    return this.hashBuffer(content);
  }

  private hashString(str: string): string {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = (hash << 5) - hash + char;
      hash = hash & hash;
    }
    return `"${Math.abs(hash).toString(36)}"`;
  }

  private hashBuffer(buffer: ArrayBuffer): string {
    const view = new Uint8Array(buffer);
    let hash = 0;
    for (let i = 0; i < view.length; i++) {
      hash = (hash << 5) - hash + view[i];
      hash = hash & hash;
    }
    return `"${Math.abs(hash).toString(36)}"`;
  }

  // Conditional request handling
  shouldReturnNotModified(
    requestHeaders: Headers,
    etag: string,
    lastModified: Date,
  ): boolean {
    const ifNoneMatch = requestHeaders.get("If-None-Match");
    const ifModifiedSince = requestHeaders.get("If-Modified-Since");

    // Check ETag
    if (ifNoneMatch && ifNoneMatch === etag) {
      return true;
    }

    // Check Last-Modified
    if (ifModifiedSince) {
      const modifiedSinceDate = new Date(ifModifiedSince);
      if (lastModified <= modifiedSinceDate) {
        return true;
      }
    }

    return false;
  }
}

type ResourceType =
  | "static-asset"
  | "versioned-bundle"
  | "image"
  | "api-data"
  | "html"
  | "sensitive";

// Usage Example
const cacheManager = new HTTPCacheManager();

// Serve static asset with caching
async function serveStaticAsset(request: Request): Promise<Response> {
  const asset = await fetchAsset(request.url);
  const etag = cacheManager.generateETag(asset.content);

  // Check if client has cached version
  if (
    cacheManager.shouldReturnNotModified(
      request.headers,
      etag,
      asset.lastModified,
    )
  ) {
    return new Response(null, { status: 304 });
  }

  const headers = cacheManager.getCacheHeaders("static-asset");
  headers.set("ETag", etag);
  headers.set("Last-Modified", asset.lastModified.toUTCString());

  return new Response(asset.content, {
    status: 200,
    headers,
  });
}

interface Asset {
  content: string | ArrayBuffer;
  lastModified: Date;
}

function fetchAsset(url: string): Promise<Asset> {
  // Implementation to fetch asset
  return Promise.resolve({
    content: "",
    lastModified: new Date(),
  });
}
```

### Vary Header Management

```typescript
// Vary Header Manager
class VaryHeaderManager {
  private varyHeaders: Set<string> = new Set();

  addVaryHeader(header: string): void {
    this.varyHeaders.add(header);
  }

  getVaryHeader(): string {
    return Array.from(this.varyHeaders).join(", ");
  }

  // Common vary patterns
  static getStandardVary(): VaryHeaderManager {
    const manager = new VaryHeaderManager();
    manager.addVaryHeader("Accept-Encoding"); // For gzip/brotli
    manager.addVaryHeader("Accept"); // For content negotiation
    return manager;
  }

  static getImageVary(): VaryHeaderManager {
    const manager = new VaryHeaderManager();
    manager.addVaryHeader("Accept"); // For WebP support
    manager.addVaryHeader("DPR"); // For device pixel ratio
    manager.addVaryHeader("Width"); // For responsive images
    return manager;
  }

  static getAPIVary(): VaryHeaderManager {
    const manager = new VaryHeaderManager();
    manager.addVaryHeader("Accept-Encoding");
    manager.addVaryHeader("Authorization"); // User-specific responses
    return manager;
  }
}

// Usage
const varyManager = VaryHeaderManager.getImageVary();
headers.set("Vary", varyManager.getVaryHeader());
```

## 2. Service Worker Caching

### Cache-First Strategy

```typescript
// Cache-First Strategy Implementation
class CacheFirstStrategy {
  private cacheName: string;

  constructor(cacheName: string) {
    this.cacheName = cacheName;
  }

  async handle(request: Request): Promise<Response> {
    const cache = await caches.open(this.cacheName);

    // Try cache first
    const cachedResponse = await cache.match(request);
    if (cachedResponse) {
      return cachedResponse;
    }

    // Fallback to network
    try {
      const networkResponse = await fetch(request);

      // Cache successful responses
      if (networkResponse.ok) {
        cache.put(request, networkResponse.clone());
      }

      return networkResponse;
    } catch (error) {
      // Return fallback if both cache and network fail
      return this.getFallbackResponse();
    }
  }

  private getFallbackResponse(): Response {
    return new Response("Offline - Content not available", {
      status: 503,
      headers: { "Content-Type": "text/plain" },
    });
  }
}
```

### Network-First Strategy

```typescript
// Network-First Strategy
class NetworkFirstStrategy {
  private cacheName: string;
  private timeout: number;

  constructor(cacheName: string, timeout: number = 3000) {
    this.cacheName = cacheName;
    this.timeout = timeout;
  }

  async handle(request: Request): Promise<Response> {
    const cache = await caches.open(this.cacheName);

    try {
      // Try network first with timeout
      const networkResponse = await this.fetchWithTimeout(request);

      // Update cache with fresh response
      if (networkResponse.ok) {
        cache.put(request, networkResponse.clone());
      }

      return networkResponse;
    } catch (error) {
      // Fallback to cache
      const cachedResponse = await cache.match(request);

      if (cachedResponse) {
        return cachedResponse;
      }

      throw error;
    }
  }

  private async fetchWithTimeout(request: Request): Promise<Response> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeout);

    try {
      const response = await fetch(request, {
        signal: controller.signal,
      });
      clearTimeout(timeoutId);
      return response;
    } catch (error) {
      clearTimeout(timeoutId);
      throw error;
    }
  }
}
```

### Stale-While-Revalidate Strategy

```typescript
// Stale-While-Revalidate Strategy
class StaleWhileRevalidateStrategy {
  private cacheName: string;
  private maxAge: number;

  constructor(cacheName: string, maxAge: number = 86400000) {
    this.cacheName = cacheName;
    this.maxAge = maxAge;
  }

  async handle(request: Request): Promise<Response> {
    const cache = await caches.open(this.cacheName);

    // Get cached response
    const cachedResponse = await cache.match(request);

    // Fetch fresh response in background
    const fetchPromise = this.fetchAndCache(request, cache);

    // Return cached response if available
    if (cachedResponse) {
      const cacheTime = this.getCacheTime(cachedResponse);
      const age = Date.now() - cacheTime;

      // If cache is fresh enough, return immediately and revalidate in background
      if (age < this.maxAge) {
        fetchPromise.catch(() => {}); // Ignore revalidation errors
        return cachedResponse;
      }
    }

    // If no cache or cache too old, wait for network
    return fetchPromise;
  }

  private async fetchAndCache(
    request: Request,
    cache: Cache,
  ): Promise<Response> {
    const response = await fetch(request);

    if (response.ok) {
      const responseToCache = response.clone();
      const headers = new Headers(responseToCache.headers);
      headers.set("X-Cache-Time", Date.now().toString());

      const cachedResponse = new Response(responseToCache.body, {
        status: responseToCache.status,
        statusText: responseToCache.statusText,
        headers,
      });

      await cache.put(request, cachedResponse);
    }

    return response;
  }

  private getCacheTime(response: Response): number {
    const cacheTime = response.headers.get("X-Cache-Time");
    return cacheTime ? parseInt(cacheTime, 10) : 0;
  }
}
```

### Service Worker with Multiple Strategies

```typescript
// Service Worker Implementation
const CACHE_VERSION = "v1";
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const DYNAMIC_CACHE = `dynamic-${CACHE_VERSION}`;
const API_CACHE = `api-${CACHE_VERSION}`;

const cacheFirstStrategy = new CacheFirstStrategy(STATIC_CACHE);
const networkFirstStrategy = new NetworkFirstStrategy(DYNAMIC_CACHE);
const staleWhileRevalidate = new StaleWhileRevalidateStrategy(API_CACHE);

// Install event: Cache static assets
self.addEventListener("install", (event: ExtendableEvent) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then((cache) => {
      return cache.addAll([
        "/",
        "/index.html",
        "/css/styles.css",
        "/js/app.js",
        "/images/logo.png",
      ]);
    }),
  );
});

// Activate event: Clean up old caches
self.addEventListener("activate", (event: ExtendableEvent) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter(
            (name) =>
              name !== STATIC_CACHE &&
              name !== DYNAMIC_CACHE &&
              name !== API_CACHE,
          )
          .map((name) => caches.delete(name)),
      );
    }),
  );
});

// Fetch event: Apply caching strategies
self.addEventListener("fetch", (event: FetchEvent) => {
  const { request } = event;
  const url = new URL(request.url);

  // Static assets: Cache-First
  if (url.pathname.match(/\.(js|css|png|jpg|jpeg|gif|svg|woff|woff2)$/)) {
    event.respondWith(cacheFirstStrategy.handle(request));
    return;
  }

  // API requests: Stale-While-Revalidate
  if (url.pathname.startsWith("/api/")) {
    event.respondWith(staleWhileRevalidate.handle(request));
    return;
  }

  // HTML pages: Network-First
  event.respondWith(networkFirstStrategy.handle(request));
});
```

## 3. localStorage / sessionStorage

```typescript
// Storage Manager with Expiration
class StorageManager {
  private storage: Storage;

  constructor(storage: Storage = localStorage) {
    this.storage = storage;
  }

  set(key: string, value: any, ttl?: number): void {
    const item: StorageItem = {
      value,
      timestamp: Date.now(),
      ttl,
    };

    this.storage.setItem(key, JSON.stringify(item));
  }

  get<T>(key: string): T | null {
    const itemStr = this.storage.getItem(key);

    if (!itemStr) {
      return null;
    }

    try {
      const item: StorageItem = JSON.parse(itemStr);

      // Check expiration
      if (item.ttl) {
        const age = Date.now() - item.timestamp;
        if (age > item.ttl) {
          this.remove(key);
          return null;
        }
      }

      return item.value as T;
    } catch {
      return null;
    }
  }

  remove(key: string): void {
    this.storage.removeItem(key);
  }

  clear(): void {
    this.storage.clear();
  }

  // Get all keys
  keys(): string[] {
    return Object.keys(this.storage);
  }

  // Clean expired items
  cleanExpired(): void {
    const keys = this.keys();

    keys.forEach((key) => {
      const itemStr = this.storage.getItem(key);
      if (!itemStr) return;

      try {
        const item: StorageItem = JSON.parse(itemStr);

        if (item.ttl) {
          const age = Date.now() - item.timestamp;
          if (age > item.ttl) {
            this.remove(key);
          }
        }
      } catch {
        // Invalid item, remove it
        this.remove(key);
      }
    });
  }

  // Get storage size
  getSize(): number {
    let size = 0;
    const keys = this.keys();

    keys.forEach((key) => {
      const value = this.storage.getItem(key);
      if (value) {
        size += key.length + value.length;
      }
    });

    return size;
  }
}

interface StorageItem {
  value: any;
  timestamp: number;
  ttl?: number;
}

// Usage
const storage = new StorageManager(localStorage);

// Store with 1 hour TTL
storage.set(
  "user-preferences",
  {
    theme: "dark",
    language: "en",
  },
  3600000,
);

// Retrieve
const preferences = storage.get<UserPreferences>("user-preferences");

// Clean expired items periodically
setInterval(() => {
  storage.cleanExpired();
}, 60000); // Every minute

interface UserPreferences {
  theme: string;
  language: string;
}
```

## 4. IndexedDB

```typescript
// IndexedDB Manager
class IndexedDBManager {
  private dbName: string;
  private version: number;
  private db: IDBDatabase | null = null;

  constructor(dbName: string, version: number = 1) {
    this.dbName = dbName;
    this.version = version;
  }

  async init(stores: StoreConfig[]): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, this.version);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => {
        this.db = request.result;
        resolve();
      };

      request.onupgradeneeded = (event: IDBVersionChangeEvent) => {
        const db = (event.target as IDBOpenDBRequest).result;

        stores.forEach((storeConfig) => {
          if (!db.objectStoreNames.contains(storeConfig.name)) {
            const objectStore = db.createObjectStore(storeConfig.name, {
              keyPath: storeConfig.keyPath,
              autoIncrement: storeConfig.autoIncrement,
            });

            // Create indexes
            storeConfig.indexes?.forEach((index) => {
              objectStore.createIndex(index.name, index.keyPath, {
                unique: index.unique,
              });
            });
          }
        });
      };
    });
  }

  async set(storeName: string, data: any): Promise<void> {
    if (!this.db) throw new Error("Database not initialized");

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], "readwrite");
      const store = transaction.objectStore(storeName);
      const request = store.put(data);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async get<T>(storeName: string, key: IDBValidKey): Promise<T | null> {
    if (!this.db) throw new Error("Database not initialized");

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], "readonly");
      const store = transaction.objectStore(storeName);
      const request = store.get(key);

      request.onsuccess = () => resolve(request.result || null);
      request.onerror = () => reject(request.error);
    });
  }

  async getAll<T>(storeName: string): Promise<T[]> {
    if (!this.db) throw new Error("Database not initialized");

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], "readonly");
      const store = transaction.objectStore(storeName);
      const request = store.getAll();

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async delete(storeName: string, key: IDBValidKey): Promise<void> {
    if (!this.db) throw new Error("Database not initialized");

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], "readwrite");
      const store = transaction.objectStore(storeName);
      const request = store.delete(key);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async clear(storeName: string): Promise<void> {
    if (!this.db) throw new Error("Database not initialized");

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], "readwrite");
      const store = transaction.objectStore(storeName);
      const request = store.clear();

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  // Query by index
  async queryByIndex<T>(
    storeName: string,
    indexName: string,
    value: IDBValidKey,
  ): Promise<T[]> {
    if (!this.db) throw new Error("Database not initialized");

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], "readonly");
      const store = transaction.objectStore(storeName);
      const index = store.index(indexName);
      const request = index.getAll(value);

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }
}

interface StoreConfig {
  name: string;
  keyPath: string;
  autoIncrement?: boolean;
  indexes?: IndexConfig[];
}

interface IndexConfig {
  name: string;
  keyPath: string;
  unique?: boolean;
}

// Usage
const db = new IndexedDBManager("my-app-cache", 1);

await db.init([
  {
    name: "products",
    keyPath: "id",
    indexes: [
      { name: "by-category", keyPath: "category" },
      { name: "by-price", keyPath: "price" },
    ],
  },
  {
    name: "users",
    keyPath: "id",
    indexes: [{ name: "by-email", keyPath: "email", unique: true }],
  },
]);

// Store data
await db.set("products", {
  id: "1",
  name: "Laptop",
  category: "electronics",
  price: 999,
});

// Retrieve data
const product = await db.get<Product>("products", "1");

// Query by index
const electronics = await db.queryByIndex<Product>(
  "products",
  "by-category",
  "electronics",
);

interface Product {
  id: string;
  name: string;
  category: string;
  price: number;
}
```

## 5. Cache Versioning

```typescript
// Cache Version Manager
class CacheVersionManager {
  private currentVersion: string;
  private versionKey = "cache-version";

  constructor(version: string) {
    this.currentVersion = version;
  }

  async checkAndUpdate(): Promise<boolean> {
    const storedVersion = localStorage.getItem(this.versionKey);

    if (storedVersion !== this.currentVersion) {
      await this.clearAllCaches();
      localStorage.setItem(this.versionKey, this.currentVersion);
      return true; // Cache was cleared
    }

    return false; // No update needed
  }

  private async clearAllCaches(): Promise<void> {
    // Clear Cache API
    const cacheNames = await caches.keys();
    await Promise.all(cacheNames.map((cacheName) => caches.delete(cacheName)));

    // Clear localStorage
    const keysToKeep = [this.versionKey, "user-session"];
    Object.keys(localStorage).forEach((key) => {
      if (!keysToKeep.includes(key)) {
        localStorage.removeItem(key);
      }
    });

    // Clear sessionStorage
    sessionStorage.clear();

    // Clear IndexedDB
    const databases = await indexedDB.databases();
    await Promise.all(
      databases.map((db) => {
        if (db.name) {
          return new Promise<void>((resolve, reject) => {
            const request = indexedDB.deleteDatabase(db.name!);
            request.onsuccess = () => resolve();
            request.onerror = () => reject(request.error);
          });
        }
        return Promise.resolve();
      }),
    );
  }

  getCurrentVersion(): string {
    return this.currentVersion;
  }
}

// Usage
const versionManager = new CacheVersionManager("1.2.3");

// Check on app initialization
const wasCleared = await versionManager.checkAndUpdate();
if (wasCleared) {
  console.log("Cache cleared due to version update");
}
```

## 6. React Query / SWR Pattern

```typescript
// React Query-like Cache Implementation
class QueryCache {
  private cache: Map<string, CacheEntry> = new Map();
  private subscribers: Map<string, Set<Subscriber>> = new Map();
  private defaultStaleTime = 5000; // 5 seconds
  private defaultCacheTime = 300000; // 5 minutes

  async query<T>(
    key: string,
    fetcher: () => Promise<T>,
    options: QueryOptions = {}
  ): Promise<CachedData<T>> {
    const {
      staleTime = this.defaultStaleTime,
      cacheTime = this.defaultCacheTime,
      refetchOnMount = true,
    } = options;

    const cached = this.cache.get(key);

    // Return cached data if fresh
    if (cached && !this.isStale(cached, staleTime)) {
      return {
        data: cached.data as T,
        isLoading: false,
        isStale: false,
      };
    }

    // Return stale data and refetch in background
    if (cached) {
      this.refetchInBackground(key, fetcher, cacheTime);
      return {
        data: cached.data as T,
        isLoading: false,
        isStale: true,
      };
    }

    // No cache, fetch fresh data
    const data = await fetcher();
    this.setCache(key, data, cacheTime);

    return {
      data,
      isLoading: false,
      isStale: false,
    };
  }

  private async refetchInBackground<T>(
    key: string,
    fetcher: () => Promise<T>,
    cacheTime: number
  ): Promise<void> {
    try {
      const data = await fetcher();
      this.setCache(key, data, cacheTime);
      this.notifySubscribers(key, data);
    } catch (error) {
      console.error('Background refetch failed:', error);
    }
  }

  private setCache(key: string, data: any, cacheTime: number): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      cacheTime,
    });

    // Set expiration timer
    setTimeout(() => {
      this.cache.delete(key);
    }, cacheTime);
  }

  private isStale(entry: CacheEntry, staleTime: number): boolean {
    return Date.now() - entry.timestamp > staleTime;
  }

  subscribe(key: string, subscriber: Subscriber): () => void {
    if (!this.subscribers.has(key)) {
      this.subscribers.set(key, new Set());
    }

    this.subscribers.get(key)!.add(subscriber);

    // Return unsubscribe function
    return () => {
      this.subscribers.get(key)?.delete(subscriber);
    };
  }

  private notifySubscribers(key: string, data: any): void {
    const subscribers = this.subscribers.get(key);
    if (subscribers) {
      subscribers.forEach(subscriber => subscriber(data));
    }
  }

  invalidate(key: string): void {
    this.cache.delete(key);
    this.notifySubscribers(key, null);
  }

  clear(): void {
    this.cache.clear();
    this.subscribers.clear();
  }
}

interface CacheEntry {
  data: any;
  timestamp: number;
  cacheTime: number;
}

interface QueryOptions {
  staleTime?: number;
  cacheTime?: number;
  refetchOnMount?: boolean;
}

interface CachedData<T> {
  data: T;
  isLoading: boolean;
  isStale: boolean;
}

type Subscriber = (data: any) => void;

// React Hook
function useQuery<T>(
  key: string,
  fetcher: () => Promise<T>,
  options?: QueryOptions
): CachedData<T> {
  const [data, setData] = useState<CachedData<T>>({
    data: null as any,
    isLoading: true,
    isStale: false,
  });

  useEffect(() => {
    const fetchData = async () => {
      const result = await queryCache.query(key, fetcher, options);
      setData(result);
    };

    fetchData();

    // Subscribe to updates
    const unsubscribe = queryCache.subscribe(key, (newData) => {
      setData({
        data: newData,
        isLoading: false,
        isStale: false,
      });
    });

    return unsubscribe;
  }, [key]);

  return data;
}

// Global cache instance
const queryCache = new QueryCache();

// Usage in component
function ProductList() {
  const { data: products, isLoading, isStale } = useQuery(
    'products',
    async () => {
      const response = await fetch('/api/products');
      return response.json();
    },
    {
      staleTime: 10000, // Consider fresh for 10 seconds
      cacheTime: 300000, // Keep in cache for 5 minutes
    }
  );

  if (isLoading) return <div>Loading...</div>;

  return (
    <div>
      {isStale && <div>Refreshing...</div>}
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

## Real-World Production Example

```typescript
// Comprehensive Caching Strategy
class ProductionCacheStrategy {
  private httpCache: HTTPCacheManager;
  private storage: StorageManager;
  private indexedDB: IndexedDBManager;
  private queryCache: QueryCache;
  private versionManager: CacheVersionManager;

  constructor(version: string) {
    this.httpCache = new HTTPCacheManager();
    this.storage = new StorageManager();
    this.indexedDB = new IndexedDBManager("app-cache", 1);
    this.queryCache = new QueryCache();
    this.versionManager = new CacheVersionManager(version);
  }

  async initialize(): Promise<void> {
    // Check version and clear if needed
    await this.versionManager.checkAndUpdate();

    // Initialize IndexedDB
    await this.indexedDB.init([
      {
        name: "api-responses",
        keyPath: "url",
        indexes: [{ name: "by-timestamp", keyPath: "timestamp" }],
      },
    ]);

    // Clean expired storage items
    this.storage.cleanExpired();
  }

  // Fetch with multi-layer caching
  async fetchWithCache<T>(url: string, options: FetchOptions = {}): Promise<T> {
    const {
      forceRefresh = false,
      cacheStrategy = "stale-while-revalidate",
      ttl = 300000,
    } = options;

    // Check memory cache first
    if (!forceRefresh) {
      const memoryCache = this.queryCache.cache.get(url);
      if (memoryCache && !this.isExpired(memoryCache, ttl)) {
        return memoryCache.data as T;
      }
    }

    // Check IndexedDB
    if (!forceRefresh) {
      const dbCache = await this.indexedDB.get<CachedResponse<T>>(
        "api-responses",
        url,
      );

      if (dbCache && !this.isExpired(dbCache, ttl)) {
        // Update memory cache
        this.queryCache.cache.set(url, {
          data: dbCache.data,
          timestamp: dbCache.timestamp,
          cacheTime: ttl,
        });

        // Revalidate in background if using SWR
        if (cacheStrategy === "stale-while-revalidate") {
          this.revalidateInBackground(url, ttl);
        }

        return dbCache.data;
      }
    }

    // Fetch from network
    const response = await fetch(url);
    const data = await response.json();

    // Update all cache layers
    await this.updateAllCaches(url, data, ttl);

    return data;
  }

  private async updateAllCaches(
    url: string,
    data: any,
    ttl: number,
  ): Promise<void> {
    const timestamp = Date.now();

    // Update memory cache
    this.queryCache.cache.set(url, {
      data,
      timestamp,
      cacheTime: ttl,
    });

    // Update IndexedDB
    await this.indexedDB.set("api-responses", {
      url,
      data,
      timestamp,
    });

    // Notify subscribers
    this.queryCache.notifySubscribers(url, data);
  }

  private async revalidateInBackground(
    url: string,
    ttl: number,
  ): Promise<void> {
    try {
      const response = await fetch(url);
      const data = await response.json();
      await this.updateAllCaches(url, data, ttl);
    } catch (error) {
      console.error("Background revalidation failed:", error);
    }
  }

  private isExpired(entry: { timestamp: number }, ttl: number): boolean {
    return Date.now() - entry.timestamp > ttl;
  }

  // Clear all caches
  async clearAll(): Promise<void> {
    this.queryCache.clear();
    this.storage.clear();
    await this.indexedDB.clear("api-responses");
  }
}

interface FetchOptions {
  forceRefresh?: boolean;
  cacheStrategy?: "cache-first" | "network-first" | "stale-while-revalidate";
  ttl?: number;
}

interface CachedResponse<T> {
  url: string;
  data: T;
  timestamp: number;
}

// Usage
const cacheStrategy = new ProductionCacheStrategy("1.0.0");
await cacheStrategy.initialize();

// Fetch with caching
const products = await cacheStrategy.fetchWithCache<Product[]>(
  "/api/products",
  {
    cacheStrategy: "stale-while-revalidate",
    ttl: 60000,
  },
);
```

## Best Practices

1. **Use appropriate cache strategies** - Static assets: Cache-First, API: Stale-While-Revalidate
2. **Set proper cache headers** - Max-age, immutable, must-revalidate based on resource type
3. **Implement cache versioning** - Clear caches on app updates
4. **Use ETags for validation** - Save bandwidth with conditional requests
5. **Layer caching strategically** - Memory > IndexedDB > Network
6. **Set expiration times** - Prevent stale data, balance freshness vs performance
7. **Handle cache failures gracefully** - Fallback to network, show offline UI
8. **Monitor cache hit rates** - Optimize strategies based on real data
9. **Respect privacy** - Don't cache sensitive data, use proper headers
10. **Test offline scenarios** - Ensure app works with cached data

## Key Takeaways

1. **HTTP caching is foundational** - Cache-Control, ETag, Last-Modified enable efficient browser and CDN caching

2. **Service Workers provide programmatic control** - Cache-First for static, Network-First for dynamic, Stale-While-Revalidate for balance

3. **localStorage is synchronous and limited** - 5-10MB limit, use for small data, implement TTL for expiration

4. **IndexedDB handles large data** - Asynchronous, no size limit (practically), perfect for offline-first apps

5. **Cache versioning prevents stale bugs** - Invalidate all caches on app updates, maintain version in storage

6. **Stale-While-Revalidate is optimal** - Instant response from cache, fresh data in background, best UX/performance trade-off

7. **Multi-layer caching maximizes performance** - Memory cache (fastest) → IndexedDB (persistent) → Network (fresh)

8. **React Query/SWR patterns simplify data fetching** - Automatic caching, revalidation, de-duplication, built-in loading states

9. **Cache invalidation is the hard part** - Purge by URL/tag, version-based strategies, time-based expiration

10. **Production requires comprehensive strategy** - Combine HTTP caching, Service Workers, client-side storage, monitor performance
