# esbuild - Extremely Fast JavaScript Bundler

## Core Concepts

esbuild is a JavaScript bundler and minifier written in Go, offering 10-100x faster build times than traditional JavaScript-based bundlers. It's the engine behind Vite's dependency pre-bundling and is increasingly used for production builds. Understanding esbuild is crucial for modern frontend performance optimization.

## Go-Based Performance

### Why esbuild is Fast

- **Written in Go**: Compiled to native code, parallelism built-in
- **Memory efficient**: Shared memory, minimal allocations
- **Parallel processing**: Uses all CPU cores
- **No parsing overhead**: AST is represented in memory efficiently

### Basic CLI Usage

```bash
# Bundle a file
esbuild src/app.ts --bundle --outfile=dist/app.js

# Minify
esbuild src/app.ts --bundle --minify --outfile=dist/app.min.js

# Watch mode
esbuild src/app.ts --bundle --outfile=dist/app.js --watch

# With source maps
esbuild src/app.ts --bundle --sourcemap --outfile=dist/app.js
```

## JavaScript API

### Basic Build Configuration

```typescript
import * as esbuild from 'esbuild'

const result = await esbuild.build({
  entryPoints: ['src/index.tsx'],
  bundle: true,
  minify: true,
  sourcemap: true,
  target: ['es2020'],
  outfile: 'dist/bundle.js',
  loader: {
    '.ts': 'ts',
    '.tsx': 'tsx',
    '.png': 'dataurl',
    '.svg': 'file',
  },
  define: {
    'process.env.NODE_ENV': JSON.stringify('production'),
  },
})

console.log('Build complete:', result)
```

### Advanced Configuration

```typescript
import * as esbuild from 'esbuild'

await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  splitting: true,
  format: 'esm',
  outdir: 'dist',
  
  // Tree shaking
  treeShaking: true,
  
  // External packages
  external: ['react', 'react-dom'],
  
  // Platform
  platform: 'browser',
  
  // Target
  target: ['chrome90', 'firefox88', 'safari14'],
  
  // Minification
  minify: true,
  minifyWhitespace: true,
  minifyIdentifiers: true,
  minifySyntax: true,
  
  // Code splitting
  splitting: true,
  chunkNames: 'chunks/[name]-[hash]',
  
  // Source maps
  sourcemap: 'external',
  
  // Meta file for analysis
  metafile: true,
  
  // Error formatting
  logLevel: 'info',
  color: true,
})
```

## Plugin System

### Custom Plugin

```typescript
import * as esbuild from 'esbuild'

const envPlugin: esbuild.Plugin = {
  name: 'env',
  setup(build) {
    // Intercept .env file imports
    build.onResolve({ filter: /\\.env\$/ }, args => ({
      path: args.path,
      namespace: 'env-ns',
    }))
    
    // Load .env files
    build.onLoad({ filter: /.*/, namespace: 'env-ns' }, async () => {
      return {
        contents: JSON.stringify(process.env),
        loader: 'json',
      }
    })
  },
}

await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  plugins: [envPlugin],
  outfile: 'dist/bundle.js',
})
```

### Markdown Plugin

```typescript
import fs from 'fs'
import * as esbuild from 'esbuild'

const markdownPlugin: esbuild.Plugin = {
  name: 'markdown',
  setup(build) {
    build.onLoad({ filter: /\\.md\$/ }, async (args) => {
      const text = await fs.promises.readFile(args.path, 'utf8')
      
      const contents = \
        export default \;
      \
      
      return {
        contents,
        loader: 'js',
      }
    })
  },
}

await esbuild.build({
  entryPoints: ['src/app.ts'],
  bundle: true,
  plugins: [markdownPlugin],
  outfile: 'dist/app.js',
})
```

## Limitations

### No Hot Module Replacement

esbuild doesn't have built-in HMR:

```typescript
// Workaround: Use with a dev server
import * as esbuild from 'esbuild'
import { serve } from 'esbuild-serve'

const ctx = await esbuild.context({
  entryPoints: ['src/index.ts'],
  bundle: true,
  outdir: 'dist',
})

await ctx.watch()
await ctx.serve({
  servedir: 'dist',
  port: 3000,
})
```

### Limited Code Splitting

```typescript
// Code splitting only works with ESM format
await esbuild.build({
  entryPoints: ['src/page1.ts', 'src/page2.ts'],
  bundle: true,
  splitting: true,
  format: 'esm', // Required for splitting
  outdir: 'dist',
})
```

## Use Cases

### Library Builds

```typescript
// Build for multiple formats
await Promise.all([
  // ESM build
  esbuild.build({
    entryPoints: ['src/index.ts'],
    bundle: true,
    format: 'esm',
    outfile: 'dist/index.esm.js',
    external: ['react', 'react-dom'],
  }),
  
  // CommonJS build
  esbuild.build({
    entryPoints: ['src/index.ts'],
    bundle: true,
    format: 'cjs',
    outfile: 'dist/index.cjs.js',
    external: ['react', 'react-dom'],
  }),
  
  // UMD build
  esbuild.build({
    entryPoints: ['src/index.ts'],
    bundle: true,
    format: 'iife',
    globalName: 'MyLibrary',
    outfile: 'dist/index.umd.js',
  }),
])
```

### Fast CI Builds

```typescript
// Optimized for CI speed
await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  minify: true,
  sourcemap: false, // Skip source maps for faster builds
  treeShaking: true,
  target: ['es2020'],
  outfile: 'dist/bundle.js',
  logLevel: 'error', // Only show errors
})
```

## esbuild-loader for Webpack

### Webpack Integration

```typescript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\\.tsx?\$/,
        loader: 'esbuild-loader',
        options: {
          loader: 'tsx',
          target: 'es2020',
        },
      },
    ],
  },
  
  optimization: {
    minimizer: [
      new ESBuildMinifyPlugin({
        target: 'es2020',
        css: true,
      }),
    ],
  },
}
```

## Real-World Production Setup

```typescript
import * as esbuild from 'esbuild'

const production = process.env.NODE_ENV === 'production'

await esbuild.build({
  entryPoints: ['src/index.tsx'],
  bundle: true,
  minify: production,
  sourcemap: !production,
  target: ['chrome90', 'firefox88', 'safari14'],
  outdir: 'dist',
  
  loader: {
    '.png': 'file',
    '.jpg': 'file',
    '.svg': 'dataurl',
  },
  
  define: {
    'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
  },
  
  external: production ? [] : ['react', 'react-dom'],
  
  metafile: true,
  
  plugins: [
    /* custom plugins */
  ],
})
```

## Best Practices

1. **Use for library builds** - Fast, simple, reliable
2. **CI/CD optimization** - 10-100x faster than Webpack/Rollup
3. **TypeScript compilation** - No need for separate tsc
4. **Webpack replacement** - Use esbuild-loader for Webpack
5. **Dev builds** - Fast rebuilds for development
6. **Production bundles** - Smaller output than alternatives
7. **Plugin sparingly** - Keep build fast
8. **External deps** - Mark peer dependencies as external
9. **Format selection** - Choose correct format (esm/cjs/iife)
10. **Monitoring** - Use metafile for bundle analysis

## 10 Key Takeaways

1. **10-100x faster** than JavaScript-based bundlers
2. **Written in Go** for native performance and parallelism
3. **No HMR** - use with Vite or custom dev server
4. **Limited code splitting** - only with ESM format
5. **Perfect for libraries** - fast, multiple format outputs
6. **Great for CI** - dramatically reduces build times
7. **Built-in TypeScript** - no separate compilation step
8. **Minimal config** - simple API, sensible defaults
9. **esbuild-loader** - speed up Webpack projects
10. **Production ready** - used by Vite, Snowpack, others
