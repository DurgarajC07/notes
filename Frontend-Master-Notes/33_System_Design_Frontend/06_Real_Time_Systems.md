# 🎯 Frontend System Design: Real-Time Dashboard

> Designing a real-time dashboard is a common senior/staff-level interview question. This requires architectural decisions around data fetching, state management, performance optimization, and scalability.

---

## 📖 1. Problem Statement

**Design a real-time analytics dashboard that displays:**
- Live metrics (updating every 1-3 seconds)
- Historical charts (last 24 hours)
- Multiple data sources (API, WebSocket, polling)
- Support for 10,000+ concurrent users
- Must handle 100+ metrics per dashboard
- <2s initial load time, <100ms update latency

**Example:** Admin dashboard for SaaS application showing:
- Active users (real-time)
- Revenue metrics (hourly updates)
- System health (live status)
- Error rates (real-time alerts)

---

## 🧠 2. Requirements Gathering

### Functional Requirements

1. **Real-Time Updates:**
   - WebSocket connection for live data
   - Fallback to polling if WebSocket fails
   - Auto-reconnect on connection loss

2. **Data Visualization:**
   - Line charts, bar charts, pie charts
   - Time-series data (1min, 5min, 1hour aggregations)
   - Drill-down capabilities

3. **User Interactions:**
   - Filter by date range
   - Select/deselect metrics
   - Export data (CSV, PDF)
   - Dashboard customization

4. **Notifications:**
   - Alert on threshold breach
   - Toast notifications
   - Sound alerts (optional)

### Non-Functional Requirements

1. **Performance:**
   - Initial load: <2s
   - Update latency: <100ms
   - Smooth 60fps animations
   - Bundle size: <500KB

2. **Scalability:**
   - Support 10,000 concurrent users
   - Handle 100+ metrics per dashboard
   - Process 1000+ updates/second

3. **Reliability:**
   - 99.9% uptime
   - Graceful degradation
   - Error recovery

4. **Security:**
   - Authentication (JWT)
   - Authorization (role-based)
   - Data encryption (TLS)
   - XSS/CSRF protection

---

## ⚙️ 3. High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│                  Client (React)                 │
├─────────────────────────────────────────────────┤
│  Components  │  State Management  │  Services   │
│  - Dashboard │  - Redux/Zustand  │  - WS Client│
│  - Charts    │  - Query Cache    │  - API Svc  │
│  - Widgets   │  - Local State    │  - Auth Svc │
└─────────────────────────────────────────────────┘
                      ↓↑
┌─────────────────────────────────────────────────┐
│              Load Balancer (Nginx)              │
└─────────────────────────────────────────────────┘
                      ↓↑
┌──────────────────────────┬──────────────────────┐
│    WebSocket Server      │    API Server        │
│    (Socket.io/WS)        │    (Node.js/Go)      │
│    - Push updates        │    - Historical data │
│    - Connection mgmt     │    - Auth/Auth       │
└──────────────────────────┴──────────────────────┘
                      ↓↑
┌─────────────────────────────────────────────────┐
│              Redis (Pub/Sub + Cache)            │
└─────────────────────────────────────────────────┘
                      ↓↑
┌─────────────────────────────────────────────────┐
│      Database (PostgreSQL + TimescaleDB)        │
│      - Metrics storage (time-series)            │
│      - User data                                │
└─────────────────────────────────────────────────┘
```

---

## ✅ 4. Component Design

### Frontend Architecture

```
src/
├── components/
│   ├── Dashboard/
│   │   ├── Dashboard.tsx
│   │   ├── DashboardHeader.tsx
│   │   ├── MetricsGrid.tsx
│   │   └── Widget.tsx
│   ├── Charts/
│   │   ├── LineChart.tsx
│   │   ├── BarChart.tsx
│   │   └── PieChart.tsx
│   └── common/
│       ├── LoadingSpinner.tsx
│       └── ErrorBoundary.tsx
├── hooks/
│   ├── useWebSocket.ts
│   ├── useMetrics.ts
│   └── useRealtimeUpdates.ts
├── services/
│   ├── websocket.ts
│   ├── api.ts
│   └── auth.ts
├── store/
│   ├── metricsSlice.ts
│   ├── uiSlice.ts
│   └── store.ts
└── utils/
    ├── dataTransform.ts
    └── chartHelpers.ts
```

### Key Components

**1. Dashboard Container:**
```typescript
// Dashboard.tsx
import { useMetrics, useRealtimeUpdates } from '@/hooks';
import { MetricsGrid } from './MetricsGrid';

export function Dashboard() {
  // Fetch historical data on mount
  const { data: historicalData, isLoading } = useMetrics({
    range: '24h',
    interval: '5m'
  });
  
  // Subscribe to real-time updates
  const realtimeData = useRealtimeUpdates({
    metrics: ['activeUsers', 'revenue', 'errorRate'],
    onUpdate: (data) => {
      // Update local state
      updateMetrics(data);
    }
  });
  
  if (isLoading) return <LoadingSpinner />;
  
  return (
    <div className="dashboard">
      <DashboardHeader />
      <MetricsGrid 
        historical={historicalData} 
        realtime={realtimeData} 
      />
    </div>
  );
}
```

**2. WebSocket Hook:**
```typescript
// hooks/useWebSocket.ts
import { useEffect, useRef, useState } from 'react';

interface UseWebSocketOptions {
  url: string;
  onMessage: (data: any) => void;
  onError?: (error: Error) => void;
  reconnect?: boolean;
  reconnectInterval?: number;
}

export function useWebSocket({
  url,
  onMessage,
  onError,
  reconnect = true,
  reconnectInterval = 3000
}: UseWebSocketOptions) {
  const [isConnected, setIsConnected] = useState(false);
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectTimeoutRef = useRef<number>();
  
  const connect = useCallback(() => {
    try {
      const ws = new WebSocket(url);
      
      ws.onopen = () => {
        console.log('WebSocket connected');
        setIsConnected(true);
      };
      
      ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        onMessage(data);
      };
      
      ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        onError?.(error);
      };
      
      ws.onclose = () => {
        console.log('WebSocket disconnected');
        setIsConnected(false);
        
        // Auto-reconnect
        if (reconnect) {
          reconnectTimeoutRef.current = window.setTimeout(() => {
            console.log('Reconnecting...');
            connect();
          }, reconnectInterval);
        }
      };
      
      wsRef.current = ws;
    } catch (error) {
      onError?.(error as Error);
    }
  }, [url, onMessage, onError, reconnect, reconnectInterval]);
  
  useEffect(() => {
    connect();
    
    return () => {
      wsRef.current?.close();
      if (reconnectTimeoutRef.current) {
        clearTimeout(reconnectTimeoutRef.current);
      }
    };
  }, [connect]);
  
  const send = useCallback((data: any) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(data));
    }
  }, []);
  
  return { isConnected, send };
}
```

**3. Real-Time Updates Hook:**
```typescript
// hooks/useRealtimeUpdates.ts
import { useEffect, useState } from 'react';
import { useWebSocket } from './useWebSocket';

interface Metric {
  id: string;
  value: number;
  timestamp: number;
  label: string;
}

export function useRealtimeUpdates({
  metrics,
  onUpdate
}: {
  metrics: string[];
  onUpdate: (data: Metric[]) => void;
}) {
  const [data, setData] = useState<Metric[]>([]);
  
  const { isConnected, send } = useWebSocket({
    url: 'wss://api.example.com/ws',
    onMessage: (payload) => {
      if (payload.type === 'metrics') {
        setData(payload.data);
        onUpdate(payload.data);
      }
    },
    onError: (error) => {
      console.error('WebSocket error:', error);
      // Fallback to polling
      startPolling();
    }
  });
  
  // Subscribe to metrics on connect
  useEffect(() => {
    if (isConnected) {
      send({
        type: 'subscribe',
        metrics
      });
    }
    
    return () => {
      send({
        type: 'unsubscribe',
        metrics
      });
    };
  }, [isConnected, metrics, send]);
  
  return { data, isConnected };
}
```

---

## 🚀 5. Performance Optimization

### 1. Data Aggregation Strategy

```typescript
// Aggregate data on client to reduce updates
class DataAggregator {
  private buffer: Metric[] = [];
  private flushInterval = 1000; // 1 second
  private timer: number | null = null;
  
  add(metric: Metric) {
    this.buffer.push(metric);
    
    if (!this.timer) {
      this.timer = window.setTimeout(() => {
        this.flush();
      }, this.flushInterval);
    }
  }
  
  flush() {
    if (this.buffer.length === 0) return;
    
    // Aggregate by metric ID
    const aggregated = this.buffer.reduce((acc, metric) => {
      if (!acc[metric.id]) {
        acc[metric.id] = {
          ...metric,
          values: []
        };
      }
      acc[metric.id].values.push(metric.value);
      return acc;
    }, {} as Record<string, any>);
    
    // Calculate aggregates (avg, min, max)
    Object.values(aggregated).forEach((metric: any) => {
      metric.value = metric.values.reduce((a, b) => a + b) / metric.values.length;
      metric.min = Math.min(...metric.values);
      metric.max = Math.max(...metric.values);
      delete metric.values;
    });
    
    // Emit aggregated data
    this.onFlush(Object.values(aggregated));
    
    // Clear buffer
    this.buffer = [];
    this.timer = null;
  }
  
  constructor(private onFlush: (data: Metric[]) => void) {}
}

// Usage
const aggregator = new DataAggregator((data) => {
  updateCharts(data);
});

wsClient.on('metric', (metric) => {
  aggregator.add(metric);
});
```

### 2. Chart Optimization

```typescript
// Virtualize large datasets
import { useMemo } from 'react';

function LineChart({ data }: { data: DataPoint[] }) {
  // Downsample data for large datasets
  const downsampledData = useMemo(() => {
    if (data.length <= 100) return data;
    
    // LTTB (Largest Triangle Three Buckets) algorithm
    return downsampleLTTB(data, 100);
  }, [data]);
  
  return (
    <ResponsiveContainer>
      <LineChart data={downsampledData}>
        <Line 
          type="monotone" 
          dataKey="value" 
          stroke="#8884d8"
          dot={false}  // Disable dots for performance
          isAnimationActive={false}  // Disable animation
        />
      </LineChart>
    </ResponsiveContainer>
  );
}

// LTTB algorithm (simplified)
function downsampleLTTB(data: DataPoint[], threshold: number): DataPoint[] {
  if (data.length <= threshold) return data;
  
  const sampled: DataPoint[] = [data[0]]; // Always keep first
  const bucketSize = (data.length - 2) / (threshold - 2);
  
  for (let i = 0; i < threshold - 2; i++) {
    const bucketStart = Math.floor(i * bucketSize) + 1;
    const bucketEnd = Math.floor((i + 1) * bucketSize) + 1;
    
    // Find point with largest triangle area
    let maxArea = -1;
    let maxAreaIndex = bucketStart;
    
    for (let j = bucketStart; j < bucketEnd; j++) {
      const area = calculateTriangleArea(
        sampled[sampled.length - 1],
        data[j],
        data[Math.floor((i + 2) * bucketSize)]
      );
      
      if (area > maxArea) {
        maxArea = area;
        maxAreaIndex = j;
      }
    }
    
    sampled.push(data[maxAreaIndex]);
  }
  
  sampled.push(data[data.length - 1]); // Always keep last
  return sampled;
}
```

### 3. Memory Management

```typescript
// Circular buffer for time-series data
class CircularBuffer<T> {
  private buffer: T[];
  private size: number;
  private pointer = 0;
  
  constructor(size: number) {
    this.size = size;
    this.buffer = new Array(size);
  }
  
  push(item: T) {
    this.buffer[this.pointer] = item;
    this.pointer = (this.pointer + 1) % this.size;
  }
  
  getAll(): T[] {
    // Return items in chronological order
    return [
      ...this.buffer.slice(this.pointer),
      ...this.buffer.slice(0, this.pointer)
    ].filter(Boolean);
  }
  
  clear() {
    this.buffer = new Array(this.size);
    this.pointer = 0;
  }
}

// Usage
const metricsBuffer = new CircularBuffer<Metric>(1000);

wsClient.on('metric', (metric) => {
  metricsBuffer.push(metric);
  // Only keep last 1000 points
});
```

### 4. Code Splitting

```typescript
// Lazy load chart libraries
const LineChart = lazy(() => import('./Charts/LineChart'));
const BarChart = lazy(() => import('./Charts/BarChart'));
const PieChart = lazy(() => import('./Charts/PieChart'));

function MetricsGrid({ widgets }) {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      {widgets.map(widget => {
        switch (widget.type) {
          case 'line':
            return <LineChart key={widget.id} {...widget} />;
          case 'bar':
            return <BarChart key={widget.id} {...widget} />;
          case 'pie':
            return <PieChart key={widget.id} {...widget} />;
          default:
            return null;
        }
      })}
    </Suspense>
  );
}
```

---

## 🔐 6. Security Considerations

### 1. Authentication & Authorization

```typescript
// JWT with refresh token
class AuthService {
  private accessToken: string | null = null;
  private refreshToken: string | null = null;
  
  async login(email: string, password: string) {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    
    const { accessToken, refreshToken } = await response.json();
    
    this.accessToken = accessToken;
    this.refreshToken = refreshToken;
    
    // Store refresh token in httpOnly cookie
    // Access token in memory only (XSS protection)
  }
  
  async refreshAccessToken() {
    const response = await fetch('/api/auth/refresh', {
      method: 'POST',
      credentials: 'include', // Send httpOnly cookie
      headers: {
        'Content-Type': 'application/json'
      }
    });
    
    const { accessToken } = await response.json();
    this.accessToken = accessToken;
  }
  
  getAccessToken(): string | null {
    return this.accessToken;
  }
}

// WebSocket authentication
class SecureWebSocket {
  connect(url: string, token: string) {
    return new WebSocket(`${url}?token=${token}`);
  }
}
```

### 2. Rate Limiting

```typescript
// Client-side rate limiting
class RateLimiter {
  private requests: number[] = [];
  private limit: number;
  private window: number;
  
  constructor(limit: number, windowMs: number) {
    this.limit = limit;
    this.window = windowMs;
  }
  
  canMakeRequest(): boolean {
    const now = Date.now();
    
    // Remove old requests outside window
    this.requests = this.requests.filter(
      time => now - time < this.window
    );
    
    if (this.requests.length < this.limit) {
      this.requests.push(now);
      return true;
    }
    
    return false;
  }
}

// Usage
const limiter = new RateLimiter(100, 60000); // 100 requests per minute

function fetchMetrics() {
  if (!limiter.canMakeRequest()) {
    console.warn('Rate limit exceeded');
    return;
  }
  
  // Make request
}
```

---

## 🏗️ 7. Scalability Strategies

### 1. Server-Side Architecture

```
┌─────────────────────────────────────────┐
│         Load Balancer (Nginx)           │
│         - Sticky sessions               │
│         - WebSocket upgrade             │
└─────────────────────────────────────────┘
                  ↓
┌──────────────┬──────────────┬───────────┐
│   WS Node 1  │  WS Node 2   │ WS Node 3 │
│   (Socket.io)│  (Socket.io) │(Socket.io)│
└──────────────┴──────────────┴───────────┘
                  ↓
┌─────────────────────────────────────────┐
│         Redis Pub/Sub                   │
│         - Broadcast to all nodes        │
└─────────────────────────────────────────┘
```

**Socket.io with Redis:**
```javascript
// server.js
const io = require('socket.io')(server);
const redisAdapter = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

io.adapter(redisAdapter(pubClient, subClient));

// Broadcast works across all nodes
io.emit('metrics', data);
```

### 2. Data Partitioning

```typescript
// Partition metrics by time range
interface TimePartition {
  range: '1m' | '5m' | '1h' | '1d';
  retention: number; // days
  aggregation: 'avg' | 'sum' | 'min' | 'max';
}

const partitions: TimePartition[] = [
  { range: '1m', retention: 1, aggregation: 'avg' },   // Raw data, 1 day
  { range: '5m', retention: 7, aggregation: 'avg' },   // 5-min avg, 7 days
  { range: '1h', retention: 30, aggregation: 'avg' },  // 1-hour avg, 30 days
  { range: '1d', retention: 365, aggregation: 'avg' }  // Daily avg, 1 year
];

// Query appropriate partition based on date range
function getPartition(dateRange: DateRange): TimePartition {
  const durationDays = (dateRange.end - dateRange.start) / (1000 * 60 * 60 * 24);
  
  if (durationDays <= 1) return partitions[0]; // 1-min resolution
  if (durationDays <= 7) return partitions[1]; // 5-min resolution
  if (durationDays <= 30) return partitions[2]; // 1-hour resolution
  return partitions[3]; // Daily resolution
}
```

### 3. Caching Strategy

```typescript
// Multi-level caching
class MetricsCache {
  private memoryCache = new Map<string, CacheEntry>();
  private redisClient: Redis;
  
  async get(key: string): Promise<any> {
    // L1: Memory cache (fastest)
    const memoryEntry = this.memoryCache.get(key);
    if (memoryEntry && !this.isExpired(memoryEntry)) {
      return memoryEntry.data;
    }
    
    // L2: Redis cache (fast)
    const redisData = await this.redisClient.get(key);
    if (redisData) {
      const parsed = JSON.parse(redisData);
      this.memoryCache.set(key, {
        data: parsed,
        timestamp: Date.now()
      });
      return parsed;
    }
    
    // L3: Database (slow)
    const dbData = await this.fetchFromDB(key);
    
    // Update caches
    await this.redisClient.setex(key, 300, JSON.stringify(dbData)); // 5 min TTL
    this.memoryCache.set(key, {
      data: dbData,
      timestamp: Date.now()
    });
    
    return dbData;
  }
  
  private isExpired(entry: CacheEntry): boolean {
    return Date.now() - entry.timestamp > 60000; // 1 min TTL for memory
  }
}
```

---

## ❓ 8. Interview Discussion Points

### Q1: How would you handle connection loss?

**Answer:**

**1. Automatic Reconnection:**
```typescript
const reconnectStrategy = {
  maxRetries: 10,
  initialDelay: 1000,
  maxDelay: 30000,
  backoffFactor: 1.5
};

let retries = 0;

function reconnect() {
  if (retries >= reconnectStrategy.maxRetries) {
    showError('Unable to connect. Please refresh.');
    return;
  }
  
  const delay = Math.min(
    reconnectStrategy.initialDelay * Math.pow(reconnectStrategy.backoffFactor, retries),
    reconnectStrategy.maxDelay
  );
  
  setTimeout(() => {
    retries++;
    connect();
  }, delay);
}
```

**2. Fallback to Polling:**
```typescript
let pollingInterval: number;

function startPolling() {
  pollingInterval = window.setInterval(async () => {
    const data = await fetchMetrics();
    updateDashboard(data);
  }, 5000); // Poll every 5 seconds
}

wsClient.on('disconnect', () => {
  startPolling();
});

wsClient.on('connect', () => {
  clearInterval(pollingInterval);
});
```

**3. Queue Missed Updates:**
```typescript
const missedUpdates: Update[] = [];

wsClient.on('disconnect', () => {
  isConnected = false;
});

function handleUpdate(update: Update) {
  if (!isConnected) {
    missedUpdates.push(update);
  } else {
    applyUpdate(update);
  }
}

wsClient.on('connect', async () => {
  // Sync missed data
  const timestamp = missedUpdates[0]?.timestamp || lastUpdateTime;
  const catchupData = await fetchMissedData(timestamp);
  
  catchupData.forEach(applyUpdate);
  missedUpdates.length = 0;
});
```

---

### Q2: How would you optimize for 10,000 concurrent users?

**Answer:**

**1. Server-Side:**
- Horizontal scaling (multiple WebSocket servers)
- Redis pub/sub for cross-server communication
- Connection pooling
- Message batching (send multiple metrics in one message)

**2. Client-Side:**
- Data downsampling (reduce data points)
- Lazy loading (load widgets on demand)
- Virtual scrolling (render only visible widgets)
- Debounce/throttle updates

**3. Network:**
- CDN for static assets
- gzip/brotli compression
- HTTP/2 for multiplexing
- WebSocket compression

**4. Database:**
- Time-series database (TimescaleDB, InfluxDB)
- Data partitioning by time
- Pre-aggregated views
- Read replicas

---

### Q3: How would you handle real-time alerts?

**Answer:**

```typescript
// Alert system with severity levels
enum AlertSeverity {
  INFO = 'info',
  WARNING = 'warning',
  ERROR = 'error',
  CRITICAL = 'critical'
}

interface Alert {
  id: string;
  severity: AlertSeverity;
  metric: string;
  threshold: number;
  currentValue: number;
  message: string;
  timestamp: number;
}

class AlertManager {
  private alerts = new Map<string, Alert>();
  private listeners: ((alert: Alert) => void)[] = [];
  
  checkThreshold(metric: Metric, thresholds: Threshold[]) {
    thresholds.forEach(threshold => {
      const violated = this.isThresholdViolated(metric.value, threshold);
      
      if (violated && !this.alerts.has(metric.id)) {
        // New alert
        const alert: Alert = {
          id: generateId(),
          severity: threshold.severity,
          metric: metric.id,
          threshold: threshold.value,
          currentValue: metric.value,
          message: `${metric.label} exceeded threshold`,
          timestamp: Date.now()
        };
        
        this.alerts.set(metric.id, alert);
        this.notify(alert);
      } else if (!violated && this.alerts.has(metric.id)) {
        // Alert resolved
        this.alerts.delete(metric.id);
      }
    });
  }
  
  private notify(alert: Alert) {
    // Show toast notification
    showToast(alert);
    
    // Play sound (if critical)
    if (alert.severity === AlertSeverity.CRITICAL) {
      playAlertSound();
    }
    
    // Send to external system (Slack, PagerDuty)
    if (alert.severity === AlertSeverity.CRITICAL) {
      sendToSlack(alert);
    }
    
    // Notify listeners
    this.listeners.forEach(listener => listener(alert));
  }
  
  subscribe(listener: (alert: Alert) => void) {
    this.listeners.push(listener);
    return () => {
      this.listeners = this.listeners.filter(l => l !== listener);
    };
  }
}
```

---

## 🎯 Summary

**Key Architectural Decisions:**

1. **Data Flow:**
   - WebSocket for real-time (push)
   - REST API for historical (pull)
   - Polling as fallback

2. **State Management:**
   - Redux/Zustand for global state
   - React Query for server state
   - Local state for UI

3. **Performance:**
   - Data downsampling
   - Virtual rendering
   - Code splitting
   - Caching strategy

4. **Scalability:**
   - Horizontal scaling
   - Redis pub/sub
   - Data partitioning
   - CDN

5. **Reliability:**
   - Auto-reconnect
   - Error boundaries
   - Graceful degradation
   - Monitoring

**Trade-offs:**
- **WebSocket vs Polling:** Real-time latency vs complexity
- **Client-side vs Server-side aggregation:** Performance vs bandwidth
- **Data resolution vs Storage:** Detail vs cost
- **Caching vs Freshness:** Speed vs accuracy

This design can handle 10,000+ users with <100ms latency! 🚀
