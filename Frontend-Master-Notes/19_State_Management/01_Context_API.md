# 🔄 Context API Deep Dive

> React Context provides a way to share data across the component tree without prop drilling. Understanding Context is essential for state management in medium to large React applications.

---

## 📖 1. Concept Explanation

### What is Context?

**Context** allows passing data through the component tree without manually passing props at every level.

```jsx
// ❌ WITHOUT Context: Prop drilling
function App() {
  const [user, setUser] = useState(null);

  return <Layout user={user} setUser={setUser} />;
}

function Layout({ user, setUser }) {
  return <Header user={user} setUser={setUser} />;
}

function Header({ user, setUser }) {
  return <UserMenu user={user} setUser={setUser} />;
}

function UserMenu({ user, setUser }) {
  // Finally used here!
  return <div>{user?.name}</div>;
}

// ✅ WITH Context: Direct access
const UserContext = createContext(null);

function App() {
  const [user, setUser] = useState(null);

  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Layout />
    </UserContext.Provider>
  );
}

function UserMenu() {
  const { user } = useContext(UserContext);
  return <div>{user?.name}</div>;
}
```

### Core API

**1. createContext:**

```jsx
const MyContext = createContext(defaultValue);
```

**2. Provider:**

```jsx
<MyContext.Provider value={/* some value */}>
  {children}
</MyContext.Provider>
```

**3. useContext:**

```jsx
const value = useContext(MyContext);
```

---

## 🧠 2. Why It Matters

### Production Use Cases

**1. Theme Management:**

```jsx
const ThemeContext = createContext("light");

function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <AppContent />
    </ThemeContext.Provider>
  );
}

function Button() {
  const { theme } = useContext(ThemeContext);

  return (
    <button className={theme === "dark" ? "btn-dark" : "btn-light"}>
      Click me
    </button>
  );
}
```

**2. Authentication:**

```jsx
const AuthContext = createContext(null);

function App() {
  const [user, setUser] = useState(null);

  const login = async (credentials) => {
    const user = await api.login(credentials);
    setUser(user);
  };

  const logout = () => {
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      <AppRoutes />
    </AuthContext.Provider>
  );
}

function ProtectedRoute({ children }) {
  const { user } = useContext(AuthContext);

  if (!user) {
    return <Navigate to="/login" />;
  }

  return children;
}
```

**3. Internationalization:**

```jsx
const LocaleContext = createContext("en");

function App() {
  const [locale, setLocale] = useState("en");

  return (
    <LocaleContext.Provider value={{ locale, setLocale }}>
      <AppContent />
    </LocaleContext.Provider>
  );
}

function Text({ id }) {
  const { locale } = useContext(LocaleContext);
  return <span>{translations[locale][id]}</span>;
}
```

---

## ⚙️ 3. Internal Working

### How Context Works

**1. Context creation:**

```javascript
// Simplified React internals
function createContext(defaultValue) {
  const context = {
    $$typeof: Symbol.for("react.context"),
    _currentValue: defaultValue,
    Provider: null,
    Consumer: null,
  };

  context.Provider = {
    $$typeof: Symbol.for("react.provider"),
    _context: context,
  };

  context.Consumer = context;

  return context;
}
```

**2. Provider component:**

```javascript
// When Provider renders
function renderProvider(Provider, newValue) {
  const context = Provider._context;

  // Update context value
  context._currentValue = newValue;

  // Notify all consumers to re-render
  propagateContextChange(context);
}
```

**3. Consumer (useContext):**

```javascript
// When component uses context
function useContext(Context) {
  // Read current value
  return Context._currentValue;
}
```

### Re-render Behavior

**Context change triggers re-renders:**

```jsx
const CountContext = createContext(0);

function App() {
  const [count, setCount] = useState(0);

  return (
    <CountContext.Provider value={count}>
      <Child1 />
      <Child2 />
    </CountContext.Provider>
  );
}

function Child1() {
  const count = useContext(CountContext);
  console.log("Child1 render"); // Re-renders when count changes
  return <div>Count: {count}</div>;
}

function Child2() {
  console.log("Child2 render"); // ALSO re-renders! (under Provider)
  return <div>I don't use context</div>;
}
```

---

## ✅ 4. Best Practices

### DO ✅

```jsx
// 1. Separate contexts by concern
const ThemeContext = createContext(null);
const AuthContext = createContext(null);
const LocaleContext = createContext(null);

// Not one giant context
// ❌ const AppContext = createContext({ theme, auth, locale });

// 2. Create custom hooks
function useTheme() {
  const context = useContext(ThemeContext);

  if (!context) {
    throw new Error("useTheme must be used within ThemeProvider");
  }

  return context;
}

// Usage
function Button() {
  const { theme, setTheme } = useTheme(); // Type-safe, validated
  return <button>{theme}</button>;
}

// 3. Memoize context value
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  // ✅ Memoize to prevent unnecessary re-renders
  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
  );
}

// 4. Split context for optimization
const UserContext = createContext(null);
const UserDispatchContext = createContext(null);

function UserProvider({ children }) {
  const [user, setUser] = useState(null);

  // Components that only need dispatch won't re-render on user change
  return (
    <UserContext.Provider value={user}>
      <UserDispatchContext.Provider value={setUser}>
        {children}
      </UserDispatchContext.Provider>
    </UserContext.Provider>
  );
}

function useUser() {
  return useContext(UserContext);
}

function useUserDispatch() {
  return useContext(UserDispatchContext);
}

// 5. Provide default values
const ThemeContext = createContext({
  theme: "light",
  setTheme: () => {
    console.warn("ThemeContext not provided");
  },
});
```

### DON'T ❌

```jsx
// ❌ 1. Don't create new object on every render
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
  // New object every render → all consumers re-render!
}

// ❌ 2. Don't use Context for frequently changing values
// High-frequency updates (e.g., mouse position, scroll)
// Use local state or specialized libraries instead

// ❌ 3. Don't overuse Context
// Not everything needs Context
// Use props when component tree is shallow

// ❌ 4. Don't put all state in one Context
const AppContext = createContext({
  theme: "light",
  user: null,
  locale: "en",
  cart: [],
  // ... (giant context)
});
// Any change re-renders ALL consumers!

// ❌ 5. Don't forget error boundaries
// Context errors can crash entire app
function useTheme() {
  const context = useContext(ThemeContext);

  if (!context) {
    throw new Error("Missing ThemeProvider"); // Good!
    // But wrap in ErrorBoundary to catch
  }

  return context;
}
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Not Memoizing Context Value

```jsx
// ❌ WRONG: New object every render
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {children}
    </AuthContext.Provider>
  );
  // { user, setUser } is a NEW object every render
  // All consumers re-render unnecessarily!
}

// ✅ CORRECT: Memoize value
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const value = useMemo(
    () => ({ user, setUser }),
    [user], // Only create new object when user changes
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

### Mistake #2: Provider Hell

```jsx
// ❌ WRONG: Nested providers
function App() {
  return (
    <ThemeProvider>
      <AuthProvider>
        <LocaleProvider>
          <ToastProvider>
            <ModalProvider>
              <Routes /> {/* Hard to read! */}
            </ModalProvider>
          </ToastProvider>
        </LocaleProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

// ✅ CORRECT: Compose providers
function composeProviders(...providers) {
  return providers.reduce((Prev, Curr) => ({ children }) => (
    <Prev>
      <Curr>{children}</Curr>
    </Prev>
  ));
}

const AppProviders = composeProviders(
  ThemeProvider,
  AuthProvider,
  LocaleProvider,
  ToastProvider,
  ModalProvider,
);

function App() {
  return (
    <AppProviders>
      <Routes />
    </AppProviders>
  );
}
```

### Mistake #3: Putting Everything in Context

```jsx
// ❌ WRONG: Counter state in Context
const CounterContext = createContext(0);

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <CounterContext.Provider value={count}>
      <Child />
    </CounterContext.Provider>
  );
}

// ✅ CORRECT: Use local state
function Parent() {
  const [count, setCount] = useState(0);

  // If Child needs count, pass as prop
  return <Child count={count} />;
}

// Use Context only when:
// - Deep prop drilling (>3 levels)
// - Many components need the data
// - Data is truly global (theme, auth, locale)
```

---

## 🚀 6. Performance Optimization

### 1. Split Contexts

```jsx
// ❌ SLOW: Single context with multiple values
const AppContext = createContext(null);

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState("light");
  const [locale, setLocale] = useState("en");

  const value = useMemo(
    () => ({ user, setUser, theme, setTheme, locale, setLocale }),
    [user, theme, locale],
  );

  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
}

// Component that only uses theme
function Button() {
  const { theme } = useContext(AppContext);
  // Re-renders when user OR locale changes! (unnecessary)
  return <button className={theme}>Click</button>;
}

// ✅ FAST: Separate contexts
const ThemeContext = createContext(null);
const AuthContext = createContext(null);
const LocaleContext = createContext(null);

function Button() {
  const { theme } = useContext(ThemeContext);
  // Only re-renders when theme changes
  return <button className={theme}>Click</button>;
}
```

### 2. Memoize Expensive Calculations

```jsx
function DataProvider({ children }) {
  const [data, setData] = useState([]);

  // ✅ Expensive calculation memoized
  const sortedData = useMemo(() => {
    return [...data].sort((a, b) => a.value - b.value);
  }, [data]);

  const value = useMemo(
    () => ({ data, sortedData, setData }),
    [data, sortedData],
  );

  return <DataContext.Provider value={value}>{children}</DataContext.Provider>;
}
```

### 3. Use Selectors

```jsx
// ✅ Custom hook with selector
function useAppState(selector) {
  const state = useContext(AppContext);
  return selector(state);
}

// Usage: Only re-renders when selected value changes
function UserName() {
  const name = useAppState((state) => state.user.name);
  return <div>{name}</div>;
}

// Component won't re-render when other state changes
```

---

## 🧪 7. Real-World Examples

### Example 1: Theme Provider

```tsx
type Theme = "light" | "dark";

interface ThemeContextValue {
  theme: Theme;
  setTheme: (theme: Theme) => void;
}

const ThemeContext = createContext<ThemeContextValue | null>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>(() => {
    const saved = localStorage.getItem("theme");
    return saved === "dark" || saved === "light" ? saved : "light";
  });

  useEffect(() => {
    localStorage.setItem("theme", theme);
    document.documentElement.setAttribute("data-theme", theme);
  }, [theme]);

  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);

  if (!context) {
    throw new Error("useTheme must be used within ThemeProvider");
  }

  return context;
}

// Usage
function ThemeToggle() {
  const { theme, setTheme } = useTheme();

  return (
    <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
      {theme === "light" ? "🌙" : "☀️"}
    </button>
  );
}
```

### Example 2: Auth Provider

```tsx
interface User {
  id: string;
  name: string;
  email: string;
}

interface AuthContextValue {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  isLoading: boolean;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Check if user is logged in on mount
    const checkAuth = async () => {
      try {
        const user = await api.getCurrentUser();
        setUser(user);
      } catch (error) {
        setUser(null);
      } finally {
        setIsLoading(false);
      }
    };

    checkAuth();
  }, []);

  const login = async (email: string, password: string) => {
    const user = await api.login(email, password);
    setUser(user);
  };

  const logout = () => {
    api.logout();
    setUser(null);
  };

  const value = useMemo(
    () => ({ user, login, logout, isLoading }),
    [user, isLoading],
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const context = useContext(AuthContext);

  if (!context) {
    throw new Error("useAuth must be used within AuthProvider");
  }

  return context;
}

// Usage
function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { user, isLoading } = useAuth();

  if (isLoading) {
    return <Spinner />;
  }

  if (!user) {
    return <Navigate to="/login" />;
  }

  return <>{children}</>;
}
```

---

## ❓ 8. Interview Questions

### Q1: When should you use Context vs Redux?

**Answer:**

| Context                    | Redux                           |
| -------------------------- | ------------------------------- |
| Simple, infrequent updates | Complex state, frequent updates |
| Small to medium apps       | Large apps                      |
| Few consumers              | Many consumers                  |
| Built-in React             | External library                |

**Use Context for:**

- Theme, locale, authentication
- Infrequent updates
- Simple state sharing

**Use Redux for:**

- Complex state logic
- Time-travel debugging
- Middleware (logging, analytics)
- Performance-critical apps

---

### Q2: How do you optimize Context performance?

**Answer:**

**1. Split contexts:**

```jsx
// Don't put everything in one context
const ThemeContext = createContext(null);
const UserContext = createContext(null);
```

**2. Memoize value:**

```jsx
const value = useMemo(() => ({ user, setUser }), [user]);
```

**3. Separate state and dispatch:**

```jsx
const StateContext = createContext(null);
const DispatchContext = createContext(null);
```

**4. Use React.memo:**

```jsx
const Child = memo(function Child() {
  // Only re-renders when props change
});
```

---

## 🎯 Summary

Context enables sharing data without prop drilling:

- ✅ Use for global state (theme, auth, locale)
- ✅ Memoize context value
- ✅ Split contexts by concern
- ✅ Create custom hooks
- ✅ Provide default values

**Anti-patterns:**

- ❌ Don't use for high-frequency updates
- ❌ Don't create new object every render
- ❌ Don't put everything in one context
- ❌ Don't overuse (props are often better)

Master Context for production React apps! ⚛️
