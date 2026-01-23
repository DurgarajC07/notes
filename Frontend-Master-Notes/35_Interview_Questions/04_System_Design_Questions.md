# 🏗️ System Design Interview Questions (Frontend)

> Frontend system design questions for Senior/Staff engineers. Focus on scalability, performance, architecture, and trade-offs.

---

## 🎯 Part 1: Large-Scale Applications

### Q1: Design a News Feed (like Twitter/Facebook)

**Requirements:**

- 100M daily active users
- Real-time updates
- Infinite scroll
- Media (images, videos)
- Responsive on mobile/desktop

**Answer:**

**1. High-Level Architecture**

```
┌─────────────────────────────────────────────────┐
│              Client (React SPA)                  │
│  - Virtual scrolling (react-window)             │
│  - Optimistic updates                           │
│  - WebSocket for real-time                      │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│              CDN (Cloudflare)                    │
│  - Static assets (JS/CSS/Images)                │
│  - Edge caching                                 │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│          Load Balancer (Nginx/AWS ALB)          │
└─────────────────────────────────────────────────┘
                      ↓
┌──────────────────┬──────────────────┬───────────┐
│   API Gateway    │  WebSocket Server│  GraphQL  │
└──────────────────┴──────────────────┴───────────┘
                      ↓
┌──────────────────────────────────────────────────┐
│              Microservices                       │
│  - Feed Service                                  │
│  - Post Service                                  │
│  - Media Service                                 │
│  - Notification Service                          │
└──────────────────────────────────────────────────┘
                      ↓
┌─────────────┬─────────────┬─────────────────────┐
│  PostgreSQL │    Redis    │    S3/CloudFront    │
│  (Posts)    │  (Cache)    │    (Media)          │
└─────────────┴─────────────┴─────────────────────┘
```

**2. Frontend Implementation**

```tsx
// Feed.tsx
import { useInfiniteQuery } from "@tanstack/react-query";
import { FixedSizeList } from "react-window";
import InfiniteLoader from "react-window-infinite-loader";

function Feed() {
  const { data, fetchNextPage, hasNextPage, isLoading } = useInfiniteQuery({
    queryKey: ["feed"],
    queryFn: ({ pageParam = 0 }) => fetchFeed(pageParam),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    staleTime: 60000, // 1 minute
  });

  const posts = data?.pages.flatMap((page) => page.posts) ?? [];

  // Virtual scrolling for performance
  return (
    <InfiniteLoader
      isItemLoaded={(index) => index < posts.length}
      itemCount={hasNextPage ? posts.length + 1 : posts.length}
      loadMoreItems={fetchNextPage}
    >
      {({ onItemsRendered, ref }) => (
        <FixedSizeList
          height={window.innerHeight}
          itemCount={posts.length}
          itemSize={400}
          onItemsRendered={onItemsRendered}
          ref={ref}
        >
          {({ index, style }) => (
            <div style={style}>
              <Post post={posts[index]} />
            </div>
          )}
        </FixedSizeList>
      )}
    </InfiniteLoader>
  );
}
```

**3. Real-Time Updates (WebSocket)**

```tsx
// useRealtimeFeed.ts
function useRealtimeFeed() {
  const queryClient = useQueryClient();

  useEffect(() => {
    const ws = new WebSocket("wss://api.example.com/feed");

    ws.onmessage = (event) => {
      const newPost = JSON.parse(event.data);

      // Optimistic update
      queryClient.setQueryData(["feed"], (old) => {
        if (!old) return old;

        return {
          ...old,
          pages: [
            { posts: [newPost, ...old.pages[0].posts] },
            ...old.pages.slice(1),
          ],
        };
      });
    };

    return () => ws.close();
  }, []);
}
```

**4. Optimizations**

- **Virtualization**: Only render visible posts (~20) instead of all (1000+)
- **Image lazy loading**: `loading="lazy"` or Intersection Observer
- **Code splitting**: Lazy load comment section, reactions
- **Prefetching**: Prefetch next page when user scrolls to 80%
- **Caching**: React Query with stale-while-revalidate
- **CDN**: Static assets served from edge locations

**5. Scalability Considerations**

- **Database sharding**: Partition by user_id or time
- **Read replicas**: Scale reads horizontally
- **Cache layer**: Redis for hot data (recent posts)
- **Message queue**: Kafka/RabbitMQ for async processing
- **Rate limiting**: Prevent abuse, protect infrastructure

---

### Q2: Design a Real-Time Collaborative Editor (like Google Docs)

**Requirements:**

- Multiple users editing simultaneously
- See cursors/selections of other users
- Conflict resolution
- Offline support
- Version history

**Answer:**

**1. Architecture**

```
┌────────────────────────────────────────────────┐
│           Client (React + Editor)              │
│  - Monaco Editor / Quill                       │
│  - Operational Transformation (OT)             │
│  - WebSocket for real-time sync                │
│  - IndexedDB for offline                       │
└────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────┐
│          WebSocket Server (Node.js)            │
│  - Broadcast changes to all clients            │
│  - Conflict resolution                         │
│  - Presence (who's online)                     │
└────────────────────────────────────────────────┘
                     ↓
┌─────────────┬──────────────┬──────────────────┐
│  Redis Pub/ │  PostgreSQL  │   S3             │
│  Sub        │  (Versions)  │  (Snapshots)     │
└─────────────┴──────────────┴──────────────────┘
```

**2. Operational Transformation (OT)**

```typescript
// operation.ts
type Operation =
  | { type: "insert"; position: number; text: string }
  | { type: "delete"; position: number; length: number };

// Transform operations for concurrent editing
function transform(op1: Operation, op2: Operation): Operation {
  if (op1.type === "insert" && op2.type === "insert") {
    if (op1.position < op2.position) {
      return op2; // No change needed
    } else {
      // Adjust position
      return { ...op2, position: op2.position + op1.text.length };
    }
  }

  if (op1.type === "delete" && op2.type === "insert") {
    if (op2.position <= op1.position) {
      return { ...op1, position: op1.position + op2.text.length };
    } else if (op2.position <= op1.position + op1.length) {
      return { ...op1, length: op1.length + op2.text.length };
    }
  }

  // More cases...
  return op2;
}
```

**3. Real-Time Sync**

```typescript
// CollaborativeEditor.tsx
function CollaborativeEditor({ documentId }: { documentId: string }) {
  const [content, setContent] = useState('');
  const [users, setUsers] = useState<User[]>([]);
  const wsRef = useRef<WebSocket>();
  const editorRef = useRef<Monaco.editor.IStandaloneCodeEditor>();

  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/docs/${documentId}`);
    wsRef.current = ws;

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);

      switch (message.type) {
        case 'operation':
          applyOperation(message.operation);
          break;
        case 'cursor':
          updateCursor(message.userId, message.cursor);
          break;
        case 'presence':
          setUsers(message.users);
          break;
      }
    };

    return () => ws.close();
  }, [documentId]);

  function handleChange(newContent: string) {
    const operation = generateOperation(content, newContent);

    // Optimistic update
    setContent(newContent);

    // Send to server
    wsRef.current?.send(JSON.stringify({
      type: 'operation',
      operation
    }));
  }

  return (
    <div>
      <UserPresence users={users} />
      <MonacoEditor
        value={content}
        onChange={handleChange}
        onDidChangeCursorPosition={(e) => {
          wsRef.current?.send(JSON.stringify({
            type: 'cursor',
            position: e.position
          }));
        }}
      />
    </div>
  );
}
```

**4. Conflict Resolution**

- **Operational Transformation**: Transform concurrent operations
- **CRDTs**: Conflict-free Replicated Data Types (Yjs, Automerge)
- **Last-write-wins**: Simple but loses data
- **Vector clocks**: Track causality

**5. Offline Support**

```typescript
// offlineSync.ts
class OfflineSync {
  private db: IDBDatabase;
  private pendingOps: Operation[] = [];

  async syncWhenOnline() {
    window.addEventListener("online", async () => {
      // Replay pending operations
      for (const op of this.pendingOps) {
        await this.sendOperation(op);
      }
      this.pendingOps = [];
    });
  }

  async handleOfflineEdit(op: Operation) {
    // Store locally
    await this.db.put("operations", op);
    this.pendingOps.push(op);
  }
}
```

---

### Q3: Design a Video Streaming Platform (like YouTube)

**Requirements:**

- Upload videos (up to 4K)
- Transcode to multiple qualities
- Adaptive bitrate streaming
- Recommendations
- Comments, likes
- Analytics

**Answer:**

**1. Architecture**

```
┌────────────────────────────────────────────────┐
│          Client (React + Video.js)             │
│  - HLS/DASH player                             │
│  - Quality selector                            │
│  - Buffering optimization                      │
└────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────┐
│              CDN (CloudFront)                   │
│  - Video segments (.m3u8, .ts)                 │
│  - Edge caching                                │
│  - Geo-distribution                            │
└────────────────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────┐
│          Origin Server (S3 + Lambda)           │
│  - Video files                                 │
│  - Thumbnail generation                        │
└────────────────────────────────────────────────┘
                     ↓
┌─────────────────┬──────────────┬───────────────┐
│  Transcoding    │  Metadata    │  Analytics    │
│  (AWS Elastic   │  (DynamoDB)  │  (BigQuery)   │
│   Transcoder)   │              │               │
└─────────────────┴──────────────┴───────────────┘
```

**2. Video Upload Flow**

```typescript
// VideoUpload.tsx
function VideoUpload() {
  const [progress, setProgress] = useState(0);

  async function handleUpload(file: File) {
    // 1. Get presigned URL
    const { uploadUrl, videoId } = await fetch('/api/videos/upload-url', {
      method: 'POST',
      body: JSON.stringify({
        filename: file.name,
        filesize: file.size,
        mimeType: file.type
      })
    }).then(r => r.json());

    // 2. Upload to S3 with multipart for large files
    if (file.size > 100 * 1024 * 1024) { // >100MB
      await multipartUpload(file, uploadUrl, setProgress);
    } else {
      await fetch(uploadUrl, {
        method: 'PUT',
        body: file
      });
    }

    // 3. Trigger transcoding
    await fetch(`/api/videos/${videoId}/transcode`, {
      method: 'POST'
    });

    return videoId;
  }

  return (
    <div>
      <input
        type="file"
        accept="video/*"
        onChange={(e) => {
          const file = e.target.files?.[0];
          if (file) handleUpload(file);
        }}
      />
      {progress > 0 && <ProgressBar progress={progress} />}
    </div>
  );
}

// Multipart upload for large files
async function multipartUpload(
  file: File,
  uploadUrl: string,
  onProgress: (progress: number) => void
) {
  const chunkSize = 10 * 1024 * 1024; // 10MB chunks
  const chunks = Math.ceil(file.size / chunkSize);

  for (let i = 0; i < chunks; i++) {
    const start = i * chunkSize;
    const end = Math.min(start + chunkSize, file.size);
    const chunk = file.slice(start, end);

    await fetch(`${uploadUrl}&partNumber=${i + 1}`, {
      method: 'PUT',
      body: chunk
    });

    onProgress((i + 1) / chunks * 100);
  }
}
```

**3. Adaptive Bitrate Player**

```typescript
// VideoPlayer.tsx
import videojs from 'video.js';
import 'video.js/dist/video-js.css';

function VideoPlayer({ videoId }: { videoId: string }) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const playerRef = useRef<ReturnType<typeof videojs>>();

  useEffect(() => {
    if (!videoRef.current) return;

    const player = videojs(videoRef.current, {
      controls: true,
      autoplay: false,
      preload: 'auto',
      fluid: true,
      sources: [{
        // HLS manifest with multiple qualities
        src: `https://cdn.example.com/videos/${videoId}/master.m3u8`,
        type: 'application/x-mpegURL'
      }],
      html5: {
        vhs: {
          // Adaptive bitrate settings
          bandwidth: 4000000,
          enableLowInitialPlaylist: true,
          smoothQualityChange: true
        }
      }
    });

    playerRef.current = player;

    // Analytics
    player.on('play', () => trackEvent('video_play'));
    player.on('pause', () => trackEvent('video_pause'));
    player.on('ended', () => trackEvent('video_complete'));

    return () => {
      player.dispose();
    };
  }, [videoId]);

  return (
    <div data-vjs-player>
      <video ref={videoRef} className="video-js" />
    </div>
  );
}
```

**4. Transcoding Pipeline**

```typescript
// Lambda function triggered on upload
async function transcodeVideo(event: S3Event) {
  const bucket = event.Records[0].s3.bucket.name;
  const key = event.Records[0].s3.object.key;

  // Create transcoding job
  const job = await elasticTranscoder.createJob({
    PipelineId: "pipeline-id",
    Input: {
      Key: key,
      Container: "auto",
    },
    Outputs: [
      // 4K (2160p)
      { Key: `${key}/2160p.mp4`, PresetId: "4k-preset" },
      // 1080p
      { Key: `${key}/1080p.mp4`, PresetId: "1080p-preset" },
      // 720p
      { Key: `${key}/720p.mp4`, PresetId: "720p-preset" },
      // 480p
      { Key: `${key}/480p.mp4`, PresetId: "480p-preset" },
      // 360p
      { Key: `${key}/360p.mp4`, PresetId: "360p-preset" },
    ],
    // Generate HLS playlist
    Playlists: [
      {
        Name: "master",
        Format: "HLSv3",
        OutputKeys: [
          `${key}/2160p.mp4`,
          `${key}/1080p.mp4`,
          `${key}/720p.mp4`,
          `${key}/480p.mp4`,
          `${key}/360p.mp4`,
        ],
      },
    ],
  });

  return job;
}
```

**5. Optimizations**

- **CDN**: Serve videos from edge locations (low latency)
- **Buffering**: Prefetch next segments
- **Quality switching**: Adapt based on bandwidth
- **Thumbnail sprites**: VTT files for preview on hover
- **Lazy loading**: Load player only when visible
- **Compression**: H.265 (better quality, smaller size)

---

## 🎯 Part 2: Performance & Scalability

### Q4: How would you optimize a slow-loading dashboard?

**Answer:**

**1. Profiling (Measure First)**

```typescript
// Performance monitoring
import { onLCP, onFID, onCLS } from 'web-vitals';

onLCP(console.log); // Largest Contentful Paint
onFID(console.log); // First Input Delay
onCLS(console.log); // Cumulative Layout Shift

// React Profiler
import { Profiler } from 'react';

<Profiler id="Dashboard" onRender={(id, phase, actualDuration) => {
  console.log(`${id} took ${actualDuration}ms`);
}}>
  <Dashboard />
</Profiler>
```

**2. Code Splitting**

```typescript
// Split heavy components
const Charts = lazy(() => import('./Charts'));
const DataTable = lazy(() => import('./DataTable'));
const Reports = lazy(() => import('./Reports'));

function Dashboard() {
  return (
    <Suspense fallback={<Skeleton />}>
      <Charts />
      <DataTable />
      <Reports />
    </Suspense>
  );
}
```

**3. Data Fetching Optimization**

```typescript
// ❌ Serial requests (slow)
const users = await fetch("/api/users");
const orders = await fetch("/api/orders");
const revenue = await fetch("/api/revenue");

// ✅ Parallel requests
const [users, orders, revenue] = await Promise.all([
  fetch("/api/users"),
  fetch("/api/orders"),
  fetch("/api/revenue"),
]);

// ✅ React Query with caching
const { data: users } = useQuery({
  queryKey: ["users"],
  queryFn: fetchUsers,
  staleTime: 5 * 60 * 1000, // 5 minutes
});
```

**4. Virtualization for Large Lists**

```typescript
import { FixedSizeList } from 'react-window';

function OrderList({ orders }: { orders: Order[] }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={orders.length}
      itemSize={80}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          <OrderRow order={orders[index]} />
        </div>
      )}
    </FixedSizeList>
  );
}
```

**5. Memoization**

```typescript
// Expensive chart calculations
const chartData = useMemo(() => {
  return processChartData(rawData);
}, [rawData]);

// Prevent re-renders
const MemoizedChart = memo(Chart);
```

**6. Bundle Optimization**

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: "all",
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendors",
          priority: 10,
        },
        charts: {
          test: /[\\/]node_modules[\\/](recharts|chart\.js)/,
          name: "charts",
          priority: 20,
        },
      },
    },
  },
};
```

**7. Results**

| Metric              | Before | After | Improvement     |
| ------------------- | ------ | ----- | --------------- |
| Initial Load        | 8.5s   | 2.1s  | **75% faster**  |
| Bundle Size         | 2.4MB  | 800KB | **67% smaller** |
| Time to Interactive | 12s    | 3.5s  | **71% faster**  |
| Lighthouse Score    | 42     | 94    | **+52 points**  |

---

### Q5: Design a rate limiter for API requests

**Answer:**

**1. Client-Side Rate Limiting**

```typescript
class RateLimiter {
  private queue: (() => Promise<any>)[] = [];
  private running = 0;

  constructor(
    private maxConcurrent: number = 5,
    private requestsPerSecond: number = 10,
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // Wait if over limit
    while (this.running >= this.maxConcurrent) {
      await new Promise((resolve) => setTimeout(resolve, 100));
    }

    this.running++;

    try {
      const result = await fn();
      return result;
    } finally {
      this.running--;

      // Enforce requests per second
      await new Promise((resolve) =>
        setTimeout(resolve, 1000 / this.requestsPerSecond),
      );
    }
  }
}

// Usage
const limiter = new RateLimiter(5, 10);

async function fetchData(url: string) {
  return limiter.execute(() => fetch(url));
}
```

**2. Token Bucket Algorithm**

```typescript
class TokenBucket {
  private tokens: number;
  private lastRefill: number;

  constructor(
    private capacity: number,
    private refillRate: number, // tokens per second
  ) {
    this.tokens = capacity;
    this.lastRefill = Date.now();
  }

  private refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    const tokensToAdd = elapsed * this.refillRate;

    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }

  async consume(tokens: number = 1): Promise<boolean> {
    this.refill();

    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }

    // Wait for tokens
    const waitTime = ((tokens - this.tokens) / this.refillRate) * 1000;
    await new Promise((resolve) => setTimeout(resolve, waitTime));

    return this.consume(tokens);
  }
}

// Usage
const bucket = new TokenBucket(100, 10); // 100 tokens, refill 10/sec

async function makeRequest(url: string) {
  await bucket.consume(1);
  return fetch(url);
}
```

**3. Server-Side (Express Middleware)**

```typescript
import rateLimit from "express-rate-limit";
import RedisStore from "rate-limit-redis";

const limiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Max 100 requests per window
  message: "Too many requests, please try again later",
  standardHeaders: true, // Return rate limit info in headers
  legacyHeaders: false,
});

app.use("/api/", limiter);
```

---

## 🎯 Summary

**Key patterns:**

- ✅ Virtual scrolling for large lists
- ✅ WebSocket for real-time updates
- ✅ React Query for data caching
- ✅ Code splitting for faster loads
- ✅ CDN for static assets
- ✅ Rate limiting to prevent abuse

**Architecture principles:**

- Horizontal scaling with load balancers
- Caching layers (Redis, CDN)
- Microservices for separation of concerns
- Message queues for async processing
- Monitoring and analytics

Practice designing systems end-to-end! 🏗️
