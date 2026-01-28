# Web Workers in React

## Core Concept

Web Workers run JavaScript in background threads, keeping the main thread responsive by offloading heavy computations.

---

## Basic Worker Setup

```typescript
// worker.ts
self.onmessage = (e: MessageEvent) => {
  const { type, data } = e.data;

  if (type === 'PROCESS') {
    const result = heavyComputation(data);
    self.postMessage({ type: 'RESULT', result });
  }
};

function heavyComputation(data: number[]): number {
  // CPU-intensive work
  return data.reduce((sum, n) => sum + Math.sqrt(n), 0);
}

// React component
function App() {
  const [result, setResult] = useState<number | null>(null);
  const workerRef = useRef<Worker>();

  useEffect(() => {
    workerRef.current = new Worker(new URL('./worker.ts', import.meta.url));

    workerRef.current.onmessage = (e: MessageEvent) => {
      if (e.data.type === 'RESULT') {
        setResult(e.data.result);
      }
    };

    return () => workerRef.current?.terminate();
  }, []);

  const process = () => {
    const data = Array.from({ length: 1000000 }, (_, i) => i);
    workerRef.current?.postMessage({ type: 'PROCESS', data });
  };

  return (
    <div>
      <button onClick={process}>Process Data</button>
      {result && <div>Result: {result}</div>}
    </div>
  );
}
```

---

## Typed Worker Messages

```typescript
// worker-types.ts
export type WorkerRequest =
  | { type: "COMPUTE"; data: number[] }
  | { type: "SORT"; data: string[] }
  | { type: "FILTER"; data: number[]; threshold: number };

export type WorkerResponse =
  | { type: "COMPUTE_RESULT"; result: number }
  | { type: "SORT_RESULT"; result: string[] }
  | { type: "FILTER_RESULT"; result: number[] }
  | { type: "ERROR"; error: string };

// worker.ts
self.onmessage = (e: MessageEvent<WorkerRequest>) => {
  const request = e.data;

  try {
    switch (request.type) {
      case "COMPUTE":
        const sum = request.data.reduce((a, b) => a + b, 0);
        self.postMessage({ type: "COMPUTE_RESULT", result: sum });
        break;

      case "SORT":
        const sorted = request.data.sort();
        self.postMessage({ type: "SORT_RESULT", result: sorted });
        break;

      case "FILTER":
        const filtered = request.data.filter((n) => n > request.threshold);
        self.postMessage({ type: "FILTER_RESULT", result: filtered });
        break;
    }
  } catch (error) {
    self.postMessage({
      type: "ERROR",
      error: error instanceof Error ? error.message : "Unknown error",
    });
  }
};
```

---

## Custom Hook for Workers

```typescript
import { useEffect, useRef, useCallback } from 'react';

function useWorker<TRequest, TResponse>(
  workerUrl: string
): [(message: TRequest) => void, TResponse | null, boolean] {
  const [response, setResponse] = useState<TResponse | null>(null);
  const [loading, setLoading] = useState(false);
  const workerRef = useRef<Worker>();

  useEffect(() => {
    workerRef.current = new Worker(new URL(workerUrl, import.meta.url));

    workerRef.current.onmessage = (e: MessageEvent<TResponse>) => {
      setResponse(e.data);
      setLoading(false);
    };

    workerRef.current.onerror = (error) => {
      console.error('Worker error:', error);
      setLoading(false);
    };

    return () => workerRef.current?.terminate();
  }, [workerUrl]);

  const postMessage = useCallback((message: TRequest) => {
    setLoading(true);
    workerRef.current?.postMessage(message);
  }, []);

  return [postMessage, response, loading];
}

// Usage
function DataProcessor() {
  const [process, result, loading] = useWorker<WorkerRequest, WorkerResponse>(
    './worker.ts'
  );

  return (
    <div>
      <button onClick={() => process({ type: 'COMPUTE', data: [1, 2, 3] })}>
        Compute
      </button>
      {loading && <Spinner />}
      {result && <Result data={result} />}
    </div>
  );
}
```

---

## Comlink: Simplified Worker API

```typescript
// worker.ts
import { expose } from 'comlink';

const api = {
  async processData(data: number[]): Promise<number> {
    return data.reduce((sum, n) => sum + Math.sqrt(n), 0);
  },

  async sortData(data: string[]): Promise<string[]> {
    return data.sort();
  }
};

expose(api);

// React component
import { wrap } from 'comlink';

function App() {
  const [result, setResult] = useState<number | null>(null);
  const workerRef = useRef<any>();

  useEffect(() => {
    const worker = new Worker(new URL('./worker.ts', import.meta.url));
    workerRef.current = wrap(worker);

    return () => worker.terminate();
  }, []);

  const process = async () => {
    const data = [1, 2, 3, 4, 5];
    const result = await workerRef.current.processData(data);
    setResult(result);
  };

  return (
    <div>
      <button onClick={process}>Process</button>
      {result && <div>Result: {result}</div>}
    </div>
  );
}
```

---

## Real-World: Image Processing

```typescript
// image-worker.ts
self.onmessage = async (e: MessageEvent) => {
  const { imageData, filter } = e.data;

  const processed = applyFilter(imageData, filter);

  self.postMessage({ imageData: processed }, [processed.data.buffer]);
};

function applyFilter(imageData: ImageData, filter: string): ImageData {
  const data = imageData.data;

  for (let i = 0; i < data.length; i += 4) {
    switch (filter) {
      case 'grayscale':
        const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
        data[i] = data[i + 1] = data[i + 2] = avg;
        break;
      case 'invert':
        data[i] = 255 - data[i];
        data[i + 1] = 255 - data[i + 1];
        data[i + 2] = 255 - data[i + 2];
        break;
    }
  }

  return imageData;
}

// React component
function ImageEditor() {
  const [processedImage, setProcessedImage] = useState<string | null>(null);
  const workerRef = useRef<Worker>();

  useEffect(() => {
    workerRef.current = new Worker(new URL('./image-worker.ts', import.meta.url));

    workerRef.current.onmessage = (e: MessageEvent) => {
      const canvas = document.createElement('canvas');
      canvas.width = e.data.imageData.width;
      canvas.height = e.data.imageData.height;
      const ctx = canvas.getContext('2d');
      ctx?.putImageData(e.data.imageData, 0, 0);
      setProcessedImage(canvas.toDataURL());
    };

    return () => workerRef.current?.terminate();
  }, []);

  const applyFilter = (filter: string) => {
    const canvas = document.getElementById('canvas') as HTMLCanvasElement;
    const ctx = canvas.getContext('2d');
    const imageData = ctx?.getImageData(0, 0, canvas.width, canvas.height);

    workerRef.current?.postMessage({ imageData, filter });
  };

  return (
    <div>
      <button onClick={() => applyFilter('grayscale')}>Grayscale</button>
      <button onClick={() => applyFilter('invert')}>Invert</button>
      {processedImage && <img src={processedImage} alt="Processed" />}
    </div>
  );
}
```

---

## SharedArrayBuffer

Share memory between threads:

```typescript
// Main thread
const buffer = new SharedArrayBuffer(1024);
const view = new Int32Array(buffer);
view[0] = 42;

worker.postMessage({ buffer });

// Worker thread
self.onmessage = (e: MessageEvent) => {
  const view = new Int32Array(e.data.buffer);
  console.log(view[0]); // 42

  // Atomic operations for thread safety
  Atomics.add(view, 0, 10);
  Atomics.notify(view, 0);
};
```

---

## Worker Pool

Manage multiple workers:

```typescript
class WorkerPool {
  private workers: Worker[] = [];
  private queue: Array<{ data: any; resolve: (value: any) => void }> = [];

  constructor(workerUrl: string, size: number) {
    for (let i = 0; i < size; i++) {
      const worker = new Worker(workerUrl);
      worker.onmessage = (e) => this.handleMessage(worker, e);
      this.workers.push(worker);
    }
  }

  execute(data: any): Promise<any> {
    return new Promise((resolve) => {
      const worker = this.workers.find((w) => !this.isBusy(w));

      if (worker) {
        worker.postMessage(data);
        this.queue.push({ data, resolve });
      } else {
        this.queue.push({ data, resolve });
      }
    });
  }

  private handleMessage(worker: Worker, e: MessageEvent) {
    const task = this.queue.shift();
    if (task) {
      task.resolve(e.data);

      // Process next queued task
      if (this.queue.length > 0) {
        const next = this.queue.shift()!;
        worker.postMessage(next.data);
      }
    }
  }

  private isBusy(worker: Worker): boolean {
    // Check if worker has pending tasks
    return this.queue.some((task) => task.data.workerId === worker);
  }

  terminate() {
    this.workers.forEach((w) => w.terminate());
  }
}

// Usage
const pool = new WorkerPool("./worker.ts", 4);

async function processMany(items: number[]) {
  const promises = items.map((item) => pool.execute({ data: item }));
  const results = await Promise.all(promises);
  return results;
}
```

---

## Best Practices

✅ **Use for CPU-intensive tasks** (parsing, computation)  
✅ **Transfer data efficiently** with Transferable objects  
✅ **Terminate workers** when done  
✅ **Use worker pools** for many tasks  
✅ **Handle errors** gracefully  
❌ **Don't access DOM** from workers  
❌ **Don't use for small tasks** - overhead exists  
❌ **Don't share non-transferable objects**

---

## Transferable Objects

Zero-copy data transfer:

```typescript
// ❌ Slow - copies data
const data = new Uint8Array(1000000);
worker.postMessage({ data });

// ✅ Fast - transfers ownership
const data = new Uint8Array(1000000);
worker.postMessage({ data }, [data.buffer]);
// data is now unusable in main thread
```

---

## Key Takeaways

1. **Web Workers run JavaScript in background threads**
2. **Use for CPU-intensive computations**
3. **Workers can't access DOM**
4. **Use Transferable objects** for efficient data transfer
5. **Comlink simplifies** worker communication
6. **Worker pools** manage multiple workers efficiently
7. **Always terminate workers** to free resources
