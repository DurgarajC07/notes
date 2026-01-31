# Memory Optimization

## Core Concepts

Memory optimization is crucial for building performant web applications that remain responsive over time. Memory leaks occur when allocated memory is not properly released, causing gradual performance degradation. Understanding JavaScript's garbage collection, heap management, and memory lifecycle is essential for preventing leaks and optimizing memory usage.

### Key Memory Concepts

- **Heap Memory**: Memory area where objects are stored
- **Stack Memory**: Memory for function execution contexts and primitive values
- **Garbage Collection**: Automatic memory reclamation for unreachable objects
- **Memory Leak**: Unintended memory retention preventing garbage collection
- **Detached DOM**: DOM nodes removed from document but still referenced in memory
- **Shallow Size**: Direct memory used by an object
- **Retained Size**: Total memory freed when object is garbage collected

## TypeScript/JavaScript Code Examples

### Example 1: Event Listener Memory Leak Prevention

```typescript
// Proper event listener management to prevent leaks
class EventManager {
  private listeners: Map<EventTarget, Map<string, EventListener[]>> = new Map();
  private abortControllers: Map<EventTarget, AbortController> = new Map();

  // Add event listener with automatic cleanup tracking
  addEventListener<K extends keyof HTMLElementEventMap>(
    target: EventTarget,
    type: K,
    listener: (evt: HTMLElementEventMap[K]) => void,
    options?: AddEventListenerOptions,
  ): void {
    // Track listeners for cleanup
    if (!this.listeners.has(target)) {
      this.listeners.set(target, new Map());
    }

    const targetListeners = this.listeners.get(target)!;
    if (!targetListeners.has(type)) {
      targetListeners.set(type, []);
    }

    targetListeners.get(type)!.push(listener as EventListener);

    // Use AbortController for easy cleanup
    if (!this.abortControllers.has(target)) {
      this.abortControllers.set(target, new AbortController());
    }

    const controller = this.abortControllers.get(target)!;
    target.addEventListener(type, listener as EventListener, {
      ...options,
      signal: controller.signal,
    });
  }

  // Remove specific event listener
  removeEventListener(
    target: EventTarget,
    type: string,
    listener: EventListener,
  ): void {
    const targetListeners = this.listeners.get(target);
    if (!targetListeners) return;

    const typeListeners = targetListeners.get(type);
    if (!typeListeners) return;

    const index = typeListeners.indexOf(listener);
    if (index > -1) {
      typeListeners.splice(index, 1);
      target.removeEventListener(type, listener);
    }

    // Cleanup empty maps
    if (typeListeners.length === 0) {
      targetListeners.delete(type);
    }
    if (targetListeners.size === 0) {
      this.listeners.delete(target);
    }
  }

  // Remove all listeners for a target
  removeAllListeners(target: EventTarget): void {
    const controller = this.abortControllers.get(target);
    if (controller) {
      controller.abort();
      this.abortControllers.delete(target);
    }

    this.listeners.delete(target);
  }

  // Clean up all listeners
  cleanup(): void {
    for (const controller of this.abortControllers.values()) {
      controller.abort();
    }
    this.abortControllers.clear();
    this.listeners.clear();
  }

  // Get listener count for debugging
  getListenerCount(target?: EventTarget): number {
    if (target) {
      const targetListeners = this.listeners.get(target);
      if (!targetListeners) return 0;

      return Array.from(targetListeners.values()).reduce(
        (sum, listeners) => sum + listeners.length,
        0,
      );
    }

    let total = 0;
    for (const targetListeners of this.listeners.values()) {
      for (const listeners of targetListeners.values()) {
        total += listeners.length;
      }
    }
    return total;
  }
}

// Usage in component
class Component {
  private eventManager = new EventManager();
  private element: HTMLElement;

  constructor(element: HTMLElement) {
    this.element = element;
    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    // Automatically tracked and cleaned up
    this.eventManager.addEventListener(
      this.element,
      "click",
      this.handleClick.bind(this),
    );

    this.eventManager.addEventListener(
      window,
      "resize",
      this.handleResize.bind(this),
    );

    this.eventManager.addEventListener(
      document,
      "scroll",
      this.handleScroll.bind(this),
      { passive: true },
    );
  }

  private handleClick(evt: MouseEvent): void {
    console.log("Click:", evt);
  }

  private handleResize(evt: Event): void {
    console.log("Resize:", evt);
  }

  private handleScroll(evt: Event): void {
    console.log("Scroll:", evt);
  }

  // Clean up when component is destroyed
  destroy(): void {
    this.eventManager.cleanup();
    console.log("All listeners cleaned up");
  }
}

// Example: Avoiding common leak
// ❌ BAD - Creates memory leak
class BadComponent {
  private element: HTMLElement;

  constructor(element: HTMLElement) {
    this.element = element;
    // Listener never removed, creates leak
    element.addEventListener("click", () => {
      console.log("Clicked");
    });
  }
}

// ✅ GOOD - Properly managed
class GoodComponent {
  private eventManager = new EventManager();
  private element: HTMLElement;

  constructor(element: HTMLElement) {
    this.element = element;
    this.eventManager.addEventListener(element, "click", () => {
      console.log("Clicked");
    });
  }

  destroy(): void {
    this.eventManager.cleanup();
  }
}
```

### Example 2: Timer Management System

```typescript
// Prevent timer-related memory leaks
class TimerManager {
  private timers: Map<string, number> = new Map();
  private intervals: Map<string, number> = new Map();
  private rafIds: Map<string, number> = new Map();

  // Set timeout with automatic tracking
  setTimeout(id: string, callback: () => void, delay: number): void {
    // Clear existing timer with same ID
    this.clearTimeout(id);

    const timerId = window.setTimeout(() => {
      callback();
      this.timers.delete(id);
    }, delay);

    this.timers.set(id, timerId);
  }

  // Set interval with automatic tracking
  setInterval(id: string, callback: () => void, interval: number): void {
    this.clearInterval(id);

    const intervalId = window.setInterval(callback, interval);
    this.intervals.set(id, intervalId);
  }

  // Request animation frame with tracking
  requestAnimationFrame(id: string, callback: () => void): void {
    this.cancelAnimationFrame(id);

    const rafId = window.requestAnimationFrame(() => {
      callback();
      this.rafIds.delete(id);
    });

    this.rafIds.set(id, rafId);
  }

  // Clear specific timeout
  clearTimeout(id: string): void {
    const timerId = this.timers.get(id);
    if (timerId !== undefined) {
      window.clearTimeout(timerId);
      this.timers.delete(id);
    }
  }

  // Clear specific interval
  clearInterval(id: string): void {
    const intervalId = this.intervals.get(id);
    if (intervalId !== undefined) {
      window.clearInterval(intervalId);
      this.intervals.delete(id);
    }
  }

  // Cancel specific animation frame
  cancelAnimationFrame(id: string): void {
    const rafId = this.rafIds.get(id);
    if (rafId !== undefined) {
      window.cancelAnimationFrame(rafId);
      this.rafIds.delete(id);
    }
  }

  // Clear all timers
  clearAll(): void {
    for (const timerId of this.timers.values()) {
      window.clearTimeout(timerId);
    }
    this.timers.clear();

    for (const intervalId of this.intervals.values()) {
      window.clearInterval(intervalId);
    }
    this.intervals.clear();

    for (const rafId of this.rafIds.values()) {
      window.cancelAnimationFrame(rafId);
    }
    this.rafIds.clear();
  }

  // Get active timer count
  getActiveCount(): { timeouts: number; intervals: number; rafs: number } {
    return {
      timeouts: this.timers.size,
      intervals: this.intervals.size,
      rafs: this.rafIds.size,
    };
  }

  // Check if timer exists
  hasTimer(id: string): boolean {
    return this.timers.has(id) || this.intervals.has(id) || this.rafIds.has(id);
  }
}

// Usage in application
class DataPolling {
  private timerManager = new TimerManager();
  private isActive = false;

  start(): void {
    this.isActive = true;
    this.poll();
  }

  private poll(): void {
    if (!this.isActive) return;

    // Fetch data
    this.fetchData()
      .then((data) => this.processData(data))
      .catch((err) => console.error("Poll error:", err))
      .finally(() => {
        // Schedule next poll if still active
        if (this.isActive) {
          this.timerManager.setTimeout("poll", () => this.poll(), 5000);
        }
      });
  }

  private async fetchData(): Promise<any> {
    const response = await fetch("/api/data");
    return response.json();
  }

  private processData(data: any): void {
    console.log("Data:", data);
  }

  stop(): void {
    this.isActive = false;
    this.timerManager.clearAll();
  }
}

// Animation manager
class AnimationController {
  private timerManager = new TimerManager();
  private element: HTMLElement;

  constructor(element: HTMLElement) {
    this.element = element;
  }

  animate(): void {
    let progress = 0;

    const step = () => {
      progress += 0.01;
      this.element.style.opacity = String(progress);

      if (progress < 1) {
        this.timerManager.requestAnimationFrame("fade", step);
      }
    };

    this.timerManager.requestAnimationFrame("fade", step);
  }

  stop(): void {
    this.timerManager.clearAll();
  }
}
```

### Example 3: WeakMap and WeakSet for Memory Management

```typescript
// Use WeakMap/WeakSet to prevent memory leaks
class ComponentRegistry {
  // WeakMap allows garbage collection of components
  private componentData = new WeakMap<HTMLElement, ComponentData>();
  private activeComponents = new WeakSet<HTMLElement>();

  // Register component with metadata
  register(element: HTMLElement, data: ComponentData): void {
    this.componentData.set(element, data);
    this.activeComponents.add(element);
  }

  // Get component data
  getData(element: HTMLElement): ComponentData | undefined {
    return this.componentData.get(element);
  }

  // Check if component is active
  isActive(element: HTMLElement): boolean {
    return this.activeComponents.has(element);
  }

  // Element is automatically garbage collected when removed from DOM
  // No manual cleanup needed!
}

interface ComponentData {
  id: string;
  type: string;
  props: Record<string, any>;
  createdAt: Date;
}

// Cache with automatic memory management
class CacheManager<K extends object, V> {
  private cache = new WeakMap<K, CachedValue<V>>();

  set(key: K, value: V, ttl?: number): void {
    this.cache.set(key, {
      value,
      timestamp: Date.now(),
      ttl,
    });
  }

  get(key: K): V | undefined {
    const cached = this.cache.get(key);
    if (!cached) return undefined;

    // Check if expired
    if (cached.ttl && Date.now() - cached.timestamp > cached.ttl) {
      return undefined;
    }

    return cached.value;
  }

  has(key: K): boolean {
    return this.cache.get(key) !== undefined;
  }
}

interface CachedValue<V> {
  value: V;
  timestamp: number;
  ttl?: number;
}

// Metadata storage using WeakMap
class ElementMetadata {
  private metadata = new WeakMap<Element, Map<string, any>>();

  set(element: Element, key: string, value: any): void {
    if (!this.metadata.has(element)) {
      this.metadata.set(element, new Map());
    }
    this.metadata.get(element)!.set(key, value);
  }

  get(element: Element, key: string): any {
    return this.metadata.get(element)?.get(key);
  }

  delete(element: Element, key: string): void {
    this.metadata.get(element)?.delete(key);
  }

  clear(element: Element): void {
    this.metadata.delete(element);
  }
}

// Usage example
const registry = new ComponentRegistry();
const metadata = new ElementMetadata();

function initializeComponent(element: HTMLElement): void {
  // Register with metadata
  registry.register(element, {
    id: generateId(),
    type: "button",
    props: { variant: "primary" },
    createdAt: new Date(),
  });

  // Store additional metadata
  metadata.set(element, "clicks", 0);
  metadata.set(element, "lastClickTime", null);

  // Event listener
  element.addEventListener("click", () => {
    const clicks = metadata.get(element, "clicks") + 1;
    metadata.set(element, "clicks", clicks);
    metadata.set(element, "lastClickTime", Date.now());
  });
}

// When element is removed from DOM and no longer referenced,
// it will be garbage collected along with all metadata
function removeComponent(element: HTMLElement): void {
  element.remove();
  // No manual cleanup needed! WeakMap/WeakSet handle it automatically
}

function generateId(): string {
  return Math.random().toString(36).substring(2);
}
```

### Example 4: Closure Memory Leak Prevention

```typescript
// Avoid memory leaks from closures
// ❌ BAD - Creates memory leak
class BadDataLoader {
  private largeData: any[] = [];

  loadData(): void {
    // Fetch large dataset
    this.largeData = new Array(10000).fill({
      /* large object */
    });

    // Closure captures entire 'this', including largeData
    document.getElementById("btn")?.addEventListener("click", () => {
      console.log(this.largeData.length); // Keeps entire array in memory
    });
  }
}

// ✅ GOOD - No memory leak
class GoodDataLoader {
  private largeData: any[] = [];

  loadData(): void {
    this.largeData = new Array(10000).fill({
      /* large object */
    });

    // Only capture what's needed
    const dataLength = this.largeData.length;

    document.getElementById("btn")?.addEventListener("click", () => {
      console.log(dataLength); // Only captures number, not entire array
    });

    // Clear reference when done
    this.largeData = [];
  }
}

// Pattern: Extract closure data
class DataProcessor {
  processLargeDataset(data: any[]): void {
    // Process data
    const summary = this.createSummary(data);

    // Don't capture entire dataset in closure
    // Only capture summary
    this.setupUI(summary);

    // Clear data reference
    data = [];
  }

  private createSummary(data: any[]): DataSummary {
    return {
      count: data.length,
      sum: data.reduce((acc, val) => acc + val, 0),
      average: data.reduce((acc, val) => acc + val, 0) / data.length,
    };
  }

  private setupUI(summary: DataSummary): void {
    // Closure only captures summary, not full dataset
    document.getElementById("display")?.addEventListener("click", () => {
      console.log(`Count: ${summary.count}, Average: ${summary.average}`);
    });
  }
}

interface DataSummary {
  count: number;
  sum: number;
  average: number;
}

// Pattern: Use function factory to limit closure scope
function createEventHandler(id: string, callback: () => void) {
  // Limited closure scope
  return () => {
    console.log(`Handler ${id} called`);
    callback();
  };
}

class ComponentWithHandlers {
  private largeState: any[] = [];

  setupHandlers(): void {
    this.largeState = new Array(10000).fill({});

    // Handler only captures ID, not entire state
    const handler = createEventHandler("btn1", () => {
      console.log("Button clicked");
    });

    document.getElementById("btn")?.addEventListener("click", handler);

    // State can be garbage collected
    this.largeState = [];
  }
}
```

### Example 5: Detached DOM Node Detection and Cleanup

```typescript
// Detect and prevent detached DOM nodes
class DOMManager {
  private attachedNodes = new WeakSet<Node>();
  private nodeReferences = new Map<string, Element>();

  // Track node attachment
  attachNode(id: string, node: Element): void {
    this.nodeReferences.set(id, node);
    this.attachedNodes.add(node);
  }

  // Remove node properly
  removeNode(id: string): void {
    const node = this.nodeReferences.get(id);
    if (node) {
      // Remove from DOM
      node.remove();

      // Clear event listeners
      this.clearEventListeners(node);

      // Remove from reference map
      this.nodeReferences.delete(id);
    }
  }

  private clearEventListeners(node: Element): void {
    // Clone and replace to remove all listeners
    const clone = node.cloneNode(true);
    node.replaceWith(clone);
  }

  // Find detached nodes
  findDetachedNodes(): Element[] {
    const detached: Element[] = [];

    for (const [id, node] of this.nodeReferences) {
      if (!document.contains(node)) {
        detached.push(node);
        console.warn(`Detached node found: ${id}`);
      }
    }

    return detached;
  }

  // Clean up detached nodes
  cleanupDetached(): void {
    for (const [id, node] of Array.from(this.nodeReferences)) {
      if (!document.contains(node)) {
        this.nodeReferences.delete(id);
        console.log(`Cleaned up detached node: ${id}`);
      }
    }
  }

  // Get memory usage info
  getInfo(): { total: number; detached: number } {
    const detached = this.findDetachedNodes();
    return {
      total: this.nodeReferences.size,
      detached: detached.length,
    };
  }
}

// Pattern: Proper node removal
class ListView {
  private domManager = new DOMManager();
  private container: HTMLElement;

  constructor(container: HTMLElement) {
    this.container = container;
  }

  addItem(id: string, content: string): void {
    const item = document.createElement("div");
    item.className = "list-item";
    item.textContent = content;

    // Add click handler
    item.addEventListener("click", () => {
      console.log(`Item ${id} clicked`);
    });

    this.container.appendChild(item);
    this.domManager.attachNode(id, item);
  }

  removeItem(id: string): void {
    // Properly remove with cleanup
    this.domManager.removeNode(id);
  }

  clear(): void {
    // Clear all items
    this.container.innerHTML = "";

    // Cleanup detached nodes
    this.domManager.cleanupDetached();
  }

  audit(): void {
    const info = this.domManager.getInfo();
    console.log("DOM Audit:", info);

    if (info.detached > 0) {
      console.warn(`⚠️ ${info.detached} detached nodes found!`);
      this.domManager.cleanupDetached();
    }
  }
}

// Usage
const listView = new ListView(document.getElementById("list")!);

// Add items
listView.addItem("item1", "First Item");
listView.addItem("item2", "Second Item");
listView.addItem("item3", "Third Item");

// Remove item properly
listView.removeItem("item2");

// Periodic audit
setInterval(() => listView.audit(), 30000); // Every 30 seconds
```

### Example 6: Memory Profiling with Performance API

```typescript
// Monitor memory usage in production
class MemoryMonitor {
  private measurements: MemoryMeasurement[] = [];
  private readonly MAX_MEASUREMENTS = 100;
  private monitoringInterval: number | null = null;

  // Start monitoring
  startMonitoring(interval: number = 10000): void {
    if (this.monitoringInterval) {
      return;
    }

    this.monitoringInterval = window.setInterval(() => {
      this.takeMeasurement();
    }, interval);

    // Initial measurement
    this.takeMeasurement();
  }

  // Stop monitoring
  stopMonitoring(): void {
    if (this.monitoringInterval) {
      clearInterval(this.monitoringInterval);
      this.monitoringInterval = null;
    }
  }

  // Take memory measurement
  private takeMeasurement(): void {
    const memory = this.getMemoryInfo();

    if (memory) {
      this.measurements.push({
        timestamp: Date.now(),
        ...memory,
      });

      // Keep only recent measurements
      if (this.measurements.length > this.MAX_MEASUREMENTS) {
        this.measurements.shift();
      }

      // Check for potential leaks
      this.detectLeaks();
    }
  }

  // Get memory information
  private getMemoryInfo(): MemoryInfo | null {
    const perf = performance as any;

    if (perf.memory) {
      return {
        usedJSHeapSize: perf.memory.usedJSHeapSize,
        totalJSHeapSize: perf.memory.totalJSHeapSize,
        jsHeapSizeLimit: perf.memory.jsHeapSizeLimit,
      };
    }

    return null;
  }

  // Detect potential memory leaks
  private detectLeaks(): void {
    if (this.measurements.length < 10) {
      return;
    }

    // Get recent measurements
    const recent = this.measurements.slice(-10);
    const first = recent[0];
    const last = recent[recent.length - 1];

    // Calculate growth rate
    const duration = (last.timestamp - first.timestamp) / 1000; // seconds
    const growth = last.usedJSHeapSize - first.usedJSHeapSize;
    const growthRate = growth / duration; // bytes per second

    // Alert if consistent growth detected
    if (growthRate > 1000000) {
      // 1MB per second
      console.warn("⚠️ Potential memory leak detected!");
      console.warn(
        `Memory growing at ${(growthRate / 1000000).toFixed(2)} MB/s`,
      );
      this.generateReport();
    }
  }

  // Generate memory report
  generateReport(): MemoryReport {
    if (this.measurements.length === 0) {
      return {
        current: null,
        peak: null,
        average: null,
        trend: "stable",
      };
    }

    const current = this.measurements[this.measurements.length - 1];
    const peak = this.measurements.reduce((max, m) =>
      m.usedJSHeapSize > max.usedJSHeapSize ? m : max,
    );

    const average =
      this.measurements.reduce((sum, m) => sum + m.usedJSHeapSize, 0) /
      this.measurements.length;

    // Determine trend
    const firstHalf = this.measurements.slice(
      0,
      Math.floor(this.measurements.length / 2),
    );
    const secondHalf = this.measurements.slice(
      Math.floor(this.measurements.length / 2),
    );

    const firstAvg =
      firstHalf.reduce((sum, m) => sum + m.usedJSHeapSize, 0) /
      firstHalf.length;
    const secondAvg =
      secondHalf.reduce((sum, m) => sum + m.usedJSHeapSize, 0) /
      secondHalf.length;

    const growthPercent = ((secondAvg - firstAvg) / firstAvg) * 100;

    let trend: "growing" | "stable" | "decreasing";
    if (growthPercent > 10) trend = "growing";
    else if (growthPercent < -10) trend = "decreasing";
    else trend = "stable";

    return {
      current: {
        used: this.formatBytes(current.usedJSHeapSize),
        total: this.formatBytes(current.totalJSHeapSize),
        limit: this.formatBytes(current.jsHeapSizeLimit),
      },
      peak: {
        used: this.formatBytes(peak.usedJSHeapSize),
        timestamp: new Date(peak.timestamp),
      },
      average: this.formatBytes(average),
      trend,
    };
  }

  private formatBytes(bytes: number): string {
    const mb = bytes / (1024 * 1024);
    return `${mb.toFixed(2)} MB`;
  }

  // Get measurements for charting
  getMeasurements(): MemoryMeasurement[] {
    return [...this.measurements];
  }
}

interface MemoryInfo {
  usedJSHeapSize: number;
  totalJSHeapSize: number;
  jsHeapSizeLimit: number;
}

interface MemoryMeasurement extends MemoryInfo {
  timestamp: number;
}

interface MemoryReport {
  current: {
    used: string;
    total: string;
    limit: string;
  } | null;
  peak: {
    used: string;
    timestamp: Date;
  } | null;
  average: string | null;
  trend: "growing" | "stable" | "decreasing";
}

// Usage
const memoryMonitor = new MemoryMonitor();

// Start monitoring
memoryMonitor.startMonitoring(5000); // Every 5 seconds

// Get report
setTimeout(() => {
  const report = memoryMonitor.generateReport();
  console.log("Memory Report:", report);
}, 60000);

// Stop monitoring
// memoryMonitor.stopMonitoring();
```

### Example 7: Object Pool Pattern

```typescript
// Reuse objects to reduce garbage collection pressure
class ObjectPool<T> {
  private available: T[] = [];
  private inUse = new Set<T>();
  private factory: () => T;
  private reset: (obj: T) => void;
  private maxSize: number;

  constructor(
    factory: () => T,
    reset: (obj: T) => void,
    initialSize: number = 10,
    maxSize: number = 100,
  ) {
    this.factory = factory;
    this.reset = reset;
    this.maxSize = maxSize;

    // Pre-create objects
    for (let i = 0; i < initialSize; i++) {
      this.available.push(factory());
    }
  }

  // Acquire object from pool
  acquire(): T {
    let obj: T;

    if (this.available.length > 0) {
      obj = this.available.pop()!;
    } else {
      obj = this.factory();
    }

    this.inUse.add(obj);
    return obj;
  }

  // Release object back to pool
  release(obj: T): void {
    if (!this.inUse.has(obj)) {
      console.warn("Attempting to release object not from pool");
      return;
    }

    this.inUse.delete(obj);
    this.reset(obj);

    if (this.available.length < this.maxSize) {
      this.available.push(obj);
    }
  }

  // Get pool statistics
  getStats(): { available: number; inUse: number; total: number } {
    return {
      available: this.available.length,
      inUse: this.inUse.size,
      total: this.available.length + this.inUse.size,
    };
  }

  // Clear pool
  clear(): void {
    this.available = [];
    this.inUse.clear();
  }
}

// Example: Vector2D object pool
interface Vector2D {
  x: number;
  y: number;
}

const vectorPool = new ObjectPool<Vector2D>(
  // Factory function
  () => ({ x: 0, y: 0 }),
  // Reset function
  (v) => {
    v.x = 0;
    v.y = 0;
  },
  50, // Initial size
  200, // Max size
);

// Usage in game/animation
class ParticleSystem {
  private particles: Particle[] = [];

  update(deltaTime: number): void {
    for (let i = this.particles.length - 1; i >= 0; i--) {
      const particle = this.particles[i];

      // Update particle
      particle.position.x += particle.velocity.x * deltaTime;
      particle.position.y += particle.velocity.y * deltaTime;
      particle.life -= deltaTime;

      // Remove dead particles
      if (particle.life <= 0) {
        this.removeParticle(i);
      }
    }
  }

  addParticle(x: number, y: number): void {
    const position = vectorPool.acquire();
    position.x = x;
    position.y = y;

    const velocity = vectorPool.acquire();
    velocity.x = (Math.random() - 0.5) * 100;
    velocity.y = (Math.random() - 0.5) * 100;

    this.particles.push({
      position,
      velocity,
      life: 1.0,
    });
  }

  private removeParticle(index: number): void {
    const particle = this.particles[index];

    // Return vectors to pool
    vectorPool.release(particle.position);
    vectorPool.release(particle.velocity);

    this.particles.splice(index, 1);
  }

  clear(): void {
    // Return all vectors to pool
    for (const particle of this.particles) {
      vectorPool.release(particle.position);
      vectorPool.release(particle.velocity);
    }
    this.particles = [];
  }
}

interface Particle {
  position: Vector2D;
  velocity: Vector2D;
  life: number;
}

// Example: DOM element pool
const divPool = new ObjectPool<HTMLDivElement>(
  () => document.createElement("div"),
  (div) => {
    div.className = "";
    div.textContent = "";
    div.style.cssText = "";
  },
  20,
  100,
);

class VirtualList {
  private container: HTMLElement;
  private visibleElements: HTMLDivElement[] = [];

  constructor(container: HTMLElement) {
    this.container = container;
  }

  render(items: string[]): void {
    // Return current elements to pool
    this.visibleElements.forEach((el) => {
      el.remove();
      divPool.release(el);
    });
    this.visibleElements = [];

    // Acquire elements from pool
    items.forEach((item) => {
      const el = divPool.acquire();
      el.textContent = item;
      el.className = "list-item";
      this.container.appendChild(el);
      this.visibleElements.push(el);
    });
  }

  clear(): void {
    this.visibleElements.forEach((el) => {
      el.remove();
      divPool.release(el);
    });
    this.visibleElements = [];
  }
}
```

### Example 8: Lazy Loading and Unloading

```typescript
// Load and unload resources based on visibility
class ResourceManager {
  private resources = new Map<string, Resource>();
  private observers = new Map<string, IntersectionObserver>();

  // Register resource with lazy loading
  registerLazy(
    element: HTMLElement,
    loader: () => Promise<any>,
    unloader?: () => void,
  ): void {
    const id = this.generateId();

    const resource: Resource = {
      element,
      loader,
      unloader,
      loaded: false,
      data: null,
    };

    this.resources.set(id, resource);

    // Setup intersection observer
    const observer = new IntersectionObserver(
      (entries) => this.handleIntersection(id, entries),
      { rootMargin: "50px" },
    );

    observer.observe(element);
    this.observers.set(id, observer);
  }

  private async handleIntersection(
    id: string,
    entries: IntersectionObserverEntry[],
  ): Promise<void> {
    const resource = this.resources.get(id);
    if (!resource) return;

    const entry = entries[0];

    if (entry.isIntersecting && !resource.loaded) {
      // Load resource
      try {
        resource.data = await resource.loader();
        resource.loaded = true;
        console.log(`Resource ${id} loaded`);
      } catch (error) {
        console.error(`Failed to load resource ${id}:`, error);
      }
    } else if (!entry.isIntersecting && resource.loaded) {
      // Unload resource if far from viewport
      if (resource.unloader) {
        resource.unloader();
        resource.data = null;
        resource.loaded = false;
        console.log(`Resource ${id} unloaded`);
      }
    }
  }

  // Unregister resource
  unregister(element: HTMLElement): void {
    for (const [id, resource] of this.resources) {
      if (resource.element === element) {
        const observer = this.observers.get(id);
        if (observer) {
          observer.disconnect();
          this.observers.delete(id);
        }

        if (resource.unloader && resource.loaded) {
          resource.unloader();
        }

        this.resources.delete(id);
        break;
      }
    }
  }

  // Clean up all resources
  cleanup(): void {
    for (const observer of this.observers.values()) {
      observer.disconnect();
    }

    for (const resource of this.resources.values()) {
      if (resource.unloader && resource.loaded) {
        resource.unloader();
      }
    }

    this.resources.clear();
    this.observers.clear();
  }

  private generateId(): string {
    return `resource-${Date.now()}-${Math.random()}`;
  }
}

interface Resource {
  element: HTMLElement;
  loader: () => Promise<any>;
  unloader?: () => void;
  loaded: boolean;
  data: any;
}

// Usage with images
const resourceManager = new ResourceManager();

function setupLazyImage(img: HTMLImageElement): void {
  const src = img.dataset.src;
  if (!src) return;

  resourceManager.registerLazy(
    img,
    // Loader
    async () => {
      return new Promise((resolve, reject) => {
        const image = new Image();
        image.onload = () => {
          img.src = src;
          resolve(image);
        };
        image.onerror = reject;
        image.src = src;
      });
    },
    // Unloader (optional)
    () => {
      img.src =
        "data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7";
    },
  );
}

// Usage with heavy components
function setupLazyComponent(container: HTMLElement): void {
  resourceManager.registerLazy(
    container,
    // Loader
    async () => {
      const module = await import("./heavy-component");
      const component = new module.HeavyComponent(container);
      component.render();
      return component;
    },
    // Unloader
    () => {
      container.innerHTML = "";
    },
  );
}
```

### Example 9: Memory-Efficient Data Structures

```typescript
// Use efficient data structures for large datasets
class CircularBuffer<T> {
  private buffer: T[];
  private head = 0;
  private tail = 0;
  private size = 0;
  private capacity: number;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.buffer = new Array(capacity);
  }

  // Add item (overwrites oldest if full)
  push(item: T): void {
    this.buffer[this.tail] = item;
    this.tail = (this.tail + 1) % this.capacity;

    if (this.size < this.capacity) {
      this.size++;
    } else {
      this.head = (this.head + 1) % this.capacity;
    }
  }

  // Remove and return oldest item
  shift(): T | undefined {
    if (this.size === 0) return undefined;

    const item = this.buffer[this.head];
    this.head = (this.head + 1) % this.capacity;
    this.size--;

    return item;
  }

  // Get item at index
  get(index: number): T | undefined {
    if (index < 0 || index >= this.size) return undefined;
    const pos = (this.head + index) % this.capacity;
    return this.buffer[pos];
  }

  // Get current size
  getSize(): number {
    return this.size;
  }

  // Check if full
  isFull(): boolean {
    return this.size === this.capacity;
  }

  // Clear buffer
  clear(): void {
    this.head = 0;
    this.tail = 0;
    this.size = 0;
  }

  // Convert to array
  toArray(): T[] {
    const result: T[] = [];
    for (let i = 0; i < this.size; i++) {
      result.push(this.get(i)!);
    }
    return result;
  }
}

// Usage for logging (fixed memory footprint)
class Logger {
  private logs = new CircularBuffer<LogEntry>(1000); // Max 1000 logs

  log(level: "info" | "warn" | "error", message: string): void {
    this.logs.push({
      level,
      message,
      timestamp: Date.now(),
    });
  }

  getRecentLogs(count: number = 10): LogEntry[] {
    const size = this.logs.getSize();
    const startIndex = Math.max(0, size - count);
    const result: LogEntry[] = [];

    for (let i = startIndex; i < size; i++) {
      const log = this.logs.get(i);
      if (log) result.push(log);
    }

    return result;
  }
}

interface LogEntry {
  level: "info" | "warn" | "error";
  message: string;
  timestamp: number;
}

// LRU Cache with size limit
class LRUCache<K, V> {
  private cache = new Map<K, V>();
  private maxSize: number;

  constructor(maxSize: number) {
    this.maxSize = maxSize;
  }

  get(key: K): V | undefined {
    if (!this.cache.has(key)) return undefined;

    // Move to end (most recently used)
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);

    return value;
  }

  set(key: K, value: V): void {
    // Remove if exists (to re-add at end)
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }

    // Add to end
    this.cache.set(key, value);

    // Evict oldest if over capacity
    if (this.cache.size > this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
  }

  has(key: K): boolean {
    return this.cache.has(key);
  }

  delete(key: K): boolean {
    return this.cache.delete(key);
  }

  clear(): void {
    this.cache.clear();
  }

  size(): number {
    return this.cache.size;
  }
}

// Usage for API response caching
const apiCache = new LRUCache<string, any>(100); // Max 100 cached responses

async function fetchWithCache(url: string): Promise<any> {
  // Check cache
  if (apiCache.has(url)) {
    return apiCache.get(url);
  }

  // Fetch and cache
  const response = await fetch(url);
  const data = await response.json();
  apiCache.set(url, data);

  return data;
}
```

### Example 10: Memory Pressure API

```typescript
// Respond to memory pressure events
class MemoryPressureHandler {
  private pressureObserver: PerformanceObserver | null = null;
  private callbacks: Set<(level: "nominal" | "moderate" | "critical") => void> =
    new Set();

  constructor() {
    this.setupPressureObserver();
  }

  private setupPressureObserver(): void {
    // Check if PressureObserver is available (Chrome 106+)
    if ("PressureObserver" in window) {
      try {
        const observer = new (window as any).PressureObserver(
          (records: any[]) => this.handlePressure(records),
        );

        observer.observe("cpu");
        console.log("Memory pressure monitoring enabled");
      } catch (error) {
        console.warn("Failed to setup pressure observer:", error);
      }
    } else {
      console.warn("PressureObserver not supported");
      this.setupFallback();
    }
  }

  private handlePressure(records: any[]): void {
    const latestRecord = records[records.length - 1];
    const level = latestRecord.state;

    console.log(`Memory pressure: ${level}`);

    // Notify callbacks
    for (const callback of this.callbacks) {
      callback(level);
    }

    // Automatic actions based on pressure
    this.handlePressureLevel(level);
  }

  private handlePressureLevel(level: string): void {
    switch (level) {
      case "nominal":
        // Normal operation
        break;
      case "moderate":
        // Start reducing memory usage
        this.reduceMemoryUsage();
        break;
      case "critical":
        // Aggressive memory reduction
        this.criticalMemoryCleanup();
        break;
    }
  }

  private reduceMemoryUsage(): void {
    // Clear caches
    console.log("Reducing memory usage...");

    // Trigger garbage collection if available
    if ((global as any).gc) {
      (global as any).gc();
    }
  }

  private criticalMemoryCleanup(): void {
    console.log("Critical memory cleanup...");

    // Aggressive cleanup
    apiCache.clear();

    // Clear Service Worker caches
    if ("caches" in window) {
      caches.keys().then((names) => {
        for (const name of names) {
          caches.delete(name);
        }
      });
    }
  }

  private setupFallback(): void {
    // Fallback: Monitor memory usage manually
    if ("memory" in performance) {
      setInterval(() => {
        const memory = (performance as any).memory;
        const usagePercent =
          (memory.usedJSHeapSize / memory.jsHeapSizeLimit) * 100;

        if (usagePercent > 90) {
          this.handlePressureLevel("critical");
        } else if (usagePercent > 70) {
          this.handlePressureLevel("moderate");
        }
      }, 10000);
    }
  }

  // Subscribe to pressure changes
  onPressureChange(
    callback: (level: "nominal" | "moderate" | "critical") => void,
  ): void {
    this.callbacks.add(callback);
  }

  // Unsubscribe
  offPressureChange(
    callback: (level: "nominal" | "moderate" | "critical") => void,
  ): void {
    this.callbacks.delete(callback);
  }
}

// Usage
const pressureHandler = new MemoryPressureHandler();

pressureHandler.onPressureChange((level) => {
  console.log(`Pressure changed to: ${level}`);

  if (level === "critical") {
    // Pause non-essential operations
    pauseBackgroundTasks();
  } else if (level === "nominal") {
    // Resume normal operations
    resumeBackgroundTasks();
  }
});

function pauseBackgroundTasks(): void {
  // Implementation
}

function resumeBackgroundTasks(): void {
  // Implementation
}
```

## Real-World Usage

### E-Commerce Application

```typescript
// Memory-optimized e-commerce product catalog
class ProductCatalog {
  private eventManager = new EventManager();
  private timerManager = new TimerManager();
  private resourceManager = new ResourceManager();
  private productCache = new LRUCache<string, Product>(200);
  private visibleProducts: Set<string> = new Set();

  async loadCategory(categoryId: string): Promise<void> {
    // Load products with lazy images
    const products = await this.fetchProducts(categoryId);

    products.forEach((product) => {
      this.renderProduct(product);
    });
  }

  private renderProduct(product: Product): void {
    const element = this.createProductElement(product);

    // Setup lazy loading for product image
    const img = element.querySelector("img")!;
    this.resourceManager.registerLazy(
      img as HTMLElement,
      () => this.loadProductImage(product.imageUrl),
      () => this.unloadProductImage(img),
    );

    // Track with WeakMap
    this.productCache.set(product.id, product);
    this.visibleProducts.add(product.id);

    // Add event listeners with automatic cleanup
    this.eventManager.addEventListener(element, "click", () =>
      this.handleProductClick(product.id),
    );

    document.getElementById("catalog")?.appendChild(element);
  }

  private async loadProductImage(url: string): Promise<void> {
    // Load high-quality image
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => resolve();
      img.onerror = reject;
      img.src = url;
    });
  }

  private unloadProductImage(img: HTMLImageElement): void {
    // Replace with placeholder
    img.src = "/placeholder.jpg";
  }

  private handleProductClick(productId: string): void {
    const product = this.productCache.get(productId);
    if (product) {
      this.showProductDetails(product);
    }
  }

  private showProductDetails(product: Product): void {
    // Implementation
  }

  private createProductElement(product: Product): HTMLElement {
    const div = document.createElement("div");
    div.className = "product";
    div.innerHTML = `
      <img data-src="${product.imageUrl}" alt="${product.name}">
      <h3>${product.name}</h3>
      <p>$${product.price}</p>
    `;
    return div;
  }

  private async fetchProducts(categoryId: string): Promise<Product[]> {
    const response = await fetch(`/api/products?category=${categoryId}`);
    return response.json();
  }

  // Cleanup when navigating away
  cleanup(): void {
    this.eventManager.cleanup();
    this.timerManager.clearAll();
    this.resourceManager.cleanup();
    this.productCache.clear();
    this.visibleProducts.clear();
  }
}

interface Product {
  id: string;
  name: string;
  price: number;
  imageUrl: string;
}
```

### Real-Time Dashboard

```typescript
// Memory-efficient real-time dashboard
class Dashboard {
  private memoryMonitor = new MemoryMonitor();
  private dataBuffer = new CircularBuffer<DataPoint>(1000);
  private eventManager = new EventManager();
  private updateInterval: NodeJS.Timeout | null = null;

  start(): void {
    // Start memory monitoring
    this.memoryMonitor.startMonitoring(10000);

    // Start data updates
    this.updateInterval = setInterval(() => {
      this.updateData();
    }, 1000);

    // Memory pressure handling
    const pressureHandler = new MemoryPressureHandler();
    pressureHandler.onPressureChange((level) => {
      if (level === "critical") {
        this.reduceDataRetention();
      }
    });
  }

  private updateData(): void {
    const dataPoint: DataPoint = {
      timestamp: Date.now(),
      value: Math.random() * 100,
      label: "Metric",
    };

    // Add to circular buffer (fixed memory)
    this.dataBuffer.push(dataPoint);

    // Update visualization
    this.renderChart();
  }

  private renderChart(): void {
    const data = this.dataBuffer.toArray();
    // Render chart with limited data points
  }

  private reduceDataRetention(): void {
    // Keep less history during memory pressure
    console.log("Reducing data retention...");
    this.dataBuffer.clear();
  }

  stop(): void {
    if (this.updateInterval) {
      clearInterval(this.updateInterval);
    }
    this.memoryMonitor.stopMonitoring();
    this.eventManager.cleanup();
  }
}

interface DataPoint {
  timestamp: number;
  value: number;
  label: string;
}
```

## Production Patterns

### 1. **Automatic Cleanup Pattern**

```typescript
// Use AbortController for automatic cleanup
class Component {
  private abortController = new AbortController();

  setupListeners(): void {
    window.addEventListener("resize", this.handleResize, {
      signal: this.abortController.signal,
    });

    fetch("/api/data", { signal: this.abortController.signal })
      .then((r) => r.json())
      .then((data) => this.processData(data));
  }

  destroy(): void {
    this.abortController.abort(); // Cleans up everything
  }

  private handleResize = (): void => {
    // Handle resize
  };

  private processData(data: any): void {
    // Process data
  }
}
```

### 2. **WeakRef Pattern for Caching**

```typescript
// Use WeakRef for optional caching
class CacheWithWeakRef<K, V extends object> {
  private cache = new Map<K, WeakRef<V>>();
  private registry = new FinalizationRegistry<K>((key) => {
    this.cache.delete(key);
  });

  set(key: K, value: V): void {
    this.cache.set(key, new WeakRef(value));
    this.registry.register(value, key);
  }

  get(key: K): V | undefined {
    const ref = this.cache.get(key);
    return ref?.deref();
  }
}
```

## Best Practices

1. **Always clean up event listeners** when components unmount
2. **Clear timers and intervals** when no longer needed
3. **Use WeakMap/WeakSet** for metadata that shouldn't prevent garbage collection
4. **Avoid closures** that capture large objects unnecessarily
5. **Remove DOM nodes properly** to prevent detached node leaks
6. **Monitor memory usage** in production with Performance API
7. **Use object pools** for frequently created/destroyed objects
8. **Implement lazy loading** for resources not immediately needed
9. **Limit cache sizes** with LRU or similar eviction policies
10. **Use AbortController** for automatic cleanup of async operations
11. **Avoid circular references** between objects
12. **Clear references** to large data structures when done
13. **Use efficient data structures** (CircularBuffer, LRUCache)
14. **Profile regularly** with Chrome DevTools heap snapshots
15. **Respond to memory pressure** by reducing memory usage

## 10 Key Takeaways

1. **Event listeners are the #1 cause** of memory leaks in web applications - always remove them or use AbortController
2. **Timers must be cleared** - forgotten setInterval/setTimeout calls accumulate and leak memory over time
3. **WeakMap/WeakSet prevent memory leaks** by allowing garbage collection when only the weak reference remains
4. **Closures capture entire scope** - only capture what you need to avoid holding references to large objects
5. **Detached DOM nodes leak memory** - elements removed from DOM but still referenced in JavaScript prevent GC
6. **Chrome DevTools heap profiler** reveals memory leaks by comparing snapshots and finding retained objects
7. **Object pooling reduces GC pressure** - reusing objects instead of creating/destroying reduces garbage collection pauses
8. **Circular buffers provide fixed memory** - perfect for logs, metrics, or any data stream with size limits
9. **LRU caching prevents unbounded growth** - automatic eviction ensures caches don't consume all available memory
10. **Memory pressure API enables adaptive behavior** - reduce memory usage when system is under pressure for better UX
