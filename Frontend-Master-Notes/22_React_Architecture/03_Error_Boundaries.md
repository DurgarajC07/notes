# React Error Boundaries

## Core Concept

Error Boundaries are React components that catch JavaScript errors anywhere in their child component tree, log those errors, and display a fallback UI. They're essential for preventing entire application crashes and providing graceful error handling in production.

---

## Basic Error Boundary

### Class Component Implementation

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
  errorInfo: ErrorInfo | null;
}

class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorInfo: null
    };
  }

  static getDerivedStateFromError(error: Error): Partial<State> {
    // Update state so next render shows fallback UI
    return {
      hasError: true,
      error
    };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Log error to error reporting service
    console.error('Error caught by boundary:', error, errorInfo);

    this.setState({
      error,
      errorInfo
    });

    // Send to error tracking service (Sentry, LogRocket, etc.)
    // logErrorToService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // Custom fallback UI
      return this.props.fallback || (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <details style={{ whiteSpace: 'pre-wrap' }}>
            {this.state.error && this.state.error.toString()}
            <br />
            {this.state.errorInfo?.componentStack}
          </details>
        </div>
      );
    }

    return this.props.children;
  }
}

export default ErrorBoundary;
```

### Usage

```typescript
function App() {
  return (
    <ErrorBoundary fallback={<div>Error occurred!</div>}>
      <UserProfile />
      <Dashboard />
    </ErrorBoundary>
  );
}
```

---

## Advanced Error Boundary with Reset

### With Recovery Mechanism

```typescript
interface Props {
  children: ReactNode;
  fallback?: (error: Error, reset: () => void) => ReactNode;
  onReset?: () => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundaryWithReset extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Error:', error, errorInfo);
  }

  reset = () => {
    this.props.onReset?.();
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError && this.state.error) {
      return this.props.fallback?.(this.state.error, this.reset) || (
        <div>
          <h2>Error occurred</h2>
          <p>{this.state.error.message}</p>
          <button onClick={this.reset}>Try Again</button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
function App() {
  const [key, setKey] = React.useState(0);

  return (
    <ErrorBoundaryWithReset
      key={key} // Remount on key change
      onReset={() => setKey(k => k + 1)}
      fallback={(error, reset) => (
        <div>
          <h2>Oops!</h2>
          <p>{error.message}</p>
          <button onClick={reset}>Retry</button>
        </div>
      )}
    >
      <ProblematicComponent />
    </ErrorBoundaryWithReset>
  );
}
```

---

## Granular Error Boundaries

### Multiple Boundaries for Isolated Errors

```typescript
function App() {
  return (
    <div>
      {/* Isolated error boundary for sidebar */}
      <ErrorBoundary fallback={<SidebarFallback />}>
        <Sidebar />
      </ErrorBoundary>

      {/* Main content has its own boundary */}
      <ErrorBoundary fallback={<MainContentFallback />}>
        <MainContent />
      </ErrorBoundary>

      {/* Footer isolated */}
      <ErrorBoundary fallback={<FooterFallback />}>
        <Footer />
      </ErrorBoundary>
    </div>
  );
}

// If Sidebar crashes, main content still works
```

---

## Error Boundary with Logging

### Integration with Error Tracking Services

```typescript
import * as Sentry from '@sentry/react';

interface Props {
  children: ReactNode;
}

interface State {
  hasError: boolean;
  eventId: string | null;
}

class SentryErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, eventId: null };
  }

  static getDerivedStateFromError(): Partial<State> {
    return { hasError: true };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    Sentry.withScope((scope) => {
      scope.setExtras(errorInfo);
      const eventId = Sentry.captureException(error);
      this.setState({ eventId });
    });
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong</h2>
          <button
            onClick={() => {
              if (this.state.eventId) {
                Sentry.showReportDialog({ eventId: this.state.eventId });
              }
            }}
          >
            Report feedback
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

---

## React Hook Error Boundary (Experimental)

### Using react-error-boundary Library

```typescript
import { ErrorBoundary, FallbackProps } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div role="alert">
      <h2>Something went wrong:</h2>
      <pre style={{ color: 'red' }}>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

function App() {
  const [queryKey, setQueryKey] = React.useState(0);

  return (
    <ErrorBoundary
      FallbackComponent={ErrorFallback}
      onReset={() => setQueryKey(k => k + 1)}
      resetKeys={[queryKey]}
    >
      <DataFetchingComponent key={queryKey} />
    </ErrorBoundary>
  );
}
```

### With useErrorHandler Hook

```typescript
import { useErrorHandler } from 'react-error-boundary';

function DataComponent() {
  const handleError = useErrorHandler();

  React.useEffect(() => {
    fetchData()
      .then(setData)
      .catch(handleError); // Throws to nearest error boundary
  }, [handleError]);

  return <div>{/* render data */}</div>;
}
```

---

## What Error Boundaries Catch

### Caught Errors

```typescript
// ✅ Render errors
function BrokenComponent() {
  const user = null;
  return <div>{user.name}</div>; // TypeError - Caught!
}

// ✅ Lifecycle method errors
class BrokenLifecycle extends Component {
  componentDidMount() {
    throw new Error('Mount error'); // Caught!
  }
  render() {
    return <div>Content</div>;
  }
}

// ✅ Constructor errors
class BrokenConstructor extends Component {
  constructor(props: any) {
    super(props);
    throw new Error('Constructor error'); // Caught!
  }
  render() {
    return <div>Content</div>;
  }
}
```

### Uncaught Errors

```typescript
// ❌ Event handler errors - NOT caught
function NotCaughtComponent() {
  const handleClick = () => {
    throw new Error('Click error'); // Not caught by error boundary!
  };

  return <button onClick={handleClick}>Click</button>;
}

// Solution: Use try-catch
function FixedComponent() {
  const handleClick = () => {
    try {
      riskyOperation();
    } catch (error) {
      // Handle or rethrow to error boundary
      logError(error);
    }
  };

  return <button onClick={handleClick}>Click</button>;
}

// ❌ Async errors - NOT caught
function AsyncComponent() {
  React.useEffect(() => {
    setTimeout(() => {
      throw new Error('Async error'); // Not caught!
    }, 1000);
  }, []);

  return <div>Content</div>;
}

// Solution: Catch and throw in render
function FixedAsyncComponent() {
  const [error, setError] = React.useState<Error | null>(null);

  React.useEffect(() => {
    fetchData().catch(setError);
  }, []);

  if (error) throw error; // Now error boundary catches it

  return <div>Content</div>;
}

// ❌ Server-side rendering errors
// Error boundaries don't work in SSR

// ❌ Errors in error boundary itself
// Parent error boundary must catch
```

---

## Real-World Pattern: Async Error Handling

### With Suspense and Error Boundaries

```typescript
interface Resource<T> {
  read(): T;
}

function wrapPromise<T>(promise: Promise<T>): Resource<T> {
  let status = 'pending';
  let result: T;
  let error: Error;

  const suspender = promise.then(
    (r) => {
      status = 'success';
      result = r;
    },
    (e) => {
      status = 'error';
      error = e;
    }
  );

  return {
    read() {
      if (status === 'pending') throw suspender;
      if (status === 'error') throw error;
      return result;
    }
  };
}

// Usage
const userResource = wrapPromise(fetchUser());

function UserProfile() {
  const user = userResource.read(); // Throws if pending or error
  return <div>{user.name}</div>;
}

function App() {
  return (
    <ErrorBoundary fallback={<div>Failed to load user</div>}>
      <React.Suspense fallback={<div>Loading...</div>}>
        <UserProfile />
      </React.Suspense>
    </ErrorBoundary>
  );
}
```

---

## Real-World Pattern: Route-Level Error Boundaries

### With React Router

```typescript
import { useRouteError, isRouteErrorResponse } from 'react-router-dom';

function RouteErrorBoundary() {
  const error = useRouteError();

  if (isRouteErrorResponse(error)) {
    if (error.status === 404) {
      return <div>Page not found</div>;
    }

    if (error.status === 401) {
      return <div>Unauthorized</div>;
    }

    if (error.status === 503) {
      return <div>Service unavailable</div>;
    }
  }

  return <div>Something went wrong</div>;
}

// Router configuration
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    errorElement: <RouteErrorBoundary />,
    children: [
      {
        path: 'dashboard',
        element: <Dashboard />,
        errorElement: <DashboardError />
      }
    ]
  }
]);
```

---

## Real-World Pattern: Error Monitoring

### Complete Error Tracking Setup

```typescript
interface ErrorLog {
  message: string;
  stack?: string;
  componentStack?: string;
  timestamp: number;
  userAgent: string;
  url: string;
}

class MonitoredErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    const errorLog: ErrorLog = {
      message: error.message,
      stack: error.stack,
      componentStack: errorInfo.componentStack,
      timestamp: Date.now(),
      userAgent: navigator.userAgent,
      url: window.location.href
    };

    // Send to multiple services
    this.logToSentry(error, errorInfo);
    this.logToAnalytics(errorLog);
    this.logToServer(errorLog);
  }

  logToSentry(error: Error, errorInfo: ErrorInfo) {
    Sentry.withScope((scope) => {
      scope.setExtras(errorInfo);
      Sentry.captureException(error);
    });
  }

  logToAnalytics(errorLog: ErrorLog) {
    analytics.track('error_occurred', errorLog);
  }

  async logToServer(errorLog: ErrorLog) {
    try {
      await fetch('/api/errors', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(errorLog)
      });
    } catch (err) {
      console.error('Failed to log error:', err);
    }
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallbackUI error={this.state.error} />;
    }
    return this.props.children;
  }
}
```

---

## Testing Error Boundaries

### Jest Testing

```typescript
import { render, screen } from '@testing-library/react';

// Suppress console.error in tests
beforeAll(() => {
  jest.spyOn(console, 'error').mockImplementation(() => {});
});

afterAll(() => {
  (console.error as jest.Mock).mockRestore();
});

const ThrowError = () => {
  throw new Error('Test error');
};

test('catches errors and displays fallback', () => {
  render(
    <ErrorBoundary fallback={<div>Error occurred</div>}>
      <ThrowError />
    </ErrorBoundary>
  );

  expect(screen.getByText('Error occurred')).toBeInTheDocument();
});

test('logs error to service', () => {
  const logSpy = jest.spyOn(console, 'log');

  render(
    <ErrorBoundary>
      <ThrowError />
    </ErrorBoundary>
  );

  expect(logSpy).toHaveBeenCalledWith(expect.stringContaining('Test error'));
});
```

---

## Best Practices

### 1. Strategic Placement

```typescript
// ✅ Good - Multiple boundaries
<ErrorBoundary name="app">
  <Header />
  <ErrorBoundary name="sidebar">
    <Sidebar />
  </ErrorBoundary>
  <ErrorBoundary name="main">
    <MainContent />
  </ErrorBoundary>
</ErrorBoundary>

// ❌ Bad - Single boundary
<ErrorBoundary>
  <EntireApp />
</ErrorBoundary>
```

### 2. Meaningful Fallback UI

```typescript
// ✅ Good - Contextual fallback
<ErrorBoundary
  fallback={<EmptyDashboard message="Unable to load dashboard data" />}
>
  <Dashboard />
</ErrorBoundary>

// ❌ Bad - Generic error
<ErrorBoundary fallback={<div>Error</div>}>
  <Dashboard />
</ErrorBoundary>
```

### 3. Development vs Production

```typescript
function ErrorFallback({ error }: { error: Error }) {
  if (process.env.NODE_ENV === 'development') {
    return (
      <div>
        <h2>Development Error</h2>
        <pre>{error.stack}</pre>
      </div>
    );
  }

  return (
    <div>
      <h2>Something went wrong</h2>
      <p>We're working on it!</p>
    </div>
  );
}
```

---

## Key Takeaways

1. **Error boundaries** catch errors in render, lifecycle, and constructors
2. **Don't catch** event handlers, async code, or SSR errors
3. Use **multiple boundaries** for isolated error handling
4. Provide **meaningful fallback UI** with recovery options
5. **Log errors** to monitoring services (Sentry, LogRocket)
6. Use **resetKeys** or key prop to remount after error
7. Combine with **Suspense** for async error handling
8. **Test error boundaries** thoroughly
9. **Different UI** for development vs production
10. Consider **react-error-boundary** library for hooks-based API
