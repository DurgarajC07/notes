# Rendering Performance Optimization

## Core Concepts

Rendering performance is critical for smooth, responsive user interfaces. The browser rendering pipeline consists of JavaScript execution, style calculation, layout (reflow), paint, and compositing. Understanding these stages and optimizing them is essential for achieving 60fps performance and preventing visual jank.

### Browser Rendering Pipeline

1. **JavaScript**: Execute scripts that modify DOM/CSSOM
2. **Style Calculation**: Compute final styles for elements
3. **Layout (Reflow)**: Calculate geometry and position of elements
4. **Paint**: Fill in pixels (text, colors, images, borders, shadows)
5. **Composite**: Draw layers in correct order to screen

### Key Concepts

- **Reflow (Layout)**: Recalculating element geometry (expensive)
- **Repaint**: Redrawing pixels without layout changes (cheaper than reflow)
- **Compositing**: Moving layers on GPU (cheapest)
- **Layout Thrashing**: Forced synchronous layouts causing performance issues
- **Layer Promotion**: Moving elements to separate composite layers
- **GPU Acceleration**: Offloading rendering work to graphics processor

## TypeScript/JavaScript Code Examples

### Example 1: Preventing Layout Thrashing

```typescript
// Avoid forced synchronous layouts
// ❌ BAD - Causes layout thrashing
function badBatchUpdate(elements: HTMLElement[]): void {
  elements.forEach((element) => {
    // Read
    const height = element.offsetHeight;

    // Write
    element.style.height = `${height + 10}px`;

    // This causes immediate layout recalculation!
    // Reading offsetHeight forces layout, then writing triggers another
  });
}

// ✅ GOOD - Batch reads and writes
function goodBatchUpdate(elements: HTMLElement[]): void {
  // First, read all measurements
  const heights = elements.map((element) => element.offsetHeight);

  // Then, apply all changes
  elements.forEach((element, i) => {
    element.style.height = `${heights[i] + 10}px`;
  });
}

// Advanced: Read/Write separation helper
class DOMBatcher {
  private readCallbacks: Array<() => void> = [];
  private writeCallbacks: Array<() => void> = [];
  private scheduled = false;

  // Schedule a DOM read
  read(callback: () => void): void {
    this.readCallbacks.push(callback);
    this.schedule();
  }

  // Schedule a DOM write
  write(callback: () => void): void {
    this.writeCallbacks.push(callback);
    this.schedule();
  }

  private schedule(): void {
    if (this.scheduled) return;

    this.scheduled = true;
    requestAnimationFrame(() => this.flush());
  }

  private flush(): void {
    // Execute all reads first
    const reads = this.readCallbacks.slice();
    this.readCallbacks = [];
    reads.forEach((callback) => callback());

    // Then execute all writes
    const writes = this.writeCallbacks.slice();
    this.writeCallbacks = [];
    writes.forEach((callback) => callback());

    this.scheduled = false;
  }
}

// Usage
const batcher = new DOMBatcher();

function updateElements(elements: HTMLElement[]): void {
  const heights: number[] = [];

  // Schedule reads
  elements.forEach((element, i) => {
    batcher.read(() => {
      heights[i] = element.offsetHeight;
    });
  });

  // Schedule writes
  elements.forEach((element, i) => {
    batcher.write(() => {
      element.style.height = `${heights[i] + 10}px`;
    });
  });
}
```

### Example 2: FastDOM Library Alternative

```typescript
// Optimized DOM manipulation scheduler
class FastDOM {
  private readQueue: Array<() => void> = [];
  private writeQueue: Array<() => void> = [];
  private rafId: number | null = null;

  // Measure DOM properties
  measure<T>(callback: () => T): Promise<T> {
    return new Promise((resolve) => {
      this.readQueue.push(() => {
        const result = callback();
        resolve(result);
      });
      this.scheduleFlush();
    });
  }

  // Mutate DOM
  mutate(callback: () => void): Promise<void> {
    return new Promise((resolve) => {
      this.writeQueue.push(() => {
        callback();
        resolve();
      });
      this.scheduleFlush();
    });
  }

  private scheduleFlush(): void {
    if (this.rafId !== null) return;

    this.rafId = requestAnimationFrame(() => {
      this.flush();
    });
  }

  private flush(): void {
    // Process reads
    let callback: (() => void) | undefined;
    while ((callback = this.readQueue.shift())) {
      callback();
    }

    // Process writes
    while ((callback = this.writeQueue.shift())) {
      callback();
    }

    this.rafId = null;

    // If new items were added during flush, schedule again
    if (this.readQueue.length || this.writeQueue.length) {
      this.scheduleFlush();
    }
  }

  // Clear all pending operations
  clear(): void {
    if (this.rafId !== null) {
      cancelAnimationFrame(this.rafId);
      this.rafId = null;
    }
    this.readQueue = [];
    this.writeQueue = [];
  }
}

// Usage
const fastdom = new FastDOM();

async function animateElements(elements: HTMLElement[]): Promise<void> {
  // Measure phase
  const measurements = await Promise.all(
    elements.map((el) =>
      fastdom.measure(() => ({
        width: el.offsetWidth,
        height: el.offsetHeight,
      })),
    ),
  );

  // Mutate phase
  await Promise.all(
    elements.map((el, i) =>
      fastdom.mutate(() => {
        el.style.width = `${measurements[i].width * 1.1}px`;
        el.style.height = `${measurements[i].height * 1.1}px`;
      }),
    ),
  );
}

// Example: Smooth scroll position tracking
class ScrollTracker {
  private fastdom = new FastDOM();
  private element: HTMLElement;
  private onScroll: (position: number) => void;

  constructor(element: HTMLElement, onScroll: (position: number) => void) {
    this.element = element;
    this.onScroll = onScroll;
    this.setupListener();
  }

  private setupListener(): void {
    this.element.addEventListener(
      "scroll",
      () => {
        // Measure scroll position without causing layout thrashing
        this.fastdom.measure(() => {
          const position = this.element.scrollTop;
          this.onScroll(position);
        });
      },
      { passive: true },
    );
  }
}
```

### Example 3: CSS Containment for Performance

```typescript
// Apply CSS containment to isolate layout/paint work
class ContainmentManager {
  private static readonly CONTAINMENT_TYPES = {
    STRICT: "strict",
    CONTENT: "content",
    LAYOUT: "layout",
    PAINT: "paint",
    SIZE: "size",
    STYLE: "style",
  } as const;

  // Apply containment to element
  static applyContainment(element: HTMLElement, types: string[]): void {
    element.style.contain = types.join(" ");
  }

  // Apply strict containment (layout + paint + size + style)
  static applyStrictContainment(element: HTMLElement): void {
    element.style.contain = "strict";
  }

  // Apply content containment (layout + paint + style)
  static applyContentContainment(element: HTMLElement): void {
    element.style.contain = "content";
  }

  // Optimize container for dynamic content
  static optimizeContainer(container: HTMLElement): void {
    // Apply layout containment to prevent child changes from affecting siblings
    container.style.contain = "layout paint";

    // Use content-visibility for off-screen optimization
    container.style.contentVisibility = "auto";

    // Set contain-intrinsic-size for placeholders
    const height = container.offsetHeight;
    container.style.containIntrinsicSize = `auto ${height}px`;
  }

  // Optimize list items
  static optimizeListItems(items: HTMLElement[]): void {
    items.forEach((item) => {
      // Each list item is isolated
      item.style.contain = "layout style paint";
    });
  }
}

// Usage for virtual list
class VirtualList {
  private container: HTMLElement;
  private items: any[] = [];
  private itemHeight = 50;

  constructor(container: HTMLElement) {
    this.container = container;
    this.setupContainment();
  }

  private setupContainment(): void {
    // Optimize container
    ContainmentManager.applyContentContainment(this.container);

    // Enable content-visibility for off-screen rendering skip
    this.container.style.contentVisibility = "auto";
    this.container.style.containIntrinsicSize = `0 ${this.items.length * this.itemHeight}px`;
  }

  render(): void {
    this.container.innerHTML = "";

    const fragment = document.createDocumentFragment();

    this.items.forEach((item) => {
      const element = this.createItemElement(item);

      // Apply containment to each item
      ContainmentManager.applyContainment(element, ["layout", "paint"]);

      fragment.appendChild(element);
    });

    this.container.appendChild(fragment);
  }

  private createItemElement(item: any): HTMLElement {
    const div = document.createElement("div");
    div.className = "list-item";
    div.textContent = item.name;
    div.style.height = `${this.itemHeight}px`;
    return div;
  }

  setItems(items: any[]): void {
    this.items = items;
    this.container.style.containIntrinsicSize = `0 ${items.length * this.itemHeight}px`;
    this.render();
  }
}
```

### Example 4: Will-Change Property Management

```typescript
// Intelligent will-change management
class WillChangeManager {
  private elements = new Map<HTMLElement, WillChangeConfig>();
  private timers = new Map<HTMLElement, number>();

  // Add will-change property before animation
  prepare(
    element: HTMLElement,
    properties: string[],
    duration: number = 1000,
  ): void {
    // Store config
    const config: WillChangeConfig = {
      properties,
      originalValue: element.style.willChange,
    };
    this.elements.set(element, config);

    // Apply will-change
    element.style.willChange = properties.join(", ");

    // Auto-remove after duration
    this.scheduleRemoval(element, duration);
  }

  // Remove will-change property
  cleanup(element: HTMLElement): void {
    const config = this.elements.get(element);
    if (!config) return;

    // Restore original value
    element.style.willChange = config.originalValue;

    // Clear timer
    const timer = this.timers.get(element);
    if (timer) {
      clearTimeout(timer);
      this.timers.delete(element);
    }

    this.elements.delete(element);
  }

  private scheduleRemoval(element: HTMLElement, duration: number): void {
    // Clear existing timer
    const existingTimer = this.timers.get(element);
    if (existingTimer) {
      clearTimeout(existingTimer);
    }

    // Schedule removal
    const timer = window.setTimeout(() => {
      this.cleanup(element);
    }, duration);

    this.timers.set(element, timer);
  }

  // Cleanup all
  cleanupAll(): void {
    for (const element of this.elements.keys()) {
      this.cleanup(element);
    }
  }
}

interface WillChangeConfig {
  properties: string[];
  originalValue: string;
}

// Usage in animations
class AnimationController {
  private willChangeManager = new WillChangeManager();

  async animateElement(element: HTMLElement): Promise<void> {
    // Prepare for animation
    this.willChangeManager.prepare(element, ["transform", "opacity"], 500);

    // Wait a frame for browser to optimize
    await this.nextFrame();

    // Perform animation
    element.style.transform = "translateX(100px)";
    element.style.opacity = "0.5";

    // Animation cleanup happens automatically after 500ms
  }

  async slideIn(element: HTMLElement): Promise<void> {
    // Initial state
    element.style.transform = "translateY(-100%)";
    element.style.opacity = "0";

    // Prepare
    this.willChangeManager.prepare(element, ["transform", "opacity"]);

    await this.nextFrame();

    // Animate
    element.style.transition = "transform 0.3s, opacity 0.3s";
    element.style.transform = "translateY(0)";
    element.style.opacity = "1";

    // Cleanup after animation
    setTimeout(() => {
      element.style.transition = "";
      this.willChangeManager.cleanup(element);
    }, 300);
  }

  private nextFrame(): Promise<void> {
    return new Promise((resolve) => {
      requestAnimationFrame(() => resolve());
    });
  }
}

// Example: Hover optimization
class HoverOptimizer {
  private willChangeManager = new WillChangeManager();

  setupHoverOptimization(element: HTMLElement): void {
    let hoverTimer: number;

    element.addEventListener("mouseenter", () => {
      // Prepare for potential animation
      hoverTimer = window.setTimeout(() => {
        this.willChangeManager.prepare(
          element,
          ["transform", "box-shadow"],
          2000,
        );
      }, 50); // Small delay to avoid unnecessary optimization
    });

    element.addEventListener("mouseleave", () => {
      clearTimeout(hoverTimer);
      this.willChangeManager.cleanup(element);
    });
  }
}
```

### Example 5: Layer Promotion and GPU Acceleration

```typescript
// Manage composite layers for optimal performance
class LayerManager {
  // Promote element to composite layer
  static promote(element: HTMLElement): void {
    // Use transform3d to trigger layer promotion
    element.style.transform = "translateZ(0)";

    // Alternative methods:
    // element.style.willChange = 'transform';
    // element.style.backfaceVisibility = 'hidden';
  }

  // Demote element from composite layer
  static demote(element: HTMLElement): void {
    element.style.transform = "";
    element.style.willChange = "";
  }

  // Create isolated stacking context
  static isolate(element: HTMLElement): void {
    element.style.isolation = "isolate";
  }

  // Optimize for animation
  static optimizeForAnimation(element: HTMLElement): void {
    // Promote to layer
    this.promote(element);

    // Enable hardware acceleration
    element.style.backfaceVisibility = "hidden";
    element.style.perspective = "1000px";
  }

  // Check if element is on composite layer (requires DevTools)
  static isComposited(element: HTMLElement): boolean {
    // This is a heuristic - actual layer info requires DevTools
    const transform = getComputedStyle(element).transform;
    return transform !== "none";
  }
}

// Animated component with layer management
class AnimatedCard {
  private element: HTMLElement;
  private isAnimating = false;

  constructor(element: HTMLElement) {
    this.element = element;
    this.setupInteractions();
  }

  private setupInteractions(): void {
    this.element.addEventListener("mouseenter", () => {
      this.startAnimation();
    });

    this.element.addEventListener("mouseleave", () => {
      this.stopAnimation();
    });
  }

  private startAnimation(): void {
    if (this.isAnimating) return;

    this.isAnimating = true;

    // Promote to composite layer before animation
    LayerManager.optimizeForAnimation(this.element);

    // Animate using transform (composited property)
    this.element.style.transition = "transform 0.3s ease-out";
    this.element.style.transform = "translateZ(0) scale(1.05)";
  }

  private stopAnimation(): void {
    this.isAnimating = false;

    // Animate back
    this.element.style.transform = "translateZ(0) scale(1)";

    // Demote after animation completes
    setTimeout(() => {
      this.element.style.transition = "";
      LayerManager.demote(this.element);
    }, 300);
  }
}

// Parallax scroller with layers
class ParallaxScroller {
  private layers: ParallaxLayer[] = [];
  private lastScrollY = 0;
  private ticking = false;

  addLayer(element: HTMLElement, speed: number): void {
    // Promote all parallax layers
    LayerManager.promote(element);

    this.layers.push({ element, speed });
  }

  start(): void {
    window.addEventListener(
      "scroll",
      () => {
        this.lastScrollY = window.scrollY;

        if (!this.ticking) {
          requestAnimationFrame(() => {
            this.update();
            this.ticking = false;
          });
          this.ticking = true;
        }
      },
      { passive: true },
    );
  }

  private update(): void {
    this.layers.forEach((layer) => {
      const offset = this.lastScrollY * layer.speed;

      // Use transform for GPU acceleration
      layer.element.style.transform = `translate3d(0, ${offset}px, 0)`;
    });
  }
}

interface ParallaxLayer {
  element: HTMLElement;
  speed: number;
}
```

### Example 6: Paint Profiling and Optimization

```typescript
// Monitor and optimize paint operations
class PaintOptimizer {
  private paintAreas = new Map<HTMLElement, PaintMetrics>();

  // Mark element for paint tracking
  trackPaint(element: HTMLElement): void {
    // Use PerformanceObserver to track paint
    if ("PerformanceObserver" in window) {
      const observer = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          if (entry.entryType === "paint") {
            this.recordPaint(element, entry);
          }
        }
      });

      try {
        observer.observe({ entryTypes: ["paint"] });
      } catch (e) {
        console.warn("Paint observation not supported");
      }
    }
  }

  private recordPaint(element: HTMLElement, entry: PerformanceEntry): void {
    const metrics = this.paintAreas.get(element) || {
      count: 0,
      lastPaint: 0,
      frequency: 0,
    };

    metrics.count++;
    metrics.lastPaint = entry.startTime;

    // Calculate paint frequency
    if (metrics.count > 1) {
      const timeSinceFirst = entry.startTime - metrics.lastPaint;
      metrics.frequency = metrics.count / (timeSinceFirst / 1000);
    }

    this.paintAreas.set(element, metrics);

    // Alert if painting too frequently
    if (metrics.frequency > 60) {
      console.warn(
        `High paint frequency detected: ${metrics.frequency.toFixed(2)}/s`,
        element,
      );
      this.optimizePaint(element);
    }
  }

  private optimizePaint(element: HTMLElement): void {
    // Apply optimizations

    // 1. Isolate with containment
    element.style.contain = "paint layout";

    // 2. Use will-change if animating
    const isAnimating = this.isElementAnimating(element);
    if (isAnimating) {
      element.style.willChange = "transform";
    }

    // 3. Promote to composite layer
    LayerManager.promote(element);
  }

  private isElementAnimating(element: HTMLElement): boolean {
    const computedStyle = getComputedStyle(element);
    return (
      computedStyle.animation !== "none" ||
      computedStyle.transition !== "all 0s ease 0s"
    );
  }

  // Get paint statistics
  getStats(element: HTMLElement): PaintMetrics | undefined {
    return this.paintAreas.get(element);
  }

  // Reduce paint area with clipping
  static clipPaintArea(element: HTMLElement): void {
    // Use clip-path to reduce paint area
    const rect = element.getBoundingClientRect();
    element.style.clipPath = `inset(0 0 0 0)`;
  }

  // Optimize box-shadow (expensive to paint)
  static optimizeBoxShadow(element: HTMLElement): void {
    // Move to pseudo-element on separate layer
    const shadow = getComputedStyle(element).boxShadow;

    if (shadow && shadow !== "none") {
      element.style.boxShadow = "none";
      element.setAttribute("data-shadow", shadow);

      // Create pseudo-element for shadow
      const style = document.createElement("style");
      style.textContent = `
        [data-shadow]::before {
          content: '';
          position: absolute;
          inset: 0;
          box-shadow: attr(data-shadow);
          z-index: -1;
        }
      `;
      document.head.appendChild(style);
    }
  }
}

interface PaintMetrics {
  count: number;
  lastPaint: number;
  frequency: number;
}
```

### Example 7: RequestAnimationFrame Optimization

```typescript
// Efficient animation scheduling
class AnimationScheduler {
  private callbacks = new Map<string, AnimationCallback>();
  private rafId: number | null = null;
  private isRunning = false;
  private fps = 0;
  private lastFrameTime = 0;

  // Register animation callback
  add(id: string, callback: (deltaTime: number) => boolean): void {
    this.callbacks.set(id, {
      callback,
      lastTime: performance.now(),
    });

    if (!this.isRunning) {
      this.start();
    }
  }

  // Remove animation callback
  remove(id: string): void {
    this.callbacks.delete(id);

    if (this.callbacks.size === 0) {
      this.stop();
    }
  }

  // Start animation loop
  private start(): void {
    this.isRunning = true;
    this.lastFrameTime = performance.now();
    this.tick();
  }

  // Stop animation loop
  private stop(): void {
    this.isRunning = false;

    if (this.rafId !== null) {
      cancelAnimationFrame(this.rafId);
      this.rafId = null;
    }
  }

  private tick(): void {
    if (!this.isRunning) return;

    const currentTime = performance.now();
    const deltaTime = currentTime - this.lastFrameTime;
    this.lastFrameTime = currentTime;

    // Calculate FPS
    this.fps = 1000 / deltaTime;

    // Execute callbacks
    for (const [id, { callback, lastTime }] of Array.from(this.callbacks)) {
      const frameDelta = currentTime - lastTime;
      const shouldContinue = callback(frameDelta);

      if (!shouldContinue) {
        this.callbacks.delete(id);
      } else {
        this.callbacks.get(id)!.lastTime = currentTime;
      }
    }

    // Schedule next frame
    if (this.callbacks.size > 0) {
      this.rafId = requestAnimationFrame(() => this.tick());
    } else {
      this.stop();
    }
  }

  // Get current FPS
  getFPS(): number {
    return Math.round(this.fps);
  }

  // Check if running
  running(): boolean {
    return this.isRunning;
  }
}

interface AnimationCallback {
  callback: (deltaTime: number) => boolean;
  lastTime: number;
}

// Usage
const scheduler = new AnimationScheduler();

// Smooth position animation
class PositionAnimator {
  private element: HTMLElement;
  private targetX = 0;
  private currentX = 0;
  private readonly EASE = 0.1;

  constructor(element: HTMLElement) {
    this.element = element;
  }

  moveTo(x: number): void {
    this.targetX = x;

    // Start animation
    scheduler.add("position-anim", (deltaTime) => {
      return this.update(deltaTime);
    });
  }

  private update(deltaTime: number): boolean {
    // Smooth interpolation
    const diff = this.targetX - this.currentX;
    this.currentX += diff * this.EASE;

    // Update element
    this.element.style.transform = `translateX(${this.currentX}px)`;

    // Continue if not close enough
    return Math.abs(diff) > 0.5;
  }

  stop(): void {
    scheduler.remove("position-anim");
  }
}

// Frame rate limiter
class FrameRateLimiter {
  private targetFPS: number;
  private frameInterval: number;
  private lastFrameTime = 0;

  constructor(targetFPS: number = 60) {
    this.targetFPS = targetFPS;
    this.frameInterval = 1000 / targetFPS;
  }

  shouldRender(currentTime: number): boolean {
    const elapsed = currentTime - this.lastFrameTime;

    if (elapsed >= this.frameInterval) {
      this.lastFrameTime = currentTime;
      return true;
    }

    return false;
  }

  setTargetFPS(fps: number): void {
    this.targetFPS = fps;
    this.frameInterval = 1000 / fps;
  }
}

// Usage with limiter
const limiter = new FrameRateLimiter(30); // Limit to 30 FPS

scheduler.add("limited-anim", (deltaTime) => {
  if (limiter.shouldRender(performance.now())) {
    // Render frame
    updateUI();
  }
  return true; // Continue animation
});

function updateUI(): void {
  // Update logic
}
```

### Example 8: Intersection Observer for Visibility-Based Rendering

```typescript
// Optimize rendering based on visibility
class VisibilityRenderer {
  private observers = new Map<HTMLElement, IntersectionObserver>();
  private visibleElements = new Set<HTMLElement>();
  private renderCallbacks = new Map<HTMLElement, RenderCallback>();

  // Register element for visibility-based rendering
  register(
    element: HTMLElement,
    callback: RenderCallback,
    options?: IntersectionObserverInit,
  ): void {
    this.renderCallbacks.set(element, callback);

    const observer = new IntersectionObserver(
      (entries) => this.handleIntersection(entries),
      {
        threshold: 0.1,
        rootMargin: "50px",
        ...options,
      },
    );

    observer.observe(element);
    this.observers.set(element, observer);
  }

  private handleIntersection(entries: IntersectionObserverEntry[]): void {
    entries.forEach((entry) => {
      const element = entry.target as HTMLElement;
      const callback = this.renderCallbacks.get(element);

      if (!callback) return;

      if (entry.isIntersecting) {
        // Element is visible
        if (!this.visibleElements.has(element)) {
          this.visibleElements.add(element);
          callback.onVisible();
        }
      } else {
        // Element is hidden
        if (this.visibleElements.has(element)) {
          this.visibleElements.delete(element);
          callback.onHidden();
        }
      }
    });
  }

  // Unregister element
  unregister(element: HTMLElement): void {
    const observer = this.observers.get(element);
    if (observer) {
      observer.disconnect();
      this.observers.delete(element);
    }

    this.renderCallbacks.delete(element);
    this.visibleElements.delete(element);
  }

  // Get visible element count
  getVisibleCount(): number {
    return this.visibleElements.size;
  }

  // Cleanup all
  cleanup(): void {
    for (const observer of this.observers.values()) {
      observer.disconnect();
    }
    this.observers.clear();
    this.renderCallbacks.clear();
    this.visibleElements.clear();
  }
}

interface RenderCallback {
  onVisible: () => void;
  onHidden: () => void;
}

// Usage: Lazy chart rendering
class ChartContainer {
  private element: HTMLElement;
  private chart: any = null;
  private renderer: VisibilityRenderer;

  constructor(element: HTMLElement, renderer: VisibilityRenderer) {
    this.element = element;
    this.renderer = renderer;
    this.setupVisibilityRendering();
  }

  private setupVisibilityRendering(): void {
    this.renderer.register(this.element, {
      onVisible: () => this.renderChart(),
      onHidden: () => this.destroyChart(),
    });
  }

  private renderChart(): void {
    if (this.chart) return;

    console.log("Rendering chart...");

    // Heavy chart rendering only when visible
    this.chart = createChart(this.element);
    this.chart.render();
  }

  private destroyChart(): void {
    if (!this.chart) return;

    console.log("Destroying chart...");

    // Free resources when not visible
    this.chart.destroy();
    this.chart = null;
  }
}

function createChart(container: HTMLElement): any {
  // Chart implementation
  return {
    render() {
      /* Render chart */
    },
    destroy() {
      /* Cleanup */
    },
  };
}

// Usage: Virtualized list
class VirtualListWithVisibility {
  private container: HTMLElement;
  private items: any[] = [];
  private renderer = new VisibilityRenderer();
  private itemHeight = 50;

  constructor(container: HTMLElement) {
    this.container = container;
  }

  setItems(items: any[]): void {
    this.items = items;
    this.render();
  }

  private render(): void {
    this.container.innerHTML = "";

    this.items.forEach((item, index) => {
      const element = this.createPlaceholder(index);
      this.container.appendChild(element);

      // Register for visibility-based rendering
      this.renderer.register(element, {
        onVisible: () => this.renderItem(element, item),
        onHidden: () => this.clearItem(element),
      });
    });
  }

  private createPlaceholder(index: number): HTMLElement {
    const div = document.createElement("div");
    div.className = "list-item-placeholder";
    div.style.height = `${this.itemHeight}px`;
    div.dataset.index = String(index);
    return div;
  }

  private renderItem(element: HTMLElement, item: any): void {
    element.className = "list-item";
    element.innerHTML = `
      <h3>${item.title}</h3>
      <p>${item.description}</p>
    `;
  }

  private clearItem(element: HTMLElement): void {
    element.className = "list-item-placeholder";
    element.innerHTML = "";
  }

  cleanup(): void {
    this.renderer.cleanup();
  }
}
```

### Example 9: Debounced and Throttled Rendering

```typescript
// Optimize frequent render calls
class RenderOptimizer {
  // Debounce: Execute after inactivity period
  static debounce<T extends (...args: any[]) => void>(
    func: T,
    wait: number,
  ): T {
    let timeout: number | null = null;

    return ((...args: any[]) => {
      if (timeout !== null) {
        clearTimeout(timeout);
      }

      timeout = window.setTimeout(() => {
        func(...args);
      }, wait);
    }) as T;
  }

  // Throttle: Execute at most once per period
  static throttle<T extends (...args: any[]) => void>(
    func: T,
    limit: number,
  ): T {
    let inThrottle = false;
    let lastArgs: any[] | null = null;

    return ((...args: any[]) => {
      if (!inThrottle) {
        func(...args);
        inThrottle = true;

        setTimeout(() => {
          inThrottle = false;

          // Call with last args if there were more calls
          if (lastArgs) {
            func(...lastArgs);
            lastArgs = null;
          }
        }, limit);
      } else {
        lastArgs = args;
      }
    }) as T;
  }

  // RAF-based throttle for visual updates
  static rafThrottle<T extends (...args: any[]) => void>(func: T): T {
    let rafId: number | null = null;
    let lastArgs: any[] | null = null;

    return ((...args: any[]) => {
      lastArgs = args;

      if (rafId === null) {
        rafId = requestAnimationFrame(() => {
          func(...lastArgs!);
          rafId = null;
          lastArgs = null;
        });
      }
    }) as T;
  }
}

// Usage: Resize handler
class ResponsiveLayout {
  private container: HTMLElement;
  private debouncedReflow: () => void;
  private throttledUpdate: () => void;

  constructor(container: HTMLElement) {
    this.container = container;

    // Debounce expensive reflow
    this.debouncedReflow = RenderOptimizer.debounce(
      () => this.performReflow(),
      250,
    );

    // Throttle visual updates
    this.throttledUpdate = RenderOptimizer.throttle(
      () => this.updateLayout(),
      100,
    );

    this.setupListeners();
  }

  private setupListeners(): void {
    window.addEventListener("resize", () => {
      // Throttled for smooth visual updates
      this.throttledUpdate();

      // Debounced for expensive calculations
      this.debouncedReflow();
    });
  }

  private updateLayout(): void {
    // Quick visual update
    const width = this.container.offsetWidth;
    console.log("Quick update:", width);
  }

  private performReflow(): void {
    // Expensive layout recalculation
    console.log("Performing reflow...");
    // Heavy calculations here
  }
}

// Usage: Scroll handler
class ScrollHandler {
  private onScroll: () => void;

  constructor(callback: () => void) {
    // Use RAF throttle for smooth 60fps updates
    this.onScroll = RenderOptimizer.rafThrottle(callback);

    window.addEventListener("scroll", this.onScroll, { passive: true });
  }

  destroy(): void {
    window.removeEventListener("scroll", this.onScroll);
  }
}

// Example: Search input with debouncing
class SearchBox {
  private input: HTMLInputElement;
  private performSearch: (query: string) => void;

  constructor(input: HTMLInputElement) {
    this.input = input;

    // Debounce search API calls
    this.performSearch = RenderOptimizer.debounce(
      (query: string) => this.search(query),
      300,
    );

    this.input.addEventListener("input", () => {
      this.performSearch(this.input.value);
    });
  }

  private async search(query: string): Promise<void> {
    if (!query) return;

    console.log("Searching for:", query);
    // API call here
  }
}
```

### Example 10: Style Recalculation Optimization

```typescript
// Minimize style recalculation triggers
class StyleOptimizer {
  // Batch style changes using CSS classes
  static batchStyleChanges(elements: HTMLElement[], className: string): void {
    // Use documentFragment to minimize reflows
    const fragment = document.createDocumentFragment();

    elements.forEach((element) => {
      const clone = element.cloneNode(true) as HTMLElement;
      clone.classList.add(className);
      fragment.appendChild(clone);
      element.replaceWith(clone);
    });
  }

  // Use CSS variables for dynamic styling
  static setCustomProperties(
    element: HTMLElement,
    properties: Record<string, string>,
  ): void {
    // Group all property changes
    const cssText = Object.entries(properties)
      .map(([key, value]) => `--${key}: ${value}`)
      .join("; ");

    element.style.cssText += "; " + cssText;
  }

  // Toggle class instead of inline styles
  static toggleState(element: HTMLElement, state: string): void {
    // Better: use classes
    element.classList.toggle(`state-${state}`);

    // Avoid: direct style manipulation
    // element.style.backgroundColor = 'red';
  }

  // Minimize selector complexity
  static optimizeSelector(element: HTMLElement): void {
    // Add specific class for styling
    element.classList.add("optimized");

    // Avoid: descendant selectors like .container .item .child
    // Use: direct classes like .optimized-child
  }
}

// Example: Dynamic theme switching
class ThemeManager {
  private root: HTMLElement;
  private themes: Map<string, Theme> = new Map();

  constructor() {
    this.root = document.documentElement;
    this.defineThemes();
  }

  private defineThemes(): void {
    this.themes.set("light", {
      primary: "#007bff",
      secondary: "#6c757d",
      background: "#ffffff",
      text: "#212529",
    });

    this.themes.set("dark", {
      primary: "#0d6efd",
      secondary: "#6c757d",
      background: "#212529",
      text: "#f8f9fa",
    });
  }

  // Efficient theme switching using CSS custom properties
  applyTheme(themeName: string): void {
    const theme = this.themes.get(themeName);
    if (!theme) return;

    // Batch all property changes
    requestAnimationFrame(() => {
      StyleOptimizer.setCustomProperties(this.root, theme);

      // Add theme class for additional styling
      this.root.classList.remove("theme-light", "theme-dark");
      this.root.classList.add(`theme-${themeName}`);
    });
  }
}

interface Theme {
  primary: string;
  secondary: string;
  background: string;
  text: string;
}

// Example: Efficient state management
class UIStateManager {
  private elements = new Map<HTMLElement, Set<string>>();

  // Add state to element
  addState(element: HTMLElement, state: string): void {
    const states = this.elements.get(element) || new Set();
    states.add(state);
    this.elements.set(element, states);

    // Apply via class
    element.classList.add(`state-${state}`);
  }

  // Remove state from element
  removeState(element: HTMLElement, state: string): void {
    const states = this.elements.get(element);
    if (!states) return;

    states.delete(state);
    element.classList.remove(`state-${state}`);
  }

  // Toggle state
  toggleState(element: HTMLElement, state: string): void {
    const states = this.elements.get(element) || new Set();

    if (states.has(state)) {
      this.removeState(element, state);
    } else {
      this.addState(element, state);
    }
  }

  // Get element states
  getStates(element: HTMLElement): string[] {
    return Array.from(this.elements.get(element) || []);
  }
}
```

## Real-World Usage

### High-Performance Data Grid

```typescript
// Optimized data grid with virtual scrolling
class DataGrid {
  private container: HTMLElement;
  private fastdom = new FastDOM();
  private visibilityRenderer = new VisibilityRenderer();
  private layerManager = new LayerManager();
  private rows: any[] = [];
  private rowHeight = 40;

  constructor(container: HTMLElement) {
    this.container = container;
    this.optimizeContainer();
    this.setupScrolling();
  }

  private optimizeContainer(): void {
    // Apply containment
    ContainmentManager.applyContentContainment(this.container);

    // Enable content-visibility
    this.container.style.contentVisibility = "auto";

    // GPU acceleration
    LayerManager.promote(this.container);
  }

  private setupScrolling(): void {
    const throttledScroll = RenderOptimizer.rafThrottle(() => {
      this.updateVisibleRows();
    });

    this.container.addEventListener("scroll", throttledScroll, {
      passive: true,
    });
  }

  setData(rows: any[]): void {
    this.rows = rows;
    this.render();
  }

  private render(): void {
    // Use documentFragment for batch insertion
    const fragment = document.createDocumentFragment();

    this.rows.forEach((row, index) => {
      const element = this.createRowElement(row, index);
      fragment.appendChild(element);
    });

    this.container.innerHTML = "";
    this.container.appendChild(fragment);
  }

  private createRowElement(row: any, index: number): HTMLElement {
    const div = document.createElement("div");
    div.className = "grid-row";
    div.style.height = `${this.rowHeight}px`;

    // Apply containment to each row
    ContainmentManager.applyContainment(div, ["layout", "paint"]);

    return div;
  }

  private updateVisibleRows(): void {
    this.fastdom.measure(() => {
      const scrollTop = this.container.scrollTop;
      const viewportHeight = this.container.clientHeight;

      const startIndex = Math.floor(scrollTop / this.rowHeight);
      const endIndex = Math.ceil((scrollTop + viewportHeight) / this.rowHeight);

      this.fastdom.mutate(() => {
        this.renderVisibleRows(startIndex, endIndex);
      });
    });
  }

  private renderVisibleRows(start: number, end: number): void {
    // Render only visible rows
    console.log(`Rendering rows ${start} to ${end}`);
  }
}
```

### Smooth Infinite Scroll

```typescript
// Optimized infinite scroll implementation
class InfiniteScroll {
  private container: HTMLElement;
  private sentinel: HTMLElement;
  private fastdom = new FastDOM();
  private loadMore: () => Promise<any[]>;
  private loading = false;

  constructor(container: HTMLElement, loadMore: () => Promise<any[]>) {
    this.container = container;
    this.loadMore = loadMore;
    this.sentinel = this.createSentinel();
    this.setupIntersectionObserver();
  }

  private createSentinel(): HTMLElement {
    const div = document.createElement("div");
    div.className = "infinite-scroll-sentinel";
    div.style.height = "1px";
    this.container.appendChild(div);
    return div;
  }

  private setupIntersectionObserver(): void {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && !this.loading) {
          this.load();
        }
      },
      { rootMargin: "100px" },
    );

    observer.observe(this.sentinel);
  }

  private async load(): Promise<void> {
    this.loading = true;

    try {
      const items = await this.loadMore();

      await this.fastdom.mutate(() => {
        this.appendItems(items);
      });
    } finally {
      this.loading = false;
    }
  }

  private appendItems(items: any[]): void {
    const fragment = document.createDocumentFragment();

    items.forEach((item) => {
      const element = this.createItemElement(item);
      fragment.appendChild(element);
    });

    this.container.insertBefore(fragment, this.sentinel);
  }

  private createItemElement(item: any): HTMLElement {
    const div = document.createElement("div");
    div.className = "scroll-item";
    div.textContent = item.title;

    // Optimize each item
    ContainmentManager.applyContainment(div, ["layout", "paint"]);

    return div;
  }
}
```

## Production Patterns

### 1. **Read/Write Separation**

Always separate DOM reads from writes to avoid layout thrashing.

### 2. **RequestAnimationFrame for Visual Updates**

Use RAF for all visual updates to sync with browser refresh rate.

### 3. **CSS Containment**

Apply `contain` property to isolate layout/paint work.

### 4. **Layer Promotion for Animations**

Promote animated elements to composite layers using `transform: translateZ(0)`.

### 5. **Visibility-Based Rendering**

Only render visible content using Intersection Observer.

## Best Practices

1. **Batch DOM reads and writes** to prevent layout thrashing
2. **Use RequestAnimationFrame** for all visual updates
3. **Apply CSS containment** to isolate rendering work
4. **Promote to composite layers** for smooth animations
5. **Use will-change sparingly** and remove after animation
6. **Optimize paint areas** with clipping and containment
7. **Debounce/throttle** frequent render triggers
8. **Use Intersection Observer** for visibility-based rendering
9. **Minimize style recalculations** by batching changes
10. **Prefer transform/opacity** for animations (GPU accelerated)
11. **Use documentFragment** for batch DOM insertions
12. **Apply content-visibility: auto** for off-screen content
13. **Avoid forced synchronous layouts** (reading layout properties after writes)
14. **Profile with DevTools** Rendering tab regularly
15. **Target 60fps** (16.67ms per frame) for smooth UX

## 10 Key Takeaways

1. **Layout thrashing is the #1 rendering performance killer** - alternating reads/writes forces multiple synchronous layouts per frame
2. **Transform and opacity are the only truly GPU-accelerated properties** - use them for animations instead of position, width, height
3. **CSS containment isolates rendering work** - `contain: layout paint` prevents child changes from affecting the rest of the page
4. **will-change must be temporary** - adding it permanently wastes GPU memory, add before animation and remove after
5. **RequestAnimationFrame syncs with refresh rate** - always use RAF for visual updates instead of setTimeout
6. **Intersection Observer eliminates scroll handlers** - efficiently detect visibility without constant polling
7. **Composite layers enable GPU acceleration** - but too many layers waste memory, promote strategically
8. **Paint profiling reveals hidden bottlenecks** - enable paint flashing in DevTools to see expensive repaints
9. **content-visibility: auto skips off-screen rendering** - browser doesn't render content until it's near viewport
10. **16.67ms frame budget for 60fps** - profile with DevTools Performance tab to identify frames exceeding budget
