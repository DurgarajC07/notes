# Controlled vs Uncontrolled Components in React

## Core Concept

The distinction between **controlled** and **uncontrolled** components determines who manages form input state: **React** or the **DOM**.

---

## Controlled Components

React state is the "single source of truth". Input value is controlled by React.

```typescript
function ControlledInput() {
  const [value, setValue] = useState('');

  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

**Key characteristics:**

- Input value comes from React state
- Every keystroke triggers re-render
- React has full control over input value

---

## Uncontrolled Components

DOM is the "single source of truth". Input value is managed by DOM.

```typescript
function UncontrolledInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleSubmit = () => {
    console.log(inputRef.current?.value);
  };

  return (
    <>
      <input ref={inputRef} defaultValue="initial" />
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
}
```

**Key characteristics:**

- Input value stored in DOM
- No re-renders on user input
- Access value via ref when needed

---

## When to Use Each

### **Use Controlled (Default)**

✅ **Validation** - Check input on every change  
✅ **Formatting** - Format as user types  
✅ **Conditional logic** - Enable/disable based on input  
✅ **Complex forms** - Multiple interdependent fields  
✅ **Real-time feedback** - Show character count, suggestions

### **Use Uncontrolled (Edge Cases)**

✅ **File inputs** - Can't be controlled  
✅ **Performance** - Very large forms with many fields  
✅ **Integrating non-React code** - Working with DOM libraries  
✅ **Simple forms** - Submit-only, no interim validation

---

## Controlled Form Example

```typescript
interface FormData {
  email: string;
  password: string;
}

function LoginForm() {
  const [formData, setFormData] = useState<FormData>({
    email: '',
    password: ''
  });
  const [errors, setErrors] = useState<Partial<FormData>>({});

  const handleChange = (field: keyof FormData) => (
    e: ChangeEvent<HTMLInputElement>
  ) => {
    setFormData(prev => ({ ...prev, [field]: e.target.value }));

    // Real-time validation
    if (field === 'email' && !e.target.value.includes('@')) {
      setErrors(prev => ({ ...prev, email: 'Invalid email' }));
    } else {
      setErrors(prev => ({ ...prev, [field]: undefined }));
    }
  };

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    console.log('Submit:', formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={formData.email}
        onChange={handleChange('email')}
      />
      {errors.email && <span>{errors.email}</span>}

      <input
        type="password"
        value={formData.password}
        onChange={handleChange('password')}
      />

      <button type="submit">Login</button>
    </form>
  );
}
```

---

## Uncontrolled Form Example

```typescript
function UncontrolledForm() {
  const formRef = useRef<HTMLFormElement>(null);

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();

    const formData = new FormData(formRef.current!);
    const email = formData.get('email');
    const password = formData.get('password');

    console.log('Submit:', { email, password });
  };

  return (
    <form ref={formRef} onSubmit={handleSubmit}>
      <input name="email" defaultValue="" />
      <input name="password" type="password" />
      <button type="submit">Login</button>
    </form>
  );
}
```

---

## File Input (Must be Uncontrolled)

```typescript
function FileUpload() {
  const fileRef = useRef<HTMLInputElement>(null);

  const handleUpload = () => {
    const file = fileRef.current?.files?.[0];
    if (file) {
      console.log('Uploading:', file.name);
    }
  };

  return (
    <>
      <input type="file" ref={fileRef} />
      <button onClick={handleUpload}>Upload</button>
    </>
  );
}
```

---

## Performance Considerations

### **Controlled - More Re-renders**

```typescript
// Every keystroke triggers re-render
<input value={value} onChange={e => setValue(e.target.value)} />
```

### **Uncontrolled - Fewer Re-renders**

```typescript
// No re-renders until form submission
<input ref={inputRef} />
```

---

## Best Practices

✅ **Default to controlled** - more predictable  
✅ **Use `defaultValue`** for uncontrolled initial value  
✅ **Never mix** value and defaultValue  
✅ **Use refs** to access uncontrolled values  
✅ **File inputs must be uncontrolled**  
❌ **Don't use value without onChange** (read-only)  
❌ **Don't update refs in render** - side effect

---

## Common Patterns

### **Debounced Controlled Input**

```typescript
function DebouncedInput() {
  const [value, setValue] = useState('');
  const [debouncedValue, setDebouncedValue] = useState('');

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, 300);

    return () => clearTimeout(timer);
  }, [value]);

  return <input value={value} onChange={e => setValue(e.target.value)} />;
}
```

---

## Key Takeaways

1. **Controlled:** React state manages value
2. **Uncontrolled:** DOM manages value, accessed via ref
3. **Use controlled by default** - better control and validation
4. **File inputs are always uncontrolled**
5. **`defaultValue` for uncontrolled**, `value` for controlled
6. **Never mix value and defaultValue**
