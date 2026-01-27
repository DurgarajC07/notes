# TypeScript Type System

## Core Concept

TypeScript uses a **structural type system** (also called "duck typing"), where type compatibility is determined by the structure of types rather than explicit declarations. This differs from nominal typing where two types are compatible only if they have the same name.

---

## Structural vs Nominal Typing

### **Structural Typing** (TypeScript)

```typescript
interface Point2D {
  x: number;
  y: number;
}

interface Vector2D {
  x: number;
  y: number;
}

function distance(point: Point2D): number {
  return Math.sqrt(point.x ** 2 + point.y ** 2);
}

const vector: Vector2D = { x: 3, y: 4 };
distance(vector); // ✅ Works! Same structure
```

### **Nominal Typing** (Java, C#)

```java
// Java example - would NOT work
class Point2D {
  int x, y;
}

class Vector2D {
  int x, y;
}

void distance(Point2D point) { ... }
distance(new Vector2D()); // ❌ Type error!
```

---

## Type Inference

TypeScript can automatically infer types based on values:

```typescript
// Explicit type
let age: number = 25;

// Inferred type (same as above)
let age = 25; // Type: number

// Complex inference
const user = {
  name: "Alice",
  age: 30,
  hobbies: ["reading", "coding"],
};
// Type inferred as:
// {
//   name: string;
//   age: number;
//   hobbies: string[];
// }

// Function return type inference
function add(a: number, b: number) {
  return a + b; // Return type inferred as number
}

// Best practice: Explicit return types for public APIs
function multiply(a: number, b: number): number {
  return a * b;
}
```

---

## Type Annotations

```typescript
// Primitive types
let isDone: boolean = false;
let count: number = 10;
let name: string = "Alice";
let nothing: null = null;
let notDefined: undefined = undefined;

// Arrays
let numbers: number[] = [1, 2, 3];
let names: Array<string> = ["Alice", "Bob"];

// Objects
let user: { name: string; age: number } = {
  name: "Alice",
  age: 30,
};

// Functions
function greet(name: string): string {
  return `Hello, ${name}`;
}

const add = (a: number, b: number): number => a + b;

// Function types
type MathOperation = (a: number, b: number) => number;
const subtract: MathOperation = (a, b) => a - b;
```

---

## Type Compatibility

### **Assignability Rules**

```typescript
interface Named {
  name: string;
}

interface Person {
  name: string;
  age: number;
}

let person: Person = { name: "Alice", age: 30 };
let named: Named = person; // ✅ OK - Person has all properties of Named

// Reverse doesn't work
let person2: Person = named; // ❌ Error: 'age' is missing
```

### **Excess Property Checks**

```typescript
interface Config {
  color: string;
  width: number;
}

// Direct assignment - strict checking
const config: Config = {
  color: "red",
  width: 100,
  height: 200, // ❌ Error: 'height' does not exist
};

// Indirect assignment - no excess property check
const temp = {
  color: "red",
  width: 100,
  height: 200,
};
const config2: Config = temp; // ✅ OK - structural typing
```

---

## Type Assertions

```typescript
// as syntax (preferred)
const myCanvas = document.getElementById("main") as HTMLCanvasElement;

// Angle bracket syntax (not usable in JSX)
const myCanvas = <HTMLCanvasElement>document.getElementById("main");

// Non-null assertion
function processValue(value: string | null) {
  console.log(value!.toUpperCase()); // ! asserts non-null
}

// Double assertion (escape hatch - avoid)
const value = "hello" as unknown as number; // Bad practice!
```

---

## Literal Types

```typescript
// String literal
let direction: "north" | "south" | "east" | "west";
direction = "north"; // ✅
direction = "up"; // ❌ Error

// Numeric literal
let diceRoll: 1 | 2 | 3 | 4 | 5 | 6;

// Boolean literal
let yes: true = true;

// Object literal
const config = {
  method: "GET" as const, // Literal type 'GET', not string
  url: "/api/users",
};

// const assertion
const directions = ["north", "south", "east", "west"] as const;
// Type: readonly ["north", "south", "east", "west"]
```

---

## Union and Intersection Types

### **Union Types** (OR)

```typescript
function printId(id: number | string) {
  if (typeof id === "string") {
    console.log(id.toUpperCase());
  } else {
    console.log(id);
  }
}

// Discriminated unions
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; sideLength: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
  }
}
```

### **Intersection Types** (AND)

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
  radius: 10,
};
```

---

## Type Narrowing

### **typeof Guards**

```typescript
function padLeft(value: string, padding: string | number) {
  if (typeof padding === "number") {
    return " ".repeat(padding) + value;
  }
  return padding + value;
}
```

### **Truthiness Narrowing**

```typescript
function printName(name: string | null | undefined) {
  if (name) {
    console.log(name.toUpperCase());
  } else {
    console.log("No name provided");
  }
}
```

### **Equality Narrowing**

```typescript
function example(x: string | number, y: string | boolean) {
  if (x === y) {
    // x and y must both be strings
    x.toUpperCase();
    y.toLowerCase();
  }
}
```

### **in Operator**

```typescript
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim();
  } else {
    animal.fly();
  }
}
```

### **instanceof**

```typescript
function logValue(x: Date | string) {
  if (x instanceof Date) {
    console.log(x.toUTCString());
  } else {
    console.log(x.toUpperCase());
  }
}
```

---

## Type Predicates

```typescript
function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined;
}

function move(pet: Fish | Bird) {
  if (isFish(pet)) {
    pet.swim(); // TypeScript knows pet is Fish
  } else {
    pet.fly(); // TypeScript knows pet is Bird
  }
}
```

---

## never Type

```typescript
// Function that never returns
function throwError(message: string): never {
  throw new Error(message);
}

// Exhaustiveness checking
type Shape = Circle | Square;

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
    default:
      const _exhaustive: never = shape; // Error if not exhaustive
      return _exhaustive;
  }
}
```

---

## unknown vs any

```typescript
// any - disables type checking
let value: any;
value.foo.bar; // No error - dangerous!

// unknown - type-safe any
let value: unknown;
value.foo.bar; // ❌ Error

// Must narrow before use
if (typeof value === "string") {
  console.log(value.toUpperCase()); // ✅ OK
}
```

---

## Best Practices

✅ **Enable strict mode** in tsconfig.json  
✅ **Use type inference** when obvious  
✅ **Explicit types for public APIs** (function signatures, exports)  
✅ **Prefer interfaces for objects**, types for unions/intersections  
✅ **Use discriminated unions** for complex types  
✅ **Avoid `any`** - use `unknown` if type is truly unknown  
✅ **Use type predicates** for custom type guards  
❌ **Don't over-annotate** - let inference work  
❌ **Don't use type assertions** as a workaround for type errors

---

## Key Takeaways

1. **TypeScript uses structural typing** - shape matters, not name
2. **Type inference reduces boilerplate** - explicit when needed
3. **Union types express alternatives** - use discriminated unions for safety
4. **Type narrowing proves types** - typeof, instanceof, custom predicates
5. **unknown is safer than any** - forces type checking
6. **never ensures exhaustiveness** - catches missing cases
7. **const assertions preserve literal types** - useful for configs
8. **Excess property checks** only apply to fresh object literals
