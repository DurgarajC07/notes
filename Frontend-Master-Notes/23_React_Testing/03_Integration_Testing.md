# Integration Testing in React

## Core Concept

Integration testing verifies that **multiple components work together correctly**, testing interactions between components, context providers, routers, and state management without mocking internal dependencies.

---

## Testing Components with Context

### **Basic Context Integration**

```typescript
import { render, screen } from '@testing-library/react';
import { ThemeContext, ThemeProvider } from './ThemeContext';
import { ThemedButton } from './ThemedButton';

describe('ThemedButton with Context', () => {
  test('renders with theme from context', () => {
    render(
      <ThemeProvider>
        <ThemedButton>Click me</ThemedButton>
      </ThemeProvider>
    );

    const button = screen.getByRole('button', { name: /click me/i });
    expect(button).toHaveStyle({ backgroundColor: 'blue' });
  });

  test('updates when theme changes', async () => {
    const { rerender } = render(
      <ThemeContext.Provider value={{ theme: 'light' }}>
        <ThemedButton>Click me</ThemedButton>
      </ThemeContext.Provider>
    );

    expect(screen.getByRole('button')).toHaveClass('theme-light');

    rerender(
      <ThemeContext.Provider value={{ theme: 'dark' }}>
        <ThemedButton>Click me</ThemedButton>
      </ThemeContext.Provider>
    );

    expect(screen.getByRole('button')).toHaveClass('theme-dark');
  });
});
```

---

## Testing with Router

### **React Router Integration**

```typescript
import { render, screen } from '@testing-library/react';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import userEvent from '@testing-library/user-event';
import { Navigation } from './Navigation';
import { HomePage } from './HomePage';
import { ProfilePage } from './ProfilePage';

describe('Navigation Integration', () => {
  const renderWithRouter = (initialRoute = '/') => {
    return render(
      <MemoryRouter initialEntries={[initialRoute]}>
        <Navigation />
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/profile" element={<ProfilePage />} />
        </Routes>
      </MemoryRouter>
    );
  };

  test('navigates between pages', async () => {
    const user = userEvent.setup();
    renderWithRouter('/');

    expect(screen.getByText(/home page/i)).toBeInTheDocument();

    const profileLink = screen.getByRole('link', { name: /profile/i });
    await user.click(profileLink);

    expect(screen.getByText(/profile page/i)).toBeInTheDocument();
    expect(screen.queryByText(/home page/i)).not.toBeInTheDocument();
  });

  test('renders correct page on initial route', () => {
    renderWithRouter('/profile');

    expect(screen.getByText(/profile page/i)).toBeInTheDocument();
  });

  test('handles protected routes', async () => {
    const user = userEvent.setup();

    render(
      <MemoryRouter initialEntries={['/']}>
        <AuthProvider>
          <Navigation />
          <Routes>
            <Route path="/" element={<HomePage />} />
            <Route
              path="/dashboard"
              element={
                <ProtectedRoute>
                  <DashboardPage />
                </ProtectedRoute>
              }
            />
            <Route path="/login" element={<LoginPage />} />
          </Routes>
        </AuthProvider>
      </MemoryRouter>
    );

    const dashboardLink = screen.getByRole('link', { name: /dashboard/i });
    await user.click(dashboardLink);

    // Should redirect to login
    expect(screen.getByText(/login page/i)).toBeInTheDocument();
  });
});
```

---

## Testing State Management Integration

### **Redux Toolkit Integration**

```typescript
import { render, screen } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import userEvent from '@testing-library/user-event';
import counterReducer from './counterSlice';
import { Counter } from './Counter';
import { CounterDisplay } from './CounterDisplay';

describe('Redux Integration', () => {
  const createTestStore = (preloadedState = {}) => {
    return configureStore({
      reducer: {
        counter: counterReducer,
      },
      preloadedState,
    });
  };

  const renderWithStore = (
    component: React.ReactElement,
    initialState = {}
  ) => {
    const store = createTestStore(initialState);
    return {
      ...render(<Provider store={store}>{component}</Provider>),
      store,
    };
  };

  test('multiple components share state', async () => {
    const user = userEvent.setup();

    renderWithStore(
      <>
        <Counter />
        <CounterDisplay />
      </>
    );

    expect(screen.getByText(/count: 0/i)).toBeInTheDocument();

    const incrementButton = screen.getByRole('button', { name: /increment/i });
    await user.click(incrementButton);

    // Both components show updated count
    expect(screen.getByText(/count: 1/i)).toBeInTheDocument();
  });

  test('starts with preloaded state', () => {
    renderWithStore(<CounterDisplay />, {
      counter: { value: 42 },
    });

    expect(screen.getByText(/count: 42/i)).toBeInTheDocument();
  });

  test('dispatches async thunks', async () => {
    const user = userEvent.setup();

    const { store } = renderWithStore(<Counter />);

    const asyncButton = screen.getByRole('button', { name: /increment async/i });
    await user.click(asyncButton);

    // Wait for async action
    await screen.findByText(/count: 1/i);

    expect(store.getState().counter.value).toBe(1);
  });
});
```

### **Zustand Integration**

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { act } from 'react-dom/test-utils';
import { useStore } from './store';
import { Counter } from './Counter';
import { DisplayValue } from './DisplayValue';

describe('Zustand Integration', () => {
  beforeEach(() => {
    // Reset store between tests
    act(() => {
      useStore.setState({ count: 0 });
    });
  });

  test('components share Zustand store', async () => {
    const user = userEvent.setup();

    render(
      <>
        <Counter />
        <DisplayValue />
      </>
    );

    expect(screen.getByText(/value: 0/i)).toBeInTheDocument();

    const button = screen.getByRole('button', { name: /increment/i });
    await user.click(button);

    expect(screen.getByText(/value: 1/i)).toBeInTheDocument();
  });

  test('can set initial state', () => {
    act(() => {
      useStore.setState({ count: 100 });
    });

    render(<DisplayValue />);

    expect(screen.getByText(/value: 100/i)).toBeInTheDocument();
  });
});
```

---

## Testing Form Workflows

### **Multi-Step Form**

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MultiStepForm } from './MultiStepForm';

describe('MultiStepForm Integration', () => {
  test('completes full form workflow', async () => {
    const user = userEvent.setup();
    const onSubmit = jest.fn();

    render(<MultiStepForm onSubmit={onSubmit} />);

    // Step 1: Personal Info
    expect(screen.getByText(/step 1/i)).toBeInTheDocument();

    await user.type(screen.getByLabelText(/name/i), 'John Doe');
    await user.type(screen.getByLabelText(/email/i), 'john@example.com');

    const nextButton = screen.getByRole('button', { name: /next/i });
    await user.click(nextButton);

    // Step 2: Address
    expect(screen.getByText(/step 2/i)).toBeInTheDocument();

    await user.type(screen.getByLabelText(/street/i), '123 Main St');
    await user.type(screen.getByLabelText(/city/i), 'Springfield');
    await user.click(screen.getByRole('button', { name: /next/i }));

    // Step 3: Review and Submit
    expect(screen.getByText(/review/i)).toBeInTheDocument();
    expect(screen.getByText(/john doe/i)).toBeInTheDocument();
    expect(screen.getByText(/123 main st/i)).toBeInTheDocument();

    await user.click(screen.getByRole('button', { name: /submit/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        personalInfo: {
          name: 'John Doe',
          email: 'john@example.com',
        },
        address: {
          street: '123 Main St',
          city: 'Springfield',
        },
      });
    });
  });

  test('can navigate back', async () => {
    const user = userEvent.setup();

    render(<MultiStepForm onSubmit={jest.fn()} />);

    await user.type(screen.getByLabelText(/name/i), 'John Doe');
    await user.click(screen.getByRole('button', { name: /next/i }));

    expect(screen.getByText(/step 2/i)).toBeInTheDocument();

    await user.click(screen.getByRole('button', { name: /back/i }));

    expect(screen.getByText(/step 1/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/name/i)).toHaveValue('John Doe'); // Preserves data
  });

  test('shows validation errors', async () => {
    const user = userEvent.setup();

    render(<MultiStepForm onSubmit={jest.fn()} />);

    // Try to proceed without filling fields
    await user.click(screen.getByRole('button', { name: /next/i }));

    expect(await screen.findByText(/name is required/i)).toBeInTheDocument();
    expect(await screen.findByText(/email is required/i)).toBeInTheDocument();
  });
});
```

---

## Testing Data Fetching Integration

### **With React Query**

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { rest } from 'msw';
import { setupServer } from 'msw/node';
import { UserProfile } from './UserProfile';
import { UserPosts } from './UserPosts';

const server = setupServer(
  rest.get('/api/user/:id', (req, res, ctx) => {
    return res(
      ctx.json({
        id: '1',
        name: 'John Doe',
        email: 'john@example.com',
      })
    );
  }),
  rest.get('/api/posts', (req, res, ctx) => {
    return res(
      ctx.json([
        { id: '1', title: 'First Post', authorId: '1' },
        { id: '2', title: 'Second Post', authorId: '1' },
      ])
    );
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('React Query Integration', () => {
  const createWrapper = () => {
    const queryClient = new QueryClient({
      defaultOptions: {
        queries: { retry: false },
      },
    });

    return ({ children }: { children: React.ReactNode }) => (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    );
  };

  test('components share cached data', async () => {
    render(
      <>
        <UserProfile userId="1" />
        <UserPosts userId="1" />
      </>,
      { wrapper: createWrapper() }
    );

    // Both components fetch and display data
    await waitFor(() => {
      expect(screen.getByText(/john doe/i)).toBeInTheDocument();
      expect(screen.getByText(/first post/i)).toBeInTheDocument();
      expect(screen.getByText(/second post/i)).toBeInTheDocument();
    });

    // Second component should use cached user data
    expect(await screen.findAllByText(/john doe/i)).toHaveLength(2);
  });

  test('handles API errors', async () => {
    server.use(
      rest.get('/api/user/:id', (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ error: 'Server error' }));
      })
    );

    render(<UserProfile userId="1" />, { wrapper: createWrapper() });

    expect(await screen.findByText(/error loading user/i)).toBeInTheDocument();
  });
});
```

---

## Testing Modal and Dialog Flows

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { App } from './App';

describe('Modal Integration', () => {
  test('opens and closes delete confirmation modal', async () => {
    const user = userEvent.setup();

    render(<App />);

    const deleteButton = screen.getByRole('button', { name: /delete item/i });
    await user.click(deleteButton);

    // Modal appears
    expect(screen.getByRole('dialog')).toBeInTheDocument();
    expect(screen.getByText(/are you sure/i)).toBeInTheDocument();

    // Cancel closes modal
    await user.click(screen.getByRole('button', { name: /cancel/i }));
    expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  });

  test('completes delete action', async () => {
    const user = userEvent.setup();
    const onDelete = jest.fn();

    render(<App onDelete={onDelete} />);

    await user.click(screen.getByRole('button', { name: /delete item/i }));
    await user.click(screen.getByRole('button', { name: /confirm/i }));

    expect(onDelete).toHaveBeenCalled();
    expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  });

  test('modal traps focus', async () => {
    const user = userEvent.setup();

    render(<App />);

    await user.click(screen.getByRole('button', { name: /delete item/i }));

    const modal = screen.getByRole('dialog');
    const cancelButton = screen.getByRole('button', { name: /cancel/i });
    const confirmButton = screen.getByRole('button', { name: /confirm/i });

    // Tab cycles through modal buttons
    await user.tab();
    expect(confirmButton).toHaveFocus();

    await user.tab();
    expect(cancelButton).toHaveFocus();

    await user.tab();
    expect(confirmButton).toHaveFocus(); // Cycles back
  });
});
```

---

## Testing Compound Components

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Tabs } from './Tabs';

describe('Tabs Compound Component', () => {
  test('integrates tab list, panels, and navigation', async () => {
    const user = userEvent.setup();

    render(
      <Tabs defaultValue="tab1">
        <Tabs.List>
          <Tabs.Tab value="tab1">Tab 1</Tabs.Tab>
          <Tabs.Tab value="tab2">Tab 2</Tabs.Tab>
          <Tabs.Tab value="tab3">Tab 3</Tabs.Tab>
        </Tabs.List>

        <Tabs.Panel value="tab1">Content 1</Tabs.Panel>
        <Tabs.Panel value="tab2">Content 2</Tabs.Panel>
        <Tabs.Panel value="tab3">Content 3</Tabs.Panel>
      </Tabs>
    );

    // Default tab is active
    expect(screen.getByRole('tab', { name: /tab 1/i })).toHaveAttribute('aria-selected', 'true');
    expect(screen.getByText(/content 1/i)).toBeVisible();

    // Click second tab
    await user.click(screen.getByRole('tab', { name: /tab 2/i }));

    expect(screen.getByRole('tab', { name: /tab 2/i })).toHaveAttribute('aria-selected', 'true');
    expect(screen.getByText(/content 2/i)).toBeVisible();
    expect(screen.queryByText(/content 1/i)).not.toBeInTheDocument();
  });

  test('keyboard navigation', async () => {
    const user = userEvent.setup();

    render(
      <Tabs defaultValue="tab1">
        <Tabs.List>
          <Tabs.Tab value="tab1">Tab 1</Tabs.Tab>
          <Tabs.Tab value="tab2">Tab 2</Tabs.Tab>
        </Tabs.List>
        <Tabs.Panel value="tab1">Content 1</Tabs.Panel>
        <Tabs.Panel value="tab2">Content 2</Tabs.Panel>
      </Tabs>
    );

    const tab1 = screen.getByRole('tab', { name: /tab 1/i });
    tab1.focus();

    await user.keyboard('{ArrowRight}');
    expect(screen.getByRole('tab', { name: /tab 2/i })).toHaveFocus();
    expect(screen.getByText(/content 2/i)).toBeVisible();
  });
});
```

---

## Testing with Multiple Providers

```typescript
import { render } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';
import { AuthProvider } from './AuthProvider';
import { ThemeProvider } from './ThemeProvider';

describe('Multiple Providers Integration', () => {
  const AllProviders = ({ children }: { children: React.ReactNode }) => {
    const queryClient = new QueryClient({
      defaultOptions: { queries: { retry: false } },
    });

    return (
      <QueryClientProvider client={queryClient}>
        <BrowserRouter>
          <AuthProvider>
            <ThemeProvider>
              {children}
            </ThemeProvider>
          </AuthProvider>
        </BrowserRouter>
      </QueryClientProvider>
    );
  };

  test('renders app with all providers', () => {
    const { container } = render(<App />, { wrapper: AllProviders });

    expect(container.firstChild).toMatchSnapshot();
  });
});
```

---

## Best Practices

### ✅ DO: Test User Workflows

```typescript
// Good - Tests complete user journey
test('user can search and filter products', async () => {
  const user = userEvent.setup();
  render(<ProductPage />);

  await user.type(screen.getByPlaceholderText(/search/i), 'laptop');
  await user.click(screen.getByRole('button', { name: /search/i }));

  expect(await screen.findByText(/laptop results/i)).toBeInTheDocument();

  await user.click(screen.getByLabelText(/in stock only/i));

  expect(screen.getAllByRole('article')).toHaveLength(5);
});
```

### ❌ DON'T: Over-Mock Internal Components

```typescript
// Bad - Mocking internal components defeats integration testing
jest.mock('./UserCard', () => ({
  UserCard: () => <div>Mocked UserCard</div>
}));

test('renders user list', () => {
  render(<UserList />);
  // This doesn't test actual integration
});
```

### ✅ DO: Use Real Providers

```typescript
// Good - Uses real providers to test integration
const wrapper = ({ children }) => (
  <ThemeProvider>
    <AuthProvider>
      {children}
    </AuthProvider>
  </ThemeProvider>
);

test('component uses theme and auth', () => {
  render(<MyComponent />, { wrapper });
});
```

---

## Key Takeaways

1. **Integration tests verify component interactions** without mocking internals
2. **Wrap components in providers** (Router, Redux, React Query, Context)
3. **Test complete user workflows** rather than isolated actions
4. **Use MSW for API mocking** instead of mocking fetch/axios
5. **MemoryRouter** enables testing React Router navigation
6. **Reset state between tests** to avoid test interdependencies
7. **Test keyboard navigation** for accessibility compliance
8. **Focus on user behavior** not implementation details
9. **Compound components** should test child component coordination
10. **Multi-step forms** test data persistence across steps
