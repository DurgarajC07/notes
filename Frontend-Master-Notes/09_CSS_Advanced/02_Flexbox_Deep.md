# Flexbox Deep Dive

## Core Concept

Flexbox (Flexible Box Layout) is a one-dimensional layout system optimized for distributing space and aligning items within a container. Understanding flex-grow, flex-shrink, flex-basis, and alignment properties is crucial for modern responsive layouts.

---

## Flex Container Properties

### display: flex

```css
/* Create flex container */
.container {
  display: flex; /* or inline-flex */

  /* Flex direction */
  flex-direction: row; /* row | row-reverse | column | column-reverse */

  /* Wrapping */
  flex-wrap: nowrap; /* nowrap | wrap | wrap-reverse */

  /* Shorthand */
  flex-flow: row wrap; /* flex-direction flex-wrap */
}
```

### Main Axis Alignment (justify-content)

```css
.container {
  /* Horizontal alignment (when flex-direction: row) */
  justify-content: flex-start; /* default */
  justify-content: flex-end;
  justify-content: center;
  justify-content: space-between; /* Equal space between items */
  justify-content: space-around; /* Equal space around items */
  justify-content: space-evenly; /* Equal space distribution */
}

/* Real-world: Navigation bar */
.nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
}

.nav-left,
.nav-right {
  display: flex;
  gap: 1rem;
}
```

### Cross Axis Alignment (align-items)

```css
.container {
  /* Vertical alignment (when flex-direction: row) */
  align-items: stretch; /* default - fill container height */
  align-items: flex-start; /* top */
  align-items: flex-end; /* bottom */
  align-items: center; /* middle */
  align-items: baseline; /* align text baselines */
}

/* Real-world: Card with header, content, footer */
.card {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.card-header {
  flex-shrink: 0; /* Don't shrink */
}

.card-content {
  flex-grow: 1; /* Take remaining space */
}

.card-footer {
  flex-shrink: 0;
  align-self: flex-end; /* Override align-items */
}
```

### Multi-Line Alignment (align-content)

```css
.container {
  flex-wrap: wrap;

  /* Align wrapped lines */
  align-content: flex-start;
  align-content: flex-end;
  align-content: center;
  align-content: space-between;
  align-content: space-around;
  align-content: stretch; /* default */
}

/* Real-world: Image gallery */
.gallery {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  align-content: flex-start; /* Align rows to top */
}

.gallery-item {
  flex: 0 0 calc(33.333% - 1rem); /* 3 columns */
}

@media (max-width: 768px) {
  .gallery-item {
    flex: 0 0 calc(50% - 1rem); /* 2 columns on mobile */
  }
}
```

---

## Flex Item Properties

### flex-grow

```css
/* Control how items grow to fill available space */
.item {
  flex-grow: 0; /* default - don't grow */
  flex-grow: 1; /* grow equally with others */
  flex-grow: 2; /* grow twice as much */
}

/* Real-world: Sidebar layout */
.layout {
  display: flex;
  min-height: 100vh;
}

.sidebar {
  flex-grow: 0; /* Fixed width */
  flex-basis: 250px;
  background: #f5f5f5;
}

.main-content {
  flex-grow: 1; /* Take remaining space */
  padding: 2rem;
}
```

### flex-shrink

```css
/* Control how items shrink when space is limited */
.item {
  flex-shrink: 1; /* default - shrink equally */
  flex-shrink: 0; /* don't shrink */
  flex-shrink: 2; /* shrink twice as much */
}

/* Real-world: Toolbar with buttons */
.toolbar {
  display: flex;
  gap: 0.5rem;
}

.toolbar-button {
  flex-shrink: 0; /* Buttons maintain size */
  padding: 0.5rem 1rem;
}

.toolbar-search {
  flex-grow: 1; /* Search expands */
  flex-shrink: 1; /* But can shrink if needed */
  min-width: 200px; /* Minimum size */
}
```

### flex-basis

```css
/* Set initial size before growing/shrinking */
.item {
  flex-basis: auto; /* default - use width/height */
  flex-basis: 200px; /* fixed size */
  flex-basis: 50%; /* percentage */
  flex-basis: content; /* based on content size */
}

/* Real-world: Responsive grid */
.grid {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}

.grid-item {
  flex-basis: calc(25% - 1rem); /* 4 columns */
  flex-grow: 0;
  flex-shrink: 0;
}

@media (max-width: 992px) {
  .grid-item {
    flex-basis: calc(33.333% - 1rem); /* 3 columns */
  }
}

@media (max-width: 768px) {
  .grid-item {
    flex-basis: calc(50% - 1rem); /* 2 columns */
  }
}

@media (max-width: 480px) {
  .grid-item {
    flex-basis: 100%; /* 1 column */
  }
}
```

### flex Shorthand

```css
/* flex: flex-grow flex-shrink flex-basis */
.item {
  flex: 0 1 auto; /* default */
  flex: 1; /* flex: 1 1 0% */
  flex: auto; /* flex: 1 1 auto */
  flex: none; /* flex: 0 0 auto */
  flex: 1 200px; /* flex: 1 1 200px */
}

/* Common patterns */
.equal-width-items {
  flex: 1; /* All items same width */
}

.fixed-width-item {
  flex: 0 0 200px; /* Fixed 200px width */
}

.shrinkable-item {
  flex: 1 1 auto; /* Can grow and shrink */
}

.non-shrinkable-item {
  flex: 1 0 auto; /* Can grow but not shrink */
}
```

---

## Order and Alignment

### order

```css
/* Change visual order without changing DOM */
.item {
  order: 0; /* default */
}

.item-1 {
  order: 2;
}
.item-2 {
  order: 1;
}
.item-3 {
  order: 3;
}

/* Real-world: Mobile reordering */
.header {
  display: flex;
}

.logo {
  order: 1;
}
.nav {
  order: 2;
}
.search {
  order: 3;
}

@media (max-width: 768px) {
  .logo {
    order: 2;
  } /* Logo in middle */
  .nav {
    order: 3;
  } /* Nav on right */
  .search {
    order: 1;
  } /* Search on left */
}
```

### align-self

```css
/* Override align-items for individual item */
.container {
  align-items: center;
}

.item {
  align-self: flex-start; /* Override to top */
  align-self: flex-end; /* Override to bottom */
  align-self: center;
  align-self: baseline;
  align-self: stretch;
}

/* Real-world: Card with different-height items */
.card-grid {
  display: flex;
  gap: 1rem;
  align-items: flex-start;
}

.card-featured {
  align-self: stretch; /* Full height */
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.card-regular {
  align-self: flex-start; /* Natural height */
}
```

---

## Real-World Pattern: Holy Grail Layout

```html
<div class="holy-grail">
  <header class="header">Header</header>
  <div class="content-wrapper">
    <aside class="sidebar-left">Left Sidebar</aside>
    <main class="main-content">Main Content</main>
    <aside class="sidebar-right">Right Sidebar</aside>
  </div>
  <footer class="footer">Footer</footer>
</div>
```

```css
.holy-grail {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.header,
.footer {
  flex-shrink: 0; /* Don't shrink */
  padding: 1rem;
  background: #333;
  color: white;
}

.content-wrapper {
  display: flex;
  flex-grow: 1; /* Take remaining vertical space */
}

.sidebar-left,
.sidebar-right {
  flex: 0 0 200px; /* Fixed width */
  padding: 1rem;
  background: #f5f5f5;
}

.main-content {
  flex-grow: 1; /* Take remaining horizontal space */
  padding: 2rem;
}

/* Mobile: Stack vertically */
@media (max-width: 768px) {
  .content-wrapper {
    flex-direction: column;
  }

  .sidebar-left,
  .sidebar-right {
    flex-basis: auto; /* Natural height */
  }

  .sidebar-left {
    order: 1;
  }
  .main-content {
    order: 2;
  }
  .sidebar-right {
    order: 3;
  }
}
```

---

## Real-World Pattern: Card Layout

```html
<div class="card">
  <img src="image.jpg" alt="Card image" class="card-image" />
  <div class="card-body">
    <h3 class="card-title">Card Title</h3>
    <p class="card-description">Description text that might be long...</p>
  </div>
  <div class="card-actions">
    <button>Action 1</button>
    <button>Action 2</button>
  </div>
</div>
```

```css
.card {
  display: flex;
  flex-direction: column;
  height: 100%; /* Full height in grid */
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
}

.card-image {
  flex-shrink: 0; /* Maintain aspect ratio */
  width: 100%;
  height: 200px;
  object-fit: cover;
}

.card-body {
  flex-grow: 1; /* Take remaining space */
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.card-title {
  flex-shrink: 0;
  margin: 0;
}

.card-description {
  flex-grow: 1; /* Push actions to bottom */
  margin: 0;
}

.card-actions {
  flex-shrink: 0; /* Stay at bottom */
  display: flex;
  gap: 0.5rem;
  padding: 1rem;
  border-top: 1px solid #eee;
}

.card-actions button {
  flex: 1; /* Equal width buttons */
}
```

---

## Real-World Pattern: Responsive Navigation

```html
<nav class="nav">
  <div class="nav-brand">
    <img src="logo.svg" alt="Logo" />
  </div>
  <ul class="nav-menu">
    <li><a href="#home">Home</a></li>
    <li><a href="#about">About</a></li>
    <li><a href="#services">Services</a></li>
    <li><a href="#contact">Contact</a></li>
  </ul>
  <div class="nav-actions">
    <button class="btn-login">Login</button>
    <button class="btn-signup">Sign Up</button>
  </div>
</nav>
```

```css
.nav {
  display: flex;
  align-items: center;
  gap: 2rem;
  padding: 1rem 2rem;
  background: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.nav-brand {
  flex-shrink: 0;
}

.nav-brand img {
  height: 40px;
}

.nav-menu {
  display: flex;
  flex-grow: 1; /* Take center space */
  gap: 2rem;
  list-style: none;
  margin: 0;
  padding: 0;
}

.nav-actions {
  display: flex;
  gap: 1rem;
  flex-shrink: 0;
}

/* Mobile: Hamburger menu */
@media (max-width: 768px) {
  .nav {
    flex-wrap: wrap;
  }

  .nav-brand {
    order: 1;
  }

  .nav-toggle {
    order: 2;
    margin-left: auto;
  }

  .nav-menu {
    order: 3;
    flex-basis: 100%;
    flex-direction: column;
    display: none; /* Toggle with JS */
  }

  .nav-menu.active {
    display: flex;
  }

  .nav-actions {
    order: 4;
    flex-basis: 100%;
    flex-direction: column;
  }
}
```

---

## Real-World Pattern: Sticky Footer

```html
<div class="page-container">
  <header class="header">Header</header>
  <main class="content">Main Content</main>
  <footer class="footer">Footer</footer>
</div>
```

```css
.page-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.header {
  flex-shrink: 0;
  padding: 2rem;
  background: #333;
  color: white;
}

.content {
  flex-grow: 1; /* Push footer to bottom */
  padding: 2rem;
}

.footer {
  flex-shrink: 0;
  padding: 2rem;
  background: #f5f5f5;
  text-align: center;
}
```

---

## Advanced Patterns

### Centering (Horizontal and Vertical)

```css
/* Perfect centering */
.center-container {
  display: flex;
  justify-content: center; /* Horizontal */
  align-items: center; /* Vertical */
  min-height: 100vh;
}

/* Multiple centered items */
.center-stack {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 1rem;
  min-height: 100vh;
}
```

### Equal Height Columns

```css
.equal-height-container {
  display: flex;
  gap: 1rem;
}

.column {
  flex: 1; /* Equal width */
  /* All columns automatically have equal height */
  background: white;
  padding: 1rem;
  border: 1px solid #ddd;
}
```

### Space Distribution

```css
/* Evenly spaced items with margins */
.spaced-container {
  display: flex;
  justify-content: space-between;
  padding: 1rem;
}

/* Alternative with gap (modern) */
.spaced-container-modern {
  display: flex;
  gap: 1rem; /* CSS Gap is better than margins */
  padding: 1rem;
}
```

---

## Performance Considerations

### Flexbox vs Grid

```css
/* Use Flexbox for: */
/* - Single-direction layouts */
/* - Content-driven layouts */
/* - Small-scale components */

.flex-layout {
  display: flex;
}

/* Use Grid for: */
/* - Two-dimensional layouts */
/* - Layout-driven designs */
/* - Large-scale page layouts */

.grid-layout {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
}
```

### Avoid Nested Flex Containers

```css
/* ❌ Excessive nesting */
.container {
  display: flex;
}

.container > div {
  display: flex;
}

.container > div > div {
  display: flex;
}

/* ✅ Flatten when possible */
.container {
  display: flex;
  flex-wrap: wrap;
}

.container > * {
  flex: 1 1 200px;
}
```

---

## Browser Compatibility

### Flexbox Bugs and Workarounds

```css
/* IE 10-11: min-height bug */
.flex-container {
  display: flex;
  flex-direction: column;
}

.flex-item {
  flex-shrink: 0; /* Fix IE min-height */
}

/* Safari: flex-basis with border-box */
.safari-fix {
  flex-basis: calc(33.333% - 1rem);
  box-sizing: border-box;
}

/* All browsers: Use gap instead of margins */
.modern-spacing {
  display: flex;
  gap: 1rem; /* Better than margin hacks */
}
```

---

## Key Takeaways

1. **flex-grow** controls how items expand to fill available space
2. **flex-shrink** controls how items shrink when space is limited
3. **flex-basis** sets the initial size before growing/shrinking
4. **flex shorthand** combines all three: `flex: grow shrink basis`
5. **justify-content** aligns items on the main axis
6. **align-items** aligns items on the cross axis
7. **align-self** overrides alignment for individual items
8. **order** changes visual order without changing DOM
9. **gap** property is modern way to add spacing (better than margins)
10. Use **flex: 1** for equal-width items, **flex: 0 0 auto** for fixed-width items
