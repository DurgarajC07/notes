# CSS Custom Properties (Variables)

## Core Concept

CSS Custom Properties (CSS Variables) enable dynamic, cascading values that can be changed at runtime, inherited through the DOM, and scoped to specific selectors. They're essential for theming, responsive design, and maintainable stylesheets.

---

## Basic Syntax

### Declaring and Using Variables

```css
:root {
  /* Declare custom properties with -- prefix */
  --primary-color: #3b82f6;
  --secondary-color: #8b5cf6;
  --spacing-unit: 8px;
  --font-size-base: 16px;
}

.button {
  /* Use with var() function */
  background-color: var(--primary-color);
  padding: var(--spacing-unit);
  font-size: var(--font-size-base);
}

/* Fallback values */
.button-alt {
  color: var(--text-color, #333); /* Use #333 if --text-color not defined */
}
```

### Scoped Variables

```css
:root {
  --button-bg: #3b82f6;
  --button-text: white;
}

/* Override for specific sections */
.dark-section {
  --button-bg: #1e3a8a;
  --button-text: #e0e7ff;
}

.button {
  background: var(--button-bg);
  color: var(--button-text);
  /* Inherits from parent scope */
}
```

---

## Theming System

### Light and Dark Themes

```css
:root {
  /* Light theme (default) */
  --bg-primary: #ffffff;
  --bg-secondary: #f3f4f6;
  --text-primary: #111827;
  --text-secondary: #6b7280;
  --border-color: #e5e7eb;
}

[data-theme="dark"] {
  /* Dark theme overrides */
  --bg-primary: #111827;
  --bg-secondary: #1f2937;
  --text-primary: #f9fafb;
  --text-secondary: #9ca3af;
  --border-color: #374151;
}

/* Components automatically adapt */
body {
  background-color: var(--bg-primary);
  color: var(--text-primary);
}

.card {
  background: var(--bg-secondary);
  border: 1px solid var(--border-color);
}

.text-muted {
  color: var(--text-secondary);
}
```

```javascript
// Toggle theme
function toggleTheme() {
  const root = document.documentElement;
  const currentTheme = root.getAttribute("data-theme");
  root.setAttribute("data-theme", currentTheme === "dark" ? "light" : "dark");
}
```

### Brand Color System

```css
:root {
  /* Primary brand colors */
  --color-primary-50: #eff6ff;
  --color-primary-100: #dbeafe;
  --color-primary-200: #bfdbfe;
  --color-primary-300: #93c5fd;
  --color-primary-400: #60a5fa;
  --color-primary-500: #3b82f6; /* Base */
  --color-primary-600: #2563eb;
  --color-primary-700: #1d4ed8;
  --color-primary-800: #1e40af;
  --color-primary-900: #1e3a8a;

  /* Semantic colors */
  --color-success: #10b981;
  --color-warning: #f59e0b;
  --color-error: #ef4444;
  --color-info: #3b82f6;
}

.button-primary {
  background: var(--color-primary-500);
  color: white;
}

.button-primary:hover {
  background: var(--color-primary-600);
}

.button-primary:active {
  background: var(--color-primary-700);
}

.alert-success {
  background: var(--color-success);
  color: white;
}
```

---

## Responsive Design with Variables

### Breakpoint-Based Values

```css
:root {
  /* Mobile-first defaults */
  --spacing: 16px;
  --font-size-h1: 32px;
  --container-width: 100%;
}

@media (min-width: 768px) {
  :root {
    --spacing: 24px;
    --font-size-h1: 48px;
    --container-width: 720px;
  }
}

@media (min-width: 1024px) {
  :root {
    --spacing: 32px;
    --font-size-h1: 64px;
    --container-width: 960px;
  }
}

.container {
  width: var(--container-width);
  padding: var(--spacing);
}

h1 {
  font-size: var(--font-size-h1);
  margin-bottom: var(--spacing);
}
```

### Fluid Typography

```css
:root {
  /* Fluid scaling between min and max viewport widths */
  --fluid-min-width: 320;
  --fluid-max-width: 1200;

  --fluid-min-size: 16;
  --fluid-max-size: 20;

  --fluid-min-ratio: 1.2;
  --fluid-max-ratio: 1.333;
}

body {
  font-size: calc(
    (var(--fluid-min-size) * 1px) +
      (var(--fluid-max-size) - var(--fluid-min-size)) *
      (
        (100vw - (var(--fluid-min-width) * 1px)) /
          (var(--fluid-max-width) - var(--fluid-min-width))
      )
  );
}

h1 {
  font-size: calc(1em * var(--fluid-max-ratio) * var(--fluid-max-ratio));
}

h2 {
  font-size: calc(1em * var(--fluid-max-ratio));
}
```

---

## Component-Level Variables

### Reusable Button Component

```css
.button {
  /* Component-specific variables */
  --btn-bg: var(--color-primary-500);
  --btn-text: white;
  --btn-border: transparent;
  --btn-padding-x: 1rem;
  --btn-padding-y: 0.5rem;
  --btn-border-radius: 0.375rem;
  --btn-font-size: 1rem;
  --btn-font-weight: 500;

  /* Apply variables */
  background: var(--btn-bg);
  color: var(--btn-text);
  border: 1px solid var(--btn-border);
  padding: var(--btn-padding-y) var(--btn-padding-x);
  border-radius: var(--btn-border-radius);
  font-size: var(--btn-font-size);
  font-weight: var(--btn-font-weight);
  cursor: pointer;
  transition: all 0.2s;
}

/* Variants override only necessary variables */
.button-outline {
  --btn-bg: transparent;
  --btn-text: var(--color-primary-500);
  --btn-border: var(--color-primary-500);
}

.button-ghost {
  --btn-bg: transparent;
  --btn-text: var(--color-primary-500);
  --btn-border: transparent;
}

.button-lg {
  --btn-padding-x: 1.5rem;
  --btn-padding-y: 0.75rem;
  --btn-font-size: 1.125rem;
}

.button-sm {
  --btn-padding-x: 0.75rem;
  --btn-padding-y: 0.375rem;
  --btn-font-size: 0.875rem;
}
```

### Card Component

```css
.card {
  --card-bg: var(--bg-primary);
  --card-border: var(--border-color);
  --card-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  --card-padding: 1.5rem;
  --card-border-radius: 0.5rem;

  background: var(--card-bg);
  border: 1px solid var(--card-border);
  box-shadow: var(--card-shadow);
  padding: var(--card-padding);
  border-radius: var(--card-border-radius);
}

.card-elevated {
  --card-shadow:
    0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
}

.card-compact {
  --card-padding: 1rem;
}
```

---

## Real-World Pattern: Design Token System

```css
:root {
  /* Spacing scale */
  --space-0: 0;
  --space-1: 0.25rem; /* 4px */
  --space-2: 0.5rem; /* 8px */
  --space-3: 0.75rem; /* 12px */
  --space-4: 1rem; /* 16px */
  --space-5: 1.25rem; /* 20px */
  --space-6: 1.5rem; /* 24px */
  --space-8: 2rem; /* 32px */
  --space-10: 2.5rem; /* 40px */
  --space-12: 3rem; /* 48px */
  --space-16: 4rem; /* 64px */

  /* Typography scale */
  --font-xs: 0.75rem; /* 12px */
  --font-sm: 0.875rem; /* 14px */
  --font-base: 1rem; /* 16px */
  --font-lg: 1.125rem; /* 18px */
  --font-xl: 1.25rem; /* 20px */
  --font-2xl: 1.5rem; /* 24px */
  --font-3xl: 1.875rem; /* 30px */
  --font-4xl: 2.25rem; /* 36px */
  --font-5xl: 3rem; /* 48px */

  /* Font weights */
  --font-light: 300;
  --font-normal: 400;
  --font-medium: 500;
  --font-semibold: 600;
  --font-bold: 700;

  /* Border radius */
  --radius-sm: 0.125rem; /* 2px */
  --radius-md: 0.375rem; /* 6px */
  --radius-lg: 0.5rem; /* 8px */
  --radius-xl: 0.75rem; /* 12px */
  --radius-2xl: 1rem; /* 16px */
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  --shadow-xl: 0 20px 25px -5px rgba(0, 0, 0, 0.1);

  /* Transitions */
  --transition-fast: 150ms ease;
  --transition-base: 200ms ease;
  --transition-slow: 300ms ease;

  /* Z-index scale */
  --z-dropdown: 1000;
  --z-sticky: 1100;
  --z-fixed: 1200;
  --z-modal-backdrop: 1300;
  --z-modal: 1400;
  --z-popover: 1500;
  --z-tooltip: 1600;
}

/* Usage */
.modal {
  position: fixed;
  z-index: var(--z-modal);
  padding: var(--space-6);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-xl);
  transition: all var(--transition-base);
}

.heading-1 {
  font-size: var(--font-4xl);
  font-weight: var(--font-bold);
  margin-bottom: var(--space-6);
}
```

---

## Real-World Pattern: Dynamic Theming with JavaScript

```css
:root {
  --theme-hue: 220;
  --theme-saturation: 75%;

  --color-primary: hsl(var(--theme-hue), var(--theme-saturation), 50%);
  --color-primary-light: hsl(var(--theme-hue), var(--theme-saturation), 90%);
  --color-primary-dark: hsl(var(--theme-hue), var(--theme-saturation), 30%);
}

.button-primary {
  background: var(--color-primary);
  color: white;
}

.button-primary:hover {
  background: var(--color-primary-dark);
}

.alert-primary {
  background: var(--color-primary-light);
  border-left: 4px solid var(--color-primary);
}
```

```javascript
// Change theme color dynamically
function setThemeColor(hue) {
  document.documentElement.style.setProperty("--theme-hue", hue);
}

// Red theme
setThemeColor(0);

// Green theme
setThemeColor(140);

// Purple theme
setThemeColor(270);
```

---

## Real-World Pattern: Component State

```css
.input {
  --input-border-color: var(--border-color);
  --input-focus-color: var(--color-primary-500);
  --input-error-color: var(--color-error);

  border: 2px solid var(--input-border-color);
  transition: border-color var(--transition-base);
}

.input:focus {
  --input-border-color: var(--input-focus-color);
  outline: none;
}

.input[aria-invalid="true"] {
  --input-border-color: var(--input-error-color);
}

/* Disabled state */
.input:disabled {
  --input-border-color: var(--border-color);
  opacity: 0.5;
  cursor: not-allowed;
}
```

---

## Real-World Pattern: Animation Variables

```css
:root {
  --animation-duration: 300ms;
  --animation-timing: cubic-bezier(0.4, 0, 0.2, 1);
  --animation-delay: 0ms;
}

.fade-in {
  animation: fadeIn var(--animation-duration) var(--animation-timing)
    var(--animation-delay);
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Staggered animation */
.list-item:nth-child(1) {
  --animation-delay: 0ms;
}
.list-item:nth-child(2) {
  --animation-delay: 50ms;
}
.list-item:nth-child(3) {
  --animation-delay: 100ms;
}
.list-item:nth-child(4) {
  --animation-delay: 150ms;
}
```

---

## Advanced Techniques

### Calculated Values

```css
:root {
  --base-spacing: 8px;
  --spacing-multiplier: 2;
}

.container {
  /* Multiply variables */
  padding: calc(var(--base-spacing) * var(--spacing-multiplier)); /* 16px */

  /* Combine with other values */
  margin: calc(var(--base-spacing) * 3 + 10px); /* 34px */
}
```

### Nested Variables

```css
:root {
  --color-primary: #3b82f6;
  --button-bg: var(--color-primary);
  --button-hover-bg: var(--button-bg);
}

.button {
  background: var(--button-bg);
}

.button:hover {
  filter: brightness(0.9);
}
```

### CSS Grid with Variables

```css
:root {
  --grid-columns: 12;
  --grid-gap: 1rem;
}

.grid-container {
  display: grid;
  grid-template-columns: repeat(var(--grid-columns), 1fr);
  gap: var(--grid-gap);
}

.col-6 {
  grid-column: span 6;
}

@media (max-width: 768px) {
  :root {
    --grid-columns: 4;
    --grid-gap: 0.5rem;
  }
}
```

---

## Performance Considerations

### CSS Variables vs Sass Variables

```scss
// ❌ Sass variables - Compiled at build time
$primary-color: #3b82f6;

.button {
  background: $primary-color;
}

// Can't change at runtime!

// ✅ CSS variables - Dynamic at runtime
:root {
  --primary-color: #3b82f6;
}

.button {
  background: var(--primary-color);
}

// Can change with JavaScript!
document.documentElement.style.setProperty('--primary-color', '#ef4444');
```

### Inheritance and Specificity

```css
:root {
  --spacing: 16px;
}

.container {
  --spacing: 24px; /* Overrides :root */
}

.container .box {
  padding: var(--spacing); /* Uses 24px from .container */
}

.box {
  padding: var(--spacing); /* Uses 16px from :root */
}
```

---

## Browser DevTools

```css
/* Variables appear in DevTools */
:root {
  --debug-color: red;
}

.debug {
  outline: 2px solid var(--debug-color);
}

/* Change in DevTools to test different values */
```

---

## Key Takeaways

1. **Custom properties** are defined with `--` prefix and accessed with `var()`
2. **:root selector** defines global variables
3. **Scoping** allows variables to be overridden at any level
4. **Cascading** follows normal CSS inheritance rules
5. **Fallback values** provide defaults: `var(--color, blue)`
6. **JavaScript access** via `getPropertyValue()` and `setProperty()`
7. **Runtime changes** enable dynamic theming and state management
8. **calc() function** works with custom properties for computed values
9. **Media queries** can change variable values for responsive design
10. **Performance** is similar to regular CSS properties, better than JavaScript DOM manipulation
