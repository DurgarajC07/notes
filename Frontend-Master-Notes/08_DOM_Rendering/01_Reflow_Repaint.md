# Reflow & Repaint

> **Understanding Layout and Paint Phases for Performance**

---

## 🎯 Overview

**Reflow (Layout)** and **Repaint (Paint)** are two critical phases in the browser rendering pipeline. Understanding their performance implications is essential for building smooth, 60fps web applications.

**Key Differences:**

- **Reflow**: Recalculates element positions and geometries (expensive)
- **Repaint**: Redraws pixels without changing layout (less expensive)
- **Composite**: Combines layers on GPU (cheapest)

---

## 📚 Core Concepts

### **1. Rendering Pipeline Overview**

```
JavaScript → Style → Layout (Reflow) → Paint → Composite
    ↓          ↓          ↓              ↓         ↓
  DOM mods   CSSOM    Geometry       Pixels     Layers
```

**Pipeline breakdown:**

```javascript
class RenderingPipeline {
  render() {
    // 1. JavaScript execution
    this.runJavaScript(); // ~3-5ms

    // 2. Style calculation
    this.recalculateStyle(); // ~2-3ms

    // 3. Layout (Reflow)
    this.calculateLayout(); // ~5-15ms ⚠️ EXPENSIVE

    // 4. Paint (Repaint)
    this.paint(); // ~5-10ms ⚠️ EXPENSIVE

    // 5. Composite
    this.composite(); // ~1-2ms ✅ CHEAP

    // Total: ~16-35ms
    // Budget: 16.67ms for 60fps
  }
}
```

### **2. What Triggers Reflow**

Reflow recalculates element positions and dimensions:

```javascript
// Properties that trigger REFLOW
const REFLOW_PROPERTIES = {
  // Box model
  width: true,
  height: true,
  padding: true,
  paddingTop: true,
  paddingRight: true,
  paddingBottom: true,
  paddingLeft: true,
  margin: true,
  marginTop: true,
  marginRight: true,
  marginBottom: true,
  marginLeft: true,
  border: true,
  borderWidth: true,

  // Position
  top: true,
  right: true,
  bottom: true,
  left: true,
  position: true,

  // Display
  display: true,
  float: true,
  clear: true,
  verticalAlign: true,

  // Text
  fontSize: true,
  fontFamily: true,
  fontWeight: true,
  lineHeight: true,
  textAlign: true,
  whiteSpace: true,
  wordWrap: true,

  // Other
  overflow: true,
  overflowX: true,
  overflowY: true,
  minWidth: true,
  maxWidth: true,
  minHeight: true,
  maxHeight: true,
};

// Example: Triggering reflow
element.style.width = "100px"; // Triggers reflow
element.style.height = "200px"; // Triggers another reflow
```

**Reading layout properties forces reflow:**

```javascript
// These properties FORCE synchronous reflow
const FORCE_REFLOW_PROPERTIES = [
  "offsetTop",
  "offsetLeft",
  "offsetWidth",
  "offsetHeight",
  "offsetParent",

  "clientTop",
  "clientLeft",
  "clientWidth",
  "clientHeight",

  "scrollTop",
  "scrollLeft",
  "scrollWidth",
  "scrollHeight",

  "getComputedStyle()",
  "getBoundingClientRect()",
  "getClientRects()",

  "scrollIntoView()",
  "scrollTo()",
  "focus()",
];

// ❌ Forces reflow
const height = element.offsetHeight; // Read
element.style.height = height + 10 + "px"; // Write
// Browser must stop and recalculate layout!
```

### **3. What Triggers Repaint**

Repaint redraws pixels without changing layout:

```javascript
// Properties that trigger REPAINT (but not reflow)
const REPAINT_PROPERTIES = {
  // Colors
  color: true,
  backgroundColor: true,
  borderColor: true,

  // Visual effects
  visibility: true,
  textDecoration: true,
  backgroundImage: true,
  backgroundPosition: true,
  backgroundRepeat: true,
  backgroundSize: true,

  // Outline
  outline: true,
  outlineColor: true,
  outlineStyle: true,
  outlineWidth: true,

  // Box shadow
  boxShadow: true,

  // Border radius
  borderRadius: true,
};

// Example: Triggering repaint (no reflow)
element.style.backgroundColor = "red"; // Repaint only
element.style.color = "blue"; // Repaint only
```

### **4. Composite-Only Changes**

Some properties only trigger composite (GPU-accelerated):

```javascript
// Properties that ONLY trigger COMPOSITE (fastest!)
const COMPOSITE_ONLY_PROPERTIES = {
  transform: true, // ✅ Best for animations
  opacity: true, // ✅ Best for fade effects
  filter: true, // ✅ (modern browsers)
  willChange: true, // ✅ Hint for layer creation
};

// Example: Compositor-only changes
element.style.transform = "translateX(100px)"; // Composite only!
element.style.opacity = "0.5"; // Composite only!

// No reflow, no repaint, just GPU composition
// ~1-2ms instead of ~20-30ms!
```

### **5. Layout Thrashing (Forced Synchronous Layout)**

Alternating between reading and writing layout properties:

```javascript
class LayoutThrashing {
  // ❌ BAD: Layout thrashing
  badThrashing() {
    const elements = document.querySelectorAll(".box");

    elements.forEach((element) => {
      // Read (forces reflow)
      const height = element.offsetHeight;

      // Write
      element.style.height = height + 10 + "px";

      // Read again (forces ANOTHER reflow)
      const width = element.offsetWidth;

      // Write again
      element.style.width = width + 10 + "px";
    });

    // Total: 2N reflows for N elements!
    // For 100 elements: 200 reflows = ~1000ms!
  }

  // ✅ GOOD: Batch reads, then batch writes
  goodBatching() {
    const elements = Array.from(document.querySelectorAll(".box"));

    // 1. Batch all reads (single reflow)
    const measurements = elements.map((el) => ({
      height: el.offsetHeight,
      width: el.offsetWidth,
    }));

    // 2. Batch all writes (single reflow at end)
    requestAnimationFrame(() => {
      elements.forEach((el, i) => {
        el.style.height = measurements[i].height + 10 + "px";
        el.style.width = measurements[i].width + 10 + "px";
      });
    });

    // Total: 2 reflows for N elements!
    // For 100 elements: 2 reflows = ~10ms!
    // 100x faster!
  }
}
```

### **6. Reflow Scope**

Reflow can be local or global:

```javascript
class ReflowScope {
  // Local reflow (affects subtree only)
  localReflow() {
    const container = document.querySelector(".isolated-container");

    // These CSS properties limit reflow scope
    container.style.contain = "layout"; // CSS containment
    container.style.position = "absolute"; // Taken out of flow

    // Changes inside container don't affect parent/siblings
    const child = container.querySelector(".child");
    child.style.width = "200px"; // Local reflow only
  }

  // Global reflow (affects entire document)
  globalReflow() {
    // These trigger global reflow
    document.body.style.fontSize = "18px"; // Affects all text
    document.body.classList.add("theme"); // Could affect everything

    // Resizing viewport
    window.resizeTo(800, 600); // Global reflow

    // Changing root font size
    document.documentElement.style.fontSize = "20px"; // Global reflow
  }

  measureReflowScope() {
    const start = performance.now();

    // Measure local reflow
    element.style.width = "100px";

    const localDuration = performance.now() - start;

    // Measure global reflow
    const start2 = performance.now();
    document.body.style.fontSize = "16px";

    const globalDuration = performance.now() - start2;

    console.log({
      local: localDuration, // ~2ms
      global: globalDuration, // ~20ms
    });
  }
}
```

### **7. Paint Optimization**

Understanding paint complexity:

```javascript
class PaintOptimization {
  // Paint cost depends on:
  // 1. Area painted
  // 2. Complexity (shadows, gradients, blur)
  // 3. Number of layers

  measurePaintCost() {
    const costs = {
      simple: this.paintSimple(), // ~1ms
      shadow: this.paintShadow(), // ~3ms
      gradient: this.paintGradient(), // ~5ms
      blur: this.paintBlur(), // ~10ms
    };

    return costs;
  }

  paintSimple() {
    // Simple solid color
    element.style.backgroundColor = "red";
    return this.measureTime();
  }

  paintShadow() {
    // Box shadow requires extra paint work
    element.style.boxShadow = "0 4px 6px rgba(0,0,0,0.1)";
    return this.measureTime();
  }

  paintGradient() {
    // Gradients require interpolation
    element.style.background = "linear-gradient(to right, red, blue)";
    return this.measureTime();
  }

  paintBlur() {
    // Blur is most expensive (multiple passes)
    element.style.filter = "blur(5px)";
    return this.measureTime();
  }

  // Reduce paint area
  reducePaintArea() {
    // ✅ Use CSS containment
    element.style.contain = "paint";

    // ✅ Use will-change for animations
    element.style.willChange = "transform, opacity";

    // ✅ Promote to layer
    element.style.transform = "translateZ(0)";
  }
}
```

### **8. DevTools Performance Analysis**

Using Chrome DevTools to analyze reflow/repaint:

```javascript
class PerformanceAnalysis {
  // Enable paint flashing
  enablePaintFlashing() {
    // Chrome DevTools > Rendering > Paint flashing
    // Green rectangles show repainted areas
  }

  // Enable layout shift regions
  enableLayoutShiftRegions() {
    // Chrome DevTools > Rendering > Layout Shift Regions
    // Blue rectangles show layout shifts
  }

  // Record performance
  recordPerformance() {
    // 1. Open DevTools Performance tab
    // 2. Click record
    // 3. Perform action
    // 4. Stop recording
    // Look for:
    // - Purple bars: Rendering (layout + paint)
    // - Green bars: Painting
    // - Long tasks: > 50ms
  }

  // Use Performance API
  measureWithPerformanceAPI() {
    // Mark start
    performance.mark("reflow-start");

    // Trigger reflow
    element.style.width = "100px";
    const height = element.offsetHeight; // Force reflow

    // Mark end
    performance.mark("reflow-end");

    // Measure
    performance.measure("reflow", "reflow-start", "reflow-end");

    // Get results
    const measures = performance.getEntriesByName("reflow");
    console.log("Reflow took:", measures[0].duration, "ms");
  }

  // Monitor paint timing
  monitorPaintTiming() {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        console.log(entry.name + ":", entry.startTime);
        // first-paint: 150ms
        // first-contentful-paint: 200ms
      }
    });

    observer.observe({ entryTypes: ["paint"] });
  }
}
```

---

## 🔥 Practical Examples

### **Example 1: Optimized DOM Manipulation**

```javascript
class OptimizedDOMManipulation {
  // ❌ BAD: Multiple reflows
  badApproach() {
    const container = document.querySelector(".container");

    // Each appendChild triggers reflow
    for (let i = 0; i < 100; i++) {
      const div = document.createElement("div");
      div.textContent = `Item ${i}`;
      container.appendChild(div); // Reflow! (100 times)
    }
    // Total: ~500ms
  }

  // ✅ GOOD: Single reflow with DocumentFragment
  goodApproach() {
    const container = document.querySelector(".container");
    const fragment = document.createDocumentFragment();

    // Build in memory (no reflow)
    for (let i = 0; i < 100; i++) {
      const div = document.createElement("div");
      div.textContent = `Item ${i}`;
      fragment.appendChild(div); // No reflow
    }

    // Single append (one reflow)
    container.appendChild(fragment); // Reflow (1 time)
    // Total: ~5ms
  }

  // ✅ BETTER: Use innerHTML (browser optimized)
  betterApproach() {
    const container = document.querySelector(".container");

    // Build HTML string
    let html = "";
    for (let i = 0; i < 100; i++) {
      html += `<div>Item ${i}</div>`;
    }

    // Single DOM update
    container.innerHTML = html; // Single reflow
    // Total: ~3ms
  }

  // ✅ BEST: Virtual DOM pattern
  bestApproach() {
    const container = document.querySelector(".container");

    // Hide element (no reflow during updates)
    container.style.display = "none";

    // Make multiple changes
    for (let i = 0; i < 100; i++) {
      const div = document.createElement("div");
      div.textContent = `Item ${i}`;
      container.appendChild(div);
    }

    // Show element (single reflow)
    container.style.display = "block";
    // Total: ~5ms + avoids intermediate reflows
  }
}
```

### **Example 2: Animation Performance**

```javascript
class AnimationPerformance {
  // ❌ BAD: Animating left (causes reflow every frame)
  badAnimation() {
    let left = 0;

    function animate() {
      element.style.left = left + "px"; // Reflow every frame!
      left += 1;

      if (left < 100) {
        requestAnimationFrame(animate);
      }
    }

    animate();
    // ~16ms per frame (barely 60fps, may drop frames)
  }

  // ✅ GOOD: Animating transform (compositor only)
  goodAnimation() {
    let x = 0;

    function animate() {
      element.style.transform = `translateX(${x}px)`; // Composite only!
      x += 1;

      if (x < 100) {
        requestAnimationFrame(animate);
      }
    }

    animate();
    // ~1-2ms per frame (smooth 60fps)
  }

  // ✅ BETTER: CSS animations (browser optimized)
  betterAnimation() {
    // Define animation in CSS
    const style = document.createElement("style");
    style.textContent = `
      @keyframes slide {
        from { transform: translateX(0); }
        to { transform: translateX(100px); }
      }
      
      .animated {
        animation: slide 1s ease-out forwards;
      }
    `;
    document.head.appendChild(style);

    // Trigger animation
    element.classList.add("animated");
    // Browser handles optimization automatically
  }

  // ✅ BEST: Web Animations API
  bestAnimation() {
    element.animate(
      [{ transform: "translateX(0)" }, { transform: "translateX(100px)" }],
      {
        duration: 1000,
        easing: "ease-out",
        fill: "forwards",
      },
    );
    // Runs on compositor thread, no main thread blocking
  }
}
```

### **Example 3: Scroll Performance**

```javascript
class ScrollPerformance {
  constructor() {
    this.ticking = false;
    this.lastScrollY = 0;
  }

  // ❌ BAD: Direct scroll handler (causes jank)
  badScrollHandler() {
    window.addEventListener("scroll", () => {
      const scrollY = window.scrollY;

      // Reading and writing in scroll handler
      elements.forEach((el) => {
        const rect = el.getBoundingClientRect(); // Forces reflow!
        if (rect.top < window.innerHeight) {
          el.classList.add("visible"); // Causes repaint
        }
      });
      // Runs on every scroll event = janky!
    });
  }

  // ✅ GOOD: Debounced scroll handler
  goodScrollHandler() {
    window.addEventListener(
      "scroll",
      () => {
        this.lastScrollY = window.scrollY;

        if (!this.ticking) {
          requestAnimationFrame(() => {
            this.updateElements(this.lastScrollY);
            this.ticking = false;
          });
          this.ticking = true;
        }
      },
      { passive: true },
    );
  }

  updateElements(scrollY) {
    // Batch reads
    const measurements = Array.from(elements).map((el) => ({
      el: el,
      rect: el.getBoundingClientRect(),
    }));

    // Batch writes
    measurements.forEach(({ el, rect }) => {
      if (rect.top < window.innerHeight) {
        el.classList.add("visible");
      }
    });
  }

  // ✅ BETTER: Intersection Observer (browser optimized)
  betterScrollHandler() {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            entry.target.classList.add("visible");
          }
        });
      },
      {
        threshold: 0.1,
        rootMargin: "50px",
      },
    );

    elements.forEach((el) => observer.observe(el));
    // Browser handles optimization, no layout thrashing!
  }
}
```

### **Example 4: Table Rendering Optimization**

```javascript
class TableOptimization {
  constructor(data) {
    this.data = data;
    this.container = document.querySelector("#table-container");
  }

  // ❌ BAD: Row-by-row rendering
  badRendering() {
    const table = document.createElement("table");

    this.data.forEach((row) => {
      const tr = document.createElement("tr");
      Object.values(row).forEach((cell) => {
        const td = document.createElement("td");
        td.textContent = cell;
        tr.appendChild(td); // Multiple reflows
      });
      table.appendChild(tr); // Multiple reflows
    });

    this.container.appendChild(table);
    // For 1000 rows: ~2000ms
  }

  // ✅ GOOD: Batch rendering with fragment
  goodRendering() {
    const table = document.createElement("table");
    const fragment = document.createDocumentFragment();

    this.data.forEach((row) => {
      const tr = document.createElement("tr");
      Object.values(row).forEach((cell) => {
        const td = document.createElement("td");
        td.textContent = cell;
        tr.appendChild(td);
      });
      fragment.appendChild(tr); // No reflow
    });

    table.appendChild(fragment);
    this.container.appendChild(table); // Single reflow
    // For 1000 rows: ~50ms
  }

  // ✅ BETTER: innerHTML (browser optimized parsing)
  betterRendering() {
    let html = "<table>";

    this.data.forEach((row) => {
      html += "<tr>";
      Object.values(row).forEach((cell) => {
        html += `<td>${cell}</td>`;
      });
      html += "</tr>";
    });

    html += "</table>";
    this.container.innerHTML = html; // Single reflow
    // For 1000 rows: ~30ms
  }

  // ✅ BEST: Virtual scrolling (only render visible)
  bestRendering() {
    const rowHeight = 40;
    const visibleRows = Math.ceil(this.container.clientHeight / rowHeight);

    let scrollTop = 0;

    const renderVisible = () => {
      const startIndex = Math.floor(scrollTop / rowHeight);
      const endIndex = startIndex + visibleRows + 5; // Buffer

      const visibleData = this.data.slice(startIndex, endIndex);

      // Render only visible rows
      let html =
        '<table><tbody style="transform: translateY(' +
        startIndex * rowHeight +
        'px)">';

      visibleData.forEach((row) => {
        html += "<tr>";
        Object.values(row).forEach((cell) => {
          html += `<td>${cell}</td>`;
        });
        html += "</tr>";
      });

      html += "</tbody></table>";
      this.container.innerHTML = html;
    };

    this.container.addEventListener(
      "scroll",
      () => {
        scrollTop = this.container.scrollTop;
        requestAnimationFrame(renderVisible);
      },
      { passive: true },
    );

    renderVisible();
    // For 1000 rows: ~5ms (only renders ~20 visible rows)
  }
}
```

---

## 🏗️ Best Practices

### **1. Avoid Layout Thrashing**

```javascript
// ❌ BAD: Read-write-read-write
function bad() {
  div1.style.height = div1.offsetHeight + 10 + "px";
  div2.style.height = div2.offsetHeight + 10 + "px";
}

// ✅ GOOD: Batch reads, then batch writes
function good() {
  const h1 = div1.offsetHeight;
  const h2 = div2.offsetHeight;

  requestAnimationFrame(() => {
    div1.style.height = h1 + 10 + "px";
    div2.style.height = h2 + 10 + "px";
  });
}
```

### **2. Use Transform Instead of Position**

```css
/* ❌ BAD: Causes reflow */
.animated {
  animation: slide 1s;
}

@keyframes slide {
  from {
    left: 0;
  }
  to {
    left: 100px;
  }
}

/* ✅ GOOD: Compositor only */
.animated {
  animation: slide 1s;
}

@keyframes slide {
  from {
    transform: translateX(0);
  }
  to {
    transform: translateX(100px);
  }
}
```

### **3. Use CSS Containment**

```css
/* Limit reflow scope */
.widget {
  contain: layout style;
}

.isolated-component {
  contain: strict; /* layout + style + paint */
}
```

### **4. Batch DOM Updates**

```javascript
// ✅ Use DocumentFragment
const fragment = document.createDocumentFragment();
items.forEach((item) => {
  fragment.appendChild(createNode(item));
});
container.appendChild(fragment); // Single reflow

// ✅ Hide element during updates
element.style.display = "none";
// ... multiple updates ...
element.style.display = "block"; // Single reflow
```

### **5. Use requestAnimationFrame for Visual Updates**

```javascript
// ✅ Sync with browser's repaint cycle
requestAnimationFrame(() => {
  element.style.transform = "translateX(100px)";
});

// ❌ Don't use setTimeout for animations
setTimeout(() => {
  element.style.left = "100px";
}, 16); // Not synced with browser
```

---

## 🧪 Interview Questions

### **Q1: What's the difference between reflow and repaint?**

**Answer:**

**Reflow (Layout):**

- Recalculates element positions and sizes
- Triggered by geometry changes (width, height, position)
- **Expensive**: ~5-20ms, affects layout tree
- Can cascade to parent/sibling elements

**Repaint (Paint):**

- Redraws pixels without changing layout
- Triggered by visual changes (color, background, shadow)
- **Less expensive**: ~3-10ms, only repaints affected area
- Doesn't affect layout

**Composite:**

- Combines layers on GPU
- Triggered by transform, opacity
- **Cheapest**: ~1-2ms, GPU accelerated

**Performance comparison:**

```javascript
// Reflow: ~20ms
element.style.width = "100px";

// Repaint: ~5ms
element.style.backgroundColor = "red";

// Composite: ~1ms
element.style.transform = "translateX(100px)";
```

**Key rule**: Reflow always includes repaint, but repaint doesn't cause reflow.

### **Q2: What is layout thrashing and how do you prevent it?**

**Answer:**

**Layout thrashing** (forced synchronous layout) occurs when you repeatedly read layout properties then write to them, forcing the browser to recalculate layout multiple times per frame.

**Problem:**

```javascript
// ❌ Layout thrashing
elements.forEach((el) => {
  const height = el.offsetHeight; // Read (forces layout)
  el.style.height = height + 10 + "px"; // Write
  // Browser recalculates layout N times!
});
```

**Solution:**

```javascript
// ✅ Batch reads, then batch writes
// 1. Read phase
const heights = elements.map((el) => el.offsetHeight);

// 2. Write phase
requestAnimationFrame(() => {
  elements.forEach((el, i) => {
    el.style.height = heights[i] + 10 + "px";
  });
});
// Browser recalculates layout only once!
```

**Using FastDOM library:**

```javascript
fastdom.measure(() => {
  const height = element.offsetHeight;

  fastdom.mutate(() => {
    element.style.height = height + 10 + "px";
  });
});
```

### **Q3: Which CSS properties trigger reflow vs repaint vs composite?**

**Answer:**

**Reflow (Layout) - Most Expensive:**

```javascript
{
  (width,
    height,
    margin,
    padding,
    border,
    top,
    left,
    right,
    bottom,
    position,
    display,
    float,
    fontSize,
    lineHeight,
    overflow,
    minHeight,
    maxHeight);
}
```

**Repaint (Paint) - Medium Cost:**

```javascript
{
  (color,
    backgroundColor,
    visibility,
    outline,
    boxShadow,
    borderRadius,
    backgroundImage,
    backgroundPosition);
}
```

**Composite - Cheapest:**

```javascript
{
  (transform, // ✅ Use for animations
    opacity, // ✅ Use for fade effects
    filter); // ✅ (modern browsers)
}
```

**Performance comparison:**

```javascript
// Reflow + Repaint + Composite: ~25ms
element.style.width = "100px";

// Repaint + Composite: ~8ms
element.style.backgroundColor = "red";

// Composite only: ~2ms
element.style.transform = "scale(1.2)";
```

**Best practice:**

```css
/* ❌ Avoid in animations */
@keyframes bad {
  from {
    width: 0;
  } /* Reflow! */
  to {
    width: 100px;
  }
}

/* ✅ Use compositor properties */
@keyframes good {
  from {
    transform: scaleX(0);
  } /* Composite! */
  to {
    transform: scaleX(1);
  }
}
```

### **Q4: How do you optimize DOM manipulation to minimize reflows?**

**Answer:**

**1. Use DocumentFragment:**

```javascript
const fragment = document.createDocumentFragment();
items.forEach((item) => {
  fragment.appendChild(createNode(item));
});
container.appendChild(fragment); // Single reflow
```

**2. Batch DOM updates:**

```javascript
// Hide → Update → Show
element.style.display = "none";
// ... many updates ...
element.style.display = "block"; // Single reflow
```

**3. Clone and replace:**

```javascript
const clone = element.cloneNode(true);
// Modify clone (off-DOM, no reflow)
modifyClone(clone);
element.parentNode.replaceChild(clone, element); // Single reflow
```

**4. Use CSS classes instead of inline styles:**

```javascript
// ❌ Multiple reflows
element.style.width = "100px";
element.style.height = "100px";
element.style.backgroundColor = "red";

// ✅ Single reflow
element.className = "styled"; // All styles applied at once
```

**5. Use position: absolute for animated elements:**

```css
/* Takes element out of flow, reduces reflow scope */
.animated {
  position: absolute;
}
```

**6. Use CSS containment:**

```css
.widget {
  contain: layout style paint;
}
```

### **Q5: Explain how the browser optimizes reflows and repaints.**

**Answer:**

**Browser optimizations:**

**1. Coalescing:**

```javascript
// Browser batches these into one reflow
element.style.width = "100px";
element.style.height = "200px";
element.style.padding = "10px";
// Reflow happens at end of script, not after each line
```

**2. Dirty bit system:**

```javascript
// Browser marks elements as "dirty"
element.style.width = "100px"; // Mark dirty
element.style.height = "200px"; // Still dirty

// Reflow only when needed (reading layout property)
const height = element.offsetHeight; // Trigger reflow now
```

**3. Incremental reflow:**

```javascript
// Browser only reflows affected subtree
child.style.width = "100px";
// Only child subtree reflowed, not entire document
```

**4. Layout containment:**

```css
/* Browser knows to limit reflow scope */
.container {
  contain: layout;
}
```

**5. Layer promotion:**

```css
/* Browser promotes to GPU layer */
.animated {
  will-change: transform;
  transform: translateZ(0);
}
```

**6. Compositor thread:**

```javascript
// Transform/opacity run on separate thread
element.style.transform = "translateX(100px)";
// Main thread stays free!
```

**Performance tools:**

```javascript
// Chrome DevTools > Performance
// - Look for purple "Rendering" blocks
// - Check "Paint flashing" in Rendering tab
// - Use Layers panel to see compositor layers
```

---

## 📚 Key Takeaways

1. **Reflow is expensive**: Recalculates geometry, can affect entire document (~5-20ms)
2. **Repaint is cheaper**: Only redraws pixels, no geometry changes (~3-10ms)
3. **Composite is cheapest**: GPU-accelerated, no CPU work (~1-2ms)
4. **Layout thrashing**: Alternating read/write of layout properties causes multiple reflows
5. **Batch DOM updates**: Use DocumentFragment or hide elements during updates
6. **Use transform/opacity**: Compositor-only properties for smooth 60fps animations
7. **CSS containment**: Limit reflow scope with `contain: layout`
8. **requestAnimationFrame**: Sync visual updates with browser's render cycle
9. **Read then write**: Batch all DOM reads before any writes
10. **DevTools Performance**: Use Paint flashing and Performance profiler to identify issues

---

**Master reflow and repaint optimization for blazing-fast web applications!**
