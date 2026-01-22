# 🎣 useState & useReducer Deep Dive

> State management hooks are the foundation of React's component state. Understanding their internals, when to use each, and optimization techniques is crucial for building performant applications.

---

## 📖 1. Concept Explanation

### useState

**useState** manages local component state with a simple API:

```javascript
const [state, setState] = useState(initialValue);
```

**How it works:**

1. First render: Returns initial value and setter
2. Subsequent renders: Returns current state and same setter
3. Setter triggers re-render with new state

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

### useReducer

**useReducer** manages complex state with a reducer function:

```javascript
const [state, dispatch] = useReducer(reducer, initialState);
```

**Reducer pattern:**

```javascript
function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    case "DECREMENT":
      return { count: state.count - 1 };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    <button onClick={() => dispatch({ type: "INCREMENT" })}>
      Count: {state.count}
    </button>
  );
}
```

### When to Use Which?

| Scenario                  | useState              | useReducer               |
| ------------------------- | --------------------- | ------------------------ |
| Simple state              | ✅ Perfect            | ❌ Overkill              |
| Multiple related values   | ⚠️ Multiple hooks     | ✅ Single object         |
| Complex transitions       | ❌ Hard to read       | ✅ Clear logic           |
| State depends on previous | ⚠️ Functional updates | ✅ Natural fit           |
| Testing                   | ⚠️ Need component     | ✅ Pure reducer function |

---

## 🧠 2. Why It Matters in Real Projects

### Production Impact

**useState Simplicity:**

```javascript
// ✅ Perfect for simple state
function Toggle() {
  const [isOn, setIsOn] = useState(false);

  return <button onClick={() => setIsOn(!isOn)}>{isOn ? "ON" : "OFF"}</button>;
}
```

**useReducer for Complex State:**

```javascript
// ✅ Better for complex state transitions
function ShoppingCart() {
  const [state, dispatch] = useReducer(cartReducer, {
    items: [],
    total: 0,
    discounts: [],
    shipping: null,
  });

  // Actions clearly describe intent
  dispatch({ type: "ADD_ITEM", item });
  dispatch({ type: "APPLY_DISCOUNT", code });
  dispatch({ type: "SET_SHIPPING", method });
}
```

### Real-World Benefits

**1. Predictability:**

- Reducers are pure functions (easier to test)
- Actions document state changes
- Time-travel debugging possible

**2. Performance:**

- Both hooks use shallow comparison
- Batching updates (React 18+)
- Can optimize with useMemo/useCallback

**3. Scalability:**

- useReducer scales better for complex state
- Easier to add new actions
- Reducer can be shared/imported

---

## ⚙️ 3. Internal Working

### How useState Works Internally

```javascript
// Simplified React implementation
let hooks = [];
let currentHook = 0;

function useState(initialValue) {
  const hookIndex = currentHook;

  // Initialize on first render
  if (hooks[hookIndex] === undefined) {
    hooks[hookIndex] = initialValue;
  }

  const setState = (newValue) => {
    // Functional update
    if (typeof newValue === "function") {
      hooks[hookIndex] = newValue(hooks[hookIndex]);
    } else {
      hooks[hookIndex] = newValue;
    }

    // Trigger re-render
    render();
  };

  currentHook++;
  return [hooks[hookIndex], setState];
}

// Reset before each render
function render() {
  currentHook = 0;
  ReactDOM.render(<App />, container);
}
```

### Fiber Integration

```javascript
// In React Fiber
{
  type: FunctionComponent,
  memoizedState: {
    // First hook
    memoizedState: 0,     // useState value
    next: {
      // Second hook
      memoizedState: false, // Another useState
      next: {
        // Third hook
        memoizedState: { count: 0 }, // useReducer
        next: null
      }
    }
  }
}
```

### Batching Updates (React 18+)

```javascript
// React 18: Automatic batching
function handleClick() {
  setCount((c) => c + 1); // Not re-rendered yet
  setFlag((f) => !f); // Not re-rendered yet
  setData((d) => [...d]); // Only re-renders ONCE here
}

// React 17: Only batched in event handlers
setTimeout(() => {
  setCount((c) => c + 1); // Re-renders
  setFlag((f) => !f); // Re-renders again
}, 1000);

// React 18: Batched everywhere
setTimeout(() => {
  setCount((c) => c + 1); // Not re-rendered yet
  setFlag((f) => !f); // Only re-renders ONCE
}, 1000);
```

### Object.is Comparison

```javascript
// React uses Object.is for comparison
Object.is(0, 0); // true
Object.is(+0, -0); // false
Object.is(NaN, NaN); // true
Object.is({}, {}); // false (different references)

// setState only re-renders if new value !== old value
setCount(0); // Initial
setCount(0); // No re-render (same value)
setCount((prev) => prev); // No re-render (same value)
```

---

## ✅ 4. Best Practices

### DO ✅

```javascript
// 1. Use functional updates when depending on previous state
const [count, setCount] = useState(0);

// ✅ GOOD: Functional update
setCount(prev => prev + 1);

// ❌ BAD: Might use stale state
setCount(count + 1);

// 2. Initialize with function for expensive computation
// ✅ GOOD: Function runs only once
const [data, setData] = useState(() => {
  return expensiveComputation(); // Only on mount
});

// ❌ BAD: Runs on every render
const [data, setData] = useState(expensiveComputation());

// 3. Group related state in useReducer
// ✅ GOOD
const [state, dispatch] = useReducer(reducer, {
  name: '',
  email: '',
  age: 0,
  errors: {}
});

// ❌ BAD: Too many separate states
const [name, setName] = useState('');
const [email, setEmail] = useState('');
const [age, setAge] = useState(0);
const [errors, setErrors] = useState({});

// 4. Use TypeScript for type safety
type State = {
  count: number;
  user: User | null;
};

type Action =
  | { type: 'INCREMENT' }
  | { type: 'SET_USER'; payload: User };

const [state, dispatch] = useReducer<React.Reducer<State, Action>>(
  reducer,
  { count: 0, user: null }
);

// 5. Extract reducer for reusability
// reducers/cartReducer.ts
export function cartReducer(state: CartState, action: CartAction) {
  switch (action.type) {
    case 'ADD_ITEM':
      return { ...state, items: [...state.items, action.item] };
    default:
      return state;
  }
}

// Component.tsx
import { cartReducer } from './reducers/cartReducer';

const [cart, dispatch] = useReducer(cartReducer, initialCart);
```

### DON'T ❌

```javascript
// 1. Don't mutate state directly
// ❌ BAD
const [items, setItems] = useState([]);
items.push(newItem); // Mutation!
setItems(items); // Won't trigger re-render

// ✅ GOOD
setItems([...items, newItem]);

// 2. Don't call setState during render
// ❌ BAD: Infinite loop
function Component({ value }) {
  const [state, setState] = useState(0);

  if (value > 10) {
    setState(value); // ❌ Called during render!
  }

  return <div>{state}</div>;
}

// ✅ GOOD: Use useEffect
function Component({ value }) {
  const [state, setState] = useState(0);

  useEffect(() => {
    if (value > 10) {
      setState(value);
    }
  }, [value]);

  return <div>{state}</div>;
}

// 3. Don't use state for derived values
// ❌ BAD
const [firstName, setFirstName] = useState("");
const [lastName, setLastName] = useState("");
const [fullName, setFullName] = useState(""); // Redundant!

useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]);

// ✅ GOOD: Calculate during render
const fullName = `${firstName} ${lastName}`;

// 4. Don't have too many useState calls
// ❌ BAD
const [name, setName] = useState("");
const [email, setEmail] = useState("");
const [age, setAge] = useState(0);
const [address, setAddress] = useState("");
const [phone, setPhone] = useState("");
const [city, setCity] = useState("");
// ... 10 more states

// ✅ GOOD: Use useReducer or single object
const [formData, setFormData] = useState({
  name: "",
  email: "",
  age: 0,
  address: "",
  phone: "",
  city: "",
});

// 5. Don't forget to handle all action types
// ❌ BAD: No default case
function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    // Missing default!
  }
}

// ✅ GOOD: Always have default
function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    default:
      return state; // or throw error
  }
}
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Stale Closure in Event Handlers

```javascript
// ❌ WRONG: Captures stale count
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setCount(count + 1); // Always uses initial count!
    }, 1000);

    return () => clearInterval(interval);
  }, []); // Empty deps = count never updates

  return <div>{count}</div>;
}

// ✅ FIX 1: Functional update
useEffect(() => {
  const interval = setInterval(() => {
    setCount((prev) => prev + 1); // Uses latest count
  }, 1000);

  return () => clearInterval(interval);
}, []);

// ✅ FIX 2: Include in dependencies
useEffect(() => {
  const interval = setInterval(() => {
    setCount(count + 1);
  }, 1000);

  return () => clearInterval(interval);
}, [count]); // Re-creates interval when count changes
```

### Mistake #2: Unnecessary State

```javascript
// ❌ WRONG: Storing derived value in state
function ProductList({ products }) {
  const [filteredProducts, setFilteredProducts] = useState([]);
  const [searchTerm, setSearchTerm] = useState('');

  useEffect(() => {
    setFilteredProducts(
      products.filter(p => p.name.includes(searchTerm))
    );
  }, [products, searchTerm]);

  return <div>{filteredProducts.map(...)}</div>;
}

// ✅ CORRECT: Calculate during render
function ProductList({ products }) {
  const [searchTerm, setSearchTerm] = useState('');

  const filteredProducts = products.filter(p =>
    p.name.includes(searchTerm)
  );

  return <div>{filteredProducts.map(...)}</div>;
}
```

### Mistake #3: State Updates Not Batched (React 17)

```javascript
// React 17: Not batched outside event handlers
function Component() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  // ❌ In React 17: 2 re-renders
  fetch("/api/data").then(() => {
    setCount(1); // Re-render
    setFlag(true); // Another re-render
  });

  // ✅ Solution: Use unstable_batchedUpdates
  import { unstable_batchedUpdates } from "react-dom";

  fetch("/api/data").then(() => {
    unstable_batchedUpdates(() => {
      setCount(1);
      setFlag(true); // Only 1 re-render
    });
  });

  // ✅ React 18: Automatically batched everywhere
  fetch("/api/data").then(() => {
    setCount(1);
    setFlag(true); // Batched automatically
  });
}
```

### Mistake #4: Mutating State Object

```javascript
// ❌ WRONG: Mutating nested object
const [user, setUser] = useState({
  name: "Alice",
  address: {
    city: "NYC",
    zip: "10001",
  },
});

// ❌ This won't work
user.address.city = "LA";
setUser(user); // Same reference, no re-render!

// ✅ CORRECT: Create new object
setUser({
  ...user,
  address: {
    ...user.address,
    city: "LA",
  },
});

// ✅ Better: Use immer
import { produce } from "immer";

setUser(
  produce((draft) => {
    draft.address.city = "LA"; // Immer handles immutability
  }),
);
```

---

## 🔐 6. Security Considerations

### Sanitize State from User Input

```javascript
// ❌ DANGEROUS: Unsanitized input
function CommentForm() {
  const [comment, setComment] = useState("");

  return (
    <div>
      <input onChange={(e) => setComment(e.target.value)} />
      <div dangerouslySetInnerHTML={{ __html: comment }} />
      {/* XSS vulnerability! */}
    </div>
  );
}

// ✅ SAFE: Sanitize or use text
import DOMPurify from "dompurify";

function CommentForm() {
  const [comment, setComment] = useState("");

  return (
    <div>
      <input onChange={(e) => setComment(e.target.value)} />
      <div
        dangerouslySetInnerHTML={{
          __html: DOMPurify.sanitize(comment),
        }}
      />
    </div>
  );
}
```

### Validate State Transitions

```javascript
// ✅ Validate in reducer
function reducer(state, action) {
  switch (action.type) {
    case "SET_AGE":
      // Validate
      const age = parseInt(action.payload);
      if (isNaN(age) || age < 0 || age > 150) {
        return state; // Reject invalid transition
      }
      return { ...state, age };

    default:
      return state;
  }
}
```

---

## 🚀 7. Performance Optimization

### 1. Lazy State Initialization

```javascript
// ❌ SLOW: Runs on every render
const [state, setState] = useState(JSON.parse(localStorage.getItem("data")));

// ✅ FAST: Only runs on mount
const [state, setState] = useState(() =>
  JSON.parse(localStorage.getItem("data")),
);
```

### 2. Bail Out of Updates

```javascript
// React bails out if state hasn't changed
setCount(5);
setCount(5); // No re-render (same value)

// Use functional update to access latest state
setCount((prev) => (prev === 5 ? prev : 5)); // Smart bailout
```

### 3. Split State for Better Re-renders

```javascript
// ❌ CAUSES UNNECESSARY RE-RENDERS
function Form() {
  const [formData, setFormData] = useState({
    name: "",
    email: "",
    heavyData: largeObject,
  });

  // Changing name re-renders entire form
  // Including heavyData consumers
}

// ✅ SPLIT STATE
function Form() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [heavyData, setHeavyData] = useState(largeObject);

  // Only name consumers re-render
}
```

### 4. Use useReducer for Complex Updates

```javascript
// ❌ Multiple setState calls
function updateUser() {
  setName(newName);
  setEmail(newEmail);
  setAge(newAge);
  // 3 re-renders (React 17), 1 re-render (React 18)
}

// ✅ Single dispatch
function updateUser() {
  dispatch({
    type: "UPDATE_USER",
    payload: { name: newName, email: newEmail, age: newAge },
  });
  // Always 1 re-render
}
```

---

## 🧪 8. Code Examples

### Example 1: Form with useState

```javascript
function LoginForm() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [errors, setErrors] = useState({});

  const validate = () => {
    const newErrors = {};

    if (!email.includes("@")) {
      newErrors.email = "Invalid email";
    }

    if (password.length < 8) {
      newErrors.password = "Password too short";
    }

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = (e) => {
    e.preventDefault();

    if (validate()) {
      // Submit form
      login(email, password);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
      {errors.email && <span>{errors.email}</span>}

      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
      />
      {errors.password && <span>{errors.password}</span>}

      <button type="submit">Login</button>
    </form>
  );
}
```

### Example 2: Complex State with useReducer

```javascript
type State = {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
  loading: boolean;
  error: string | null;
};

type Action =
  | { type: 'ADD_TODO'; payload: Todo }
  | { type: 'TOGGLE_TODO'; payload: string }
  | { type: 'DELETE_TODO'; payload: string }
  | { type: 'SET_FILTER'; payload: State['filter'] }
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: Todo[] }
  | { type: 'FETCH_ERROR'; payload: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [...state.todos, action.payload]
      };

    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload
            ? { ...todo, completed: !todo.completed }
            : todo
        )
      };

    case 'DELETE_TODO':
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.payload)
      };

    case 'SET_FILTER':
      return {
        ...state,
        filter: action.payload
      };

    case 'FETCH_START':
      return {
        ...state,
        loading: true,
        error: null
      };

    case 'FETCH_SUCCESS':
      return {
        ...state,
        loading: false,
        todos: action.payload
      };

    case 'FETCH_ERROR':
      return {
        ...state,
        loading: false,
        error: action.payload
      };

    default:
      return state;
  }
}

function TodoApp() {
  const [state, dispatch] = useReducer(reducer, {
    todos: [],
    filter: 'all',
    loading: false,
    error: null
  });

  const addTodo = (text: string) => {
    dispatch({
      type: 'ADD_TODO',
      payload: {
        id: Date.now().toString(),
        text,
        completed: false
      }
    });
  };

  const filteredTodos = state.todos.filter(todo => {
    if (state.filter === 'active') return !todo.completed;
    if (state.filter === 'completed') return todo.completed;
    return true;
  });

  return (
    <div>
      <TodoInput onAdd={addTodo} />
      <FilterButtons
        current={state.filter}
        onChange={filter => dispatch({ type: 'SET_FILTER', payload: filter })}
      />
      <TodoList
        todos={filteredTodos}
        onToggle={id => dispatch({ type: 'TOGGLE_TODO', payload: id })}
        onDelete={id => dispatch({ type: 'DELETE_TODO', payload: id })}
      />
    </div>
  );
}
```

### Example 3: Optimistic Updates

```javascript
function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);

  const toggleTodo = async (id: string) => {
    // Optimistic update
    setTodos(prev => prev.map(todo =>
      todo.id === id
        ? { ...todo, completed: !todo.completed }
        : todo
    ));

    try {
      await api.toggleTodo(id);
    } catch (error) {
      // Rollback on error
      setTodos(prev => prev.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      ));

      toast.error('Failed to update todo');
    }
  };

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => toggleTodo(todo.id)}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: Multi-Step Form

```javascript
type FormState = {
  step: number;
  data: {
    personalInfo: PersonalInfo;
    address: Address;
    payment: Payment;
  };
  errors: Record<string, string>;
};

function reducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'NEXT_STEP':
      return { ...state, step: state.step + 1, errors: {} };

    case 'PREV_STEP':
      return { ...state, step: state.step - 1, errors: {} };

    case 'UPDATE_FIELD':
      return {
        ...state,
        data: {
          ...state.data,
          [action.section]: {
            ...state.data[action.section],
            [action.field]: action.value
          }
        }
      };

    case 'SET_ERRORS':
      return { ...state, errors: action.payload };

    default:
      return state;
  }
}

function MultiStepForm() {
  const [state, dispatch] = useReducer(reducer, {
    step: 1,
    data: {
      personalInfo: {},
      address: {},
      payment: {}
    },
    errors: {}
  });

  // Render current step based on state.step
}
```

### Use Case 2: Shopping Cart

```javascript
function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM':
      const existing = state.items.find(i => i.id === action.item.id);

      if (existing) {
        return {
          ...state,
          items: state.items.map(i =>
            i.id === action.item.id
              ? { ...i, quantity: i.quantity + 1 }
              : i
          )
        };
      }

      return {
        ...state,
        items: [...state.items, { ...action.item, quantity: 1 }]
      };

    case 'REMOVE_ITEM':
      return {
        ...state,
        items: state.items.filter(i => i.id !== action.id)
      };

    case 'UPDATE_QUANTITY':
      return {
        ...state,
        items: state.items.map(i =>
          i.id === action.id
            ? { ...i, quantity: action.quantity }
            : i
        )
      };

    case 'APPLY_DISCOUNT':
      return {
        ...state,
        discount: action.discount
      };

    case 'CLEAR_CART':
      return initialCartState;

    default:
      return state;
  }
}
```

---

## ❓ 10. Interview Questions

### Q1: When should you use useState vs useReducer?

**Answer:**

| Use useState when...                         | Use useReducer when...                       |
| -------------------------------------------- | -------------------------------------------- |
| State is primitive (number, string, boolean) | State is complex object with multiple fields |
| State updates are simple                     | State updates have complex logic             |
| No related state transitions                 | State transitions are related                |
| Component is small                           | Component logic is complex                   |

**Example:**

```javascript
// useState: Simple toggle
const [isOpen, setIsOpen] = useState(false);

// useReducer: Complex form
const [formState, dispatch] = useReducer(formReducer, {
  values: {},
  errors: {},
  touched: {},
  isSubmitting: false,
});
```

**Follow-up:** Can you convert useState to useReducer?

```javascript
// useState
const [count, setCount] = useState(0);

// Equivalent useReducer
const [count, dispatch] = useReducer((state, action) => action, 0);
dispatch(count + 1); // Same as setCount(count + 1)
```

---

### Q2: What are functional updates and why use them?

**Answer:**

Functional updates pass a function to setState that receives the previous state:

```javascript
// Regular update
setCount(count + 1);

// Functional update
setCount((prev) => prev + 1);
```

**Why use functional updates:**

1. **Avoid stale closures:**

```javascript
// ❌ Stale closure
useEffect(() => {
  const interval = setInterval(() => {
    setCount(count + 1); // Always uses initial count
  }, 1000);
}, []); // Empty deps

// ✅ Always uses latest
useEffect(() => {
  const interval = setInterval(() => {
    setCount((prev) => prev + 1); // Uses current count
  }, 1000);
}, []);
```

2. **Multiple updates in same cycle:**

```javascript
// ❌ Only last update wins
setCount(count + 1);
setCount(count + 1);
setCount(count + 1);
// Result: count + 1 (not count + 3)

// ✅ All updates applied
setCount((prev) => prev + 1);
setCount((prev) => prev + 1);
setCount((prev) => prev + 1);
// Result: count + 3
```

---

### Q3: How does React batch state updates?

**Answer:**

**React 18+:** Automatic batching everywhere

```javascript
// All batched (one re-render)
function handleClick() {
  setCount((c) => c + 1);
  setFlag((f) => !f);
  setData((d) => [...d]);
}

// Even in promises
fetch("/api").then(() => {
  setCount(1);
  setFlag(true); // Batched!
});

// Even in setTimeout
setTimeout(() => {
  setCount(1);
  setFlag(true); // Batched!
}, 1000);
```

**React 17:** Only batched in event handlers

```javascript
// Batched (one re-render)
function handleClick() {
  setCount(1);
  setFlag(true);
}

// NOT batched (two re-renders)
fetch("/api").then(() => {
  setCount(1); // Re-render
  setFlag(true); // Another re-render
});

// Fix: unstable_batchedUpdates
import { unstable_batchedUpdates } from "react-dom";

fetch("/api").then(() => {
  unstable_batchedUpdates(() => {
    setCount(1);
    setFlag(true); // Now batched
  });
});
```

**Opt-out of batching (React 18):**

```javascript
import { flushSync } from "react-dom";

flushSync(() => {
  setCount(1); // Renders immediately
});
flushSync(() => {
  setFlag(true); // Renders again
});
```

---

### Q4: What happens if you mutate state directly?

**Answer:**

**Problem:** React won't detect the change because object reference is the same.

```javascript
// ❌ WRONG: Mutation
const [items, setItems] = useState([1, 2, 3]);

items.push(4); // Mutates array
setItems(items); // Same reference!
// Component won't re-render

// ✅ CORRECT: New array
setItems([...items, 4]);
```

**Why it matters:**

1. **Shallow comparison:**

```javascript
// React's comparison
if (Object.is(newState, oldState)) {
  return; // Don't re-render
}

// Mutation = same reference
Object.is(mutatedArray, originalArray); // true (same reference)
```

2. **Breaks optimization:**

```javascript
const MemoizedList = memo(({ items }) => {
  // Won't update if items array is mutated
  return items.map((item) => <div>{item}</div>);
});
```

**Solution:** Always create new objects/arrays

```javascript
// Objects
setState({ ...state, newProp: value });

// Arrays
setState([...array, newItem]);

// Nested updates
setState({
  ...state,
  nested: {
    ...state.nested,
    value: newValue,
  },
});

// Or use Immer
import { produce } from "immer";

setState(
  produce((draft) => {
    draft.nested.value = newValue; // Immer handles immutability
  }),
);
```

---

### Q5: How do you test components with useState/useReducer?

**Answer:**

**Testing useState:**

```javascript
import { render, screen, fireEvent } from "@testing-library/react";

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <span>{count}</span>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

test("increments counter", () => {
  render(<Counter />);

  expect(screen.getByText("0")).toBeInTheDocument();

  fireEvent.click(screen.getByText("Increment"));

  expect(screen.getByText("1")).toBeInTheDocument();
});
```

**Testing useReducer (test reducer separately):**

```javascript
// reducer.ts
export function counterReducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    default:
      return state;
  }
}

// reducer.test.ts
import { counterReducer } from "./reducer";

test("increments count", () => {
  const state = { count: 0 };
  const newState = counterReducer(state, { type: "INCREMENT" });

  expect(newState.count).toBe(1);
});

// Component test
test("dispatches increment action", () => {
  render(<Counter />);

  fireEvent.click(screen.getByText("Increment"));

  expect(screen.getByText("1")).toBeInTheDocument();
});
```

---

## 🧩 11. Design Patterns

### Pattern 1: State Machine with useReducer

```javascript
type State =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: Data }
  | { status: 'error'; error: Error };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'FETCH':
      return { status: 'loading' };

    case 'SUCCESS':
      return { status: 'success', data: action.data };

    case 'ERROR':
      return { status: 'error', error: action.error };

    case 'RESET':
      return { status: 'idle' };

    default:
      return state;
  }
}
```

### Pattern 2: Undo/Redo

```javascript
function undoableReducer(reducer) {
  const initialState = {
    past: [],
    present: reducer(undefined, {}),
    future: [],
  };

  return function (state = initialState, action) {
    const { past, present, future } = state;

    switch (action.type) {
      case "UNDO":
        if (past.length === 0) return state;
        return {
          past: past.slice(0, -1),
          present: past[past.length - 1],
          future: [present, ...future],
        };

      case "REDO":
        if (future.length === 0) return state;
        return {
          past: [...past, present],
          present: future[0],
          future: future.slice(1),
        };

      default:
        const newPresent = reducer(present, action);
        if (present === newPresent) return state;
        return {
          past: [...past, present],
          present: newPresent,
          future: [],
        };
    }
  };
}
```

---

## 🎯 Summary

useState and useReducer are fundamental React hooks:

- **useState:** Simple state, primitive values
- **useReducer:** Complex state, multiple related updates
- **Functional updates:** Access latest state
- **Batching:** Automatic in React 18+
- **Immutability:** Always create new objects/arrays

**Production checklist:**

- ✅ Use functional updates when depending on previous state
- ✅ Initialize expensive state with function
- ✅ Group related state in useReducer
- ✅ Test reducers separately (pure functions)
- ✅ Never mutate state directly
- ✅ Split state for better performance
- ✅ Use TypeScript for type safety

Master these hooks to build robust, performant React applications! 🚀
