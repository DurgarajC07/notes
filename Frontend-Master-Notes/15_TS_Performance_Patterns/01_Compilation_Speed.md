# TypeScript Compilation Speed

## Core Concept

Large TypeScript projects can suffer from slow compilation times. Project references, incremental builds, and strategic configuration improve build performance without sacrificing type safety.

---

## Incremental Compilation

```json
// tsconfig.json
{
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo",
    "composite": true
  }
}
```

This creates a `.tsbuildinfo` file that caches type information between builds, dramatically speeding up subsequent compilations.

---

## Project References

```
my-app/
├── packages/
│   ├── shared/
│   │   ├── src/
│   │   └── tsconfig.json
│   ├── api/
│   │   ├── src/
│   │   └── tsconfig.json
│   └── ui/
│       ├── src/
│       └── tsconfig.json
└── tsconfig.json
```

```json
// packages/shared/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "outDir": "dist"
  },
  "include": ["src"]
}

// packages/api/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "dist"
  },
  "references": [
    { "path": "../shared" }
  ]
}

// Root tsconfig.json
{
  "files": [],
  "references": [
    { "path": "./packages/shared" },
    { "path": "./packages/api" },
    { "path": "./packages/ui" }
  ]
}
```

Build with: `tsc --build` or `tsc -b`

---

## Skip Lib Check

```json
{
  "compilerOptions": {
    "skipLibCheck": true
  }
}
```

Skips type checking of declaration files (`.d.ts`). **Huge speed improvement** but may miss errors in dependencies.

**Use when:** Dependencies are well-typed and you trust them.

---

## Exclude Unnecessary Files

```json
{
  "compilerOptions": {
    "outDir": "dist"
  },
  "include": ["src/**/*"],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.spec.ts",
    "**/*.test.ts",
    "**/__tests__/**"
  ]
}
```

Don't compile test files, build outputs, or dependencies.

---

## Module Resolution Strategy

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler", // or "node16"
    "resolveJsonModule": false // if not needed
  }
}
```

`"bundler"` is faster for modern bundlers like Vite, esbuild.

---

## Disable Unused Options

```json
{
  "compilerOptions": {
    // Disable if not needed
    "sourceMap": false,
    "declarationMap": false,
    "declaration": false, // for apps (not libraries)

    // Disable strict checks selectively (not recommended)
    "strictNullChecks": true, // keep this
    "noImplicitAny": true, // keep this
    "strictFunctionTypes": false // only if needed
  }
}
```

---

## Use Type-Only Imports

```typescript
// ❌ Slow - imports runtime value
import { User } from "./types";

// ✅ Fast - type-only import
import type { User } from "./types";

// ✅ Mixed imports
import { fetchUser, type User } from "./api";
```

Type-only imports are erased at compile time, reducing work.

---

## Lazy Type Loading

```typescript
// ❌ Eager loading
import { HugeType } from "./huge-library";

function process(data: HugeType) {
  // ...
}

// ✅ Lazy loading with dynamic import type
function process(data: import("./huge-library").HugeType) {
  // ...
}

// ✅ Or use type parameter
async function processAsync<T extends import("./huge-library").HugeType>(
  data: T,
) {
  // ...
}
```

---

## Optimize TypeScript Cache

```bash
# Clear TypeScript cache
rm -rf node_modules/.cache/typescript
rm -f tsconfig.tsbuildinfo

# Rebuild
npm run build
```

---

## Parallel Compilation

```json
// package.json
{
  "scripts": {
    "build": "tsc -b --parallel",
    "watch": "tsc -b --watch --preserveWatchOutput"
  }
}
```

---

## Watch Mode Optimization

```json
{
  "compilerOptions": {
    "incremental": true,
    "assumeChangesOnlyAffectDirectDependencies": true
  },
  "watchOptions": {
    "watchFile": "useFsEvents",
    "watchDirectory": "useFsEvents",
    "fallbackPolling": "dynamicPriority",
    "synchronousWatchDirectory": true,
    "excludeDirectories": ["**/node_modules", "**/.git"]
  }
}
```

---

## Webpack Integration

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: [
          {
            loader: "ts-loader",
            options: {
              transpileOnly: true, // Skip type checking (use fork-ts-checker-webpack-plugin)
              experimentalWatchApi: true,
            },
          },
        ],
      },
    ],
  },
  plugins: [
    new ForkTsCheckerWebpackPlugin({
      typescript: {
        configFile: "tsconfig.json",
      },
    }),
  ],
};
```

---

## Vite/esbuild (Fastest)

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  esbuild: {
    logOverride: { "this-is-undefined-in-esm": "silent" },
  },
});
```

Vite uses esbuild for transpilation (100x faster than tsc) and only type-checks on demand.

---

## Best Practices

✅ **Enable incremental compilation**  
✅ **Use project references** for monorepos  
✅ **Skip lib check** in production builds  
✅ **Exclude test files** from compilation  
✅ **Use type-only imports**  
✅ **Leverage modern bundlers** (Vite, esbuild)  
❌ **Don't compile node_modules**  
❌ **Don't generate unnecessary source maps**  
❌ **Don't check types twice** (use fork-ts-checker)

---

## Key Takeaways

1. **Incremental builds cache type info** for speed
2. **Project references** enable parallel compilation
3. **skipLibCheck** dramatically improves speed
4. **Type-only imports** reduce work
5. **Modern bundlers** (Vite) are 100x faster
6. **Separate type checking** from transpilation
7. **Watch mode optimizations** improve DX
