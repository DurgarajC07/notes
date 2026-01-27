# Generics in TypeScript

## Core Concept

Generics allow you to write reusable, type-safe code that works with multiple types while preserving type information. Instead of using `any` (which loses type safety), generics create **type variables** that act as placeholders for actual types.

---

## Basic Generic Function

```typescript
// Without generics - loses type info
function identityAny(value: any): any {
  return value;
}

const result = identityAny("hello"); // Type: any (lost type info)

// With generics - preserves type
function identity<T>(value: T): T {
  return value;
}

const result = identity("hello"); // Type: string ✅
const num = identity(42); // Type: number ✅
```

---

## Generic Arrays

```typescript
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

const first = firstElement([1, 2, 3]); // Type: number
const firstStr = firstElement(["a", "b"]); // Type: string
```

---

## Multiple Type Parameters

```typescript
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const result = pair("hello", 42); // Type: [string, number]

// Real-world: Map function
function map<Input, Output>(
  arr: Input[],
  fn: (item: Input) => Output,
): Output[] {
  return arr.map(fn);
}

const lengths = map(["a", "bb", "ccc"], (s) => s.length);
// Type: number[]
```

---

## Generic Constraints

```typescript
// Constraint: T must have a length property
interface Lengthwise {
  length: number;
}

function logLength<T extends Lengthwise>(item: T): void {
  console.log(item.length);
}

logLength("hello"); // ✅ string has length
logLength([1, 2, 3]); // ✅ array has length
logLength(123); // ❌ Error: number doesn't have length
```

---

## Generic Interfaces

```typescript
interface Box<T> {
  value: T;
}

const stringBox: Box<string> = { value: "hello" };
const numberBox: Box<number> = { value: 42 };

// Generic API Response
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

interface User {
  id: number;
  name: string;
}

const response: ApiResponse<User> = {
  data: { id: 1, name: "Alice" },
  status: 200,
  message: "Success",
};
```

---

## Generic Classes

```typescript
class Queue<T> {
  private items: T[] = [];

  enqueue(item: T): void {
    this.items.push(item);
  }

  dequeue(): T | undefined {
    return this.items.shift();
  }

  peek(): T | undefined {
    return this.items[0];
  }

  get size(): number {
    return this.items.length;
  }
}

const numberQueue = new Queue<number>();
numberQueue.enqueue(1);
numberQueue.enqueue(2);

const stringQueue = new Queue<string>();
stringQueue.enqueue("hello");
```

---

## Default Type Parameters

```typescript
interface Options<T = string> {
  value: T;
  label: string;
}

const option1: Options = { value: "hello", label: "Greeting" }; // T defaults to string
const option2: Options<number> = { value: 42, label: "Answer" };
```

---

## Generic Type Aliases

```typescript
type Result<T, E = Error> =
  | { success: true; value: T }
  | { success: false; error: E };

function divide(a: number, b: number): Result<number> {
  if (b === 0) {
    return { success: false, error: new Error("Division by zero") };
  }
  return { success: true, value: a / b };
}
```

---

## keyof and Generic Constraints

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 30 };
const name = getProperty(user, "name"); // Type: string
const age = getProperty(user, "age"); // Type: number
// getProperty(user, 'email'); // ❌ Error: 'email' not in user
```

---

## Real-World Examples

### **Async Data Fetcher**

```typescript
async function fetchData<T>(url: string): Promise<T> {
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }
  return (await response.json()) as T;
}

interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

const todo = await fetchData<Todo>("https://api.example.com/todos/1");
console.log(todo.title); // Type-safe access
```

### **State Management**

```typescript
interface Action<T extends string, P = void> {
  type: T;
  payload: P;
}

type IncrementAction = Action<"INCREMENT">;
type SetCountAction = Action<"SET_COUNT", number>;
type SetUserAction = Action<"SET_USER", { id: number; name: string }>;

function createAction<T extends string, P = void>(
  type: T,
  payload: P,
): Action<T, P> {
  return { type, payload };
}

const increment = createAction("INCREMENT", undefined);
const setCount = createAction("SET_COUNT", 42);
```

---

## Advanced Patterns

### **Generic Factory**

```typescript
class Animal {
  constructor(public name: string) {}
}

class Dog extends Animal {
  bark() {
    console.log("Woof!");
  }
}

class Cat extends Animal {
  meow() {
    console.log("Meow!");
  }
}

function createAnimal<T extends Animal>(
  AnimalClass: new (name: string) => T,
  name: string,
): T {
  return new AnimalClass(name);
}

const dog = createAnimal(Dog, "Buddy"); // Type: Dog
dog.bark(); // ✅ Type-safe

const cat = createAnimal(Cat, "Whiskers"); // Type: Cat
cat.meow(); // ✅ Type-safe
```

---

## Best Practices

✅ **Use descriptive type parameter names** for complex generics (T, U for simple, TData, TError for clarity)  
✅ **Add constraints** when generic needs specific properties  
✅ **Provide default types** when sensible defaults exist  
✅ **Avoid excessive type parameters** - more than 3-4 is a code smell  
✅ **Use inference when possible** - don't force explicit type arguments  
❌ **Don't use generics if not needed** - simple types are clearer  
❌ **Don't create overly abstract generics** - balance reusability with readability

---

## Key Takeaways

1. **Generics preserve type information** while enabling reusability
2. **Type constraints** ensure generics have required properties
3. **keyof creates type-safe property access**
4. **Default type parameters** improve ergonomics
5. **Inference reduces boilerplate** - explicit when needed for clarity
