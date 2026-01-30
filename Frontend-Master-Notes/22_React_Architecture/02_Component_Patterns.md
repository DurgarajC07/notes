# React Component Patterns

## Core Concept

Component patterns are reusable solutions to common design problems in React applications. Understanding compound components, render props, higher-order components, and provider patterns enables building flexible, maintainable, and scalable UIs.

---

## Compound Components

### Basic Pattern

```typescript
// Tab.tsx - Compound component with shared state
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (id: string) => void;
}

const TabsContext = React.createContext<TabsContextValue | undefined>(undefined);

function useTabs() {
  const context = React.useContext(TabsContext);
  if (!context) {
    throw new Error('Tab components must be used within Tabs');
  }
  return context;
}

interface TabsProps {
  defaultTab: string;
  children: React.ReactNode;
}

export function Tabs({ defaultTab, children }: TabsProps) {
  const [activeTab, setActiveTab] = React.useState(defaultTab);

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

// Child components
interface TabListProps {
  children: React.ReactNode;
}

Tabs.TabList = function TabList({ children }: TabListProps) {
  return <div className="tab-list" role="tablist">{children}</div>;
};

interface TabProps {
  id: string;
  children: React.ReactNode;
}

Tabs.Tab = function Tab({ id, children }: TabProps) {
  const { activeTab, setActiveTab } = useTabs();

  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      onClick={() => setActiveTab(id)}
      className={activeTab === id ? 'tab-active' : 'tab'}
    >
      {children}
    </button>
  );
};

interface TabPanelProps {
  id: string;
  children: React.ReactNode;
}

Tabs.TabPanel = function TabPanel({ id, children }: TabPanelProps) {
  const { activeTab } = useTabs();

  if (activeTab !== id) return null;

  return (
    <div role="tabpanel" className="tab-panel">
      {children}
    </div>
  );
};

// Usage
function App() {
  return (
    <Tabs defaultTab="home">
      <Tabs.TabList>
        <Tabs.Tab id="home">Home</Tabs.Tab>
        <Tabs.Tab id="profile">Profile</Tabs.Tab>
        <Tabs.Tab id="settings">Settings</Tabs.Tab>
      </Tabs.TabList>

      <Tabs.TabPanel id="home">
        <h2>Home Content</h2>
      </Tabs.TabPanel>
      <Tabs.TabPanel id="profile">
        <h2>Profile Content</h2>
      </Tabs.TabPanel>
      <Tabs.TabPanel id="settings">
        <h2>Settings Content</h2>
      </Tabs.TabPanel>
    </Tabs>
  );
}
```

---

## Render Props Pattern

### Basic Implementation

```typescript
interface MousePosition {
  x: number;
  y: number;
}

interface MouseTrackerProps {
  children: (position: MousePosition) => React.ReactNode;
  // Or: render prop
  // render: (position: MousePosition) => React.ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
  const [position, setPosition] = React.useState<MousePosition>({ x: 0, y: 0 });

  React.useEffect(() => {
    const handleMove = (e: MouseEvent) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };

    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);

  return <>{children(position)}</>;
}

// Usage
function App() {
  return (
    <MouseTracker>
      {({ x, y }) => (
        <div>
          Mouse position: {x}, {y}
          <div
            style={{
              position: 'fixed',
              left: x,
              top: y,
              width: 20,
              height: 20,
              borderRadius: '50%',
              background: 'red',
              transform: 'translate(-50%, -50%)',
              pointerEvents: 'none'
            }}
          />
        </div>
      )}
    </MouseTracker>
  );
}
```

### Advanced: Data Fetching with Render Props

```typescript
interface FetchDataProps<T> {
  url: string;
  children: (state: {
    data: T | null;
    loading: boolean;
    error: Error | null;
    refetch: () => void;
  }) => React.ReactNode;
}

function FetchData<T>({ url, children }: FetchDataProps<T>) {
  const [data, setData] = React.useState<T | null>(null);
  const [loading, setLoading] = React.useState(true);
  const [error, setError] = React.useState<Error | null>(null);

  const fetchData = React.useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error('Failed to fetch');
      const json = await response.json();
      setData(json);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [url]);

  React.useEffect(() => {
    fetchData();
  }, [fetchData]);

  return <>{children({ data, loading, error, refetch: fetchData })}</>;
}

// Usage
interface User {
  id: number;
  name: string;
  email: string;
}

function UserProfile() {
  return (
    <FetchData<User> url="/api/user/1">
      {({ data, loading, error, refetch }) => {
        if (loading) return <div>Loading...</div>;
        if (error) return <div>Error: {error.message}</div>;
        if (!data) return null;

        return (
          <div>
            <h2>{data.name}</h2>
            <p>{data.email}</p>
            <button onClick={refetch}>Refresh</button>
          </div>
        );
      }}
    </FetchData>
  );
}
```

---

## Provider Pattern

### Context + Custom Hook

```typescript
// ThemeProvider.tsx
interface Theme {
  colors: {
    primary: string;
    secondary: string;
    background: string;
    text: string;
  };
  spacing: {
    sm: string;
    md: string;
    lg: string;
  };
}

const lightTheme: Theme = {
  colors: {
    primary: '#3b82f6',
    secondary: '#8b5cf6',
    background: '#ffffff',
    text: '#111827'
  },
  spacing: {
    sm: '0.5rem',
    md: '1rem',
    lg: '2rem'
  }
};

const darkTheme: Theme = {
  colors: {
    primary: '#60a5fa',
    secondary: '#a78bfa',
    background: '#111827',
    text: '#f9fafb'
  },
  spacing: {
    sm: '0.5rem',
    md: '1rem',
    lg: '2rem'
  }
};

interface ThemeContextValue {
  theme: Theme;
  isDark: boolean;
  toggleTheme: () => void;
}

const ThemeContext = React.createContext<ThemeContextValue | undefined>(undefined);

export function useTheme() {
  const context = React.useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}

interface ThemeProviderProps {
  children: React.ReactNode;
}

export function ThemeProvider({ children }: ThemeProviderProps) {
  const [isDark, setIsDark] = React.useState(false);

  const value: ThemeContextValue = {
    theme: isDark ? darkTheme : lightTheme,
    isDark,
    toggleTheme: () => setIsDark(prev => !prev)
  };

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// Usage in components
function ThemedButton() {
  const { theme, toggleTheme } = useTheme();

  return (
    <button
      onClick={toggleTheme}
      style={{
        background: theme.colors.primary,
        color: 'white',
        padding: theme.spacing.md,
        border: 'none',
        borderRadius: '4px'
      }}
    >
      Toggle Theme
    </button>
  );
}
```

---

## Higher-Order Components (HOC)

### WithAuth HOC

```typescript
interface WithAuthProps {
  user: User | null;
  isAuthenticated: boolean;
}

function withAuth<P extends WithAuthProps>(
  Component: React.ComponentType<P>
): React.FC<Omit<P, keyof WithAuthProps>> {
  return function WithAuthComponent(props) {
    const { user, isAuthenticated } = useAuth();

    if (!isAuthenticated) {
      return <Navigate to="/login" />;
    }

    return <Component {...(props as P)} user={user} isAuthenticated={isAuthenticated} />;
  };
}

// Usage
interface DashboardProps extends WithAuthProps {
  title: string;
}

function Dashboard({ user, title }: DashboardProps) {
  return (
    <div>
      <h1>{title}</h1>
      <p>Welcome, {user?.name}</p>
    </div>
  );
}

export default withAuth(Dashboard);
```

---

## Container/Presentational Pattern

### Separation of Concerns

```typescript
// UserListContainer.tsx - Logic
interface User {
  id: number;
  name: string;
  email: string;
}

function UserListContainer() {
  const [users, setUsers] = React.useState<User[]>([]);
  const [loading, setLoading] = React.useState(true);
  const [searchTerm, setSearchTerm] = React.useState('');

  React.useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(data => {
        setUsers(data);
        setLoading(false);
      });
  }, []);

  const filteredUsers = users.filter(user =>
    user.name.toLowerCase().includes(searchTerm.toLowerCase())
  );

  const handleDelete = (id: number) => {
    setUsers(users.filter(u => u.id !== id));
  };

  return (
    <UserListPresentation
      users={filteredUsers}
      loading={loading}
      searchTerm={searchTerm}
      onSearchChange={setSearchTerm}
      onDelete={handleDelete}
    />
  );
}

// UserListPresentation.tsx - UI
interface UserListPresentationProps {
  users: User[];
  loading: boolean;
  searchTerm: string;
  onSearchChange: (value: string) => void;
  onDelete: (id: number) => void;
}

function UserListPresentation({
  users,
  loading,
  searchTerm,
  onSearchChange,
  onDelete
}: UserListPresentationProps) {
  if (loading) return <div>Loading...</div>;

  return (
    <div>
      <input
        type="text"
        placeholder="Search users..."
        value={searchTerm}
        onChange={(e) => onSearchChange(e.target.value)}
      />

      <ul>
        {users.map(user => (
          <li key={user.id}>
            <span>{user.name} - {user.email}</span>
            <button onClick={() => onDelete(user.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## Custom Hook Pattern

### Reusable Logic

```typescript
// useLocalStorage.ts
function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = React.useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  };

  return [storedValue, setValue] as const;
}

// Usage
function TodoApp() {
  const [todos, setTodos] = useLocalStorage<string[]>('todos', []);

  const addTodo = (todo: string) => {
    setTodos(prev => [...prev, todo]);
  };

  return (
    <div>
      {todos.map((todo, i) => (
        <div key={i}>{todo}</div>
      ))}
      <button onClick={() => addTodo('New Todo')}>Add</button>
    </div>
  );
}
```

---

## State Reducer Pattern

### Flexible State Management

```typescript
type Action =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'SET'; payload: number }
  | { type: 'RESET' };

interface State {
  count: number;
}

const initialState: State = { count: 0 };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };
    case 'DECREMENT':
      return { count: state.count - 1 };
    case 'SET':
      return { count: action.payload };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}

interface CounterProps {
  reducer?: typeof reducer;
  initialState?: State;
}

function Counter({ reducer: customReducer, initialState: customInitialState }: CounterProps) {
  const [state, dispatch] = React.useReducer(
    customReducer || reducer,
    customInitialState || initialState
  );

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
      <button onClick={() => dispatch({ type: 'RESET' })}>Reset</button>
    </div>
  );
}

// Custom reducer with max limit
function limitedReducer(state: State, action: Action): State {
  const newState = reducer(state, action);
  return {
    ...newState,
    count: Math.min(Math.max(newState.count, 0), 10)
  };
}

function LimitedCounter() {
  return <Counter reducer={limitedReducer} />;
}
```

---

## Control Props Pattern

### Controlled vs Uncontrolled

```typescript
interface ToggleProps {
  // Controlled props
  on?: boolean;
  onChange?: (on: boolean) => void;
  // Uncontrolled props
  defaultOn?: boolean;
}

function Toggle({ on: controlledOn, onChange, defaultOn = false }: ToggleProps) {
  const [uncontrolledOn, setUncontrolledOn] = React.useState(defaultOn);

  // Determine if controlled
  const isControlled = controlledOn !== undefined;
  const on = isControlled ? controlledOn : uncontrolledOn;

  const handleToggle = () => {
    if (isControlled) {
      onChange?.(!controlledOn);
    } else {
      setUncontrolledOn(prev => !prev);
      onChange?.(!uncontrolledOn);
    }
  };

  return (
    <button onClick={handleToggle}>
      {on ? 'ON' : 'OFF'}
    </button>
  );
}

// Uncontrolled usage
function UncontrolledExample() {
  return <Toggle defaultOn={false} onChange={(on) => console.log(on)} />;
}

// Controlled usage
function ControlledExample() {
  const [on, setOn] = React.useState(false);
  return <Toggle on={on} onChange={setOn} />;
}
```

---

## Key Takeaways

1. **Compound components** share implicit state through context
2. **Render props** provide maximum flexibility for rendering logic
3. **Provider pattern** manages global/shared state with context
4. **HOCs** add behavior to components without modifying them
5. **Container/Presentational** separates logic from UI for testability
6. **Custom hooks** extract reusable stateful logic
7. **State reducer pattern** allows customization of state management
8. **Control props** support both controlled and uncontrolled modes
9. Choose patterns based on **flexibility, reusability, and complexity** needs
10. Modern React favors **hooks and composition** over HOCs and render props
