# Component Lifecycle in React

## Modern React (Hooks Era)

React hooks have replaced class lifecycle methods with a more composable model.

---

## Component Phases

### **1. Mount** - Component first appears

### **2. Update** - Props or state change

### **3. Unmount** - Component is removed

---

## useEffect Lifecycle Mapping

### **componentDidMount**

```typescript
useEffect(() => {
  // Runs once after first render
  console.log("Component mounted");
}, []); // Empty dependency array
```

### **componentDidUpdate**

```typescript
useEffect(() => {
  // Runs on every render after mount
  console.log("Component updated");
}); // No dependency array

// Or with specific dependencies
useEffect(() => {
  console.log("Count changed");
}, [count]); // Runs when count changes
```

### **componentWillUnmount**

```typescript
useEffect(() => {
  const timer = setInterval(() => console.log("tick"), 1000);

  return () => {
    clearInterval(timer); // Cleanup
  };
}, []);
```

---

## Complete Lifecycle Example

```typescript
function UserProfile({ userId }: { userId: number }) {
  const [user, setUser] = useState<User | null>(null);

  // Mount: Fetch user data
  useEffect(() => {
    let cancelled = false;

    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) {
          setUser(data);
        }
      });

    // Cleanup on unmount
    return () => {
      cancelled = true;
    };
  }, [userId]); // Re-run when userId changes

  if (!user) return <div>Loading...</div>;
  return <div>{user.name}</div>;
}
```

---

## Class Component Equivalents

```typescript
class Counter extends React.Component {
  componentDidMount() {
    // Setup: subscriptions, timers, API calls
  }

  componentDidUpdate(prevProps, prevState) {
    // Respond to prop/state changes
    if (prevProps.id !== this.props.id) {
      this.fetchData(this.props.id);
    }
  }

  componentWillUnmount() {
    // Cleanup: clear timers, cancel requests
  }

  render() {
    return <div>{this.state.count}</div>;
  }
}
```

---

## Common Patterns

### **Fetch on Mount**

```typescript
useEffect(() => {
  async function fetchData() {
    const response = await fetch("/api/data");
    const data = await response.json();
    setData(data);
  }
  fetchData();
}, []);
```

### **Subscribe/Unsubscribe**

```typescript
useEffect(() => {
  const subscription = eventEmitter.subscribe((event) => {
    console.log(event);
  });

  return () => subscription.unsubscribe();
}, []);
```

### **Timer/Interval**

```typescript
useEffect(() => {
  const timer = setTimeout(() => {
    console.log("Delayed action");
  }, 1000);

  return () => clearTimeout(timer);
}, []);
```

---

## Best Practices

✅ **Always cleanup** side effects in return function  
✅ **List all dependencies** in dependency array  
✅ **Use separate effects** for unrelated logic  
✅ **Handle race conditions** with cleanup flags  
❌ **Don't omit dependencies** - use ESLint rule  
❌ **Don't put async functions directly** in useEffect

---

## Key Takeaways

1. **useEffect handles all lifecycle** phases
2. **Empty deps []** = mount only
3. **Return function** = cleanup/unmount
4. **Deps array** = when to re-run
5. **Multiple effects** for separation of concerns
