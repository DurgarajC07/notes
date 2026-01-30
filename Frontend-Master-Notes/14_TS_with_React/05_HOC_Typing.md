# Higher-Order Component (HOC) Typing

## Core Concept

Higher-Order Components (HOCs) are functions that take a component and return a new component with additional props or behavior. TypeScript requires careful typing to preserve prop types and enable type inference.

---

## Basic HOC

```typescript
import { ComponentType } from 'react';

// Simple HOC that adds a prop
interface WithLoadingProps {
  isLoading: boolean;
}

function withLoading<P extends object>(
  Component: ComponentType<P>
) {
  return function WithLoadingComponent(
    props: P & WithLoadingProps
  ) {
    const { isLoading, ...rest } = props;

    if (isLoading) {
      return <div>Loading...</div>;
    }

    return <Component {...(rest as P)} />;
  };
}

// Usage
interface UserProps {
  name: string;
  email: string;
}

function User({ name, email }: UserProps) {
  return <div>{name} - {email}</div>;
}

const UserWithLoading = withLoading(User);

// Type-safe usage
<UserWithLoading name="John" email="john@example.com" isLoading={false} />
```

---

## HOC with Injected Props

```typescript
// HOC that injects props (removes them from required props)
interface InjectedAuthProps {
  user: User | null;
  logout: () => void;
}

function withAuth<P extends InjectedAuthProps>(
  Component: ComponentType<P>
) {
  return function WithAuthComponent(
    props: Omit<P, keyof InjectedAuthProps>
  ) {
    const { user, logout } = useAuth(); // Custom hook

    return <Component {...(props as P)} user={user} logout={logout} />;
  };
}

// Usage
interface ProfileProps extends InjectedAuthProps {
  theme: string;
}

function Profile({ user, logout, theme }: ProfileProps) {
  return (
    <div>
      <div>User: {user?.name}</div>
      <div>Theme: {theme}</div>
      <button onClick={logout}>Logout</button>
    </div>
  );
}

const ProfileWithAuth = withAuth(Profile);

// user and logout are injected - don't pass them!
<ProfileWithAuth theme="dark" />
```

---

## HOC with Optional Props

```typescript
interface WithAnalyticsOptions {
  trackPageView?: boolean;
  trackClicks?: boolean;
}

function withAnalytics<P extends object>(
  Component: ComponentType<P>,
  options: WithAnalyticsOptions = {}
) {
  return function WithAnalyticsComponent(props: P) {
    useEffect(() => {
      if (options.trackPageView) {
        analytics.trackPageView();
      }
    }, []);

    const handleClick = (e: MouseEvent) => {
      if (options.trackClicks) {
        analytics.trackClick(e.target);
      }
    };

    return (
      <div onClick={handleClick}>
        <Component {...props} />
      </div>
    );
  };
}

// Usage
const UserWithAnalytics = withAnalytics(User, {
  trackPageView: true,
  trackClicks: true
});
```

---

## HOC with Generic Props

```typescript
// HOC that works with generic components
interface WithDataProps<T> {
  data: T[];
  isLoading: boolean;
  error: Error | null;
}

function withData<T, P extends WithDataProps<T>>(
  Component: ComponentType<P>,
  fetchData: () => Promise<T[]>
) {
  return function WithDataComponent(props: Omit<P, keyof WithDataProps<T>>) {
    const [data, setData] = useState<T[]>([]);
    const [isLoading, setIsLoading] = useState(true);
    const [error, setError] = useState<Error | null>(null);

    useEffect(() => {
      fetchData()
        .then(setData)
        .catch(setError)
        .finally(() => setIsLoading(false));
    }, []);

    return (
      <Component
        {...(props as P)}
        data={data}
        isLoading={isLoading}
        error={error}
      />
    );
  };
}

// Usage
interface User {
  id: string;
  name: string;
}

interface UserListProps extends WithDataProps<User> {
  onUserClick: (user: User) => void;
}

function UserList({ data, isLoading, error, onUserClick }: UserListProps) {
  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {data.map(user => (
        <li key={user.id} onClick={() => onUserClick(user)}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}

const UserListWithData = withData(UserList, fetchUsers);
```

---

## Preserving Display Name

```typescript
function withDisplayName<P extends object>(
  Component: ComponentType<P>
) {
  function WithDisplayNameComponent(props: P) {
    return <Component {...props} />;
  }

  // Preserve component name for debugging
  WithDisplayNameComponent.displayName = `withDisplayName(${
    Component.displayName || Component.name || 'Component'
  })`;

  return WithDisplayNameComponent;
}
```

---

## Composing Multiple HOCs

```typescript
import { compose } from "redux"; // or implement your own

// Individual HOCs
const withAuth = <P extends object>(Component: ComponentType<P>) => {
  /* ... */
};

const withLogging = <P extends object>(Component: ComponentType<P>) => {
  /* ... */
};

const withTheme = <P extends object>(Component: ComponentType<P>) => {
  /* ... */
};

// Compose HOCs
const enhance = compose(withAuth, withLogging, withTheme);

const EnhancedComponent = enhance(MyComponent);

// Or manually
const EnhancedComponent = withAuth(withLogging(withTheme(MyComponent)));
```

---

## HOC with Ref Forwarding

```typescript
import { forwardRef, ComponentType, ForwardedRef } from 'react';

interface ButtonProps {
  label: string;
}

function withLogger<P extends object>(
  Component: ComponentType<P>
) {
  return forwardRef<HTMLButtonElement, P>((props, ref) => {
    useEffect(() => {
      console.log('Component rendered with props:', props);
    });

    return <Component ref={ref} {...props} />;
  });
}

// Usage with forwarded ref
const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ label }, ref) => {
    return <button ref={ref}>{label}</button>;
  }
);

const ButtonWithLogger = withLogger(Button);

// Can use ref
const ref = useRef<HTMLButtonElement>(null);
<ButtonWithLogger ref={ref} label="Click" />
```

---

## HOC with Context

```typescript
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

interface WithThemeProps {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

function withTheme<P extends WithThemeProps>(
  Component: ComponentType<P>
) {
  return function WithThemeComponent(
    props: Omit<P, keyof WithThemeProps>
  ) {
    const context = useContext(ThemeContext);

    if (!context) {
      throw new Error('withTheme must be used within ThemeProvider');
    }

    return (
      <Component
        {...(props as P)}
        theme={context.theme}
        toggleTheme={context.toggleTheme}
      />
    );
  };
}
```

---

## Conditional HOC

```typescript
// Apply HOC conditionally
function withFeatureFlag<P extends object>(
  Component: ComponentType<P>,
  featureFlag: string
) {
  return function WithFeatureFlagComponent(props: P) {
    const isEnabled = useFeatureFlag(featureFlag);

    if (!isEnabled) {
      return <div>Feature not available</div>;
    }

    return <Component {...props} />;
  };
}

// Or return original component if condition not met
function maybeWithAuth<P extends object>(
  Component: ComponentType<P>,
  requireAuth: boolean
) {
  if (!requireAuth) {
    return Component; // Return original
  }

  return withAuth(Component); // Apply HOC
}
```

---

## HOC with Props Transformation

```typescript
interface ExternalProps {
  userId: string;
}

interface InternalProps {
  user: User;
  isLoading: boolean;
}

function withUserData<P extends InternalProps>(
  Component: ComponentType<P>
) {
  return function WithUserDataComponent(
    props: Omit<P, keyof InternalProps> & ExternalProps
  ) {
    const { userId, ...rest } = props;
    const { user, isLoading } = useFetchUser(userId);

    return (
      <Component
        {...(rest as P)}
        user={user}
        isLoading={isLoading}
      />
    );
  };
}

// Transforms userId prop into user object
interface ProfileProps extends InternalProps {
  theme: string;
}

function Profile({ user, isLoading, theme }: ProfileProps) {
  if (isLoading) return <div>Loading...</div>;
  return <div>{user.name} - {theme}</div>;
}

const ProfileWithData = withUserData(Profile);

// Pass userId, get user object injected
<ProfileWithData userId="123" theme="dark" />
```

---

## Real-World: Permission HOC

```typescript
type Permission = 'read' | 'write' | 'delete';

interface WithPermissionProps {
  permissions: Permission[];
}

function withPermission<P extends object>(
  Component: ComponentType<P>,
  requiredPermissions: Permission[]
) {
  return function WithPermissionComponent(
    props: P & WithPermissionProps
  ) {
    const { permissions, ...rest } = props;

    const hasPermission = requiredPermissions.every(perm =>
      permissions.includes(perm)
    );

    if (!hasPermission) {
      return <div>Access Denied</div>;
    }

    return <Component {...(rest as P)} />;
  };
}

// Usage
interface AdminPanelProps {
  onSave: () => void;
}

function AdminPanel({ onSave }: AdminPanelProps) {
  return <button onClick={onSave}>Save</button>;
}

const ProtectedAdminPanel = withPermission(AdminPanel, ['write', 'delete']);

// Must pass permissions
<ProtectedAdminPanel
  permissions={['read', 'write', 'delete']}
  onSave={handleSave}
/>
```

---

## Type-Safe HOC Factory

```typescript
// Create HOC with type-safe configuration
interface HOCConfig<InjectedProps> {
  mapPropsToInjected: (props: any) => InjectedProps;
}

function createHOC<InjectedProps extends object>(
  config: HOCConfig<InjectedProps>
) {
  return function withInjectedProps<P extends InjectedProps>(
    Component: ComponentType<P>
  ) {
    return function WithInjectedPropsComponent(
      props: Omit<P, keyof InjectedProps>
    ) {
      const injectedProps = config.mapPropsToInjected(props);

      return <Component {...(props as P)} {...injectedProps} />;
    };
  };
}

// Usage
const withFormattedDate = createHOC<{ formattedDate: string }>({
  mapPropsToInjected: (props: { date: Date }) => ({
    formattedDate: props.date.toLocaleDateString()
  })
});
```

---

## Best Practices

✅ **Use generics properly** for type safety  
✅ **Preserve component display name**  
✅ **Forward refs when needed**  
✅ **Document injected props** clearly  
✅ **Use `Omit` for injected props**  
✅ **Compose HOCs with compose utility**  
❌ **Don't overuse HOCs** - prefer hooks  
❌ **Don't mutate component** in HOC  
❌ **Don't forget static methods**

---

## Key Takeaways

1. **HOCs add behavior without modifying components**
2. **Use `Omit` to remove injected props** from type
3. **Generics enable type preservation**
4. **Forward refs for DOM access**
5. **Compose multiple HOCs** for complex behavior
6. **Hooks often better** than HOCs in modern React
7. **TypeScript catches prop mismatches**
