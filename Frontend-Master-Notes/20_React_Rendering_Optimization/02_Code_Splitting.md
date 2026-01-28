# Code Splitting in React

## Core Concept

Code splitting breaks your bundle into smaller chunks that are loaded on-demand, reducing initial load time and improving performance.

---

## React.lazy

Load components dynamically:

```typescript
import { lazy, Suspense } from 'react';

// ❌ Static import - loads immediately
import HeavyComponent from './HeavyComponent';

// ✅ Dynamic import - loads when needed
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}
```

---

## Route-Based Splitting

Most common pattern - split by routes:

```typescript
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

---

## Named Exports

Can't use named exports directly with React.lazy:

```typescript
// ❌ Doesn't work
const { Button } = lazy(() => import("./components"));

// ✅ Re-export as default
// components/Button.tsx
export { Button as default } from "./Button";

// App.tsx
const Button = lazy(() => import("./components/Button"));

// Or use intermediate module
const Button = lazy(() =>
  import("./components").then((module) => ({ default: module.Button })),
);
```

---

## Error Boundaries

Handle loading errors gracefully:

```typescript
import { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
}

interface State {
  hasError: boolean;
}

class ErrorBoundary extends Component<Props, State> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('Error loading component:', error, info);
  }

  render() {
    if (this.state.hasError) {
      return <div>Failed to load component. Please refresh.</div>;
    }
    return this.props.children;
  }
}

// Usage
function App() {
  return (
    <ErrorBoundary>
      <Suspense fallback={<Loading />}>
        <LazyComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## Preloading

Load components before they're needed:

```typescript
const HeavyComponent = lazy(() => import('./HeavyComponent'));

// Preload on hover
function Menu() {
  const preloadHeavy = () => {
    import('./HeavyComponent');
  };

  return (
    <nav>
      <Link to="/heavy" onMouseEnter={preloadHeavy}>
        Heavy Page
      </Link>
    </nav>
  );
}

// Preload after initial render
useEffect(() => {
  const timer = setTimeout(() => {
    import('./HeavyComponent');
  }, 2000);

  return () => clearTimeout(timer);
}, []);
```

---

## Dynamic Imports

Import based on runtime conditions:

```typescript
function Editor() {
  const [mode, setMode] = useState<'simple' | 'advanced'>('simple');

  const EditorComponent = lazy(() => {
    if (mode === 'advanced') {
      return import('./AdvancedEditor');
    }
    return import('./SimpleEditor');
  });

  return (
    <div>
      <button onClick={() => setMode('advanced')}>
        Switch to Advanced
      </button>
      <Suspense fallback={<Loading />}>
        <EditorComponent />
      </Suspense>
    </div>
  );
}
```

---

## Webpack Magic Comments

Control how chunks are loaded:

```typescript
// Name the chunk
const Dashboard = lazy(
  () => import(/* webpackChunkName: "dashboard" */ "./Dashboard"),
);

// Prefetch (load during idle time)
const Settings = lazy(() => import(/* webpackPrefetch: true */ "./Settings"));

// Preload (load in parallel with parent)
const Profile = lazy(() => import(/* webpackPreload: true */ "./Profile"));
```

---

## Component-Level Splitting

Split heavy components:

```typescript
function ProductPage() {
  const [showReviews, setShowReviews] = useState(false);

  const Reviews = lazy(() => import('./Reviews'));

  return (
    <div>
      <ProductDetails />
      <button onClick={() => setShowReviews(true)}>
        Show Reviews
      </button>
      {showReviews && (
        <Suspense fallback={<Spinner />}>
          <Reviews />
        </Suspense>
      )}
    </div>
  );
}
```

---

## Library Code Splitting

Split large libraries:

```typescript
// Heavy libraries can be split too
const Chart = lazy(() =>
  import("react-chartjs-2").then((module) => ({
    default: module.Line,
  })),
);

// PDF viewer
const PDFViewer = lazy(() => import("react-pdf"));

// Rich text editor
const Editor = lazy(() => import("@tinymce/tinymce-react"));
```

---

## Webpack Bundle Analyzer

Visualize bundle size:

```bash
npm install --save-dev webpack-bundle-analyzer
```

```javascript
// webpack.config.js
const BundleAnalyzerPlugin =
  require("webpack-bundle-analyzer").BundleAnalyzerPlugin;

module.exports = {
  plugins: [new BundleAnalyzerPlugin()],
};
```

---

## Real-World Pattern: Tab System

```typescript
function TabContainer() {
  const [activeTab, setActiveTab] = useState('overview');

  const TabContent = lazy(() => {
    switch (activeTab) {
      case 'overview':
        return import('./tabs/Overview');
      case 'analytics':
        return import('./tabs/Analytics');
      case 'settings':
        return import('./tabs/Settings');
      default:
        return import('./tabs/Overview');
    }
  });

  return (
    <div>
      <Tabs active={activeTab} onChange={setActiveTab} />
      <Suspense fallback={<TabLoader />}>
        <TabContent />
      </Suspense>
    </div>
  );
}
```

---

## Best Practices

✅ **Split by routes** - most effective strategy  
✅ **Use Suspense boundaries** near split points  
✅ **Preload on hover** for better UX  
✅ **Name chunks** for debugging  
✅ **Split large libraries** separately  
✅ **Add error boundaries** around lazy components  
❌ **Don't over-split** - too many small chunks hurt  
❌ **Don't split critical path** - core UI should load fast

---

## Performance Metrics

Track impact of code splitting:

```typescript
// Measure chunk load time
const startTime = performance.now();

const Component = lazy(() =>
  import("./Component").then((module) => {
    console.log(`Loaded in ${performance.now() - startTime}ms`);
    return module;
  }),
);
```

---

## Key Takeaways

1. **Code splitting reduces initial bundle** size
2. **React.lazy + Suspense** for component splitting
3. **Route-based splitting** is most common
4. **Preload on hover** improves perceived performance
5. **Error boundaries** handle loading failures
6. **Webpack magic comments** control loading strategy
7. **Don't over-split** - balance is key
