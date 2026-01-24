# JavaScript Module System

> **ESM vs CommonJS, module resolution, and dynamic imports**

---

## 🎯 Overview

JavaScript's module system enables code organization, encapsulation, and reusability. Understanding the differences between CommonJS (CJS) and ECMAScript Modules (ESM), how module resolution works, and dynamic imports is crucial for modern JavaScript development.

---

## 📚 Core Concepts

### **Concept 1: Module Types and History**

**CommonJS (CJS)** was created for Node.js and uses `require()` and `module.exports`:

```javascript
// CommonJS - math.js
function add(a, b) {
  return a + b;
}

module.exports = { add };

// CommonJS - main.js
const math = require("./math");
console.log(math.add(2, 3)); // 5
```

**ECMAScript Modules (ESM)** is the standard module system for JavaScript:

```javascript
// ESM - math.js
export function add(a, b) {
  return a + b;
}

// ESM - main.js
import { add } from "./math.js";
console.log(add(2, 3)); // 5
```

**Key Differences:**

| Feature                | CommonJS                     | ESM                 |
| ---------------------- | ---------------------------- | ------------------- |
| Syntax                 | `require()`/`module.exports` | `import`/`export`   |
| Loading                | Synchronous                  | Asynchronous        |
| When resolved          | Runtime                      | Parse time (static) |
| Tree-shaking           | No                           | Yes                 |
| Top-level await        | No                           | Yes                 |
| `this` at module level | `module.exports`             | `undefined`         |
| File extension         | `.js`                        | `.js` or `.mjs`     |

### **Concept 2: Export Patterns**

**Named Exports (ESM):**

```javascript
// Multiple named exports
export const PI = 3.14159;
export function square(x) {
  return x * x;
}
export class Calculator {
  add(a, b) {
    return a + b;
  }
}

// Or export at the end
const PI = 3.14159;
function square(x) {
  return x * x;
}
export { PI, square };

// Renamed exports
const internalName = "value";
export { internalName as publicName };
```

**Default Exports (ESM):**

```javascript
// Default export - one per module
export default function multiply(a, b) {
  return a * b;
}

// Or
function multiply(a, b) {
  return a * b;
}
export default multiply;

// Default + named exports
export default function main() {}
export const helper = () => {};
```

**CommonJS Exports:**

```javascript
// Single export
module.exports = function () {};

// Multiple exports
module.exports = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
};

// Or using exports shorthand
exports.add = (a, b) => a + b;
exports.subtract = (a, b) => a - b;
```

---

## 🔥 Practical Examples

### **Example 1: Dynamic Imports**

Dynamic imports load modules on-demand, enabling code splitting:

```javascript
// Static import - loaded upfront
import { heavyFunction } from "./heavy-module.js";

// Dynamic import - loaded when needed
async function loadModule() {
  const module = await import("./heavy-module.js");
  module.heavyFunction();
}

// Conditional loading
const language = "es";
const translations = await import(`./i18n/${language}.js`);

// Route-based code splitting (React)
const HomePage = lazy(() => import("./pages/Home"));
const AboutPage = lazy(() => import("./pages/About"));

// Real-world: load analytics only on production
if (process.env.NODE_ENV === "production") {
  const analytics = await import("./analytics.js");
  analytics.init();
}
```

### **Example 2: Module Singleton Pattern**

Modules are cached and execute once:

```javascript
// database.js
let connection = null;

export function connect() {
  if (!connection) {
    connection = createDatabaseConnection();
    console.log("Database connected");
  }
  return connection;
}

// Multiple imports share the same instance
// app.js
import { connect } from "./database.js";
connect(); // Logs: Database connected

// routes.js
import { connect } from "./database.js";
connect(); // No log - reuses cached connection
```

---

## 🏗️ Best Practices

1. **Use ESM for new projects** - ESM is the standard and enables better tooling, tree-shaking, and static analysis
2. **Always include file extensions in ESM** - Use `.js`, `.mjs`, or specify extensions to ensure compatibility across environments

3. **Prefer named exports** - Named exports are more refactor-friendly and prevent naming conflicts

---

## 🧪 Interview Questions

### **Q1: What's the difference between CommonJS and ESM?**

**Answer:** CommonJS is synchronous and uses `require()`/`module.exports`, resolved at runtime. ESM is asynchronous, uses `import`/`export`, resolved at parse time (static), supports tree-shaking and top-level await. ESM is the JavaScript standard.

### **Q2: How does tree-shaking work with ES modules?**

**Answer:** Tree-shaking is dead code elimination enabled by ESM's static structure. Since imports/exports are static, bundlers can analyze which exports are used and remove unused code. CommonJS can't be tree-shaken because `require()` is dynamic.

---

## 📚 Key Takeaways

- ESM is the JavaScript standard - use it for new projects
- CommonJS is synchronous, ESM is asynchronous with static analysis
- Always include file extensions in ESM imports
- Use dynamic imports for code splitting and performance
- Modules are cached and execute once - they act as singletons
- Top-level await only works in ESM

---

**Master javascript module system for production-ready code!**
