# 🎨 CSS Grid Layout Deep Dive

> CSS Grid is a powerful two-dimensional layout system for the web. It allows you to create complex, responsive layouts with clean, semantic HTML and minimal CSS. Understanding Grid is essential for modern frontend development.

---

## 📖 1. Concept Explanation

### What is CSS Grid?

**CSS Grid Layout** is a two-dimensional layout system that controls both rows and columns simultaneously, unlike Flexbox which is primarily one-dimensional.

```css
.container {
  display: grid;
  grid-template-columns: 1fr 2fr 1fr; /* 3 columns */
  grid-template-rows: 100px auto 50px; /* 3 rows */
  gap: 20px; /* Spacing */
}
```

**Grid Container:**

```html
<div class="grid-container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
  <div class="item">4</div>
</div>
```

### Core Concepts

**1. Grid Container Properties:**

```css
.container {
  display: grid;

  /* Define columns */
  grid-template-columns: 200px 1fr 2fr;

  /* Define rows */
  grid-template-rows: 100px auto minmax(50px, auto);

  /* Gaps */
  gap: 20px; /* row-gap & column-gap */

  /* Alignment */
  justify-items: start; /* Horizontal (items)  */
  align-items: center; /* Vertical (items) */
  justify-content: center; /* Horizontal (grid) */
  align-content: start; /* Vertical (grid) */

  /* Auto behavior */
  grid-auto-rows: 100px;
  grid-auto-columns: 1fr;
  grid-auto-flow: row; /* or column, dense */
}
```

**2. Grid Item Properties:**

```css
.item {
  /* Position by line numbers */
  grid-column: 1 / 3; /* Start at line 1, end at line 3 */
  grid-row: 2 / 4;

  /* Position by span */
  grid-column: span 2; /* Span 2 columns */
  grid-row: span 1;

  /* Named areas */
  grid-area: header; /* Place in named area */

  /* Self-alignment */
  justify-self: end;
  align-self: start;
}
```

**3. FR Unit (Fractional Unit):**

```css
/* 1fr = 1 fraction of available space */
.container {
  grid-template-columns: 1fr 2fr 1fr;
  /* Available space split: 25% 50% 25% */
}
```

**4. Grid Lines:**

```
      1       2       3       4
    ┌───────┬───────┬───────┐
  1 │ Cell  │ Cell  │ Cell  │
    ├───────┼───────┼───────┤
  2 │ Cell  │ Cell  │ Cell  │
    ├───────┼───────┼───────┤
  3 │ Cell  │ Cell  │ Cell  │
    └───────┴───────┴───────┘
```

### Grid Template Areas

```css
.container {
  display: grid;
  grid-template-areas:
    "header header header"
    "sidebar main main"
    "footer footer footer";
  grid-template-columns: 200px 1fr 1fr;
  grid-template-rows: auto 1fr auto;
}

.header {
  grid-area: header;
}
.sidebar {
  grid-area: sidebar;
}
.main {
  grid-area: main;
}
.footer {
  grid-area: footer;
}
```

---

## 🧠 2. Why It Matters in Real Projects

### Production Benefits

**1. Clean, Semantic HTML:**

```html
<!-- ❌ OLD: Div soup with classes -->
<div class="row">
  <div class="col-md-4">Sidebar</div>
  <div class="col-md-8">
    <div class="row">
      <div class="col-md-6">Content 1</div>
      <div class="col-md-6">Content 2</div>
    </div>
  </div>
</div>

<!-- ✅ NEW: Clean Grid -->
<div class="layout">
  <aside>Sidebar</aside>
  <main>Content 1</main>
  <section>Content 2</section>
</div>

<style>
  .layout {
    display: grid;
    grid-template-columns: 300px 1fr 1fr;
  }
</style>
```

**2. Responsive Without Media Queries:**

```css
/* Auto-fit creates responsive columns */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
}

/* Automatically adjusts:
   - Wide screen: 4 columns
   - Tablet: 2 columns
   - Mobile: 1 column
*/
```

**3. Complex Layouts Simplified:**

```css
/* Magazine-style layout */
.magazine {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 1rem;
}

.feature {
  grid-column: span 4; /* Takes 4 columns */
  grid-row: span 2; /* Takes 2 rows */
}

.article {
  grid-column: span 2; /* Takes 2 columns */
}
```

### Real-World Performance

**Memory & Paint:**

- Grid is GPU-accelerated
- Fewer DOM nodes (no wrapper divs)
- Efficient reflows (grid-aware layout engine)

**Business Impact:**

- **Faster development:** 50% less CSS code
- **Better maintainability:** Clear visual structure
- **Improved UX:** Consistent layouts across devices

---

## ⚙️ 3. Internal Working

### How Browser Renders Grid

```javascript
// Simplified browser grid algorithm
function layoutGrid(container) {
  // 1. Parse grid template
  const tracks = parseGridTemplate(container);

  // 2. Resolve track sizes
  tracks.forEach((track) => {
    if (track.unit === "fr") {
      track.size = calculateFractionSpace(track, availableSpace);
    } else if (track.unit === "auto") {
      track.size = calculateAutoSize(track);
    } else if (track.unit === "minmax") {
      track.size = clamp(track.min, track.max, contentSize);
    }
  });

  // 3. Position items
  items.forEach((item) => {
    const startLine = resolveGridLine(item.gridColumnStart);
    const endLine = resolveGridLine(item.gridColumnEnd);
    const position = calculatePosition(startLine, endLine, tracks);

    item.x = position.x;
    item.y = position.y;
    item.width = position.width;
    item.height = position.height;
  });

  // 4. Handle auto-placement
  handleAutoPlacement(items, tracks);
}
```

### Track Sizing Algorithm

```css
/* Track sizing resolution order: */
.container {
  grid-template-columns:
    100px /* 1. Fixed: Use exact size */
    minmax(50px, 1fr) /* 2. Minmax: Clamp between min/max */
    auto /* 3. Auto: Based on content */
    1fr; /* 4. Fraction: Distribute remaining space */
}
```

**Algorithm steps:**

1. **Resolve fixed sizes** (px, em, %)
2. **Calculate minimums** (minmax, auto)
3. **Distribute remaining space** to fr units
4. **Apply maximums** (minmax)

### Auto-Placement Algorithm

```css
.container {
  grid-auto-flow: row; /* or column, dense */
}
```

**Row flow:**

```
1. Place items with explicit position
2. For remaining items:
   a. Find next empty cell in current row
   b. If item doesn't fit, move to next row
   c. Place item
```

**Dense packing:**

```
1. Same as row flow
2. But try to fill earlier gaps
3. Can reorder items visually
```

---

## ✅ 4. Best Practices

### DO ✅

```css
/* 1. Use semantic grid-template-areas */
.layout {
  display: grid;
  grid-template-areas:
    "header header header"
    "nav    main   aside"
    "footer footer footer";
  grid-template-columns: 200px 1fr 300px;
  grid-template-rows: auto 1fr auto;
}

/* Clear, readable, maintainable */

/* 2. Use repeat() for patterns */
.grid {
  grid-template-columns: repeat(12, 1fr); /* 12-column grid */
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); /* Responsive */
}

/* 3. Use gap instead of margins */
.grid {
  display: grid;
  gap: 1rem; /* Consistent spacing */
}
/* No more margin hacks! */

/* 4. Use minmax() for flexible sizing */
.grid {
  grid-template-columns: minmax(200px, 1fr) 2fr;
  /* Column never smaller than 200px, grows up to 1fr */
}

/* 5. Use auto-fit/auto-fill for responsive */
.gallery {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
}
/* Automatically responsive! */

/* 6. Name grid lines for clarity */
.container {
  grid-template-columns:
    [sidebar-start] 250px
    [sidebar-end main-start] 1fr
    [main-end];
}

.sidebar {
  grid-column: sidebar-start / sidebar-end;
}
```

### DON'T ❌

```css
/* ❌ Don't use grid for everything */
.navbar {
  display: grid; /* Overkill for 1D layout */
  grid-template-columns: 1fr auto auto auto;
}
/* Use Flexbox for 1D layouts */

/* ❌ Don't mix grid units unnecessarily */
.container {
  grid-template-columns: 200px 30% 1fr 2fr auto;
  /* Hard to reason about */
}
/* Stick to consistent units */

/* ❌ Don't overuse dense packing */
.grid {
  grid-auto-flow: dense; /* Can confuse reading order */
}
/* Use sparingly, consider a11y */

/* ❌ Don't forget fallbacks for old browsers */
.grid {
  display: grid; /* IE11 needs prefixes */
  display: -ms-grid;
}
/* Or use @supports */

/* ❌ Don't use negative line numbers without understanding */
.item {
  grid-column: 1 / -1; /* Spans all columns */
}
/* -1 = last line, -2 = second to last, etc. */
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Forgetting Grid Container Display

```css
/* ❌ WRONG: No display: grid */
.container {
  grid-template-columns: 1fr 1fr 1fr;
  /* Doesn't work! */
}

/* ✅ CORRECT */
.container {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
}
```

### Mistake #2: Confusing auto-fit vs auto-fill

```css
/* auto-fit: Collapses empty tracks */
.grid {
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
}
/* 3 items in 6-column grid → 3 columns (each 1fr) */

/* auto-fill: Keeps empty tracks */
.grid {
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
}
/* 3 items in 6-column grid → 6 columns (3 empty) */

/* ✅ Use auto-fit for equal-width items */
/* ✅ Use auto-fill for fixed-width items */
```

### Mistake #3: Grid Line Numbering Confusion

```css
/* Grid lines start at 1, not 0 */
.container {
  grid-template-columns: 1fr 1fr 1fr;
}

/* Lines: 1  2  3  4 */
/*       |  |  |  | */

/* ❌ WRONG */
.item {
  grid-column: 0 / 2; /* Line 0 doesn't exist */
}

/* ✅ CORRECT */
.item {
  grid-column: 1 / 3; /* Start line 1, end line 3 */
}
```

### Mistake #4: Overlapping Items Accidentally

```css
/* ❌ Items overlap unexpectedly */
.item1 {
  grid-column: 1 / 3;
  grid-row: 1 / 2;
}

.item2 {
  grid-column: 2 / 4; /* Overlaps item1! */
  grid-row: 1 / 2;
}

/* ✅ Use z-index if overlap is intentional */
.item1 {
  grid-column: 1 / 3;
  grid-row: 1 / 2;
  z-index: 1; /* On top */
}

.item2 {
  grid-column: 2 / 4;
  grid-row: 1 / 2;
  z-index: 0; /* Below */
}
```

---

## 🔐 6. Security Considerations

### Content Injection

```css
/* ❌ VULNERABLE: User-controlled grid areas */
.layout {
  grid-template-areas: var(--user-template); /* Dangerous! */
}

/* ✅ SAFE: Whitelist allowed layouts */
.layout[data-layout="default"] {
  grid-template-areas: "header" "main" "footer";
}

.layout[data-layout="sidebar"] {
  grid-template-areas: "header header" "sidebar main" "footer footer";
}
```

### Performance DoS

```css
/* ❌ VULNERABLE: Unrestricted repeat */
.grid {
  grid-template-columns: repeat(var(--user-columns), 1fr);
  /* User sets --user-columns: 100000 → browser crash */
}

/* ✅ SAFE: Limit maximum */
.grid {
  grid-template-columns: repeat(min(var(--user-columns, 4), 12), 1fr);
  /* Clamped to max 12 columns */
}
```

---

## 🚀 7. Performance Optimization

### 1. Minimize Reflows

```css
/* ✅ FAST: Static grid */
.container {
  display: grid;
  grid-template-columns: 300px 1fr;
  /* No reflow on content changes */
}

/* ❌ SLOW: Dynamic grid */
.container {
  display: grid;
  grid-template-columns: auto 1fr;
  /* Reflows when auto column content changes */
}
```

### 2. Use Subgrid for Nested Grids

```css
/* ✅ BETTER: Subgrid (inherits parent tracks) */
.parent {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
}

.child {
  display: grid;
  grid-template-columns: subgrid; /* Aligns with parent */
  grid-column: span 3;
}

/* Fewer layout calculations, better alignment */
```

### 3. Avoid Percentage Gaps with FR Units

```css
/* ❌ SLOWER: Percentage gap */
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 5%; /* Recalculated on resize */
}

/* ✅ FASTER: Fixed gap */
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 2rem; /* Fixed size */
}
```

---

## 🧪 8. Code Examples

### Example 1: Responsive Card Grid

```css
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 2rem;
  padding: 2rem;
}

.card {
  background: white;
  border-radius: 8px;
  padding: 1.5rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* Automatically responsive:
   - 4 columns on wide screens
   - 3 columns on desktop
   - 2 columns on tablet
   - 1 column on mobile
*/
```

### Example 2: Holy Grail Layout

```css
.holy-grail {
  display: grid;
  grid-template-areas:
    "header header header"
    "nav    main   aside"
    "footer footer footer";
  grid-template-columns: 200px 1fr 300px;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
  gap: 1rem;
}

.header {
  grid-area: header;
}
.nav {
  grid-area: nav;
}
.main {
  grid-area: main;
}
.aside {
  grid-area: aside;
}
.footer {
  grid-area: footer;
}

/* Responsive */
@media (max-width: 768px) {
  .holy-grail {
    grid-template-areas:
      "header"
      "main"
      "nav"
      "aside"
      "footer";
    grid-template-columns: 1fr;
  }
}
```

### Example 3: Magazine Layout

```css
.magazine {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  grid-auto-rows: 200px;
  gap: 1rem;
}

.article--large {
  grid-column: span 4;
  grid-row: span 2;
}

.article--medium {
  grid-column: span 2;
  grid-row: span 2;
}

.article--small {
  grid-column: span 2;
  grid-row: span 1;
}

/* Creates Pinterest-style masonry */
.magazine {
  grid-auto-flow: dense; /* Fill gaps */
}
```

### Example 4: Dashboard Layout

```css
.dashboard {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  grid-template-rows: auto;
  gap: 1.5rem;
  padding: 1.5rem;
}

.widget--full {
  grid-column: 1 / -1; /* Span all columns */
}

.widget--half {
  grid-column: span 6; /* Half width */
}

.widget--third {
  grid-column: span 4; /* Third width */
}

.widget--tall {
  grid-row: span 2; /* Twice height */
}

/* Example usage */
<div class="dashboard">
  <div class="widget widget--full">Header Stats</div>
  <div class="widget widget--half widget--tall">Chart</div>
  <div class="widget widget--half">Table</div>
  <div class="widget widget--third">Metric 1</div>
  <div class="widget widget--third">Metric 2</div>
  <div class="widget widget--third">Metric 3</div>
</div>
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: Admin Dashboard

```css
.admin-layout {
  display: grid;
  grid-template-areas:
    "sidebar header"
    "sidebar main";
  grid-template-columns: 250px 1fr;
  grid-template-rows: 60px 1fr;
  height: 100vh;
}

.sidebar {
  grid-area: sidebar;
  background: #2c3e50;
  color: white;
  overflow-y: auto;
}

.header {
  grid-area: header;
  background: white;
  border-bottom: 1px solid #ddd;
  display: flex;
  align-items: center;
  padding: 0 2rem;
}

.main {
  grid-area: main;
  overflow-y: auto;
  padding: 2rem;
}

/* Collapsible sidebar */
.admin-layout--collapsed {
  grid-template-columns: 60px 1fr;
}
```

### Use Case 2: E-commerce Product Grid

```css
.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 2rem;
}

/* Featured product (larger) */
.product--featured {
  grid-column: span 2;
  grid-row: span 2;
}

/* Sale badge positioning using Grid */
.product {
  display: grid;
  grid-template-areas:
    "image image"
    "title price"
    "desc  desc"
    "button button";
  gap: 0.5rem 1rem;
}

.product__image {
  grid-area: image;
}
.product__title {
  grid-area: title;
}
.product__price {
  grid-area: price;
  justify-self: end;
}
.product__desc {
  grid-area: desc;
}
.product__button {
  grid-area: button;
}
```

### Use Case 3: Responsive Form Layout

```css
.form {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 1.5rem;
}

.form-field--full {
  grid-column: 1 / -1; /* Full width */
}

/* Group related fields */
.form-group {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

/* Example */
<form class="form">
  <div class="form-field form-field--full">
    <input type="text" placeholder="Full Name">
  </div>

  <div class="form-group">
    <input type="email" placeholder="Email">
    <input type="tel" placeholder="Phone">
  </div>

  <div class="form-field form-field--full">
    <textarea placeholder="Message"></textarea>
  </div>

  <button class="form-field--full">Submit</button>
</form>
```

---

## ❓ 10. Interview Questions

### Q1: Explain CSS Grid and when to use it vs Flexbox.

**Answer:**

| Feature             | CSS Grid                       | Flexbox                     |
| ------------------- | ------------------------------ | --------------------------- |
| **Dimensions**      | 2D (rows & columns)            | 1D (row or column)          |
| **Use for**         | Page layouts, complex grids    | Navigation, toolbars, cards |
| **Control**         | Both axes simultaneously       | One axis at a time          |
| **Alignment**       | Items independently positioned | Items flex within container |
| **Browser support** | Modern browsers                | Wider support               |

**When to use Grid:**

```css
/* Complex 2D layouts */
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
}
```

**When to use Flexbox:**

```css
/* 1D alignment, navigation */
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
```

**Can be combined:**

```css
.grid-container {
  display: grid; /* Outer 2D layout */
}

.grid-item {
  display: flex; /* Inner 1D alignment */
}
```

---

### Q2: Explain fr unit and how it's calculated.

**Answer:**

**FR (Fractional Unit)** represents a fraction of available space **after** fixed and content-based sizes.

**Calculation:**

```css
.container {
  width: 1000px;
  grid-template-columns: 200px 1fr 2fr;
  gap: 20px;
}

/* Step 1: Calculate available space
   Total: 1000px
   Fixed: 200px
   Gaps: 20px
   Available: 1000 - 200 - 20 = 780px
   
   Step 2: Sum fr values
   1fr + 2fr = 3fr
   
   Step 3: Calculate fr size
   780px ÷ 3fr = 260px per fr
   
   Step 4: Assign sizes
   Column 1: 200px (fixed)
   Column 2: 1fr = 260px
   Column 3: 2fr = 520px
*/
```

**With minmax:**

```css
.container {
  grid-template-columns: 200px minmax(100px, 1fr) 2fr;
}

/* 1fr respects min/max constraints */
```

---

### Q3: What's the difference between auto-fit and auto-fill?

**Answer:**

Both create responsive columns, but handle empty space differently.

**auto-fill:** Creates as many tracks as fit, even if empty

```css
.grid {
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  width: 1000px;
}

/* 3 items in 1000px container:
   - Creates 4 columns (4 × 200px = 800px < 1000px)
   - Items take 200px each
   - 1 empty column remains
*/
```

**auto-fit:** Collapses empty tracks, expands filled ones

```css
.grid {
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  width: 1000px;
}

/* 3 items in 1000px container:
   - Creates 3 columns (empty ones collapse)
   - Items expand to 333px each (1000px ÷ 3)
*/
```

**Visual comparison:**

```
auto-fill:  [Item 1][Item 2][Item 3][ empty ]
auto-fit:   [  Item 1  ][  Item 2  ][  Item 3  ]
```

**Use cases:**

- **auto-fit:** Card galleries (items expand)
- **auto-fill:** Fixed-width items (maintain size)

---

### Q4: How do you create a responsive layout without media queries?

**Answer:**

Use `repeat()` with `auto-fit/auto-fill` and `minmax()`:

```css
.responsive-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 2rem;
}

/* Automatically:
   - 4+ columns on ultra-wide (1400px+)
   - 3 columns on desktop (900-1400px)
   - 2 columns on tablet (600-900px)
   - 1 column on mobile (<600px)
   
   Based on 280px minimum
*/
```

**Advanced responsive patterns:**

```css
/* Responsive with max columns */
.grid {
  grid-template-columns: repeat(auto-fit, minmax(min(280px, 100%), 1fr));
  /* On narrow screens, 100% prevents overflow */
}

/* Responsive sidebar */
.layout {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
}
/* Sidebar collapses below content on narrow screens */

/* Responsive 12-column */
.container {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
}

.item {
  grid-column: span min(12, max(4, var(--span)));
  /* Responsive column spanning */
}
```

---

### Q5: Explain grid-auto-flow and when to use dense packing.

**Answer:**

**grid-auto-flow** controls how auto-placed items are inserted:

```css
/* Default: row (left-to-right, top-to-bottom) */
.grid {
  grid-auto-flow: row;
}

/* Column (top-to-bottom, left-to-right) */
.grid {
  grid-auto-flow: column;
}

/* Dense packing (fill gaps) */
.grid {
  grid-auto-flow: dense;
}
```

**Dense packing example:**

```css
.gallery {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-auto-flow: row dense;
}

.item--wide {
  grid-column: span 2;
}
.item--tall {
  grid-row: span 2;
}

/* Without dense:
   [1][2  ][__][__]
   [3][4][5][6]
   
   With dense:
   [1][2  ][3][4]
   [5][6][7][8]
   
   Item 3 fills gap
*/
```

**When to use dense:**

- ✅ Image galleries (Pinterest-style)
- ✅ Dashboard widgets (any order)
- ❌ Content with reading order (accessibility issue)
- ❌ Forms (confusing tab order)

**Accessibility warning:**

```css
/* Dense changes visual order, not DOM order */
.grid {
  grid-auto-flow: dense; /* Screen readers still read DOM order */
}

/* Add ARIA if needed */
<div aria-flowto="item-3">Item 1</div>
```

---

## 🧩 11. Design Patterns

### Pattern 1: Responsive Container Queries (Future)

```css
/* With Container Queries (upcoming) */
.card {
  container-type: inline-size;
}

.card__grid {
  display: grid;
  grid-template-columns: 1fr;
}

@container (min-width: 400px) {
  .card__grid {
    grid-template-columns: 1fr 1fr;
  }
}
```

### Pattern 2: Aspect Ratio Grid

```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
}

.grid__item {
  aspect-ratio: 16 / 9; /* Maintain ratio */
  overflow: hidden;
}
```

---

## 🎯 Summary

CSS Grid is the most powerful layout system in CSS:

- **2D layout:** Control rows and columns simultaneously
- **Semantic:** grid-template-areas for readable layouts
- **Responsive:** auto-fit/auto-fill for automatic columns
- **Flexible:** fr units, minmax(), repeat()

**Production checklist:**

- ✅ Use Grid for 2D layouts, Flexbox for 1D
- ✅ Name grid areas for maintainability
- ✅ Use repeat(auto-fit, minmax()) for responsive
- ✅ Avoid dense packing for content
- ✅ Use gap instead of margins
- ✅ Test with various content sizes
- ✅ Check browser support (@supports)

Master CSS Grid to build modern, responsive layouts with ease! 🎨
