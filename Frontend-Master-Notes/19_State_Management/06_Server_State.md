# Server State with React Query and SWR

## Core Concept

Server state differs from client state - it's remote, async, potentially stale, and shared across users. React Query and SWR specialize in caching, synchronizing, and managing server data with automatic refetching, background updates, and optimistic mutations.

---

## React Query Basics

```typescript
import { useQuery } from '@tanstack/react-query';

interface User {
  id: string;
  name: string;
  email: string;
}

// Fetch users
function UserList() {
  const { data, isLoading, error } = useQuery<User[]>({
    queryKey: ['users'],
    queryFn: async () => {
      const response = await fetch('/api/users');
      if (!response.ok) throw new Error('Failed to fetch');
      return response.json();
    }
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {data.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

---

## Query Keys and Caching

```typescript
// Simple key
useQuery({ queryKey: ['users'], queryFn: fetchUsers });

// Parameterized key
useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId)
});

// Hierarchical key for filtering
useQuery({
  queryKey: ['users', { status: 'active', role: 'admin' }],
  queryFn: () => fetchUsers({ status: 'active', role: 'admin' })
});

// Dependent queries
function UserPosts({ userId }: { userId: string }) {
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId)
  });

  const { data: posts } = useQuery({
    queryKey: ['posts', userId],
    queryFn: () => fetchPosts(userId),
    enabled: !!user // Only run when user is loaded
  });

  return <div>{/* Render posts */}</div>;
}
```

---

## Mutations

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreateUser() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: async (newUser: Partial<User>) => {
      const response = await fetch('/api/users', {
        method: 'POST',
        body: JSON.stringify(newUser)
      });
      return response.json();
    },
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['users'] });
    }
  });

  return (
    <button
      onClick={() => {
        mutation.mutate({ name: 'John', email: 'john@example.com' });
      }}
      disabled={mutation.isPending}
    >
      {mutation.isPending ? 'Creating...' : 'Create User'}
    </button>
  );
}
```

---

## Optimistic Updates

```typescript
function UpdateUser({ userId }: { userId: string }) {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: async (updatedUser: Partial<User>) => {
      const response = await fetch(`/api/users/${userId}`, {
        method: 'PUT',
        body: JSON.stringify(updatedUser)
      });
      return response.json();
    },
    onMutate: async (newUser) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['user', userId] });

      // Snapshot previous value
      const previousUser = queryClient.getQueryData(['user', userId]);

      // Optimistically update
      queryClient.setQueryData(['user', userId], (old: User) => ({
        ...old,
        ...newUser
      }));

      // Return context with snapshot
      return { previousUser };
    },
    onError: (err, newUser, context) => {
      // Rollback on error
      queryClient.setQueryData(['user', userId], context?.previousUser);
    },
    onSettled: () => {
      // Always refetch after error or success
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    }
  });

  return (
    <button onClick={() => mutation.mutate({ name: 'Updated Name' })}>
      Update
    </button>
  );
}
```

---

## Pagination

```typescript
function UsersPaginated() {
  const [page, setPage] = useState(1);

  const { data, isLoading, isPlaceholderData } = useQuery({
    queryKey: ['users', page],
    queryFn: () => fetchUsers(page),
    placeholderData: keepPreviousData // Keep previous data while fetching
  });

  return (
    <div>
      {data?.users.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}

      <button
        onClick={() => setPage(p => Math.max(1, p - 1))}
        disabled={page === 1}
      >
        Previous
      </button>

      <button
        onClick={() => setPage(p => p + 1)}
        disabled={isPlaceholderData || !data?.hasMore}
      >
        Next
      </button>
    </div>
  );
}
```

---

## Infinite Queries

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

function InfiniteUsers() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage
  } = useInfiniteQuery({
    queryKey: ['users', 'infinite'],
    queryFn: async ({ pageParam = 0 }) => {
      const response = await fetch(`/api/users?cursor=${pageParam}`);
      return response.json();
    },
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    initialPageParam: 0
  });

  return (
    <div>
      {data?.pages.map((page, i) => (
        <div key={i}>
          {page.users.map((user: User) => (
            <div key={user.id}>{user.name}</div>
          ))}
        </div>
      ))}

      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage
          ? 'Loading more...'
          : hasNextPage
          ? 'Load More'
          : 'No more data'}
      </button>
    </div>
  );
}
```

---

## SWR Basics

```typescript
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(r => r.json());

function UserList() {
  const { data, error, isLoading } = useSWR<User[]>('/api/users', fetcher);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error</div>;

  return (
    <ul>
      {data?.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

---

## SWR Mutations

```typescript
import { mutate } from 'swr';

function CreateUser() {
  const [loading, setLoading] = useState(false);

  const handleCreate = async () => {
    setLoading(true);

    try {
      // Optimistic update
      mutate('/api/users', async (users: User[]) => {
        const newUser = { id: 'temp', name: 'New User', email: '' };
        return [...users, newUser];
      }, false); // Don't revalidate yet

      // Actual API call
      const response = await fetch('/api/users', {
        method: 'POST',
        body: JSON.stringify({ name: 'New User' })
      });

      const created = await response.json();

      // Update with real data
      mutate('/api/users');
    } finally {
      setLoading(false);
    }
  };

  return <button onClick={handleCreate}>Create</button>;
}
```

---

## SWR Global Configuration

```typescript
import { SWRConfig } from 'swr';

function App() {
  return (
    <SWRConfig
      value={{
        fetcher: (url) => fetch(url).then(r => r.json()),
        revalidateOnFocus: true,
        revalidateOnReconnect: true,
        dedupingInterval: 2000,
        refreshInterval: 0,
        onError: (error) => {
          console.error('SWR Error:', error);
        }
      }}
    >
      <UserList />
    </SWRConfig>
  );
}
```

---

## React Query Global Configuration

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      gcTime: 10 * 60 * 1000, // 10 minutes (was cacheTime)
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
      retry: 3,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000)
    },
    mutations: {
      onError: (error) => {
        console.error('Mutation error:', error);
      }
    }
  }
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <UserList />
    </QueryClientProvider>
  );
}
```

---

## Real-World: Search with Debouncing

```typescript
function UserSearch() {
  const [search, setSearch] = useState('');
  const debouncedSearch = useDebounce(search, 300);

  const { data, isLoading } = useQuery({
    queryKey: ['users', 'search', debouncedSearch],
    queryFn: () => fetchUsers({ search: debouncedSearch }),
    enabled: debouncedSearch.length > 0
  });

  return (
    <div>
      <input
        value={search}
        onChange={e => setSearch(e.target.value)}
        placeholder="Search users..."
      />

      {isLoading && <div>Searching...</div>}

      {data?.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
    </div>
  );
}

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}
```

---

## Comparison: React Query vs SWR

### **React Query Advantages:**

```typescript
// ✅ More features (infinite queries, mutations, prefetching)
// ✅ Better TypeScript support
// ✅ Devtools included
// ✅ More control over cache
// ✅ Better for complex apps

useQuery({ queryKey: ["users"], queryFn: fetchUsers });
```

### **SWR Advantages:**

```typescript
// ✅ Simpler API
// ✅ Smaller bundle size
// ✅ Great for simple use cases
// ✅ Vercel maintained

useSWR("/api/users", fetcher);
```

---

## Best Practices

✅ **Use query keys consistently**  
✅ **Invalidate queries after mutations**  
✅ **Implement optimistic updates** for better UX  
✅ **Configure stale times** appropriately  
✅ **Handle loading and error states**  
✅ **Use dependent queries** when needed  
❌ **Don't store server state in useState**  
❌ **Don't refetch too aggressively**  
❌ **Don't forget error boundaries**

---

## Key Takeaways

1. **Server state needs specialized tools**
2. **Automatic caching prevents unnecessary requests**
3. **Background refetching keeps data fresh**
4. **Optimistic updates improve UX**
5. **React Query has more features**, SWR is simpler
6. **Query keys are critical** for cache management
7. **Essential for any app with API calls**
