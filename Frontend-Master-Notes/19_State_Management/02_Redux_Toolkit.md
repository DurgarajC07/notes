# Redux Toolkit (RTK)

## Core Concept

Redux Toolkit is the **official, opinionated toolset** for Redux that simplifies store setup, reduces boilerplate, and includes best practices by default.

---

## Installation

```bash
npm install @reduxjs/toolkit react-redux
```

---

## configureStore

Simplified store setup with good defaults:

```typescript
import { configureStore } from "@reduxjs/toolkit";
import counterReducer from "./features/counter/counterSlice";
import userReducer from "./features/user/userSlice";

const store = configureStore({
  reducer: {
    counter: counterReducer,
    user: userReducer,
  },
  // Automatically includes:
  // - Redux DevTools Extension
  // - Thunk middleware
  // - Serializable check middleware (dev only)
  // - Immutability check middleware (dev only)
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

---

## createSlice

Define reducer logic and actions together:

```typescript
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

interface CounterState {
  value: number;
  status: "idle" | "loading" | "failed";
}

const initialState: CounterState = {
  value: 0,
  status: "idle",
};

const counterSlice = createSlice({
  name: "counter",
  initialState,
  reducers: {
    increment: (state) => {
      // Immer allows "mutating" syntax
      state.value += 1;
    },
    decrement: (state) => {
      state.value -= 1;
    },
    incrementByAmount: (state, action: PayloadAction<number>) => {
      state.value += action.payload;
    },
    reset: (state) => {
      state.value = 0;
    },
  },
});

export const { increment, decrement, incrementByAmount, reset } =
  counterSlice.actions;
export default counterSlice.reducer;
```

---

## Using in Components

```typescript
import { useSelector, useDispatch } from 'react-redux';
import { increment, decrement, incrementByAmount } from './counterSlice';
import type { RootState } from '../../store';

function Counter() {
  const count = useSelector((state: RootState) => state.counter.value);
  const dispatch = useDispatch();

  return (
    <div>
      <h1>{count}</h1>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(decrement())}>-</button>
      <button onClick={() => dispatch(incrementByAmount(5))}>+5</button>
    </div>
  );
}
```

---

## createAsyncThunk

Handle async logic with automatic loading states:

```typescript
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";

interface User {
  id: number;
  name: string;
}

interface UserState {
  user: User | null;
  loading: "idle" | "pending" | "succeeded" | "failed";
  error: string | null;
}

// Async thunk
export const fetchUserById = createAsyncThunk(
  "users/fetchById",
  async (userId: number) => {
    const response = await fetch(`/api/users/${userId}`);
    return (await response.json()) as User;
  },
);

const userSlice = createSlice({
  name: "user",
  initialState: {
    user: null,
    loading: "idle",
    error: null,
  } as UserState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchUserById.pending, (state) => {
        state.loading = "pending";
        state.error = null;
      })
      .addCase(fetchUserById.fulfilled, (state, action) => {
        state.loading = "succeeded";
        state.user = action.payload;
      })
      .addCase(fetchUserById.rejected, (state, action) => {
        state.loading = "failed";
        state.error = action.error.message || "Failed to fetch user";
      });
  },
});

export default userSlice.reducer;
```

---

## Using Async Thunks

```typescript
function UserProfile({ userId }: { userId: number }) {
  const dispatch = useDispatch();
  const { user, loading, error } = useSelector((state: RootState) => state.user);

  useEffect(() => {
    dispatch(fetchUserById(userId));
  }, [userId, dispatch]);

  if (loading === 'pending') return <div>Loading...</div>;
  if (loading === 'failed') return <div>Error: {error}</div>;
  if (!user) return null;

  return <div>{user.name}</div>;
}
```

---

## RTK Query

Powerful data fetching and caching:

```typescript
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

interface Post {
  id: number;
  title: string;
  body: string;
}

export const postsApi = createApi({
  reducerPath: "postsApi",
  baseQuery: fetchBaseQuery({ baseUrl: "/api" }),
  tagTypes: ["Post"],
  endpoints: (builder) => ({
    getPosts: builder.query<Post[], void>({
      query: () => "/posts",
      providesTags: ["Post"],
    }),
    getPostById: builder.query<Post, number>({
      query: (id) => `/posts/${id}`,
      providesTags: (result, error, id) => [{ type: "Post", id }],
    }),
    createPost: builder.mutation<Post, Partial<Post>>({
      query: (body) => ({
        url: "/posts",
        method: "POST",
        body,
      }),
      invalidatesTags: ["Post"],
    }),
    updatePost: builder.mutation<Post, { id: number; body: Partial<Post> }>({
      query: ({ id, body }) => ({
        url: `/posts/${id}`,
        method: "PUT",
        body,
      }),
      invalidatesTags: (result, error, { id }) => [{ type: "Post", id }],
    }),
    deletePost: builder.mutation<void, number>({
      query: (id) => ({
        url: `/posts/${id}`,
        method: "DELETE",
      }),
      invalidatesTags: ["Post"],
    }),
  }),
});

export const {
  useGetPostsQuery,
  useGetPostByIdQuery,
  useCreatePostMutation,
  useUpdatePostMutation,
  useDeletePostMutation,
} = postsApi;
```

---

## Using RTK Query

```typescript
// Add to store
import { postsApi } from './postsApi';

const store = configureStore({
  reducer: {
    [postsApi.reducerPath]: postsApi.reducer,
    // other reducers
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(postsApi.middleware)
});

// Use in components
function PostsList() {
  const { data: posts, error, isLoading } = useGetPostsQuery();

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading posts</div>;

  return (
    <ul>
      {posts?.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}

function CreatePost() {
  const [createPost, { isLoading }] = useCreatePostMutation();

  const handleSubmit = async (data: Partial<Post>) => {
    try {
      await createPost(data).unwrap();
      alert('Post created!');
    } catch (error) {
      alert('Failed to create post');
    }
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

---

## createEntityAdapter

Manage normalized data:

```typescript
import { createEntityAdapter, createSlice } from "@reduxjs/toolkit";

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

const todosAdapter = createEntityAdapter<Todo>({
  selectId: (todo) => todo.id,
  sortComparer: (a, b) => a.text.localeCompare(b.text),
});

const todosSlice = createSlice({
  name: "todos",
  initialState: todosAdapter.getInitialState(),
  reducers: {
    todoAdded: todosAdapter.addOne,
    todosReceived: todosAdapter.setAll,
    todoUpdated: todosAdapter.updateOne,
    todoRemoved: todosAdapter.removeOne,
  },
});

export const { todoAdded, todosReceived, todoUpdated, todoRemoved } =
  todosSlice.actions;

// Selectors
export const {
  selectAll: selectAllTodos,
  selectById: selectTodoById,
  selectIds: selectTodoIds,
} = todosAdapter.getSelectors((state: RootState) => state.todos);
```

---

## TypeScript Hooks

```typescript
import { TypedUseSelectorHook, useDispatch, useSelector } from "react-redux";
import type { RootState, AppDispatch } from "./store";

// Use throughout app instead of plain `useDispatch` and `useSelector`
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

---

## Best Practices

✅ **Use Redux Toolkit** - reduces boilerplate  
✅ **One slice per feature** - modular organization  
✅ **Use createAsyncThunk** for async logic  
✅ **Use RTK Query** for server state  
✅ **TypeScript types** for safety  
✅ **Normalize data** with createEntityAdapter  
❌ **Don't use vanilla Redux** - RTK is better  
❌ **Don't store derived data** - use selectors

---

## Key Takeaways

1. **Redux Toolkit simplifies Redux** significantly
2. **createSlice combines** reducers and actions
3. **Immer enables** "mutating" syntax safely
4. **createAsyncThunk handles** async with loading states
5. **RTK Query provides** data fetching/caching
6. **createEntityAdapter** for normalized data
7. **TypeScript integration** is first-class
