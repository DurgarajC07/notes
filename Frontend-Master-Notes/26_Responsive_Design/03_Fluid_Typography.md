# Fluid Typography

> **Scale typography smoothly across screen sizes using modern CSS techniques**

---

## Core Concept

Fluid typography scales smoothly between minimum and maximum font sizes based on viewport width, eliminating the need for multiple breakpoint-based font size declarations. The key is using `clamp()`, viewport units, and calc() to create responsive type scales.

**Benefits:**

- Smooth scaling without breakpoint jumps
- Better readability across all screen sizes
- Reduced CSS maintenance
- Improved accessibility with relative units

---

## The clamp() Function

### **Basic Syntax**

```css
/* clamp(MIN, PREFERRED, MAX) */
.heading {
  font-size: clamp(1.5rem, 5vw, 3rem);
  /* Scales from 24px to 48px based on viewport */
}

/* Breakdown:
   - MIN: 1.5rem (24px) - never smaller
   - PREFERRED: 5vw - scales with viewport
   - MAX: 3rem (48px) - never larger
*/
```

### **Fluid Type Scale**

```css
:root {
  /* Base font size */
  --font-size-base: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);

  /* Type scale using golden ratio (1.618) */
  --font-size-sm: clamp(0.875rem, 0.8rem + 0.375vw, 1rem);
  --font-size-md: var(--font-size-base);
  --font-size-lg: clamp(1.25rem, 1.1rem + 0.75vw, 1.875rem);
  --font-size-xl: clamp(1.5rem, 1.2rem + 1.5vw, 2.5rem);
  --font-size-2xl: clamp(2rem, 1.5rem + 2.5vw, 3.5rem);
  --font-size-3xl: clamp(2.5rem, 1.75rem + 3.75vw, 4.5rem);
}

body {
  font-size: var(--font-size-base);
}

h1 {
  font-size: var(--font-size-3xl);
}
h2 {
  font-size: var(--font-size-2xl);
}
h3 {
  font-size: var(--font-size-xl);
}
h4 {
  font-size: var(--font-size-lg);
}
p {
  font-size: var(--font-size-md);
}
small {
  font-size: var(--font-size-sm);
}
```

---

## Viewport Units

### **Understanding vw, vh, vmin, vmax**

```css
/* vw: 1% of viewport width */
.full-width-text {
  font-size: 5vw; /* 5% of viewport width */
}

/* vh: 1% of viewport height */
.hero-title {
  font-size: 10vh; /* 10% of viewport height */
}

/* vmin: 1% of smaller dimension (width or height) */
.square-text {
  font-size: 5vmin; /* Always relative to smaller dimension */
}

/* vmax: 1% of larger dimension */
.large-text {
  font-size: 8vmax;
}
```

### **Practical Viewport Unit Usage**

```css
/* Responsive heading without clamp() */
h1 {
  font-size: calc(1.5rem + 2vw);
  /* Starts at 24px and grows with viewport */
}

/* Combining units for better control */
.subtitle {
  font-size: calc(1rem + 0.5vw + 0.5vh);
  /* Scales with both width and height */
}
```

---

## calc() for Custom Scaling

### **Linear Interpolation Formula**

```css
/* Formula: calc(MIN + (MAX - MIN) * ((100vw - MIN_VP) / (MAX_VP - MIN_VP))) */

/* Scale from 16px at 320px viewport to 24px at 1200px viewport */
.body-text {
  font-size: calc(1rem + 8 * ((100vw - 320px) / 880));
  /* (24 - 16) = 8px difference
     (1200 - 320) = 880px viewport range */
}

/* More readable with CSS custom properties */
:root {
  --min-font: 1rem;
  --max-font: 1.5rem;
  --min-vw: 320px;
  --max-vw: 1200px;
}

.heading {
  font-size: calc(
    var(--min-font) + (var(--max-font) - var(--min-font)) *
      ((100vw - var(--min-vw)) / (var(--max-vw) - var(--min-vw)))
  );
}
```

---

## Modular Scale Ratios

### **Common Scale Ratios**

```css
:root {
  /* Minor Third (1.2) */
  --ratio-minor-third: 1.2;

  /* Major Third (1.25) */
  --ratio-major-third: 1.25;

  /* Perfect Fourth (1.333) */
  --ratio-perfect-fourth: 1.333;

  /* Golden Ratio (1.618) */
  --ratio-golden: 1.618;
}

/* Using golden ratio */
:root {
  --base-size: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);

  /* Multiply base by ratio for each level */
  --size-1: var(--base-size);
  --size-2: calc(var(--base-size) * 1.618);
  --size-3: calc(var(--base-size) * 1.618 * 1.618);
  --size-4: calc(var(--base-size) * 1.618 * 1.618 * 1.618);
  --size-5: calc(var(--base-size) * 1.618 * 1.618 * 1.618 * 1.618);
}

h1 {
  font-size: var(--size-5);
}
h2 {
  font-size: var(--size-4);
}
h3 {
  font-size: var(--size-3);
}
h4 {
  font-size: var(--size-2);
}
p {
  font-size: var(--size-1);
}
```

---

## Responsive Line Height

### **Fluid Line Height**

```css
/* Line height scales with font size */
.body-text {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);
  line-height: clamp(1.5, 1.4 + 0.2vw, 1.7);
  /* Unitless line-height is relative to font-size */
}

/* Different line heights for different contexts */
.heading {
  font-size: clamp(2rem, 1.5rem + 2.5vw, 3.5rem);
  line-height: 1.2; /* Tighter for headings */
}

.body {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);
  line-height: 1.6; /* More space for body text */
}

.caption {
  font-size: clamp(0.875rem, 0.8rem + 0.375vw, 1rem);
  line-height: 1.4;
}
```

---

## rem vs em Units

### **rem: Root-relative**

```css
/* rem is always relative to root font-size */
html {
  font-size: 16px; /* Base size */
}

.button {
  font-size: 1rem; /* 16px */
  padding: 0.5rem 1rem; /* 8px 16px */
}

/* Scales with root */
@media (min-width: 768px) {
  html {
    font-size: 18px; /* Everything scales up */
  }
  /* .button is now 18px with 9px 18px padding */
}
```

### **em: Parent-relative**

```css
/* em is relative to parent font-size */
.card {
  font-size: 1.25rem; /* 20px */
}

.card-title {
  font-size: 1.5em; /* 1.5 * 20px = 30px */
  margin-bottom: 0.5em; /* 0.5 * 30px = 15px */
}

.card-text {
  font-size: 1em; /* 1 * 20px = 20px */
  line-height: 1.6em; /* 1.6 * 20px = 32px */
}
```

### **When to Use Each**

```css
/* Use rem for: */
- Font sizes (consistent scaling)
- Spacing (padding, margin)
- Layout dimensions

/* Use em for: */
- Component-relative spacing
- Icon sizing relative to text
- Responsive padding/margin relative to parent
```

---

## TypeScript Fluid Type Utility

### **Fluid Type Calculator**

```typescript
// fluidType.ts
interface FluidTypeOptions {
  minFontSize: number; // in rem
  maxFontSize: number; // in rem
  minViewport?: number; // in px, default 320
  maxViewport?: number; // in px, default 1200
}

export function generateFluidType({
  minFontSize,
  maxFontSize,
  minViewport = 320,
  maxViewport = 1200,
}: FluidTypeOptions): string {
  const minVw = minViewport / 100;
  const maxVw = maxViewport / 100;

  // Calculate preferred value
  const slope = (maxFontSize - minFontSize) / (maxVw - minVw);
  const intercept = minFontSize - slope * minVw;

  return `clamp(${minFontSize}rem, ${intercept.toFixed(4)}rem + ${slope.toFixed(4)}vw, ${maxFontSize}rem)`;
}

// Usage
const h1Size = generateFluidType({
  minFontSize: 2,
  maxFontSize: 4,
});
// Returns: "clamp(2rem, 0.5455rem + 4.5455vw, 4rem)"
```

### **React Fluid Typography Component**

```typescript
// FluidText.tsx
import React, { CSSProperties } from 'react';

interface FluidTextProps {
  as?: keyof JSX.IntrinsicElements;
  minSize: number;
  maxSize: number;
  children: React.ReactNode;
  className?: string;
}

export function FluidText({
  as: Component = 'p',
  minSize,
  maxSize,
  children,
  className
}: FluidTextProps) {
  const style: CSSProperties = {
    fontSize: `clamp(${minSize}rem, ${minSize}rem + ${maxSize - minSize}vw, ${maxSize}rem)`
  };

  return (
    <Component className={className} style={style}>
      {children}
    </Component>
  );
}

// Usage
<FluidText as="h1" minSize={2} maxSize={4}>
  Responsive Heading
</FluidText>
```

---

## Real-World Example: Blog Layout

```css
/* blog.css */
:root {
  /* Fluid type scale */
  --font-size-xs: clamp(0.75rem, 0.7rem + 0.25vw, 0.875rem);
  --font-size-sm: clamp(0.875rem, 0.8rem + 0.375vw, 1rem);
  --font-size-base: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);
  --font-size-lg: clamp(1.125rem, 1rem + 0.625vw, 1.375rem);
  --font-size-xl: clamp(1.375rem, 1.2rem + 0.875vw, 1.875rem);
  --font-size-2xl: clamp(1.75rem, 1.4rem + 1.75vw, 2.75rem);
  --font-size-3xl: clamp(2.25rem, 1.6rem + 3.25vw, 4rem);

  /* Fluid line heights */
  --line-height-tight: 1.2;
  --line-height-normal: 1.6;
  --line-height-relaxed: 1.8;

  /* Fluid spacing */
  --space-xs: clamp(0.5rem, 0.4rem + 0.5vw, 0.75rem);
  --space-sm: clamp(0.75rem, 0.6rem + 0.75vw, 1.25rem);
  --space-md: clamp(1rem, 0.8rem + 1vw, 1.75rem);
  --space-lg: clamp(1.5rem, 1.2rem + 1.5vw, 2.5rem);
  --space-xl: clamp(2rem, 1.5rem + 2.5vw, 4rem);
}

.blog-post {
  max-width: 65ch; /* Optimal line length */
  margin: 0 auto;
  padding: var(--space-md);
}

.blog-title {
  font-size: var(--font-size-3xl);
  line-height: var(--line-height-tight);
  margin-bottom: var(--space-md);
}

.blog-meta {
  font-size: var(--font-size-sm);
  color: #666;
  margin-bottom: var(--space-lg);
}

.blog-content p {
  font-size: var(--font-size-base);
  line-height: var(--line-height-relaxed);
  margin-bottom: var(--space-md);
}

.blog-content h2 {
  font-size: var(--font-size-2xl);
  line-height: var(--line-height-tight);
  margin-top: var(--space-xl);
  margin-bottom: var(--space-sm);
}

.blog-content h3 {
  font-size: var(--font-size-xl);
  line-height: var(--line-height-normal);
  margin-top: var(--space-lg);
  margin-bottom: var(--space-xs);
}

.blog-content code {
  font-size: 0.9em; /* Relative to parent */
  padding: 0.2em 0.4em;
  background: #f5f5f5;
  border-radius: 3px;
}

.blog-content blockquote {
  font-size: var(--font-size-lg);
  line-height: var(--line-height-relaxed);
  font-style: italic;
  padding-left: var(--space-md);
  border-left: 4px solid #ddd;
  margin: var(--space-lg) 0;
}
```

---

## Best Practices

1. **Use clamp() for fluid scaling** - Modern, clean syntax
2. **Maintain readable line length** - 45-75 characters (65ch ideal)
3. **Use unitless line-height** - Scales proportionally
4. **rem for consistency** - em for component-relative sizing
5. **Test with actual content** - Not lorem ipsum
6. **Respect user preferences** - Don't disable zoom
7. **Minimum 16px body text** - Smaller is hard to read
8. **Use CSS custom properties** - Centralized type system
9. **Test with browser zoom** - Should scale properly
10. **Consider performance** - clamp() is well-supported and fast

---

## Key Takeaways

1. **clamp() is the modern solution** - Clean syntax, smooth scaling, no breakpoints needed
2. **Viewport units enable fluid scaling** - But always constrain with min/max
3. **Modular scale creates harmony** - Use consistent ratios (1.25, 1.333, 1.618)
4. **rem vs em: Different purposes** - rem for consistency, em for relative sizing
5. **Line height matters** - 1.5-1.8 for body text, 1.1-1.3 for headings
6. **65 characters per line is optimal** - Use ch unit for max-width
7. **Fluid spacing complements fluid type** - Use same techniques for margins/padding
8. **Respect user preferences** - Never disable browser zoom
9. **Performance is excellent** - Modern CSS properties are highly optimized
10. **Test across viewports** - Especially at extreme sizes (320px and 2560px+)

---

**Remember:** Fluid typography isn't just about scaling font size—it's about creating a harmonious reading experience across all devices.
