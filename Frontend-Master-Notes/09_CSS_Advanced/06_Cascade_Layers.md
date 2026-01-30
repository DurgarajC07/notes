# CSS Cascade Layers (@layer)

## Core Concept

CSS Cascade Layers (`@layer`) provide explicit control over the cascade, allowing developers to organize CSS into layers with defined precedence. This solves specificity battles and makes CSS more maintainable, especially in large codebases with third-party libraries.

---

## Basic Syntax

### Defining Layers

```css
/* Define layer order upfront */
@layer reset, base, components, utilities;

/* Add styles to layers */
@layer reset {
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }
}

@layer base {
  body {
    font-family:
      system-ui,
      -apple-system,
      sans-serif;
    line-height: 1.5;
    color: #333;
  }
}

@layer components {
  .button {
    padding: 0.5rem 1rem;
    background: #3b82f6;
    color: white;
  }
}

@layer utilities {
  .text-center {
    text-align: center;
  }
}
```

### Cascade Order

```css
/* Lower layers are weaker, higher layers are stronger */
@layer base, components, utilities;

@layer base {
  .text {
    color: black; /* Weakest */
  }
}

@layer components {
  .text {
    color: blue; /* Stronger */
  }
}

@layer utilities {
  .text {
    color: red; /* Strongest - wins! */
  }
}

/* Result: text is red */
```

---

## Layer Nesting

### Nested Layers

```css
@layer framework {
  @layer reset {
    * {
      margin: 0;
      padding: 0;
    }
  }

  @layer components {
    .button {
      padding: 0.5rem 1rem;
    }
  }
}

/* Access nested layers */
@layer framework.components {
  .button-large {
    padding: 1rem 2rem;
  }
}
```

### Anonymous Layers

```css
/* Layer without name - lowest priority */
@layer {
  .button {
    background: gray;
  }
}

/* Named layer - higher priority */
@layer components {
  .button {
    background: blue; /* Wins */
  }
}
```

---

## Real-World Pattern: Design System

```css
/* Define layer hierarchy for design system */
@layer reset, tokens, base, layout, components, utilities, overrides;

/* 1. Reset - Normalize browser defaults */
@layer reset {
  *,
  *::before,
  *::after {
    box-sizing: border-box;
  }

  body,
  h1,
  h2,
  h3,
  h4,
  h5,
  h6,
  p,
  ul,
  ol,
  figure {
    margin: 0;
    padding: 0;
  }

  button {
    font: inherit;
    cursor: pointer;
  }
}

/* 2. Tokens - CSS variables */
@layer tokens {
  :root {
    --color-primary: #3b82f6;
    --color-secondary: #8b5cf6;
    --spacing-unit: 0.25rem;
    --font-family-base: system-ui, sans-serif;
  }
}

/* 3. Base - Element defaults */
@layer base {
  body {
    font-family: var(--font-family-base);
    line-height: 1.5;
    color: #111827;
  }

  a {
    color: var(--color-primary);
    text-decoration: none;
  }

  a:hover {
    text-decoration: underline;
  }
}

/* 4. Layout - Page structure */
@layer layout {
  .container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 1rem;
  }

  .grid {
    display: grid;
    gap: 1rem;
  }

  .flex {
    display: flex;
    gap: 1rem;
  }
}

/* 5. Components - Reusable UI */
@layer components {
  .button {
    padding: calc(var(--spacing-unit) * 2) calc(var(--spacing-unit) * 4);
    border: none;
    border-radius: 0.375rem;
    font-weight: 500;
    transition: all 0.2s;
  }

  .button-primary {
    background: var(--color-primary);
    color: white;
  }

  .button-primary:hover {
    background: #2563eb;
  }

  .card {
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 0.5rem;
    padding: 1.5rem;
  }
}

/* 6. Utilities - Single-purpose classes */
@layer utilities {
  .text-center {
    text-align: center;
  }

  .mt-4 {
    margin-top: 1rem;
  }

  .hidden {
    display: none !important;
  }
}

/* 7. Overrides - Project-specific customizations */
@layer overrides {
  .button-large {
    padding: 1rem 2rem;
    font-size: 1.125rem;
  }
}
```

---

## Real-World Pattern: Third-Party Integration

```css
/* Define layer order */
@layer vendor, custom;

/* Import third-party CSS into vendor layer */
@import url("bootstrap.css") layer(vendor);
@import url("datepicker.css") layer(vendor);

/* Your custom styles in higher layer */
@layer custom {
  /* Override Bootstrap button with lower specificity */
  .btn {
    border-radius: 0.5rem; /* Overrides Bootstrap */
  }

  /* No need for !important or high specificity */
  .datepicker {
    z-index: 1000; /* Overrides plugin */
  }
}

/* Even simpler selectors win over vendor specificity */
@layer custom {
  button {
    /* This beats vendor's .btn.btn-primary selector! */
    font-weight: 500;
  }
}
```

---

## Real-World Pattern: Component Library

```css
/* Component library structure */
@layer library {
  @layer reset, base, components, variants, modifiers;
}

@layer library.reset {
  .lib-button {
    all: unset; /* Reset all properties */
  }
}

@layer library.base {
  .lib-button {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0.5rem 1rem;
    border-radius: 0.375rem;
    font-weight: 500;
    cursor: pointer;
  }
}

@layer library.components {
  .lib-button-primary {
    background: #3b82f6;
    color: white;
  }

  .lib-button-secondary {
    background: #6b7280;
    color: white;
  }
}

@layer library.variants {
  .lib-button-outline {
    background: transparent;
    border: 2px solid currentColor;
  }

  .lib-button-ghost {
    background: transparent;
  }
}

@layer library.modifiers {
  .lib-button-sm {
    padding: 0.375rem 0.75rem;
    font-size: 0.875rem;
  }

  .lib-button-lg {
    padding: 0.75rem 1.5rem;
    font-size: 1.125rem;
  }
}

/* Consumer can override any layer */
@layer library.components {
  .lib-button-primary {
    background: #ef4444; /* Custom brand color */
  }
}
```

---

## Real-World Pattern: Responsive Design

```css
@layer base, responsive;

@layer base {
  .container {
    padding: 1rem;
  }

  .grid {
    display: grid;
    grid-template-columns: 1fr;
  }
}

@layer responsive {
  @media (min-width: 768px) {
    .container {
      padding: 2rem;
    }

    .grid {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (min-width: 1024px) {
    .container {
      padding: 3rem;
    }

    .grid {
      grid-template-columns: repeat(3, 1fr);
    }
  }
}

/* Responsive layer takes precedence at all breakpoints */
```

---

## Real-World Pattern: Theme Variants

```css
@layer base, themes;

@layer base {
  .button {
    padding: 0.5rem 1rem;
    border: none;
    border-radius: 0.375rem;
  }
}

@layer themes {
  /* Light theme (default) */
  .button {
    background: #3b82f6;
    color: white;
  }

  /* Dark theme override */
  [data-theme="dark"] .button {
    background: #1e3a8a;
    color: #e0e7ff;
  }

  /* High contrast theme */
  [data-theme="high-contrast"] .button {
    background: black;
    color: yellow;
    border: 2px solid yellow;
  }
}
```

---

## Unlayered Styles

```css
/* Define layers */
@layer base, components;

@layer base {
  .text {
    color: blue;
  }
}

@layer components {
  .text {
    color: green;
  }
}

/* Unlayered styles beat ALL layers */
.text {
  color: red; /* WINS - unlayered styles have highest priority */
}

/* Use unlayered styles for:
 * - Critical overrides
 * - Developer tools
 * - Debugging
 */
```

---

## Import with Layers

```css
/* Import files into layers */
@import url("reset.css") layer(reset);
@import url("tokens.css") layer(tokens);
@import url("components.css") layer(components);
@import url("utilities.css") layer(utilities);

/* Define order separately if needed */
@layer reset, tokens, components, utilities;
```

---

## !important in Layers

```css
@layer base, utilities;

@layer base {
  .text {
    color: blue;
  }
}

@layer utilities {
  .text-red {
    color: red !important;
  }
}

/* !important follows layer order:
 * - utilities layer !important beats base layer !important
 * - But unlayered !important beats layered !important
 */

/* Unlayered !important (highest priority) */
.text-override {
  color: green !important; /* WINS over everything */
}
```

---

## Debugging Layers

### Visualize Layer Order

```css
/* Check what layer styles belong to */
@layer debug;

@layer debug {
  [data-debug-layer]::before {
    content: attr(data-debug-layer);
    position: absolute;
    top: 0;
    left: 0;
    background: yellow;
    padding: 2px 4px;
    font-size: 10px;
  }
}
```

```html
<button class="button" data-debug-layer="components">Click me</button>
```

### Browser DevTools

```css
/* Layers appear in DevTools cascade view */
@layer components {
  .button {
    background: blue;
  }
}

/* DevTools shows:
 * @layer components
 *   .button { background: blue; }
 */
```

---

## Performance Considerations

### Layer Definition

```css
/* ✅ Define layer order once at the top */
@layer reset, base, components, utilities;

/* ❌ Avoid redefining layers */
@layer reset, base;
@layer base, components; /* Creates confusion */
```

### Bundle Size

```css
/* Layers don't increase CSS size significantly
 * Small overhead: ~10-20 bytes per @layer declaration
 */

/* Benefit: Simpler selectors = smaller CSS */
@layer components {
  /* Simple selector in layer */
  .button {
    background: blue;
  }
}

/* Instead of high-specificity selector */
.page .section .card .button.primary {
  background: blue; /* Larger and harder to override */
}
```

---

## Migration Strategy

### Step 1: Wrap Existing CSS

```css
/* Wrap existing CSS in layers */
@layer legacy {
  /* Existing CSS */
  .button {
    background: blue;
  }
}
```

### Step 2: Add New Layers

```css
@layer legacy, modern;

@layer modern {
  /* New styles override legacy */
  .button {
    background: #3b82f6;
    border-radius: 0.375rem;
  }
}
```

### Step 3: Gradually Refactor

```css
/* Move refactored styles from legacy to modern */
@layer legacy, modern;

@layer modern {
  .button {
    /* Fully refactored button */
  }
}

@layer legacy {
  /* Old button styles removed */
}
```

---

## Browser Support

### Feature Detection

```css
@supports (at-rule(@layer)) {
  @layer base, components;

  @layer base {
    .button {
      background: blue;
    }
  }
}

/* Fallback for browsers without @layer support */
@supports not (at-rule(@layer)) {
  .button {
    background: blue;
  }
}
```

```javascript
// JavaScript detection
if (CSS.supports("at-rule(@layer)")) {
  console.log("Cascade layers supported");
}
```

---

## Best Practices

### 1. Define Layer Order Upfront

```css
/* ✅ Clear layer hierarchy */
@layer reset, tokens, base, layout, components, utilities;
```

### 2. Use Semantic Layer Names

```css
/* ✅ Clear purpose */
@layer reset, base, components, utilities;

/* ❌ Vague names */
@layer layer1, layer2, layer3;
```

### 3. Minimize Unlayered Styles

```css
/* Use unlayered styles only when necessary */
@layer base, components;

@layer components {
  .button {
    background: blue;
  }
}

/* Unlayered only for critical overrides */
.button-emergency-override {
  background: red;
}
```

### 4. Document Layer Purpose

```css
/**
 * Layer hierarchy:
 * - reset: Browser normalization
 * - tokens: Design tokens (CSS variables)
 * - base: Element defaults
 * - layout: Page structure
 * - components: Reusable UI components
 * - utilities: Single-purpose classes
 */
@layer reset, tokens, base, layout, components, utilities;
```

---

## Key Takeaways

1. **@layer** provides explicit cascade control
2. **Layer order** determines precedence (earlier = weaker)
3. **Unlayered styles** beat all layered styles
4. **Nested layers** create hierarchy (framework.components)
5. **!important in layers** follows layer order
6. **Third-party CSS** can be wrapped in lower layers
7. **Simpler selectors** work because layer order matters more than specificity
8. **Migration-friendly** - wrap legacy code in layers
9. **Browser support**: Chrome 99+, Firefox 97+, Safari 15.4+
10. **Best for**: Design systems, component libraries, third-party integration
