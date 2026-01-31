# Scaling Patterns - Complete Guide

## Core Concepts

Scaling frontend applications involves strategies to handle increased traffic, larger datasets, and growing complexity while maintaining performance and reliability.

### Types of Scaling

- **Horizontal Scaling**: Add more servers/instances
- **Vertical Scaling**: Increase server resources
- **Load Balancing**: Distribute traffic across servers
- **Caching**: Reduce redundant computations
- **Database Scaling**: Sharding, replication, read replicas

## 1. Horizontal Scaling

### Auto-Scaling Configuration

```typescript
// Auto-Scaling Manager
class AutoScalingManager {
  private instances: ServerInstance[] = [];
  private minInstances: number;
  private maxInstances: number;
  private targetCPU: number;
  private checkInterval: number;

  constructor(config: AutoScalingConfig) {
    this.minInstances = config.minInstances;
    this.maxInstances = config.maxInstances;
    this.targetCPU = config.targetCPU;
    this.checkInterval = config.checkInterval;

    this.startMonitoring();
  }

  private startMonitoring(): void {
    setInterval(() => {
      this.checkAndScale();
    }, this.checkInterval);
  }

  private async checkAndScale(): Promise<void> {
    const metrics = await this.getMetrics();
    const avgCPU = this.calculateAverageCPU(metrics);

    if (avgCPU > this.targetCPU && this.instances.length < this.maxInstances) {
      await this.scaleOut();
    } else if (
      avgCPU < this.targetCPU * 0.5 &&
      this.instances.length > this.minInstances
    ) {
      await this.scaleIn();
    }
  }

  private async scaleOut(): Promise<void> {
    console.log("Scaling out: Adding new instance");

    const newInstance: ServerInstance = {
      id: crypto.randomUUID(),
      status: "starting",
      cpu: 0,
      memory: 0,
      startTime: Date.now(),
    };

    this.instances.push(newInstance);

    // Simulate instance startup
    setTimeout(() => {
      newInstance.status = "running";
      console.log(`Instance ${newInstance.id} is now running`);
    }, 5000);
  }

  private async scaleIn(): Promise<void> {
    if (this.instances.length <= this.minInstances) return;

    console.log("Scaling in: Removing instance");

    // Find instance with least connections
    const instanceToRemove = this.instances.reduce((prev, current) =>
      prev.connections < current.connections ? prev : current,
    );

    instanceToRemove.status = "terminating";

    // Drain connections before terminating
    await this.drainConnections(instanceToRemove);

    this.instances = this.instances.filter((i) => i.id !== instanceToRemove.id);
    console.log(`Instance ${instanceToRemove.id} terminated`);
  }

  private async drainConnections(instance: ServerInstance): Promise<void> {
    // Wait for active connections to complete
    return new Promise((resolve) => {
      const checkInterval = setInterval(() => {
        if (instance.connections === 0) {
          clearInterval(checkInterval);
          resolve();
        }
      }, 1000);

      // Force termination after 30 seconds
      setTimeout(() => {
        clearInterval(checkInterval);
        resolve();
      }, 30000);
    });
  }

  private async getMetrics(): Promise<InstanceMetrics[]> {
    // Simulate fetching metrics from monitoring service
    return this.instances.map((instance) => ({
      instanceId: instance.id,
      cpu: Math.random() * 100,
      memory: Math.random() * 100,
      connections: Math.floor(Math.random() * 1000),
    }));
  }

  private calculateAverageCPU(metrics: InstanceMetrics[]): number {
    if (metrics.length === 0) return 0;
    const sum = metrics.reduce((acc, m) => acc + m.cpu, 0);
    return sum / metrics.length;
  }

  getInstances(): ServerInstance[] {
    return [...this.instances];
  }
}

interface AutoScalingConfig {
  minInstances: number;
  maxInstances: number;
  targetCPU: number;
  checkInterval: number;
}

interface ServerInstance {
  id: string;
  status: "starting" | "running" | "terminating";
  cpu: number;
  memory: number;
  startTime: number;
  connections?: number;
}

interface InstanceMetrics {
  instanceId: string;
  cpu: number;
  memory: number;
  connections: number;
}

// Usage
const autoScaler = new AutoScalingManager({
  minInstances: 2,
  maxInstances: 10,
  targetCPU: 70,
  checkInterval: 60000, // Check every minute
});
```

## 2. Load Balancing

### Load Balancing Algorithms

```typescript
// Load Balancer Implementation
class LoadBalancer {
  private servers: Server[] = [];
  private algorithm: LoadBalancingAlgorithm;
  private currentIndex: number = 0;

  constructor(
    servers: Server[],
    algorithm: LoadBalancingAlgorithm = "round-robin",
  ) {
    this.servers = servers;
    this.algorithm = algorithm;
  }

  // Round Robin
  private roundRobin(): Server {
    const server = this.servers[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.servers.length;
    return server;
  }

  // Least Connections
  private leastConnections(): Server {
    return this.servers.reduce((prev, current) =>
      prev.activeConnections < current.activeConnections ? prev : current,
    );
  }

  // Weighted Round Robin
  private weightedRoundRobin(): Server {
    const totalWeight = this.servers.reduce((sum, s) => sum + s.weight, 0);
    let random = Math.random() * totalWeight;

    for (const server of this.servers) {
      random -= server.weight;
      if (random <= 0) {
        return server;
      }
    }

    return this.servers[0];
  }

  // IP Hash
  private ipHash(clientIP: string): Server {
    let hash = 0;
    for (let i = 0; i < clientIP.length; i++) {
      hash = (hash << 5) - hash + clientIP.charCodeAt(i);
      hash = hash & hash;
    }

    const index = Math.abs(hash) % this.servers.length;
    return this.servers[index];
  }

  // Least Response Time
  private leastResponseTime(): Server {
    return this.servers.reduce((prev, current) => {
      const prevScore = prev.activeConnections * prev.avgResponseTime;
      const currentScore = current.activeConnections * current.avgResponseTime;
      return prevScore < currentScore ? prev : current;
    });
  }

  // Get next server
  getServer(clientIP?: string): Server {
    // Filter healthy servers
    const healthyServers = this.servers.filter((s) => s.healthy);

    if (healthyServers.length === 0) {
      throw new Error("No healthy servers available");
    }

    // Temporarily update servers list
    const originalServers = this.servers;
    this.servers = healthyServers;

    let server: Server;

    switch (this.algorithm) {
      case "round-robin":
        server = this.roundRobin();
        break;
      case "least-connections":
        server = this.leastConnections();
        break;
      case "weighted-round-robin":
        server = this.weightedRoundRobin();
        break;
      case "ip-hash":
        if (!clientIP) throw new Error("Client IP required for IP hash");
        server = this.ipHash(clientIP);
        break;
      case "least-response-time":
        server = this.leastResponseTime();
        break;
      default:
        server = this.roundRobin();
    }

    this.servers = originalServers;
    return server;
  }

  // Health check
  async performHealthCheck(): Promise<void> {
    const promises = this.servers.map(async (server) => {
      try {
        const response = await fetch(`${server.url}/health`, {
          method: "HEAD",
          timeout: 5000,
        } as any);

        server.healthy = response.ok;

        if (response.ok) {
          server.consecutiveFailures = 0;
        } else {
          server.consecutiveFailures++;
        }
      } catch (error) {
        server.healthy = false;
        server.consecutiveFailures++;
      }

      // Mark server as down after 3 consecutive failures
      if (server.consecutiveFailures >= 3) {
        server.healthy = false;
      }
    });

    await Promise.all(promises);
  }

  // Start periodic health checks
  startHealthChecks(interval: number = 10000): void {
    setInterval(() => {
      this.performHealthCheck();
    }, interval);
  }

  // Update server metrics
  updateServerMetrics(
    serverId: string,
    metrics: { activeConnections?: number; avgResponseTime?: number },
  ): void {
    const server = this.servers.find((s) => s.id === serverId);
    if (server) {
      if (metrics.activeConnections !== undefined) {
        server.activeConnections = metrics.activeConnections;
      }
      if (metrics.avgResponseTime !== undefined) {
        server.avgResponseTime = metrics.avgResponseTime;
      }
    }
  }

  getServerStats(): ServerStats[] {
    return this.servers.map((server) => ({
      id: server.id,
      url: server.url,
      healthy: server.healthy,
      activeConnections: server.activeConnections,
      avgResponseTime: server.avgResponseTime,
      weight: server.weight,
    }));
  }
}

interface Server {
  id: string;
  url: string;
  healthy: boolean;
  activeConnections: number;
  avgResponseTime: number;
  weight: number;
  consecutiveFailures: number;
}

interface ServerStats {
  id: string;
  url: string;
  healthy: boolean;
  activeConnections: number;
  avgResponseTime: number;
  weight: number;
}

type LoadBalancingAlgorithm =
  | "round-robin"
  | "least-connections"
  | "weighted-round-robin"
  | "ip-hash"
  | "least-response-time";

// Usage
const loadBalancer = new LoadBalancer(
  [
    {
      id: "server-1",
      url: "https://server1.example.com",
      healthy: true,
      activeConnections: 0,
      avgResponseTime: 50,
      weight: 1,
      consecutiveFailures: 0,
    },
    {
      id: "server-2",
      url: "https://server2.example.com",
      healthy: true,
      activeConnections: 0,
      avgResponseTime: 45,
      weight: 2, // Higher weight = more traffic
      consecutiveFailures: 0,
    },
    {
      id: "server-3",
      url: "https://server3.example.com",
      healthy: true,
      activeConnections: 0,
      avgResponseTime: 60,
      weight: 1,
      consecutiveFailures: 0,
    },
  ],
  "weighted-round-robin",
);

// Start health checks
loadBalancer.startHealthChecks(10000);

// Get server for request
const server = loadBalancer.getServer("192.168.1.1");
console.log("Routing to:", server.url);
```

## 3. CDN Offloading

```typescript
// CDN Offloading Strategy
class CDNOffloadingManager {
  private cdnUrl: string;
  private originUrl: string;
  private offloadablePatterns: RegExp[];

  constructor(config: CDNOffloadingConfig) {
    this.cdnUrl = config.cdnUrl;
    this.originUrl = config.originUrl;
    this.offloadablePatterns = config.offloadablePatterns;
  }

  shouldOffloadToCDN(url: string): boolean {
    return this.offloadablePatterns.some((pattern) => pattern.test(url));
  }

  rewriteUrl(url: string): string {
    if (!this.shouldOffloadToCDN(url)) {
      return url;
    }

    // Replace origin with CDN
    return url.replace(this.originUrl, this.cdnUrl);
  }

  // Intercept fetch requests
  interceptFetch(): void {
    const originalFetch = window.fetch;

    window.fetch = async (input: RequestInfo | URL, init?: RequestInit) => {
      const url =
        typeof input === "string"
          ? input
          : input instanceof URL
            ? input.href
            : input.url;
      const rewrittenUrl = this.rewriteUrl(url);

      if (rewrittenUrl !== url) {
        console.log(`Offloading to CDN: ${url} -> ${rewrittenUrl}`);
        return originalFetch(rewrittenUrl, init);
      }

      return originalFetch(input, init);
    };
  }

  // Rewrite image sources
  rewriteImageSources(): void {
    document.querySelectorAll("img").forEach((img) => {
      const originalSrc = img.src;
      const rewrittenSrc = this.rewriteUrl(originalSrc);

      if (rewrittenSrc !== originalSrc) {
        img.src = rewrittenSrc;
      }
    });
  }

  // Calculate offload metrics
  calculateOffloadRate(): OffloadMetrics {
    const totalRequests = performance.getEntriesByType("resource").length;
    const cdnRequests = performance
      .getEntriesByType("resource")
      .filter((entry: any) => entry.name.includes(this.cdnUrl)).length;

    return {
      totalRequests,
      cdnRequests,
      offloadRate: totalRequests > 0 ? (cdnRequests / totalRequests) * 100 : 0,
      bandwidthSaved: this.estimateBandwidthSaved(cdnRequests),
    };
  }

  private estimateBandwidthSaved(cdnRequests: number): number {
    // Estimate based on average resource size
    const avgResourceSize = 50 * 1024; // 50KB
    return cdnRequests * avgResourceSize;
  }
}

interface CDNOffloadingConfig {
  cdnUrl: string;
  originUrl: string;
  offloadablePatterns: RegExp[];
}

interface OffloadMetrics {
  totalRequests: number;
  cdnRequests: number;
  offloadRate: number;
  bandwidthSaved: number;
}

// Usage
const cdnManager = new CDNOffloadingManager({
  cdnUrl: "https://cdn.example.com",
  originUrl: "https://www.example.com",
  offloadablePatterns: [
    /\.(jpg|jpeg|png|gif|svg|webp)$/i,
    /\.(css|js)$/i,
    /\.(woff|woff2|ttf|eot)$/i,
    /\/static\//i,
  ],
});

// Intercept and rewrite URLs
cdnManager.interceptFetch();

// Get offload metrics
const metrics = cdnManager.calculateOffloadRate();
console.log(`CDN offload rate: ${metrics.offloadRate.toFixed(2)}%`);
console.log(
  `Bandwidth saved: ${(metrics.bandwidthSaved / 1024 / 1024).toFixed(2)} MB`,
);
```

## 4. Database Scaling

### Read Replicas

```typescript
// Database Connection Pool with Read Replicas
class DatabasePool {
  private primary: DatabaseConnection;
  private replicas: DatabaseConnection[];
  private replicaIndex: number = 0;

  constructor(config: DatabasePoolConfig) {
    this.primary = {
      id: "primary",
      url: config.primaryUrl,
      type: "primary",
      healthy: true,
    };

    this.replicas = config.replicaUrls.map((url, index) => ({
      id: `replica-${index}`,
      url,
      type: "replica",
      healthy: true,
    }));
  }

  // Execute write query (always goes to primary)
  async executeWrite<T>(query: string, params?: any[]): Promise<T> {
    if (!this.primary.healthy) {
      throw new Error("Primary database is not healthy");
    }

    return this.executeQuery(this.primary, query, params);
  }

  // Execute read query (load balanced across replicas)
  async executeRead<T>(query: string, params?: any[]): Promise<T> {
    const replica = this.getHealthyReplica();

    if (!replica) {
      console.warn("No healthy replicas, falling back to primary");
      return this.executeQuery(this.primary, query, params);
    }

    try {
      return await this.executeQuery(replica, query, params);
    } catch (error) {
      console.error("Replica query failed, falling back to primary:", error);
      return this.executeQuery(this.primary, query, params);
    }
  }

  private getHealthyReplica(): DatabaseConnection | null {
    const healthyReplicas = this.replicas.filter((r) => r.healthy);

    if (healthyReplicas.length === 0) {
      return null;
    }

    // Round-robin load balancing
    const replica = healthyReplicas[this.replicaIndex % healthyReplicas.length];
    this.replicaIndex++;

    return replica;
  }

  private async executeQuery<T>(
    connection: DatabaseConnection,
    query: string,
    params?: any[],
  ): Promise<T> {
    // Simulate database query
    console.log(`Executing on ${connection.id}:`, query);

    // In real implementation, use actual database driver
    const response = await fetch(`${connection.url}/query`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ query, params }),
    });

    return response.json();
  }

  async performHealthCheck(): Promise<void> {
    const allConnections = [this.primary, ...this.replicas];

    const promises = allConnections.map(async (connection) => {
      try {
        const response = await fetch(`${connection.url}/health`, {
          method: "HEAD",
          timeout: 5000,
        } as any);

        connection.healthy = response.ok;
      } catch (error) {
        connection.healthy = false;
      }
    });

    await Promise.all(promises);
  }

  startHealthChecks(interval: number = 30000): void {
    setInterval(() => {
      this.performHealthCheck();
    }, interval);
  }
}

interface DatabasePoolConfig {
  primaryUrl: string;
  replicaUrls: string[];
}

interface DatabaseConnection {
  id: string;
  url: string;
  type: "primary" | "replica";
  healthy: boolean;
}

// Usage
const dbPool = new DatabasePool({
  primaryUrl: "postgresql://primary.db.example.com:5432",
  replicaUrls: [
    "postgresql://replica1.db.example.com:5432",
    "postgresql://replica2.db.example.com:5432",
    "postgresql://replica3.db.example.com:5432",
  ],
});

dbPool.startHealthChecks();

// Write operations go to primary
await dbPool.executeWrite("INSERT INTO users (name, email) VALUES ($1, $2)", [
  "John Doe",
  "john@example.com",
]);

// Read operations load balanced across replicas
const users = await dbPool.executeRead(
  "SELECT * FROM users WHERE active = true",
);
```

### Database Sharding

```typescript
// Database Sharding Manager
class ShardingManager {
  private shards: Shard[];
  private shardingStrategy: ShardingStrategy;

  constructor(shards: Shard[], strategy: ShardingStrategy = "range") {
    this.shards = shards;
    this.shardingStrategy = strategy;
  }

  // Determine which shard to use
  getShard(shardKey: string | number): Shard {
    switch (this.shardingStrategy) {
      case "hash":
        return this.hashShard(shardKey);
      case "range":
        return this.rangeShard(shardKey as number);
      case "directory":
        return this.directoryShard(shardKey);
      default:
        return this.shards[0];
    }
  }

  // Hash-based sharding
  private hashShard(key: string | number): Shard {
    const keyStr = String(key);
    let hash = 0;

    for (let i = 0; i < keyStr.length; i++) {
      hash = (hash << 5) - hash + keyStr.charCodeAt(i);
      hash = hash & hash;
    }

    const index = Math.abs(hash) % this.shards.length;
    return this.shards[index];
  }

  // Range-based sharding
  private rangeShard(key: number): Shard {
    for (const shard of this.shards) {
      if (shard.rangeStart !== undefined && shard.rangeEnd !== undefined) {
        if (key >= shard.rangeStart && key < shard.rangeEnd) {
          return shard;
        }
      }
    }

    // Default to first shard if no range matches
    return this.shards[0];
  }

  // Directory-based sharding
  private directoryShard(key: string | number): Shard {
    const keyStr = String(key);

    // Look up in directory (simplified example)
    for (const shard of this.shards) {
      if (shard.keys?.includes(keyStr)) {
        return shard;
      }
    }

    // Default to first shard
    return this.shards[0];
  }

  // Execute query on appropriate shard
  async executeQuery<T>(
    shardKey: string | number,
    query: string,
    params?: any[],
  ): Promise<T> {
    const shard = this.getShard(shardKey);

    const response = await fetch(`${shard.url}/query`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ query, params }),
    });

    return response.json();
  }

  // Execute query across all shards (scatter-gather)
  async executeQueryAllShards<T>(query: string, params?: any[]): Promise<T[]> {
    const promises = this.shards.map((shard) =>
      fetch(`${shard.url}/query`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ query, params }),
      }).then((res) => res.json()),
    );

    return Promise.all(promises);
  }

  // Rebalance shards (add new shard)
  async addShard(newShard: Shard): Promise<void> {
    console.log("Adding new shard:", newShard.id);

    // In production, this would involve data migration
    this.shards.push(newShard);

    // Recalculate ranges for range-based sharding
    if (this.shardingStrategy === "range") {
      this.recalculateRanges();
    }
  }

  private recalculateRanges(): void {
    const rangeSize = Math.floor(Number.MAX_SAFE_INTEGER / this.shards.length);

    this.shards.forEach((shard, index) => {
      shard.rangeStart = index * rangeSize;
      shard.rangeEnd = (index + 1) * rangeSize;
    });
  }
}

interface Shard {
  id: string;
  url: string;
  rangeStart?: number;
  rangeEnd?: number;
  keys?: string[];
}

type ShardingStrategy = "hash" | "range" | "directory";

// Usage
const shardingManager = new ShardingManager(
  [
    {
      id: "shard-1",
      url: "postgresql://shard1.db.example.com:5432",
      rangeStart: 0,
      rangeEnd: 1000000,
    },
    {
      id: "shard-2",
      url: "postgresql://shard2.db.example.com:5432",
      rangeStart: 1000000,
      rangeEnd: 2000000,
    },
    {
      id: "shard-3",
      url: "postgresql://shard3.db.example.com:5432",
      rangeStart: 2000000,
      rangeEnd: 3000000,
    },
  ],
  "hash",
);

// Query specific shard
const user = await shardingManager.executeQuery(
  "user-123", // Shard key
  "SELECT * FROM users WHERE id = $1",
  ["user-123"],
);

// Query all shards
const allUsers = await shardingManager.executeQueryAllShards(
  "SELECT COUNT(*) FROM users",
);
```

## 5. Caching Layers (Redis, Memcached)

```typescript
// Multi-Layer Cache Manager
class MultiLayerCacheManager {
  private memoryCache: Map<string, CacheEntry> = new Map();
  private redisUrl: string;
  private memoryCacheSize: number = 1000;

  constructor(redisUrl: string) {
    this.redisUrl = redisUrl;
  }

  // Get from cache (check memory first, then Redis)
  async get<T>(key: string): Promise<T | null> {
    // Check memory cache
    const memoryEntry = this.memoryCache.get(key);
    if (memoryEntry && !this.isExpired(memoryEntry)) {
      return memoryEntry.value as T;
    }

    // Check Redis
    const redisValue = await this.getFromRedis<T>(key);
    if (redisValue !== null) {
      // Populate memory cache
      this.setMemoryCache(key, redisValue);
      return redisValue;
    }

    return null;
  }

  // Set in cache (both memory and Redis)
  async set(key: string, value: any, ttl?: number): Promise<void> {
    // Set in memory cache
    this.setMemoryCache(key, value, ttl);

    // Set in Redis
    await this.setInRedis(key, value, ttl);
  }

  private setMemoryCache(key: string, value: any, ttl?: number): void {
    // Evict oldest entry if cache is full
    if (this.memoryCache.size >= this.memoryCacheSize) {
      const oldestKey = this.memoryCache.keys().next().value;
      this.memoryCache.delete(oldestKey);
    }

    this.memoryCache.set(key, {
      value,
      timestamp: Date.now(),
      ttl,
    });
  }

  private async getFromRedis<T>(key: string): Promise<T | null> {
    try {
      const response = await fetch(`${this.redisUrl}/get/${key}`);

      if (!response.ok) {
        return null;
      }

      const data = await response.json();
      return data.value as T;
    } catch (error) {
      console.error("Redis get failed:", error);
      return null;
    }
  }

  private async setInRedis(
    key: string,
    value: any,
    ttl?: number,
  ): Promise<void> {
    try {
      await fetch(`${this.redisUrl}/set`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ key, value, ttl }),
      });
    } catch (error) {
      console.error("Redis set failed:", error);
    }
  }

  private isExpired(entry: CacheEntry): boolean {
    if (!entry.ttl) return false;
    return Date.now() - entry.timestamp > entry.ttl;
  }

  // Invalidate cache entry
  async invalidate(key: string): Promise<void> {
    this.memoryCache.delete(key);
    await this.deleteFromRedis(key);
  }

  private async deleteFromRedis(key: string): Promise<void> {
    try {
      await fetch(`${this.redisUrl}/delete/${key}`, {
        method: "DELETE",
      });
    } catch (error) {
      console.error("Redis delete failed:", error);
    }
  }

  // Cache-aside pattern (lazy loading)
  async getOrFetch<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl?: number,
  ): Promise<T> {
    // Try cache first
    const cached = await this.get<T>(key);
    if (cached !== null) {
      return cached;
    }

    // Fetch from source
    const value = await fetcher();

    // Store in cache
    await this.set(key, value, ttl);

    return value;
  }

  // Write-through pattern
  async setAndPersist<T>(
    key: string,
    value: T,
    persister: (value: T) => Promise<void>,
    ttl?: number,
  ): Promise<void> {
    // Write to database first
    await persister(value);

    // Then update cache
    await this.set(key, value, ttl);
  }

  // Get cache statistics
  getStats(): CacheStats {
    return {
      memorySize: this.memoryCache.size,
      memoryCapacity: this.memoryCacheSize,
      memoryUtilization: (this.memoryCache.size / this.memoryCacheSize) * 100,
    };
  }
}

interface CacheEntry {
  value: any;
  timestamp: number;
  ttl?: number;
}

interface CacheStats {
  memorySize: number;
  memoryCapacity: number;
  memoryUtilization: number;
}

// Usage
const cacheManager = new MultiLayerCacheManager("http://redis.example.com");

// Cache-aside pattern
const user = await cacheManager.getOrFetch(
  "user:123",
  async () => {
    const response = await fetch("/api/users/123");
    return response.json();
  },
  3600000, // 1 hour TTL
);

// Write-through pattern
await cacheManager.setAndPersist(
  "user:123",
  updatedUser,
  async (user) => {
    await fetch("/api/users/123", {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(user),
    });
  },
  3600000,
);
```

## 6. WebSocket Scaling

```typescript
// WebSocket Scaling Manager
class WebSocketScalingManager {
  private connections: Map<string, WebSocket> = new Map();
  private redisUrl: string;
  private serverId: string;

  constructor(redisUrl: string) {
    this.redisUrl = redisUrl;
    this.serverId = crypto.randomUUID();
    this.subscribeToRedis();
  }

  // Handle new WebSocket connection
  handleConnection(connectionId: string, ws: WebSocket): void {
    this.connections.set(connectionId, ws);

    // Register connection in Redis
    this.registerConnection(connectionId);

    ws.addEventListener("message", (event) => {
      this.handleMessage(connectionId, event.data);
    });

    ws.addEventListener("close", () => {
      this.handleDisconnect(connectionId);
    });
  }

  private async registerConnection(connectionId: string): Promise<void> {
    // Store connection metadata in Redis
    await fetch(`${this.redisUrl}/set`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        key: `connection:${connectionId}`,
        value: { serverId: this.serverId, timestamp: Date.now() },
      }),
    });
  }

  private async handleMessage(
    connectionId: string,
    message: string,
  ): Promise<void> {
    try {
      const data = JSON.parse(message);

      // Publish message to Redis for distribution across servers
      await this.publishMessage({
        type: data.type,
        payload: data.payload,
        from: connectionId,
        serverId: this.serverId,
      });
    } catch (error) {
      console.error("Failed to handle message:", error);
    }
  }

  private async publishMessage(message: WebSocketMessage): Promise<void> {
    await fetch(`${this.redisUrl}/publish`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        channel: "websocket-messages",
        message,
      }),
    });
  }

  private subscribeToRedis(): void {
    // In real implementation, use Redis pub/sub
    // This is simplified for demonstration
    const eventSource = new EventSource(
      `${this.redisUrl}/subscribe/websocket-messages`,
    );

    eventSource.onmessage = (event) => {
      const message: WebSocketMessage = JSON.parse(event.data);

      // Only process messages from other servers
      if (message.serverId !== this.serverId) {
        this.broadcastToLocalConnections(message);
      }
    };
  }

  private broadcastToLocalConnections(message: WebSocketMessage): void {
    this.connections.forEach((ws, connectionId) => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify(message));
      }
    });
  }

  // Send message to specific user across all servers
  async sendToUser(userId: string, message: any): Promise<void> {
    // Look up user connections in Redis
    const connections = await this.getUserConnections(userId);

    const wsMessage: WebSocketMessage = {
      type: "user-message",
      payload: message,
      from: "system",
      serverId: this.serverId,
      targetUser: userId,
    };

    // Publish to Redis for distribution
    await this.publishMessage(wsMessage);
  }

  private async getUserConnections(userId: string): Promise<string[]> {
    const response = await fetch(
      `${this.redisUrl}/get/user:${userId}:connections`,
    );
    const data = await response.json();
    return data.value || [];
  }

  private handleDisconnect(connectionId: string): void {
    this.connections.delete(connectionId);
    this.unregisterConnection(connectionId);
  }

  private async unregisterConnection(connectionId: string): Promise<void> {
    await fetch(`${this.redisUrl}/delete/connection:${connectionId}`, {
      method: "DELETE",
    });
  }

  getConnectionCount(): number {
    return this.connections.size;
  }
}

interface WebSocketMessage {
  type: string;
  payload: any;
  from: string;
  serverId: string;
  targetUser?: string;
}

// Usage
const wsManager = new WebSocketScalingManager("http://redis.example.com");

// Handle new connection
const ws = new WebSocket("ws://localhost:8080");
wsManager.handleConnection("conn-123", ws);

// Send message to specific user across all servers
await wsManager.sendToUser("user-456", {
  type: "notification",
  message: "You have a new message",
});
```

## Real-World Example: Netflix Scale

```typescript
// Netflix-Scale Architecture
class NetflixScaleArchitecture {
  private autoScaler: AutoScalingManager;
  private loadBalancer: LoadBalancer;
  private dbPool: DatabasePool;
  private cacheManager: MultiLayerCacheManager;
  private cdnManager: CDNOffloadingManager;

  constructor() {
    // Auto-scaling: 10-100 instances
    this.autoScaler = new AutoScalingManager({
      minInstances: 10,
      maxInstances: 100,
      targetCPU: 70,
      checkInterval: 60000,
    });

    // Load balancer with 50 backend servers
    this.loadBalancer = new LoadBalancer(
      Array.from({ length: 50 }, (_, i) => ({
        id: `server-${i}`,
        url: `https://backend-${i}.netflix.example.com`,
        healthy: true,
        activeConnections: 0,
        avgResponseTime: 50,
        weight: 1,
        consecutiveFailures: 0,
      })),
      "least-response-time",
    );

    // Database pool: 1 primary + 10 read replicas
    this.dbPool = new DatabasePool({
      primaryUrl: "postgresql://primary.db.netflix.example.com:5432",
      replicaUrls: Array.from(
        { length: 10 },
        (_, i) => `postgresql://replica-${i}.db.netflix.example.com:5432`,
      ),
    });

    // Multi-layer cache
    this.cacheManager = new MultiLayerCacheManager(
      "http://redis.netflix.example.com",
    );

    // CDN offloading
    this.cdnManager = new CDNOffloadingManager({
      cdnUrl: "https://cdn.netflix.example.com",
      originUrl: "https://www.netflix.example.com",
      offloadablePatterns: [
        /\.(jpg|jpeg|png|gif|svg|webp)$/i,
        /\.(mp4|webm)$/i,
        /\.(css|js)$/i,
      ],
    });

    this.initialize();
  }

  private initialize(): void {
    this.loadBalancer.startHealthChecks(10000);
    this.dbPool.startHealthChecks(30000);
    this.cdnManager.interceptFetch();
  }

  // Handle video streaming request
  async streamVideo(videoId: string, quality: string): Promise<Response> {
    // Check cache first
    const cacheKey = `video:${videoId}:${quality}`;
    const cached = await this.cacheManager.get<string>(cacheKey);

    if (cached) {
      return new Response(cached, {
        headers: {
          "Content-Type": "video/mp4",
          "X-Cache": "HIT",
        },
      });
    }

    // Get server from load balancer
    const server = this.loadBalancer.getServer();

    // Fetch video from origin/CDN
    const videoUrl = this.cdnManager.rewriteUrl(
      `${server.url}/videos/${videoId}/${quality}.mp4`,
    );

    const response = await fetch(videoUrl);

    // Cache for future requests
    if (response.ok) {
      const buffer = await response.arrayBuffer();
      await this.cacheManager.set(cacheKey, buffer, 3600000); // 1 hour
    }

    return response;
  }

  // Handle user profile request
  async getUserProfile(userId: string): Promise<any> {
    // Try cache
    const profile = await this.cacheManager.getOrFetch(
      `profile:${userId}`,
      async () => {
        // Read from replica
        return await this.dbPool.executeRead(
          "SELECT * FROM users WHERE id = $1",
          [userId],
        );
      },
      300000, // 5 minutes
    );

    return profile;
  }

  // Handle user update (write operation)
  async updateUserProfile(userId: string, updates: any): Promise<void> {
    // Write to primary database
    await this.dbPool.executeWrite("UPDATE users SET data = $1 WHERE id = $2", [
      JSON.stringify(updates),
      userId,
    ]);

    // Invalidate cache
    await this.cacheManager.invalidate(`profile:${userId}`);
  }

  // Get system metrics
  getMetrics(): SystemMetrics {
    return {
      instances: this.autoScaler.getInstances().length,
      servers: this.loadBalancer.getServerStats(),
      cacheStats: this.cacheManager.getStats(),
      cdnOffloadRate: this.cdnManager.calculateOffloadRate(),
    };
  }
}

interface SystemMetrics {
  instances: number;
  servers: ServerStats[];
  cacheStats: CacheStats;
  cdnOffloadRate: OffloadMetrics;
}

// Usage
const netflix = new NetflixScaleArchitecture();

// Stream video
const videoResponse = await netflix.streamVideo("video-123", "1080p");

// Get user profile
const profile = await netflix.getUserProfile("user-456");

// Get metrics
const metrics = netflix.getMetrics();
console.log("System metrics:", metrics);
```

## Best Practices

1. **Start with vertical scaling** - Easier to implement, upgrade hardware first
2. **Design for horizontal scaling** - Stateless services, shared-nothing architecture
3. **Use load balancing wisely** - Choose algorithm based on workload characteristics
4. **Implement health checks** - Automatic failover for unhealthy instances
5. **Cache aggressively** - Multiple layers (memory, Redis, CDN)
6. **Offload to CDN** - Static assets, video streaming, global distribution
7. **Use read replicas** - Distribute read traffic, keep primary for writes only
8. **Shard carefully** - Choose sharding key wisely, plan for rebalancing
9. **Monitor everything** - Metrics, logs, traces for all components
10. **Test at scale** - Load testing, chaos engineering, disaster recovery

## Key Takeaways

1. **Horizontal scaling enables unlimited growth** - Add more servers instead of bigger servers, stateless design is critical

2. **Load balancing distributes traffic** - Round-robin, least connections, weighted, IP hash - choose based on workload

3. **Auto-scaling adapts to demand** - Scale out on high load, scale in during low traffic, saves costs while maintaining performance

4. **CDN offloading reduces origin load** - 80%+ of static assets served from edge, massive bandwidth savings

5. **Read replicas scale read-heavy workloads** - Netflix: 90% reads, 10% writes, replicas handle all reads

6. **Database sharding enables horizontal database scaling** - Hash, range, or directory-based, partition data across multiple databases

7. **Multi-layer caching maximizes performance** - Memory (fastest) → Redis (fast, shared) → Database (slow)

8. **WebSocket scaling requires pub/sub** - Redis pub/sub distributes messages across servers, enables horizontal scaling

9. **Health checks enable automatic recovery** - Remove unhealthy instances, prevent cascading failures

10. **Netflix scale requires all patterns** - Auto-scaling + load balancing + CDN + caching + database scaling + monitoring = 200M+ users
