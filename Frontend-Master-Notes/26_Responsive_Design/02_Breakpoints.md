# Strategic Breakpoints

> **Choose breakpoints based on content, not devices, for truly responsive designs**

---

## Core Concept

Breakpoints are the viewport widths where your responsive design changes layout. Strategic breakpoint selection is crucial for maintainability, performance, and user experience. Modern best practices favor content-driven breakpoints over device-specific ones.

**Key Principles:**

- **Content-first**: Let content dictate where breaks occur
- **Fewer is better**: 3-5 breakpoints typically sufficient
- **Use em/rem units**: Better for accessibility
- **Progressive enhancement**: Start mobile, add complexity upward

---

## Common Breakpoint Strategies

### **Device-Agnostic Breakpoints**

```css
/* ✅ GOOD: Content-driven, device-agnostic */
:root {
  --bp-xs: 20rem; /* 320px */
  --bp-sm: 30rem; /* 480px */
  --bp-md: 48rem; /* 768px */
  --bp-lg: 64rem; /* 1024px */
  --bp-xl: 80rem; /* 1280px */
  --bp-2xl: 96rem; /* 1536px */
}

/* Mobile first (default styles are for mobile) */
.container {
  padding: 1rem;
}

/* Small devices */
@media (min-width: 30rem) {
  .container {
    padding: 1.5rem;
  }
}

/* Medium devices */
@media (min-width: 48rem) {
  .container {
    padding: 2rem;
    max-width: 48rem;
    margin: 0 auto;
  }
}

/* Large devices */
@media (min-width: 64rem) {
  .container {
    max-width: 64rem;
    padding: 3rem;
  }
}

/* Extra large */
@media (min-width: 80rem) {
  .container {
    max-width: 75rem;
  }
}
```

### **❌ Avoid Device-Specific Breakpoints**

```css
/* ❌ BAD: Device-specific, brittle */
@media (min-width: 375px) {
  /* iPhone X */
}
@media (min-width: 414px) {
  /* iPhone Plus */
}
@media (min-width: 768px) {
  /* iPad */
}
@media (min-width: 1024px) {
  /* iPad Pro */
}
@media (min-width: 1366px) {
  /* Laptop */
}
@media (min-width: 1920px) {
  /* Desktop */
}
```

---

## Breakpoint Naming Conventions

### **Size-Based (T-shirt sizing)**

```typescript
// breakpoints.ts
export const breakpoints = {
  xs: "20em", // 320px
  sm: "30em", // 480px
  md: "48em", // 768px
  lg: "64em", // 1024px
  xl: "80em", // 1280px
  "2xl": "96em", // 1536px
} as const;

export type Breakpoint = keyof typeof breakpoints;

// Usage in styled-components
import styled from "styled-components";

const Container = styled.div`
  padding: 1rem;

  @media (min-width: ${breakpoints.md}) {
    padding: 2rem;
    max-width: 48rem;
    margin: 0 auto;
  }

  @media (min-width: ${breakpoints.lg}) {
    max-width: 64rem;
  }
`;
```

### **Semantic Naming**

```typescript
// breakpoints-semantic.ts
export const breakpoints = {
  mobile: "0",
  mobileLarge: "30em", // 480px
  tablet: "48em", // 768px
  desktop: "64em", // 1024px
  desktopLarge: "80em", // 1280px
  wide: "96em", // 1536px
} as const;
```

---

## Container Queries

### **Component-Level Breakpoints**

```css
/* Container queries: Respond to parent container, not viewport */
.card-container {
  container-type: inline-size;
  container-name: card;
}

.card {
  padding: 1rem;
  display: flex;
  flex-direction: column;
}

/* When container is > 400px wide */
@container card (min-width: 400px) {
  .card {
    flex-direction: row;
    gap: 1.5rem;
  }
}

/* When container is > 600px wide */
@container card (min-width: 600px) {
  .card {
    padding: 2rem;
  }
}
```

### **TypeScript Container Query Hook**

```typescript
// useContainerQuery.ts
import { useEffect, useState, useRef } from 'react';

interface ContainerQueryOptions {
  minWidth?: number;
  maxWidth?: number;
}

export function useContainerQuery(options: ContainerQueryOptions) {
  const [matches, setMatches] = useState(false);
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const resizeObserver = new ResizeObserver((entries) => {
      const entry = entries[0];
      const width = entry.contentRect.width;

      const minMatch = options.minWidth ? width >= options.minWidth : true;
      const maxMatch = options.maxWidth ? width <= options.maxWidth : true;

      setMatches(minMatch && maxMatch);
    });

    resizeObserver.observe(container);

    return () => resizeObserver.disconnect();
  }, [options.minWidth, options.maxWidth]);

  return { matches, containerRef };
}

// Usage
function Card() {
  const { matches, containerRef } = useContainerQuery({ minWidth: 400 });

  return (
    <div ref={containerRef} className="card">
      {matches ? (
        <div className="card-horizontal">Horizontal Layout</div>
      ) : (
        <div className="card-vertical">Vertical Layout</div>
      )}
    </div>
  );
}
```

---

## Responsive Grid System

### **CSS Grid with Breakpoints**

```css
.grid {
  display: grid;
  gap: 1rem;
  grid-template-columns: 1fr;
}

/* 2 columns on small screens */
@media (min-width: 30em) {
  .grid {
    grid-template-columns: repeat(2, 1fr);
    gap: 1.5rem;
  }
}

/* 3 columns on medium screens */
@media (min-width: 48em) {
  .grid {
    grid-template-columns: repeat(3, 1fr);
    gap: 2rem;
  }
}

/* 4 columns on large screens */
@media (min-width: 64em) {
  .grid {
    grid-template-columns: repeat(4, 1fr);
  }
}

/* Auto-fit: Responsive without media queries */
.grid-auto {
  display: grid;
  gap: 1rem;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
}
```

---

## TypeScript Breakpoint Utilities

### **Breakpoint Manager**

```typescript
// BreakpointManager.ts
export class BreakpointManager {
  private static breakpoints = {
    xs: 320,
    sm: 480,
    md: 768,
    lg: 1024,
    xl: 1280,
    "2xl": 1536,
  };

  static getBreakpoint(
    name: keyof typeof BreakpointManager.breakpoints,
  ): string {
    const px = this.breakpoints[name];
    return `${px / 16}em`; // Convert to em
  }

  static getCurrentBreakpoint(): string {
    const width = window.innerWidth;

    if (width < this.breakpoints.sm) return "xs";
    if (width < this.breakpoints.md) return "sm";
    if (width < this.breakpoints.lg) return "md";
    if (width < this.breakpoints.xl) return "lg";
    if (width < this.breakpoints["2xl"]) return "xl";
    return "2xl";
  }

  static matches(
    breakpoint: keyof typeof BreakpointManager.breakpoints,
  ): boolean {
    const px = this.breakpoints[breakpoint];
    return window.matchMedia(`(min-width: ${px}px)`).matches;
  }

  static between(
    min: keyof typeof BreakpointManager.breakpoints,
    max: keyof typeof BreakpointManager.breakpoints,
  ): boolean {
    const minPx = this.breakpoints[min];
    const maxPx = this.breakpoints[max];
    const width = window.innerWidth;
    return width >= minPx && width < maxPx;
  }
}

// Usage
if (BreakpointManager.matches("md")) {
  // Desktop layout
} else {
  // Mobile layout
}
```

### **React Breakpoint Hook**

```typescript
// useBreakpoint.ts
import { useEffect, useState } from 'react';

const breakpoints = {
  xs: 320,
  sm: 480,
  md: 768,
  lg: 1024,
  xl: 1280,
  '2xl': 1536
} as const;

type BreakpointKey = keyof typeof breakpoints;

export function useBreakpoint() {
  const [breakpoint, setBreakpoint] = useState<BreakpointKey>('xs');

  useEffect(() => {
    const getBreakpoint = (): BreakpointKey => {
      const width = window.innerWidth;

      if (width >= breakpoints['2xl']) return '2xl';
      if (width >= breakpoints.xl) return 'xl';
      if (width >= breakpoints.lg) return 'lg';
      if (width >= breakpoints.md) return 'md';
      if (width >= breakpoints.sm) return 'sm';
      return 'xs';
    };

    const handleResize = () => {
      setBreakpoint(getBreakpoint());
    };

    // Set initial breakpoint
    handleResize();

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return {
    breakpoint,
    isMobile: breakpoint === 'xs' || breakpoint === 'sm',
    isTablet: breakpoint === 'md',
    isDesktop: breakpoint === 'lg' || breakpoint === 'xl' || breakpoint === '2xl',
    isXs: breakpoint === 'xs',
    isSm: breakpoint === 'sm',
    isMd: breakpoint === 'md',
    isLg: breakpoint === 'lg',
    isXl: breakpoint === 'xl',
    is2Xl: breakpoint === '2xl'
  };
}

// Usage in component
function ResponsiveComponent() {
  const { isMobile, isDesktop, breakpoint } = useBreakpoint();

  return (
    <div>
      <p>Current breakpoint: {breakpoint}</p>
      {isMobile && <MobileNav />}
      {isDesktop && <DesktopNav />}
    </div>
  );
}
```

---

## Real-World Example: Responsive Dashboard

```typescript
// Dashboard.tsx
import { useBreakpoint } from './useBreakpoint';

function Dashboard() {
  const { isMobile, isTablet, isDesktop } = useBreakpoint();

  return (
    <div className="dashboard">
      {/* Sidebar: Hidden on mobile */}
      {!isMobile && <Sidebar />}

      <main className="dashboard-main">
        {/* Header with different layouts */}
        <header className="dashboard-header">
          {isMobile && <MenuButton />}
          <h1>Dashboard</h1>
          {isDesktop && <UserProfile />}
        </header>

        {/* Stats grid: Responsive columns */}
        <div className="stats-grid">
          <StatCard title="Users" value="1,234" />
          <StatCard title="Revenue" value="$45,678" />
          <StatCard title="Orders" value="890" />
          <StatCard title="Growth" value="+12%" />
        </div>

        {/* Charts: Stack on mobile */}
        <div className="charts-container">
          <Chart type="line" title="Revenue Over Time" />
          {!isMobile && <Chart type="bar" title="Top Products" />}
        </div>
      </main>
    </div>
  );
}
```

```css
/* dashboard.css */
.dashboard {
  display: flex;
  min-height: 100vh;
}

/* Mobile: No sidebar, full-width main */
.dashboard-main {
  flex: 1;
  padding: 1rem;
}

.dashboard-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}

/* Stats grid: 1 column on mobile */
.stats-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1rem;
  margin-bottom: 2rem;
}

/* Charts: Stack vertically on mobile */
.charts-container {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

/* Tablet: 2-column stats */
@media (min-width: 48em) {
  .stats-grid {
    grid-template-columns: repeat(2, 1fr);
    gap: 1.5rem;
  }
}

/* Desktop: Sidebar + 4-column stats */
@media (min-width: 64em) {
  .dashboard {
    padding-left: 250px; /* Sidebar width */
  }

  .stats-grid {
    grid-template-columns: repeat(4, 1fr);
    gap: 2rem;
  }

  .charts-container {
    flex-direction: row;
  }
}
```

---

## Best Practices

1. **Use em/rem units for breakpoints** - Better accessibility with browser zoom
2. **3-5 breakpoints are usually enough** - Don't over-complicate
3. **Content dictates breaks** - Not specific devices
4. **Test with real content** - Don't rely on lorem ipsum
5. **Avoid max-width queries** - Use min-width (mobile-first)
6. **Consider container queries** - For component-level responsiveness
7. **Use CSS custom properties** - Centralize breakpoint values
8. **Test between breakpoints** - Not just at exact widths
9. **Document breakpoint decisions** - Help future maintainers
10. **Use matchMedia in JavaScript** - Better performance than resize events

---

## Key Takeaways

1. **Content-first breakpoints** - Let your content dictate where layout changes occur
2. **Fewer breakpoints are better** - 3-5 strategic breakpoints typically sufficient
3. **Use em/rem units** - Better for accessibility with browser zoom
4. **Mobile-first approach** - Start small, enhance upward with min-width queries
5. **Container queries change the game** - Component responds to parent, not viewport
6. **Avoid device-specific breakpoints** - Too brittle, new devices constantly released
7. **Test real content at all widths** - Not just at exact breakpoint values
8. **matchMedia > resize events** - More performant for JavaScript breakpoint detection
9. **CSS custom properties for breakpoints** - Single source of truth, easier maintenance
10. **Document your breakpoint strategy** - Helps team understand decisions and maintain consistency

---

**Remember:** Breakpoints are about content, not devices. Choose them based on when your design needs to change, not what devices exist today.
