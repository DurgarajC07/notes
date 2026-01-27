# React Component Types

## Functional Component Types

### **Basic FC Type**

```typescript
import React, { FC } from 'react';

interface Props {
  name: string;
  age: number;
}

// FC type includes children by default
const Greeting: FC<Props> = ({ name, age }) => {
  return <div>Hello {name}, age {age}</div>;
};

// Explicit props (preferred - more explicit about children)
const Greeting = ({ name, age }: Props) => {
  return <div>Hello {name}, age {age}</div>;
};
```

### **With Children**

```typescript
import { ReactNode } from 'react';

interface Props {
  title: string;
  children: ReactNode;
}

const Card = ({ title, children }: Props) => {
  return (
    <div>
      <h2>{title}</h2>
      {children}
    </div>
  );
};
```

### **PropsWithChildren**

```typescript
import { PropsWithChildren } from 'react';

interface Props {
  title: string;
}

const Container = ({ title, children }: PropsWithChildren<Props>) => {
  return (
    <section>
      <h1>{title}</h1>
      {children}
    </section>
  );
};
```

---

## Event Handler Types

```typescript
import { MouseEvent, ChangeEvent, FormEvent } from 'react';

interface Props {
  onSubmit: (value: string) => void;
}

const Form = ({ onSubmit }: Props) => {
  const [value, setValue] = useState('');

  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    setValue(e.target.value);
  };

  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    onSubmit(value);
  };

  const handleClick = (e: MouseEvent<HTMLButtonElement>) => {
    console.log('Button clicked', e.currentTarget);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={value} onChange={handleChange} />
      <button onClick={handleClick}>Submit</button>
    </form>
  );
};
```

---

## Generic Components

```typescript
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => ReactNode;
}

function List<T>({ items, renderItem }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage
interface User {
  id: number;
  name: string;
}

<List<User>
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
/>
```

---

## Ref Types

```typescript
import { useRef, RefObject } from 'react';

const Input = () => {
  const inputRef = useRef<HTMLInputElement>(null);

  const focusInput = () => {
    inputRef.current?.focus();
  };

  return (
    <div>
      <input ref={inputRef} />
      <button onClick={focusInput}>Focus</button>
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
