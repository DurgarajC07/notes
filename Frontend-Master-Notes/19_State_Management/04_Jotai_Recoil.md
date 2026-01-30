# Jotai and Recoil - Atomic State Management

## Core Concept

Atomic state management treats state as independent atoms that can be composed. Components subscribe only to the atoms they use, enabling fine-grained reactivity without prop drilling or context boilerplate.

---

## Jotai Basics

```typescript
import { atom, useAtom } from 'jotai';

// Create atoms
const countAtom = atom(0);
const nameAtom = atom('John');

// Use in components
function Counter() {
  const [count, setCount] = useAtom(countAtom);

  return (
    <div>
      <div>Count: {count}</div>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// Another component - independent subscription
function DisplayCount() {
  const [count] = useAtom(countAtom);
  return <div>Count is: {count}</div>;
}
```

---

## Derived Atoms

```typescript
// Base atoms
const firstNameAtom = atom('John');
const lastNameAtom = atom('Doe');

// Derived atom (read-only)
const fullNameAtom = atom((get) => {
  const first = get(firstNameAtom);
  const last = get(lastNameAtom);
  return `${first} ${last}`;
});

// Usage
function Profile() {
  const [firstName, setFirstName] = useAtom(firstNameAtom);
  const [fullName] = useAtom(fullNameAtom);

  return (
    <div>
      <input value={firstName} onChange={e => setFirstName(e.target.value)} />
      <div>Full Name: {fullName}</div>
    </div>
  );
}
```

---

## Writable Derived Atoms

```typescript
const celsiusAtom = atom(0);

// Read and write derived atom
const fahrenheitAtom = atom(
  (get) => get(celsiusAtom) * 9/5 + 32, // Read
  (get, set, newValue: number) => {     // Write
    set(celsiusAtom, (newValue - 32) * 5/9);
  }
);

function Temperature() {
  const [celsius, setCelsius] = useAtom(celsiusAtom);
  const [fahrenheit, setFahrenheit] = useAtom(fahrenheitAtom);

  return (
    <div>
      <input
        type="number"
        value={celsius}
        onChange={e => setCelsius(Number(e.target.value))}
      />
      <input
        type="number"
        value={fahrenheit}
        onChange={e => setFahrenheit(Number(e.target.value))}
      />
    </div>
  );
}
```

---

## Async Atoms

```typescript
import { atom } from 'jotai';

// Async atom with suspense
const userIdAtom = atom<string | null>(null);

const userAtom = atom(async (get) => {
  const userId = get(userIdAtom);
  if (!userId) return null;

  const response = await fetch(`/api/users/${userId}`);
  return response.json();
});

// Usage with Suspense
function UserProfile() {
  const [userId, setUserId] = useAtom(userIdAtom);
  const [user] = useAtom(userAtom);

  return (
    <Suspense fallback={<Loading />}>
      <div>
        {user && <div>{user.name}</div>}
      </div>
    </Suspense>
  );
}
```

---

## Atom Families

```typescript
import { atomFamily } from 'jotai/utils';

// Create atoms dynamically
const todoAtomFamily = atomFamily((id: string) =>
  atom({
    id,
    text: '',
    completed: false
  })
);

function TodoItem({ id }: { id: string }) {
  const [todo, setTodo] = useAtom(todoAtomFamily(id));

  return (
    <div>
      <input
        value={todo.text}
        onChange={e => setTodo({ ...todo, text: e.target.value })}
      />
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={e => setTodo({ ...todo, completed: e.target.checked })}
      />
    </div>
  );
}
```

---

## Jotai with TypeScript

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

// Typed atom
const userAtom = atom<User | null>(null);

// Typed derived atom
const userNameAtom = atom((get) => {
  const user = get(userAtom);
  return user?.name ?? "Guest";
});

// Typed async atom
const usersAtom = atom<Promise<User[]>>(
  fetch("/api/users").then((r) => r.json()),
);
```

---

## Recoil Basics

```typescript
import { atom, useRecoilState, useRecoilValue } from 'recoil';

// Create atom with unique key
const countState = atom({
  key: 'countState',
  default: 0
});

// Use in components
function Counter() {
  const [count, setCount] = useRecoilState(countState);

  return (
    <div>
      <div>Count: {count}</div>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// Read-only
function DisplayCount() {
  const count = useRecoilValue(countState);
  return <div>{count}</div>;
}
```

---

## Recoil Selectors

```typescript
import { selector } from "recoil";

const firstNameState = atom({
  key: "firstName",
  default: "John",
});

const lastNameState = atom({
  key: "lastName",
  default: "Doe",
});

// Derived state
const fullNameState = selector({
  key: "fullName",
  get: ({ get }) => {
    const first = get(firstNameState);
    const last = get(lastNameState);
    return `${first} ${last}`;
  },
});

// Writable selector
const fahrenheitState = atom({
  key: "fahrenheit",
  default: 32,
});

const celsiusState = selector({
  key: "celsius",
  get: ({ get }) => {
    const f = get(fahrenheitState);
    return ((f - 32) * 5) / 9;
  },
  set: ({ set }, newValue) => {
    const f = ((newValue as number) * 9) / 5 + 32;
    set(fahrenheitState, f);
  },
});
```

---

## Async Selectors

```typescript
const userIdState = atom<string | null>({
  key: 'userId',
  default: null
});

const currentUserState = selector({
  key: 'currentUser',
  get: async ({ get }) => {
    const userId = get(userIdState);
    if (!userId) return null;

    const response = await fetch(`/api/users/${userId}`);
    return response.json();
  }
});

// Usage with Suspense
function UserProfile() {
  const user = useRecoilValue(currentUserState);

  return (
    <Suspense fallback={<Loading />}>
      {user && <div>{user.name}</div>}
    </Suspense>
  );
}
```

---

## Atom Effects (Recoil)

```typescript
const persistedState = atom({
  key: "persistedState",
  default: "",
  effects: [
    ({ onSet, setSelf }) => {
      // Load from localStorage on init
      const saved = localStorage.getItem("persistedState");
      if (saved) {
        setSelf(saved);
      }

      // Save to localStorage on change
      onSet((newValue) => {
        localStorage.setItem("persistedState", newValue);
      });
    },
  ],
});

// Synchronization effect
const syncedState = atom({
  key: "synced",
  default: 0,
  effects: [
    ({ onSet, trigger }) => {
      if (trigger === "get") {
        // Sync from server on first read
        fetch("/api/state").then((r) => r.json());
      }

      onSet((newValue) => {
        // Sync to server on change
        fetch("/api/state", {
          method: "POST",
          body: JSON.stringify({ value: newValue }),
        });
      });
    },
  ],
});
```

---

## Comparison: Jotai vs Recoil

### **Jotai Advantages:**

```typescript
// ✅ No string keys needed
const countAtom = atom(0);

// ✅ Simpler API
const [count, setCount] = useAtom(countAtom);

// ✅ Smaller bundle size (~3KB)
// ✅ Better TypeScript inference
// ✅ No provider needed (can be used anywhere)
```

### **Recoil Advantages:**

```typescript
// ✅ Atom effects for side effects
// ✅ Built-in devtools
// ✅ Async selector dependencies
// ✅ More mature ecosystem
// ✅ Facebook backing

// ❌ Requires RecoilRoot provider
<RecoilRoot>
  <App />
</RecoilRoot>
```

---

## Real-World: Shopping Cart

```typescript
// Jotai implementation
import { atom, useAtom } from 'jotai';
import { atomFamily } from 'jotai/utils';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

// Cart items as atom family
const cartItemAtom = atomFamily((id: string) =>
  atom<CartItem | null>(null)
);

// List of item IDs in cart
const cartItemIdsAtom = atom<string[]>([]);

// Derived total
const cartTotalAtom = atom((get) => {
  const ids = get(cartItemIdsAtom);
  return ids.reduce((total, id) => {
    const item = get(cartItemAtom(id));
    return total + (item ? item.price * item.quantity : 0);
  }, 0);
});

// Add to cart action
const addToCartAtom = atom(
  null,
  (get, set, item: CartItem) => {
    const ids = get(cartItemIdsAtom);

    if (!ids.includes(item.id)) {
      set(cartItemIdsAtom, [...ids, item.id]);
    }

    const existing = get(cartItemAtom(item.id));
    set(cartItemAtom(item.id), {
      ...item,
      quantity: (existing?.quantity ?? 0) + item.quantity
    });
  }
);

// Component
function ShoppingCart() {
  const [itemIds] = useAtom(cartItemIdsAtom);
  const [total] = useAtom(cartTotalAtom);
  const [, addToCart] = useAtom(addToCartAtom);

  return (
    <div>
      {itemIds.map(id => (
        <CartItemRow key={id} id={id} />
      ))}
      <div>Total: ${total.toFixed(2)}</div>
    </div>
  );
}

function CartItemRow({ id }: { id: string }) {
  const [item, setItem] = useAtom(cartItemAtom(id));

  if (!item) return null;

  return (
    <div>
      <span>{item.name}</span>
      <input
        type="number"
        value={item.quantity}
        onChange={e => setItem({ ...item, quantity: Number(e.target.value) })}
      />
      <span>${(item.price * item.quantity).toFixed(2)}</span>
    </div>
  );
}
```

---

## Best Practices

✅ **Use atoms for independent state**  
✅ **Derive state instead of duplicating**  
✅ **Split large atoms into smaller ones**  
✅ **Use atom families for dynamic collections**  
✅ **Leverage suspense for async atoms**  
✅ **Type atoms properly with TypeScript**  
❌ **Don't put everything in atoms** - use local state  
❌ **Don't create circular dependencies**  
❌ **Don't forget suspense boundaries**

---

## Key Takeaways

1. **Atomic state is fine-grained** and composable
2. **No boilerplate** compared to Context/Redux
3. **Automatic dependency tracking** prevents unnecessary renders
4. **Suspense integration** for async state
5. **Jotai is simpler**, Recoil more features
6. **Atom families** for dynamic state
7. **Great for medium-sized apps**
