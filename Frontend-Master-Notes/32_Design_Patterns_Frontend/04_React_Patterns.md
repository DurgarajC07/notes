# React Design Patterns

## Core Concepts

React patterns are architectural approaches specifically designed for building React applications. These patterns address component composition, state management, performance optimization, and code reusability. Understanding these patterns is essential for building scalable, maintainable React applications and is frequently tested in senior-level interviews.

## Compound Components Pattern

### Basic Tabs Implementation

```typescript
import React from 'react';

interface TabsContextValue {
  activeTab: string;
  setActiveTab: (id: string) => void;
}

const TabsContext = React.createContext<TabsContextValue | undefined>(undefined);

function useTabs() {
  const context = React.useContext(TabsContext);
  if (!context) {
    throw new Error('Tabs compound components must be used within Tabs');
  }
  return context;
}

interface TabsProps {
  defaultTab?: string;
  children: React.ReactNode;
  onChange?: (tabId: string) => void;
}

function Tabs({ defaultTab, children, onChange }: TabsProps) {
  const [activeTab, setActiveTab] = React.useState(defaultTab || '');

  const handleTabChange = (tabId: string) => {
    setActiveTab(tabId);
    onChange?.(tabId);
  };

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab: handleTabChange }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

interface TabListProps {
  children: React.ReactNode;
}

function TabList({ children }: TabListProps) {
  return <div className="tab-list" role="tablist">{children}</div>;
}

interface TabProps {
  id: string;
  children: React.ReactNode;
  disabled?: boolean;
}

function Tab({ id, children, disabled }: TabProps) {
  const { activeTab, setActiveTab } = useTabs();
  const isActive = activeTab === id;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      aria-controls={`panel-${id}`}
      disabled={disabled}
      className={`tab ${isActive ? 'active' : ''}`}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

interface TabPanelsProps {
  children: React.ReactNode;
}

function TabPanels({ children }: TabPanelsProps) {
  return <div className="tab-panels">{children}</div>;
}

interface TabPanelProps {
  id: string;
  children: React.ReactNode;
}

function TabPanel({ id, children }: TabPanelProps) {
  const { activeTab } = useTabs();
  const isActive = activeTab === id;

  if (!isActive) return null;

  return (
    <div
      role="tabpanel"
      id={`panel-${id}`}
      className="tab-panel"
    >
      {children}
    </div>
  );
}

// Compose the compound component
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;

// Usage
function App() {
  return (
    <Tabs defaultTab="profile" onChange={(tabId) => console.log(tabId)}>
      <Tabs.List>
        <Tabs.Tab id="profile">Profile</Tabs.Tab>
        <Tabs.Tab id="settings">Settings</Tabs.Tab>
        <Tabs.Tab id="notifications">Notifications</Tabs.Tab>
      </Tabs.List>

      <Tabs.Panels>
        <Tabs.Panel id="profile">
          <h2>Profile Content</h2>
          <p>User profile information...</p>
        </Tabs.Panel>
        <Tabs.Panel id="settings">
          <h2>Settings Content</h2>
          <p>Application settings...</p>
        </Tabs.Panel>
        <Tabs.Panel id="notifications">
          <h2>Notifications Content</h2>
          <p>Your notifications...</p>
        </Tabs.Panel>
      </Tabs.Panels>
    </Tabs>
  );
}
```

### Advanced Accordion with Compound Components

```typescript
interface AccordionContextValue {
  openItems: Set<string>;
  toggleItem: (id: string) => void;
  allowMultiple: boolean;
}

const AccordionContext = React.createContext<AccordionContextValue | undefined>(undefined);

function useAccordion() {
  const context = React.useContext(AccordionContext);
  if (!context) {
    throw new Error('Accordion components must be used within Accordion');
  }
  return context;
}

interface AccordionProps {
  allowMultiple?: boolean;
  defaultOpen?: string[];
  children: React.ReactNode;
  onChange?: (openItems: string[]) => void;
}

function Accordion({ allowMultiple = false, defaultOpen = [], children, onChange }: AccordionProps) {
  const [openItems, setOpenItems] = React.useState(new Set(defaultOpen));

  const toggleItem = (id: string) => {
    setOpenItems(prev => {
      const next = new Set(prev);

      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!allowMultiple) {
          next.clear();
        }
        next.add(id);
      }

      onChange?.(Array.from(next));
      return next;
    });
  };

  return (
    <AccordionContext.Provider value={{ openItems, toggleItem, allowMultiple }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

interface AccordionItemProps {
  id: string;
  children: React.ReactNode;
}

function AccordionItem({ id, children }: AccordionItemProps) {
  const { openItems } = useAccordion();
  const isOpen = openItems.has(id);

  return (
    <div className={`accordion-item ${isOpen ? 'open' : ''}`}>
      {children}
    </div>
  );
}

interface AccordionTriggerProps {
  id: string;
  children: React.ReactNode;
}

function AccordionTrigger({ id, children }: AccordionTriggerProps) {
  const { openItems, toggleItem } = useAccordion();
  const isOpen = openItems.has(id);

  return (
    <button
      className="accordion-trigger"
      aria-expanded={isOpen}
      onClick={() => toggleItem(id)}
    >
      {children}
      <span className="accordion-icon">{isOpen ? '−' : '+'}</span>
    </button>
  );
}

interface AccordionContentProps {
  id: string;
  children: React.ReactNode;
}

function AccordionContent({ id, children }: AccordionContentProps) {
  const { openItems } = useAccordion();
  const isOpen = openItems.has(id);

  return (
    <div className={`accordion-content ${isOpen ? 'open' : ''}`}>
      {isOpen && children}
    </div>
  );
}

Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;

// Usage
function FAQ() {
  return (
    <Accordion allowMultiple defaultOpen={['q1']}>
      <Accordion.Item id="q1">
        <Accordion.Trigger id="q1">What is React?</Accordion.Trigger>
        <Accordion.Content id="q1">
          <p>React is a JavaScript library for building user interfaces.</p>
        </Accordion.Content>
      </Accordion.Item>

      <Accordion.Item id="q2">
        <Accordion.Trigger id="q2">What are hooks?</Accordion.Trigger>
        <Accordion.Content id="q2">
          <p>Hooks are functions that let you use state and lifecycle features.</p>
        </Accordion.Content>
      </Accordion.Item>
    </Accordion>
  );
}
```

## Render Props Pattern

### DataFetcher Component

```typescript
interface DataFetcherProps<T> {
  url: string;
  children: (state: {
    data: T | null;
    loading: boolean;
    error: Error | null;
    refetch: () => void;
  }) => React.ReactNode;
}

function DataFetcher<T>({ url, children }: DataFetcherProps<T>) {
  const [data, setData] = React.useState<T | null>(null);
  const [loading, setLoading] = React.useState(true);
  const [error, setError] = React.useState<Error | null>(null);

  const fetchData = React.useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch(url);
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      const json = await response.json();
      setData(json);
    } catch (e) {
      setError(e as Error);
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

function UserList() {
  return (
    <DataFetcher<User[]> url="/api/users">
      {({ data, loading, error, refetch }) => {
        if (loading) return <div>Loading...</div>;
        if (error) return <div>Error: {error.message}</div>;
        if (!data) return null;

        return (
          <div>
            <button onClick={refetch}>Refresh</button>
            <ul>
              {data.map(user => (
                <li key={user.id}>{user.name} - {user.email}</li>
              ))}
            </ul>
          </div>
        );
      }}
    </DataFetcher>
  );
}
```

### Mouse Tracker with Render Props

```typescript
interface MouseTrackerProps {
  children: (position: { x: number; y: number }) => React.ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
  const [position, setPosition] = React.useState({ x: 0, y: 0 });

  React.useEffect(() => {
    const handleMouseMove = (e: MouseEvent) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };

    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);

  return <>{children(position)}</>;
}

// Usage
function App() {
  return (
    <MouseTracker>
      {({ x, y }) => (
        <div>
          <h1>Mouse Position Tracker</h1>
          <p>X: {x}, Y: {y}</p>
          <div
            style={{
              position: 'absolute',
              left: x - 10,
              top: y - 10,
              width: 20,
              height: 20,
              borderRadius: '50%',
              background: 'red',
              pointerEvents: 'none',
            }}
          />
        </div>
      )}
    </MouseTracker>
  );
}
```

## Higher-Order Components (HOC)

### withAuth HOC

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  role: string;
}

interface WithAuthProps {
  user: User | null;
  isAuthenticated: boolean;
  logout: () => void;
}

function withAuth<P extends WithAuthProps>(
  Component: React.ComponentType<P>
): React.FC<Omit<P, keyof WithAuthProps>> {
  return (props) => {
    const [user, setUser] = React.useState<User | null>(null);
    const [loading, setLoading] = React.useState(true);

    React.useEffect(() => {
      // Check authentication status
      const checkAuth = async () => {
        try {
          const response = await fetch('/api/auth/me');
          if (response.ok) {
            const userData = await response.json();
            setUser(userData);
          }
        } catch (error) {
          console.error('Auth check failed:', error);
        } finally {
          setLoading(false);
        }
      };

      checkAuth();
    }, []);

    const logout = () => {
      setUser(null);
      // Call logout API
    };

    if (loading) {
      return <div>Loading...</div>;
    }

    if (!user) {
      return <Navigate to="/login" />;
    }

    return (
      <Component
        {...(props as P)}
        user={user}
        isAuthenticated={true}
        logout={logout}
      />
    );
  };
}

// Usage
interface DashboardProps extends WithAuthProps {
  title: string;
}

function Dashboard({ user, isAuthenticated, logout, title }: DashboardProps) {
  return (
    <div>
      <h1>{title}</h1>
      <p>Welcome, {user?.name}!</p>
      <button onClick={logout}>Logout</button>
    </div>
  );
}

const ProtectedDashboard = withAuth(Dashboard);

// Use in app
<ProtectedDashboard title="My Dashboard" />;
```

### withLoading HOC

```typescript
interface WithLoadingProps {
  isLoading?: boolean;
}

function withLoading<P extends object>(
  Component: React.ComponentType<P>,
  LoadingComponent: React.ComponentType = () => <div>Loading...</div>
): React.FC<P & WithLoadingProps> {
  return ({ isLoading, ...props }: WithLoadingProps & P) => {
    if (isLoading) {
      return <LoadingComponent />;
    }

    return <Component {...(props as P)} />;
  };
}

// Usage
interface UserListProps {
  users: User[];
}

function UserList({ users }: UserListProps) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}

const UserListWithLoading = withLoading(UserList);

// Use in app
<UserListWithLoading users={users} isLoading={loading} />;
```

### Composing Multiple HOCs

```typescript
interface WithLoggingProps {
  componentName: string;
}

function withLogging<P extends object>(
  Component: React.ComponentType<P>
): React.FC<P & WithLoggingProps> {
  return (props) => {
    React.useEffect(() => {
      console.log(`${props.componentName} mounted`);
      return () => console.log(`${props.componentName} unmounted`);
    }, [props.componentName]);

    return <Component {...(props as P)} />;
  };
}

function withErrorBoundary<P extends object>(
  Component: React.ComponentType<P>
): React.ComponentType<P> {
  return class extends React.Component<P, { hasError: boolean }> {
    state = { hasError: false };

    static getDerivedStateFromError() {
      return { hasError: true };
    }

    componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
      console.error('Error:', error, errorInfo);
    }

    render() {
      if (this.state.hasError) {
        return <div>Something went wrong</div>;
      }

      return <Component {...this.props} />;
    }
  };
}

// Compose multiple HOCs
const EnhancedComponent = withAuth(
  withLoading(
    withLogging(
      withErrorBoundary(Dashboard)
    )
  )
);
```

## Custom Hooks Pattern

### useLocalStorage Hook

```typescript
function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void, () => void] {
  const [storedValue, setStoredValue] = React.useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error('Error reading from localStorage:', error);
      return initialValue;
    }
  });

  const setValue = (value: T | ((prev: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error('Error writing to localStorage:', error);
    }
  };

  const removeValue = () => {
    try {
      window.localStorage.removeItem(key);
      setStoredValue(initialValue);
    } catch (error) {
      console.error('Error removing from localStorage:', error);
    }
  };

  return [storedValue, setValue, removeValue];
}

// Usage
function UserSettings() {
  const [theme, setTheme, removeTheme] = useLocalStorage('theme', 'light');
  const [language, setLanguage] = useLocalStorage('language', 'en');

  return (
    <div>
      <select value={theme} onChange={(e) => setTheme(e.target.value)}>
        <option value="light">Light</option>
        <option value="dark">Dark</option>
      </select>

      <select value={language} onChange={(e) => setLanguage(e.target.value)}>
        <option value="en">English</option>
        <option value="es">Spanish</option>
      </select>

      <button onClick={removeTheme}>Reset Theme</button>
    </div>
  );
}
```

### useFetch Hook

```typescript
interface UseFetchOptions {
  method?: string;
  headers?: Record<string, string>;
  body?: any;
}

interface UseFetchReturn<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

function useFetch<T>(url: string, options?: UseFetchOptions): UseFetchReturn<T> {
  const [data, setData] = React.useState<T | null>(null);
  const [loading, setLoading] = React.useState(true);
  const [error, setError] = React.useState<Error | null>(null);

  const fetchData = React.useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch(url, {
        method: options?.method || 'GET',
        headers: {
          'Content-Type': 'application/json',
          ...options?.headers,
        },
        body: options?.body ? JSON.stringify(options.body) : undefined,
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const json = await response.json();
      setData(json);
    } catch (e) {
      setError(e as Error);
    } finally {
      setLoading(false);
    }
  }, [url, options?.method, options?.headers, options?.body]);

  React.useEffect(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
}

// Usage
function UserProfile({ userId }: { userId: string }) {
  const { data: user, loading, error, refetch } = useFetch<User>(
    `/api/users/${userId}`
  );

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!user) return null;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={refetch}>Refresh</button>
    </div>
  );
}
```

### useDebounce Hook

```typescript
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = React.useState(value);

  React.useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
}

// Usage with search
function SearchComponent() {
  const [searchTerm, setSearchTerm] = React.useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 500);

  const { data: results } = useFetch<any[]>(
    `/api/search?q=${debouncedSearchTerm}`,
    { method: 'GET' }
  );

  return (
    <div>
      <input
        type="text"
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search..."
      />

      {results && (
        <ul>
          {results.map((result, index) => (
            <li key={index}>{result.title}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### usePrevious Hook

```typescript
function usePrevious<T>(value: T): T | undefined {
  const ref = React.useRef<T>();

  React.useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

// Usage
function Counter() {
  const [count, setCount] = React.useState(0);
  const previousCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {previousCount}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

## Provider Pattern

### Theme Provider

```typescript
interface Theme {
  colors: {
    primary: string;
    secondary: string;
    background: string;
    text: string;
  };
  fonts: {
    body: string;
    heading: string;
  };
  spacing: {
    small: string;
    medium: string;
    large: string;
  };
}

const lightTheme: Theme = {
  colors: {
    primary: '#007bff',
    secondary: '#6c757d',
    background: '#ffffff',
    text: '#000000',
  },
  fonts: {
    body: 'Arial, sans-serif',
    heading: 'Georgia, serif',
  },
  spacing: {
    small: '8px',
    medium: '16px',
    large: '24px',
  },
};

const darkTheme: Theme = {
  colors: {
    primary: '#0d6efd',
    secondary: '#6c757d',
    background: '#1a1a1a',
    text: '#ffffff',
  },
  fonts: {
    body: 'Arial, sans-serif',
    heading: 'Georgia, serif',
  },
  spacing: {
    small: '8px',
    medium: '16px',
    large: '24px',
  },
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
  const theme = isDark ? darkTheme : lightTheme;

  const toggleTheme = () => setIsDark(prev => !prev);

  return (
    <ThemeContext.Provider value={{ theme, isDark, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// Usage
function ThemedButton() {
  const { theme, toggleTheme } = useTheme();

  return (
    <button
      onClick={toggleTheme}
      style={{
        backgroundColor: theme.colors.primary,
        color: theme.colors.text,
        padding: theme.spacing.medium,
        fontFamily: theme.fonts.body,
      }}
    >
      Toggle Theme
    </button>
  );
}

function App() {
  return (
    <ThemeProvider>
      <ThemedButton />
    </ThemeProvider>
  );
}
```

### Auth Provider

```typescript
interface AuthContextValue {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  signup: (email: string, password: string, name: string) => Promise<void>;
  isAuthenticated: boolean;
  isLoading: boolean;
}

const AuthContext = React.createContext<AuthContextValue | undefined>(undefined);

export function useAuth() {
  const context = React.useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

interface AuthProviderProps {
  children: React.ReactNode;
}

export function AuthProvider({ children }: AuthProviderProps) {
  const [user, setUser] = React.useState<User | null>(null);
  const [isLoading, setIsLoading] = React.useState(true);

  React.useEffect(() => {
    // Check if user is already logged in
    const checkAuth = async () => {
      try {
        const response = await fetch('/api/auth/me');
        if (response.ok) {
          const userData = await response.json();
          setUser(userData);
        }
      } catch (error) {
        console.error('Auth check failed:', error);
      } finally {
        setIsLoading(false);
      }
    };

    checkAuth();
  }, []);

  const login = async (email: string, password: string) => {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });

    if (!response.ok) {
      throw new Error('Login failed');
    }

    const userData = await response.json();
    setUser(userData);
  };

  const logout = async () => {
    await fetch('/api/auth/logout', { method: 'POST' });
    setUser(null);
  };

  const signup = async (email: string, password: string, name: string) => {
    const response = await fetch('/api/auth/signup', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password, name }),
    });

    if (!response.ok) {
      throw new Error('Signup failed');
    }

    const userData = await response.json();
    setUser(userData);
  };

  const value = {
    user,
    login,
    logout,
    signup,
    isAuthenticated: !!user,
    isLoading,
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

// Usage
function LoginForm() {
  const { login } = useAuth();
  const [email, setEmail] = React.useState('');
  const [password, setPassword] = React.useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      await login(email, password);
    } catch (error) {
      console.error('Login failed:', error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

## Container/Presentational Pattern

### UserList Example

```typescript
// Presentational Component (Pure UI)
interface UserListViewProps {
  users: User[];
  loading: boolean;
  error: Error | null;
  onUserClick: (userId: string) => void;
  onRefresh: () => void;
}

function UserListView({ users, loading, error, onUserClick, onRefresh }: UserListViewProps) {
  if (loading) {
    return <div className="loading">Loading users...</div>;
  }

  if (error) {
    return (
      <div className="error">
        <p>Error: {error.message}</p>
        <button onClick={onRefresh}>Retry</button>
      </div>
    );
  }

  return (
    <div className="user-list">
      <button onClick={onRefresh}>Refresh</button>
      <ul>
        {users.map(user => (
          <li key={user.id} onClick={() => onUserClick(user.id)}>
            <h3>{user.name}</h3>
            <p>{user.email}</p>
          </li>
        ))}
      </ul>
    </div>
  );
}

// Container Component (Logic)
function UserListContainer() {
  const { data: users, loading, error, refetch } = useFetch<User[]>('/api/users');
  const navigate = useNavigate();

  const handleUserClick = (userId: string) => {
    navigate(`/users/${userId}`);
  };

  return (
    <UserListView
      users={users || []}
      loading={loading}
      error={error}
      onUserClick={handleUserClick}
      onRefresh={refetch}
    />
  );
}
```

## Controlled vs Uncontrolled Components

### Controlled Component

```typescript
function ControlledForm() {
  const [formData, setFormData] = React.useState({
    username: '',
    email: '',
    password: '',
  });

  const handleChange = (field: keyof typeof formData) => (
    e: React.ChangeEvent<HTMLInputElement>
  ) => {
    setFormData(prev => ({
      ...prev,
      [field]: e.target.value,
    }));
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log('Submitting:', formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={formData.username}
        onChange={handleChange('username')}
        placeholder="Username"
      />
      <input
        type="email"
        value={formData.email}
        onChange={handleChange('email')}
        placeholder="Email"
      />
      <input
        type="password"
        value={formData.password}
        onChange={handleChange('password')}
        placeholder="Password"
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Uncontrolled Component

```typescript
function UncontrolledForm() {
  const usernameRef = React.useRef<HTMLInputElement>(null);
  const emailRef = React.useRef<HTMLInputElement>(null);
  const passwordRef = React.useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    const formData = {
      username: usernameRef.current?.value || '',
      email: emailRef.current?.value || '',
      password: passwordRef.current?.value || '',
    };

    console.log('Submitting:', formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        ref={usernameRef}
        type="text"
        defaultValue=""
        placeholder="Username"
      />
      <input
        ref={emailRef}
        type="email"
        defaultValue=""
        placeholder="Email"
      />
      <input
        ref={passwordRef}
        type="password"
        defaultValue=""
        placeholder="Password"
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Real-World Usage

### Complete Dashboard with Multiple Patterns

```typescript
// Using Provider, Custom Hooks, Compound Components, and HOC together
function Dashboard() {
  return (
    <AuthProvider>
      <ThemeProvider>
        <DashboardContent />
      </ThemeProvider>
    </AuthProvider>
  );
}

const DashboardContent = withAuth(function DashboardContentComponent({ user }: WithAuthProps) {
  const { theme } = useTheme();
  const [selectedTab, setSelectedTab] = useLocalStorage('dashboard-tab', 'overview');

  return (
    <div style={{ backgroundColor: theme.colors.background, color: theme.colors.text }}>
      <h1>Welcome, {user?.name}!</h1>

      <Tabs defaultTab={selectedTab} onChange={setSelectedTab}>
        <Tabs.List>
          <Tabs.Tab id="overview">Overview</Tabs.Tab>
          <Tabs.Tab id="analytics">Analytics</Tabs.Tab>
          <Tabs.Tab id="settings">Settings</Tabs.Tab>
        </Tabs.List>

        <Tabs.Panels>
          <Tabs.Panel id="overview">
            <OverviewPanel />
          </Tabs.Panel>
          <Tabs.Panel id="analytics">
            <AnalyticsPanel />
          </Tabs.Panel>
          <Tabs.Panel id="settings">
            <SettingsPanel />
          </Tabs.Panel>
        </Tabs.Panels>
      </Tabs>
    </div>
  );
});

function OverviewPanel() {
  const { data: stats, loading } = useFetch<any>('/api/dashboard/stats');

  return (
    <DataFetcher url="/api/dashboard/overview">
      {({ data, loading, error }) => {
        if (loading) return <div>Loading...</div>;
        if (error) return <div>Error loading overview</div>;

        return (
          <div>
            <h2>Dashboard Overview</h2>
            {/* Render overview data */}
          </div>
        );
      }}
    </DataFetcher>
  );
}
```

## Production Patterns

### Error Boundary with Fallback UI

```typescript
interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ReactNode;
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    this.props.onError?.(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div>
          <h2>Something went wrong</h2>
          <details>
            <summary>Error details</summary>
            <pre>{this.state.error?.message}</pre>
          </details>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
<ErrorBoundary
  fallback={<ErrorFallback />}
  onError={(error, errorInfo) => {
    // Log to error tracking service
    console.error(error, errorInfo);
  }}
>
  <App />
</ErrorBoundary>
```

## Best Practices

1. **Use Compound Components** for related UI elements that work together
2. **Prefer Custom Hooks** over HOCs and Render Props for reusability
3. **Provider Pattern** for global state that many components need
4. **Container/Presentational** separation for testability and reusability
5. **Controlled Components** for form validation and complex interactions
6. **Uncontrolled Components** for simple forms and performance
7. **Memoization** with React.memo for expensive renders
8. **Error Boundaries** around major application sections
9. **Type safety** with TypeScript for pattern implementations
10. **Composition over Inheritance** - React's core philosophy

## 10 Key Takeaways

1. **Compound Components** share state through Context for flexible composition
2. **Render Props** provide maximum flexibility but can lead to callback hell
3. **HOCs** add behavior to components but can cause prop collision
4. **Custom Hooks** are the modern, preferred way to share logic
5. **Provider Pattern** manages global state without prop drilling
6. **Container/Presentational** separates logic from UI for better testing
7. **Controlled Components** give full control, uncontrolled are simpler
8. **React patterns** solve specific problems - choose based on use case
9. **Composition** is more powerful than inheritance in React
10. **TypeScript** makes pattern implementation safer and more maintainable
