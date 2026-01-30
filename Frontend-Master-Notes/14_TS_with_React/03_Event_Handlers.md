# Event Handlers in TypeScript

## Core Concept

React's event system uses synthetic events for cross-browser compatibility. TypeScript provides specific types for each event, enabling type-safe event handling and preventing runtime errors.

---

## SyntheticEvent Basics

```typescript
import { SyntheticEvent } from "react";

// Generic synthetic event
function handleGeneric(e: SyntheticEvent) {
  e.preventDefault();
  e.stopPropagation();
  console.log(e.currentTarget);
}

// Specific element type
function handleButton(e: SyntheticEvent<HTMLButtonElement>) {
  console.log(e.currentTarget.name); // ✅ Type-safe
}
```

---

## Mouse Events

```typescript
import { MouseEvent } from 'react';

function Button() {
  const handleClick = (e: MouseEvent<HTMLButtonElement>) => {
    console.log('Button:', e.button); // 0 = left, 1 = middle, 2 = right
    console.log('Position:', e.clientX, e.clientY);
    console.log('Alt key:', e.altKey);
    console.log('Ctrl key:', e.ctrlKey);
    console.log('Shift key:', e.shiftKey);
  };

  const handleDoubleClick = (e: MouseEvent<HTMLDivElement>) => {
    e.preventDefault();
    console.log('Double clicked');
  };

  const handleContextMenu = (e: MouseEvent<HTMLDivElement>) => {
    e.preventDefault(); // Prevent right-click menu
    console.log('Right-click detected');
  };

  return (
    <div
      onContextMenu={handleContextMenu}
      onDoubleClick={handleDoubleClick}
    >
      <button onClick={handleClick}>Click me</button>
    </div>
  );
}
```

---

## Keyboard Events

```typescript
import { KeyboardEvent } from 'react';

function SearchInput() {
  const handleKeyDown = (e: KeyboardEvent<HTMLInputElement>) => {
    console.log('Key:', e.key);
    console.log('Code:', e.code);
    console.log('Key code:', e.keyCode); // Deprecated but still used

    // Check specific keys
    if (e.key === 'Enter') {
      console.log('Enter pressed');
    }

    if (e.key === 'Escape') {
      e.currentTarget.blur();
    }

    // Modifier keys
    if (e.ctrlKey && e.key === 's') {
      e.preventDefault();
      console.log('Ctrl+S pressed');
    }
  };

  const handleKeyPress = (e: KeyboardEvent<HTMLInputElement>) => {
    // Only fires for printable characters
    console.log('Character:', e.key);
  };

  return (
    <input
      type="text"
      onKeyDown={handleKeyDown}
      onKeyPress={handleKeyPress}
    />
  );
}
```

---

## Form Events

```typescript
import { FormEvent, ChangeEvent } from 'react';

interface FormData {
  email: string;
  password: string;
  remember: boolean;
}

function LoginForm() {
  const [form, setForm] = useState<FormData>({
    email: '',
    password: '',
    remember: false
  });

  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    console.log('Submit:', form);
  };

  const handleInputChange = (e: ChangeEvent<HTMLInputElement>) => {
    const { name, value, type, checked } = e.target;

    setForm(prev => ({
      ...prev,
      [name]: type === 'checkbox' ? checked : value
    }));
  };

  const handleSelectChange = (e: ChangeEvent<HTMLSelectElement>) => {
    console.log('Selected:', e.target.value);
  };

  const handleTextareaChange = (e: ChangeEvent<HTMLTextAreaElement>) => {
    console.log('Text:', e.target.value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        name="email"
        value={form.email}
        onChange={handleInputChange}
      />

      <input
        type="password"
        name="password"
        value={form.password}
        onChange={handleInputChange}
      />

      <input
        type="checkbox"
        name="remember"
        checked={form.remember}
        onChange={handleInputChange}
      />

      <select onChange={handleSelectChange}>
        <option value="1">Option 1</option>
        <option value="2">Option 2</option>
      </select>

      <textarea onChange={handleTextareaChange} />

      <button type="submit">Login</button>
    </form>
  );
}
```

---

## Focus Events

```typescript
import { FocusEvent } from 'react';

function Input() {
  const handleFocus = (e: FocusEvent<HTMLInputElement>) => {
    console.log('Focused');
    e.target.select(); // Select all text
  };

  const handleBlur = (e: FocusEvent<HTMLInputElement>) => {
    console.log('Blurred');
    console.log('Value:', e.target.value);
  };

  return (
    <input
      type="text"
      onFocus={handleFocus}
      onBlur={handleBlur}
    />
  );
}
```

---

## Clipboard Events

```typescript
import { ClipboardEvent } from 'react';

function SecureInput() {
  const handlePaste = (e: ClipboardEvent<HTMLInputElement>) => {
    e.preventDefault();

    const pastedText = e.clipboardData.getData('text');
    console.log('Pasted:', pastedText);

    // Custom handling (e.g., sanitize)
    const sanitized = pastedText.trim();
    e.currentTarget.value = sanitized;
  };

  const handleCopy = (e: ClipboardEvent<HTMLInputElement>) => {
    console.log('Copied');
  };

  const handleCut = (e: ClipboardEvent<HTMLInputElement>) => {
    console.log('Cut');
  };

  return (
    <input
      type="text"
      onPaste={handlePaste}
      onCopy={handleCopy}
      onCut={handleCut}
    />
  );
}
```

---

## Drag Events

```typescript
import { DragEvent } from 'react';

function DraggableItem({ id, text }: { id: string; text: string }) {
  const handleDragStart = (e: DragEvent<HTMLDivElement>) => {
    e.dataTransfer.effectAllowed = 'move';
    e.dataTransfer.setData('text/plain', id);
  };

  const handleDragEnd = (e: DragEvent<HTMLDivElement>) => {
    console.log('Drag ended');
  };

  return (
    <div
      draggable
      onDragStart={handleDragStart}
      onDragEnd={handleDragEnd}
    >
      {text}
    </div>
  );
}

function DropZone() {
  const handleDragOver = (e: DragEvent<HTMLDivElement>) => {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';
  };

  const handleDrop = (e: DragEvent<HTMLDivElement>) => {
    e.preventDefault();

    const id = e.dataTransfer.getData('text/plain');
    console.log('Dropped item:', id);
  };

  return (
    <div
      onDragOver={handleDragOver}
      onDrop={handleDrop}
      style={{ minHeight: '200px', border: '2px dashed #ccc' }}
    >
      Drop here
    </div>
  );
}
```

---

## Generic Event Handlers

```typescript
// Reusable event handler type
type EventHandler<T = HTMLElement> = (e: SyntheticEvent<T>) => void;

interface ButtonProps {
  onClick?: EventHandler<HTMLButtonElement>;
  onDoubleClick?: EventHandler<HTMLButtonElement>;
}

function CustomButton({ onClick, onDoubleClick }: ButtonProps) {
  return (
    <button
      onClick={onClick}
      onDoubleClick={onDoubleClick}
    >
      Click
    </button>
  );
}

// Usage
<CustomButton
  onClick={(e) => console.log('Clicked', e.currentTarget)}
  onDoubleClick={(e) => console.log('Double clicked')}
/>
```

---

## Event Handler Props

```typescript
interface InputProps {
  value: string;
  onChange: (value: string) => void;
  onBlur?: () => void;
  onFocus?: () => void;
}

// Wrapper that simplifies API
function Input({ value, onChange, onBlur, onFocus }: InputProps) {
  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    onChange(e.target.value); // Pass just the value
  };

  return (
    <input
      value={value}
      onChange={handleChange}
      onBlur={onBlur}
      onFocus={onFocus}
    />
  );
}

// Usage - simpler for consumers
<Input
  value={name}
  onChange={setName} // Just pass value, not event
/>
```

---

## Event Delegation

```typescript
function TodoList({ items }: { items: Todo[] }) {
  // Single handler for all items
  const handleClick = (e: MouseEvent<HTMLUListElement>) => {
    const target = e.target as HTMLElement;

    // Check if clicked on delete button
    if (target.classList.contains('delete-btn')) {
      const itemId = target.dataset.id;
      console.log('Delete item:', itemId);
    }

    // Check if clicked on item
    if (target.classList.contains('todo-item')) {
      const itemId = target.dataset.id;
      console.log('Select item:', itemId);
    }
  };

  return (
    <ul onClick={handleClick}>
      {items.map(item => (
        <li key={item.id} className="todo-item" data-id={item.id}>
          {item.text}
          <button className="delete-btn" data-id={item.id}>
            Delete
          </button>
        </li>
      ))}
    </ul>
  );
}
```

---

## Preventing Default and Propagation

```typescript
function Form() {
  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault(); // Prevent page reload
    console.log('Form submitted');
  };

  const handleInnerClick = (e: MouseEvent<HTMLDivElement>) => {
    e.stopPropagation(); // Don't trigger parent handlers
    console.log('Inner clicked');
  };

  const handleOuterClick = (e: MouseEvent<HTMLDivElement>) => {
    console.log('Outer clicked');
  };

  return (
    <form onSubmit={handleSubmit}>
      <div onClick={handleOuterClick}>
        <div onClick={handleInnerClick}>
          Click me
        </div>
      </div>
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Real-World: Keyboard Shortcuts

```typescript
function App() {
  useEffect(() => {
    const handleKeyDown = (e: globalThis.KeyboardEvent) => {
      // Cmd/Ctrl + K for search
      if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
        e.preventDefault();
        openSearch();
      }

      // Cmd/Ctrl + S for save
      if ((e.metaKey || e.ctrlKey) && e.key === 's') {
        e.preventDefault();
        save();
      }

      // Escape to close modal
      if (e.key === 'Escape') {
        closeModal();
      }
    };

    window.addEventListener('keydown', handleKeyDown);

    return () => {
      window.removeEventListener('keydown', handleKeyDown);
    };
  }, []);

  return <div>App</div>;
}
```

---

## Best Practices

✅ **Use specific event types** (MouseEvent, KeyboardEvent)  
✅ **Type element generics** properly  
✅ **Call preventDefault/stopPropagation** when needed  
✅ **Simplify event handler props** for consumers  
✅ **Use event delegation** for performance  
✅ **Type global event listeners** correctly  
❌ **Don't use `any`** for events  
❌ **Don't forget to prevent default** on forms  
❌ **Don't attach too many listeners** - use delegation

---

## Key Takeaways

1. **SyntheticEvent wraps** native browser events
2. **TypeScript provides specific types** for each event
3. **Element generics ensure** type safety
4. **Event delegation improves** performance
5. **preventDefault/stopPropagation** control event flow
6. **Custom event props** simplify component APIs
7. **Keyboard shortcuts require** global listeners
