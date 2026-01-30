# CSS Containment

## Core Concept

CSS Containment (`contain` property) is a performance optimization that allows the browser to isolate a subtree of the page from the rest of the document. By indicating that an element's contents don't affect other parts of the page, the browser can optimize rendering and layout calculations.

---

## The contain Property

### Syntax

```css
.element {
  contain: none; /* default */
  contain: layout;
  contain: paint;
  contain: size;
  contain: style;
  contain: content; /* layout + paint */
  contain: strict; /* layout + paint + size */
  contain: layout paint;
}
```

### Layout Containment

```css
/* Isolate layout calculations */
.card {
  contain: layout;
}

/* Benefits:
 * - Element creates new formatting context
 * - Internal layout changes don't affect external elements
 * - Floats and margins contained within
 * - Position: absolute children positioned relative to this
 */

.sidebar {
  contain: layout;
  width: 250px;
}

.sidebar-content {
  /* Changes here don't trigger layout outside .sidebar */
  padding: 1rem;
}
```

### Paint Containment

```css
/* Isolate painting */
.modal-content {
  contain: paint;
  overflow: hidden; /* Required for paint containment */
}

/* Benefits:
 * - Content can't paint outside element's bounds
 * - Browser can skip repainting unaffected areas
 * - Improves scrolling performance
 */

.chat-message {
  contain: paint;
  max-width: 600px;
  overflow: hidden;
}
```

### Size Containment

```css
/* Size determined independently of children */
.card {
  contain: size;
  width: 300px;
  height: 400px; /* Must specify both dimensions */
}

/* Benefits:
 * - Element size doesn't depend on children
 * - Children can be measured without affecting parent
 * - Useful for virtual scrolling and lazy loading
 */

/* ⚠️ Warning: Children can overflow if size is too small */
.fixed-size-container {
  contain: size;
  width: 200px;
  height: 200px;
  overflow: auto; /* Handle overflow */
}
```

### Style Containment

```css
/* Isolate CSS counters and quotes */
.article {
  contain: style;
  counter-reset: section;
}

/* Benefits:
 * - CSS counters scoped to element
 * - Quotes don't affect external content
 * - Minimal performance impact
 */
```

---

## Combined Containment

### content Containment

```css
/* Equivalent to: layout + paint */
.widget {
  contain: content;
}

/* Best for:
 * - Independent widgets
 * - Cards and list items
 * - Components that don't need external size
 */

.news-card {
  contain: content;
  padding: 1rem;
  border-radius: 8px;
  background: white;
}
```

### strict Containment

```css
/* Equivalent to: layout + paint + size */
.thumbnail {
  contain: strict;
  width: 200px;
  height: 200px;
}

/* Best for:
 * - Fixed-size elements
 * - Virtual scrolling items
 * - Placeholders and skeletons
 */

.product-thumbnail {
  contain: strict;
  width: 250px;
  height: 350px;
  overflow: hidden;
}
```

---

## Real-World Pattern: Infinite Scroll

```html
<div class="infinite-list">
  <div class="list-item">Item 1</div>
  <div class="list-item">Item 2</div>
  <!-- ... thousands of items ... -->
</div>
```

```css
.infinite-list {
  height: 600px;
  overflow-y: auto;
}

.list-item {
  /* Each item is independent */
  contain: content;
  padding: 1rem;
  border-bottom: 1px solid #eee;
}

/* Browser can:
 * - Skip layout calculations for off-screen items
 * - Only repaint visible items on scroll
 * - Improve scroll performance significantly
 */
```

### With Virtual Scrolling

```css
.virtual-scroller {
  position: relative;
  height: 600px;
  overflow-y: auto;
}

.virtual-item {
  /* Fixed size for virtual scrolling */
  contain: strict;
  position: absolute;
  width: 100%;
  height: 80px;
  left: 0;
}

/* JavaScript calculates which items are visible
 * and sets top position dynamically
 */
```

---

## Real-World Pattern: Card Grid

```html
<div class="card-grid">
  <article class="card">
    <img src="image.jpg" alt="Card image" />
    <h2>Card Title</h2>
    <p>Description...</p>
  </article>
  <!-- More cards -->
</div>
```

```css
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.5rem;
}

.card {
  /* Isolate each card */
  contain: content;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  overflow: hidden;
  transition: transform 0.2s;
}

.card:hover {
  transform: translateY(-4px);
  /* Transform doesn't trigger layout on other cards */
}

/* Benefits:
 * - Hovering one card doesn't affect others
 * - Adding/removing cards is optimized
 * - Animations are isolated
 */
```

---

## Real-World Pattern: Modal Dialog

```html
<div class="modal-overlay">
  <div class="modal-container">
    <div class="modal-content">
      <!-- Large content that might scroll -->
    </div>
  </div>
</div>
```

```css
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-container {
  /* Isolate modal from page */
  contain: layout paint;
  max-width: 90vw;
  max-height: 90vh;
  background: white;
  border-radius: 8px;
  box-shadow: 0 20px 25px rgba(0, 0, 0, 0.15);
}

.modal-content {
  overflow-y: auto;
  padding: 2rem;
  max-height: calc(90vh - 4rem);
}

/* Benefits:
 * - Modal content doesn't affect page layout
 * - Scrolling modal doesn't repaint page
 * - Better animation performance
 */
```

---

## Real-World Pattern: Sidebar Navigation

```html
<div class="layout">
  <aside class="sidebar">
    <nav>
      <!-- Navigation items -->
    </nav>
  </aside>
  <main class="main-content">
    <!-- Page content -->
  </main>
</div>
```

```css
.layout {
  display: flex;
  min-height: 100vh;
}

.sidebar {
  /* Isolate sidebar from main content */
  contain: layout paint;
  width: 250px;
  background: #f5f5f5;
  overflow-y: auto;
  border-right: 1px solid #e0e0e0;
}

.main-content {
  flex: 1;
  padding: 2rem;
  overflow-y: auto;
}

/* Benefits:
 * - Sidebar scrolling doesn't affect main content
 * - Expanding/collapsing sidebar items is optimized
 * - Independent repaint boundaries
 */
```

---

## Real-World Pattern: Chat Application

```html
<div class="chat-container">
  <div class="chat-messages">
    <div class="message">Message 1</div>
    <div class="message">Message 2</div>
    <!-- Hundreds of messages -->
  </div>
</div>
```

```css
.chat-container {
  height: 600px;
  overflow-y: auto;
  display: flex;
  flex-direction: column-reverse; /* New messages at bottom */
}

.message {
  /* Each message is independent */
  contain: content;
  padding: 0.75rem 1rem;
  margin: 0.25rem 0;
  border-radius: 8px;
  max-width: 70%;
}

.message-sent {
  align-self: flex-end;
  background: #3b82f6;
  color: white;
}

.message-received {
  align-self: flex-start;
  background: #f3f4f6;
  color: #111827;
}

/* Benefits:
 * - Adding new messages doesn't recalculate all previous messages
 * - Smooth scrolling with hundreds of messages
 * - Better memory usage
 */
```

---

## Real-World Pattern: Image Gallery

```html
<div class="gallery">
  <div class="gallery-item">
    <img src="photo1.jpg" alt="Photo 1" />
  </div>
  <!-- More photos -->
</div>
```

```css
.gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1rem;
}

.gallery-item {
  /* Fixed size for predictable layout */
  contain: strict;
  width: 100%;
  height: 250px;
  overflow: hidden;
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.2s;
}

.gallery-item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.gallery-item:hover {
  transform: scale(1.05);
}

/* Benefits:
 * - Predictable layout calculation
 * - Hover effects don't shift other images
 * - Fast rendering with many images
 */
```

---

## Container Queries (Related Feature)

### @container Rule

```css
/* Define container */
.card {
  contain: layout inline-size; /* Required for container queries */
  container-type: inline-size;
  container-name: card;
}

/* Query container width */
@container card (min-width: 400px) {
  .card-title {
    font-size: 1.5rem;
  }

  .card-image {
    float: left;
    width: 40%;
    margin-right: 1rem;
  }
}

@container card (max-width: 399px) {
  .card-title {
    font-size: 1.125rem;
  }

  .card-image {
    width: 100%;
    margin-bottom: 1rem;
  }
}
```

### Container Query Units

```css
.sidebar {
  container-type: inline-size;
}

.widget {
  /* Relative to container width */
  padding: 5cqi; /* 5% of container's inline size */
  font-size: 3cqi; /* 3% of container's inline size */
}

/* Container Query Units:
 * cqw - 1% of container width
 * cqh - 1% of container height
 * cqi - 1% of container inline size
 * cqb - 1% of container block size
 * cqmin - smaller of cqi or cqb
 * cqmax - larger of cqi or cqb
 */
```

---

## Performance Benchmarks

### Without Containment

```css
/* ❌ No containment */
.list-item {
  padding: 1rem;
}

/* Browser must:
 * - Recalculate layout for entire page on change
 * - Repaint all affected areas
 * - Check if other elements are affected
 */
```

### With Containment

```css
/* ✅ With containment */
.list-item {
  contain: content;
  padding: 1rem;
}

/* Browser can:
 * - Skip layout calculations outside element
 * - Repaint only this element
 * - Parallelize rendering of items
 * 
 * Result: 3-5x faster rendering in large lists
 */
```

---

## Browser Compatibility

### Feature Detection

```css
@supports (contain: paint) {
  .card {
    contain: content;
  }
}
```

```javascript
// JavaScript detection
if ("contain" in document.documentElement.style) {
  // Browser supports containment
  element.style.contain = "content";
}
```

---

## Best Practices

### 1. Use on Repeated Elements

```css
/* ✅ Good - Cards in grid */
.product-card {
  contain: content;
}

/* ✅ Good - List items */
.todo-item {
  contain: content;
}

/* ❌ Avoid - Unique page elements */
.page-header {
  /* Usually doesn't need containment */
}
```

### 2. Combine with overflow

```css
.card {
  contain: paint;
  overflow: hidden; /* Required for paint containment */
}
```

### 3. Be Careful with size Containment

```css
/* ❌ Bad - Size containment without dimensions */
.flexible-card {
  contain: size; /* Must specify width AND height */
}

/* ✅ Good - Size containment with dimensions */
.fixed-card {
  contain: size;
  width: 300px;
  height: 400px;
}
```

### 4. Test Before and After

```javascript
// Measure performance improvement
console.time("render");
// Render large list
console.timeEnd("render");

// Add containment and measure again
```

---

## Debugging

### Visualize Containment Boundaries

```css
.contained {
  contain: content;
  outline: 2px dashed red; /* Visualize boundary */
}
```

### Check in DevTools

```javascript
// Check computed contain value
const style = getComputedStyle(element);
console.log(style.contain); // e.g., "layout paint"
```

---

## Key Takeaways

1. **Layout containment** creates new formatting context
2. **Paint containment** prevents painting outside bounds
3. **Size containment** makes size independent of children (use with caution)
4. **content** = layout + paint (most common)
5. **strict** = layout + paint + size (for fixed-size elements)
6. Use on **repeated elements** for best performance
7. Essential for **virtual scrolling** and **infinite lists**
8. Enables **container queries** for component-based responsive design
9. Can improve rendering by **3-5x** in large lists
10. **Browser support** is excellent in modern browsers (Chrome 52+, Firefox 69+, Safari 15.4+)
