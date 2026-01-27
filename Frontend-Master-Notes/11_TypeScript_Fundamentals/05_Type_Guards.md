# Type Guards in TypeScript

## Core Concept

Type guards are **expressions that perform runtime checks** to narrow types within conditional blocks. They help TypeScript understand type refinement.

---

## Built-in Type Guards

### **typeof**

```typescript
function printValue(value: string | number) {
  if (typeof value === "string") {
    console.log(value.toUpperCase()); // Type: string
  } else {
    console.log(value.toFixed(2)); // Type: number
  }
}
```

### **instanceof**

```typescript
class Dog {
  bark() {
    console.log("Woof!");
  }
}

class Cat {
  meow() {
    console.log("Meow!");
  }
}

function makeSound(animal: Dog | Cat) {
  if (animal instanceof Dog) {
    animal.bark(); // Type: Dog
  } else {
    animal.meow(); // Type: Cat
  }
}
```

### **in Operator**

```typescript
interface Fish {
  swim: () => void;
}

interface Bird {
  fly: () => void;
}

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim(); // Type: Fish
  } else {
    animal.fly(); // Type: Bird
  }
}
```

---

## Custom Type Guards

### **User-Defined Type Guard**

```typescript
interface User {
  name: string;
  email: string;
}

interface Admin extends User {
  role: "admin";
  permissions: string[];
}

function isAdmin(user: User | Admin): user is Admin {
  return (user as Admin).role === "admin";
}

function greet(user: User | Admin) {
  if (isAdmin(user)) {
    console.log(`Admin: ${user.permissions}`); // Type: Admin
  } else {
    console.log(`User: ${user.name}`); // Type: User
  }
}
```

---

## Discriminated Unions

```typescript
interface Circle {
  kind: "circle";
  radius: number;
}

interface Square {
  kind: "square";
  sideLength: number;
}

type Shape = Circle | Square;

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2; // Type: Circle
    case "square":
      return shape.sideLength ** 2; // Type: Square
  }
}
```

---

## Truthiness Narrowing

```typescript
function printName(name: string | null | undefined) {
  if (name) {
    console.log(name.toUpperCase()); // Type: string
  }
}
```

---

## Equality Narrowing

```typescript
function example(x: string | number, y: string | boolean) {
  if (x === y) {
    // Both must be strings
    x.toUpperCase();
    y.toLowerCase();
  }
}
```

---

## Array.isArray()

```typescript
function process(value: string | string[]) {
  if (Array.isArray(value)) {
    value.forEach((s) => console.log(s)); // Type: string[]
  } else {
    console.log(value.toUpperCase()); // Type: string
  }
}
```

---

## Best Practices

✅ **Use discriminated unions** for type-safe state machines  
✅ **Create custom type guards** for complex type checks  
✅ **Use `in` operator** for checking property existence  
✅ **Leverage truthiness** for null/undefined checks  
❌ **Don't overuse type assertions** - use guards instead

---

## Key Takeaways

1. **Type guards narrow types** within conditional blocks
2. **`typeof`, `instanceof`, `in`** are built-in guards
3. **Custom type guards** use `is` predicate
4. **Discriminated unions** provide exhaustive checking
5. **Truthiness narrowing** handles null/undefined
