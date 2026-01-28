# Zustand State Management

## Core Concept

Zustand is a minimal, fast state management library using hooks. No providers, no boilerplate - just simple, scalable state management.

---

## Basic Store

```typescript
import create from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

// Usage in component
function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);

  return (
    <div>
      <div>Count: {count}</div>
      <button onClick={increment}>Increment</button>
    </div>
  );
}
```

---

## Selecting State

```typescript
interface UserState {
  user: {
    id: string;
    name: string;
    email: string;
  } | null;
  isLoading: boolean;
  error: string | null;
}

const useUserStore = create<UserState>(() => ({
  user: null,
  isLoading: false,
  error: null,
}));

// ❌ BAD - Re-renders on any state change
function Profile() {
  const state = useUserStore();
  return <div>{state.user?.name}</div>;
}

// ✅ GOOD - Only re-renders when name changes
function Profile() {
  const userName = useUserStore((state) => state.user?.name);
  return <div>{userName}</div>;
}

// ✅ BETTER - Use shallow for multiple properties
import { shallow } from 'zustand/shallow';

function Profile() {
  const { user, isLoading } = useUserStore(
    (state) => ({ user: state.user, isLoading: state.isLoading }),
    shallow
  );

  return (
    <div>
      {isLoading ? <Spinner /> : <div>{user?.name}</div>}
    </div>
  );
}
```

---

## Async Actions

```typescript
interface TodoState {
  todos: Todo[];
  isLoading: boolean;
  error: string | null;
  fetchTodos: () => Promise<void>;
  addTodo: (text: string) => Promise<void>;
  deleteTodo: (id: string) => Promise<void>;
}

const useTodoStore = create<TodoState>((set, get) => ({
  todos: [],
  isLoading: false,
  error: null,

  fetchTodos: async () => {
    set({ isLoading: true, error: null });

    try {
      const response = await fetch("/api/todos");
      const todos = await response.json();
      set({ todos, isLoading: false });
    } catch (error) {
      set({ error: error.message, isLoading: false });
    }
  },

  addTodo: async (text: string) => {
    try {
      const response = await fetch("/api/todos", {
        method: "POST",
        body: JSON.stringify({ text }),
      });
      const newTodo = await response.json();

      // Access current state with get()
      set({ todos: [...get().todos, newTodo] });
    } catch (error) {
      set({ error: error.message });
    }
  },

  deleteTodo: async (id: string) => {
    const previousTodos = get().todos;

    // Optimistic update
    set({ todos: previousTodos.filter((t) => t.id !== id) });

    try {
      await fetch(`/api/todos/${id}`, { method: "DELETE" });
    } catch (error) {
      // Rollback on error
      set({ todos: previousTodos, error: error.message });
    }
  },
}));
```

---

## Middleware

### **Persist**

```typescript
import { persist } from "zustand/middleware";

interface SettingsState {
  theme: "light" | "dark";
  language: string;
  setTheme: (theme: "light" | "dark") => void;
  setLanguage: (lang: string) => void;
}

const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      theme: "light",
      language: "en",
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    {
      name: "settings-storage", // localStorage key
      partialize: (state) => ({ theme: state.theme }), // Only persist theme
    },
  ),
);
```

### **Devtools**

```typescript
import { devtools } from "zustand/middleware";

const useStore = create<State>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () =>
        set((state) => ({ count: state.count + 1 }), false, "increment"),
      // Third parameter is action name for devtools
    }),
    { name: "CounterStore" },
  ),
);
```

### **Immer**

```typescript
import { immer } from "zustand/middleware/immer";

interface NestedState {
  user: {
    profile: {
      name: string;
      age: number;
    };
    settings: {
      notifications: boolean;
    };
  };
  updateName: (name: string) => void;
}

const useStore = create<NestedState>()(
  immer((set) => ({
    user: {
      profile: { name: "John", age: 30 },
      settings: { notifications: true },
    },

    // With immer, mutate directly
    updateName: (name) =>
      set((state) => {
        state.user.profile.name = name;
      }),
  })),
);
```

---

## Slices Pattern

```typescript
// Split large stores into slices
interface UserSlice {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  total: number;
}

const createUserSlice = (set: SetState<Store>): UserSlice => ({
  user: null,
  login: async (email, password) => {
    const user = await api.login(email, password);
    set({ user });
  },
  logout: () => set({ user: null }),
});

const createCartSlice = (
  set: SetState<Store>,
  get: GetState<Store>,
): CartSlice => ({
  items: [],
  addItem: (item) => set({ items: [...get().items, item] }),
  removeItem: (id) => set({ items: get().items.filter((i) => i.id !== id) }),
  get total() {
    return get().items.reduce((sum, item) => sum + item.price, 0);
  },
});

type Store = UserSlice & CartSlice;

const useStore = create<Store>((set, get) => ({
  ...createUserSlice(set),
  ...createCartSlice(set, get),
}));
```

---

## Computed Values

```typescript
interface ProductState {
  products: Product[];
  selectedCategory: string | null;
  searchQuery: string;

  // Computed getter
  get filteredProducts(): Product[] {
    const { products, selectedCategory, searchQuery } = this;

    return products
      .filter(p => !selectedCategory || p.category === selectedCategory)
      .filter(p => p.name.toLowerCase().includes(searchQuery.toLowerCase()));
  };
}

const useProductStore = create<ProductState>((set, get) => ({
  products: [],
  selectedCategory: null,
  searchQuery: '',

  get filteredProducts() {
    const state = get();
    return state.products
      .filter(p => !state.selectedCategory || p.category === state.selectedCategory)
      .filter(p => p.name.toLowerCase().includes(state.searchQuery.toLowerCase()));
  },
}));

// Usage
function ProductList() {
  const filteredProducts = useProductStore((state) => state.filteredProducts);
  return <div>{filteredProducts.map(/* ... */)}</div>;
}
```

---

## Subscribing to Changes

```typescript
const useStore = create<State>(() => ({
  count: 0,
}));

// Subscribe outside React
const unsubscribe = useStore.subscribe((state) =>
  console.log("Count changed:", state.count),
);

// Selective subscription
const unsubscribe = useStore.subscribe(
  (state) => state.count,
  (count) => console.log("Count is now:", count),
);

// Cleanup
unsubscribe();
```

---

## Transient Updates

```typescript
// Updates that don't trigger re-renders
const useStore = create<State>(() => ({
  x: 0,
  y: 0,
}));

// Component doesn't re-render, but state updates
function MouseTracker() {
  useEffect(() => {
    const handleMove = (e: MouseEvent) => {
      // Use getState/setState for transient updates
      useStore.setState({ x: e.clientX, y: e.clientY }, true); // true = replace
    };

    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);

  // This component never re-renders!
  return null;
}

// Another component reads the position
function MousePosition() {
  const position = useStore((state) => ({ x: state.x, y: state.y }), shallow);
  return <div>X: {position.x}, Y: {position.y}</div>;
}
```

---

## Real-World: E-commerce

```typescript
import create from "zustand";
import { persist, devtools } from "zustand/middleware";

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface StoreState {
  // Cart
  cart: CartItem[];
  addToCart: (item: Omit<CartItem, "quantity">) => void;
  removeFromCart: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;

  // Computed
  get total(): number;
  get itemCount(): number;

  // User
  user: User | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;

  // UI
  isCartOpen: boolean;
  toggleCart: () => void;
}

const useStore = create<StoreState>()(
  devtools(
    persist(
      (set, get) => ({
        // Cart state
        cart: [],

        addToCart: (item) => {
          const existing = get().cart.find((i) => i.id === item.id);

          if (existing) {
            set({
              cart: get().cart.map((i) =>
                i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i,
              ),
            });
          } else {
            set({ cart: [...get().cart, { ...item, quantity: 1 }] });
          }
        },

        removeFromCart: (id) => {
          set({ cart: get().cart.filter((i) => i.id !== id) });
        },

        updateQuantity: (id, quantity) => {
          if (quantity <= 0) {
            get().removeFromCart(id);
          } else {
            set({
              cart: get().cart.map((i) =>
                i.id === id ? { ...i, quantity } : i,
              ),
            });
          }
        },

        clearCart: () => set({ cart: [] }),

        // Computed
        get total() {
          return get().cart.reduce(
            (sum, item) => sum + item.price * item.quantity,
            0,
          );
        },

        get itemCount() {
          return get().cart.reduce((sum, item) => sum + item.quantity, 0);
        },

        // User
        user: null,

        login: async (credentials) => {
          const user = await api.login(credentials);
          set({ user });
        },

        logout: () => {
          set({ user: null });
          get().clearCart();
        },

        // UI
        isCartOpen: false,
        toggleCart: () => set({ isCartOpen: !get().isCartOpen }),
      }),
      {
        name: "shop-storage",
        partialize: (state) => ({ cart: state.cart, user: state.user }),
      },
    ),
    { name: "ShopStore" },
  ),
);

export default useStore;
```

---

## Best Practices

✅ **Select only needed state** - prevents unnecessary re-renders  
✅ **Use shallow for multiple properties**  
✅ **Split large stores** with slices pattern  
✅ **Use middleware** for persistence and devtools  
✅ **Compute derived state** with getters  
❌ **Don't select entire state** - kills performance  
❌ **Don't put everything in one store** - split by domain  
❌ **Don't forget cleanup** when subscribing manually

---

## Key Takeaways

1. **Zustand is minimal** - no providers needed
2. **Select state efficiently** to prevent re-renders
3. **Middleware adds persistence** and devtools
4. **Slices pattern scales** large stores
5. **Async actions** are simple functions
6. **Computed values** with getters
7. **Transient updates** don't trigger renders
