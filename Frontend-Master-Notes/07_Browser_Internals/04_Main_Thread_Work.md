# Main Thread Work

> **Long Tasks and Blocking**

---

## 🎯 Overview

The **main thread** is the single JavaScript execution thread in browsers responsible for parsing HTML, executing JavaScript, computing styles, performing layout, and painting. Understanding main thread work is essential for avoiding jank and ensuring smooth 60fps performance.

**Key Responsibilities:**

- JavaScript execution
- Style calculation
- Layout (reflow)
- Paint
- Handling user events
- Garbage collection

**The Problem:** Main thread is single-threaded and blocking - long tasks freeze the UI!

---

## 📚 Core Concepts

### **1. Main Thread Anatomy**

The browser's main thread handles multiple responsibilities:

```javascript
// Conceptual representation of main thread work
class MainThread {
  constructor() {
    this.taskQueue = [];
    this.isBlocked = false;
  }

  // Main thread event loop
  async runEventLoop() {
    while (true) {
      // 1. Run microtasks (Promises)
      await this.runMicrotasks();

      // 2. Run next macrotask
      const task = this.taskQueue.shift();
      if (task) {
        this.executeTask(task);
      }

      // 3. Update rendering (if needed)
      if (this.needsRender()) {
        await this.render();
      }

      // 4. Run idle callbacks
      if (this.hasIdleTime()) {
        this.runIdleCallbacks();
      }
    }
  }

  async render() {
    // Rendering pipeline
    this.recalculateStyle(); // ~2-5ms
    this.layout(); // ~5-15ms
    this.paint(); // ~5-10ms
    this.composite(); // ~1-2ms
    // Total: ~13-32ms (leaves 3-3ms for 60fps budget)
  }

  executeTask(task) {
    const start = performance.now();
    task.execute();
    const duration = performance.now() - start;

    // Long task: > 50ms
    if (duration > 50) {
      console.warn("Long task detected:", duration + "ms");
      // Causes jank, dropped frames
    }
  }
}
```

### **2. The 16.67ms Frame Budget**

For 60fps, each frame has 16.67ms budget:

```javascript
class FrameBudget {
  constructor() {
    this.fps = 60;
    this.frameBudget = 1000 / this.fps; // 16.67ms
  }

  // Break down frame budget
  getFrameBudget() {
    return {
      total: this.frameBudget, // 16.67ms

      // Typical breakdown:
      javascript: 3, // ~3ms
      styleCalculation: 2, // ~2ms
      layout: 4, // ~4ms
      paint: 5, // ~5ms
      composite: 2, // ~2ms
      // Remaining: ~0.67ms buffer

      // If any phase exceeds budget = dropped frame
    };
  }

  measureFrameWork() {
    let frameStart = performance.now();
    let frameCount = 0;
    let longFrames = 0;

    const measureFrame = () => {
      const frameTime = performance.now() - frameStart;

      if (frameTime > this.frameBudget) {
        longFrames++;
        console.warn(
          `Frame ${frameCount}: ${frameTime.toFixed(2)}ms (DROPPED)`,
        );
      }

      frameCount++;
      frameStart = performance.now();
      requestAnimationFrame(measureFrame);
    };

    requestAnimationFrame(measureFrame);

    // Report after 5 seconds
    setTimeout(() => {
      console.log({
        totalFrames: frameCount,
        longFrames: longFrames,
        percentageDropped: ((longFrames / frameCount) * 100).toFixed(2) + "%",
      });
    }, 5000);
  }
}

// Monitor frame budget
const budget = new FrameBudget();
budget.measureFrameWork();
```

### **3. Long Tasks**

Tasks over 50ms are considered "long tasks" and cause jank:

```javascript
class LongTaskDetector {
  constructor() {
    this.observer = null;
    this.longTasks = [];
  }

  start() {
    // Use PerformanceObserver to detect long tasks
    this.observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > 50) {
          this.longTasks.push({
            duration: entry.duration,
            startTime: entry.startTime,
            name: entry.name,
            attribution: entry.attribution,
          });

          console.warn("Long Task:", {
            duration: entry.duration.toFixed(2) + "ms",
            startTime: entry.startTime.toFixed(2) + "ms",
            name: entry.name,
          });
        }
      }
    });

    this.observer.observe({ entryTypes: ["longtask"] });
  }

  stop() {
    if (this.observer) {
      this.observer.disconnect();
    }
  }

  getReport() {
    const totalDuration = this.longTasks.reduce(
      (sum, task) => sum + task.duration,
      0,
    );

    return {
      count: this.longTasks.length,
      totalBlockingTime: totalDuration.toFixed(2) + "ms",
      tasks: this.longTasks,
      averageDuration:
        (totalDuration / this.longTasks.length).toFixed(2) + "ms",
    };
  }
}

// Example: Detecting long tasks
const detector = new LongTaskDetector();
detector.start();

// Simulate long task
function heavyComputation() {
  const start = Date.now();
  let result = 0;
  while (Date.now() - start < 100) {
    // 100ms task
    result += Math.random();
  }
  return result;
}

heavyComputation(); // Will be detected as long task

setTimeout(() => {
  console.log("Long Task Report:", detector.getReport());
  detector.stop();
}, 5000);
```

### **4. Breaking Up Long Tasks**

Strategies to split work into smaller chunks:

```javascript
class TaskScheduler {
  // ❌ Bad: Blocks main thread
  processAllItemsSync(items) {
    const start = performance.now();

    items.forEach((item) => {
      this.processItem(item); // Heavy computation
    });

    const duration = performance.now() - start;
    console.log(`Processed ${items.length} items in ${duration}ms`);
    // If duration > 50ms, this is a long task!
  }

  // ✅ Good: Split with setTimeout
  async processItemsAsync(items) {
    const batchSize = 10; // Process 10 items per frame

    for (let i = 0; i < items.length; i += batchSize) {
      const batch = items.slice(i, i + batchSize);

      batch.forEach((item) => {
        this.processItem(item);
      });

      // Yield to browser
      await new Promise((resolve) => setTimeout(resolve, 0));
    }
  }

  // ✅ Better: Split with requestIdleCallback
  processItemsIdle(items) {
    let index = 0;

    const processBatch = (deadline) => {
      // Process while we have idle time
      while (deadline.timeRemaining() > 0 && index < items.length) {
        this.processItem(items[index]);
        index++;
      }

      // Schedule next batch if needed
      if (index < items.length) {
        requestIdleCallback(processBatch);
      } else {
        console.log("All items processed!");
      }
    };

    requestIdleCallback(processBatch);
  }

  // ✅ Best: Scheduler API (if available)
  async processItemsScheduler(items) {
    if ("scheduler" in window) {
      for (const item of items) {
        await scheduler.yield(); // Yield between items
        this.processItem(item);
      }
    } else {
      // Fallback to requestIdleCallback
      this.processItemsIdle(items);
    }
  }

  processItem(item) {
    // Heavy computation
    let sum = 0;
    for (let i = 0; i < 100000; i++) {
      sum += Math.random();
    }
    item.result = sum;
  }
}

// Usage comparison
const scheduler = new TaskScheduler();
const items = Array.from({ length: 1000 }, (_, i) => ({ id: i }));

// This will freeze UI for ~500ms
// scheduler.processAllItemsSync(items);

// This yields to browser, UI stays responsive
scheduler.processItemsIdle(items);
```

### **5. Input Delay and Responsiveness**

Long tasks increase input delay:

```javascript
class InputResponsiveness {
  constructor() {
    this.interactions = [];
  }

  // Measure input delay
  measureInputDelay() {
    // Track pointer events
    ["click", "keydown", "touchstart"].forEach((eventType) => {
      document.addEventListener(eventType, (e) => {
        const eventTime = e.timeStamp;

        // Measure delay until next frame
        requestAnimationFrame((frameTime) => {
          const delay = frameTime - eventTime;

          this.interactions.push({
            type: eventType,
            delay: delay,
            timestamp: eventTime,
          });

          if (delay > 100) {
            console.warn(
              `Slow interaction (${eventType}): ${delay.toFixed(2)}ms`,
            );
          }
        });
      });
    });
  }

  // Use Event Timing API
  measureWithEventTiming() {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        const inputDelay = entry.processingStart - entry.startTime;
        const processingTime = entry.processingEnd - entry.processingStart;
        const presentationDelay =
          entry.startTime + entry.duration - entry.processingEnd;

        console.log({
          name: entry.name,
          inputDelay: inputDelay.toFixed(2) + "ms",
          processingTime: processingTime.toFixed(2) + "ms",
          presentationDelay: presentationDelay.toFixed(2) + "ms",
          totalDuration: entry.duration.toFixed(2) + "ms",
        });

        // First Input Delay (FID)
        if (entry.name === "first-input") {
          console.log("FID:", inputDelay.toFixed(2) + "ms");
        }
      }
    });

    observer.observe({
      type: "event",
      buffered: true,
      durationThreshold: 16, // Report events > 16ms
    });
  }

  // Optimize event handler
  optimizeEventHandler(element) {
    // ❌ Bad: Heavy work in handler
    element.addEventListener("click", (e) => {
      const result = heavyComputation(); // Blocks main thread
      updateUI(result);
    });

    // ✅ Good: Defer heavy work
    element.addEventListener("click", async (e) => {
      // Quick UI feedback first
      element.classList.add("loading");

      // Defer heavy work
      await new Promise((resolve) => setTimeout(resolve, 0));
      const result = heavyComputation();

      // Update UI
      element.classList.remove("loading");
      updateUI(result);
    });
  }
}

// Monitor input responsiveness
const responsiveness = new InputResponsiveness();
responsiveness.measureInputDelay();
responsiveness.measureWithEventTiming();
```

### **6. Style Calculation Performance**

Style recalculation can be expensive:

```javascript
class StyleOptimization {
  // Measure style calculation time
  measureStyleCalc() {
    const start = performance.now();

    // Force style recalculation
    const elements = document.querySelectorAll("*");
    elements.forEach((el) => {
      getComputedStyle(el).color; // Forces style calc
    });

    const duration = performance.now() - start;
    console.log("Style calculation:", duration.toFixed(2) + "ms");
  }

  // Reduce selector complexity
  optimizeSelectors() {
    // ❌ Bad: Complex selectors
    const badCSS = `
      div.container > ul.list li.item:nth-child(odd) span.text {
        color: red;
      }
    `;

    // ✅ Good: Simple, specific selectors
    const goodCSS = `
      .item-text-odd {
        color: red;
      }
    `;

    // Use BEM or similar methodology
    const bemCSS = `
      .list__item--odd .list__text {
        color: red;
      }
    `;
  }

  // Minimize affected elements
  containStyle() {
    // Use CSS containment
    const style = document.createElement("style");
    style.textContent = `
      .widget {
        /* Tell browser this element is independent */
        contain: layout style;
      }
      
      .isolated-component {
        /* Maximum containment */
        contain: strict;
        /* Equivalent to: contain: size layout style paint; */
      }
    `;
    document.head.appendChild(style);
  }

  // Batch style changes
  batchStyleChanges(elements) {
    // ❌ Bad: Causes multiple style recalcs
    elements.forEach((el) => {
      el.style.color = "red";
      el.style.fontSize = "16px";
      el.style.padding = "10px";
    });

    // ✅ Good: Single style recalc
    const className = "styled-element";
    elements.forEach((el) => {
      el.className = className;
    });
  }
}
```

### **7. Layout Thrashing (Forced Synchronous Layout)**

Reading layout properties forces synchronous layout:

```javascript
class LayoutThrashing {
  // ❌ Bad: Layout thrashing (Read-Write-Read-Write)
  badLayoutThrashing() {
    const elements = document.querySelectorAll(".box");

    elements.forEach((element) => {
      // Read (forces layout)
      const height = element.offsetHeight;

      // Write
      element.style.height = height + 10 + "px";

      // Read again (forces layout AGAIN)
      const width = element.offsetWidth;

      // Write again
      element.style.width = width + 10 + "px";
    });
    // Causes N * 2 layouts for N elements!
  }

  // ✅ Good: Batch reads, then batch writes
  goodLayoutBatching() {
    const elements = Array.from(document.querySelectorAll(".box"));

    // 1. Batch all reads (single layout)
    const measurements = elements.map((element) => ({
      height: element.offsetHeight,
      width: element.offsetWidth,
    }));

    // 2. Batch all writes (no layout until next frame)
    elements.forEach((element, i) => {
      element.style.height = measurements[i].height + 10 + "px";
      element.style.width = measurements[i].width + 10 + "px";
    });
    // Only 1 layout triggered!
  }

  // Use fastdom library for automatic batching
  useFastdom() {
    const elements = document.querySelectorAll(".box");

    elements.forEach((element) => {
      // fastdom batches reads
      fastdom.measure(() => {
        const height = element.offsetHeight;
        const width = element.offsetWidth;

        // fastdom batches writes
        fastdom.mutate(() => {
          element.style.height = height + 10 + "px";
          element.style.width = width + 10 + "px";
        });
      });
    });
  }

  // Detect layout thrashing
  detectThrashing() {
    let layoutCount = 0;

    // Monkey-patch layout properties
    const properties = [
      "offsetTop",
      "offsetLeft",
      "offsetWidth",
      "offsetHeight",
      "clientTop",
      "clientLeft",
      "clientWidth",
      "clientHeight",
      "scrollTop",
      "scrollLeft",
      "scrollWidth",
      "scrollHeight",
    ];

    properties.forEach((prop) => {
      const original = Object.getOwnPropertyDescriptor(
        HTMLElement.prototype,
        prop,
      ).get;

      Object.defineProperty(HTMLElement.prototype, prop, {
        get: function () {
          layoutCount++;
          if (layoutCount > 10) {
            console.warn("Possible layout thrashing detected!");
            console.trace();
          }
          return original.call(this);
        },
      });
    });

    // Reset count each frame
    requestAnimationFrame(function resetCount() {
      layoutCount = 0;
      requestAnimationFrame(resetCount);
    });
  }
}
```

### **8. Web Workers for Heavy Computation**

Offload work to background threads:

```javascript
// Main thread
class WorkerManager {
  constructor() {
    this.worker = new Worker("heavy-computation.worker.js");
    this.setupMessageHandler();
  }

  setupMessageHandler() {
    this.worker.addEventListener("message", (e) => {
      const { type, result } = e.data;

      if (type === "computation-complete") {
        this.handleResult(result);
      }
    });
  }

  // Offload heavy computation
  async processData(data) {
    return new Promise((resolve) => {
      const handleMessage = (e) => {
        if (e.data.type === "computation-complete") {
          this.worker.removeEventListener("message", handleMessage);
          resolve(e.data.result);
        }
      };

      this.worker.addEventListener("message", handleMessage);
      this.worker.postMessage({
        type: "process-data",
        data: data,
      });
    });
  }

  handleResult(result) {
    // Update UI (fast, on main thread)
    document.getElementById("result").textContent = result;
  }

  terminate() {
    this.worker.terminate();
  }
}

// Worker thread (heavy-computation.worker.js)
self.addEventListener("message", (e) => {
  const { type, data } = e.data;

  if (type === "process-data") {
    // Heavy computation on worker thread
    const result = performHeavyComputation(data);

    // Send result back to main thread
    self.postMessage({
      type: "computation-complete",
      result: result,
    });
  }
});

function performHeavyComputation(data) {
  // CPU-intensive work
  let result = 0;
  for (let i = 0; i < 1000000000; i++) {
    result += Math.sqrt(i);
  }
  return result;
}

// Usage
const manager = new WorkerManager();
manager.processData([1, 2, 3, 4, 5]).then((result) => {
  console.log("Computation complete:", result);
});
```

---

## 🔥 Practical Examples

### **Example 1: Optimized Data Table Rendering**

```javascript
class DataTableOptimized {
  constructor(container, data) {
    this.container = container;
    this.data = data;
    this.batchSize = 50;
  }

  // ❌ Bad: Renders all rows at once (blocks main thread)
  renderAllSync() {
    const start = performance.now();

    this.data.forEach((row) => {
      const tr = this.createRow(row);
      this.container.appendChild(tr);
    });

    const duration = performance.now() - start;
    console.log(`Rendered ${this.data.length} rows in ${duration}ms`);
    // For 10,000 rows: ~500ms (blocks UI!)
  }

  // ✅ Good: Renders in batches
  async renderInBatches() {
    for (let i = 0; i < this.data.length; i += this.batchSize) {
      const batch = this.data.slice(i, i + this.batchSize);

      // Use DocumentFragment for efficiency
      const fragment = document.createDocumentFragment();
      batch.forEach((row) => {
        fragment.appendChild(this.createRow(row));
      });

      this.container.appendChild(fragment);

      // Yield to browser
      await new Promise((resolve) => setTimeout(resolve, 0));

      // Update progress
      const progress = (
        ((i + this.batchSize) / this.data.length) *
        100
      ).toFixed(0);
      console.log(`Progress: ${progress}%`);
    }
  }

  // ✅ Better: Use requestIdleCallback
  renderDuringIdle() {
    let index = 0;
    const fragment = document.createDocumentFragment();

    const renderBatch = (deadline) => {
      // Render while we have idle time (max 50ms chunks)
      const batchEnd = Math.min(index + this.batchSize, this.data.length);

      while (index < batchEnd && deadline.timeRemaining() > 0) {
        fragment.appendChild(this.createRow(this.data[index]));
        index++;
      }

      // Append batch
      if (fragment.childNodes.length > 0) {
        this.container.appendChild(fragment.cloneNode(true));
        while (fragment.firstChild) {
          fragment.removeChild(fragment.firstChild);
        }
      }

      // Continue if more rows
      if (index < this.data.length) {
        requestIdleCallback(renderBatch);
      } else {
        console.log("All rows rendered!");
      }
    };

    requestIdleCallback(renderBatch);
  }

  // ✅ Best: Virtual scrolling (only render visible)
  setupVirtualScroll() {
    const rowHeight = 40;
    const visibleRows = Math.ceil(this.container.clientHeight / rowHeight);
    const buffer = 5; // Extra rows for smooth scrolling

    let scrollTop = 0;

    this.container.addEventListener(
      "scroll",
      () => {
        scrollTop = this.container.scrollTop;

        requestAnimationFrame(() => {
          const startIndex = Math.floor(scrollTop / rowHeight) - buffer;
          const endIndex = startIndex + visibleRows + buffer * 2;

          this.renderVisibleRows(
            Math.max(0, startIndex),
            Math.min(this.data.length, endIndex),
          );
        });
      },
      { passive: true },
    );

    // Initial render
    this.renderVisibleRows(0, visibleRows + buffer);
  }

  renderVisibleRows(start, end) {
    // Clear container
    this.container.innerHTML = "";

    // Add spacer for scroll position
    const spacerTop = document.createElement("div");
    spacerTop.style.height = start * 40 + "px";
    this.container.appendChild(spacerTop);

    // Render visible rows
    const fragment = document.createDocumentFragment();
    for (let i = start; i < end; i++) {
      if (this.data[i]) {
        fragment.appendChild(this.createRow(this.data[i]));
      }
    }
    this.container.appendChild(fragment);

    // Add spacer for remaining rows
    const spacerBottom = document.createElement("div");
    spacerBottom.style.height = (this.data.length - end) * 40 + "px";
    this.container.appendChild(spacerBottom);
  }

  createRow(data) {
    const tr = document.createElement("tr");
    tr.innerHTML = `
      <td>${data.id}</td>
      <td>${data.name}</td>
      <td>${data.email}</td>
    `;
    return tr;
  }
}

// Usage
const data = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  name: `User ${i}`,
  email: `user${i}@example.com`,
}));

const table = new DataTableOptimized(
  document.querySelector("#table-container"),
  data,
);

// Choose rendering strategy
table.renderDuringIdle(); // Non-blocking
// table.setupVirtualScroll(); // For very large datasets
```

### **Example 2: Debounced Search with Main Thread Optimization**

```javascript
class OptimizedSearch {
  constructor(input, resultsContainer) {
    this.input = input;
    this.resultsContainer = resultsContainer;
    this.searchWorker = new Worker("search.worker.js");
    this.abortController = null;

    this.setupEventListeners();
  }

  setupEventListeners() {
    let timeoutId;

    this.input.addEventListener("input", (e) => {
      const query = e.target.value;

      // Debounce input
      clearTimeout(timeoutId);
      timeoutId = setTimeout(() => {
        this.search(query);
      }, 300);
    });

    // Handle results from worker
    this.searchWorker.addEventListener("message", (e) => {
      const { results, query } = e.data;
      this.displayResults(results, query);
    });
  }

  search(query) {
    if (query.length < 2) {
      this.resultsContainer.innerHTML = "";
      return;
    }

    // Show loading state (quick, main thread)
    this.showLoading();

    // Offload search to worker (heavy computation)
    this.searchWorker.postMessage({
      type: "search",
      query: query,
      data: this.getAllData(),
    });
  }

  showLoading() {
    this.resultsContainer.innerHTML = '<div class="loading">Searching...</div>';
  }

  displayResults(results, query) {
    // Clear results
    this.resultsContainer.innerHTML = "";

    if (results.length === 0) {
      this.resultsContainer.innerHTML = "<div>No results found</div>";
      return;
    }

    // Render results in batches (avoid blocking)
    this.renderResultsBatched(results, query);
  }

  async renderResultsBatched(results, query) {
    const batchSize = 20;

    for (let i = 0; i < results.length; i += batchSize) {
      const batch = results.slice(i, i + batchSize);
      const fragment = document.createDocumentFragment();

      batch.forEach((result) => {
        const div = this.createResultElement(result, query);
        fragment.appendChild(div);
      });

      this.resultsContainer.appendChild(fragment);

      // Yield to browser
      if (i + batchSize < results.length) {
        await new Promise((resolve) => setTimeout(resolve, 0));
      }
    }
  }

  createResultElement(result, query) {
    const div = document.createElement("div");
    div.className = "result";

    // Highlight query in result
    const highlighted = result.text.replace(
      new RegExp(query, "gi"),
      (match) => `<mark>${match}</mark>`,
    );

    div.innerHTML = `
      <h3>${result.title}</h3>
      <p>${highlighted}</p>
    `;

    return div;
  }

  getAllData() {
    // In real app, this would come from API/database
    return Array.from({ length: 10000 }, (_, i) => ({
      id: i,
      title: `Item ${i}`,
      text: `Description for item ${i} with lots of searchable text`,
    }));
  }
}

// Worker (search.worker.js)
self.addEventListener("message", (e) => {
  const { type, query, data } = e.data;

  if (type === "search") {
    // Heavy search computation on worker thread
    const results = data.filter((item) => {
      return (
        item.title.toLowerCase().includes(query.toLowerCase()) ||
        item.text.toLowerCase().includes(query.toLowerCase())
      );
    });

    // Sort by relevance
    results.sort((a, b) => {
      const aScore = calculateRelevance(a, query);
      const bScore = calculateRelevance(b, query);
      return bScore - aScore;
    });

    // Send results back
    self.postMessage({
      results: results.slice(0, 100), // Top 100 results
      query: query,
    });
  }
});

function calculateRelevance(item, query) {
  let score = 0;
  const lowerQuery = query.toLowerCase();

  if (item.title.toLowerCase().includes(lowerQuery)) {
    score += 10;
  }
  if (item.text.toLowerCase().includes(lowerQuery)) {
    score += 5;
  }

  return score;
}
```

### **Example 3: Smooth Animations Without Blocking**

```javascript
class SmoothAnimationManager {
  constructor() {
    this.animations = new Map();
  }

  // Animate element without blocking main thread
  animate(element, keyframes, options) {
    // Use Web Animations API (compositor-friendly)
    const animation = element.animate(keyframes, options);

    this.animations.set(element, animation);

    return animation;
  }

  // Complex animation with data processing
  async animateWithDataProcessing(element, data) {
    // 1. Process data on worker (heavy computation)
    const processed = await this.processDataOnWorker(data);

    // 2. Apply results with compositor-friendly animation
    const keyframes = this.createKeyframesFromData(processed);

    // 3. Animate (runs on compositor thread)
    return this.animate(element, keyframes, {
      duration: 1000,
      easing: "ease-out",
    });
  }

  processDataOnWorker(data) {
    return new Promise((resolve) => {
      const worker = new Worker("process.worker.js");

      worker.addEventListener("message", (e) => {
        resolve(e.data.result);
        worker.terminate();
      });

      worker.postMessage({ data });
    });
  }

  createKeyframesFromData(data) {
    return [
      {
        transform: "translateX(0) scale(1)",
        opacity: 1,
      },
      {
        transform: `translateX(${data.x}px) scale(${data.scale})`,
        opacity: data.opacity,
      },
    ];
  }

  // Staggered animations without blocking
  async staggerAnimation(elements, delay = 50) {
    for (let i = 0; i < elements.length; i++) {
      const element = elements[i];

      // Animate current element
      this.animate(
        element,
        [
          { transform: "translateY(20px)", opacity: 0 },
          { transform: "translateY(0)", opacity: 1 },
        ],
        {
          duration: 300,
          delay: i * delay,
          fill: "forwards",
        },
      );

      // Yield every 10 elements
      if (i % 10 === 0 && i > 0) {
        await new Promise((resolve) => setTimeout(resolve, 0));
      }
    }
  }
}

// Usage
const manager = new SmoothAnimationManager();

// Animate single element
const box = document.querySelector(".box");
manager.animate(
  box,
  [
    { transform: "scale(1)" },
    { transform: "scale(1.2)" },
    { transform: "scale(1)" },
  ],
  {
    duration: 500,
    iterations: Infinity,
  },
);

// Stagger animation for list
const listItems = document.querySelectorAll(".list-item");
manager.staggerAnimation(listItems);
```

---

## 🏗️ Best Practices

### **1. Break Up Long Tasks**

```javascript
// ❌ Bad: Long task blocks UI
function processAll(items) {
  items.forEach((item) => heavyProcess(item)); // 500ms+
}

// ✅ Good: Break into chunks
async function processAllChunked(items) {
  const chunkSize = 10;
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    chunk.forEach((item) => heavyProcess(item));
    await new Promise((resolve) => setTimeout(resolve, 0));
  }
}

// ✅ Better: Use requestIdleCallback
function processAllIdle(items) {
  let index = 0;

  function processChunk(deadline) {
    while (deadline.timeRemaining() > 0 && index < items.length) {
      heavyProcess(items[index++]);
    }

    if (index < items.length) {
      requestIdleCallback(processChunk);
    }
  }

  requestIdleCallback(processChunk);
}
```

### **2. Avoid Layout Thrashing**

```javascript
// ❌ Bad: Read-write-read-write
function badUpdate() {
  const height1 = div1.offsetHeight; // Read (layout)
  div1.style.height = height1 + 10 + "px"; // Write

  const height2 = div2.offsetHeight; // Read (layout again!)
  div2.style.height = height2 + 10 + "px"; // Write
}

// ✅ Good: Batch reads, then batch writes
function goodUpdate() {
  // Batch reads
  const height1 = div1.offsetHeight;
  const height2 = div2.offsetHeight;

  // Batch writes
  requestAnimationFrame(() => {
    div1.style.height = height1 + 10 + "px";
    div2.style.height = height2 + 10 + "px";
  });
}
```

### **3. Use Web Workers for Heavy Computation**

```javascript
// ✅ Offload heavy work to worker
const worker = new Worker("heavy.worker.js");

worker.postMessage({ data: largeDataset });

worker.addEventListener("message", (e) => {
  const result = e.data;
  updateUI(result); // Quick UI update on main thread
});
```

### **4. Defer Non-Critical Work**

```javascript
// ✅ Use requestIdleCallback for non-critical work
requestIdleCallback(
  () => {
    // Analytics, logging, preloading
    sendAnalytics();
    preloadImages();
  },
  { timeout: 2000 },
);

// ✅ Use setTimeout for deferred work
setTimeout(() => {
  // Non-critical initialization
  initializeFeature();
}, 0);
```

### **5. Monitor Main Thread Performance**

```javascript
// Track Total Blocking Time (TBT)
class TBTMonitor {
  constructor() {
    this.tbt = 0;
    this.startMonitoring();
  }

  startMonitoring() {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > 50) {
          this.tbt += entry.duration - 50;
        }
      }
    });

    observer.observe({ entryTypes: ["longtask"] });
  }

  getTBT() {
    return this.tbt;
  }
}
```

---

## 🧪 Interview Questions

### **Q1: What is the main thread and why is it a bottleneck?**

**Answer:**

The main thread is the single JavaScript execution thread in browsers that handles:

- JavaScript execution
- HTML parsing
- Style calculation
- Layout (reflow)
- Paint
- Event handling
- Garbage collection

**Why it's a bottleneck:**

1. **Single-threaded**: Only one task executes at a time
2. **Blocking**: Long tasks freeze the entire UI
3. **Frame budget**: Only 16.67ms per frame for 60fps
4. **Synchronous**: Most operations block further execution

**Example:**

```javascript
// This blocks the main thread for 100ms
function blockingTask() {
  const start = Date.now();
  while (Date.now() - start < 100) {
    // Busy loop
  }
}

button.addEventListener("click", () => {
  blockingTask(); // UI freezes for 100ms
  // No scrolling, clicking, or animations during this time
});
```

**Solutions:**

- Break tasks into smaller chunks
- Use Web Workers for heavy computation
- Use requestIdleCallback for non-critical work
- Optimize layout/paint operations

### **Q2: What is a long task and how do you detect them?**

**Answer:**

A **long task** is any JavaScript execution that takes longer than 50ms on the main thread. Long tasks cause:

- Dropped frames (< 60fps)
- Unresponsive UI
- Poor user experience

**Detection methods:**

```javascript
// 1. PerformanceObserver API
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn("Long task:", {
      duration: entry.duration,
      startTime: entry.startTime,
      name: entry.name,
    });
  }
});

observer.observe({ entryTypes: ["longtask"] });

// 2. Manual timing
function measureTask(fn) {
  const start = performance.now();
  fn();
  const duration = performance.now() - start;

  if (duration > 50) {
    console.warn(`Long task: ${duration}ms`);
  }
}

// 3. Chrome DevTools Performance panel
// Look for red triangles and long yellow blocks
```

**How to fix:**

```javascript
// ❌ Long task
function processAllSync(items) {
  items.forEach((item) => process(item)); // 200ms
}

// ✅ Break into chunks
async function processAllAsync(items) {
  const chunkSize = 10;
  for (let i = 0; i < items.length; i += chunkSize) {
    items.slice(i, i + chunkSize).forEach((item) => process(item));
    await new Promise((resolve) => setTimeout(resolve, 0));
  }
}
```

### **Q3: Explain layout thrashing and how to avoid it.**

**Answer:**

**Layout thrashing** (forced synchronous layout) occurs when you repeatedly read layout properties and then modify the DOM, forcing the browser to recalculate layout multiple times in a single frame.

**What causes layout thrashing:**

```javascript
// ❌ Layout thrashing
function thrash() {
  for (let i = 0; i < 100; i++) {
    const height = element.offsetHeight; // Read (forces layout)
    element.style.height = height + 1 + "px"; // Write
    // Browser must recalculate layout 100 times!
  }
}
```

**Properties that trigger layout:**

- offsetTop, offsetLeft, offsetWidth, offsetHeight
- clientTop, clientLeft, clientWidth, clientHeight
- scrollTop, scrollLeft, scrollWidth, scrollHeight
- getComputedStyle()
- getBoundingClientRect()

**How to avoid:**

```javascript
// ✅ Batch reads, then batch writes
function noThrash() {
  // 1. Batch all reads (single layout)
  const measurements = [];
  for (let i = 0; i < 100; i++) {
    measurements.push(element.offsetHeight);
  }

  // 2. Batch all writes (no layout until next frame)
  requestAnimationFrame(() => {
    measurements.forEach((height, i) => {
      elements[i].style.height = height + 1 + "px";
    });
  });
}

// ✅ Use libraries like fastdom
fastdom.measure(() => {
  const height = element.offsetHeight;

  fastdom.mutate(() => {
    element.style.height = height + 1 + "px";
  });
});
```

**Performance impact:**

- Layout thrashing: 100 layouts = ~500ms
- Batched approach: 1 layout = ~5ms
- **100x faster!**

### **Q4: How do Web Workers help with main thread performance?**

**Answer:**

Web Workers run JavaScript on separate background threads, keeping the main thread free for UI updates.

**Benefits:**

1. **Non-blocking**: Heavy computation doesn't freeze UI
2. **Parallel execution**: Utilize multiple CPU cores
3. **Better responsiveness**: Main thread handles user input immediately

**Example:**

```javascript
// Main thread
const worker = new Worker("compute.worker.js");

// Heavy computation on worker thread
worker.postMessage({ data: largeArray });

// Main thread stays responsive
button.addEventListener("click", () => {
  console.log("Button works while worker computes!");
});

// Receive result
worker.addEventListener("message", (e) => {
  const result = e.data;
  updateUI(result); // Quick update on main thread
});

// Worker thread (compute.worker.js)
self.addEventListener("message", (e) => {
  const { data } = e.data;

  // Heavy computation (doesn't block main thread)
  const result = data.map((item) => {
    return complexCalculation(item); // 1000ms
  });

  // Send result back
  self.postMessage(result);
});
```

**Limitations:**

- No DOM access (can't manipulate elements)
- No access to window, document
- Communication via postMessage (serialization overhead)
- Best for: data processing, image manipulation, cryptography

**When to use:**

- ✅ Heavy calculations
- ✅ Data parsing/processing
- ✅ Image/video processing
- ❌ DOM manipulation
- ❌ Small, quick tasks (overhead not worth it)

### **Q5: What is the frame budget and how do you stay within it?**

**Answer:**

The **frame budget** is the time available to render one frame. For smooth 60fps, each frame must complete in **16.67ms**.

**Frame breakdown:**

```
16.67ms total budget:
  - JavaScript:      ~3-5ms
  - Style calc:      ~2-3ms
  - Layout:          ~3-5ms
  - Paint:           ~3-5ms
  - Composite:       ~1-2ms
  - Browser overhead: ~2ms
```

**If any phase exceeds budget → dropped frame!**

**Monitoring:**

```javascript
class FrameBudgetMonitor {
  constructor() {
    this.frameStart = performance.now();
    this.droppedFrames = 0;
    this.monitor();
  }

  monitor() {
    const frameTime = performance.now() - this.frameStart;

    if (frameTime > 16.67) {
      this.droppedFrames++;
      console.warn(`Dropped frame: ${frameTime.toFixed(2)}ms`);
    }

    this.frameStart = performance.now();
    requestAnimationFrame(() => this.monitor());
  }
}
```

**Staying within budget:**

```javascript
// ❌ Exceeds budget
function heavyFrame() {
  // 10ms JavaScript
  complexCalculation();

  // 8ms Layout
  element.offsetHeight;
  element.style.height = "100px";

  // 5ms Paint
  element.style.backgroundColor = "red";

  // Total: 23ms → DROPPED FRAME
}

// ✅ Within budget
function lightFrame() {
  // 2ms - compositor only
  element.style.transform = "translateX(100px)";
  element.style.opacity = 0.8;

  // Total: 2ms → SMOOTH
}

// ✅ Defer heavy work
function deferredWork() {
  // Quick update this frame
  element.style.transform = "translateX(100px)";

  // Heavy work next frame
  requestIdleCallback(() => {
    complexCalculation();
  });
}
```

**Tools:**

- Chrome DevTools Performance panel
- Performance Observer API
- requestAnimationFrame timing
- Long Task API

**Best practices:**

- Use transform/opacity for animations (compositor-only)
- Break long JavaScript tasks into chunks
- Batch DOM reads and writes
- Defer non-critical work with requestIdleCallback
- Monitor FPS and frame timing regularly

---

## 📚 Key Takeaways

1. **Main thread is single-threaded**: Only one task at a time, blocking operations freeze UI
2. **16.67ms frame budget**: All work must complete in 16.67ms for 60fps
3. **Long tasks > 50ms**: Cause jank, dropped frames, and poor UX
4. **Layout thrashing kills performance**: Batch reads then writes, never interleave
5. **Web Workers for heavy lifting**: Offload computation to background threads
6. **requestIdleCallback for non-critical work**: Use idle time without blocking frames
7. **Break up long tasks**: Use setTimeout/requestAnimationFrame to yield to browser
8. **Monitor performance**: Use PerformanceObserver and Chrome DevTools
9. **Optimize CSS**: Simple selectors, containment, avoid forced layout
10. **Prioritize user input**: Quick response > background computation

---

**Master main thread optimization for smooth, responsive applications!**
