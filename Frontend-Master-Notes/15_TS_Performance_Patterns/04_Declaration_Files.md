# TypeScript Declaration Files (.d.ts)

## Core Concept

Declaration files (`.d.ts`) provide type information for JavaScript code without changing runtime behavior. They're essential for TypeScript performance, third-party library integration, and maintaining type safety in mixed codebases.

---

## Declaration File Basics

### What Are Declaration Files?

```typescript
// user.js (JavaScript implementation)
export function createUser(name, age) {
  return {
    name,
    age,
    greet() {
      return `Hello, I'm ${name}`;
    },
  };
}

// user.d.ts (TypeScript declarations)
export interface User {
  name: string;
  age: number;
  greet(): string;
}

export function createUser(name: string, age: number): User;
```

### Automatic Declaration Generation

```json
// tsconfig.json - Generate .d.ts files
{
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true, // Source maps for declarations
    "declarationDir": "./dist/types",
    "emitDeclarationOnly": false
  }
}
```

---

## Ambient Declarations

### Global Declarations

```typescript
// global.d.ts - Extend global scope
declare global {
  interface Window {
    APP_CONFIG: {
      apiUrl: string;
      version: string;
    };
  }

  var ENV: "development" | "production" | "test";

  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: string;
      API_KEY: string;
      DATABASE_URL: string;
    }
  }
}

// Usage in any file
window.APP_CONFIG.apiUrl; // string
ENV; // 'development' | 'production' | 'test'
process.env.API_KEY; // string

export {}; // Make this a module
```

### Module Augmentation

```typescript
// express-extensions.d.ts
import "express";

declare module "express" {
  interface Request {
    user?: {
      id: string;
      email: string;
      role: "admin" | "user";
    };
  }

  interface Response {
    sendSuccess<T>(data: T): void;
    sendError(message: string, code?: number): void;
  }
}

// Usage in route handlers
app.get("/profile", (req, res) => {
  if (!req.user) {
    return res.sendError("Unauthorized", 401);
  }
  res.sendSuccess({ profile: req.user });
});
```

---

## DefinitelyTyped (@types)

### Using Community Type Definitions

```bash
# Install type definitions from DefinitelyTyped
npm install --save-dev @types/node
npm install --save-dev @types/react
npm install --save-dev @types/lodash
npm install --save-dev @types/jest
```

### When @types Don't Exist

```typescript
// types/my-library.d.ts - Custom declarations for untyped library
declare module "my-untyped-library" {
  export interface LibraryConfig {
    apiKey: string;
    timeout?: number;
  }

  export function initialize(config: LibraryConfig): void;

  export function fetchData<T>(url: string): Promise<T>;

  export default class Library {
    constructor(config: LibraryConfig);
    connect(): Promise<void>;
    disconnect(): void;
  }
}

// Usage
import Library, { initialize, fetchData } from "my-untyped-library";

initialize({ apiKey: "abc123", timeout: 5000 });
const data = await fetchData<User>("/users/1");
```

---

## Declaration Merging

### Interface Merging

```typescript
// user.d.ts
interface User {
  id: string;
  name: string;
}

// user-extensions.d.ts - Merged with above
interface User {
  email: string;
  age: number;
}

// Result: User has all properties
const user: User = {
  id: "1",
  name: "Alice",
  email: "alice@example.com",
  age: 30,
};
```

### Namespace Merging

```typescript
// api.d.ts
declare namespace API {
  interface User {
    id: string;
    name: string;
  }

  function fetchUser(id: string): Promise<User>;
}

// api-extensions.d.ts
declare namespace API {
  interface Product {
    id: string;
    title: string;
  }

  function fetchProduct(id: string): Promise<Product>;
}

// Usage - Both User and Product available
const user = await API.fetchUser("1");
const product = await API.fetchProduct("2");
```

---

## Real-World Pattern: Third-Party Library Integration

### Wrapping Untyped JavaScript Library

```typescript
// types/legacy-library.d.ts
declare module "legacy-data-grid" {
  export interface GridOptions {
    columns: ColumnConfig[];
    data: any[];
    pagination?: {
      enabled: boolean;
      pageSize: number;
    };
    sorting?: {
      enabled: boolean;
      multiColumn?: boolean;
    };
  }

  export interface ColumnConfig {
    field: string;
    header: string;
    width?: number;
    sortable?: boolean;
    formatter?: (value: any) => string;
  }

  export interface GridInstance {
    render(container: HTMLElement): void;
    refresh(data: any[]): void;
    destroy(): void;
    on(event: "rowClick", handler: (row: any) => void): void;
    on(
      event: "sort",
      handler: (field: string, direction: "asc" | "desc") => void,
    ): void;
  }

  export default function createGrid(options: GridOptions): GridInstance;
}

// Usage with type safety
import createGrid, { GridOptions } from "legacy-data-grid";

const options: GridOptions = {
  columns: [
    { field: "name", header: "Name", sortable: true },
    { field: "age", header: "Age", width: 100 },
  ],
  data: users,
  pagination: {
    enabled: true,
    pageSize: 20,
  },
};

const grid = createGrid(options);
grid.on("rowClick", (row) => {
  console.log("Clicked:", row);
});
```

---

## Real-World Pattern: Environment Variables

### Type-Safe Environment Configuration

```typescript
// types/env.d.ts
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      // Required variables
      NODE_ENV: "development" | "production" | "test";
      DATABASE_URL: string;
      API_KEY: string;

      // Optional variables
      PORT?: string;
      LOG_LEVEL?: "debug" | "info" | "warn" | "error";
      REDIS_URL?: string;

      // Feature flags
      ENABLE_ANALYTICS?: "true" | "false";
      ENABLE_CACHING?: "true" | "false";
    }
  }
}

export {};

// Usage with autocomplete and type checking
const config = {
  port: parseInt(process.env.PORT || "3000", 10),
  database: process.env.DATABASE_URL, // string (required)
  apiKey: process.env.API_KEY, // string (required)
  logLevel: process.env.LOG_LEVEL || "info",
  analytics: process.env.ENABLE_ANALYTICS === "true",
};

// TypeScript error if typo
process.env.API_KY; // Property 'API_KY' does not exist
```

---

## Real-World Pattern: Window Extensions

### Custom Window Properties

```typescript
// types/window.d.ts
interface AppConfig {
  apiUrl: string;
  version: string;
  features: {
    darkMode: boolean;
    notifications: boolean;
  };
}

interface Analytics {
  track(event: string, properties?: Record<string, any>): void;
  identify(userId: string, traits?: Record<string, any>): void;
}

declare global {
  interface Window {
    APP_CONFIG: AppConfig;
    analytics: Analytics;

    // Third-party scripts
    gtag?: (...args: any[]) => void;
    dataLayer?: any[];

    // Custom utilities
    __DEV__: boolean;
    __REDUX_DEVTOOLS_EXTENSION__?: any;
  }
}

export {};

// Usage with type safety
if (window.__DEV__) {
  console.log("Development mode");
}

window.analytics.track("page_view", {
  path: location.pathname,
});

// Access config
const apiUrl = window.APP_CONFIG.apiUrl;
```

---

## Real-World Pattern: Module Wildcards

### Importing Non-TypeScript Files

```typescript
// types/assets.d.ts
declare module '*.svg' {
  const content: React.FC<React.SVGProps<SVGSVGElement>>;
  export default content;
}

declare module '*.png' {
  const content: string;
  export default content;
}

declare module '*.jpg' {
  const content: string;
  export default content;
}

declare module '*.css' {
  const classes: Record<string, string>;
  export default classes;
}

declare module '*.module.css' {
  const classes: Record<string, string>;
  export default classes;
}

// Usage in React components
import Logo from './logo.svg';
import avatarImage from './avatar.png';
import styles from './Button.module.css';

function Header() {
  return (
    <header className={styles.header}>
      <Logo width={100} height={50} />
      <img src={avatarImage} alt="Avatar" />
    </header>
  );
}
```

---

## Real-World Pattern: API Client Types

### Generating Types from OpenAPI/Swagger

```typescript
// types/api-client.d.ts
declare module "api-client" {
  // Auto-generated from OpenAPI spec
  export interface User {
    id: string;
    email: string;
    name: string;
    createdAt: string;
    updatedAt: string;
  }

  export interface Product {
    id: string;
    title: string;
    price: number;
    inventory: number;
  }

  export interface ApiResponse<T> {
    data: T;
    meta: {
      page: number;
      perPage: number;
      total: number;
    };
  }

  export interface ApiClient {
    users: {
      getAll(params?: { page?: number }): Promise<ApiResponse<User[]>>;
      getById(id: string): Promise<User>;
      create(data: Omit<User, "id" | "createdAt" | "updatedAt">): Promise<User>;
      update(id: string, data: Partial<User>): Promise<User>;
      delete(id: string): Promise<void>;
    };

    products: {
      getAll(): Promise<Product[]>;
      getById(id: string): Promise<Product>;
    };
  }

  export function createClient(config: {
    apiUrl: string;
    apiKey: string;
  }): ApiClient;
}

// Usage
import { createClient, User } from "api-client";

const client = createClient({
  apiUrl: "https://api.example.com",
  apiKey: process.env.API_KEY,
});

const users = await client.users.getAll({ page: 1 });
const user = await client.users.getById("123");
```

---

## Performance Optimization with Declaration Files

### Skip Type Checking for node_modules

```json
// tsconfig.json
{
  "compilerOptions": {
    "skipLibCheck": true, // Skip .d.ts type checking in node_modules
    "skipDefaultLibCheck": true // Skip default lib.d.ts files
  }
}
```

### Separate Declaration and Implementation Builds

```json
// tsconfig.build.json - For building JS
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": false,  // Don't generate .d.ts during dev builds
    "noEmit": false
  }
}

// tsconfig.types.json - For generating types only
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "emitDeclarationOnly": true,  // Only generate .d.ts files
    "outDir": "./dist/types"
  }
}
```

```bash
# Fast dev build without types
tsc -p tsconfig.build.json

# Separate types generation
tsc -p tsconfig.types.json
```

---

## Best Practices

### 1. Use Triple-Slash Directives Sparingly

```typescript
// ❌ Avoid unless necessary
/// <reference types="node" />

// ✅ Prefer explicit imports
import { EventEmitter } from "events";
```

### 2. Keep Declarations DRY

```typescript
// ❌ Duplicated types
declare module "lib-a" {
  export interface User {
    id: string;
    name: string;
  }
}

declare module "lib-b" {
  export interface User {
    id: string;
    name: string;
  }
}

// ✅ Share types
// types/shared.d.ts
export interface User {
  id: string;
  name: string;
}

// types/lib-a.d.ts
import { User } from "./shared";
declare module "lib-a" {
  export { User };
}
```

### 3. Document Complex Declarations

```typescript
/**
 * Utility type for creating a deeply partial version of T.
 * Useful for patch operations and partial updates.
 *
 * @example
 * type User = { profile: { name: string; age: number } };
 * type PartialUser = DeepPartial<User>;
 * // { profile?: { name?: string; age?: number } }
 */
export type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};
```

### 4. Version Control for @types

```json
// package.json - Lock @types versions
{
  "devDependencies": {
    "@types/node": "18.11.9",
    "@types/react": "18.0.25"
  }
}
```

---

## Troubleshooting

### Declaration File Not Found

```json
// tsconfig.json
{
  "compilerOptions": {
    "typeRoots": [
      "./node_modules/@types",
      "./types" // Custom type declarations
    ],
    "types": [] // Include all types (default)
  }
}
```

### Module Resolution Issues

```typescript
// types/custom-types.d.ts
declare module 'my-library' {
  const value: any;
  export default value;
}

// If still not found, check tsconfig paths
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "my-library": ["./types/custom-types"]
    }
  }
}
```

---

## Key Takeaways

1. **Declaration files** provide types without runtime overhead
2. **skipLibCheck** significantly improves compilation speed
3. **Module augmentation** extends third-party types safely
4. **Global declarations** for window, process, and environment variables
5. **@types packages** from DefinitelyTyped for popular libraries
6. **Wildcard modules** for asset imports (images, CSS, etc.)
7. **Triple-slash directives** are legacy - prefer imports
8. **Declaration merging** combines interface definitions
9. Generate types separately with **emitDeclarationOnly**
10. Keep custom types in **./types** directory with proper **typeRoots** configuration
