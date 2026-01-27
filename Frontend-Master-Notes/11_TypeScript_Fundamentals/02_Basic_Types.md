# TypeScript Basic Types

## Primitives

```typescript
// Boolean
let isDone: boolean = false;

// Number (all numbers are floating point)
let decimal: number = 6;
let hex: number = 0xf00d;
let binary: number = 0b1010;
let octal: number = 0o744;
let big: bigint = 100n;

// String
let color: string = "blue";
let fullName: string = `Bob Bobbington`;
let sentence: string = `Hello, my name is ${fullName}`;

// Null and Undefined
let u: undefined = undefined;
let n: null = null;

// In strict mode, null and undefined are only assignable to themselves and any
let num: number = null; // Error in strict mode
```

---

## Arrays

```typescript
// Array of numbers
let list: number[] = [1, 2, 3];
let list2: Array<number> = [1, 2, 3]; // Generic syntax

// Array of strings
let names: string[] = ["Alice", "Bob"];

// Mixed types (not recommended - use union)
let mixed: (string | number)[] = ["Alice", 42];

// Readonly array
let readonlyList: ReadonlyArray<number> = [1, 2, 3];
readonlyList[0] = 5; // ❌ Error
```

---

## Tuples

Fixed-length arrays with specific types at each position.

```typescript
// Declare a tuple type
let x: [string, number];
x = ["hello", 10]; // ✅ OK
x = [10, "hello"]; // ❌ Error

// Accessing elements
console.log(x[0].substring(1)); // OK
console.log(x[1].substring(1)); // Error: number doesn't have substring

// Optional tuple elements
type Point = [number, number, number?];
const point2D: Point = [10, 20];
const point3D: Point = [10, 20, 30];

// Rest elements in tuples
type StringNumberBooleans = [string, number, ...boolean[]];
const data: StringNumberBooleans = ["hello", 1, true, false, true];
```

---

## Enums

```typescript
// Numeric enum (default starts at 0)
enum Direction {
  Up, // 0
  Down, // 1
  Left, // 2
  Right, // 3
}

let dir: Direction = Direction.Up;

// Custom numeric values
enum Status {
  Pending = 1,
  Approved = 2,
  Rejected = 3,
}

// String enum (recommended for debugging)
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}

// Const enum (inlined at compile time)
const enum LogLevel {
  Error,
  Warn,
  Info,
}

let level = LogLevel.Error; // Compiles to: let level = 0;
```

**Better alternative:** Use union of string literals

```typescript
type Color = "RED" | "GREEN" | "BLUE";
const color: Color = "RED";
```

---

## Any

```typescript
let notSure: any = 4;
notSure = "maybe a string instead";
notSure = false; // All OK - no type checking

// ⚠️ Avoid any - loses type safety!
```

---

## Unknown

Type-safe version of `any`.

```typescript
let value: unknown;

value = "hello";
value = 42;
value = true; // All OK

// But can't use without type checking
value.toUpperCase(); // ❌ Error

// Must narrow type first
if (typeof value === "string") {
  console.log(value.toUpperCase()); // ✅ OK
}
```

---

## Void

```typescript
// Function with no return value
function warnUser(): void {
  console.log("Warning!");
}

// Void variables can only be null or undefined
let unusable: void = undefined;
```

---

## Never

Type for values that never occur.

```typescript
// Function that never returns
function error(message: string): never {
  throw new Error(message);
}

// Function with infinite loop
function infiniteLoop(): never {
  while (true) {}
}

// Type narrowing to never
function processValue(value: string | number) {
  if (typeof value === "string") {
    return value.toUpperCase();
  } else if (typeof value === "number") {
    return value.toFixed(2);
  } else {
    // value is never here
    const exhaustive: never = value;
    return exhaustive;
  }
}
```

---

## Object

```typescript
// Non-primitive type
declare function create(o: object | null): void;

create({ prop: 0 }); // OK
create(null); // OK
create(42); // ❌ Error
create("string"); // ❌ Error
```

---

## Type Assertions

```typescript
// as syntax (preferred)
let someValue: unknown = "this is a string";
let strLength: number = (someValue as string).length;

// Angle-bracket syntax (not usable in JSX)
let strLength2: number = (<string>someValue).length;

// Non-null assertion
function getValue(): string | null {
  return "hello";
}
let value = getValue()!; // Asserts non-null
```

---

## Literal Types

```typescript
// String literal
let alignment: "left" | "right" | "center";

// Numeric literal
let diceRoll: 1 | 2 | 3 | 4 | 5 | 6;

// Boolean literal
let yes: true = true;

// Object literal with const assertion
const config = {
  endpoint: "/api",
  method: "GET",
} as const;
// Type: { readonly endpoint: "/api"; readonly method: "GET"; }
```

---

## Best Practices

✅ **Use strict mode** (`"strict": true` in tsconfig.json)  
✅ **Prefer unknown over any** for truly unknown types  
✅ **Use const assertions** for immutable values  
✅ **Avoid enums** - use union of string literals instead  
✅ **Use tuples** for fixed-length, heterogeneous arrays  
❌ **Don't use any** - breaks type safety  
❌ **Don't over-annotate** - let inference work

---

## Key Takeaways

1. **TypeScript has all JavaScript types** plus additional ones
2. **unknown is safer than any** - requires type checking
3. **never represents impossible** values (exhaustiveness)
4. **Tuples are typed arrays** with fixed length
5. **Literal types** constrain to specific values
6. **Enums are rarely needed** - use union types instead
