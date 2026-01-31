# Mocking in React Testing

## Core Concept

Mocking replaces **real dependencies with controlled test doubles**, enabling isolated testing, predictable behavior, and faster test execution without external dependencies.

---

## Types of Test Doubles

```typescript
// Dummy - Placeholder with no behavior
const dummyLogger = { log: () => {} };

// Stub - Returns predefined data
const stubUserService = {
  getUser: () => ({ id: "1", name: "John" }),
};

// Spy - Records how it was called
const spyLogger = jest.fn();

// Mock - Pre-programmed with expectations
const mockUserService = {
  getUser: jest.fn().mockResolvedValue({ id: "1", name: "John" }),
};

// Fake - Working implementation (simplified)
class FakeUserService {
  private users = new Map();

  getUser(id: string) {
    return this.users.get(id);
  }

  addUser(user: User) {
    this.users.set(user.id, user);
  }
}
```

---

## Jest Function Mocks

### **Basic Function Mocking**

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  test('calls onClick when clicked', async () => {
    const user = userEvent.setup();
    const handleClick = jest.fn();

    render(<Button onClick={handleClick}>Click me</Button>);

    await user.click(screen.getByRole('button'));

    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  test('passes event to onClick', async () => {
    const user = userEvent.setup();
    const handleClick = jest.fn();

    render(<Button onClick={handleClick}>Click me</Button>);

    await user.click(screen.getByRole('button'));

    expect(handleClick).toHaveBeenCalledWith(
      expect.objectContaining({
        type: 'click'
      })
    );
  });

  test('does not call onClick when disabled', async () => {
    const user = userEvent.setup();
    const handleClick = jest.fn();

    render(<Button onClick={handleClick} disabled>Click me</Button>);

    await user.click(screen.getByRole('button'));

    expect(handleClick).not.toHaveBeenCalled();
  });
});
```

### **Mock Return Values**

```typescript
describe("Mock Return Values", () => {
  test("mock return value", () => {
    const mockFn = jest.fn();
    mockFn.mockReturnValue(42);

    expect(mockFn()).toBe(42);
    expect(mockFn()).toBe(42); // Always returns 42
  });

  test("mock return value once", () => {
    const mockFn = jest.fn();
    mockFn.mockReturnValueOnce(1).mockReturnValueOnce(2).mockReturnValue(3);

    expect(mockFn()).toBe(1); // First call
    expect(mockFn()).toBe(2); // Second call
    expect(mockFn()).toBe(3); // Third call
    expect(mockFn()).toBe(3); // All subsequent calls
  });

  test("mock resolved value (Promise)", async () => {
    const mockFn = jest.fn();
    mockFn.mockResolvedValue({ id: "1", name: "John" });

    const result = await mockFn();
    expect(result).toEqual({ id: "1", name: "John" });
  });

  test("mock rejected value", async () => {
    const mockFn = jest.fn();
    mockFn.mockRejectedValue(new Error("API Error"));

    await expect(mockFn()).rejects.toThrow("API Error");
  });

  test("mock implementation", () => {
    const mockFn = jest.fn((a: number, b: number) => a + b);

    expect(mockFn(2, 3)).toBe(5);
    expect(mockFn).toHaveBeenCalledWith(2, 3);
  });
});
```

### **Spy on Function Calls**

```typescript
describe("Function Spy", () => {
  test("tracks function calls", () => {
    const mockFn = jest.fn();

    mockFn("arg1", "arg2");
    mockFn("arg3");

    // Check call count
    expect(mockFn).toHaveBeenCalledTimes(2);

    // Check called with specific args
    expect(mockFn).toHaveBeenCalledWith("arg1", "arg2");
    expect(mockFn).toHaveBeenLastCalledWith("arg3");

    // Check all calls
    expect(mockFn.mock.calls).toEqual([["arg1", "arg2"], ["arg3"]]);
  });

  test("tracks return values", () => {
    const mockFn = jest.fn((x: number) => x * 2);

    mockFn(5);
    mockFn(10);

    expect(mockFn.mock.results).toEqual([
      { type: "return", value: 10 },
      { type: "return", value: 20 },
    ]);
  });
});
```

---

## Module Mocking

### **Mock Entire Module**

```typescript
// utils/api.ts
export const fetchUser = async (id: string) => {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
};

// UserProfile.test.tsx
import { render, screen } from '@testing-library/react';
import { UserProfile } from './UserProfile';
import * as api from './utils/api';

// Mock entire module
jest.mock('./utils/api');

describe('UserProfile', () => {
  test('displays user data', async () => {
    // Type-safe mock
    const mockFetchUser = api.fetchUser as jest.MockedFunction<typeof api.fetchUser>;

    mockFetchUser.mockResolvedValue({
      id: '1',
      name: 'John Doe',
      email: 'john@example.com'
    });

    render(<UserProfile userId="1" />);

    expect(await screen.findByText('John Doe')).toBeInTheDocument();
    expect(await screen.findByText('john@example.com')).toBeInTheDocument();

    expect(mockFetchUser).toHaveBeenCalledWith('1');
  });
});
```

### **Mock Specific Functions**

```typescript
// utils/helpers.ts
export const calculateTotal = (items: Item[]) => { /* ... */ };
export const formatCurrency = (amount: number) => { /* ... */ };

// Cart.test.tsx
import { render, screen } from '@testing-library/react';
import { Cart } from './Cart';
import * as helpers from './utils/helpers';

describe('Cart', () => {
  test('displays formatted total', () => {
    // Mock only specific function
    jest.spyOn(helpers, 'formatCurrency').mockReturnValue('$99.99');

    render(<Cart items={items} />);

    expect(screen.getByText('$99.99')).toBeInTheDocument();
    expect(helpers.formatCurrency).toHaveBeenCalledWith(99.99);
  });
});
```

### **Partial Module Mock**

```typescript
// Mock only specific exports
jest.mock("./utils/api", () => ({
  ...jest.requireActual("./utils/api"), // Keep other exports
  fetchUser: jest.fn(), // Mock only this one
}));
```

---

## Mock Service Worker (MSW)

### **Setup and Basic Usage**

```typescript
// mocks/handlers.ts
import { rest } from "msw";

export const handlers = [
  // Mock GET request
  rest.get("/api/users/:id", (req, res, ctx) => {
    const { id } = req.params;

    return res(
      ctx.status(200),
      ctx.json({
        id,
        name: "John Doe",
        email: "john@example.com",
      }),
    );
  }),

  // Mock POST request
  rest.post("/api/users", (req, res, ctx) => {
    return res(
      ctx.status(201),
      ctx.json({
        id: "123",
        name: req.body.name,
      }),
    );
  }),

  // Mock error response
  rest.get("/api/error", (req, res, ctx) => {
    return res(ctx.status(500), ctx.json({ error: "Internal server error" }));
  }),
];

// mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);

// setupTests.ts
import { server } from "./mocks/server";

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### **MSW in Tests**

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import { rest } from 'msw';
import { server } from './mocks/server';
import { UserProfile } from './UserProfile';

describe('UserProfile with MSW', () => {
  test('fetches and displays user', async () => {
    render(<UserProfile userId="1" />);

    expect(await screen.findByText('John Doe')).toBeInTheDocument();
  });

  test('handles error response', async () => {
    // Override handler for this test
    server.use(
      rest.get('/api/users/:id', (req, res, ctx) => {
        return res(
          ctx.status(404),
          ctx.json({ error: 'User not found' })
        );
      })
    );

    render(<UserProfile userId="999" />);

    expect(await screen.findByText(/error/i)).toBeInTheDocument();
  });

  test('handles network error', async () => {
    server.use(
      rest.get('/api/users/:id', (req, res, ctx) => {
        return res.networkError('Network connection failed');
      })
    );

    render(<UserProfile userId="1" />);

    expect(await screen.findByText(/network error/i)).toBeInTheDocument();
  });

  test('simulates slow response', async () => {
    server.use(
      rest.get('/api/users/:id', async (req, res, ctx) => {
        await ctx.delay(2000); // 2 second delay
        return res(ctx.json({ id: '1', name: 'John' }));
      })
    );

    render(<UserProfile userId="1" />);

    // Loading indicator should show
    expect(screen.getByText(/loading/i)).toBeInTheDocument();

    // Then data appears
    expect(await screen.findByText('John', {}, { timeout: 3000 })).toBeInTheDocument();
  });
});
```

---

## Mocking React Hooks

### **Mock useState**

```typescript
import React from 'react';
import { render, screen } from '@testing-library/react';
import { Counter } from './Counter';

describe('Counter', () => {
  test('mocks useState', () => {
    const mockSetCount = jest.fn();

    jest.spyOn(React, 'useState').mockImplementation(() => [5, mockSetCount]);

    render(<Counter />);

    expect(screen.getByText(/count: 5/i)).toBeInTheDocument();
  });
});
```

### **Mock Custom Hooks**

```typescript
// hooks/useUser.ts
export const useUser = (id: string) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUser(id).then(setUser).finally(() => setLoading(false));
  }, [id]);

  return { user, loading };
};

// UserProfile.test.tsx
import { render, screen } from '@testing-library/react';
import { UserProfile } from './UserProfile';
import * as hooks from './hooks/useUser';

jest.mock('./hooks/useUser');

describe('UserProfile', () => {
  test('displays loading state', () => {
    (hooks.useUser as jest.Mock).mockReturnValue({
      user: null,
      loading: true
    });

    render(<UserProfile userId="1" />);

    expect(screen.getByText(/loading/i)).toBeInTheDocument();
  });

  test('displays user data', () => {
    (hooks.useUser as jest.Mock).mockReturnValue({
      user: { id: '1', name: 'John Doe' },
      loading: false
    });

    render(<UserProfile userId="1" />);

    expect(screen.getByText('John Doe')).toBeInTheDocument();
  });
});
```

---

## Mocking Context and Providers

### **Mock Context Values**

```typescript
import { render, screen } from '@testing-library/react';
import { AuthContext } from './AuthContext';
import { Dashboard } from './Dashboard';

describe('Dashboard', () => {
  const renderWithAuth = (authValue: any) => {
    return render(
      <AuthContext.Provider value={authValue}>
        <Dashboard />
      </AuthContext.Provider>
    );
  };

  test('shows content for authenticated user', () => {
    renderWithAuth({
      user: { id: '1', name: 'John', role: 'admin' },
      isAuthenticated: true
    });

    expect(screen.getByText(/welcome, john/i)).toBeInTheDocument();
    expect(screen.getByText(/admin panel/i)).toBeInTheDocument();
  });

  test('shows login prompt for unauthenticated user', () => {
    renderWithAuth({
      user: null,
      isAuthenticated: false
    });

    expect(screen.getByText(/please log in/i)).toBeInTheDocument();
    expect(screen.queryByText(/admin panel/i)).not.toBeInTheDocument();
  });
});
```

---

## Mocking Third-Party Libraries

### **Mock Axios**

```typescript
import axios from 'axios';
import { render, screen } from '@testing-library/react';
import { UserList } from './UserList';

jest.mock('axios');
const mockedAxios = axios as jest.Mocked<typeof axios>;

describe('UserList', () => {
  test('fetches and displays users', async () => {
    mockedAxios.get.mockResolvedValue({
      data: [
        { id: '1', name: 'John' },
        { id: '2', name: 'Jane' }
      ]
    });

    render(<UserList />);

    expect(await screen.findByText('John')).toBeInTheDocument();
    expect(await screen.findByText('Jane')).toBeInTheDocument();

    expect(mockedAxios.get).toHaveBeenCalledWith('/api/users');
  });

  test('handles axios error', async () => {
    mockedAxios.get.mockRejectedValue(new Error('Network error'));

    render(<UserList />);

    expect(await screen.findByText(/error/i)).toBeInTheDocument();
  });
});
```

### **Mock React Router**

```typescript
import { render, screen } from '@testing-library/react';
import { useNavigate, useParams } from 'react-router-dom';
import { UserDetail } from './UserDetail';

jest.mock('react-router-dom', () => ({
  ...jest.requireActual('react-router-dom'),
  useNavigate: jest.fn(),
  useParams: jest.fn(),
}));

describe('UserDetail', () => {
  const mockNavigate = jest.fn();

  beforeEach(() => {
    (useNavigate as jest.Mock).mockReturnValue(mockNavigate);
    (useParams as jest.Mock).mockReturnValue({ id: '1' });
  });

  test('navigates back when clicking back button', async () => {
    const user = userEvent.setup();

    render(<UserDetail />);

    await user.click(screen.getByRole('button', { name: /back/i }));

    expect(mockNavigate).toHaveBeenCalledWith(-1);
  });
});
```

### **Mock React Query**

```typescript
import { render, screen } from '@testing-library/react';
import { useQuery } from '@tanstack/react-query';
import { UserProfile } from './UserProfile';

jest.mock('@tanstack/react-query');

describe('UserProfile', () => {
  test('displays loading state', () => {
    (useQuery as jest.Mock).mockReturnValue({
      data: null,
      isLoading: true,
      error: null
    });

    render(<UserProfile userId="1" />);

    expect(screen.getByText(/loading/i)).toBeInTheDocument();
  });

  test('displays user data', () => {
    (useQuery as jest.Mock).mockReturnValue({
      data: { id: '1', name: 'John Doe' },
      isLoading: false,
      error: null
    });

    render(<UserProfile userId="1" />);

    expect(screen.getByText('John Doe')).toBeInTheDocument();
  });

  test('displays error', () => {
    (useQuery as jest.Mock).mockReturnValue({
      data: null,
      isLoading: false,
      error: new Error('Failed to fetch')
    });

    render(<UserProfile userId="1" />);

    expect(screen.getByText(/error/i)).toBeInTheDocument();
  });
});
```

---

## Mocking Timers

### **jest.useFakeTimers()**

```typescript
import { render, screen } from '@testing-library/react';
import { AutoSaveForm } from './AutoSaveForm';

describe('AutoSaveForm', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  test('auto-saves after delay', () => {
    const onSave = jest.fn();

    render(<AutoSaveForm onSave={onSave} />);

    const input = screen.getByRole('textbox');
    fireEvent.change(input, { target: { value: 'test' } });

    // Not saved yet
    expect(onSave).not.toHaveBeenCalled();

    // Fast-forward time
    jest.advanceTimersByTime(2000);

    // Now saved
    expect(onSave).toHaveBeenCalledWith('test');
  });

  test('debounces multiple changes', () => {
    const onSave = jest.fn();

    render(<AutoSaveForm onSave={onSave} />);

    const input = screen.getByRole('textbox');

    fireEvent.change(input, { target: { value: 'a' } });
    jest.advanceTimersByTime(500);

    fireEvent.change(input, { target: { value: 'ab' } });
    jest.advanceTimersByTime(500);

    fireEvent.change(input, { target: { value: 'abc' } });
    jest.advanceTimersByTime(2000);

    // Should only save final value
    expect(onSave).toHaveBeenCalledTimes(1);
    expect(onSave).toHaveBeenCalledWith('abc');
  });
});
```

---

## Mocking Browser APIs

### **localStorage**

```typescript
const localStorageMock = (() => {
  let store: Record<string, string> = {};

  return {
    getItem: jest.fn((key: string) => store[key] || null),
    setItem: jest.fn((key: string, value: string) => {
      store[key] = value;
    }),
    removeItem: jest.fn((key: string) => {
      delete store[key];
    }),
    clear: jest.fn(() => {
      store = {};
    }),
  };
})();

Object.defineProperty(window, "localStorage", {
  value: localStorageMock,
});

describe("useLocalStorage", () => {
  beforeEach(() => {
    localStorageMock.clear();
  });

  test("saves to localStorage", () => {
    const { result } = renderHook(() => useLocalStorage("key", "default"));

    act(() => {
      result.current[1]("new value");
    });

    expect(localStorageMock.setItem).toHaveBeenCalledWith("key", '"new value"');
  });
});
```

### **window.matchMedia**

```typescript
Object.defineProperty(window, "matchMedia", {
  writable: true,
  value: jest.fn().mockImplementation((query) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: jest.fn(),
    removeListener: jest.fn(),
    addEventListener: jest.fn(),
    removeEventListener: jest.fn(),
    dispatchEvent: jest.fn(),
  })),
});
```

### **IntersectionObserver**

```typescript
const mockIntersectionObserver = jest.fn();
mockIntersectionObserver.mockReturnValue({
  observe: jest.fn(),
  unobserve: jest.fn(),
  disconnect: jest.fn(),
});

window.IntersectionObserver = mockIntersectionObserver as any;
```

---

## Best Practices

### ✅ DO: Mock External Dependencies

```typescript
// Good - Mock API calls
jest.mock("./api");

test("component with mocked API", () => {
  mockApi.fetchData.mockResolvedValue(testData);
  // Test component
});
```

### ❌ DON'T: Mock Internal Implementation

```typescript
// Bad - Testing implementation details
jest.mock("./useCounter", () => ({
  useCounter: () => ({ count: 5 }),
}));
```

### ✅ DO: Use MSW for HTTP Mocking

```typescript
// Good - Realistic API mocking
server.use(
  rest.get("/api/users", (req, res, ctx) => {
    return res(ctx.json(users));
  }),
);
```

### ✅ DO: Reset Mocks Between Tests

```typescript
beforeEach(() => {
  jest.clearAllMocks();
});
```

### ✅ DO: Verify Mock Calls

```typescript
// Good - Verify behavior
expect(mockFn).toHaveBeenCalledWith("expected", "args");
expect(mockFn).toHaveBeenCalledTimes(1);
```

---

## Key Takeaways

1. **Jest mocks** enable isolated component testing
2. **MSW** provides realistic HTTP mocking without changing code
3. **Mock external dependencies** (APIs, libraries) not internal logic
4. **Spy functions** track calls and arguments
5. **mockResolvedValue** for async functions
6. **Reset mocks** between tests to avoid interference
7. **Fake timers** test time-dependent code deterministically
8. **Mock browser APIs** (localStorage, IntersectionObserver) in tests
9. **Partial mocks** keep some real implementation
10. **Verify mock interactions** to ensure correct behavior
