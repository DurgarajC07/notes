# Compositor Thread

> **GPU Acceleration and Smooth Animations**

---

## 🎯 Overview

The **compositor thread** is a separate thread in the browser that handles layer compositing and animations independently of the main thread. It enables smooth 60fps animations even when JavaScript is busy, making it crucial for performance optimization.

**Key Benefits:**

- Animations run on GPU independently of main thread
- No JavaScript blocking during smooth scrolling/animations
- Efficient layer management and composition
- Touch/wheel input handling without main thread involvement

---

## 📚 Core Concepts

### **1. Thread Architecture**

Modern browsers use multiple threads for rendering:

```
┌─────────────────────────────────────────┐
│         Browser Process                 │
├─────────────────────────────────────────┤
│                                         │
│  ┌──────────────┐  ┌──────────────┐   │
│  │ Main Thread  │  │  Compositor  │   │
│  │              │  │    Thread    │   │
│  │ - JavaScript │  │              │   │
│  │ - Layout     │  │ - Transform  │   │
│  │ - Paint      │  │ - Scroll     │   │
│  │ - Style      │  │ - Composite  │   │
│  └──────┬───────┘  └──────┬───────┘   │
│         │                  │           │
│         └──────────┬───────┘           │
│                    │                   │
│            ┌───────▼────────┐          │
│            │   GPU Process  │          │
│            │  - Rasterize   │          │
│            │  - Draw Quads  │          │
│            └────────────────┘          │
└─────────────────────────────────────────┘
```

**Thread Responsibilities:**

```javascript
// Conceptual representation
class BrowserThreads {
  constructor() {
    this.mainThread = new MainThread();
    this.compositorThread = new CompositorThread();
    this.gpuProcess = new GPUProcess();
  }

  // Main thread handles
  class MainThread {
    responsibilities() {
      return [
        'JavaScript execution',
        'Style calculation',
        'Layout (reflow)',
        'Paint (create display lists)',
        'Update layer tree'
      ];
    }
  }

  // Compositor thread handles
  class CompositorThread {
    responsibilities() {
      return [
        'Transform animations',
        'Opacity animations',
        'Smooth scrolling',
        'Touch/wheel input',
        'Layer compositing',
        'Tile management'
      ];
    }
  }

  // GPU process handles
  class GPUProcess {
    responsibilities() {
      return [
        'Rasterization',
        'Texture uploads',
        'Draw quad execution',
        'Hardware acceleration'
      ];
    }
  }
}
```

### **2. Compositor-Only Properties**

Certain CSS properties can be animated entirely on the compositor thread:

```javascript
// Properties that compositor can handle alone
const COMPOSITOR_PROPERTIES = {
  // Transform operations
  transform: [
    "translateX",
    "translateY",
    "translateZ",
    "translate3d",
    "matrix3d",
    "scale",
    "scaleX",
    "scaleY",
    "scaleZ",
    "rotate",
    "rotateX",
    "rotateY",
    "rotateZ",
    "skewX",
    "skewY",
  ],

  // Opacity
  opacity: true,

  // Some filters (GPU accelerated)
  filter: [
    "blur",
    "brightness",
    "contrast",
    "drop-shadow",
    "grayscale",
    "hue-rotate",
    "invert",
    "saturate",
    "sepia",
  ],
};

// Example: Compositor-friendly animation
class CompositorAnimation {
  // ✅ Runs on compositor thread
  animateWithTransform(element) {
    element.style.transition = "transform 0.3s ease-out";
    element.style.transform = "translateX(100px)";
    // No main thread involvement after initial setup
  }

  // ❌ Runs on main thread
  animateWithLeft(element) {
    element.style.transition = "left 0.3s ease-out";
    element.style.left = "100px";
    // Requires layout on every frame
  }

  // Compare performance
  comparePerformance() {
    const element = document.querySelector(".box");

    // Test transform (compositor)
    console.time("transform");
    this.animateWithTransform(element);
    setTimeout(() => {
      console.timeEnd("transform"); // ~1ms setup
    }, 300);

    // Test left (main thread)
    console.time("left");
    this.animateWithLeft(element);
    setTimeout(() => {
      console.timeEnd("left"); // ~16ms per frame
    }, 300);
  }
}
```

### **3. Layer Creation and Promotion**

The compositor works with layers. Understanding when layers are created is crucial:

```javascript
class LayerManagement {
  // Conditions that create a new compositing layer
  static needsLayer(element) {
    const style = getComputedStyle(element);

    return (
      // 3D transforms
      (
        style.transform &&
          (style.transform.includes("3d") ||
            style.transform.includes("translateZ")),
        // will-change hint
        style.willChange === "transform" || style.willChange === "opacity",
        // Opacity < 1 with animations
        parseFloat(style.opacity) < 1 && style.transition.includes("opacity"),
        // Position fixed/sticky
        style.position === "fixed" || style.position === "sticky",
        // CSS filters
        style.filter && style.filter !== "none",
        // Video/Canvas elements
        element.tagName === "VIDEO" || element.tagName === "CANVAS",
        // Overflow scroll
        style.overflow === "scroll" || style.overflow === "auto",
        // Explicit layer promotion
        style.transform === "translateZ(0)" ||
          style.backfaceVisibility === "hidden"
      )
    );
  }

  // Promote element to its own layer
  static promoteToLayer(element) {
    // Method 1: will-change (preferred)
    element.style.willChange = "transform, opacity";

    // Method 2: translateZ hack (older approach)
    // element.style.transform = 'translateZ(0)';

    // Method 3: 3D transform
    // element.style.transform = 'translate3d(0, 0, 0)';
  }

  // Remove layer promotion (important for memory)
  static demoteLayer(element) {
    element.style.willChange = "auto";
  }

  // Check if element has its own layer
  static async hasLayer(element) {
    // Use Chrome DevTools Layers panel or:
    const layers = await this.getCompositingLayers();
    return layers.some((layer) => layer.element === element);
  }
}

// Example usage
const animatedElement = document.querySelector(".animated");

// Promote before animation
LayerManagement.promoteToLayer(animatedElement);

// Animate
animatedElement.style.transform = "translateX(100px)";

// Demote after animation complete
animatedElement.addEventListener(
  "transitionend",
  () => {
    LayerManagement.demoteLayer(animatedElement);
  },
  { once: true },
);
```

### **4. Compositor Thread Animation**

How the compositor handles animations:

```javascript
class CompositorAnimation {
  // Animation lifecycle
  setupAnimation(element, properties) {
    // 1. Main thread: Setup
    const animation = element.animate(
      [
        { transform: "translateX(0)", opacity: 1 },
        { transform: "translateX(100px)", opacity: 0.5 },
      ],
      {
        duration: 1000,
        easing: "ease-out",
        fill: "forwards",
      },
    );

    // 2. Compositor thread takes over
    // Main thread is now free
    // Animation runs at 60fps on compositor

    return animation;
  }

  // Check if animation is compositor-friendly
  isCompositorAnimation(keyframes) {
    const compositorProps = ["transform", "opacity"];

    return keyframes.every((frame) => {
      const properties = Object.keys(frame);
      return properties.every((prop) => compositorProps.includes(prop));
    });
  }

  // Convert non-compositor to compositor animation
  convertToCompositorAnimation(element) {
    // ❌ Bad: Animates left (main thread)
    // element.animate([
    //   { left: '0' },
    //   { left: '100px' }
    // ], { duration: 1000 });

    // ✅ Good: Animates transform (compositor)
    element.animate(
      [{ transform: "translateX(0)" }, { transform: "translateX(100px)" }],
      { duration: 1000 },
    );
  }
}

// Performance monitoring
class AnimationPerformanceMonitor {
  monitor(animation) {
    const startTime = performance.now();
    let frameCount = 0;
    let droppedFrames = 0;

    const checkPerformance = () => {
      frameCount++;
      const elapsed = performance.now() - startTime;
      const expectedFrames = (elapsed / 1000) * 60; // 60fps

      droppedFrames = expectedFrames - frameCount;

      if (animation.playState === "running") {
        requestAnimationFrame(checkPerformance);
      } else {
        console.log({
          totalFrames: frameCount,
          droppedFrames: Math.max(0, Math.round(droppedFrames)),
          avgFPS: (frameCount / (elapsed / 1000)).toFixed(2),
        });
      }
    };

    requestAnimationFrame(checkPerformance);
  }
}
```

### **5. Scroll and Input Handling**

The compositor handles scrolling independently:

```javascript
class CompositorScrolling {
  // Compositor-driven scroll
  enableSmoothScroll(container) {
    // Compositor handles scroll without main thread
    container.style.overflowY = "scroll";
    container.style.willChange = "scroll-position";

    // Optional: smooth scrolling
    container.style.scrollBehavior = "smooth";
  }

  // Passive event listeners (don't block compositor)
  setupPassiveListeners(element) {
    // ✅ Passive: Compositor can scroll immediately
    element.addEventListener(
      "touchstart",
      (e) => {
        console.log("Touch started");
        // Can't call e.preventDefault()
      },
      { passive: true },
    );

    // ❌ Non-passive: Compositor must wait for main thread
    element.addEventListener(
      "touchstart",
      (e) => {
        if (shouldPreventScroll) {
          e.preventDefault(); // Blocks compositor
        }
      },
      { passive: false },
    );
  }

  // Intersection Observer for scroll effects
  setupScrollEffects() {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            // Compositor-friendly animation
            entry.target.style.transform = "translateY(0)";
            entry.target.style.opacity = "1";
          }
        });
      },
      {
        threshold: 0.1,
        rootMargin: "0px 0px -100px 0px",
      },
    );

    document
      .querySelectorAll(".animate-on-scroll")
      .forEach((el) => observer.observe(el));
  }

  // Virtual scrolling (efficient for long lists)
  createVirtualScroller(items, itemHeight) {
    const container = document.querySelector(".scroll-container");
    const viewport = document.querySelector(".viewport");

    let scrollTop = 0;
    const viewportHeight = viewport.clientHeight;
    const totalHeight = items.length * itemHeight;

    // Set total height
    container.style.height = `${totalHeight}px`;

    // Compositor handles scroll
    viewport.addEventListener(
      "scroll",
      () => {
        scrollTop = viewport.scrollTop;

        // Calculate visible range
        const startIndex = Math.floor(scrollTop / itemHeight);
        const endIndex = Math.ceil((scrollTop + viewportHeight) / itemHeight);

        // Render only visible items (on main thread)
        this.renderVisibleItems(items, startIndex, endIndex);
      },
      { passive: true },
    );
  }
}
```

### **6. Tile-Based Rendering**

The compositor uses tiles for efficient rendering:

```javascript
class TileBasedRendering {
  constructor() {
    this.tileSize = 256; // pixels
    this.tiles = new Map();
  }

  // Divide layer into tiles
  createTiles(layer, width, height) {
    const tilesX = Math.ceil(width / this.tileSize);
    const tilesY = Math.ceil(height / this.tileSize);

    for (let y = 0; y < tilesY; y++) {
      for (let x = 0; x < tilesX; x++) {
        const tile = {
          x: x * this.tileSize,
          y: y * this.tileSize,
          width: this.tileSize,
          height: this.tileSize,
          priority: this.calculatePriority(x, y, viewport),
          texture: null,
        };

        this.tiles.set(`${layer.id}-${x}-${y}`, tile);
      }
    }
  }

  // Prioritize visible tiles
  calculatePriority(x, y, viewport) {
    const tileCenter = {
      x: x * this.tileSize + this.tileSize / 2,
      y: y * this.tileSize + this.tileSize / 2,
    };

    const viewportCenter = {
      x: viewport.scrollLeft + viewport.width / 2,
      y: viewport.scrollTop + viewport.height / 2,
    };

    // Distance from viewport center
    const distance = Math.sqrt(
      Math.pow(tileCenter.x - viewportCenter.x, 2) +
        Math.pow(tileCenter.y - viewportCenter.y, 2),
    );

    // Lower number = higher priority
    return distance;
  }

  // Rasterize tiles on GPU
  async rasterizeTiles() {
    // Sort by priority
    const sortedTiles = Array.from(this.tiles.values()).sort(
      (a, b) => a.priority - b.priority,
    );

    // Rasterize visible tiles first
    for (const tile of sortedTiles) {
      if (tile.priority < this.viewportDiagonal) {
        await this.rasterizeTile(tile);
      }
    }
  }

  rasterizeTile(tile) {
    return new Promise((resolve) => {
      // GPU process rasterizes tile
      requestIdleCallback(() => {
        tile.texture = this.createTexture(tile);
        resolve();
      });
    });
  }
}
```

### **7. Debugging Compositor Layers**

Tools and techniques for debugging:

```javascript
class CompositorDebugger {
  // Enable layer visualization
  enableLayerVisualization() {
    // Chrome DevTools: More Tools > Layers
    // or use CSS
    const style = document.createElement("style");
    style.textContent = `
      * {
        /* Show layer borders */
        outline: 1px solid rgba(255, 0, 0, 0.5) !important;
      }
    `;
    document.head.appendChild(style);
  }

  // Count compositing layers
  getLayerCount() {
    // Use Chrome DevTools Protocol
    chrome.devtools.inspectedWindow.eval(
      `(function() {
        const layers = window.internals.layerCount();
        return layers;
      })()`,
      (result) => {
        console.log("Total layers:", result);
      },
    );
  }

  // Analyze layer memory usage
  analyzeLayerMemory() {
    const layers = document.querySelectorAll('[style*="will-change"]');

    let totalMemory = 0;
    layers.forEach((layer) => {
      const rect = layer.getBoundingClientRect();
      const area = rect.width * rect.height;
      // Estimate: 4 bytes per pixel (RGBA)
      const memory = area * 4;
      totalMemory += memory;
    });

    console.log("Estimated layer memory:", {
      bytes: totalMemory,
      kb: (totalMemory / 1024).toFixed(2),
      mb: (totalMemory / 1024 / 1024).toFixed(2),
    });
  }

  // Detect layout thrashing
  detectLayoutThrashing() {
    let readCount = 0;
    let writeCount = 0;
    let thrashingDetected = false;

    const originalOffsetHeight = Object.getOwnPropertyDescriptor(
      HTMLElement.prototype,
      "offsetHeight",
    ).get;

    Object.defineProperty(HTMLElement.prototype, "offsetHeight", {
      get: function () {
        readCount++;
        if (writeCount > 0 && !thrashingDetected) {
          console.warn("Layout thrashing detected!");
          console.trace();
          thrashingDetected = true;
        }
        return originalOffsetHeight.call(this);
      },
    });
  }
}
```

---

## 🔥 Practical Examples

### **Example 1: Optimized Infinite Scroll**

```javascript
class InfiniteScrollOptimized {
  constructor(container) {
    this.container = container;
    this.items = [];
    this.loading = false;

    this.setupCompositorScroll();
  }

  setupCompositorScroll() {
    // Promote container to layer
    this.container.style.willChange = "scroll-position";

    // Use passive listener for compositor
    let lastScrollTop = 0;
    let ticking = false;

    this.container.addEventListener(
      "scroll",
      () => {
        lastScrollTop = this.container.scrollTop;

        if (!ticking) {
          requestAnimationFrame(() => {
            this.handleScroll(lastScrollTop);
            ticking = false;
          });
          ticking = true;
        }
      },
      { passive: true },
    );
  }

  handleScroll(scrollTop) {
    const scrollHeight = this.container.scrollHeight;
    const clientHeight = this.container.clientHeight;

    // Check if near bottom
    if (scrollHeight - scrollTop - clientHeight < 200) {
      this.loadMore();
    }
  }

  async loadMore() {
    if (this.loading) return;

    this.loading = true;
    const newItems = await this.fetchItems();

    // Use DocumentFragment for efficient DOM manipulation
    const fragment = document.createDocumentFragment();

    newItems.forEach((item) => {
      const element = this.createItemElement(item);
      // Promote animated items to layers
      element.style.willChange = "transform, opacity";
      fragment.appendChild(element);
    });

    this.container.appendChild(fragment);

    // Demote after animation
    setTimeout(() => {
      newItems.forEach((_, index) => {
        const element =
          this.container.children[
            this.container.children.length - newItems.length + index
          ];
        element.style.willChange = "auto";
      });
    }, 1000);

    this.loading = false;
  }

  createItemElement(item) {
    const element = document.createElement("div");
    element.className = "item";
    element.textContent = item.title;

    // Compositor-friendly enter animation
    element.style.transform = "translateY(20px)";
    element.style.opacity = "0";

    requestAnimationFrame(() => {
      element.style.transition = "transform 0.3s, opacity 0.3s";
      element.style.transform = "translateY(0)";
      element.style.opacity = "1";
    });

    return element;
  }
}

// Usage
const scroller = new InfiniteScrollOptimized(
  document.querySelector(".scroll-container"),
);
```

### **Example 2: Parallax Scrolling (Compositor-Optimized)**

```javascript
class ParallaxCompositor {
  constructor() {
    this.layers = [];
    this.setupLayers();
    this.setupScroll();
  }

  setupLayers() {
    // Create parallax layers
    const layerConfigs = [
      { selector: ".bg-layer", speed: 0.5 },
      { selector: ".mid-layer", speed: 0.75 },
      { selector: ".fg-layer", speed: 1.0 },
    ];

    layerConfigs.forEach((config) => {
      const elements = document.querySelectorAll(config.selector);
      elements.forEach((element) => {
        // Promote to compositor layer
        element.style.willChange = "transform";

        this.layers.push({
          element,
          speed: config.speed,
        });
      });
    });
  }

  setupScroll() {
    let scrollY = 0;
    let ticking = false;

    // Passive listener for compositor
    window.addEventListener(
      "scroll",
      () => {
        scrollY = window.scrollY;

        if (!ticking) {
          requestAnimationFrame(() => {
            this.updateParallax(scrollY);
            ticking = false;
          });
          ticking = true;
        }
      },
      { passive: true },
    );
  }

  updateParallax(scrollY) {
    this.layers.forEach((layer) => {
      const offset = scrollY * (1 - layer.speed);

      // Use transform for compositor
      layer.element.style.transform = `translate3d(0, ${offset}px, 0)`;
    });
  }
}

// CSS for compositor layers
const parallaxCSS = `
  .bg-layer,
  .mid-layer,
  .fg-layer {
    will-change: transform;
    /* Force layer creation */
    transform: translateZ(0);
  }
`;
```

### **Example 3: Smooth Page Transitions**

```javascript
class PageTransitions {
  constructor() {
    this.currentPage = null;
    this.nextPage = null;
  }

  async transition(fromPage, toPage) {
    // Promote both pages to layers
    fromPage.style.willChange = "transform, opacity";
    toPage.style.willChange = "transform, opacity";

    // Position new page off-screen
    toPage.style.transform = "translateX(100%)";
    toPage.style.position = "fixed";
    toPage.style.top = "0";
    toPage.style.left = "0";
    toPage.style.width = "100%";
    toPage.style.height = "100%";

    // Start transition (compositor handles animation)
    fromPage.style.transition = "transform 0.4s cubic-bezier(0.4, 0.0, 0.2, 1)";
    toPage.style.transition = "transform 0.4s cubic-bezier(0.4, 0.0, 0.2, 1)";

    // Trigger animations
    requestAnimationFrame(() => {
      fromPage.style.transform = "translateX(-100%)";
      toPage.style.transform = "translateX(0)";
    });

    // Wait for transition
    await new Promise((resolve) => {
      toPage.addEventListener("transitionend", resolve, { once: true });
    });

    // Clean up
    fromPage.style.willChange = "auto";
    toPage.style.willChange = "auto";
    fromPage.style.display = "none";
  }

  // Cross-fade transition
  async crossFade(fromPage, toPage) {
    fromPage.style.willChange = "opacity";
    toPage.style.willChange = "opacity";

    toPage.style.opacity = "0";
    toPage.style.position = "fixed";
    toPage.style.top = "0";
    toPage.style.left = "0";
    toPage.style.width = "100%";
    toPage.style.height = "100%";

    fromPage.style.transition = "opacity 0.3s";
    toPage.style.transition = "opacity 0.3s";

    requestAnimationFrame(() => {
      fromPage.style.opacity = "0";
      toPage.style.opacity = "1";
    });

    await new Promise((resolve) => {
      toPage.addEventListener("transitionend", resolve, { once: true });
    });

    fromPage.style.willChange = "auto";
    toPage.style.willChange = "auto";
    fromPage.style.display = "none";
  }
}
```

---

## 🏗️ Best Practices

### **1. Use Compositor-Only Properties**

```css
/* ✅ Good: Compositor-only */
.animate-transform {
  transform: translateX(100px);
  transition: transform 0.3s;
}

.animate-opacity {
  opacity: 0.5;
  transition: opacity 0.3s;
}

/* ❌ Bad: Main thread required */
.animate-left {
  left: 100px;
  transition: left 0.3s;
}

.animate-margin {
  margin-left: 100px;
  transition: margin-left 0.3s;
}
```

### **2. Strategic Layer Promotion**

```javascript
// Promote before animation
element.style.willChange = "transform, opacity";

// Animate
element.style.transform = "scale(1.2)";

// Demote after animation
element.addEventListener(
  "transitionend",
  () => {
    element.style.willChange = "auto";
  },
  { once: true },
);

// ⚠️ Don't over-promote - uses GPU memory
// Only promote elements that will animate soon
```

### **3. Passive Event Listeners**

```javascript
// ✅ Good: Compositor can scroll immediately
element.addEventListener("wheel", handler, { passive: true });
element.addEventListener("touchstart", handler, { passive: true });

// ❌ Bad: Blocks compositor
element.addEventListener("touchstart", (e) => {
  e.preventDefault(); // Compositor must wait
});
```

### **4. Avoid Forcing Synchronous Layout**

```javascript
// ❌ Bad: Forces layout on every frame
function badAnimate() {
  element.style.left = element.offsetLeft + 1 + "px";
  requestAnimationFrame(badAnimate);
}

// ✅ Good: Uses compositor
function goodAnimate() {
  let x = 0;
  function frame() {
    element.style.transform = `translateX(${x++}px)`;
    requestAnimationFrame(frame);
  }
  requestAnimationFrame(frame);
}
```

### **5. Optimize Layer Count**

```javascript
class LayerOptimizer {
  optimizeLayers() {
    // Group related animations
    const container = document.createElement("div");
    container.style.willChange = "transform";

    // All children share same layer
    animatedElements.forEach((el) => {
      container.appendChild(el);
    });

    // Instead of promoting each child individually
  }

  auditLayers() {
    const promoted = document.querySelectorAll('[style*="will-change"]');
    console.log("Promoted layers:", promoted.length);

    if (promoted.length > 20) {
      console.warn("Too many layers! May hurt performance.");
    }
  }
}
```

---

## 🧪 Interview Questions

### **Q1: What is the compositor thread and why is it important?**

**Answer:**

The compositor thread is a separate thread in the browser responsible for combining layers and handling certain animations/scrolling independently of the main thread.

**Importance:**

1. **Non-blocking animations**: Transform and opacity animations run smoothly even when JavaScript is busy on main thread
2. **Smooth scrolling**: Input handling on compositor means scroll doesn't wait for main thread
3. **60fps guarantee**: Compositor can maintain frame rate even during heavy computation
4. **Better responsiveness**: User interactions feel instant

**How it works:**

```
Main Thread              Compositor Thread
    │                          │
    │ 1. Create layers         │
    ├─────────────────────────>│
    │                          │
    │                          │ 2. Handle scroll/input
    │                          │ 3. Animate transform/opacity
    │                          │ 4. Composite layers
    │                          │
    │ 5. JavaScript runs       │ 6. Animation continues
    │    (can be slow)         │    (stays smooth)
```

**Example:**

```javascript
// Compositor thread handles this animation
element.style.transition = "transform 1s";
element.style.transform = "translateX(100px)";

// Even if main thread blocks:
let sum = 0;
for (let i = 0; i < 1000000000; i++) {
  sum += i; // Blocks main thread
}
// Animation still runs smoothly at 60fps!
```

### **Q2: Which CSS properties trigger compositor-only updates vs main thread work?**

**Answer:**

**Compositor-Only (GPU-accelerated, ~1ms):**

- `transform` (translate, rotate, scale)
- `opacity`
- `filter` (most modern browsers)

**Paint Required (CPU, ~5-10ms):**

- `color`
- `background-color`
- `box-shadow`
- `border-radius`
- `visibility`

**Layout + Paint Required (CPU, ~16-50ms):**

- `width`, `height`
- `margin`, `padding`
- `position`, `top`, `left`
- `font-size`
- `display`

**Performance comparison:**

```javascript
// Benchmark
class PropertyPerformance {
  benchmark(property, value, iterations = 1000) {
    const element = document.createElement("div");
    document.body.appendChild(element);

    const start = performance.now();

    for (let i = 0; i < iterations; i++) {
      element.style[property] = value;
      // Force style recalc
      getComputedStyle(element)[property];
    }

    const duration = performance.now() - start;
    console.log(`${property}: ${duration.toFixed(2)}ms`);

    document.body.removeChild(element);
  }

  runAll() {
    this.benchmark("transform", "translateX(100px)"); // ~10ms
    this.benchmark("opacity", "0.5"); // ~12ms
    this.benchmark("backgroundColor", "red"); // ~85ms
    this.benchmark("left", "100px"); // ~245ms
  }
}
```

### **Q3: What is layer promotion and when should you use it?**

**Answer:**

Layer promotion creates a separate compositing layer for an element, allowing the compositor thread to handle its animations independently.

**When to promote:**

1. **Frequent animations**: Elements that animate transform/opacity repeatedly
2. **Scrolling containers**: Elements with `overflow: scroll`
3. **Fixed position**: Elements with `position: fixed`
4. **Complex effects**: Elements with filters or 3D transforms

**How to promote:**

```css
/* Method 1: will-change (preferred) */
.animated {
  will-change: transform, opacity;
}

/* Method 2: 3D transform (older) */
.animated {
  transform: translateZ(0);
}

/* Method 3: backface-visibility */
.animated {
  backface-visibility: hidden;
}
```

**Best practices:**

```javascript
class LayerPromotion {
  // ✅ Good: Promote temporarily
  animateTemporary(element) {
    // Promote before animation
    element.style.willChange = "transform";

    // Animate
    element.classList.add("animating");

    // Demote after animation
    element.addEventListener(
      "transitionend",
      () => {
        element.style.willChange = "auto";
      },
      { once: true },
    );
  }

  // ❌ Bad: Promote everything forever
  promoteTooMuch() {
    document.querySelectorAll("*").forEach((el) => {
      el.style.willChange = "transform";
      // Wastes GPU memory!
    });
  }

  // ✅ Good: Promote only what needs it
  promoteStrategically() {
    document.querySelectorAll(".will-animate").forEach((el) => {
      el.style.willChange = "transform, opacity";
    });

    // Monitor memory usage
    if (performance.memory.usedJSHeapSize > threshold) {
      console.warn("Too much GPU memory used!");
    }
  }
}
```

**Trade-offs:**

- **Pro**: Smooth 60fps animations
- **Con**: Increased GPU memory usage
- **Con**: Layer management overhead
- **Rule**: Only promote 5-10 elements max at once

### **Q4: How do passive event listeners improve scrolling performance?**

**Answer:**

Passive event listeners tell the browser that `preventDefault()` will NOT be called, allowing the compositor to scroll immediately without waiting for the event handler.

**Problem without passive:**

```javascript
// ❌ Blocks compositor
element.addEventListener("touchstart", (e) => {
  // Browser must wait to see if preventDefault is called
  if (someCondition) {
    e.preventDefault();
  }
  // Meanwhile, scroll is blocked (jank!)
});
```

**Solution with passive:**

```javascript
// ✅ Compositor scrolls immediately
element.addEventListener(
  "touchstart",
  (e) => {
    // Browser knows preventDefault won't be called
    // Scroll happens on compositor thread
    // e.preventDefault() would be ignored
  },
  { passive: true },
);
```

**Performance impact:**

```javascript
class PassiveListenerDemo {
  // Non-passive: ~200ms scroll delay
  nonPassive() {
    element.addEventListener("touchstart", (e) => {
      // Compositor waits for this to execute
      expensiveOperation(); // 100ms
    });
  }

  // Passive: No delay, smooth scroll
  passive() {
    element.addEventListener(
      "touchstart",
      (e) => {
        // Compositor scrolls immediately
        expensiveOperation(); // Runs in parallel
      },
      { passive: true },
    );
  }
}
```

**Browser defaults:**

```javascript
// These are passive by default in modern browsers:
window.addEventListener("wheel", handler); // passive: true
document.addEventListener("touchstart", handler); // passive: true
document.addEventListener("touchmove", handler); // passive: true

// To override:
element.addEventListener("touchstart", handler, { passive: false });
```

### **Q5: What are the memory implications of having too many compositor layers?**

**Answer:**

Each compositor layer consumes GPU memory, which is limited (typically 256MB-1GB on mobile).

**Memory calculation:**

```javascript
class LayerMemoryCalculator {
  calculateLayerMemory(element) {
    const rect = element.getBoundingClientRect();
    const width = rect.width;
    const height = rect.height;

    // Each pixel uses 4 bytes (RGBA)
    const pixelCount = width * height;
    const bytesPerPixel = 4;
    const memory = pixelCount * bytesPerPixel;

    return {
      pixels: pixelCount,
      bytes: memory,
      kb: (memory / 1024).toFixed(2),
      mb: (memory / 1024 / 1024).toFixed(2),
    };
  }

  auditAllLayers() {
    const layers = document.querySelectorAll('[style*="will-change"]');
    let totalMemory = 0;

    layers.forEach((layer) => {
      const memory = this.calculateLayerMemory(layer);
      totalMemory += memory.bytes;
      console.log(layer, memory);
    });

    console.log("Total GPU memory:", {
      mb: (totalMemory / 1024 / 1024).toFixed(2),
      warning: totalMemory > 50 * 1024 * 1024 ? "Too much memory!" : "OK",
    });
  }
}

// Example calculation:
// 1920x1080 element = 2,073,600 pixels
// × 4 bytes = 8,294,400 bytes = ~8MB
// 10 such layers = 80MB GPU memory
```

**Consequences of too many layers:**

1. **Memory pressure**: GPU runs out of memory
2. **Slower compositing**: More layers = more work for GPU
3. **Increased power usage**: More GPU work = faster battery drain
4. **Browser limits**: Browser may refuse to create more layers

**Best practices:**

```javascript
class LayerOptimization {
  // ❌ Bad: Too many layers
  badApproach() {
    document.querySelectorAll(".item").forEach((item) => {
      item.style.willChange = "transform"; // 100+ layers!
    });
  }

  // ✅ Good: Minimal layers
  goodApproach() {
    // Only promote what's currently animating
    const activeItems = document.querySelectorAll(".item.animating");
    activeItems.forEach((item) => {
      item.style.willChange = "transform";
    });

    // Demote when done
    item.addEventListener("animationend", () => {
      item.style.willChange = "auto";
    });
  }

  // ✅ Better: Group in container
  bestApproach() {
    const container = document.querySelector(".items");
    container.style.willChange = "transform";
    // All children share one layer
  }
}
```

---

## 📚 Key Takeaways

1. **Compositor thread runs independently**: Animations continue smoothly even when main thread is busy
2. **Transform and opacity are special**: Only properties compositor can animate without main thread
3. **Layer promotion is expensive**: Each layer uses GPU memory, promote strategically
4. **Passive listeners are crucial**: Allow compositor to scroll immediately without waiting
5. **Tile-based rendering**: Large layers divided into tiles for efficient GPU usage
6. **will-change is a hint**: Browser may or may not create layer depending on heuristics
7. **Demote after animation**: Clean up layers to free GPU memory
8. **Monitor layer count**: Keep under 10-20 promoted layers for best performance
9. **GPU vs CPU trade-off**: Layer promotion moves work to GPU but uses memory
10. **Input handling on compositor**: Touch/wheel events handled without main thread involvement

---

**Master the compositor thread for buttery-smooth 60fps animations!**
