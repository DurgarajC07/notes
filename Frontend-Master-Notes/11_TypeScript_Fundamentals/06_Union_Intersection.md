# Union and Intersection Types

## Union Types (OR)

A union type describes a value that can be **one of several types**.

```typescript
function printId(id: number | string) {
  console.log("ID:", id);
}

printId(101); // ✅
printId("202"); // ✅
printId(true); // ❌ Error
```

---

## Type Narrowing with Unions

```typescript
function printId(id: number | string) {
  // Must narrow type before using specific methods
  if (typeof id === "string") {
    console.log(id.toUpperCase()); // Type: string
  } else {
    console.log(id.toFixed(2)); // Type: number
  }
}
```

---

## Discriminated Unions

Tagged unions for type-safe state management.

```typescript
interface Loading {
  status: "loading";
}

interface Success<T> {
  status: "success";
  data: T;
}

interface Error {
  status: "error";
  error: string;
}

type ApiState<T> = Loading | Success<T> | Error;

function handleState<T>(state: ApiState<T>) {
  switch (state.status) {
    case "loading":
      return "Loading...";
    case "success":
      return state.data; // TypeScript knows data exists
    case "error":
      return state.error; // TypeScript knows error exists
  }
}
```

---

## Intersection Types (AND)

An intersection type combines multiple types into one.

```typescript
interface Colorful {
  color: string;
}

interface Circle {
  radius: number;
}

type ColorfulCircle = Colorful & Circle;

const cc: ColorfulCircle = {
  color: "red",
  radius: 42,
}; // Must have both properties
```

---

## Real-World Union Example

```typescript
type SuccessResponse = {
  success: true;
  data: any;
};

type ErrorResponse = {
  success: false;
  error: string;
};

type ApiResponse = SuccessResponse | ErrorResponse;

function handleResponse(response: ApiResponse) {
  if (response.success) {
    console.log(response.data); // Type narrowed
  } else {
    console.error(response.error); // Type narrowed
  }
}
```

---

## Best Practices

✅ **Use discriminated unions** for state machines  
✅ **Narrow types** with typeof, instanceof, in  
✅ **Use intersections** to combine capabilities  
✅ **Prefer unions over enums** for string constants  
❌ **Don't create deeply nested unions** - hard to maintain

---

## Key Takeaways

1. **Union = one of** several types (OR)
2. **Intersection = all of** several types (AND)
3. **Discriminated unions** enable exhaustive type checking
4. **Type narrowing** is essential with unions
5. **Unions model alternatives**, intersections model combinations
