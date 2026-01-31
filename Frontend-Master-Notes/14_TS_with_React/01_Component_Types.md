# React Component Types in TypeScript

## Core Concept

React component typing in TypeScript ensures **type-safe props**, **proper event handling**, and **compile-time validation** for components, hooks, and JSX elements.

---

## Functional Component Types

### **Basic Component with Props**

```typescript
import React from 'react';

interface UserProps {
  id: string;
  name: string;
  email: string;
  isActive?: boolean; // Optional prop
}

// Preferred: Explicit props typing
const UserCard = ({ id, name, email, isActive = true }: UserProps) => {
  return (
    <div className="user-card">
      <h3>{name}</h3>
      <p>{email}</p>
      {isActive && <span className="badge">Active</span>}
    </div>
  );
};

// Alternative: FC type (includes children implicitly)
const UserCard: React.FC<UserProps> = ({ id, name, email, isActive = true }) => {
  return (
    <div className="user-card">
      <h3>{name}</h3>
      <p>{email}</p>
      {isActive && <span className="badge">Active</span>}
    </div>
  );
};
```

### **Component with Explicit Return Type**

```typescript
import { ReactElement, JSX } from 'react';

interface Props {
  title: string;
  count: number;
}

// Explicit return type: ReactElement
const Counter = ({ title, count }: Props): ReactElement => {
  return (
    <div>
      <h2>{title}</h2>
      <span>{count}</span>
    </div>
  );
};

// Alternative: JSX.Element
const Counter = ({ title, count }: Props): JSX.Element => {
  return (
    <div>
      <h2>{title}</h2>
      <span>{count}</span>
    </div>
  );
};

// Nullable return type
const MaybeComponent = ({ show }: { show: boolean }): ReactElement | null => {
  if (!show) return null;
  return <div>Visible</div>;
};
```

### **With Children**

```typescript
import { ReactNode } from 'react';

interface CardProps {
  title: string;
  children: ReactNode; // Accepts any valid React child
}

const Card = ({ title, children }: CardProps) => {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="card-body">{children}</div>
    </div>
  );
};

// Usage
<Card title="Welcome">
  <p>This is card content</p>
  <button>Click me</button>
</Card>
```

### **PropsWithChildren Utility**

```typescript
import { PropsWithChildren } from 'react';

interface ContainerProps {
  className?: string;
  testId?: string;
}

// PropsWithChildren<T> automatically adds 'children?: ReactNode'
const Container = ({
  className,
  testId,
  children
}: PropsWithChildren<ContainerProps>) => {
  return (
    <section className={className} data-testid={testId}>
      {children}
    </section>
  );
};
```

### **Specific Children Types**

```typescript
import { ReactElement } from 'react';

interface TabsProps {
  children: ReactElement<TabProps> | ReactElement<TabProps>[]; // Only Tab components
}

interface TabProps {
  label: string;
  children: ReactNode;
}

const Tabs = ({ children }: TabsProps) => {
  const tabs = React.Children.toArray(children) as ReactElement<TabProps>[];

  return (
    <div className="tabs">
      {tabs.map((tab, index) => (
        <button key={index}>{tab.props.label}</button>
      ))}
    </div>
  );
};

const Tab = ({ label, children }: TabProps) => {
  return <div>{children}</div>;
};

// Usage - Type-safe: only Tab components allowed
<Tabs>
  <Tab label="Home">Home content</Tab>
  <Tab label="Profile">Profile content</Tab>
</Tabs>
```

---

## Event Handler Types

### **Common Event Types**

```typescript
import {
  MouseEvent,
  ChangeEvent,
  FormEvent,
  KeyboardEvent,
  FocusEvent,
  DragEvent
} from 'react';

interface FormProps {
  onSubmit: (data: { email: string; password: string }) => void;
}

const LoginForm = ({ onSubmit }: FormProps) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  // Input change handler
  const handleEmailChange = (e: ChangeEvent<HTMLInputElement>) => {
    setEmail(e.target.value); // e.target is HTMLInputElement
  };

  // Textarea change handler
  const handleTextAreaChange = (e: ChangeEvent<HTMLTextAreaElement>) => {
    console.log(e.target.value); // e.target is HTMLTextAreaElement
  };

  // Form submit handler
  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    onSubmit({ email, password });
  };

  // Button click handler
  const handleButtonClick = (e: MouseEvent<HTMLButtonElement>) => {
    console.log('Button clicked', e.currentTarget); // currentTarget is HTMLButtonElement
  };

  // Keyboard handler
  const handleKeyPress = (e: KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
      handleSubmit(e as unknown as FormEvent<HTMLFormElement>);
    }
  };

  // Focus handler
  const handleFocus = (e: FocusEvent<HTMLInputElement>) => {
    console.log('Input focused', e.target);
  };

  // Drag handler
  const handleDrag = (e: DragEvent<HTMLDivElement>) => {
    console.log('Dragging', e.clientX, e.clientY);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={handleEmailChange}
        onFocus={handleFocus}
        onKeyPress={handleKeyPress}
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <button type="submit" onClick={handleButtonClick}>
        Login
      </button>
    </form>
  );
};
```

### **Generic Event Handlers**

```typescript
// Generic change handler for any input element
const handleChange = <
  T extends HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement,
>(
  e: ChangeEvent<T>,
) => {
  const { name, value } = e.target;
  console.log(`${name}: ${value}`);
};

// Generic click handler
const handleClick = <T extends HTMLElement>(e: MouseEvent<T>) => {
  e.stopPropagation();
  console.log("Clicked element:", e.currentTarget.tagName);
};
```

---

## Generic Components

### **Basic Generic Component**

```typescript
import { ReactNode } from 'react';

interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => ReactNode;
  keyExtractor: (item: T) => string | number;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>
          {renderItem(item, index)}
        </li>
      ))}
    </ul>
  );
}

// Usage with User type
interface User {
  id: number;
  name: string;
  email: string;
}

const users: User[] = [
  { id: 1, name: 'Alice', email: 'alice@example.com' },
  { id: 2, name: 'Bob', email: 'bob@example.com' }
];

<List<User>
  items={users}
  keyExtractor={(user) => user.id}
  renderItem={(user, index) => (
    <div>
      <strong>{index + 1}. {user.name}</strong>
      <span>{user.email}</span>
    </div>
  )}
/>
```

### **Generic Component with Constraints**

```typescript
interface Identifiable {
  id: string | number;
}

interface TableProps<T extends Identifiable> {
  data: T[];
  columns: {
    key: keyof T;
    header: string;
    render?: (value: T[keyof T], item: T) => ReactNode;
  }[];
  onRowClick?: (item: T) => void;
}

function Table<T extends Identifiable>({
  data,
  columns,
  onRowClick
}: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={String(col.key)}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((item) => (
          <tr key={item.id} onClick={() => onRowClick?.(item)}>
            {columns.map((col) => (
              <td key={String(col.key)}>
                {col.render
                  ? col.render(item[col.key], item)
                  : String(item[col.key])
                }
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Usage
interface Product extends Identifiable {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
}

<Table<Product>
  data={products}
  columns={[
    { key: 'name', header: 'Product Name' },
    {
      key: 'price',
      header: 'Price',
      render: (value) => `$${value}`
    },
    {
      key: 'inStock',
      header: 'Status',
      render: (value) => value ? '✅ In Stock' : '❌ Out of Stock'
    }
  ]}
  onRowClick={(product) => console.log('Selected:', product.name)}
/>
```

### **Generic Form Component**

```typescript
interface FormProps<T> {
  initialValues: T;
  onSubmit: (values: T) => void;
  validate?: (values: T) => Partial<Record<keyof T, string>>;
  children: (props: {
    values: T;
    errors: Partial<Record<keyof T, string>>;
    handleChange: <K extends keyof T>(field: K, value: T[K]) => void;
    handleSubmit: () => void;
  }) => ReactNode;
}

function Form<T extends Record<string, any>>({
  initialValues,
  onSubmit,
  validate,
  children
}: FormProps<T>) {
  const [values, setValues] = useState<T>(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});

  const handleChange = <K extends keyof T>(field: K, value: T[K]) => {
    setValues(prev => ({ ...prev, [field]: value }));
    // Clear error when field changes
    setErrors(prev => {
      const newErrors = { ...prev };
      delete newErrors[field];
      return newErrors;
    });
  };

  const handleSubmit = () => {
    if (validate) {
      const validationErrors = validate(values);
      if (Object.keys(validationErrors).length > 0) {
        setErrors(validationErrors);
        return;
      }
    }
    onSubmit(values);
  };

  return (
    <>
      {children({ values, errors, handleChange, handleSubmit })}
    </>
  );
}

// Usage
interface LoginFormValues {
  email: string;
  password: string;
}

<Form<LoginFormValues>
  initialValues={{ email: '', password: '' }}
  validate={(values) => {
    const errors: Partial<Record<keyof LoginFormValues, string>> = {};
    if (!values.email.includes('@')) {
      errors.email = 'Invalid email';
    }
    if (values.password.length < 6) {
      errors.password = 'Password too short';
    }
    return errors;
  }}
  onSubmit={(values) => console.log('Submit:', values)}
>
  {({ values, errors, handleChange, handleSubmit }) => (
    <div>
      <input
        value={values.email}
        onChange={(e) => handleChange('email', e.target.value)}
      />
      {errors.email && <span className="error">{errors.email}</span>}

      <input
        type="password"
        value={values.password}
        onChange={(e) => handleChange('password', e.target.value)}
      />
      {errors.password && <span className="error">{errors.password}</span>}

      <button onClick={handleSubmit}>Login</button>
    </div>
  )}
</Form>
```

---

## Ref Types

### **DOM Element Refs**

```typescript
import { useRef, RefObject } from 'react';

const InputComponent = () => {
  // Ref for input element
  const inputRef = useRef<HTMLInputElement>(null);

  // Ref for div element
  const divRef = useRef<HTMLDivElement>(null);

  // Ref for button element
  const buttonRef = useRef<HTMLButtonElement>(null);

  const focusInput = () => {
    inputRef.current?.focus(); // Optional chaining for null safety
  };

  const scrollToDiv = () => {
    divRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  const getButtonWidth = () => {
    if (buttonRef.current) {
      return buttonRef.current.offsetWidth;
    }
    return 0;
  };

  return (
    <div>
      <input ref={inputRef} placeholder="Type here" />
      <button ref={buttonRef} onClick={focusInput}>
        Focus Input
      </button>
      <div ref={divRef} style={{ marginTop: '1000px' }}>
        Scroll target
      </div>
    </div>
  );
};
```

### **Forwarding Refs**

```typescript
import { forwardRef, useRef, useImperativeHandle, Ref } from 'react';

interface InputProps {
  label: string;
  placeholder?: string;
}

// ForwardRef with explicit type
const FancyInput = forwardRef<HTMLInputElement, InputProps>(
  ({ label, placeholder }, ref) => {
    return (
      <div>
        <label>{label}</label>
        <input ref={ref} placeholder={placeholder} />
      </div>
    );
  }
);

// Usage
const ParentComponent = () => {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleClick = () => {
    inputRef.current?.focus();
  };

  return (
    <div>
      <FancyInput ref={inputRef} label="Name" placeholder="Enter name" />
      <button onClick={handleClick}>Focus Input</button>
    </div>
  );
};
```

### **useImperativeHandle with Custom Methods**

```typescript
interface VideoPlayerRef {
  play: () => void;
  pause: () => void;
  seek: (time: number) => void;
  getCurrentTime: () => number;
}

interface VideoPlayerProps {
  src: string;
}

const VideoPlayer = forwardRef<VideoPlayerRef, VideoPlayerProps>(
  ({ src }, ref) => {
    const videoRef = useRef<HTMLVideoElement>(null);

    useImperativeHandle(ref, () => ({
      play() {
        videoRef.current?.play();
      },
      pause() {
        videoRef.current?.pause();
      },
      seek(time: number) {
        if (videoRef.current) {
          videoRef.current.currentTime = time;
        }
      },
      getCurrentTime() {
        return videoRef.current?.currentTime || 0;
      }
    }));

    return <video ref={videoRef} src={src} />;
  }
);

// Usage
const App = () => {
  const playerRef = useRef<VideoPlayerRef>(null);

  const handlePlay = () => {
    playerRef.current?.play();
  };

  const handleSeek = () => {
    playerRef.current?.seek(30); // Seek to 30 seconds
  };

  const handleGetTime = () => {
    const time = playerRef.current?.getCurrentTime();
    console.log('Current time:', time);
  };

  return (
    <div>
      <VideoPlayer ref={playerRef} src="video.mp4" />
      <button onClick={handlePlay}>Play</button>
      <button onClick={handleSeek}>Skip to 30s</button>
      <button onClick={handleGetTime}>Get Time</button>
    </div>
  );
};
```

---

## Component Props Extraction

### **Extract Props from Component**

```typescript
import { ComponentProps, ComponentPropsWithoutRef, ComponentPropsWithRef } from 'react';

// Extract props from existing component
const Button = ({ onClick, children, disabled }: {
  onClick: () => void;
  children: ReactNode;
  disabled?: boolean;
}) => <button onClick={onClick} disabled={disabled}>{children}</button>;

type ButtonProps = ComponentProps<typeof Button>;
// ButtonProps = { onClick: () => void; children: ReactNode; disabled?: boolean; }

// Extract props from HTML element
type DivProps = ComponentProps<'div'>;
// DivProps = React.DetailedHTMLProps<React.HTMLAttributes<HTMLDivElement>, HTMLDivElement>

type InputProps = ComponentProps<'input'>;
// All standard input props (value, onChange, placeholder, etc.)

// Without ref
type ButtonPropsWithoutRef = ComponentPropsWithoutRef<'button'>;

// With ref
type ButtonPropsWithRef = ComponentPropsWithRef<'button'>;
```

### **Extend HTML Element Props**

```typescript
interface CustomButtonProps extends ComponentPropsWithoutRef<'button'> {
  variant: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  isLoading?: boolean;
}

const CustomButton = ({
  variant,
  size = 'medium',
  isLoading,
  children,
  className,
  ...rest // All other button props
}: CustomButtonProps) => {
  const variantClass = `btn-${variant}`;
  const sizeClass = `btn-${size}`;

  return (
    <button
      className={`btn ${variantClass} ${sizeClass} ${className || ''}`}
      disabled={isLoading || rest.disabled}
      {...rest}
    >
      {isLoading ? 'Loading...' : children}
    </button>
  );
};

// Usage - All button props available
<CustomButton
  variant="primary"
  size="large"
  onClick={() => console.log('clicked')}
  type="submit"
  aria-label="Submit form"
  data-testid="submit-btn"
>
  Submit
</CustomButton>
```

---

## Render Props Pattern

```typescript
interface RenderProps<T> {
  data: T;
  isLoading: boolean;
  error: Error | null;
}

interface DataFetcherProps<T> {
  url: string;
  children: (props: RenderProps<T>) => ReactNode;
}

function DataFetcher<T>({ url, children }: DataFetcherProps<T>) {
  const [data, setData] = useState<T | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err);
        setIsLoading(false);
      });
  }, [url]);

  return <>{children({ data: data as T, isLoading, error })}</>;
}

// Usage
interface User {
  id: number;
  name: string;
}

<DataFetcher<User> url="/api/user">
  {({ data, isLoading, error }) => {
    if (isLoading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;
    return <div>User: {data.name}</div>;
  }}
</DataFetcher>
```

---

## Best Practices

### ✅ DO: Use Explicit Props Types

```typescript
// Good - Clear props interface
interface UserCardProps {
  user: User;
  onEdit: (user: User) => void;
}

const UserCard = ({ user, onEdit }: UserCardProps) => { ... };
```

### ❌ DON'T: Use 'any' for Props

```typescript
// Bad - No type safety
const UserCard = (props: any) => { ... };
```

### ✅ DO: Use Optional Chaining with Refs

```typescript
// Good - Safe access
const handleClick = () => {
  inputRef.current?.focus();
};
```

### ✅ DO: Type Event Handlers Correctly

```typescript
// Good - Specific event type
const handleClick = (e: MouseEvent<HTMLButtonElement>) => {
  console.log(e.currentTarget); // HTMLButtonElement
};
```

### ✅ DO: Use Generic Components for Reusability

```typescript
// Good - Reusable with any type
function List<T>({ items }: { items: T[] }) { ... }
```

---

## Key Takeaways

1. **Prefer explicit props typing** over FC type for clarity
2. **ReactNode** is the most flexible type for children
3. **PropsWithChildren** utility automatically adds children prop
4. **Event types** should specify the element type (MouseEvent<HTMLButtonElement>)
5. **Generic components** enable type-safe reusable components
6. **forwardRef** requires explicit type parameters for ref and props
7. **useImperativeHandle** enables custom ref APIs
8. **ComponentProps** utility extracts props from components or elements
9. **Extend HTML props** to inherit all standard attributes
10. **Optional chaining** with refs prevents null reference errors
    </div>
    );
    };

// Forwarding refs
interface Props {
placeholder: string;
}

const Input = forwardRef<HTMLInputElement, Props>(
({ placeholder }, ref) => {
return <input ref={ref} placeholder={placeholder} />;
}
);

```

---

## Best Practices

✅ **Use explicit props over FC** for better type inference
✅ **Use PropsWithChildren** when children are expected
✅ **Generic components** for reusable list/table components
✅ **Type event handlers** with specific element types
✅ **useRef with null** and optional chaining for safe access
```
