# Type Coercion & Type Conversion in JavaScript

> **Understanding how JavaScript converts values between different types**

---

## 🎯 What is Type Coercion?

**Type Coercion** is the automatic or implicit conversion of values from one data type to another. JavaScript is a dynamically typed language and will attempt to convert types when operations are performed on incompatible types.

**Type Conversion** (or Type Casting) is the explicit conversion of values using functions like `String()`, `Number()`, or `Boolean()`.

---

## 📚 Types of Conversion

### **1. Implicit Coercion (Automatic)**

JavaScript automatically converts types when needed:

```javascript
console.log(5 + "5"); // "55" - number to string
console.log("5" - 2); // 3 - string to number
console.log(true + 1); // 2 - boolean to number
console.log(false + "5"); // "false5" - boolean to string
```

### **2. Explicit Conversion (Manual)**

You manually convert types using built-in functions:

```javascript
String(123); // "123"
Number("456"); // 456
Boolean(0); // false
parseInt("42px"); // 42
parseFloat("3.14"); // 3.14
```

---

## 🔢 String Coercion

### **To String Conversion**

**Implicit:**

```javascript
"Value: " + 42; // "Value: 42"
"Value: " + true; // "Value: true"
"Value: " + null; // "Value: null"
"Value: " + undefined; // "Value: undefined"
"Value: " + {}; // "Value: [object Object]"
"Value: " + [1, 2, 3]; // "Value: 1,2,3"
```

**Explicit:**

```javascript
String(42); // "42"
String(true); // "true"
String(null); // "null"
String(undefined); // "undefined"
String([1, 2, 3]); // "1,2,3"
String({ a: 1 }); // "[object Object]"

// Using toString()
(42).toString(); // "42"
[1, 2, 3].toString(); // "1,2,3"
```

**Template Literals:**

```javascript
const num = 42;
`Value: ${num}`; // "Value: 42"
```

---

## 🔢 Number Coercion

### **To Number Conversion**

**Implicit:**

```javascript
+"42"; // 42
"42" - 0; // 42
"42" * 1; // 42
"42" / 1; // 42
true + 1; // 2
false + 1; // 1
null + 1; // 1
undefined + 1; // NaN
```

**Explicit:**

```javascript
Number("42"); // 42
Number("42.5"); // 42.5
Number("42px"); // NaN
Number(true); // 1
Number(false); // 0
Number(null); // 0
Number(undefined); // NaN
Number(""); // 0
Number(" "); // 0

parseInt("42px"); // 42
parseInt("42.8"); // 42
parseFloat("42.8"); // 42.8
parseFloat("42.8px"); // 42.8
```

**Special Cases:**

```javascript
Number([]); // 0
Number([42]); // 42
Number([1, 2]); // NaN
Number({}); // NaN
```

---

## ✅ Boolean Coercion

### **Falsy Values**

These values coerce to `false`:

```javascript
Boolean(0); // false
Boolean(-0); // false
Boolean(NaN); // false
Boolean(""); // false
Boolean(null); // false
Boolean(undefined); // false
Boolean(false); // false
```

### **Truthy Values**

Everything else is truthy:

```javascript
Boolean(1); // true
Boolean(-1); // true
Boolean("0"); // true (non-empty string)
Boolean("false"); // true (non-empty string)
Boolean([]); // true (empty array)
Boolean({}); // true (empty object)
Boolean(function () {}); // true
```

**Implicit Boolean Conversion:**

```javascript
if ("hello") {
  // truthy
  console.log("Runs");
}

const value = "text" || "default"; // "text"
const value2 = "" || "default"; // "default"
const value3 = 0 && "text"; // 0
const value4 = 1 && "text"; // "text"
```

---

## ⚖️ Comparison Operators

### **Loose Equality (==)**

Performs type coercion before comparison:

```javascript
// Number vs String
5 == "5"; // true
0 == ""; // true
0 == "0"; // true

// Boolean coercion
true == 1; // true
false == 0; // true
true == "1"; // true

// Null and Undefined
null == undefined; // true
null == 0; // false
undefined == 0; // false

// Objects
[] == false; // true
[] == ""; // true
[] == 0; // true
[""] == ""; // true
[0] == 0; // true
[1] == true; // true
```

**How == Works:**

1. If types are the same, compare with `===`
2. If `null` or `undefined`, they're equal to each other
3. If number and string, convert string to number
4. If boolean, convert to number
5. If object and primitive, convert object to primitive

---

### **Strict Equality (===)**

No type coercion - types and values must match:

```javascript
5 === "5"; // false
0 === ""; // false
true === 1; // false
null === undefined; // false
[] === []; // false (different references)
```

---

### **Comparison Operators (<, >, <=, >=)**

```javascript
// String to Number
"10" > 5; // true
"10" > "5"; // false (string comparison)

// String comparison (lexicographic)
"apple" < "banana"; // true
"10" < "9"; // true (string comparison!)

// Boolean to Number
true > 0; // true (1 > 0)
false < 1; // true (0 < 1)

// Null and Undefined
null >= 0; // true (null converts to 0)
null == 0; // false (special rule)
undefined < 0; // false
undefined > 0; // false
```

---

## 🔄 Object to Primitive Conversion

### **ToPrimitive Algorithm**

When an object needs to be converted to a primitive, JavaScript follows these steps:

1. Check for `Symbol.toPrimitive` method
2. Call `valueOf()`
3. Call `toString()`
4. Throw TypeError if no primitive is returned

```javascript
const obj = {
  valueOf() {
    return 42;
  },
  toString() {
    return "Object";
  },
};

console.log(obj + 1); // 43 (valueOf used)
console.log(String(obj)); // "Object" (toString used)
```

**With Symbol.toPrimitive:**

```javascript
const obj = {
  [Symbol.toPrimitive](hint) {
    if (hint === "number") {
      return 42;
    }
    if (hint === "string") {
      return "Object";
    }
    return null;
  },
};

console.log(obj + 1); // 43
console.log(String(obj)); // "Object"
console.log(obj == 42); // true
```

---

## ⚡ The Plus (+) Operator

The `+` operator has special behavior:

### **String Concatenation**

If either operand is a string, concatenate:

```javascript
"Hello" + " World"; // "Hello World"
"5" + 5; // "55"
5 + "5"; // "55"
"Value: " + null; // "Value: null"
```

### **Numeric Addition**

If both are numbers (or convertible):

```javascript
5 + 5; // 10
5 + true; // 6
5 + false; // 5
5 + null; // 5
```

### **Order Matters**

```javascript
1 + 2 + "3"; // "33" (left-to-right: (1+2)+'3')
"1" + 2 + 3; // "123" ('1'+2 = "12", then "12"+3)
```

---

## 🎯 Common Pitfalls

### **1. Array Coercion**

```javascript
// Arrays convert to strings
[] + []; // "" (empty string)
[] + {}; // "[object Object]"
{
}
+[]; // 0 (parsed as empty block + [])
{
}
+{}; // "[object Object][object Object]"

// Single element arrays
[1] + [2]; // "12"
[1, 2] + [3, 4]; // "1,23,4"
```

### **2. NaN Comparisons**

```javascript
NaN == NaN; // false
NaN === NaN; // false
Number.isNaN(NaN); // true
isNaN("hello"); // true (coerces to number first)
Number.isNaN("hello"); // false (no coercion)
```

### **3. Object.is() vs ===**

```javascript
Object.is(0, -0); // false (=== returns true)
Object.is(NaN, NaN); // true (=== returns false)
Object.is(5, 5); // true
```

### **4. Unexpected Coercions**

```javascript
true + true + true; // 3
true - true; // 0
[] == ![]; // true
9 + "1"; // "91"
9 - "1"; // 8
"2" * "3"; // 6
"10" / "2"; // 5
```

---

## 🏗️ Real-World Scenarios

### **1. Form Input Validation**

```javascript
const age = document.getElementById("age").value;

// WRONG - "18" == 18 is true
if (age == 18) {
  console.log("Exactly 18");
}

// CORRECT - Use explicit conversion
if (Number(age) === 18) {
  console.log("Exactly 18");
}

// Or use strict equality with conversion
if (parseInt(age, 10) === 18) {
  console.log("Exactly 18");
}
```

### **2. Default Values**

```javascript
// Using || (coerces to boolean)
function greet(name) {
  name = name || "Guest"; // Problem: empty string becomes "Guest"
  return `Hello, ${name}`;
}

// Better: Use nullish coalescing
function greet(name) {
  name = name ?? "Guest"; // Only null/undefined become "Guest"
  return `Hello, ${name}`;
}
```

### **3. Truthy/Falsy Checks**

```javascript
// Checking if array has items
if (array.length) {
  // Coerces length to boolean
  console.log("Has items");
}

// Checking if string exists
if (string) {
  // Coerces to boolean
  console.log("Has value");
}

// Be explicit when needed
if (value !== null && value !== undefined) {
  // More clear than if (value)
}
```

---

## 🧪 Interview Questions

### **Q1: What will these output?**

```javascript
console.log([] + []); // ?
console.log([] + {}); // ?
console.log({} + []); // ?
console.log(true + false); // ?
console.log("5" - 3); // ?
console.log("5" + 3); // ?
```

**Answers:**

- `""` - Empty string
- `"[object Object]"`
- `0` (or `"[object Object]"` in some contexts)
- `1`
- `2`
- `"53"`

---

### **Q2: Explain this behavior:**

```javascript
console.log(0 == ""); // true
console.log(0 == "0"); // true
console.log("" == "0"); // false
```

**Answer:** `==` applies different coercion rules. Empty string coerces to 0, but when comparing two strings, no coercion happens.

---

### **Q3: What's wrong with this code?**

```javascript
if (x == true) {
  // Do something
}
```

**Answer:** Using `== true` is redundant and can lead to unexpected behavior. Use `if (x)` instead, which checks for truthiness directly.

---

## 🎯 Best Practices

1. **Always use `===` and `!==`** unless you specifically need type coercion
2. **Be explicit with conversions** - use `Number()`, `String()`, `Boolean()`
3. **Use `parseInt()` with radix** - `parseInt(value, 10)`
4. **Validate user input** - Never trust implicit coercion with user data
5. **Use `Number.isNaN()`** instead of `isNaN()` to avoid coercion
6. **Use nullish coalescing (`??`)** for default values when 0 or `''` are valid
7. **Understand operator behavior** - `+` does concatenation if any operand is a string
8. **Avoid** comparing objects with primitives using `==`

---

## 📚 Quick Reference Table

| Value       | `Number()` | `String()`          | `Boolean()` |
| ----------- | ---------- | ------------------- | ----------- |
| `undefined` | `NaN`      | `"undefined"`       | `false`     |
| `null`      | `0`        | `"null"`            | `false`     |
| `true`      | `1`        | `"true"`            | `true`      |
| `false`     | `0`        | `"false"`           | `false`     |
| `""`        | `0`        | `""`                | `false`     |
| `"0"`       | `0`        | `"0"`               | `true`      |
| `"42"`      | `42`       | `"42"`              | `true`      |
| `[]`        | `0`        | `""`                | `true`      |
| `[42]`      | `42`       | `"42"`              | `true`      |
| `{}`        | `NaN`      | `"[object Object]"` | `true`      |

---

## 🎓 Key Takeaways

- JavaScript performs **automatic type coercion** in many operations
- **`==` performs coercion**, `===` does not
- **Falsy values**: `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`
- Everything else is **truthy**
- **Be explicit** with conversions to avoid bugs
- Understand **operator behavior** - especially `+`
- **ToPrimitive** algorithm: `Symbol.toPrimitive` → `valueOf()` → `toString()`

---

**Remember:** When in doubt, be explicit with your type conversions!
