# Interfaces vs Type Aliases in TypeScript

## Core Difference

Both `interface` and `type` can describe object shapes, but they have different capabilities and use cases.

---

## Basic Syntax

### **Interface**

```typescript
interface User {
  name: string;
  age: number;
}

const user: User = {
  name: "Alice",
  age: 30,
};
```

### **Type Alias**

```typescript
type User = {
  name: string;
  age: number;
};

const user: User = {
  name: "Alice",
  age: 30,
};
```

---

## When to Use Each

### **Use Interface For:**

✅ **Object shapes** that might be extended  
✅ **Public APIs** (better error messages)  
✅ **Declaration merging** (augmenting existing types)  
✅ **Class implementations**

### **Use Type Alias For:**

✅ **Union types** (`string | number`)  
✅ **Intersection types** (combining types)  
✅ **Tuple types** (`[string, number]`)  
✅ **Mapped types** (`{ [K in keyof T]: ... }`)  
✅ **Primitive aliases** (`type ID = string`)  
✅ **Function types** (`type Fn = (x: number) => number`)

---

## Extending

### **Interface Extension**

```typescript
interface Animal {
  name: string;
}

interface Dog extends Animal {
  breed: string;
}

const dog: Dog = {
  name: "Buddy",
  breed: "Labrador",
};
```

### **Type Intersection**

```typescript
type Animal = {
  name: string;
};

type Dog = Animal & {
  breed: string;
};

const dog: Dog = {
  name: "Buddy",
  breed: "Labrador",
};
```

---

## Declaration Merging (Interface Only)

```typescript
interface User {
  name: string;
}

interface User {
  age: number;
}

// Merges into one interface
const user: User = {
  name: "Alice",
  age: 30,
};
```

**Not possible with type:**

```typescript
type User = { name: string };
type User = { age: number }; // ❌ Error: Duplicate identifier
```

---

## Union Types (Type Only)

```typescript
type Status = "pending" | "approved" | "rejected";

type Result = SuccessResult | ErrorResult;

type ID = string | number;
```

**Interface cannot create unions directly:**

```typescript
interface Status { ... }  // Can't do union with interface
```

---

## Computed Properties

### **Type Alias** (flexible)

```typescript
type Keys = "name" | "age";

type Person = {
  [K in Keys]: string;
};
// { name: string; age: string; }
```

### **Interface** (limited)

```typescript
interface Person {
  [key: string]: string; // Index signature only
}
```

---

## Real-World Examples

### **API Response (Type - Union)**

```typescript
type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: string };

async function fetchUser(id: number): Promise<ApiResponse<User>> {
  // ...
}
```

### **Component Props (Interface - Extensible)**

```typescript
interface BaseButtonProps {
  onClick: () => void;
  disabled?: boolean;
}

interface PrimaryButtonProps extends BaseButtonProps {
  variant: "primary";
  size: "small" | "large";
}
```

---

## Performance & Compilation

**No runtime difference** - both compile to JavaScript objects. Performance is identical.

---

## Best Practices

✅ **Default to `type`** for flexibility (union, intersection)  
✅ **Use `interface`** for public APIs and extensible contracts  
✅ **Be consistent** within a project  
✅ **Use `interface`** for React components (convention)  
❌ **Don't mix unnecessarily** - pick one style per domain

---

## Quick Reference

| Feature             | Interface | Type       |
| ------------------- | --------- | ---------- |
| Object shapes       | ✅        | ✅         |
| Extends             | ✅        | ✅ (via &) |
| Declaration merging | ✅        | ❌         |
| Union types         | ❌        | ✅         |
| Intersection        | ❌        | ✅         |
| Tuple types         | ❌        | ✅         |
| Mapped types        | Limited   | ✅         |
| Computed properties | Limited   | ✅         |
| Implements (class)  | ✅        | ✅         |

---

## Key Takeaways

1. **`interface` for object shapes**, especially public APIs
2. **`type` for unions, intersections, and advanced types**
3. **Declaration merging** is interface-only
4. **No performance difference** - only ergonomics
5. **Consistency matters** more than the choice itself
