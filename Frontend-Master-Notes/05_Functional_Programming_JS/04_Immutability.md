# Immutability

> **Immutable data structures, patterns, and libraries like Immer**

---

## 🎯 Overview

Immutability means data cannot be changed after creation. Instead of modifying existing data, you create new data with the desired changes. This approach prevents bugs, makes code predictable, and enables powerful features like time-travel debugging.

---

## 📚 Core Concepts

### **Concept 1: Immutable vs Mutable**

```javascript
// ❌ Mutable - modifies original
const user = { name: "Alice", age: 25 };
user.age = 26; // Original object mutated
console.log(user); // { name: 'Alice', age: 26 }

// ✅ Immutable - creates new object
const user = { name: "Alice", age: 25 };
const updatedUser = { ...user, age: 26 };
console.log(user); // { name: 'Alice', age: 25 } - unchanged
console.log(updatedUser); // { name: 'Alice', age: 26 }

// Arrays
// ❌ Mutable
const arr = [1, 2, 3];
arr.push(4); // Mutates original

// ✅ Immutable
const arr = [1, 2, 3];
const newArr = [...arr, 4]; // Creates new array
```

### **Concept 2: Immutable Array Operations**

```javascript
const numbers = [1, 2, 3, 4, 5];

// Add element
const withAdded = [...numbers, 6]; // [1, 2, 3, 4, 5, 6]
const withPrepended = [0, ...numbers]; // [0, 1, 2, 3, 4, 5]

// Remove element
const withoutFirst = numbers.slice(1); // [2, 3, 4, 5]
const withoutLast = numbers.slice(0, -1); // [1, 2, 3, 4]
const withoutIndex = [...numbers.slice(0, 2), ...numbers.slice(3)]; // [1, 2, 4, 5]

// Update element
const updated = numbers.map((n, i) => (i === 2 ? 10 : n));
// [1, 2, 10, 4, 5]

// Filter
const evens = numbers.filter((n) => n % 2 === 0); // [2, 4]

// Reverse
const reversed = [...numbers].reverse(); // [5, 4, 3, 2, 1]
// Or: numbers.slice().reverse();
```

### **Concept 3: Immutable Object Operations**

```javascript
const user = {
  name: "Alice",
  age: 25,
  address: {
    city: "NYC",
    zip: "10001",
  },
};

// Add/Update property
const updated = { ...user, age: 26 };

// Remove property
const { age, ...withoutAge } = user;

// Nested update (shallow spread doesn't work)
// ❌ Wrong - mutates nested object
const wrong = { ...user };
wrong.address.city = "LA"; // Mutates original user.address!

// ✅ Correct - deep immutable update
const correct = {
  ...user,
  address: {
    ...user.address,
    city: "LA",
  },
};

// Multiple nested levels
const deepUser = {
  profile: {
    personal: {
      name: "Alice",
    },
  },
};

const updated = {
  ...deepUser,
  profile: {
    ...deepUser.profile,
    personal: {
      ...deepUser.profile.personal,
      name: "Bob",
    },
  },
};
```

---

## 🔥 Practical Examples

### **Example 1: Immutable State Updates (Redux Pattern)**

```javascript
// Reducer with immutable updates
function todoReducer(state = { todos: [] }, action) {
  switch (action.type) {
    case "ADD_TODO":
      return {
        ...state,
        todos: [...state.todos, action.todo],
      };

    case "REMOVE_TODO":
      return {
        ...state,
        todos: state.todos.filter((todo) => todo.id !== action.id),
      };

    case "UPDATE_TODO":
      return {
        ...state,
        todos: state.todos.map((todo) =>
          todo.id === action.id ? { ...todo, ...action.updates } : todo,
        ),
      };

    case "TOGGLE_TODO":
      return {
        ...state,
        todos: state.todos.map((todo) =>
          todo.id === action.id
            ? { ...todo, completed: !todo.completed }
            : todo,
        ),
      };

    default:
      return state;
  }
}
```

### **Example 2: Using Immer**

Immer lets you write "mutable" code that produces immutable results:

```javascript
import produce from "immer";

// Without Immer - verbose
function updateUser(user, updates) {
  return {
    ...user,
    profile: {
      ...user.profile,
      settings: {
        ...user.profile.settings,
        ...updates,
      },
    },
  };
}

// With Immer - simple
function updateUser(user, updates) {
  return produce(user, (draft) => {
    Object.assign(draft.profile.settings, updates);
  });
}

// Redux reducer with Immer
const todoReducer = produce((draft, action) => {
  switch (action.type) {
    case "ADD_TODO":
      draft.todos.push(action.todo);
      break;

    case "REMOVE_TODO":
      const index = draft.todos.findIndex((t) => t.id === action.id);
      draft.todos.splice(index, 1);
      break;

    case "UPDATE_TODO":
      const todo = draft.todos.find((t) => t.id === action.id);
      Object.assign(todo, action.updates);
      break;
  }
});
```

### **Example 3: Structural Sharing**

Immutable updates share unchanged parts:

```javascript
const state = {
  users: [
    { id: 1, name: "Alice" },
    { id: 2, name: "Bob" },
  ],
  posts: [{ id: 1, title: "Post 1" }],
};

// Update only users
const newState = {
  ...state,
  users: [...state.users, { id: 3, name: "Carol" }],
};

// Structural sharing
console.log(newState.users === state.users); // false (changed)
console.log(newState.posts === state.posts); // true (unchanged - shared)

// Benefits:
// 1. Memory efficient (shares unchanged data)
// 2. Fast equality checks (reference comparison)
// 3. Enables React optimization (React.memo, useMemo)
```

### **Example 4: Immutable Helpers**

```javascript
// Update nested property
function updateIn(obj, path, value) {
  const [head, ...tail] = path;

  if (tail.length === 0) {
    return { ...obj, [head]: value };
  }

  return {
    ...obj,
    [head]: updateIn(obj[head], tail, value),
  };
}

const user = {
  profile: {
    settings: {
      theme: "dark",
    },
  },
};

const updated = updateIn(user, ["profile", "settings", "theme"], "light");

// Get nested property
function getIn(obj, path) {
  return path.reduce((current, key) => current?.[key], obj);
}

getIn(user, ["profile", "settings", "theme"]); // 'dark'
```

### **Example 5: Immutable Class**

```javascript
class ImmutableUser {
  constructor(data) {
    Object.assign(this, data);
    Object.freeze(this);
  }

  with(updates) {
    return new ImmutableUser({
      ...this,
      ...updates,
    });
  }

  withNested(path, value) {
    const keys = path.split(".");
    const [head, ...tail] = keys;

    if (tail.length === 0) {
      return this.with({ [head]: value });
    }

    const nested = this[head];
    const updated =
      typeof nested.withNested === "function"
        ? nested.withNested(tail.join("."), value)
        : updateIn(nested, tail, value);

    return this.with({ [head]: updated });
  }
}

// Usage
const user = new ImmutableUser({
  name: "Alice",
  age: 25,
});

// user.age = 26; // TypeError: Cannot assign to read only property

const updated = user.with({ age: 26 });
console.log(user.age); // 25
console.log(updated.age); // 26
```

---

## 🏗️ Best Practices

1. **Use spread operator for shallow updates**

   ```javascript
   const updated = { ...obj, key: newValue };
   const newArr = [...arr, newItem];
   ```

2. **Avoid direct mutations**

   ```javascript
   // ❌ Bad
   state.value = newValue;
   arr.push(item);

   // ✅ Good
   state = { ...state, value: newValue };
   arr = [...arr, item];
   ```

3. **Use Immer for complex nested updates**

   ```javascript
   produce(state, (draft) => {
     draft.deeply.nested.value = newValue;
   });
   ```

4. **Leverage structural sharing**

   ```javascript
   // Only update what changed
   const newState = {
     ...state,
     users: updatedUsers, // Other properties are shared
   };
   ```

5. **Freeze objects in development**
   ```javascript
   if (process.env.NODE_ENV === "development") {
     Object.freeze(state);
   }
   ```

---

## 🧪 Interview Questions

### **Q1: What is immutability and why is it important?**

**Answer:** Immutability means data cannot be changed after creation. Benefits:

1. **Predictability** - No unexpected mutations
2. **Debugging** - Easy to track changes
3. **Performance** - Fast equality checks (reference comparison)
4. **Time-travel** - Keep history of states
5. **Concurrency** - Safe to share across threads

### **Q2: How do you perform immutable updates on nested objects?**

**Answer:** Use spread operator at each level:

```javascript
const updated = {
  ...obj,
  nested: {
    ...obj.nested,
    value: newValue,
  },
};
```

Or use Immer for complex updates:

```javascript
produce(obj, (draft) => {
  draft.nested.value = newValue;
});
```

### **Q3: What is structural sharing?**

**Answer:** Structural sharing means immutable updates only copy changed parts, sharing unchanged parts from the original. This is memory-efficient and enables fast equality checks.

```javascript
const newState = { ...state, users: newUsers };
// newState.posts === state.posts (shared)
```

---

## 📚 Key Takeaways

- Immutability creates new data instead of modifying existing data
- Use spread operator for shallow updates, nested spreads for deep updates
- Immer simplifies complex nested immutable updates
- Structural sharing makes immutability memory-efficient
- Immutability enables predictable state, easy debugging, and performance optimizations
- Essential for Redux, React state management, and functional programming
- Freeze objects in development to catch mutations early

---

**Master immutability for production-ready code!**
