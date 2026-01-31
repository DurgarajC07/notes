# State Management Patterns in Frontend

## Core Concepts

State management is critical for building scalable React applications. This guide covers modern state management patterns from Redux to Zustand, helping you choose the right solution for your needs. Understanding these patterns is essential for senior-level interviews and production applications.

## Flux Architecture

### Basic Flux Implementation

```typescript
// Action Types
enum ActionType {
  ADD_TODO = "ADD_TODO",
  TOGGLE_TODO = "TOGGLE_TODO",
  DELETE_TODO = "DELETE_TODO",
}

// Action Creators
interface Action<T = any> {
  type: ActionType;
  payload?: T;
}

const ActionCreators = {
  addTodo: (text: string): Action<{ text: string }> => ({
    type: ActionType.ADD_TODO,
    payload: { text },
  }),

  toggleTodo: (id: string): Action<{ id: string }> => ({
    type: ActionType.TOGGLE_TODO,
    payload: { id },
  }),

  deleteTodo: (id: string): Action<{ id: string }> => ({
    type: ActionType.DELETE_TODO,
    payload: { id },
  }),
};

// Dispatcher
class Dispatcher {
  private callbacks: Array<(action: Action) => void> = [];

  register(callback: (action: Action) => void): () => void {
    this.callbacks.push(callback);
    return () => {
      const index = this.callbacks.indexOf(callback);
      if (index > -1) {
        this.callbacks.splice(index, 1);
      }
    };
  }

  dispatch(action: Action): void {
    this.callbacks.forEach((callback) => callback(action));
  }
}

// Store
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

class TodoStore {
  private todos: Todo[] = [];
  private listeners: Set<() => void> = new Set();

  constructor(private dispatcher: Dispatcher) {
    dispatcher.register((action) => {
      switch (action.type) {
        case ActionType.ADD_TODO:
          this.todos.push({
            id: Date.now().toString(),
            text: action.payload.text,
            completed: false,
          });
          this.emitChange();
          break;

        case ActionType.TOGGLE_TODO:
          const todo = this.todos.find((t) => t.id === action.payload.id);
          if (todo) {
            todo.completed = !todo.completed;
            this.emitChange();
          }
          break;

        case ActionType.DELETE_TODO:
          this.todos = this.todos.filter((t) => t.id !== action.payload.id);
          this.emitChange();
          break;
      }
    });
  }

  getTodos(): Todo[] {
    return [...this.todos];
  }

  subscribe(listener: () => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  private emitChange(): void {
    this.listeners.forEach((listener) => listener());
  }
}

// Usage
const dispatcher = new Dispatcher();
const todoStore = new TodoStore(dispatcher);

todoStore.subscribe(() => {
  console.log("Todos updated:", todoStore.getTodos());
});

dispatcher.dispatch(ActionCreators.addTodo("Learn Flux"));
dispatcher.dispatch(ActionCreators.addTodo("Build an app"));
```

## Redux Pattern

### Complete Redux Implementation

```typescript
// Action Types
const ADD_TODO = "todos/add";
const TOGGLE_TODO = "todos/toggle";
const DELETE_TODO = "todos/delete";
const FETCH_TODOS = "todos/fetch";
const FETCH_TODOS_SUCCESS = "todos/fetchSuccess";
const FETCH_TODOS_ERROR = "todos/fetchError";

// Action Interfaces
interface AddTodoAction {
  type: typeof ADD_TODO;
  payload: { text: string };
}

interface ToggleTodoAction {
  type: typeof TOGGLE_TODO;
  payload: { id: string };
}

interface DeleteTodoAction {
  type: typeof DELETE_TODO;
  payload: { id: string };
}

interface FetchTodosAction {
  type: typeof FETCH_TODOS;
}

interface FetchTodosSuccessAction {
  type: typeof FETCH_TODOS_SUCCESS;
  payload: Todo[];
}

interface FetchTodosErrorAction {
  type: typeof FETCH_TODOS_ERROR;
  payload: { error: string };
}

type TodoAction =
  | AddTodoAction
  | ToggleTodoAction
  | DeleteTodoAction
  | FetchTodosAction
  | FetchTodosSuccessAction
  | FetchTodosErrorAction;

// Action Creators
const todoActions = {
  addTodo: (text: string): AddTodoAction => ({
    type: ADD_TODO,
    payload: { text },
  }),

  toggleTodo: (id: string): ToggleTodoAction => ({
    type: TOGGLE_TODO,
    payload: { id },
  }),

  deleteTodo: (id: string): DeleteTodoAction => ({
    type: DELETE_TODO,
    payload: { id },
  }),

  fetchTodos: (): FetchTodosAction => ({
    type: FETCH_TODOS,
  }),

  fetchTodosSuccess: (todos: Todo[]): FetchTodosSuccessAction => ({
    type: FETCH_TODOS_SUCCESS,
    payload: todos,
  }),

  fetchTodosError: (error: string): FetchTodosErrorAction => ({
    type: FETCH_TODOS_ERROR,
    payload: { error },
  }),
};

// State Interface
interface TodoState {
  items: Todo[];
  loading: boolean;
  error: string | null;
}

const initialState: TodoState = {
  items: [],
  loading: false,
  error: null,
};

// Reducer
function todoReducer(state = initialState, action: TodoAction): TodoState {
  switch (action.type) {
    case ADD_TODO:
      return {
        ...state,
        items: [
          ...state.items,
          {
            id: Date.now().toString(),
            text: action.payload.text,
            completed: false,
          },
        ],
      };

    case TOGGLE_TODO:
      return {
        ...state,
        items: state.items.map((todo) =>
          todo.id === action.payload.id
            ? { ...todo, completed: !todo.completed }
            : todo,
        ),
      };

    case DELETE_TODO:
      return {
        ...state,
        items: state.items.filter((todo) => todo.id !== action.payload.id),
      };

    case FETCH_TODOS:
      return {
        ...state,
        loading: true,
        error: null,
      };

    case FETCH_TODOS_SUCCESS:
      return {
        ...state,
        items: action.payload,
        loading: false,
        error: null,
      };

    case FETCH_TODOS_ERROR:
      return {
        ...state,
        loading: false,
        error: action.payload.error,
      };

    default:
      return state;
  }
}

// Store
interface Store<S> {
  getState(): S;
  dispatch(action: any): any;
  subscribe(listener: () => void): () => void;
}

function createStore<S>(
  reducer: (state: S, action: any) => S,
  initialState: S,
): Store<S> {
  let state = initialState;
  const listeners: Set<() => void> = new Set();

  return {
    getState: () => state,

    dispatch: (action: any) => {
      state = reducer(state, action);
      listeners.forEach((listener) => listener());
      return action;
    },

    subscribe: (listener: () => void) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}

// Middleware
type Middleware<S> = (
  store: Store<S>,
) => (next: (action: any) => any) => (action: any) => any;

const loggerMiddleware: Middleware<any> = (store) => (next) => (action) => {
  console.log("Dispatching:", action);
  const result = next(action);
  console.log("New state:", store.getState());
  return result;
};

const thunkMiddleware: Middleware<any> = (store) => (next) => (action) => {
  if (typeof action === "function") {
    return action(store.dispatch, store.getState);
  }
  return next(action);
};

function applyMiddleware<S>(...middlewares: Middleware<S>[]) {
  return (createStore: any) => (reducer: any, initialState: S) => {
    const store = createStore(reducer, initialState);
    let dispatch = store.dispatch;

    middlewares.forEach((middleware) => {
      dispatch = middleware(store)(dispatch);
    });

    return { ...store, dispatch };
  };
}

// Thunk Action Creator
const fetchTodosThunk = () => async (dispatch: any) => {
  dispatch(todoActions.fetchTodos());

  try {
    const response = await fetch("/api/todos");
    const todos = await response.json();
    dispatch(todoActions.fetchTodosSuccess(todos));
  } catch (error) {
    dispatch(todoActions.fetchTodosError((error as Error).message));
  }
};

// Usage with React
function useTodos() {
  const [state, setState] = React.useState(store.getState());

  React.useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      setState(store.getState());
    });
    return unsubscribe;
  }, []);

  return {
    todos: state.items,
    loading: state.loading,
    error: state.error,
    addTodo: (text: string) => store.dispatch(todoActions.addTodo(text)),
    toggleTodo: (id: string) => store.dispatch(todoActions.toggleTodo(id)),
    deleteTodo: (id: string) => store.dispatch(todoActions.deleteTodo(id)),
    fetchTodos: () => store.dispatch(fetchTodosThunk()),
  };
}

const store = applyMiddleware(thunkMiddleware, loggerMiddleware)(createStore)(
  todoReducer,
  initialState,
);
```

### Redux Toolkit (Modern Redux)

```typescript
import { createSlice, configureStore, PayloadAction, createAsyncThunk } from '@reduxjs/toolkit';

// Async Thunk
export const fetchTodos = createAsyncThunk(
  'todos/fetchTodos',
  async () => {
    const response = await fetch('/api/todos');
    return response.json();
  }
);

// Slice
const todosSlice = createSlice({
  name: 'todos',
  initialState: {
    items: [] as Todo[],
    loading: false,
    error: null as string | null,
  },
  reducers: {
    addTodo: (state, action: PayloadAction<{ text: string }>) => {
      state.items.push({
        id: Date.now().toString(),
        text: action.payload.text,
        completed: false,
      });
    },

    toggleTodo: (state, action: PayloadAction<{ id: string }>) => {
      const todo = state.items.find(t => t.id === action.payload.id);
      if (todo) {
        todo.completed = !todo.completed;
      }
    },

    deleteTodo: (state, action: PayloadAction<{ id: string }>) => {
      state.items = state.items.filter(t => t.id !== action.payload.id);
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchTodos.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to fetch';
      });
  },
});

export const { addTodo, toggleTodo, deleteTodo } = todosSlice.actions;

// Store
export const store = configureStore({
  reducer: {
    todos: todosSlice.reducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// React Hooks
import { useDispatch, useSelector } from 'react-redux';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector = <T>(selector: (state: RootState) => T) =>
  useSelector<RootState, T>(selector);

// Component
function TodoList() {
  const dispatch = useAppDispatch();
  const { items, loading } = useAppSelector(state => state.todos);

  React.useEffect(() => {
    dispatch(fetchTodos());
  }, [dispatch]);

  return (
    <div>
      {loading ? (
        <div>Loading...</div>
      ) : (
        <ul>
          {items.map(todo => (
            <li key={todo.id}>
              <input
                type="checkbox"
                checked={todo.completed}
                onChange={() => dispatch(toggleTodo({ id: todo.id }))}
              />
              <span>{todo.text}</span>
              <button onClick={() => dispatch(deleteTodo({ id: todo.id }))}>
                Delete
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## Zustand (Lightweight Alternative)

### Basic Zustand Store

```typescript
import create from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface TodoStore {
  todos: Todo[];
  loading: boolean;
  error: string | null;

  addTodo: (text: string) => void;
  toggleTodo: (id: string) => void;
  deleteTodo: (id: string) => void;
  fetchTodos: () => Promise<void>;
}

const useTodoStore = create<TodoStore>()(
  devtools(
    persist(
      (set, get) => ({
        todos: [],
        loading: false,
        error: null,

        addTodo: (text: string) => {
          set((state) => ({
            todos: [
              ...state.todos,
              {
                id: Date.now().toString(),
                text,
                completed: false,
              },
            ],
          }));
        },

        toggleTodo: (id: string) => {
          set((state) => ({
            todos: state.todos.map(todo =>
              todo.id === id ? { ...todo, completed: !todo.completed } : todo
            ),
          }));
        },

        deleteTodo: (id: string) => {
          set((state) => ({
            todos: state.todos.filter(todo => todo.id !== id),
          }));
        },

        fetchTodos: async () => {
          set({ loading: true, error: null });

          try {
            const response = await fetch('/api/todos');
            const todos = await response.json();
            set({ todos, loading: false });
          } catch (error) {
            set({ error: (error as Error).message, loading: false });
          }
        },
      }),
      {
        name: 'todo-storage',
        partialize: (state) => ({ todos: state.todos }),
      }
    )
  )
);

// Usage in Component
function TodoList() {
  const { todos, loading, addTodo, toggleTodo, deleteTodo, fetchTodos } = useTodoStore();

  React.useEffect(() => {
    fetchTodos();
  }, [fetchTodos]);

  const [inputValue, setInputValue] = React.useState('');

  const handleAdd = () => {
    if (inputValue.trim()) {
      addTodo(inputValue);
      setInputValue('');
    }
  };

  return (
    <div>
      <input
        value={inputValue}
        onChange={(e) => setInputValue(e.target.value)}
        placeholder="Add todo"
      />
      <button onClick={handleAdd}>Add</button>

      {loading ? (
        <div>Loading...</div>
      ) : (
        <ul>
          {todos.map(todo => (
            <li key={todo.id}>
              <input
                type="checkbox"
                checked={todo.completed}
                onChange={() => toggleTodo(todo.id)}
              />
              <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
                {todo.text}
              </span>
              <button onClick={() => deleteTodo(todo.id)}>Delete</button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

// Selective subscription
function TodoCount() {
  const count = useTodoStore((state) => state.todos.length);
  return <div>Total: {count}</div>;
}
```

### Advanced Zustand with Slices

```typescript
interface UserSlice {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

interface SettingsSlice {
  theme: 'light' | 'dark';
  language: string;
  setTheme: (theme: 'light' | 'dark') => void;
  setLanguage: (language: string) => void;
}

const createUserSlice = (set: any): UserSlice => ({
  user: null,

  login: async (email, password) => {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });
    const user = await response.json();
    set({ user });
  },

  logout: () => {
    set({ user: null });
  },
});

const createSettingsSlice = (set: any): SettingsSlice => ({
  theme: 'light',
  language: 'en',

  setTheme: (theme) => set({ theme }),
  setLanguage: (language) => set({ language }),
});

type StoreState = UserSlice & SettingsSlice;

const useStore = create<StoreState>()((...a) => ({
  ...createUserSlice(...a),
  ...createSettingsSlice(...a),
}));

// Usage
function Profile() {
  const user = useStore((state) => state.user);
  const logout = useStore((state) => state.logout);

  return (
    <div>
      <h2>{user?.name}</h2>
      <button onClick={logout}>Logout</button>
    </div>
  );
}

function ThemeSwitcher() {
  const theme = useStore((state) => state.theme);
  const setTheme = useStore((state) => state.setTheme);

  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Toggle Theme
    </button>
  );
}
```

## Jotai (Atomic State Management)

### Basic Jotai Atoms

```typescript
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';

// Primitive atoms
const todosAtom = atom<Todo[]>([]);
const filterAtom = atom<'all' | 'active' | 'completed'>('all');

// Derived atom (computed)
const filteredTodosAtom = atom((get) => {
  const todos = get(todosAtom);
  const filter = get(filterAtom);

  if (filter === 'all') return todos;
  if (filter === 'active') return todos.filter(t => !t.completed);
  return todos.filter(t => t.completed);
});

// Async atom
const todosAPIAtom = atom(async (get) => {
  const response = await fetch('/api/todos');
  return response.json();
});

// Write-only atom (action)
const addTodoAtom = atom(
  null,
  (get, set, text: string) => {
    const todos = get(todosAtom);
    set(todosAtom, [
      ...todos,
      {
        id: Date.now().toString(),
        text,
        completed: false,
      },
    ]);
  }
);

const toggleTodoAtom = atom(
  null,
  (get, set, id: string) => {
    const todos = get(todosAtom);
    set(
      todosAtom,
      todos.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  }
);

// Usage in Component
function TodoList() {
  const [todos, setTodos] = useAtom(todosAtom);
  const filteredTodos = useAtomValue(filteredTodosAtom);
  const addTodo = useSetAtom(addTodoAtom);
  const toggleTodo = useSetAtom(toggleTodoAtom);

  const [inputValue, setInputValue] = React.useState('');

  const handleAdd = () => {
    if (inputValue.trim()) {
      addTodo(inputValue);
      setInputValue('');
    }
  };

  return (
    <div>
      <input
        value={inputValue}
        onChange={(e) => setInputValue(e.target.value)}
        placeholder="Add todo"
      />
      <button onClick={handleAdd}>Add</button>

      <ul>
        {filteredTodos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span>{todo.text}</span>
          </li>
        ))}
      </ul>
    </div>
  );
}

function FilterButtons() {
  const [filter, setFilter] = useAtom(filterAtom);

  return (
    <div>
      <button onClick={() => setFilter('all')}>All</button>
      <button onClick={() => setFilter('active')}>Active</button>
      <button onClick={() => setFilter('completed')}>Completed</button>
    </div>
  );
}
```

## Recoil (Facebook's State Management)

### Recoil Atoms and Selectors

```typescript
import { atom, selector, useRecoilState, useRecoilValue, useSetRecoilState } from 'recoil';

// Atoms
const todosState = atom<Todo[]>({
  key: 'todosState',
  default: [],
});

const filterState = atom<'all' | 'active' | 'completed'>({
  key: 'filterState',
  default: 'all',
});

// Selectors (derived state)
const filteredTodosState = selector({
  key: 'filteredTodosState',
  get: ({ get }) => {
    const todos = get(todosState);
    const filter = get(filterState);

    switch (filter) {
      case 'active':
        return todos.filter(t => !t.completed);
      case 'completed':
        return todos.filter(t => t.completed);
      default:
        return todos;
    }
  },
});

const todoStatsState = selector({
  key: 'todoStatsState',
  get: ({ get }) => {
    const todos = get(todosState);

    return {
      total: todos.length,
      completed: todos.filter(t => t.completed).length,
      active: todos.filter(t => !t.completed).length,
    };
  },
});

// Async selector
const todosAPIState = selector({
  key: 'todosAPIState',
  get: async () => {
    const response = await fetch('/api/todos');
    return response.json();
  },
});

// Usage
function TodoList() {
  const [todos, setTodos] = useRecoilState(todosState);
  const filteredTodos = useRecoilValue(filteredTodosState);

  const addTodo = (text: string) => {
    setTodos(prev => [
      ...prev,
      {
        id: Date.now().toString(),
        text,
        completed: false,
      },
    ]);
  };

  const toggleTodo = (id: string) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  };

  return (
    <ul>
      {filteredTodos.map(todo => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => toggleTodo(todo.id)}
          />
          <span>{todo.text}</span>
        </li>
      ))}
    </ul>
  );
}

function TodoStats() {
  const stats = useRecoilValue(todoStatsState);

  return (
    <div>
      <p>Total: {stats.total}</p>
      <p>Completed: {stats.completed}</p>
      <p>Active: {stats.active}</p>
    </div>
  );
}
```

## State Machines (XState)

### Traffic Light State Machine

```typescript
import { createMachine, interpret } from 'xstate';

const trafficLightMachine = createMachine({
  id: 'trafficLight',
  initial: 'red',
  states: {
    red: {
      after: {
        5000: 'green',
      },
    },
    yellow: {
      after: {
        2000: 'red',
      },
    },
    green: {
      after: {
        10000: 'yellow',
      },
    },
  },
});

// Usage with React
import { useMachine } from '@xstate/react';

function TrafficLight() {
  const [state, send] = useMachine(trafficLightMachine);

  return (
    <div>
      <div className={`light red ${state.matches('red') ? 'active' : ''}`} />
      <div className={`light yellow ${state.matches('yellow') ? 'active' : ''}`} />
      <div className={`light green ${state.matches('green') ? 'active' : ''}`} />
    </div>
  );
}
```

### Form Wizard State Machine

```typescript
const formMachine = createMachine({
  id: 'form',
  initial: 'personalInfo',
  context: {
    personalInfo: {},
    address: {},
    payment: {},
  },
  states: {
    personalInfo: {
      on: {
        NEXT: {
          target: 'address',
          actions: 'savePersonalInfo',
        },
      },
    },
    address: {
      on: {
        NEXT: {
          target: 'payment',
          actions: 'saveAddress',
        },
        BACK: 'personalInfo',
      },
    },
    payment: {
      on: {
        SUBMIT: {
          target: 'submitting',
          actions: 'savePayment',
        },
        BACK: 'address',
      },
    },
    submitting: {
      invoke: {
        src: 'submitForm',
        onDone: 'success',
        onError: 'error',
      },
    },
    success: {
      type: 'final',
    },
    error: {
      on: {
        RETRY: 'submitting',
      },
    },
  },
}, {
  actions: {
    savePersonalInfo: (context, event) => {
      context.personalInfo = event.data;
    },
    saveAddress: (context, event) => {
      context.address = event.data;
    },
    savePayment: (context, event) => {
      context.payment = event.data;
    },
  },
  services: {
    submitForm: async (context) => {
      const response = await fetch('/api/form', {
        method: 'POST',
        body: JSON.stringify(context),
      });
      return response.json();
    },
  },
});

function FormWizard() {
  const [state, send] = useMachine(formMachine);

  return (
    <div>
      {state.matches('personalInfo') && (
        <PersonalInfoForm onNext={(data) => send({ type: 'NEXT', data })} />
      )}
      {state.matches('address') && (
        <AddressForm
          onNext={(data) => send({ type: 'NEXT', data })}
          onBack={() => send('BACK')}
        />
      )}
      {state.matches('payment') && (
        <PaymentForm
          onSubmit={(data) => send({ type: 'SUBMIT', data })}
          onBack={() => send('BACK')}
        />
      )}
      {state.matches('submitting') && <div>Submitting...</div>}
      {state.matches('success') && <div>Form submitted successfully!</div>}
      {state.matches('error') && (
        <div>
          Error submitting form
          <button onClick={() => send('RETRY')}>Retry</button>
        </div>
      )}
    </div>
  );
}
```

## Context + Reducer Pattern

### Advanced Context with Reducer

```typescript
interface State {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
  loading: boolean;
  error: string | null;
}

type Action =
  | { type: 'ADD_TODO'; payload: { text: string } }
  | { type: 'TOGGLE_TODO'; payload: { id: string } }
  | { type: 'DELETE_TODO'; payload: { id: string } }
  | { type: 'SET_FILTER'; payload: { filter: State['filter'] } }
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: { todos: Todo[] } }
  | { type: 'FETCH_ERROR'; payload: { error: string } };

function todoReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [
          ...state.todos,
          {
            id: Date.now().toString(),
            text: action.payload.text,
            completed: false,
          },
        ],
      };

    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload.id
            ? { ...todo, completed: !todo.completed }
            : todo
        ),
      };

    case 'DELETE_TODO':
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.payload.id),
      };

    case 'SET_FILTER':
      return {
        ...state,
        filter: action.payload.filter,
      };

    case 'FETCH_START':
      return {
        ...state,
        loading: true,
        error: null,
      };

    case 'FETCH_SUCCESS':
      return {
        ...state,
        loading: false,
        todos: action.payload.todos,
      };

    case 'FETCH_ERROR':
      return {
        ...state,
        loading: false,
        error: action.payload.error,
      };

    default:
      return state;
  }
}

const TodoContext = React.createContext<{
  state: State;
  dispatch: React.Dispatch<Action>;
} | undefined>(undefined);

function TodoProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = React.useReducer(todoReducer, {
    todos: [],
    filter: 'all',
    loading: false,
    error: null,
  });

  return (
    <TodoContext.Provider value={{ state, dispatch }}>
      {children}
    </TodoContext.Provider>
  );
}

function useTodoContext() {
  const context = React.useContext(TodoContext);
  if (!context) {
    throw new Error('useTodoContext must be used within TodoProvider');
  }
  return context;
}

// Usage
function TodoList() {
  const { state, dispatch } = useTodoContext();

  const filteredTodos = React.useMemo(() => {
    if (state.filter === 'all') return state.todos;
    if (state.filter === 'active') return state.todos.filter(t => !t.completed);
    return state.todos.filter(t => t.completed);
  }, [state.todos, state.filter]);

  return (
    <ul>
      {filteredTodos.map(todo => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() =>
              dispatch({ type: 'TOGGLE_TODO', payload: { id: todo.id } })
            }
          />
          <span>{todo.text}</span>
          <button
            onClick={() =>
              dispatch({ type: 'DELETE_TODO', payload: { id: todo.id } })
            }
          >
            Delete
          </button>
        </li>
      ))}
    </ul>
  );
}
```

## State Management Comparison Matrix

```typescript
interface StateLibraryComparison {
  name: string;
  boilerplate: "low" | "medium" | "high";
  learningCurve: "easy" | "moderate" | "steep";
  performance: "good" | "excellent";
  devTools: boolean;
  asyncSupport: "built-in" | "requires-middleware" | "external";
  typescript: "excellent" | "good" | "fair";
  bundleSize: string;
  bestFor: string[];
}

const stateLibraries: StateLibraryComparison[] = [
  {
    name: "Redux",
    boilerplate: "high",
    learningCurve: "steep",
    performance: "good",
    devTools: true,
    asyncSupport: "requires-middleware",
    typescript: "excellent",
    bundleSize: "~46KB",
    bestFor: ["Large apps", "Complex state", "Time-travel debugging"],
  },
  {
    name: "Redux Toolkit",
    boilerplate: "medium",
    learningCurve: "moderate",
    performance: "good",
    devTools: true,
    asyncSupport: "built-in",
    typescript: "excellent",
    bundleSize: "~52KB",
    bestFor: ["Modern Redux", "Reduced boilerplate", "Best practices"],
  },
  {
    name: "Zustand",
    boilerplate: "low",
    learningCurve: "easy",
    performance: "excellent",
    devTools: true,
    asyncSupport: "built-in",
    typescript: "excellent",
    bundleSize: "~1.2KB",
    bestFor: ["Simple state", "Quick setup", "Small bundle"],
  },
  {
    name: "Jotai",
    boilerplate: "low",
    learningCurve: "easy",
    performance: "excellent",
    devTools: true,
    asyncSupport: "built-in",
    typescript: "excellent",
    bundleSize: "~2.9KB",
    bestFor: ["Atomic updates", "Derived state", "Bottom-up approach"],
  },
  {
    name: "Recoil",
    boilerplate: "low",
    learningCurve: "moderate",
    performance: "excellent",
    devTools: true,
    asyncSupport: "built-in",
    typescript: "excellent",
    bundleSize: "~14KB",
    bestFor: ["Complex dependencies", "Facebook ecosystem", "Concurrent mode"],
  },
  {
    name: "Context + Reducer",
    boilerplate: "medium",
    learningCurve: "easy",
    performance: "good",
    devTools: false,
    asyncSupport: "external",
    typescript: "good",
    bundleSize: "0KB (built-in)",
    bestFor: ["Simple apps", "No dependencies", "Learning React"],
  },
  {
    name: "XState",
    boilerplate: "medium",
    learningCurve: "steep",
    performance: "excellent",
    devTools: true,
    asyncSupport: "built-in",
    typescript: "excellent",
    bundleSize: "~11KB",
    bestFor: ["State machines", "Complex workflows", "Predictable transitions"],
  },
];
```

## Best Practices

1. **Choose based on complexity** - Context for simple, Redux/Zustand for complex apps
2. **Normalize state shape** - Avoid nested structures, use IDs for relationships
3. **Keep state minimal** - Don't store derived data, compute it on demand
4. **Use selectors** - Memoize derived state to prevent unnecessary re-renders
5. **Split concerns** - Separate UI state from server state
6. **Immutable updates** - Always return new objects/arrays in reducers
7. **Type safety** - Use TypeScript for actions and state shape
8. **DevTools** - Use Redux DevTools or equivalent for debugging
9. **Async patterns** - Use thunks, sagas, or built-in async support properly
10. **Performance** - Profile and optimize re-renders with React.memo and selectors

## 10 Key Takeaways

1. **Redux** is powerful but verbose - use Redux Toolkit for modern apps
2. **Zustand** offers minimal API with excellent performance and small bundle
3. **Jotai** provides atomic state management with bottom-up approach
4. **Recoil** excels at derived state and complex dependencies
5. **Context + Reducer** is built-in, good for simple apps without dependencies
6. **XState** is ideal for state machines and complex workflows
7. **Bundle size matters** - consider Zustand/Jotai for smaller apps
8. **TypeScript support** varies - Redux Toolkit, Zustand, Jotai excel at it
9. **DevTools** are essential for debugging - most libraries support them
10. **No silver bullet** - choose based on app complexity, team experience, requirements
