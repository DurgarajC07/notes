# Unit Testing React Components

## Core Concept

Unit testing focuses on testing individual components in isolation, ensuring they render correctly, handle props appropriately, and respond to user interactions as expected. Jest and React Testing Library are the standard tools for unit testing React applications.

---

## Jest Fundamentals

### Test Structure

```typescript
// Button.test.tsx
import { render, screen } from '@testing-library/react';
import { Button } from './Button';

describe('Button', () => {
  it('renders button with text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button', { name: 'Click me' })).toBeInTheDocument();
  });

  it('calls onClick when clicked', () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click</Button>);

    screen.getByRole('button').click();
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });
});
```

### Test Organization

```typescript
describe("UserProfile", () => {
  describe("rendering", () => {
    it("displays user name", () => {});
    it("displays user email", () => {});
    it("displays avatar", () => {});
  });

  describe("editing", () => {
    it("shows edit form when edit button clicked", () => {});
    it("saves changes on submit", () => {});
    it("cancels changes on cancel", () => {});
  });

  describe("error handling", () => {
    it("displays error message when save fails", () => {});
    it("retries on retry button click", () => {});
  });
});
```

---

## Testing Component Props

### Different Prop Scenarios

```typescript
// Card.tsx
interface CardProps {
  title: string;
  description?: string;
  image?: string;
  actions?: React.ReactNode;
}

export function Card({ title, description, image, actions }: CardProps) {
  return (
    <div className="card">
      {image && <img src={image} alt={title} />}
      <h3>{title}</h3>
      {description && <p>{description}</p>}
      {actions && <div className="card-actions">{actions}</div>}
    </div>
  );
}

// Card.test.tsx
describe('Card', () => {
  const requiredProps = {
    title: 'Test Card'
  };

  it('renders with required props only', () => {
    render(<Card {...requiredProps} />);
    expect(screen.getByText('Test Card')).toBeInTheDocument();
  });

  it('renders description when provided', () => {
    render(<Card {...requiredProps} description="Test description" />);
    expect(screen.getByText('Test description')).toBeInTheDocument();
  });

  it('does not render description when not provided', () => {
    render(<Card {...requiredProps} />);
    expect(screen.queryByText(/description/i)).not.toBeInTheDocument();
  });

  it('renders image when provided', () => {
    render(<Card {...requiredProps} image="/test.jpg" />);
    const img = screen.getByRole('img', { name: 'Test Card' });
    expect(img).toHaveAttribute('src', '/test.jpg');
  });

  it('renders custom actions', () => {
    const actions = <button>Edit</button>;
    render(<Card {...requiredProps} actions={actions} />);
    expect(screen.getByRole('button', { name: 'Edit' })).toBeInTheDocument();
  });
});
```

---

## Testing State and Events

### useState Testing

```typescript
// Counter.tsx
export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(count - 1)}>Decrement</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}

// Counter.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('Counter', () => {
  it('starts at 0', () => {
    render(<Counter />);
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });

  it('increments count', async () => {
    const user = userEvent.setup();
    render(<Counter />);

    await user.click(screen.getByRole('button', { name: 'Increment' }));
    expect(screen.getByText('Count: 1')).toBeInTheDocument();

    await user.click(screen.getByRole('button', { name: 'Increment' }));
    expect(screen.getByText('Count: 2')).toBeInTheDocument();
  });

  it('decrements count', async () => {
    const user = userEvent.setup();
    render(<Counter />);

    await user.click(screen.getByRole('button', { name: 'Decrement' }));
    expect(screen.getByText('Count: -1')).toBeInTheDocument();
  });

  it('resets count to 0', async () => {
    const user = userEvent.setup();
    render(<Counter />);

    await user.click(screen.getByRole('button', { name: 'Increment' }));
    await user.click(screen.getByRole('button', { name: 'Increment' }));
    await user.click(screen.getByRole('button', { name: 'Reset' }));

    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });
});
```

---

## Testing Forms

### Input Validation and Submission

```typescript
// LoginForm.tsx
export function LoginForm({ onSubmit }: { onSubmit: (data: LoginData) => void }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<Record<string, string>>({});

  const validate = () => {
    const newErrors: Record<string, string> = {};

    if (!email) newErrors.email = 'Email is required';
    if (!email.includes('@')) newErrors.email = 'Invalid email';
    if (!password) newErrors.password = 'Password is required';
    if (password.length < 8) newErrors.password = 'Password must be 8+ characters';

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (validate()) {
      onSubmit({ email, password });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        placeholder="Email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        aria-label="Email"
      />
      {errors.email && <span role="alert">{errors.email}</span>}

      <input
        type="password"
        placeholder="Password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        aria-label="Password"
      />
      {errors.password && <span role="alert">{errors.password}</span>}

      <button type="submit">Login</button>
    </form>
  );
}

// LoginForm.test.tsx
describe('LoginForm', () => {
  it('submits form with valid data', async () => {
    const user = userEvent.setup();
    const onSubmit = jest.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'test@example.com');
    await user.type(screen.getByLabelText('Password'), 'password123');
    await user.click(screen.getByRole('button', { name: 'Login' }));

    expect(onSubmit).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password123'
    });
  });

  it('shows email error when email is empty', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={jest.fn()} />);

    await user.click(screen.getByRole('button', { name: 'Login' }));

    expect(screen.getByRole('alert')).toHaveTextContent('Email is required');
  });

  it('shows email error when email is invalid', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={jest.fn()} />);

    await user.type(screen.getByLabelText('Email'), 'invalid-email');
    await user.click(screen.getByRole('button', { name: 'Login' }));

    expect(screen.getByRole('alert')).toHaveTextContent('Invalid email');
  });

  it('shows password error when password is too short', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={jest.fn()} />);

    await user.type(screen.getByLabelText('Email'), 'test@example.com');
    await user.type(screen.getByLabelText('Password'), 'short');
    await user.click(screen.getByRole('button', { name: 'Login' }));

    expect(screen.getByRole('alert')).toHaveTextContent('Password must be 8+ characters');
  });
});
```

---

## Testing Async Components

### With useEffect

```typescript
// UserProfile.tsx
export function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    fetchUser(userId)
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!user) return <div>User not found</div>;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

// UserProfile.test.tsx
import { render, screen, waitFor } from '@testing-library/react';

jest.mock('./api', () => ({
  fetchUser: jest.fn()
}));

const mockFetchUser = fetchUser as jest.MockedFunction<typeof fetchUser>;

describe('UserProfile', () => {
  beforeEach(() => {
    mockFetchUser.mockClear();
  });

  it('shows loading state initially', () => {
    mockFetchUser.mockImplementation(() => new Promise(() => {})); // Never resolves

    render(<UserProfile userId="1" />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  it('displays user data after loading', async () => {
    mockFetchUser.mockResolvedValue({
      id: '1',
      name: 'John Doe',
      email: 'john@example.com'
    });

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });

    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  it('displays error message on fetch failure', async () => {
    mockFetchUser.mockRejectedValue(new Error('Network error'));

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText('Error: Network error')).toBeInTheDocument();
    });
  });

  it('refetches when userId changes', async () => {
    mockFetchUser.mockResolvedValue({
      id: '1',
      name: 'John Doe',
      email: 'john@example.com'
    });

    const { rerender } = render(<UserProfile userId="1" />);

    await waitFor(() => screen.getByText('John Doe'));

    mockFetchUser.mockResolvedValue({
      id: '2',
      name: 'Jane Smith',
      email: 'jane@example.com'
    });

    rerender(<UserProfile userId="2" />);

    await waitFor(() => {
      expect(screen.getByText('Jane Smith')).toBeInTheDocument();
    });

    expect(mockFetchUser).toHaveBeenCalledTimes(2);
  });
});
```

---

## Testing Custom Hooks

### renderHook Utility

```typescript
// useCounter.ts
export function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);

  const increment = () => setCount((c) => c + 1);
  const decrement = () => setCount((c) => c - 1);
  const reset = () => setCount(initialValue);

  return { count, increment, decrement, reset };
}

// useCounter.test.ts
import { renderHook, act } from "@testing-library/react";

describe("useCounter", () => {
  it("initializes with default value", () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it("initializes with provided value", () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it("increments count", () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it("decrements count", () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(4);
  });

  it("resets to initial value", () => {
    const { result } = renderHook(() => useCounter(10));

    act(() => {
      result.current.increment();
      result.current.increment();
    });

    expect(result.current.count).toBe(12);

    act(() => {
      result.current.reset();
    });

    expect(result.current.count).toBe(10);
  });
});
```

---

## Snapshot Testing

### Component Snapshots

```typescript
// Button.test.tsx
it('matches snapshot', () => {
  const { container } = render(<Button variant="primary">Click me</Button>);
  expect(container.firstChild).toMatchSnapshot();
});

// Generated snapshot file:
// exports[`Button matches snapshot 1`] = `
// <button
//   class="btn btn-primary"
//   type="button"
// >
//   Click me
// </button>
// `;

// Update snapshots when intentional changes are made
// npm test -- -u
```

### Inline Snapshots

```typescript
it('renders loading state', () => {
  render(<Button loading>Submit</Button>);

  expect(screen.getByRole('button')).toMatchInlineSnapshot(`
    <button
      class="btn btn-loading"
      disabled=""
    >
      <span class="spinner" />
      Submit
    </button>
  `);
});
```

---

## Test Coverage

### Coverage Configuration

```json
// package.json
{
  "jest": {
    "collectCoverageFrom": [
      "src/**/*.{ts,tsx}",
      "!src/**/*.test.{ts,tsx}",
      "!src/**/*.stories.{ts,tsx}",
      "!src/index.tsx"
    ],
    "coverageThresholds": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

```bash
# Run tests with coverage
npm test -- --coverage

# Coverage report shows:
# - Statement coverage
# - Branch coverage
# - Function coverage
# - Line coverage
```

---

## Best Practices

### 1. Test User Behavior, Not Implementation

```typescript
// ❌ Bad - Testing implementation details
it('calls useState with initial value', () => {
  const spy = jest.spyOn(React, 'useState');
  render(<Counter />);
  expect(spy).toHaveBeenCalledWith(0);
});

// ✅ Good - Testing user-visible behavior
it('displays initial count of 0', () => {
  render(<Counter />);
  expect(screen.getByText('Count: 0')).toBeInTheDocument();
});
```

### 2. Use Accessible Queries

```typescript
// ❌ Bad - Using test IDs
screen.getByTestId("submit-button");

// ✅ Good - Using accessible queries
screen.getByRole("button", { name: "Submit" });
screen.getByLabelText("Email");
screen.getByText("Welcome");
```

### 3. Avoid Testing Library Implementation

```typescript
// ❌ Bad - Testing React implementation
expect(wrapper.state("count")).toBe(5);

// ✅ Good - Testing rendered output
expect(screen.getByText("Count: 5")).toBeInTheDocument();
```

---

## Key Takeaways

1. **Jest** provides test runner, assertions, and mocking capabilities
2. **React Testing Library** encourages testing user behavior over implementation
3. **Unit tests** focus on individual components in isolation
4. Use **userEvent** for simulating user interactions
5. **waitFor** handles async operations and updates
6. **renderHook** tests custom hooks without creating test components
7. **Snapshot testing** captures component output for regression testing
8. **Coverage thresholds** ensure adequate test coverage
9. **Mock external dependencies** (API calls, context, etc.)
10. Test **accessibility** by using semantic queries (role, label, text)
